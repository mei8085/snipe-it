# Snipe-IT 资产数据 API 与网页表格字段差异分析

## 1. 整体架构与数据流向

### 1.1 API 数据路径
```
HTTP 请求
    ↓
Api\AssetsController@index (app/Http/Controllers/Api/AssetsController.php:65)
    ↓ 权限校验: $this->authorize('index', Asset::class)
    ↓ 数据查询 (自动应用 CompanyableScope)
    ↓ 关联预加载: model, location, status, company, assignedTo, etc.
    ↓
AssetsTransformer@transformAssets (app/Http/Transformers/AssetsTransformer.php:20)
    ↓ 循环调用 transformAsset()
    ↓
AssetsTransformer@transformAsset (app/Http/Transformers/AssetsTransformer.php:30)
    ↓ 字段级权限检查
    ↓ 自定义字段解密/脱敏
    ↓ available_actions 权限判断
    ↓
DatatablesTransformer@transformDatatables (app/Http/Transformers/DatatablesTransformer.php:10)
    ↓
JSON 响应
```

### 1.2 网页表格数据路径
```
浏览器访问 /hardware
    ↓
Assets\AssetsController@index (app/Http/Controllers/Assets/AssetsController.php:66)
    ↓ 权限校验: $this->authorize('index', Asset::class)
    ↓
视图: hardware/index.blade.php
    ↓
<x-table.assets> 组件
    ↓ 传入 :route="route('api.assets.index')"
    ↓
Bootstrap Table 初始化 (resources/views/partials/bootstrap-table.blade.php:478)
    ↓ AJAX 请求 api.assets.index 路由
    ↓ （后续与 API 路径完全相同）
    ↓
接收 JSON 数据
    ↓
列配置过滤: AssetPresenter::dataTableLayout() (app/Presenters/AssetPresenter.php:20)
    ↓ 根据 visible 字段判断是否显示
    ↓ 根据 formatter 调用对应的 JS 函数
    ↓
JavaScript Formatter 二次加工
    ↓
表格渲染完成
```

---

## 2. 三层核心差异点

### 2.1 Transformer 层 - API 输出规范

**文件位置**: `app/Http/Transformers/AssetsTransformer.php`

**核心方法**: `transformAsset(Asset $asset)` (第30行)

**输出特性**:

1. **完整字段输出** (第35-127行):
   - 基础字段: `id`, `name`, `asset_tag`, `serial`, `model`, `model_number`, `byod`, `requestable`
   - 关联对象: `status_label`(兼容), `status`, `category`, `manufacturer`, `depreciation`, `supplier`, `company`, `location`, `rtd_location`, `assigned_to`
   - 日期字段: 统一格式化为 `{ datetime: "Y-m-d H:i:s", formatted: "Y/m/d" }` 结构，包括 `created_at`, `updated_at`, `purchase_date`, `last_audit_date`, `next_audit_date`, `last_checkout`, `last_checkin`, `expected_checkin`
   - 数值字段: `purchase_cost`, `book_value`, `checkin_counter`, `checkout_counter`, `requests_counter`
   - 其他: `image`, `qr`, `alt_barcode`, `warranty_months`, `warranty_expires`, `notes`, `order_number`, `age`, `eol`, `asset_eol_date`

2. **自定义字段处理** (第129-171行):
   ```php
   // 加密字段权限检查 (第135行)
   $value = (Gate::allows('assets.view.encrypted_custom_fields')) 
       ? $decrypted 
       : strtoupper(trans('admin/custom_fields/general.encrypted'));
   
   // 输出结构
   $fields_array[$field->name] = [
       'field' => e($field->db_column),
       'value' => e($value),
       'field_format' => $field->format,
       'element' => $field->element,
   ];
   ```

3. **可用动作权限** (第173-181行):
   ```php
   $permissions_array['available_actions'] = [
       'checkout' => ($asset->deleted_at == '' && Gate::allows('checkout', Asset::class)),
       'checkin' => ($asset->deleted_at == '' && Gate::allows('checkin', Asset::class)),
       'clone' => Gate::allows('create', Asset::class),
       'restore' => ($asset->deleted_at != '' && Gate::allows('create', Asset::class)),
       'update' => ($asset->deleted_at == '' && Gate::allows('update', Asset::class)),
       'audit' => Gate::allows('audit', Asset::class),
       'delete' => ($asset->deleted_at == '' && $asset->assigned_to == '' && Gate::allows('delete', Asset::class)),
   ];
   ```

4. **可选关联加载** (第183-203行):
   - 当请求参数 `components=true` 时，额外输出组件信息

---

### 2.2 Presenter 层 - 表格列配置

**文件位置**: `app/Presenters/AssetPresenter.php`

**核心方法**: `dataTableLayout()` (第20行)

**列配置特性**:

1. **显示控制**:
   - `visible` 属性: 控制列默认显示/隐藏
   - `switchable` 属性: 控制列是否可被用户切换显示
   - `searchable`/`sortable` 属性: 控制搜索和排序功能

2. **典型列配置示例**:
   ```php
   // 资产标签 (默认显示，带链接格式化)
   [
       'field' => 'asset_tag',
       'searchable' => true,
       'sortable' => true,
       'switchable' => false,
       'title' => trans('admin/hardware/table.asset_tag'),
       'visible' => true,
       'formatter' => 'hardwareLinkFormatter',
   ],
   
   // 公司 (默认隐藏)
   [
       'field' => 'company',
       'searchable' => true,
       'sortable' => true,
       'switchable' => true,
       'title' => trans('general.company'),
       'visible' => false,
       'formatter' => 'companiesLinkObjFormatter',
   ],
   
   // 操作按钮 (不可切换，固定显示)
   [
       'field' => 'actions',
       'searchable' => false,
       'sortable' => false,
       'switchable' => false,
       'title' => trans('table.actions'),
       'formatter' => 'hardwareActionsFormatter',
       'printIgnore' => true,
       'class' => 'hidden-print',
   ],
   ```

3. **自定义字段列动态生成** (第335-354行):
   ```php
   foreach ($fields as $field) {
       $layout[] = [
           'field' => $field->db_column,
           'searchable' => true,
           'sortable' => true,
           'switchable' => true,
           'title' => $field->name,
           'formatter' => 'customFieldsFormatter',
           'class' => ($field->field_encrypted == '1') ? 'css-padlock' : '',
           'visible' => ($field->show_in_listview == '1') ? true : false,
       ];
   }
   ```

---

### 2.3 前端 Formatter 层 - 显示格式化

**文件位置**: `resources/views/partials/bootstrap-table.blade.php`

**核心格式化器函数**:

1. **`customFieldsFormatter(value, row)`** (第1928行):
   ```javascript
   function customFieldsFormatter(value, row) {
       var field_column = this.title;
       var field_column_plain = field_column.replace(/<(?:.|\n)*?> ?/gm, '');
       
       if (row.custom_fields && row.custom_fields[field_column_plain]) {
           var field = row.custom_fields[field_column_plain];
           
           // 根据字段类型特殊格式化
           if (field.field_format == 'URL') {
               return '<a href="' + field.value + '" target="_blank">' + field.value + '</a>';
           } else if (field.field_format == 'BOOLEAN') {
               return (field.value == 1) 
                   ? "<span class='fas fa-check-circle' style='color:green'>" 
                   : "<span class='fas fa-times-circle' style='color:red' />";
           } else if (field.field_format == 'EMAIL') {
               return '<a href="mailto:' + field.value + '"><x-icon type="email" /> ' + field.value + '</a>';
           }
           return field.value;
       }
   }
   ```

2. **`dateDisplayFormatter(value)`** (第2113行):
   - 从 API 返回的日期对象中提取 `formatted` 字段供人类阅读

3. **`imageFormatter(value, row)`** (第2191行):
   - 将图片 URL 渲染为 `<img>` 标签，设置高度宽度

4. **`polymorphicItemFormatter(value)`** (第1710行):
   - 处理 `assigned_to` 字段，根据类型（用户/资产/位置）渲染不同的链接格式

5. **`hardwareActionsFormatter(value, row)`**:
   - 根据 `row.available_actions` 中的布尔值渲染具体的操作按钮组
   - 包括编辑、删除、借出、归还、审计等按钮

6. **`hardwareInOutFormatter(value, row)`**:
   - 渲染快速借出/归还操作按钮列

---

## 3. 权限过滤三层架构

### 3.1 数据级权限 - Global Scope

**文件位置**: `app/Models/CompanyableScope.php`, `app/Models/Traits/CompanyableTrait.php`

**触发时机**: 模型查询时自动应用，无需手动调用

**核心逻辑** (`app/Models/Company.php:289-328`):
```php
public static function scopeCompanyables($query, $column = 'company_id', $table_name = null)
{
    // 超级用户或 FMCS 未启用时不过滤
    if (! self::isFullMultipleCompanySupportEnabled() 
        || (Auth::hasUser() && auth()->user()->isSuperUser()) 
        || (! Auth::hasUser())) {
        return $query;
    }
    
    // 普通用户只看所属公司数据
    return self::scopeCompanyablesDirectly($query, $column, $table_name);
}

private static function scopeCompanyablesDirectly($query, $column = 'company_id', $table_name = null)
{
    $company_id = auth()->user()->company_id;
    $table = ($table_name) ? $table_name.'.' : $query->getModel()->getTable().'.';
    
    return $query->where($table.$column, '=', $company_id);
}
```

**应用范围**: 所有使用 `CompanyableTrait` 的模型自动应用此 Scope

---

### 3.2 动作级权限 - Policy

**文件位置**: `app/Policies/AssetPolicy.php` → `CheckoutablePermissionsPolicy.php` → `SnipePermissionsPolicy.php`

**核心校验点**:

1. **前置检查** (`SnipePermissionsPolicy.php:38-73`):
   ```php
   public function before(User $user, $ability, $item)
   {
       // 管理员跳过后续检查（但仍受公司限制）
       if ($user->hasAccess('admin')) {
           return true;
       }
       
       // 非模型实例跳过公司检查
       if (! $item instanceof Model) {
           return;
       }
       
       // 公司权限检查
       if (! Company::isCurrentUserHasAccess($item)) {
           return false;
       }
   }
   ```

2. **标准权限方法**:
   - `index(User $user)`: 检查 `assets.view` 权限
   - `view(User $user, $item)`: 检查 `assets.view` 权限
   - `create(User $user)`: 检查 `assets.create` 权限
   - `update(User $user, $item)`: 检查 `assets.edit` 权限
   - `delete(User $user, $item)`: 检查 `assets.delete` 权限
   - `checkout(User $user, $item)`: 检查 `assets.checkout` 权限
   - `checkin(User $user, $item)`: 检查 `assets.checkin` 权限
   - `audit(User $user, $item)`: 检查 `assets.audit` 权限

---

### 3.3 字段级权限 - Transformer 内部

**文件位置**: `app/Http/Transformers/AssetsTransformer.php`

**加密自定义字段** (第133-143行):
```php
if ($field->isFieldDecryptable($asset->{$field->db_column})) {
    $decrypted = Helper::gracefulDecrypt($field, $asset->{$field->db_column});
    $value = (Gate::allows('assets.view.encrypted_custom_fields')) 
        ? $decrypted 
        : strtoupper(trans('admin/custom_fields/general.encrypted'));
    
    if ($field->format == 'DATE') {
        if (Gate::allows('assets.view.encrypted_custom_fields')) {
            $value = Helper::getFormattedDateObject($value, 'date', false);
        } else {
            $value = strtoupper(trans('admin/custom_fields/general.encrypted'));
        }
    }
}
```

**可用动作过滤** (第173-181行):
- 根据用户权限和资产状态（是否删除、是否分配）动态决定可用操作

---

## 4. 字段差异对照表

| 字段/特性 | API 输出格式 | 网页表格显示 | 差异原因 |
|---------|-------------|-------------|---------|
| **基础字段** | | | |
| `id` | 整数 | 默认隐藏列 | Presenter `visible: false` |
| `asset_tag` | 纯字符串 | 带超链接的标签 | Formatter `hardwareLinkFormatter` |
| `name` | 纯字符串 | 带超链接的名称 | Formatter `hardwareLinkFormatter` |
| `serial` | 纯字符串 | 带超链接的序列号 | Formatter `hardwareLinkFormatter` |
| **关联对象** | | | |
| `status` | 对象: `{id, name, status_type, status_meta}` | 带样式和颜色的状态标签链接 | Formatter `statuslabelsLinkObjFormatter` |
| `category` | 对象: `{id, name, tag_color}` | 分类名称链接 | Formatter `categoriesLinkObjFormatter` |
| `manufacturer` | 对象: `{id, name, tag_color}` | 厂商名称链接（默认隐藏） | Presenter `visible: false` + Formatter |
| `model` | 对象: `{id, name}` | 型号名称链接 | Formatter `modelsLinkObjFormatter` |
| `company` | 对象: `{id, name, tag_color}` | 公司名称链接（默认隐藏） | Presenter `visible: false` + Formatter |
| `location` | 对象: `{id, name, tag_color}` | 位置名称链接 | Formatter `deployedLocationFormatter` |
| `rtd_location` | 对象: `{id, name, tag_color}` | 默认位置（默认隐藏） | Presenter `visible: false` |
| `supplier` | 对象: `{id, name, tag_color}` | 供应商（默认隐藏） | Presenter `visible: false` |
| **分配对象** | | | |
| `assigned_to` (用户) | 对象: `{id, username, name, first_name, last_name, email, employee_number, jobtitle, type}` | 仅显示用户名称链接 | Formatter `polymorphicItemFormatter` 提取 display_name |
| `assigned_to` (资产/位置) | 对象: `{id, name, type}` | 仅显示名称链接 | Formatter `polymorphicItemFormatter` |
| **日期字段** | | | |
| `created_at`, `updated_at`, `purchase_date`, etc. | 对象: `{ datetime: "Y-m-d H:i:s", formatted: "Y/m/d" }` | 仅显示 `formatted` 日期字符串 | Formatter `dateDisplayFormatter` |
| `age` | 人类可读字符串 (e.g. "2 years ago") | 默认隐藏 | Presenter `visible: false` |
| **数值字段** | | | |
| `purchase_cost` | 格式化货币字符串 (e.g. "$1,299.00") | 右对齐显示，支持页脚求和 | `class: 'text-right'` + `footerFormatter: 'sumFormatter'` |
| `book_value` | 格式化货币字符串 | 右对齐显示，支持页脚求和（默认隐藏） | Presenter `visible: false` + 样式配置 |
| `checkout_counter`, `checkin_counter`, `requests_counter` | 整数 | 默认隐藏列 | Presenter `visible: false` |
| **自定义字段** | | | |
| `custom_fields` | 对象: `{field_name: {field: db_column, value: "...", field_format: "...", element: "..."}}` | 根据 `field_format` 渲染为链接/图标/纯文本；仅 `show_in_listview=1` 的字段默认显示 | 1. Presenter 过滤 `visible` 属性<br>2. Formatter 按类型格式化 |
| 加密自定义字段 | 有权限: 明文值<br>无权限: "ENCRYPTED" | 同 API，额外添加 🔒 图标 CSS | 1. Transformer 权限检查<br>2. Formatter `class: 'css-padlock'` |
| **权限与操作** | | | |
| `available_actions` | 对象: `{checkout: true, checkin: false, clone: true, ...}` | 渲染为具体操作按钮组（编辑、删除、借出、归还等） | Formatter `hardwareActionsFormatter` |
| `user_can_checkout` | 布尔值 | 不直接显示，用于控制借出按钮状态 | Formatter 内部逻辑 |
| **特殊字段** | | | |
| `checkbox` | API 不输出 | 表格第一列复选框 | Presenter 新增配置列 |
| `checkincheckout` | API 不输出 | 快速借出/归还按钮列 | Presenter 新增配置列 + Formatter |
| `actions` | API 不输出 | 操作按钮列 | Presenter 新增配置列 + Formatter |
| `image` | URL 字符串 | `<img>` 标签（50x50） | Formatter `imageFormatter` |
| `qr`, `alt_barcode` | URL 字符串 | API 独有，表格不显示 | Presenter 未定义对应列 |
| `notes` | Markdown 解析后的 HTML | 默认隐藏列 | Presenter `visible: false` |
| `order_number` | 纯字符串 | 默认隐藏列 | Presenter `visible: false` |
| `warranty_months` | 字符串 (e.g. "12 months") | 默认隐藏列 | Presenter `visible: false` |
| `eol` | 字符串 (e.g. "24 months") | 默认隐藏列 | Presenter `visible: false` |
| `byod`, `requestable` | 布尔值 | 默认隐藏，显示为 ✓/✗ 图标 | Presenter `visible: false` + Formatter `trueFalseFormatter` |
| `created_by` | 对象: `{id, name}` | 默认隐藏列 | Presenter `visible: false` + Formatter |

---

## 5. 设计意图与架构思考

### 5.1 API 层设计目标
- **完整性优先**: 输出所有可用字段，便于第三方集成
- **机器可读性**: 日期、关联对象采用结构化格式，便于程序处理
- **扩展性**: 通过 `available_actions` 提供元数据，客户端可自主决策 UI 呈现
- **向后兼容**: 保留 `status_label` 等 legacy 字段（标记待移除）

### 5.2 网页表格层设计目标
- **用户体验优先**: 数据格式化为最易读的形式（链接、图标、颜色）
- **可配置性**: 列可通过 `visible`、`switchable` 灵活控制显示
- **交互性**: 操作按钮直接嵌入表格，减少页面跳转
- **性能**: 通过前端 Formatter 减轻后端渲染压力

### 5.3 权限分层设计
1. **数据层 (Global Scope)**: 最底层防护，确保查询结果从根源上就是安全的
2. **动作层 (Policy)**: 中间层校验，确保用户只能执行授权操作
3. **字段层 (Transformer)**: 最细粒度控制，敏感字段按需脱敏

这种三层权限设计遵循了**深度防御**原则，即使某一层被绕过，其他层仍能提供保护。

---

## 6. 潜在问题与优化建议

### 6.1 已知问题
1. **数据重复查询**: API 和网页表格使用相同的后端接口，但网页表格可能不需要完整的关联对象数据（如 `assigned_to` 的完整用户信息）
2. **前端权限逻辑重复**: `available_actions` 在后端已计算，但前端 Formatter 可能需要再次判断状态
3. **自定义字段处理不一致**: API 输出完整的字段元数据，但前端格式化时需要重新解析

### 6.2 优化建议
1. **可考虑添加字段裁剪参数**: API 支持 `?fields=id,name,asset_tag` 风格的字段选择，减少数据传输
2. **Formatter 逻辑可复用**: 部分格式化逻辑（如日期、布尔值）可考虑移至后端 Transformer 统一处理
3. **缓存 Presenter 配置**: `dataTableLayout()` 在每次页面加载时重新生成，可考虑缓存结果

---

## 7. 关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| API 控制器 | `app/Http/Controllers/Api/AssetsController.php` | 65 (`index`), 560 (`show`) |
| 网页控制器 | `app/Http/Controllers/Assets/AssetsController.php` | 66 (`index`) |
| 资产 Transformer | `app/Http/Transformers/AssetsTransformer.php` | 30 (`transformAsset`), 129 (自定义字段), 173 (可用动作) |
| 表格列配置 | `app/Presenters/AssetPresenter.php` | 20 (`dataTableLayout`) |
| 前端格式化器 | `resources/views/partials/bootstrap-table.blade.php` | 1628 (通用动作), 1928 (自定义字段), 2191 (图片) |
| 公司 Scope | `app/Models/CompanyableScope.php` | 16 |
| 公司 Scope 逻辑 | `app/Models/Company.php` | 289 (`scopeCompanyables`), 305 (`scopeCompanyablesDirectly`) |
| 权限基类 | `app/Policies/SnipePermissionsPolicy.php` | 38 (`before`), 80 (`index`) |
| 资产权限类 | `app/Policies/AssetPolicy.php` | 8 |
| 表格初始化 | `resources/views/partials/bootstrap-table.blade.php` | 478 (bootstrapTable 初始化) |
| 资产列表视图 | `resources/views/hardware/index.blade.php` | 58 (`<x-table.assets>` 组件) |
