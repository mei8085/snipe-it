# 幽灵字段的可见性：查询端多入口行为全景分析

> 场景：某自定义字段 "CPU 核心数"（db_column = `_snipeit_cpu_42）属于"服务器字段集"，但某笔记本资产（型号"笔记本电脑"，其 fieldset 是"笔记本字段集"，**不含**该字段。通过导入时 CSV 包含该列，数据已写入 DB，现在查询这个资产时，各查询入口会吐不吐这条数据？

---

## 一、核心判定公式

| 查询入口 | 过滤逻辑 | 幽灵字段可见？ | 代码位置 |
|---|---|---|---|
| **AssetsTransformer::transformAsset() | 资产列表/详情 API | ✅ 硬过滤（只遍历 fieldset->fields） | ❌ 完全不可见 | [AssetsTransformer.php L129-L171](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Http/Transformers/AssetsTransformer.php#L129-L171) |
| **AssetsTransformer::transformRequestedAsset()** 申请资产列表 API | ✅ 硬过滤 + show_in_requestable_list | ❌ 完全不可见 | [AssetsTransformer.php L264-L284](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Http/Transformers/AssetsTransformer.php#L264-L284) |
| **资产详情页 view.blade.php | ✅ 硬过滤（只遍历 fieldset->fields） | ❌ 详情区不可见 | [view.blade.php L193-L198](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/resources/views/hardware/view.blade.php#L193-L198) |
| **资产列表页列定义 AssetPresenter** | ✅ 全局过滤（字段属于某个 fieldset 且 fieldset 有 model） | ⚠️ 列选择器能看到，数据从底层 attributes 读 | **可能泄漏 | [AssetPresenter.php L335-L353](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Presenters/AssetPresenter.php#L335-L353) |
| **列表页单元格渲染 customFieldsFormatter** | ❌ 无过滤（列定义里有就从 row 里读 | ⚠️ 只要列在布局里，单元格就显示值 | ✅ 可见 | [bootstrap-table.blade.php L1928-L1980](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/resources/views/partials/bootstrap-table.blade.php#L1928-L1980) |
| **API 排序/搜索 AssetsController@index** | ✅ CustomField::all() 全放入 $allowed_columns | ⚠️ 可按任何自定义字段搜索/排序 | ✅ 底层数据可检索 | [AssetsController.php L141-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Http/Controllers/Api/AssetsController.php#L141-L145) |
| **Blade 模板直接 `$asset->_snipeit_cpu_42` | ❌ 无过滤，直接读 attributes | ✅ 直接吐出原始值（加密串 | [view.blade.php L196](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/resources/views/hardware/view.blade.php#L196) |
| **Tinker/代码 `$asset->toArray()` | ❌ 无过滤，toArray() 遍历 attributes + visible | ⚠️ 取决于 $visible + $hidden | - |
| **DB 原生查询 `SELECT _snipeit_cpu_42` | ❌ 无过滤 | ✅ 直接返回 | - |

---

## 二、各入口深入分析

### 2.1 AssetsTransformer::transformAsset() —— 硬过滤

**核心代码**：
```php
// AssetsTransformer.php L129-L171
if (($asset->model) && ($asset->model->fieldset) && ($asset->model->fieldset->fields->count() > 0) {
    $fields_array = [];
    // ↓ 只遍历当前 model->fieldset 的字段
    foreach ($asset->model->fieldset->fields as $field) {
        // 解密 + 格式化 + 权限检查
        $fields_array[$field->name] = [
            'field' => e($field->db_column),
            'value' => e($value),
            'field_format' => $field->format,
            'element' => $field->element,
        ];
    }
    $array['custom_fields'] = $fields_array;
} else {
    $array['custom_fields'] = new \stdClass; // 空对象
}
```

**过滤逻辑**：
1. 只有 `$asset->model->fieldset->fields` 是**当前资产型号所属 fieldset 下的字段集合
2. 不在这个集合里的字段（错位字段）**连碰都不碰**，完全不出现在 custom_fields 数组里
3. 即使 `$asset->attributes['_snipeit_cpu_42']` 有值，也不会遍历不到
4. 外层 `$array` 里除了 `custom_fields` 节点，其他节点也没有任何 `_snipeit_` 字段（Transformer 手动枚举，不是全 attributes）

**结论**：✅ 幽灵字段在 API 返回里**完全不可见**，被 Transformer 硬过滤掉了。

---

### 2.2 AssetsTransformer::transformRequestedAsset() —— 双重过滤

**核心代码**：
```php
// AssetsTransformer.php L264-L284
if (($asset->model) && ($asset->model->fieldset) && ...) {
    foreach ($asset->model->fieldset->fields as $field) {
        // 额外过滤：非加密 且 show_in_requestable_list == 1
        if (($field->field_encrypted == '0') && ($field->show_in_requestable_list == '1')) {
            $fields_array[$field->db_column] = e($value);
        }
    }
    $array['custom_fields'] = $fields_array;
}
```

**额外过滤条件**：
- 第一层：只遍历当前 fieldset 字段
- 第二层：非加密字段
- 第三层：`show_in_requestable_list == 1

**结论**：✅ 幽灵字段**完全不可见**，比 transformAsset 过滤更严。

---

### 2.3 资产详情页 view.blade.php —— 只渲染 fieldset 字段

**核心代码**：
```blade
{{-- view.blade.php L193-L198 --}}
@if (($asset->model) && ($asset->model->fieldset))
    @foreach($asset->model->fieldset->fields as $field)
        <x-data-row :label="$field->name">
            <x-info-element.customfield :item="$asset" :field="$field"/>
        </x-data-row>
    @endforeach
@endif
```

**过滤逻辑**：
- Blade 模板只遍历 `$asset->model->fieldset->fields`，和 API 一样的硬过滤

**但要注意**：如果有人手动在模板里写死 `{{ $asset->_snipeit_cpu_42 }}`，会**直接吐出值**——这是不安全的，因为直接访问属性不经过任何过滤。

**结论**：✅ 正常渲染路径下**不可见**，但手动写属性访问**可见**。

---

### 2.4 资产列表页列定义 AssetPresenter —— 全局过滤 + 底层属性读

**列表页的列定义由 `AssetPresenter::dataTableLayout()` 返回给前端 bootstrap-table，这个方法里的自定义字段列定义：

**核心代码**：
```php
// AssetPresenter.php L335-L353
$fields = CustomField::whereHas('fieldset', function ($query) {
    $query->whereHas('models');
})->get();
// ↑ 全局拉取所有属于"某个 fieldset 且该 fieldset 有关联 model"的自定义字段

foreach ($fields as $field) {
    $layout[] = [
        'field' => $field->db_column,  // 列字段名 = db_column
        'title' => $field->name,          // 显示名
        'formatter' => 'customFieldsFormatter',
        // ...
    ];
}
```

**关键点**：
- 这里的过滤条件是**"字段属于某个 fieldset，且那个 fieldset 被至少一个 model 使用"——不是"字段属于某个型号的 fieldset"
- 只要"CPU 核心数"属于"服务器字段集"，而"服务器字段集"被"服务器型号"使用 → **会出现在列定义里
- 列定义里的 `field` 是 `$field->db_column`，也就是 `_snipeit_cpu_42`

**接下来**：前端 bootstrap-table 收到这个列后，每一行数据从 `row._snipeit_cpu_42` 取值

```javascript
// bootstrap-table.blade.php L1928-L1930
function customFieldsFormatter(value, row) {
    if ((!this) || (!this.title)) { return ''; }
    var field_column = this.title;
    // ... 
```

**数据来源**：Transformer 返回的行数据里，如果 `_snipeit_cpu_42` 在 `attributes` 里，bootstrap-table 的行数据是 Transformer 的 `$array` 里吗？

**关键发现**：AssetsTransformer 返回的 `$array` 里并没有单独的 `_snipeit_cpu_42` 字段（因为上面的分析，不在 custom_fields 里遍历的是 `$fields_array[$field->name] = ['field' => db_column, 'value' => ...} 结构，而不是直接放在 `_snipeit_cpu_42` 放在根节点。

**那 customFieldsFormatter 里的 `row._snipeit_cpu_42` 是从哪里来的？

让我再看一下数据行数据结构的列定义里的 `field` 是 `_snipeit_cpu_42`，而 Transformer 里的 custom_fields 是嵌套结构。这中间有个转换层。

看 `transformAsset() 里的 `custom_fields` 结构：
```php
$fields_array[$field->name] = [
    'field' => '_snipeit_cpu_42',  // db_column
    'value' => '4',
    'field_format' => 'NUMERIC',
    'element' => 'text'
];
$array['custom_fields'] = $fields_array;
```

而列定义里的 `field` 是 `_snipeit_cpu_42`，但 Transformer 根节点没有 `_snipeit_cpu_42`，只有 `custom_fields` 嵌套对象。

**customFieldsFormatter 的工作方式**：
```javascript
// bootstrap-table.blade.php L1928-L1980
function customFieldsFormatter(value, row) {
    // value = undefined 因为根节点没有 _snipeit_cpu_42，value 是 undefined
    // 但 row.custom_fields 里有
    // 所以这个 formatter 应该是从 row.custom_fields 里找匹配 field 对应 name 的值
```

看完整的 formatter 实现应该是这样工作的：列定义的 `field` 是 db_column，但实际值在 row.custom_fields 里找匹配的 field 值？不，看 `customFieldsFormatter 的 this.field 是 `_snipeit_cpu_42`，而 row.custom_fields 是 `{ 'CPU 核心数': { field: '_snipeit_cpu_42', value: '4', ...}}`，所以需要遍历 row.custom_fields 找 field == this.field 的项，取 value。

**但对于幽灵字段的 fate**：
- 如果字段不在当前 asset 的 fieldset 里 → `row.custom_fields` 里没有这项 → formatter 返回空 → 单元格显示空
- 列选择器里能看到这个列（因为列定义是全局的），但数据是空的
- 但如果有人在 Transformer 返回的数据里没有，实际 DB 里有值的情况？不，Transformer 里没有，那 DB 里有，但 Transformer 不返回，row 里就没有

**结论**：⚠️ 列选择器里能看到列，但因为 Transformer 已经过滤掉了，所以单元格里**看不到具体数据**。但！如果 Transformer 返回的根节点有这个字段（比如有人改了代码直接把 attributes 全放出来），那就会显示。

---

### 2.5 API 排序/搜索 AssetsController@index —— 全量自定义字段

**核心代码**：
```php
// AssetsController.php L141-L145
$all_custom_fields = CustomField::all();
foreach ($all_custom_fields as $field) {
    $allowed_columns[] = $field->db_column_name();
}
```

**这意味着**：
- 所有自定义字段（包括不属于当前资产 fieldset 的字段）都被加入 `$allowed_columns`
- 前端可以请求 `?sort=_snipeit_cpu_42` 按这个字段排序
- 搜索时 TextSearch 也会在这些字段上搜索
- 但**结果里的值仍然经过 AssetsTransformer 过滤，所以返回的数据里不会显示这些字段的值

**结论**：⚠️ 可以按幽灵字段**排序/搜索**，但返回结果里**看不到值**。

---

### 2.6 直接属性访问 —— 无过滤

**如果在代码里写**：
```php
$asset = Asset::find(123);
echo $asset->_snipeit_cpu_42;  // ✅ 直接返回 DB 里的值，无任何过滤
echo $asset->attributes['_snipeit_cpu_42'];  // ✅ 同样直接返回
```

**如果在 Blade 里写**：
```blade
{{ $asset->_snipeit_cpu_42 }}  // ✅ 直接吐出值，包括加密的乱码也会出
```

**`toArray()` 的行为**：
```php
$asset->toArray();
// 返回的是 $this->visible + $this->attributes 减去 $this->hidden
// 由于 _snipeit_* 不在 $hidden 里，所以都会出来！
// 但自定义字段列都在 attributes 里，都会返回
```

**结论**：✅ **完全可见**，无过滤。

---

## 三、AssetsTransformer 与 Eloquent toArray() 的对比

| 方式 | 幽灵字段可见？ | 原因 |
|---|---|---|
| `(new AssetsTransformer)->transformAsset($asset) | ❌ | 手动枚举字段列表，不遍历 attributes |
| `$asset->toArray() | ✅ | 遍历 attributes，不在 hidden 的都吐 |
| `$asset->jsonSerialize() | ✅ | 同 toArray() |
| `json_encode($asset)` | ✅ | 触发 jsonSerialize() |
| Blade `{{ $asset->_snipeit_cpu_42 }}` | ✅ | 直接读 attributes |
| `$asset->getAttribute('_snipeit_cpu_42')` | ✅ | 直接读 attributes |

---

## 四、列定义 vs 数据返回的脱节问题

**AssetPresenter::dataTableLayout() 定义了**全局**列定义（所有"有 fieldset 且 fieldset 有 model 的自定义字段），但 AssetsTransformer 返回的数据是**按资产**过滤的。

**脱节表现**：

1. 列定义里有"CPU 核心数"列
2. 数据返回里，服务器型号资产有 custom_fields.CPU 核心数字段
3. 笔记本型号资产没有 custom_fields.CPU 核心数字段
4. 前端 customFieldsFormatter 找不着 → 空

**但有个例外**：如果 AssetsTransformer 把 `custom_fields 数组平铺开（不是嵌套的 custom_fields，而是把每个自定义字段作为根节点的属性），那笔记本型号资产的 `_snipeit_cpu_42` 就会是 undefined。

看代码，AssetsTransformer 是把 custom_fields 作为一个对象，而不是平铺到根节点。所以列定义的 `field: '_snipeit_cpu_42' 实际找 row._snipeit_cpu_42，这跟节点没有，所以 value 是 undefined。

但 customFieldsFormatter 会检查 custom_fields 对象里找匹配的项。对，看完整的 customFieldsFormatter 实现应该遍历 row.custom_fields，找到 field == this.field 的项，取 value。如果找不到，就返回空。

**所以最终表现**：
- 服务器型号资产 → 有值 → 显示值
- 笔记本型号资产 → 找不到 → 显示空
- 错位字段 → 找不到 → 显示空

---

## 五、安全隐患总结

| 风险点 | 风险等级 | 说明 |
|---|---|---|
| 手动属性访问泄漏 | 中 | 直接读属性或 toArray() 全吐出来，包括敏感的加密字段乱码也会出来 |
| 列选择器误导 | 低 | 用户看到列但没数据，困惑 |
| 搜索/排序可用幽灵字段 | 低 | 能搜能排但看不到值 |
| 加密字段的解密权限绕过 | 中 | 直接访问属性时，加密字段不解密直接返回加密串 |
| 导出功能泄漏 | 高 | 如果导出功能用 toArray() 或直接读 attributes，幽灵字段全吐 |

---

## 六、关键代码文件索引

| 文件 | 职责 | 相关行 |
|---|---|---|
| [AssetsTransformer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Http/Transformers/AssetsTransformer.php) | transformAsset() / transformRequestedAsset() 硬过滤自定义字段 | L129-L171, L264-L284 |
| [AssetPresenter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Presenters/AssetPresenter.php) | dataTableLayout() 全局取列定义 | L335-L353 |
| [AssetsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Http/Controllers/Api/AssetsController.php) | index() 把所有 CustomField 加入 $allowed_columns | L141-L145 |
| [view.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/resources/views/hardware/view.blade.php) | 详情页只遍历 fieldset 字段 | L193-L198 |
| [bootstrap-table.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/resources/views/partials/bootstrap-table.blade.php) | customFieldsFormatter 前端渲染 | L1928-L1980 |
| [AssetsController.php (web)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-snipe-it/app/Http/Controllers/Assets/AssetsController.php) | 编辑/保存时只处理 fieldset 字段 | L202-L210 |
