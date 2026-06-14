# 资产导入：自定义字段注入链路 & Update分支 Checkout/Checkin 判定

---

## 一、自定义字段（CustomField）在导入链路中的动态注入

### 总览：三条注入链路

```
CSV 表头匹配（用户可选项）
  └─ 注入 fieldMap → getColumns() 追加 CustomField 到下拉选项
CSV 行数据解析
  └─ 绕过 fillable → Model::unguard() + 直接属性赋值 $asset->{$db_column} = $val
模型 save() 校验
  └─ 注入 rules → Asset::save() 前合并 customFieldValidationRules()
```

---

### 1.1 注入 fieldMap：自定义字段如何进入用户可选列

**触发位置**：[Livewire/Importer.php - getColumns()](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Livewire/Importer.php#L100-L150)

```php
private function getColumns($type)
{
    // ... 先从硬编码的 $assets_fields / $users_fields 等数组取基础字段 ...

    if ($type == 'asset') {
        // 步骤1：加一个分隔符条目
        $results['-'] = '———'.trans('admin/custom_fields/general.custom_fields').'———'';

        // 步骤2：遍历数据库中的所有 CustomField，动态追加
        foreach (CustomField::orderBy('name')->get() as $field) {
            // 键 = db_column（形如 _snipeit_cpu_cores_42）
            // 值 = 用户友好名称（如 "CPU 核心数"）
            $results[$field->db_column_name()] = $field->name;
        }
    }
    return $results;
}
```

**db_column_name() 的实现**：
```php
// CustomField.php L353-L356
public function db_column_name()
{
    return $this->db_column;  // 直接返回 DB 存储的列名，格式如 _snipeit_{slug}_{id}
}
```

**db_column 的命名规则（由 convertUnicodeDbSlug() 生成）**：
```php
// CustomField.php L460-L472
public function convertUnicodeDbSlug($original = null)
{
    $name = $original ? $original : $this->name;
    $id = $this->id ? $this->id : 'xx';

    // 统一转成 slug，前缀固定 _snipeit_
    $long_slug = '_snipeit_'.Utf8Slugger::slugify($name, '_');

    // 截断到 50 字符 + '_' + id，确保列名唯一且不超限
    return substr($long_slug, 0, 50).'_'.$id;
}
```

**自动匹配阶段对自定义字段的识别**：
```
[自动匹配逻辑 updatingTypeOfImport()]
  ├─ 第1级：strcasecmp(表头名, 系统字段显示名)
  │      → 对自定义字段：比较表头名 vs CustomField.name（用户友好名）
  │      → 例如 CSV 表头 "CPU 核心数" 会匹配到 $results['_snipeit_cpu_42'] = "CPU 核心数"
  │
  └─ 第2级：遍历 $aliases_fields 别名表
         → 注意：$aliases_fields 是硬编码的，不包含自定义字段别名
         → 自定义字段只能靠第1级精确匹配（case-insensitive）
```

---

### 1.2 绕过 fillable：自定义字段不在 `$fillable` 里如何写入 DB

**关键事实**：Asset 模型的 `$fillable` 数组是**硬编码**的约 26 个字段，不包含任何动态自定义字段。

```php
// Asset.php L147-L175
protected $fillable = [
    'asset_tag', 'name', 'model_id', 'status_id', ... // 约 26 个
    // ⚠️ 这里没有任何 _snipeit_ 开头的自定义字段列
];
```

**绕过方案 A：全局 `Model::unguard()`**

在导入最外层开启：
```php
// Importer.php L169-L171
DB::transaction(function () use ($headerRow) {
    $importedItemsCount = 0;
    Model::unguard();  // ← 全局关闭 fillable 保护！

    foreach ($this->csv->getRecords($headerRow) as $row) {
        // ... 所有操作都在 unguard 范围内 ...
        $this->handle($row);
    }
    Model::reguard();  // ← 循环结束后恢复
});
```

> 💡 这意味着：**整个导入事务期间，任何模型的任何属性都可以被批量赋值**，不经过 fillable 白名单。

**绕过方案 B：自定义字段单独直接属性赋值**

即使有 unguard，自定义字段也**不通过 fill → update/save 的批量赋值路径**，而是单独处理：

```php
// AssetImporter.php L185-L206
// 注意：$customFieldsToSave 是从 $this->item['custom_fields'] 来的，不经过 sanitizeItemForStoring()

// Apply custom fields to asset attributes if they exist
$customFieldsToSave = [];
if (array_key_exists('custom_fields', $this->item)) {
    foreach ($this->item['custom_fields'] as $custom_field => $val) {
        // ↓ 直接属性赋值（即使不在 fillable 里也能写）
        $asset->{$custom_field} = $val;
        $customFieldsToSave[$custom_field] = $val;
    }
}

// Update 模式下的二次 update 调用（避免 save() 两次）
if ($editingAsset && ! empty($customFieldsToSave)) {
    $asset->update($customFieldsToSave);  // ← 在 unguard 下，可直接批量赋值
    $success = true;
} elseif (! $editingAsset) {
    // 新建模式：直接 save() 就把上面 $asset->{$custom_field} 持久化了
    $success = $asset->save();
}
```

**自定义字段数据的来源**：
```php
// AssetImporter.php L43-L60
if ($this->customFields) {
    foreach ($this->customFields as $customField) {
        // ↓ 直接从 CSV row 取，不走 fieldMap（用 CustomField.name 匹配表头）
        $customFieldValue = $this->array_smart_custom_field_fetch($row, $customField);

        if (! is_null($customFieldValue)) {
            if ($customField->field_encrypted == 1) {
                // 加密字段 → Crypt::encrypt() 后存储
                $this->item['custom_fields'][$customField->db_column_name()] = Crypt::encrypt($customFieldValue);
            } else {
                $this->item['custom_fields'][$customField->db_column_name()] = $customFieldValue;
            }
        } else {
            // CSV 中没有该列值 → 设为 null（清空旧值）
            $this->item['custom_fields'][$customField->db_column_name()] = null;
        }
    }
}
```

**`array_smart_custom_field_fetch()` 的实现**：
```php
// Importer.php L279-L284
public function array_smart_custom_field_fetch(array $array, $key)
{
    // ↓ 注意：用的是 $key->name（自定义字段的显示名，如 "CPU 核心数"）
    //   而不是 db_column_name（_snipeit_cpu_42）
    $index_name = strtolower($key->name);

    return array_key_exists($index_name, $array) ? trim($array[$index_name]) : false;
}
```

> ⚠️ **关键差异**：系统标准字段走 `findCsvMatch()` → 通过 `fieldMap` 把系统字段标识翻译成 CSV 列名；自定义字段走 `array_smart_custom_field_fetch()` → 直接用 CustomField.name 匹配 CSV 列名，**不经过 fieldMap 的映射反转**。

**`populateCustomFields()` 只保留表头中出现的自定义字段**：
```php
// Importer.php L203-L221
protected function populateCustomFields($headerRow)
{
    // 步骤1：把所有 CustomField 转成 name => field 对象的索引数组
    $this->customFields = CustomField::All()->reduce(function ($nameLookup, $field) {
        $nameLookup[$field['name']] = $field;
        return $nameLookup;
    });

    // 步骤2：只保留那些在 CSV 表头中确实存在的自定义字段
    // （比较 name 的小写 vs 表头行的翻转键）
    if ($this->customFields) {
        $this->customFields = array_intersect_key(
            array_change_key_case($this->customFields),       // 键转小写
            array_change_key_case(array_flip($headerRow))      // 表头数组翻转后键转小写
        );
    }
}
```

**sanitizeItemForStoring() 对自定义字段的影响**：
```php
// ItemImporter.php L135-L150
protected function sanitizeItemForStoring($model, $updating = false)
{
    $item = collect($this->item);
    // ↓ 只保留模型 $fillable 里有的字段
    $item = $item->only($model->getFillable());
    // ...
}
```
> 结论：自定义字段**不在 `$fillable` 中**，所以会被 `sanitizeItemForStoring()` **完全过滤掉**。因此它们在代码中走**独立路径**——通过 `$asset->{$db_column} = $val` 直接赋值，**绕过** sanitize 过滤。

---

### 1.3 注入 rules：自定义字段校验规则的动态合并

**Asset 模型重写了 `save()` 方法**，在调用 Eloquent 原生 save 前注入自定义字段规则：

```php
// Asset.php L285-L290
public function save(array $params = [])
{
    // ← 关键：每次 save() 前，把当前 model 的 fieldset 的验证规则合并到 $this->rules
    $this->rules += $this->customFieldValidationRules();

    return parent::save($params);
}
```

**`customFieldValidationRules()` 的完整逻辑**：
```php
// Asset.php L255-L278
public function customFieldValidationRules()
{
    $customFieldValidationRules = [];

    // 前提条件：Asset 必须关联了 model，model 关联了 fieldset
    if (($this->model) && ($this->model->fieldset)) {

        // 步骤1：处理 BOOLEAN 类型的布尔转换
        foreach ($this->model->fieldset->fields as $field) {
            if ($field->format == 'BOOLEAN' && ! $field->field_encrypted) {
                // 把 "true"/"false"/"1"/"0" 字符串转成真正的 boolean
                $this->{$field->db_column} = filter_var($this->{$field->db_column}, FILTER_VALIDATE_BOOLEAN);
            }
        }

        // 步骤2：从 fieldset 取完整的 validation_rules（含加密/非加密分支）
        $customFieldValidationRules += $this->model->fieldset->validation_rules();
    }

    return $customFieldValidationRules;
}
```

**`CustomFieldset::validation_rules()` 的规则构建**：
```php
// CustomFieldset.php L121-L217
public function validation_rules(): array
{
    $rules = [];
    foreach ($this->fields as $field) {
        $rule = [];

        // 规则1：required / nullable
        //   - 非加密字段：按 pivot.required 决定
        //   - 加密字段：仅 admin 用户才校验 required（因为非 admin 看不到明文）
        if (($field->field_encrypted != '1')
            || (($field->field_encrypted == '1') && (Gate::allows('admin')))
        ) {
            $rule[] = ($field->pivot->required == '1') ? 'required' : 'nullable';
        }

        // 规则2：唯一约束（附加 unique_undeleted 规则）
        if ($field->is_unique == '1') {
            $rule[] = 'unique_undeleted';
        }

        // 规则3：格式校验（numeric / email / date / regex 等）
        if ($field->attributes['format'] != '') {
            array_push($rule, $field->attributes['format']);
        }

        // 用 db_column 作为规则键（如 '_snipeit_cpu_42'）
        $rules[$field->db_column_name()] = $rule;

        // ---------- 加密字段的特殊规则替换 ----------
        // 加密后的值无法直接用 Laravel 原生规则校验（全是乱码）
        // → 替换成自定义的 *Encrypted 规则类，内部先解密再校验
        if ($field->field_encrypted) {
            switch ($field->format) {
                case 'NUMERIC':  $rules[...] = new NumericEncrypted;  break;
                case 'ALPHA':    $rules[...] = new AlphaEncrypted;    break;
                case 'EMAIL':    $rules[...] = new EmailEncrypted;    break;
                case 'DATE':     $rules[...] = new DateEncrypted;     break;
                case 'URL':      $rules[...] = new UrlEncrypted;      break;
                case 'IP':       $rules[...] = new IPEncrypted;       break;
                case 'IPV4':     $rules[...] = new IPv4Encrypted;     break;
                case 'IPV6':     $rules[...] = new IPv6Encrypted;     break;
                case 'MAC':      $rules[...] = new MacEncrypted;      break;
                case 'BOOLEAN':  $rules[...] = new BooleanEncrypted;  break;
                case 自定义regex: $rules[...] = new RegexEncrypted;    break;
            }
        }

        // 规则4：非 checkbox 字段加 not_array（防止数组注入）
        if ($field->element != 'checkbox') {
            $rules[$field->db_column_name()][] = 'not_array';
        }

        // 规则5：checkbox 字段加 checkboxes 规则，radio 加 radio_buttons
        if ($field->element == 'checkbox') {
            $rules[$field->db_column_name()][] = 'checkboxes';
        }
        if ($field->element == 'radio') {
            $rules[$field->db_column_name()][] = 'radio_buttons';
        }
    }
    return $rules;
}
```

---

### 1.4 自定义字段注入全景时序

```
用户打开导入页面
  │
  ▼ Livewire Importer mount()
getColumns('asset')
  ├─ 取硬编码 $assets_fields（约40个标准字段）
  └─ CustomField::orderBy('name')->get() 追加到数组
       键 = db_column_name()  如 "_snipeit_cpu_42"
       值 = 字段显示名          如 "CPU 核心数"
  │
  ▼ 用户上传 CSV 后
updatingTypeOfImport() 自动匹配
  ├─ 第1级：strcasecmp(表头, 系统字段显示名) → 匹配 CustomField.name
  └─ 第2级：别名表（不含自定义字段，跳过）
  │
  ▼ 用户点击"开始导入"
ItemImportRequest@import()
  ├─ 翻转 column-mappings → 构建 $fieldMappings（不含自定义字段）
  ├─ 实例化 AssetImporter，调用 importer->import()
  │     │
  │     ▼ Importer::import()
  │     ├─ populateCustomFields($headerRow)
  │     │    └─ array_intersect_key 只保留 CSV 表头中存在的自定义字段
  │     ├─ Model::unguard() ← 全局关闭 fillable 保护
  │     └─ foreach rows:
  │           │
  │           ▼ AssetImporter::handle($row)
  │           ├─ parent::handle() → fill 标准字段
  │           ├─ if ($this->customFields) foreach:
  │           │    ├─ array_smart_custom_field_fetch($row, $customField)
  │           │    │   └─ 用 CustomField.name 直接匹配 $row（不走 fieldMap）
  │           │    ├─ 加密字段 → Crypt::encrypt()
  │           │    └─ 放入 $this->item['custom_fields'][$db_column]
  │           └─ createAssetIfNotExists($row)
  │                 │
  │                 ▼
  │                 ├─ sanitizeItemForStoring() → 过滤掉 custom_fields（不在fillable）
  │                 ├─ $asset->update($item) / fill($item)  ← 标准字段
  │                 ├─ foreach custom_fields: $asset->{$db_column} = $val  ← 绕过 fillable 直写
  │                 ├─ $asset->save()  ← 触发 ValidatingTrait
  │                 │     │
  │                 │     ▼ Asset::save()
  │                 │     └─ $this->rules += customFieldValidationRules()
  │                 │           ├─ BOOLEAN 类型 filter_var 转换
  │                 │           └─ fieldset->validation_rules()
  │                 │                 ├─ required/nullable（加密字段有Gate判断）
  │                 │                 ├─ unique_undeleted
  │                 │                 ├─ format（加密字段替换为 Encrypted 规则类）
  │                 │                 ├─ not_array / checkboxes / radio_buttons
  │                 │                 └─ 键 = db_column_name()
  │                 └─ ValidatingTrait 执行 rules → 通过/失败
  │
  ▼ Model::reguard() 恢复 fillable 保护
```

---

## 二、Update 分支：asset_tag 已存在时的 Checkout / Checkin 判定

### 2.1 Update 分支的进入条件

```php
// AssetImporter.php L74-L105
public function createAssetIfNotExists(array $row)
{
    $asset_tag = $this->findCsvMatch($row, 'asset_tag');
    if (empty($asset_tag)) {
        $asset_tag = Asset::autoincrement_asset();  // 空值自动生成
    }

    // 优先通过 id 查找（如果 CSV 提供了 id 列）
    if ($this->findCsvMatch($row, 'id') != '') {
        $asset = Asset::with('assignedTo')->find($this->findCsvMatch($row, 'id'));
    } else {
        // 否则按 asset_tag 查找
        $asset = Asset::with('assignedTo')->where(['asset_tag' => (string) $asset_tag])->first();
    }

    if ($asset) {
        // ↓ 关键：asset 已存在，但用户未勾选"更新模式"→ 直接报错退出
        if (! $this->updating) {
            $exists_error = trans('general.import_asset_tag_exists', ...);
            $this->addErrorToBag($asset, 'asset_tag', $exists_error);
            return $exists_error;
        }
        $editingAsset = true;   // ← 进入 Update 分支
    } else {
        $editingAsset = false;  // ← 走 Create 分支
        $asset = new Asset;
    }
}
```

> **进入 Update 分支的必要条件**：① asset 存在（by id or asset_tag）+ ② `$this->updating = true`（由 `import-update` POST 参数控制）

---

### 2.2 checkout_target 的来源：哪些 CSV 列组合决定了"要分配给谁"

**核心调用链**：
```
ItemImporter::handle($row)
  └─ $this->item['checkout_target'] = $this->determineCheckout($row)
        ├─ determineCheckout() 判断资产类型是否需要 checkout
        └─ 根据 checkout_class 决定走 Location 还是 User 分支
```

**`determineCheckout()` 的判定树**：
```php
// ItemImporter.php L108-L120
protected function determineCheckout($row)
{
    // 步骤0：某些导入类型（位置/型号/供应商/厂商/分类）不需要分配，直接 return
    if (in_array(get_class($this), [
        LocationImporter::class, AssetModelImporter::class,
        SupplierImporter::class, ManufacturerImporter::class, CategoryImporter::class
    ])) {
        return;
    }

    // 步骤1：显式指定了 checkout_class = 'location' + checkout_location 非空
    // CSV 列：checkout_type（映射到系统字段 checkout_class） + Checkout Location
    if (strtolower($this->item['checkout_class']) === 'location'
        && $this->findCsvMatch($row, 'checkout_location') != null) {
        return Location::findOrFail(
            $this->createOrFetchLocation(
                $this->findCsvMatch($row, 'checkout_location')
            )
        );
    }

    // 步骤2：默认走 User 分支（只要提供了足够的用户信息列就会返回 User 模型）
    return $this->createOrFetchUser($row);
}
```

**`createOrFetchUser()` 返回非 false 的 CSV 列条件**：

```php
// Importer.php L322-L419
protected function createOrFetchUser($row, $type = 'user')
{
    // 先读取所有可能的用户相关列
    $user_array = [
        'full_name'    => findCsvMatch($row, 'full_name'),     // CSV: Full Name / 领用人姓名
        'first_name'   => findCsvMatch($row, 'first_name'),    // CSV: First Name
        'last_name'    => findCsvMatch($row, 'last_name'),     // CSV: Last Name
        'display_name' => findCsvMatch($row, 'display_name'),  // CSV: Display Name
        'email'        => findCsvMatch($row, 'email'),         // CSV: Email
        'username'     => findCsvMatch($row, 'username'),      // CSV: Username
    ];

    // 命中条件1：username 非空，数据库已有对应用户 → 返回该 User
    if (! empty($user_array['username'])
        && $user = User::where('username', $user_array['username'])->first()) {
        return $user;
    }

    // 中止条件：username、full_name、first_name 三者全部为空 → return false
    // （意味着 CSV 没有提供任何用户身份识别信息）
    if (empty($username) && empty($full_name) && empty($first_name)) {
        return false;   // ← 不会触发 checkout
    }

    // （以下是自动补全缺失字段的逻辑）
    // 自动补 email：空的话从 full_name 生成
    // 自动补 first_name / last_name：从 full_name 拆分
    // 自动补 username：full_name → 按用户名格式规则生成，或直接用 email
    // 再次查找：生成 username 后再查一次 DB，看是否已存在

    // 最终兜底中止：补全后 username 或 first_name 仍为空 → return false
    if (empty($username) || empty($first_name)) {
        return false;
    }

    // 创建新用户并返回
    return $user;  // ← 返回 User 模型（即使是新建的）
}
```

**产生非 false checkout_target 的 CSV 列组合（满足任一即可）**：

| 组合序号 | 需要的 CSV 列（任一非空即可触发查找/创建） | 结果 |
|---|---|---|
| A | `Username` 非空（且该用户存在） | 返回已有 User |
| B | `Username` 非空（用户不存在）+ `First Name` 或 `Full Name` 非空（用于补全） | 创建并返回新 User |
| C | `Full Name` 非空（能从中生成 username + first_name） | 创建并返回新 User |
| D | `First Name` 非空（+ 可空的 Last Name） | 创建并返回新 User |
| E | `Checkout Type` = "location" + `Checkout Location` 非空 | 查找/创建 Location 并返回 |

> 以上 A~D 只要有任意**一列**有有效值（且满足补全条件），就会产生 checkout_target。

---

### 2.3 Checkout / Checkin 的最终判定逻辑

```php
// AssetImporter.php L208-L230
if ($success) {
    // 资产 save/update 成功后，处理分配逻辑
    if (isset($target) && ($target !== false)) {
        //  ↓ 重新取数据库最新状态（update 可能影响了 assigned_to）
        $asset = $asset->fresh();
        $targetType = get_class($target);

        // 关键判定1：目标和当前分配是否完全相同（id + type 同时匹配）
        // 同目标 → 跳过，不产生任何 checkin/checkout 日志
        $alreadyCheckedOutToTarget = ($asset->assigned_to == $target->id)
                                     && ($asset->assigned_type === $targetType);

        if (! $alreadyCheckedOutToTarget) {

            // 关键判定2：当前已有分配人（assigned_to 非 null）
            // → 先"归还"（触发 Checkin 事件，不改变 DB 状态的 assigned_to）
            if (! is_null($asset->assigned_to)) {
                event(new CheckoutableCheckedIn(
                    $asset,
                    $asset->assigned,          // 原分配人（User/Location 模型）
                    auth()->user(),             // 操作人
                    'Checkin from CSV Importer',// note
                    $checkin_date               // 时间：last_checkin 列值 or 当前时间
                ));
            }

            // 关键判定3：执行新的 checkout（覆盖 assigned_to/assigned_type/location_id 等）
            $asset->checkOut(
                $target,                  // 新分配目标
                $this->created_by,        // 创建人
                $checkout_date,           // 时间：last_checkout 列值 or 当前时间
                null,                     // expected_checkin（此处不单独传，走模型自带）
                'Checkout from CSV Importer',
                $asset->name
            );
        }
    }
    return;
}
```

---

### 2.4 时间回溯：last_checkin / last_checkout 列对事件时间的影响

```php
// AssetImporter.php L143-L175

/** Checkin 事件的时间 */
$checkin_date = date('Y-m-d H:i:s');           // 默认：当前时间
if ($this->item['last_checkin'] != '') {
    $item['last_checkin'] = $this->parseOrNullDate('last_checkin', 'datetime');
    $checkin_date = $this->item['last_checkin']; // ← 有值则用 CSV 的 last_checkin 回写
}

/** Checkout 操作的时间 */
$checkout_date = date('Y-m-d H:i:s');           // 默认：当前时间
if ($this->item['last_checkout'] != '') {
    $item['last_checkout'] = $this->parseOrNullDate('last_checkout', 'datetime');
    $checkout_date = $this->item['last_checkout']; // ← 有值则用 CSV 的 last_checkout 回写
}
```

> 💡 这是为了**历史数据迁移**而设计的：可以把过去的 checkin/checkout 时间从旧系统通过 CSV 导进来，事件日志会显示那个历史时间而非当前时间。

---

### 2.5 Update 模式下的决策真值表

假设已存在的资产当前状态：`assigned_to = UserA.id, assigned_type = User::class`

| # | CSV 用户列 | CSV 分配类型 | checkout_target | alreadyCheckedOutToTarget | asset.assigned_to | 触发动作 |
|---|---|---|---|---|---|---|
| 1 | 全空 | — | false | — | 保持 UserA | 无动作 |
| 2 | Full Name = UserA | 默认（User） | UserA（同一人） | ✅ true | 保持 UserA | 无动作（跳过重复） |
| 3 | Full Name = UserB | 默认（User） | UserB（不同人） | ❌ false | UserA → 先触发 Checkin 事件 → 再 checkOut 到 UserB |
| 4 | 全空 + Checkout Type=Location + Checkout Location=Loc1 | Location | Loc1 | ❌ false（类型不同） | UserA → 先触发 Checkin 事件 → 再 checkOut 到 Loc1 |
| 5 | Full Name = UserB + Checkout Type=Location + Checkout Location=Loc1 | Location | Loc1（checkout_class 优先） | ❌ false | UserA → 先 Checkin → 再 checkOut 到 Loc1 |
| 6 | Username = UserA（原值） | 默认（User） | UserA（同 id+同 type） | ✅ true | 保持 UserA | 无动作（跳过重复） |

> **测试验证**：`update_mode_logs_asset_update_in_actionlog` 测试的 L479 断言：重复 checkout（相同目标）**不应**产生新的 checkout 日志。

---

### 2.6 注意事项与潜在陷阱

1. **checkout_class 优先级最高**：如果 CSV 同时提供了用户列（Full Name 等）和 `Checkout Type = "Location"` + `Checkout Location`，会**忽略用户信息**，直接分配给 Location。

2. **checkin 只是发事件，不调 checkIn()**：代码走的是 `event(new CheckoutableCheckedIn(...))` 而非 `$asset->checkIn()`。前者只触发事件写日志，后者还会清空 `assigned_to`/`assigned_type`/`location_id` 等字段。**所以当需要从 A 转到 B 时，只需要一次 checkOut(B) 就够了，不需要先显式 checkIn。**

3. **新建资产的 checkout**：Create 分支下 `$asset->assigned_to` 初始为 null，所以只执行 `checkOut($target)`，不触发 Checkin 事件。

4. **sanitizeItemForStoring 对 checkout 没有直接影响**：checkout_target 在 sanitize 之前就保存到局部变量 `$target` 了（AssetImporter.php L130-L132），即使后续 sanitize 会移除 checkout_target 键，也不影响判定。

---

## 三、关键代码文件索引

| 文件 | 职责 |
|---|---|
| [CustomField.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/CustomField.php) | db_column 生成、字段格式定义、boot() 自动修改 assets 表结构 |
| [CustomFieldset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/CustomFieldset.php) | `validation_rules()` 构建完整校验规则（含加密字段分支）、fields() 多对多关联 |
| [Importer.php (Livewire)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Livewire/Importer.php) | `getColumns()` 追加自定义字段到 UI 选项；自动匹配算法 |
| [Importer.php (抽象基类)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/Importer.php) | `populateCustomFields()` 过滤、`array_smart_custom_field_fetch()` 取数据、`createOrFetchUser()` 判定用户目标 |
| [ItemImporter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/ItemImporter.php) | `determineCheckout()` 分配目标判定树、sanitize 过滤逻辑 |
| [AssetImporter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/AssetImporter.php) | 自定义字段加密/赋值、Update vs Create 分支、Checkin/Checkout 判定 |
| [Asset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/Asset.php) | `save()` 重写合并 rules、`customFieldValidationRules()` 注入规则 |
| [ImportAssetsTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/tests/Feature/Importing/Api/ImportAssetsTest.php) | `update_asset_from_import`、`update_mode_logs_asset_update_in_actionlog` 等测试用例 |
