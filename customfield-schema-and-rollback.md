# 资产导入：CustomField Schema 同步 & 批量导入事务回滚策略

---

## 一、CustomField 与 assets 表 Schema 的同步路径

### 总览：三种操作，三条模型事件

```
CustomField 模型 CRUD
  ├─ 创建新 CustomField  → created 事件  → Schema::table ADD COLUMN text nullable
  ├─ 修改 CustomField     → updating 事件 → isDirty('name') 时 Schema::table RENAME COLUMN
  └─ 删除 CustomField     → deleting 事件 → Schema::table DROP COLUMN
```

> ⚠️ **核心发现**：代码中**没有 "改列类型" 的逻辑**！所有自定义字段在 DB 层一律是 `text` 类型。`format` / `element` / `field_encrypted` 等属性的变化**不会触发任何 ALTER TABLE 修改列类型的操作**。类型校验完全在应用层通过 validation rules 实现。

---

### 1.1 新增列：created 事件

**触发代码**：[CustomField.php L161-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/CustomField.php#L161-L181)

```php
self::created(
    function ($custom_field) {

        // 防御性检查：列已存在则跳过（理论上不该发生）
        if (Schema::hasColumn(self::$table_name, $custom_field->db_column)) {
            return false;
        }

        // 步骤1：给 assets 表 ADD COLUMN
        // 列名由 convertUnicodeDbSlug() 生成：_snipeit_{slug}_{id}
        // 列类型固定为 text，固定为 nullable
        Schema::table(
            self::$table_name, function ($table) use ($custom_field) {
                $table->text($custom_field->convertUnicodeDbSlug())->nullable();
            }
        );

        // 步骤2：更新 custom_fields 表自己的 db_column 字段
        $custom_field->db_column = $custom_field->convertUnicodeDbSlug();
        $custom_field->save();   // ← 注意：这里在 created 事件里再次 save()，会触发 updating 事件
    }
);
```

**列名生成规则 `convertUnicodeDbSlug()`**：
```php
// CustomField.php L460-L472
public function convertUnicodeDbSlug($original = null)
{
    $name = $original ? $original : $this->name;
    $id = $this->id ? $this->id : 'xx';  // ← 创建时 id 还未生成，用 'xx' 临时占位？

    // 转 slug："CPU 核心数" → "_snipeit_cpu_he_xin_shu"
    $long_slug = '_snipeit_'.Utf8Slugger::slugify($name, '_');

    // 截断到 50 字符 + '_' + id
    return substr($long_slug, 0, 50).'_'.$id;
}
```

> 🚨 **嵌套 save 问题**：created 事件内部又调用了 `$custom_field->save()`，这会**触发 updating 事件**。但 updating 事件里判断 `isDirty('name')`，此时 name 没变（刚创建的），所以 updating 里的 renameColumn 逻辑不会执行。

---

### 1.2 重命名列：updating 事件（仅当 name 改变时）

**触发代码**：[CustomField.php L183-L208](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/CustomField.php#L183-L208)

```php
self::updating(
    function ($custom_field) {

        // 只在 name 字段改变时才执行
        if ($custom_field->isDirty('name')) {

            // 防御性检查：新列名已存在则直接 return（不改名了）
            if (Schema::hasColumn(self::$table_name, $custom_field->convertUnicodeDbSlug())) {
                return true;
            }

            // 步骤1：RENAME COLUMN 旧名 → 新名
            // 旧名用 getOriginal('name') 计算
            // 新名用当前 name 计算
            Schema::table(
                self::$table_name, function ($table) use ($custom_field) {
                    $table->renameColumn(
                        $custom_field->convertUnicodeDbSlug($custom_field->getOriginal('name')),
                        $custom_field->convertUnicodeDbSlug()
                    );
                }
            );

            // 步骤2：更新 custom_fields 表的 db_column 字段
            $custom_field->db_column = $custom_field->convertUnicodeDbSlug();
            $custom_field->save();   // ← 同样会再次触发 updating，需警惕死循环

            return true;
        }

        return true;  // 非 name 改动（如 format / element / required）直接 return
    }
);
```

**⚠️ 重要：非 name 属性改动不会改 Schema**

如果用户只是修改了以下属性，**不会触发任何 ALTER TABLE**：
- `format`：numeric / email / date / regex 等（只影响 validation rules）
- `element`：text / listbox / textarea / checkbox / radio（只影响前端表单渲染）
- `field_encrypted`：0/1（只影响存/取时的加密/解密逻辑）
- `required`（pivot）：是否必填（只影响 validation rules）
- `is_unique`：是否唯一（只影响 validation rules）
- `help_text` / `field_values` / `display_checkout` 等

> 💡 **设计意图**：所有自定义字段在 DB 层统一用 `text` 存储，应用层负责**输入校验**和**类型转换**（如 BOOLEAN 类型用 `filter_var` 转布尔）。这样无需每次修改格式都执行昂贵的 ALTER TABLE 操作。

---

### 1.3 删除列：deleting 事件

**触发代码**：[CustomField.php L211-L219](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/CustomField.php#L211-L219)

```php
self::deleting(
    function ($custom_field) {
        // DROP COLUMN，把数据也一起删了
        return Schema::table(
            self::$table_name, function ($table) use ($custom_field) {
                $table->dropColumn($custom_field->db_column);
            }
        );
    }
);
```

> ⚠️ **风险**：删除 CustomField 会**物理删除 assets 表该列的所有数据**，不可恢复。

---

### 1.4 Schema 同步的时序全景（以创建为例）

```
用户在后台点击 "新建自定义字段"
  │
  ▼ CustomFieldRequest 校验
CustomField::create($request->validated())
  │
  ▼ Eloquent 插入 custom_fields 表记录（此时 id 已生成，但 db_column 还不是最终值）
  │
  ▼ created 事件触发
  │
  ├─ [1] Schema::table('assets', function ($table) {
  │       $table->text('_snipeit_cpu_cores_42')->nullable();
  │    });
  │       ↳ 执行 SQL：ALTER TABLE assets ADD COLUMN _snipeit_cpu_cores_42 text NULL
  │
  └─ [2] $custom_field->db_column = '_snipeit_cpu_cores_42';
       $custom_field->save();
       │
       └─ 触发 updating 事件
          ├─ isDirty('name') = false（db_column 改动不算 name 改动）
          └─ return true → 不执行 renameColumn
```

---

## 二、批量导入的事务与回滚策略

### 2.1 外层事务结构

**触发代码**：[Importer.php L162-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/Importer.php#L162-L189)

```php
public function import()
{
    $headerRow = $this->csv->fetchOne();
    $this->csv->setHeaderOffset(0);

    $this->populateCustomFields($headerRow);

    // ↓ 整个导入过程在一个 DB 事务中
    DB::transaction(function () use ($headerRow) {
        $importedItemsCount = 0;
        Model::unguard();  // 全局关闭 fillable 保护

        foreach ($this->csv->getRecords($headerRow) as $row) {
            $row = array_change_key_case($row, CASE_LOWER);

            $this->handle($row);  // ← 处理单行（可能包含多次 DB 写入）

            $importedItemsCount++;
            // ... progress callback ...
        }

        Model::reguard();  // 恢复 fillable 保护
    });
}
```

> 💡 Laravel 的 `DB::transaction()` 机制：
> - 自动开启事务
> - 回调**正常返回** → `COMMIT` 提交
> - 回调**抛出任何异常** → `ROLLBACK` 回滚

---

### 2.2 单行 save() 失败的处理链路

让我们从 `AssetImporter::createAssetIfNotExists()` 开始追溯：

```php
// AssetImporter.php L197-L232
if ($editingAsset && ! empty($customFieldsToSave)) {
    $asset->update($customFieldsToSave);
    $success = true;
} elseif (! $editingAsset) {
    $success = $asset->save();   // ← 新建时的 save() 校验
} else {
    $success = true;
}

if ($success) {
    // ... 处理 checkout ...
    return;
}

// ↓ save() 返回 false 时走到这里
$this->logError($asset, 'Asset "'.$this->item['name'].'"');
```

**`logError()` 的实现**：
```php
// Importer.php L293-L298
protected function logError($item, $field)
{
    if ($this->errorCallback) {
        // 调用 ItemImportRequest 注入的 errorCallback
        call_user_func($this->errorCallback, $item, $field, $item->getErrors());
    }
}
```

**`errorCallback()` 的实现**（在 ItemImportRequest 中）：
```php
// ItemImportRequest.php L83-L86
public function errorCallback($item, $field, $errorString)
{
    // ← 只是把错误信息放入 $this->errors 数组，没有抛出任何异常！
    $this->errors[$item->name][$field] = $errorString;
}
```

---

### 2.3 关键判定：ValidatingTrait 的默认行为

项目使用的是 `watson/validating` 包的 `ValidatingTrait`。该 trait 的核心行为由 `$throwValidationExceptions` 属性控制：

```
$throwValidationExceptions = true  → 校验失败时抛出 ValidationException
$throwValidationExceptions = false → 校验失败时返回 false，不抛出（默认值）
```

**项目中各模型的设置**：

| 模型 | $throwValidationExceptions | 行为 |
|---|---|---|
| `Asset` | **未设置**（默认 false） | save() 失败返回 false |
| `CustomField` | **未设置**（默认 false） | save() 失败返回 false |
| `SCIMUser` | `true` | save() 失败抛出异常 |

> 验证：`SCIMUser.php` 第 9 行明确设置了 `protected $throwValidationExceptions = true;`，而 `Asset.php` 中没有这个设置。

---

### 2.4 事务回滚判定真值表

| 场景 | 是否抛出异常 | 事务行为 | 已成功行 | 失败行 |
|---|---|---|---|---|
| **单行 save() 校验失败**（如 model_id 必填） | ❌ 否（仅 logError 收集错误） | ✅ COMMIT（正常提交） | ✅ 已持久化 | ❌ 未写入，错误记录在 response |
| **关联模型 save 失败**（如 createOrFetchUser 创建用户失败） | ❌ 否（logError 后 return false） | ✅ COMMIT | ✅ 已持久化 | ❌ 用户未创建，但资产可能写入 |
| **SQL 异常**（如 DB 连不上、死锁） | ✅ 是（PDOException 向上冒泡） | ❌ ROLLBACK（整个事务回滚） | ❌ 全部回滚 | ❌ 全部回滚 |
| **PHP 致命错误**（内存溢出、调用未定义方法） | ✅ 是（Error 向上冒泡） | ❌ ROLLBACK | ❌ 全部回滚 | ❌ 全部回滚 |
| **主动 abort() / 抛出异常**（代码主动 throw） | ✅ 是 | ❌ ROLLBACK | ❌ 全部回滚 | ❌ 全部回滚 |

**最终结论**：**校验类错误（validation failures）不会触发回滚，只有未捕获的异常/错误才会**。这是一个**"尽力而为"**的策略——能导入多少算多少，失败的单独返回错误信息给用户。

---

### 2.5 三种错误上报方式的调用场景

| 方法 | 调用场景 | 错误来源 | 格式 |
|---|---|---|---|
| `logError($item, $field)` | 模型 `save()` 返回 false 时 | `$item->getErrors()`（Validator 的 MessageBag） | `[$item->name][$field] = $item->getErrors()` |
| `addErrorToBag($item, $field, $msg)` | 业务逻辑主动报错（如 asset_tag 已存在但未勾选更新） | 自定义字符串 | `[$item->name][$field] = [$field => [$msg]]` |
| `errorCallback()` 直接调用 | 预校验失败（如 field_map 有 null） | 自定义字符串 | `[$import->name][$field] = [$field => [$msg]]` |

**错误返回给前端的格式**：
```php
// ItemImportRequest@import() 最后 return $this->errors;
return [
    "My Laptop" => [
        "Asset \"My Laptop\"" => [
            "model_id" => ["The model id field is required."],
            "purchase_cost" => ["The purchase cost must be a number."]
        ]
    ],
    "Asset002" => [
        "Asset \"Asset002\"" => [
            "asset_tag" => ["An asset with the asset tag A002 already exists..."]
        ]
    ]
];
```

前端收到 HTTP 500 响应，body 就是这个 errors 数组。

---

### 2.6 潜在风险点

1. **部分成功的数据一致性问题**：
   比如第 3 行创建了一个新的 Category（`createOrFetchCategory`），但第 5 行资产 save 失败——Category 已经被创建了，且不会回滚。后续再导入时会复用这个 Category（不会重复创建），所以这不算大问题，但确实是"脏数据"。

2. **嵌套事务问题**：
   如果 `handle($row)` 内部又开了 `DB::transaction()`（目前代码里没有），嵌套事务在 MySQL 下会被忽略，行为会很诡异。

3. **`Model::reguard()` 可能不被调用**：
   如果 `foreach` 循环中某行抛出了未捕获的异常，`DB::transaction` 会回滚，但 `Model::reguard()` 放在循环结束后，**不会被执行**。此时后续请求如果复用这个 PHP 进程（如 Octane），会处于全局 unguard 状态，有安全风险。不过标准 PHP-FPM 模式下进程会销毁，影响不大。

---

### 2.7 导入事务时序图

```
import() 被调用
  │
  ├─ $headerRow = 读 CSV 表头
  ├─ populateCustomFields($headerRow)
  │
  ▼ DB::transaction(function () { ... })
  │  ├─ DB::beginTransaction();
  │  ├─ Model::unguard();
  │  │
  │  ├─ foreach rows:
  │  │   │
  │  │   ├─ $row = array_change_key_case($row, CASE_LOWER);
  │  │   ├─ handle($row)
  │  │   │   ├─ 解析字段
  │  │   │   ├─ createOrFetchCategory() → INSERT/SELECT
  │  │   │   ├─ createOrFetchManufacturer() → INSERT/SELECT
  │  │   │   ├─ ... 其他 lookup ...
  │  │   │   ├─ $asset->fill($item) / $asset->update($item)
  │  │   │   ├─ 自定义字段直写 $asset->{$db_column} = $val
  │  │   │   ├─ $success = $asset->save();
  │  │   │   │   ├─ ValidatingTrait 校验 rules
  │  │   │   │   └─ 失败 → return false ✅ 不抛异常
  │  │   │   └─ $success == false → logError($asset, $field)
  │  │   │                          → call_user_func($errorCallback, ...)
  │  │   │                          → $this->errors[...] = ...
  │  │   │                          ✅ 继续下一行
  │  │   │
  │  │   └─ $importedItemsCount++
  │  │
  │  └─ Model::reguard();  ← 正常结束才会走到这里
  │
  ├─ 回调正常返回 → DB::commit();
  │
  └─ 有未捕获异常 → DB::rollBack();

return $this->errors;  // 无论成功失败都返回 errors 数组（空或有值）
```

---

## 三、关键代码文件索引

| 文件 | 职责 |
|---|---|
| [CustomField.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/CustomField.php) | `boot()` 中注册 created/updating/deleting 事件，同步 Schema；`convertUnicodeDbSlug()` 生成列名 |
| [Importer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/Importer.php) | `import()` 开启 DB::transaction；`logError()` / `addErrorToBag()` 错误上报 |
| [ItemImportRequest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Http/Requests/ItemImportRequest.php) | `errorCallback()` 错误收集；`@import()` 注入回调并启动导入 |
| [AssetImporter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Importer/AssetImporter.php) | `createAssetIfNotExists()` 中 save() 失败后调用 `logError()` |
| [Asset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/Asset.php) | 未设置 `$throwValidationExceptions`，save() 失败返回 false 不抛异常 |
| [SCIMUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/SCIMUser.php) | 反例：设置了 `$throwValidationExceptions = true`，失败会抛异常 |
| [CustomFieldset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Models/CustomFieldset.php) | `validation_rules()` 按 format 生成应用层校验规则（替代 DB 层类型约束） |
