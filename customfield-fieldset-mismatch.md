# 资产导入：自定义字段与 Fieldset 错位路径分析

> 场景：CSV 文件里包含某自定义字段列，但该字段**不属于**当前资产型号（AssetModel）所属的字段集（Fieldset）。导入时数据会怎样？校验走还是不走？能落库吗？落库后能读出来吗？

---

## 一、错位路径全景：两侧范围对比

### 1.1 导入侧（写入端）——全集 CustomField

**`populateCustomFields()` 的范围**：[Importer.php L203-L221](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/Importer.php#L203-L221)

```php
protected function populateCustomFields($headerRow)
{
    // 步骤1：拉取 CustomField 全集！不是当前 model 的 fieldset 字段
    $this->customFields = CustomField::All()->reduce(function ($nameLookup, $field) {
        $nameLookup[$field['name']] = $field;
        return $nameLookup;
    });

    // 步骤2：只保留 CSV 表头中存在的（用 name 匹配）
    if ($this->customFields) {
        $this->customFields = array_intersect_key(
            array_change_key_case($this->customFields),
            array_change_key_case(array_flip($headerRow))
        );
    }
}
```

**关键点**：
- 拉取的是 `CustomField::All()` —— **整个系统定义的所有自定义字段**
- 过滤条件只有一个：**CSV 表头里有没有这个列名**（用 `CustomField.name` 做大小写不敏感匹配）
- **完全不关心**该字段属于哪个 fieldset、是否和当前资产型号的 fieldset 匹配

---

### 1.2 校验侧——仅当前 AssetModel 的 Fieldset 字段

**`customFieldValidationRules()` 的范围**：[Asset.php L255-L277](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/Asset.php#L255-L277)

```php
public function customFieldValidationRules()
{
    $customFieldValidationRules = [];

    // 前置条件：必须有 model，且 model 有关联的 fieldset
    if (($this->model) && ($this->model->fieldset)) {

        // 遍历 model->fieldset->fields —— 只取当前 fieldset 下的字段
        foreach ($this->model->fieldset->fields as $field) {
            // BOOLEAN 类型转换
            if ($field->format == 'BOOLEAN' && ! $field->field_encrypted) {
                $this->{$field->db_column} = filter_var($this->{$field->db_column}, FILTER_VALIDATE_BOOLEAN);
            }
        }

        // 合并当前 fieldset 的校验规则
        $customFieldValidationRules += $this->model->fieldset->validation_rules();
    }

    return $customFieldValidationRules;
}
```

**关键点**：
- 只取 `$this->model->fieldset->fields` —— **当前资产型号所属 fieldset 的字段**
- 如果 model 没有 fieldset，或者 fieldset 为空，**自定义字段校验规则就是空数组**
- `save()` 时合并到 `$this->rules` 里，由 ValidatingTrait 执行

---

### 1.3 两侧范围对比图

```
┌─────────────────────────────────────────────────────────────┐
│  CustomField::All()  ——  全集（N 个字段）                    │
│  ┌──────────────────────────────────────────────────┐      │
│  │ CSV 表头中存在的自定义字段（M 个，M ≤ N）           │      │
│  │  ┌────────────────────────────────────┐          │      │
│  │  │ 当前 model 的 fieldset 字段（K 个） │          │      │
│  │  │                                    │          │      │
│  │  │   ▲ 这里是"正确匹配"区              │          │      │
│  │  │   │ 有校验 + 能写入 + 能正常展示    │          │      │
│  │  └───┴────────────────────────────────┘          │      │
│  │                                                   │      │
│  │   ○ 错位区（M - K 个）                             │      │
│  │     - 能写入 DB                                   │      │
│  │     - 跳过校验（校验规则里没有这个字段）             │      │
│  │     - 展示端不可见（不在 fieldset 里）              │      │
│  └──────────────────────────────────────────────────┘      │
│                                                             │
│  （剩下的 N-M 个字段：CSV 里没有，连碰都碰不到）              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、错位场景的逐步骤调用链

假设场景：
- 系统有自定义字段 `CPU 核心数`（db_column = `_snipeit_cpu_42`），属于 fieldset "服务器字段集"
- 当前资产属于型号 "笔记本电脑"，其 fieldset 是 "笔记本字段集"，**不含** "CPU 核心数"
- CSV 文件里包含列 `CPU 核心数 = "not a number"`（故意填非数字，如果字段格式是 numeric）

---

### 步骤 1：populateCustomFields —— 纳入处理范围

```
CSV 表头: [..., "CPU 核心数", ...]
CustomField::All() → 包含 "CPU 核心数" 字段
array_intersect_key → 命中，纳入 $this->customFields
```

结果：`$this->customFields['cpu 核心数'] = CustomField 对象（含 db_column: "_snipeit_cpu_42"）`

**代码位置**：[Importer.php L203-L221](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/Importer.php#L203-L221)

---

### 步骤 2：handle() 中读取 CSV 值 —— 放入 item

```php
// AssetImporter.php L43-L60
if ($this->customFields) {
    foreach ($this->customFields as $customField) {
        // 直接用 name 从 CSV 行取
        $customFieldValue = $this->array_smart_custom_field_fetch($row, $customField);

        if (! is_null($customFieldValue)) {
            if ($customField->field_encrypted == 1) {
                $this->item['custom_fields'][$customField->db_column_name()] = Crypt::encrypt($customFieldValue);
            } else {
                $this->item['custom_fields'][$customField->db_column_name()] = $customFieldValue;
            }
        } else {
            // CSV 里这列有表头但此行值为空 → 设 null（清空旧值）
            $this->item['custom_fields'][$customField->db_column_name()] = null;
        }
    }
}
```

结果：`$this->item['custom_fields']['_snipeit_cpu_42'] = "not a number"`

**代码位置**：[AssetImporter.php L43-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/AssetImporter.php#L43-L60)

> ⚠️ **注意**：这里完全没有检查"这个字段是否属于当前 asset 的 model 的 fieldset"。只要 CSV 里有列名匹配到 CustomField.name，就会被读取并放入 item。

---

### 步骤 3：createAssetIfNotExists —— 直接属性赋值写入

```php
// AssetImporter.php L185-L206
$customFieldsToSave = [];
if (array_key_exists('custom_fields', $this->item)) {
    foreach ($this->item['custom_fields'] as $custom_field => $val) {
        // 直接属性赋值 —— 在 Model::unguard() 环境下，不在 fillable 里也能写
        $asset->{$custom_field} = $val;
        $customFieldsToSave[$custom_field] = $val;
    }
}

// 更新模式 + 有自定义字段 → 二次 update
if ($editingAsset && ! empty($customFieldsToSave)) {
    $asset->update($customFieldsToSave);  // ← unguard 下可直接写
    $success = true;
} elseif (! $editingAsset) {
    // 新建模式 → 一次 save()，上面赋值的属性都在
    $success = $asset->save();
}
```

结果：`$asset->_snipeit_cpu_42 = "not a number"`，**已经写入模型属性**，就等 save 了。

**代码位置**：[AssetImporter.php L185-L206](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/AssetImporter.php#L185-L206)

---

### 步骤 4：save() 触发校验 —— 校验规则里没有这个字段！

```php
// Asset.php L285-L290
public function save(array $params = [])
{
    // 合并自定义字段校验规则
    $this->rules += $this->customFieldValidationRules();
    // ↑ 这里只合并当前 model->fieldset 的字段规则
    //   "CPU 核心数" 不在笔记本字段集里 → rules 里没有 _snipeit_cpu_42 的任何规则

    return parent::save($params);  // ValidatingTrait 执行校验
}
```

**关键判定**：
- `$this->model->fieldset` 是 "笔记本字段集"，不包含 "CPU 核心数"
- `customFieldValidationRules()` 返回的数组里**没有** `_snipeit_cpu_42` 这条规则
- ValidatingTrait 校验时，**不会检查不在 rules 里的属性**（Laravel 校验器默认行为）
- 所以 `_snipeit_cpu_42 = "not a number"` 即使格式是 numeric，**也不会被校验到**

**结果**：`save()` 返回 `true`，数据成功写入 DB。

**代码位置**：[Asset.php L285-L289](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/Asset.php#L285-L289)

---

### 步骤 5：落库后 —— "幽灵数据"

数据已经写到 `assets._snipeit_cpu_42` 列里了，值是 `"not a number"`。

**但在前端表现如何？**

| 场景 | 能否看到/使用 | 原因 |
|---|---|---|
| 资产详情页的自定义字段区 | ❌ 看不到 | 详情页按 model->fieldset 渲染字段，不在 fieldset 里的不渲染 |
| 资产列表页的列选择器 | ⚠️ 可能看到 | `AssetPresenter` 的自定义字段列是按 "有 fieldset 关联且 fieldset 有 model" 全局取的，不是按当前资产 |
| 列表页该列的单元格值 | ✅ 能显示 | 列存在就会从 attributes 里读，数据在 DB 里就能读出来 |
| 编辑页的表单 | ❌ 看不到字段输入框 | 编辑页按 fieldset 渲染表单 |
| 编辑提交后的值 | ❌ 不会被修改 | 编辑页没这个字段 → 不会提交 → 保留原值 |
| API 返回 | ⚠️ 视 transformer 而定 | 若 transformer 按 fieldset 过滤则没有，若直接 all() 则有 |

> 💡 本质是**"数据在库里，但 UI 上看不到"**的幽灵字段。只有直接查数据库、或者该字段属于某个 fieldset 且你从那个型号的资产列表里看，才能发现。

---

## 三、三种错位场景的行为对比

| # | 错位类型 | 写入？ | 校验？ | 前端可见？ | 风险等级 |
|---|---|---|---|---|---|
| 1 | CSV 含字段 X，字段 X 属于其他型号的 fieldset | ✅ 写入 | ❌ 跳过校验 | ❌ 详情页不可见，列表页可能在列里 | 中：静默写入脏数据 |
| 2 | CSV 含字段 X，字段 X 不属于任何 fieldset | ✅ 写入 | ❌ 跳过校验 | ❌ 完全不可见（Presenter 过滤掉了） | 高：完全幽灵数据 |
| 3 | 字段 X 属于当前 fieldset，但 CSV 没有这列 | — | — | — | 低：正常行为，不写入 |

> 场景 2 特别要注意：一个游离的（不属于任何 fieldset）自定义字段，只要 CSV 里有列名精确匹配，数据就会静默写入数据库，而且**前端完全看不到**。

---

## 四、Update 模式下的特殊行为

更新模式（`import-update = true`）下，错位字段的行为：

```php
// AssetImporter.php L43-L60
if ($this->customFields) {
    foreach ($this->customFields as $customField) {
        $customFieldValue = $this->array_smart_custom_field_fetch($row, $customField);

        if (! is_null($customFieldValue)) {
            // CSV 里有值 → 赋值（加密 or 原文）
            $this->item['custom_fields'][$customField->db_column_name()] = ...;
        } else {
            // CSV 里这列有表头但值为空 → 设 null 清空
            $this->item['custom_fields'][$customField->db_column_name()] = null;
        }
    }
}
```

**关键点**：
- 即使字段不在当前 fieldset 里，只要 CSV 有这一列的表头（populateCustomFields 阶段已经纳入），**更新时就会覆盖**
- 如果 CSV 里这行的值为空字符串，**会把该列设为 null**（清空原有值）
- 整个过程**不经过校验**

> ⚠️ 潜在风险：用更新模式批量导入一批数据，CSV 里不小心包含了某个不属于当前型号的自定义字段列（比如从别的型号导出的模板），会**静默清空或覆盖**这些字段的值，而且前端看不到变化。

---

## 五、为什么会这样？设计意图 vs 实际后果

### 设计意图
1. **导入通用性**：`Importer` 基类不应该知道"当前导入的是哪种型号"，它只是按 CSV 表头和 CustomField 全集做匹配——这是为了让导入逻辑更通用，不依赖具体业务。
2. **DB 列存在就能写**：既然 `assets` 表上确实有这一列（`_snipeit_*`），写进去也没什么坏处——最多是"多余的数据"。
3. **校验按业务场景走**：校验规则和展示层按 fieldset 走，确保**用户能看到的字段才校验**——看不到的字段校验了也没意义。

### 实际后果（潜在问题）
1. **校验绕过**：格式是 numeric 的字段可以写入任意字符串，只要 CSV 列名匹配且字段不在当前 fieldset 里。
2. **数据污染**：误导入的数据可能悄悄躺在 DB 里，没人发现。
3. **不一致性**：同样的字段，在这个型号的资产上是"导入了但不校验"，在另一个型号的资产上是"导入了也校验"，行为不一致。
4. **更新时的意外清空**：更新模式 + 错位列 + 空值 → 静默清空数据。

---

## 六、关键代码文件索引

| 文件 | 职责 | 相关行 |
|---|---|---|
| [Importer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/Importer.php) | `populateCustomFields()` 全集拉取 + CSV 表头过滤 | L203-L221 |
| [AssetImporter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/AssetImporter.php) | `handle()` 读 CSV → 放入 custom_fields；`createAssetIfNotExists()` 直写属性 | L43-L60, L185-L206 |
| [Asset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/Asset.php) | `customFieldValidationRules()` 只取当前 model->fieldset 的规则；`save()` 合并规则 | L255-L277, L285-L290 |
| [CustomFieldset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/CustomFieldset.php) | `validation_rules()` 构建 fieldset 级别的校验规则 | L121-L217 |
| [AssetModel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/AssetModel.php) | `fieldset()` 关联，`customFields()` 快捷访问 | L221-L228 |
| [AssetPresenter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Presenters/AssetPresenter.php) | 列表页自定义字段列定义（按"有 fieldset 且 fieldset 有 model"过滤） | L328-L354 |
