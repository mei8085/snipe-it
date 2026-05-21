# Snipe-IT 资产数据 API 与网页表格字段差异分析

## 1. 整体架构与数据流向

### 1.1 API 数据路径
```
HTTP 请求
    ↓
Api\AssetsController@index (app/Http/Controllers/Api/AssetsController.php:65)
    ↓ 权限校验: $this->authorize('index', Asset::class) [第91行]
    ↓ 字段白名单校验: $allowed_columns 数组 [第96-145行]
    ↓ 数据查询 (自动应用 CompanyableScope)
    ↓ 关联预加载: model, location, status, company, assignedTo, etc.
    ↓ 排序白名单校验: in_array($sort_override, $allowed_columns) [第381行]
    ↓ 分页处理: skip($offset)->take($limit) [第452行]
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
    ↓ 包装分页外层结构
    ↓
JSON 响应: { total, rows, current_page, per_page, total_pages, prev_page_url, next_page_url }
```

### 1.2 网页表格数据路径
```
浏览器访问 /hardware
    ↓
Assets\AssetsController@index (app/Http/Controllers/Assets/AssetsController.php:66)
    ↓ 权限校验: $this->authorize('index', Asset::class) [第68行]
    ↓
视图: hardware/index.blade.php [第58行]
    ↓
<x-table.assets> 组件
    ↓ 传入 :route="route('api.assets.index', [...])"
    ↓
Bootstrap Table 初始化 (resources/views/partials/bootstrap-table.blade.php:478)
    ↓ sidePagination: 'server' 模式
    ↓ AJAX 请求 api.assets.index 路由（携带 CSRF Token）
    ↓ （数据查询与转换路径与 API 完全相同）
    ↓
接收 JSON 数据
    ↓
列配置过滤: AssetPresenter::dataTableLayout() (app/Presenters/AssetPresenter.php:20)
    ↓ 根据 visible 字段判断是否显示
    ↓ 未设置 visible 属性的列默认显示（如 purchase_cost, book_value）
    ↓ 根据 formatter 调用对应的 JS 函数
    ↓
JavaScript Formatter 二次加工
    ↓
表格渲染完成
```

---

## 2. 网页列表入口可见性门禁与接口鉴权衔接

### 2.1 双重权限校验机制

**网页入口控制器** (`app/Http/Controllers/Assets/AssetsController.php:66-73`):
```php
public function index(Request $request): View
{
    // 第一道门禁：检查用户是否有资产列表访问权限
    $this->authorize('index', Asset::class);  // [第68行]
    
    $companyId = $request->input('company_id');
    $company = is_scalar($companyId) ? Company::find($companyId) : null;

    return view('hardware/index')->with('company', $company);
}
```

**API 接口控制器** (`app/Http/Controllers/Api/AssetsController.php:65-92`):
```php
public function index(FilterRequest $request, $action = null, $upcoming_status = null): JsonResponse|array
{
    // 根据路由动态决定权限检查
    if (Route::currentRouteName() == 'api.depreciation-report.index') {
        $filter_non_deprecable_assets = true;
        $transformer = 'App\Http\Transformers\DepreciationReportTransformer';
        $this->authorize('reports.view');  // 报表权限 [第88行]
    } else {
        $transformer = 'App\Http\Transformers\AssetsTransformer';
        // 第二道门禁：与网页入口使用完全相同的权限检查
        $this->authorize('index', Asset::class);  // [第91行]
    }
    // ...
}
```

### 2.2 权限衔接机制

| 层次 | 网页入口 | API 接口 | 衔接方式 |
|------|---------|---------|---------|
| **认证层** | `$this->middleware('auth')` [构造函数第53行] | Laravel API 中间件（auth:sanctum 或 session） | 共享同一用户身份 |
| **动作权限层** | `$this->authorize('index', Asset::class)` | `$this->authorize('index', Asset::class)` | 调用同一 Policy 方法 |
| **数据权限层** | 视图渲染时无数据查询 | 自动应用 `CompanyableScope` | 查询时自动过滤所属公司 |
| **字段权限层** | 无（由前端列配置控制） | Transformer 内部加密字段检查 | 最终由 Transformer 统一控制 |

### 2.3 Policy 权限校验流程

**基类前置检查** (`app/Policies/SnipePermissionsPolicy.php:38-73`):
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
    
    // 公司权限检查 - 确保用户只能访问所属公司数据
    if (! Company::isCurrentUserHasAccess($item)) {
        return false;
    }
}
```

**资产权限类** (`app/Policies/AssetPolicy.php:8-25`):
```php
class AssetPolicy extends CheckoutablePermissionsPolicy
{
    protected $section = 'assets';
    
    // index 权限检查 assets.view
    // view 权限检查 assets.view
    // create 权限检查 assets.create
    // update 权限检查 assets.edit
    // delete 权限检查 assets.delete
    // checkout 权限检查 assets.checkout
    // checkin 权限检查 assets.checkin
    // audit 权限检查 assets.audit
}
```

---

## 3. 列表接口字段白名单与排序白名单

### 3.1 字段白名单定义

**文件位置**: `app/Http/Controllers/Api/AssetsController.php:96-145`

```php
$allowed_columns = [
    // 基础字段 - 直接对应 assets 表列
    'id', 'name', 'asset_tag', 'serial', 'model_number',
    'last_checkout', 'last_checkin', 'notes', 'expected_checkin',
    'order_number', 'image', 'assigned_to', 'created_at', 'updated_at',
    'purchase_date', 'purchase_cost', 'last_audit_date', 'next_audit_date',
    'warranty_months', 'checkout_counter', 'checkin_counter', 'requests_counter',
    'byod', 'asset_eol_date', 'requestable', 'jobtitle',
    
    // 关联字段 - 特殊处理排序逻辑
    // 注释明确说明：这些是*relationships*，需要在下方 switch 中特殊处理
    'company', 'model', 'location', 'rtd_location', 'category',
    'manufacturer', 'supplier', 'status', 'jobtitle', 'assigned_to', 'created_by',
];

// 动态添加所有自定义字段的数据库列名
$all_custom_fields = CustomField::all();
foreach ($all_custom_fields as $field) {
    $allowed_columns[] = $field->db_column_name();  // [第143-145行]
}
```

### 3.2 排序白名单校验机制

**文件位置**: `app/Http/Controllers/Api/AssetsController.php:374-445`

```php
// 处理 Bootstrap Table 传递的 custom_fields.xxx 格式
$sort_override = str_replace('custom_fields.', '', $request->input('sort'));  // [第377行]

// 白名单校验：不在允许列表中的字段，默认使用 created_at 排序
$column_sort = in_array($sort_override, $allowed_columns) ? $sort_override : 'assets.created_at';  // [第381行]

$order = $request->input('order') === 'asc' ? 'asc' : 'desc';  // [第383行]

// 关联字段特殊排序处理
switch ($sort_override) {
    case 'model':      $assets->OrderModels($order);      break;
    case 'category':   $assets->OrderCategory($order);    break;
    case 'manufacturer': $assets->OrderManufacturer($order); break;
    case 'company':    $assets->OrderCompany($order);     break;
    case 'location':   $assets->OrderLocation($order);    break;
    case 'rtd_location': $assets->OrderRtdLocation($order); break;
    case 'status':     $assets->OrderStatus($order);      break;
    case 'supplier':   $assets->OrderSupplier($order);    break;
    case 'assigned_to': $assets->OrderAssigned($order);   break;
    case 'jobtitle':   $assets->OrderByJobTitle($order);  break;
    case 'created_by': $assets->OrderByCreatedByName($order); break;
    default:
        // 自定义字段排序 - 检查是否为 NUMERIC 类型
        if (array_search($column_sort, $all_custom_fields->pluck('db_column')->toArray()) !== false) {
            $numeric_sort = false;
            foreach ($all_custom_fields as $field) {
                if (($field->db_column == $sort_override) && ($field->format == 'NUMERIC')) {
                    $numeric_sort = true;
                    break;
                }
            }
            // 数值类型使用 * 1 转换排序，字符串直接排序
            if ($numeric_sort) {
                $assets->orderByRaw(DB::getTablePrefix().'assets.'.$sort_override.' * 1 '.$order);
            } else {
                $assets->orderBy($sort_override, $order);
            }
        } else {
            // 普通字段直接排序
            $assets->orderBy($column_sort, $order);  // [第442行]
        }
        break;
}
```

### 3.3 白名单对可见字段的影响

1. **直接影响**: 只有在 `$allowed_columns` 中的字段才能作为排序参数使用，非法字段会被静默替换为 `created_at`

2. **间接影响**: 
   - 白名单包含 `model`、`status` 等关联字段名称，但这些不是真实的数据库列
   - 需要通过 `switch` 语句转换为 `join` + `orderBy` 的复杂查询
   - 自定义字段通过动态添加到白名单，支持按自定义字段排序

3. **安全意义**: 防止 SQL 注入，用户无法通过构造 `sort=(select password from users...)` 进行攻击

---

## 4. 分配对象展示字段与净值列默认显示状态

### 4.1 分配对象 (assigned_to) 字段

**Presenter 配置** (`app/Presenters/AssetPresenter.php:103-108`):
```php
[
    'field' => 'assigned_to',
    'searchable' => true,
    'sortable' => true,
    'title' => trans('admin/hardware/form.checkedout_to'),
    'visible' => true,  // 明确设置为可见
    'formatter' => 'polymorphicItemFormatter',  // 多态格式化器
],
```

**API 输出格式** (不同类型的分配对象):
```php
// 分配给用户时
'assigned_to' => [
    'id' => (int) $asset->assignedTo->id,
    'username' => e($asset->assignedTo->username),
    'name' => e($asset->assignedTo->getFullNameAttribute()),
    'first_name' => e($asset->assignedTo->first_name),
    'last_name' => e($asset->assignedTo->last_name),
    'email' => e($asset->assignedTo->email),
    'employee_number' => e($asset->assignedTo->employee_num),
    'jobtitle' => e($asset->assignedTo->jobtitle),
    'type' => 'user',  // 关键标记
],

// 分配给资产时
'assigned_to' => [
    'id' => (int) $asset->assignedTo->id,
    'name' => e($asset->assignedTo->name),
    'type' => 'asset',
],

// 分配给位置时
'assigned_to' => [
    'id' => (int) $asset->assignedTo->id,
    'name' => e($asset->assignedTo->name),
    'type' => 'location',
],
```

**前端格式化器** (`resources/views/partials/bootstrap-table.blade.php:1710-1765`):
```javascript
function polymorphicItemFormatter(value) {
    var item_destination = '';
    var item_icon;

    if ((value) && (value.type)) {
        // 根据 type 字段决定链接目标和图标
        if (value.type == 'asset') {
            item_destination = 'hardware';
            item_icon = 'fas fa-barcode';
        } else if (value.type == 'user') {
            item_destination = 'users';
            item_icon = 'fas fa-user';
        } else if (value.type == 'location') {
            item_destination = 'locations';
            item_icon = 'fas fa-map-marker-alt';
        }
        // ... 其他类型: accessory, component, consumable, license, maintenance
        
        // 只显示 display_name（用户显示全名，其他显示 name）
        var display_name = value.name;
        if (value.type == 'user') {
            display_name = (value.first_name || value.last_name) 
                ? (value.first_name + ' ' + value.last_name).trim() 
                : value.username;
        }
        
        return '<a href="{{ config('app.url') }}/' + item_destination + '/' + value.id + '">'
            + '<i class="' + item_icon + '" aria-hidden="true"></i> '
            + display_name + '</a>';
    }
    return '';
}
```

### 4.2 净值列 (book_value) 与采购成本列 (purchase_cost)

**Presenter 配置对比**:

| 字段 | 配置项 | purchase_cost (第174-179行) | book_value (第181-186行) |
|------|--------|----------------------------|--------------------------|
| `field` | | `'purchase_cost'` | `'book_value'` |
| `searchable` | | `true` | `false` |
| `sortable` | | `true` | `false` |
| `title` | | `trans('general.purchase_cost')` | `trans('admin/hardware/table.book_value')` |
| `footerFormatter` | | `'sumFormatter'` | `'sumFormatter'` |
| `class` | | `'text-right'` | `'text-right'` |
| **`visible`** | | **未设置** | **未设置** |

**真实行为分析**:
- Bootstrap Table 中 `visible` 属性默认为 `true`
- 两列均未显式设置 `visible` 属性，因此**默认都显示**
- `purchase_cost` 可搜索可排序，`book_value` 不可搜索不可排序
- 两列都支持页脚求和 (`footerFormatter: 'sumFormatter'`)
- 两列都右对齐 (`class: 'text-right'`)

**对比其他隐藏列的配置**:
```php
// model_number 明确设置为隐藏
[
    'field' => 'model_number',
    'visible' => false,  // [第86行] 显式隐藏
],

// company 明确设置为隐藏
[
    'field' => 'company',
    'visible' => false,  // [第57行] 显式隐藏
],
```

> **结论**: `purchase_cost` 和 `book_value` 由于未设置 `visible: false`，在网页表格中默认显示。这是代码事实，而非文档声称的「默认隐藏」。

---

## 5. 分页外层结构在表格消费端的作用

### 5.1 分页外层结构定义

**DatatablesTransformer** (`app/Http/Transformers/DatatablesTransformer.php:10-32`):
```php
public function transformDatatables($objects, $total = null)
{
    $objects_array = [
        'total' => $total ?? count($objects),        // 总记录数
        'rows' => $objects,                          // 当前页数据
    ];
    
    $current_page = app('api_current_page');        // 当前页码
    $limit = (int) app('api_limit_value');          // 每页条数
    $total_pages = $limit > 0 ? (int) ceil($objects_array['total'] / $limit) : 1;

    $objects_array['current_page'] = $current_page;
    $objects_array['per_page'] = $limit;
    $objects_array['total_pages'] = $total_pages;
    
    // 前后页链接
    $objects_array['prev_page_url'] = $current_page > 1
        ? request()->fullUrlWithQuery(['page' => $current_page - 1])
        : null;
    $objects_array['next_page_url'] = $current_page < $total_pages
        ? request()->fullUrlWithQuery(['page' => $current_page + 1])
        : null;

    return $objects_array;
}
```

### 5.2 完整响应结构示例

```json
{
    "total": 156,
    "rows": [
        { "id": 1, "name": "MacBook Pro", "asset_tag": "ASSET-001", ... },
        { "id": 2, "name": "Dell XPS", "asset_tag": "ASSET-002", ... },
        ... 最多 50 条记录 ...
    ],
    "current_page": 1,
    "per_page": 50,
    "total_pages": 4,
    "prev_page_url": null,
    "next_page_url": "https://snipe-it.example.com/api/v1/hardware?page=2&limit=50"
}
```

### 5.3 Bootstrap Table 服务端分页配置

**表格 HTML 属性** (以 `checkin-due.blade.php` 为例):
```html
<table
    data-columns="{{ \App\Presenters\AssetPresenter::dataTableLayout() }}"
    data-side-pagination="server"        <!-- 关键：启用服务端分页 -->
    data-url="{{ route('api.assets.index') }}"  <!-- API 地址 -->
    data-pagination="true"               <!-- 启用分页控件 -->
    data-page-size="50"                  <!-- 默认每页条数 -->
    data-page-list="[25, 50, 100, 500]"  <!-- 可选每页条数 -->
    ...
>
</table>
```

**JavaScript 初始化配置** (`resources/views/partials/bootstrap-table.blade.php:518-580`):
```javascript
$(this).bootstrapTable({
    // AJAX 请求携带 CSRF Token
    ajaxOptions: {
        headers: {
            'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
        }
    },
    
    sidePagination: 'server',    // 服务端分页模式
    pagination: true,            // 显示分页控件
    paginationFirstText: "{{ trans('general.first') }}",
    paginationLastText: "{{ trans('general.last') }}",
    paginationNextText: "{{ trans('general.next') }}",
    paginationPreText: "{{ trans('general.previous') }}",
    
    // 高级搜索在服务端模式下跳过本地搜索
    if (!this.options.advancedSearch || this.options.sidePagination === 'server') {
        return;  // [第391-392行] 服务端模式不执行本地过滤
    }
});
```

### 5.4 各字段在表格消费端的具体作用

| 字段 | 作用 | 消费端位置 |
|------|------|-----------|
| **`total`** | 显示总记录数，计算总页数 | Bootstrap Table 内部 `options.totalField = 'total'` (默认值) |
| **`rows`** | 表格行数据数组 | Bootstrap Table 内部 `options.dataField = 'rows'` (默认值) |
| **`current_page`** | 标记当前页码，用于翻页按钮状态 | 主要用于 API 调用方，Bootstrap Table 通过 `offset`/`limit` 自行维护 |
| **`per_page`** | 每页记录数 | 对应 Bootstrap Table 的 `pageSize` 选项 |
| **`total_pages`** | 总页数 | 对应 `options.totalPages`，控制分页按钮数量 |
| **`prev_page_url`** | 上一页完整链接 | 供 API 消费者直接调用，Bootstrap Table 不直接使用 |
| **`next_page_url`** | 下一页完整链接 | 供 API 消费者直接调用，Bootstrap Table 不直接使用 |

### 5.5 分页交互流程

```
Bootstrap Table 初始化
    ↓
用户点击「下一页」或选择每页条数
    ↓
Bootstrap Table 自动构造请求参数:
    ?offset=50&limit=50&sort=asset_tag&order=asc
    ↓
API 控制器接收参数:
    $offset = $request->input('offset', 0)  [第448行]
    $limit = app('api_limit_value')         [第449行]
    ↓
数据库查询:
    $total = $assets->count();               [第451行]
    $assets = $assets->skip($offset)->take($limit)->get();  [第452行]
    ↓
DatatablesTransformer 包装分页结构
    ↓
返回 JSON 响应
    ↓
Bootstrap Table 解析响应:
    - 使用 total 更新分页控件
    - 使用 rows 渲染表格行
    - 更新页码按钮状态
```

---

## 6. 三层核心差异点（补充修正版）

### 6.1 Transformer 层 - API 输出规范

**文件位置**: `app/Http/Transformers/AssetsTransformer.php`

**核心方法**: `transformAsset(Asset $asset)` (第30行)

**输出特性**:

1. **完整字段输出** (第35-127行):
   - 基础字段: `id`, `name`, `asset_tag`, `serial`, `model`, `model_number`, `byod`, `requestable`
   - 关联对象: `status_label`(兼容), `status`, `category`, `manufacturer`, `depreciation`, `supplier`, `company`, `location`, `rtd_location`, `assigned_to`
   - 日期字段: 统一格式化为 `{ datetime: "Y-m-d H:i:s", formatted: "Y/m/d" }` 结构
   - 数值字段: `purchase_cost`, `book_value`, `checkin_counter`, `checkout_counter`, `requests_counter`
   - 其他: `image`, `qr`, `alt_barcode`, `warranty_months`, `warranty_expires`, `notes`, `order_number`, `age`, `eol`, `asset_eol_date`

2. **自定义字段处理** (第129-171行):
   ```php
   // 加密字段权限检查 (第135行)
   $value = (Gate::allows('assets.view.encrypted_custom_fields')) 
       ? $decrypted 
       : strtoupper(trans('admin/custom_fields/general.encrypted'));
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

### 6.2 Presenter 层 - 表格列配置

**文件位置**: `app/Presenters/AssetPresenter.php`

**核心方法**: `dataTableLayout()` (第20行)

**关键列配置事实**:

| 字段 | visible 值 | 默认状态 |
|------|-----------|---------|
| `id` | `false` | 隐藏 |
| `asset_tag` | `true` | 显示 |
| `name` | `true` | 显示 |
| `model` | `true` | 显示 |
| `category` | `true` | 显示 |
| `status` | `true` | 显示 |
| `assigned_to` | `true` | 显示 |
| `location` | `true` | 显示 |
| `purchase_cost` | **未设置** | **显示（默认）** |
| `book_value` | **未设置** | **显示（默认）** |
| `company` | `false` | 隐藏 |
| `model_number` | `false` | 隐藏 |
| `order_number` | `false` | 隐藏 |
| `notes` | `false` | 隐藏 |

> **重要修正**: 之前分析错误地认为 `book_value` 默认隐藏。实际代码中该列未设置 `visible` 属性，Bootstrap Table 默认为 `true`，因此默认显示。

### 6.3 前端 Formatter 层 - 显示格式化

**文件位置**: `resources/views/partials/bootstrap-table.blade.php`

**核心格式化器函数**:

1. **`customFieldsFormatter(value, row)`** (第1928行): 按 URL/BOOLEAN/EMAIL 类型格式化
2. **`dateDisplayFormatter(value)`** (第2113行): 提取日期对象的 `formatted` 字段
3. **`polymorphicItemFormatter(value)`** (第1710行): 处理多态关联对象的显示
4. **`imageFormatter(value, row)`** (第2191行): 渲染图片标签
5. **`hardwareActionsFormatter(value, row)`**: 根据 `available_actions` 渲染操作按钮

---

## 7. 完整字段差异对照表（修正版）

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
| `assigned_to` (用户) | 对象: `{id, username, name, first_name, last_name, email, employee_number, jobtitle, type}` | 仅显示用户名称链接 + 用户图标 | 1. Presenter `visible: true`<br>2. Formatter `polymorphicItemFormatter` 提取 display_name + 图标 |
| `assigned_to` (资产/位置) | 对象: `{id, name, type}` | 仅显示名称链接 + 对应图标 | Formatter `polymorphicItemFormatter` |
| **日期字段** | | | |
| `created_at`, `updated_at`, `purchase_date`, etc. | 对象: `{ datetime: "Y-m-d H:i:s", formatted: "Y/m/d" }` | 仅显示 `formatted` 日期字符串 | Formatter `dateDisplayFormatter` |
| `age` | 人类可读字符串 (e.g. "2 years ago") | 默认隐藏 | Presenter `visible: false` |
| **数值字段** | | | |
| `purchase_cost` | 格式化货币字符串 (e.g. "$1,299.00") | 右对齐显示，支持页脚求和，**默认显示** | Presenter 未设置 visible（默认 true） + `class: 'text-right'` + `footerFormatter: 'sumFormatter'` |
| `book_value` | 格式化货币字符串 | 右对齐显示，支持页脚求和，**默认显示** | Presenter 未设置 visible（默认 true） + `class: 'text-right'` + `footerFormatter: 'sumFormatter'` |
| `checkout_counter`, `checkin_counter`, `requests_counter` | 整数 | 默认隐藏列 | Presenter `visible: false` |
| **自定义字段** | | | |
| `custom_fields` | 对象: `{field_name: {field: db_column, value: "...", field_format: "...", element: "..."}}` | 根据 `field_format` 渲染为链接/图标/纯文本；仅 `show_in_listview=1` 的字段默认显示 | 1. Presenter 过滤 `visible` 属性<br>2. Formatter 按类型格式化 |
| 加密自定义字段 | 有权限: 明文值<br>无权限: "ENCRYPTED" | 同 API，额外添加 🔒 图标 CSS | 1. Transformer 权限检查<br>2. Formatter `class: 'css-padlock'` |
| **权限与操作** | | | |
| `available_actions` | 对象: `{checkout: true, checkin: false, clone: true, ...}` | 渲染为具体操作按钮组（编辑、删除、借出、归还等） | Formatter `hardwareActionsFormatter` |
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

## 8. 关键代码位置索引（精确版）

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 网页入口控制器 | `app/Http/Controllers/Assets/AssetsController.php` | 66 (`index`), 68 (`authorize`) |
| API 控制器 | `app/Http/Controllers/Api/AssetsController.php` | 65 (`index`), 91 (`authorize`), 96-145 (`$allowed_columns`), 377-445 (排序逻辑), 451-452 (分页) |
| 资产 Transformer | `app/Http/Transformers/AssetsTransformer.php` | 30 (`transformAsset`), 129 (自定义字段), 135 (加密检查), 173 (可用动作) |
| 分页 Transformer | `app/Http/Transformers/DatatablesTransformer.php` | 10 (`transformDatatables`) |
| 表格列配置 | `app/Presenters/AssetPresenter.php` | 20 (`dataTableLayout`), 103-108 (`assigned_to`), 174-179 (`purchase_cost`), 181-186 (`book_value`) |
| 多态格式化器 | `resources/views/partials/bootstrap-table.blade.php` | 1710 (`polymorphicItemFormatter`) |
| 自定义字段格式化器 | `resources/views/partials/bootstrap-table.blade.php` | 1928 (`customFieldsFormatter`) |
| Bootstrap Table 初始化 | `resources/views/partials/bootstrap-table.blade.php` | 478 (destroy + each), 518 (bootstrapTable init), 557 (pagination) |
| 公司 Scope | `app/Models/CompanyableScope.php` | 16 |
| 公司 Scope 逻辑 | `app/Models/Company.php` | 289 (`scopeCompanyables`), 305 (`scopeCompanyablesDirectly`) |
| 权限基类 | `app/Policies/SnipePermissionsPolicy.php` | 38 (`before`), 80 (`index`) |
| 资产权限类 | `app/Policies/AssetPolicy.php` | 8 |
| 资产列表视图 | `resources/views/hardware/index.blade.php` | 58 (`<x-table.assets>` 组件) |

---

## 9. 设计意图与架构思考

### 9.1 分层设计的权衡

| 层次 | 设计目标 | 优点 | 缺点 |
|------|---------|------|------|
| **API 层** | 完整性、机器可读性、向后兼容 | 第三方集成方便，字段齐全 | 数据量大，部分字段前端不用 |
| **Presenter 层** | 可配置性、列权限控制 | 后端统一控制列显示，用户可自定义 | visible 属性存在隐式默认值（未设置即为显示） |
| **Formatter 层** | 用户体验、交互友好 | 数据格式化为最易读形式，减轻后端压力 | 部分逻辑与 Transformer 重复 |
| **Scope 层** | 数据安全、深度防御 | 查询自动过滤，无法绕过 | 调试困难，SQL 变得复杂 |
| **Policy 层** | 动作权限、灵活授权 | 统一权限入口，支持细粒度控制 | 多层检查有性能开销 |

### 9.2 潜在问题

1. **Presenter `visible` 隐式默认值**: `purchase_cost` 和 `book_value` 未显式设置 `visible` 属性，依赖 Bootstrap Table 默认值 `true`。如果后续 Bootstrap Table 版本变更默认行为，这两列可能意外隐藏。

2. **白名单维护成本**: `$allowed_columns` 需要手动维护，新增字段时容易遗漏，导致排序功能失效。

3. **分页字段冗余**: `current_page`、`prev_page_url`、`next_page_url` 对 Bootstrap Table 没有实际作用，仅对纯 API 消费者有用。

4. **排序关联字段**: `model`、`status` 等关联字段排序需要 `join` 多张表，性能开销较大。

### 9.3 优化建议

1. **显式设置所有列的 `visible` 属性**: 避免依赖框架默认值，提高代码可读性。

2. **白名单自动生成**: 从模型 `$fillable` 属性或数据库 schema 自动生成 `$allowed_columns`，减少手动维护。

3. **分页结构可选**: 通过请求参数控制是否返回 `prev_page_url` 等冗余字段。

4. **关联排序优化**: 考虑使用子查询或预先计算的排序列优化关联字段排序性能。
