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

### 2.2 AssetPolicy 实际的权限入口写法

**真实的继承链与方法分布**：

```
AssetPolicy (app/Policies/AssetPolicy.php:8)
    ↓ extends
CheckoutablePermissionsPolicy (app/Policies/CheckoutablePermissionsPolicy.php:7)
    ↓ extends
SnipePermissionsPolicy (app/Policies/SnipePermissionsPolicy.php:25)
```

**AssetPolicy 真实代码** (`app/Policies/AssetPolicy.php:8-28`):
```php
class AssetPolicy extends CheckoutablePermissionsPolicy
{
    // 【关键】不是 $section 属性，而是 columnName() 方法
    protected function columnName()
    {
        return 'assets';
    }

    // 资产特有的权限方法
    public function viewRequestable(User $user, ?Asset $asset = null)
    {
        return $user->hasAccess('assets.view.requestable');
    }

    public function audit(User $user, ?Asset $asset = null)
    {
        return $user->hasAccess('assets.audit');
    }

    // 覆盖父类的 files 方法
    public function files(User $user, $item = null)
    {
        return $user->hasAccess($this->columnName().'.files');
    }
}
```

**CheckoutablePermissionsPolicy 提供的方法** (`app/Policies/CheckoutablePermissionsPolicy.php:15-42`):
```php
abstract class CheckoutablePermissionsPolicy extends SnipePermissionsPolicy
{
    public function checkout(User $user, $item = null)
    {
        return $user->hasAccess($this->columnName().'.checkout');
    }

    public function checkin(User $user, $item = null)
    {
        return $user->hasAccess($this->columnName().'.checkin');
    }

    // 覆盖父类 manage 方法，包含 checkout/checkin/edit
    public function manage(User $user, $item = null)
    {
        return $user->hasAccess($this->columnName().'.checkin')
             || $user->hasAccess($this->columnName().'.edit')
             || $user->hasAccess($this->columnName().'.checkout');
    }
}
```

**SnipePermissionsPolicy 提供的通用方法** (`app/Policies/SnipePermissionsPolicy.php:80-163`):
```php
abstract class SnipePermissionsPolicy
{
    abstract protected function columnName();  // 子类必须实现

    // 前置检查：管理员跳过、公司权限检查
    public function before(User $user, $ability, $item) { ... }

    public function index(User $user)        { return $user->hasAccess($this->columnName().'.view'); }
    public function view(User $user, $item = null) { return $user->hasAccess($this->columnName().'.view'); }
    public function create(User $user)       { return $user->hasAccess($this->columnName().'.create'); }
    public function update(User $user, $item = null) { return $user->hasAccess($this->columnName().'.edit'); }
    public function delete(User $user, $item = null) { ... }
    public function history(User $user, $item = null) { ... }
    public function journal(User $user, $item = null) { ... }
    public function files(User $user, $item = null) { return $user->hasAccess($this->columnName().'.files'); }
    public function manage(User $user, $item = null) { return $user->hasAccess($this->columnName().'.edit'); }
}
```

### 2.3 权限衔接机制

| 层次 | 网页入口 | API 接口 | 衔接方式 |
|------|---------|---------|---------|
| **认证层** | `$this->middleware('auth')` [构造函数第53行] | Laravel API 中间件（auth:sanctum 或 session） | 共享同一用户身份 |
| **动作权限层** | `$this->authorize('index', Asset::class)` | `$this->authorize('index', Asset::class)` | 调用同一 Policy 方法链，最终检查 `assets.view` 权限 |
| **数据权限层** | 视图渲染时无数据查询 | 自动应用 `CompanyableScope` | 查询时自动过滤所属公司 |
| **字段权限层** | 无（由前端列配置控制） | Transformer 内部加密字段检查 | 最终由 Transformer 统一控制 |

### 2.4 Policy 权限校验完整流程

```
$this->authorize('index', Asset::class)
    ↓
Gate::resolvePolicy(Asset::class)
    ↓
AssetPolicy@before(User $user, 'index', Asset::class)  [SnipePermissionsPolicy.php:38]
    ↓ 检查1：管理员？
    → 是 → return true （但仍受公司 Scope 限制）
    ↓ 检查2：是否为模型实例？
    → 否（Asset::class 是类名）→ return null（继续执行后续方法）
    ↓
AssetPolicy@index(User $user)  [继承自 SnipePermissionsPolicy.php:80]
    ↓
return $user->hasAccess('assets.view');  // columnName() 返回 'assets'
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
    // DO NOT ADD ANYTHING TO THIS LIST WITHOUT CHECKING THE ORDERING SWITCH BELOW!
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
    case 'model':           $assets->OrderModels($order);              break;
    case 'category':        $assets->OrderCategory($order);            break;
    case 'manufacturer':    $assets->OrderManufacturer($order);        break;
    case 'company':         $assets->OrderCompany($order);             break;
    case 'location':        $assets->OrderLocation($order);            break;
    case 'rtd_location':    $assets->OrderRtdLocation($order);         break;
    case 'status':          $assets->OrderStatus($order);              break;
    case 'supplier':        $assets->OrderSupplier($order);            break;
    case 'assigned_to':     $assets->OrderAssigned($order);            break;
    case 'jobtitle':        $assets->OrderByJobTitle($order);          break;
    case 'created_by':      $assets->OrderByCreatedByName($order);     break;
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

1. **直接影响**: 只有在 `$allowed_columns` 中的字段才能作为排序参数使用，非法字段会被静默替换为 `assets.created_at`

2. **间接影响**: 
   - 白名单包含 `model`、`status` 等关联字段名称，但这些不是真实的数据库列
   - 需要通过 `switch` 语句转换为 `join` + `orderBy` 的复杂查询
   - 自定义字段通过动态添加到白名单，支持按自定义字段排序

3. **安全意义**: 防止 SQL 注入，用户无法通过构造 `sort=(select password from users...)` 进行攻击

---

## 4. assigned_to 在 Transformer 与 Formatter 之间的真实字段使用关系

### 4.1 Transformer 层输出：transformAssignedTo()

**文件位置**: `app/Http/Transformers/AssetsTransformer.php:215-236`

```php
public function transformAssignedTo($asset)
{
    // 分配给用户时 - 输出完整字段
    if ($asset->checkedOutToUser()) {
        return $asset->assigned ? [
            'id' => (int) $asset->assigned->id,
            'username' => e($asset->assigned->username),
            'name' => e($asset->assigned->display_name),      // 【关键】display_name，不是 getFullNameAttribute()
            'first_name' => e($asset->assigned->first_name),
            'last_name' => ($asset->assigned->last_name) ? e($asset->assigned->last_name) : null,
            'email' => ($asset->assigned->email) ? e($asset->assigned->email) : null,
            'employee_number' => ($asset->assigned->employee_num) ? e($asset->assigned->employee_num) : null,
            'jobtitle' => $asset->assigned->jobtitle ? e($asset->assigned->jobtitle) : null,
            'type' => 'user',
        ] : null;
    }

    // 分配给其他类型（资产/位置/配件等）- 输出精简字段
    return $asset->assigned ? [
        'id' => $asset->assigned->id,
        'name' => e($asset->assigned->display_name),  // 【关键】都是 display_name
        'type' => $asset->assignedType(),              // 由 assignedType() 方法决定类型字符串
    ] : null;
}
```

**API 输出格式示例**:
```json
// 分配给用户
"assigned_to": {
    "id": 42,
    "username": "zhangsan",
    "name": "张三",
    "first_name": "张三",
    "last_name": null,
    "email": "zhangsan@example.com",
    "employee_number": "EMP001",
    "jobtitle": "软件工程师",
    "type": "user"
}

// 分配给位置
"assigned_to": {
    "id": 12,
    "name": "北京办公室-3楼-会议室A",
    "type": "location"
}

// 分配给资产
"assigned_to": {
    "id": 156,
    "name": "服务器机架 A1",
    "type": "asset"
}
```

### 4.2 Formatter 层处理：polymorphicItemFormatter()

**文件位置**: `resources/views/partials/bootstrap-table.blade.php:1710-1763`

```javascript
function polymorphicItemFormatter(value) {
    var item_destination = '';
    var item_icon;

    if ((value) && (value.type)) {

        // 【步骤1】根据 type 字段决定链接目标和图标
        if (value.type == 'asset') {
            item_destination = 'hardware';
            item_icon = 'fas fa-barcode';
        } else if (value.type == 'accessory') {
            item_destination = 'accessories';
            item_icon = 'far fa-keyboard';
        } else if (value.type == 'component') {
            item_destination = 'components';
            item_icon = 'far fa-hdd';
        } else if (value.type == 'consumable') {
            item_destination = 'consumables';
            item_icon = 'fas fa-tint';
        } else if (value.type == 'license') {
            item_destination = 'licenses';
            item_icon = 'far fa-save';
        } else if (value.type == 'user') {
            item_destination = 'users';
            item_icon = 'fas fa-user';
        } else if (value.type == 'location') {
            item_destination = 'locations';
            item_icon = 'fas fa-map-marker-alt';
        } else if (value.type == 'maintenance') {
            item_destination = 'maintenances';
            item_icon = 'fa-solid fa-screwdriver-wrench';
        } else if (value.type == 'model') {
            item_destination = 'models';
            item_icon = '';
        }

        // 【步骤2】用户类型特殊处理：追加用户名显示
        // display the username if it's checked out to a user, but don't do it if the username's there already
        if (value.username && !value.name.match('\\(') && !value.name.match('\\)')) {
            value.name = value.name + ' (' + value.username + ')';
        }
        // 例："张三" → "张三 (zhangsan)"

        // 【步骤3】已删除项显示删除线
        // Show as strikethrough if it's been deleted
        if (value.deleted_at && value.deleted_at != '') {
            return '<nobr><span class="text-muted" data-tooltip="true" title="{{ trans('general.deleted') }} ' + value.type + '"><del><i class="' + item_icon + ' fa-fw"></i> ' + value.name + '</del></span></nobr>';
        }

        // 【步骤4】正常显示：图标 + 名称链接
        return '<nobr><a href="{{ config('app.url') }}/' + item_destination +'/' + value.id + '" data-tooltip="true" title="' + value.type + '"><i class="' + item_icon + ' fa-fw"></i> ' + value.name + '</a></nobr>';

    } else {
        return '';
    }
}
```

### 4.3 字段使用关系对照表

| Transformer 输出字段 | Formatter 是否使用 | 使用方式 |
|---------------------|-------------------|---------|
| `id` | ✅ 是 | 构造链接 URL |
| `type` | ✅ 是 | 决定链接目标路径、图标、tooltip 文本 |
| `name` | ✅ 是 | 显示文本基础，用户类型会追加用户名 |
| `username` | ✅ 是（仅用户类型） | 追加到 name 后：`name + ' (' + username + ')'` |
| `first_name` | ❌ 否 | 完全不使用 |
| `last_name` | ❌ 否 | 完全不使用 |
| `email` | ❌ 否 | 完全不使用 |
| `employee_number` | ❌ 否 | 完全不使用 |
| `jobtitle` | ❌ 否 | 完全不使用 |
| `deleted_at` | ⚠️ 部分使用 | Formatter 有检查逻辑，但资产列表场景下 Transformer 不输出此字段 |

> **重要发现**：Transformer 为用户类型输出了 9 个字段，但 Formatter 只使用了 `id`、`type`、`name`、`username` 这 4 个字段，其余 5 个字段（`first_name`、`last_name`、`email`、`employee_number`、`jobtitle`）在网页表格中完全浪费。

### 4.4 deleted_at 在 assigned_to 渲染链路中的完整处理逻辑

#### 4.4.1 deleted_at 的携带位置

**模型关系层** - `deleted_at` 的获取 (`app/Models/Asset.php:734-736`):
```php
public function assignedTo()
{
    // 【关键】使用 withTrashed()，关联对象包含 deleted_at 属性
    return $this->morphTo('assigned', 'assigned_type', 'assigned_to')->withTrashed();
}
```

**重要说明**:
- `deleted_at` 是**分配目标对象**（用户/位置/资产等）的属性，不是当前资产的属性
- `->withTrashed()` 确保即使分配目标已被软删除，也能查询到关联对象
- 原始 Eloquent 模型对象 `$asset->assigned` 确实包含 `deleted_at` 属性

#### 4.4.2 deleted_at 在哪一步进入（或未进入）Formatter

**场景 A：资产列表 assigned_to 字段 - 数据丢失**

```
资产模型 assignedTo() 关系
    ↓ withTrashed() → 关联对象包含 deleted_at 属性
    ↓
AssetsTransformer@transformAssignedTo()  [第215-236行]
    ↓ ❌ 【BUG 点】没有提取 deleted_at 到输出数组
    ↓ 无论是用户类型还是其他类型，都没有输出 deleted_at
    ↓
API 响应 assigned_to 字段（无 deleted_at）
    ↓
Bootstrap Table 接收数据
    ↓
polymorphicItemFormatter(value)  [第1752行]
    ↓ value.deleted_at 是 undefined
    ↓ ❌ 删除线逻辑永远不触发
    ↓
正常显示（没有删除线）
```

**Transformer 代码事实** (`app/Http/Transformers/AssetsTransformer.php:215-236`):
```php
public function transformAssignedTo($asset)
{
    if ($asset->checkedOutToUser()) {
        return $asset->assigned ? [
            'id' => (int) $asset->assigned->id,
            'username' => e($asset->assigned->username),
            'name' => e($asset->assigned->display_name),
            'first_name' => e($asset->assigned->first_name),
            'last_name' => ($asset->assigned->last_name) ? e($asset->assigned->last_name) : null,
            'email' => ($asset->assigned->email) ? e($asset->assigned->email) : null,
            'employee_number' => ($asset->assigned->employee_num) ? e($asset->assigned->employee_num) : null,
            'jobtitle' => $asset->assigned->jobtitle ? e($asset->assigned->jobtitle) : null,
            'type' => 'user',
            // ❌ 缺失：没有 'deleted_at' 字段
        ] : null;
    }

    return $asset->assigned ? [
        'id' => $asset->assigned->id,
        'name' => e($asset->assigned->display_name),
        'type' => $asset->assignedType(),
        // ❌ 缺失：没有 'deleted_at' 字段
    ] : null;
}
```

**场景 B：组件检出列表 name 字段 - 数据完整**

```
ComponentCheckout 模型 component 关系
    ↓ withTrashed() → 组件对象包含 deleted_at 属性
    ↓
AssetsTransformer@transformCheckedoutComponents()  [第405-409行]
    ↓ ✅ 正确输出 deleted_at
    ↓
API 响应 name 字段（包含 deleted_at）
    ↓
Bootstrap Table 接收数据
    ↓
polymorphicItemFormatter(value)  [第1752行]
    ↓ value.deleted_at 存在（如果组件已删除）
    ↓ ✅ 删除线逻辑正常触发
    ↓
显示为删除线样式
```

**Transformer 代码事实** (`app/Http/Transformers/AssetsTransformer.php:405-409`):
```php
'name' => [
    'id' => $component_checkout->component?->id,
    'name' => e($component_checkout->component?->display_name),
    'type' => 'component',
    'deleted_at' => $component_checkout->component?->deleted_at,  // ✅ 有输出！
],
```

**场景 C：配件检出 assigned_to 字段 - 数据丢失**

配件检出使用不同的 Transformer 方法链，同样缺失 `deleted_at`:
```php
// AccessoriesTransformer.php:117-129
public function transformAssignedTo($accessoryCheckout)
{
    if ($accessoryCheckout->checkedOutToUser()) {
        return (new UsersTransformer)->transformUserCompact($accessoryCheckout->assigned);
        // ❌ transformUserCompact 不输出 deleted_at
    } elseif ($accessoryCheckout->checkedOutToLocation()) {
        return (new LocationsTransformer)->transformLocationCompact($accessoryCheckout->assigned);
        // ❌ transformLocationCompact 不输出 deleted_at
    } elseif ($accessoryCheckout->checkedOutToAsset()) {
        return (new AssetsTransformer)->transformAssetCompact($accessoryCheckout->assigned);
        // ❌ transformAssetCompact 不输出 deleted_at
    }
}
```

#### 4.4.3 Formatter 层的检查逻辑

**文件位置**: `resources/views/partials/bootstrap-table.blade.php:1752-1753`

```javascript
// Show as strikethrough if it's been deleted
if (value.deleted_at && value.deleted_at != '') {
    return '<nobr><span class="text-muted" data-tooltip="true" title="{{ trans('general.deleted') }} ' + value.type + '"><del><i class="' + item_icon + ' fa-fw"></i> ' + value.name + '</del></span></nobr>';
}
```

#### 4.4.4 现有描述与真实代码路径的差异

| 现有描述 | 真实代码事实 |
|---------|-------------|
| 「`deleted_at` 字段完全不使用」 | 错误。Formatter 有检查逻辑，但 Transformer 在资产列表场景下没有输出该字段 |
| 「`polymorphicItemFormatter` 会检查此状态显示删除线」 | 部分正确。代码中有这个逻辑，但在资产列表场景下由于数据缺失永远不会执行 |
| 暗示 `deleted_at` 由资产对象携带 | 错误。`deleted_at` 是**分配目标对象**（用户/位置/资产等）的属性 |
| 没有说明场景差异 | 实际上不同场景表现不同：<br>• 资产列表 assigned_to：❌ 无 deleted_at<br>• 组件检出 name：✅ 有 deleted_at<br>• 配件检出 assigned_to：❌ 无 deleted_at |

#### 4.4.5 跨场景一致性对比表

| 场景 | Presenter | Transformer 方法 | 是否输出 deleted_at | Formatter 删除线是否可用 |
|------|-----------|-----------------|---------------------|-------------------------|
| 资产列表 | `AssetPresenter.php:108` | `transformAssignedTo()` | ❌ 否 | ❌ 不可用 |
| 组件检出 | `ComponentPresenter.php:215` | `transformCheckedoutComponents()` | ✅ 是 | ✅ 可用 |
| 配件检出 | `AccessoryPresenter.php:219` | `transformAssignedTo()` | ❌ 否 | ❌ 不可用 |
| 维护记录 | `MaintenancesPresenter.php:97` | 取决于维护 Transformer | 待确认 | 待确认 |
| 历史记录 | `HistoryPresenter.php:100` | 取决于历史 Transformer | 待确认 | 待确认 |
| 资产审计 | `AssetAuditPresenter.php:103` | 取决于审计 Transformer | 待确认 | 待确认 |

### 4.4 最终显示效果

| 分配类型 | API 输出字段数 | 表格实际显示 | 显示格式 |
|---------|--------------|-------------|---------|
| **用户** | 9 个字段 | 图标 + "张三 (zhangsan)" 链接 | `<i class="fas fa-user"></i> 张三 (zhangsan)` |
| **位置** | 3 个字段 | 图标 + "北京办公室" 链接 | `<i class="fas fa-map-marker-alt"></i> 北京办公室` |
| **资产** | 3 个字段 | 图标 + "服务器机架 A1" 链接 | `<i class="fas fa-barcode"></i> 服务器机架 A1` |
| **未分配** | null | 空字符串 | `''` |

---

## 5. 净值列与采购成本列默认显示状态的真实行为

### 5.1 Presenter 配置对比

**文件位置**: `app/Presenters/AssetPresenter.php:174-186`

```php
// 采购成本列 - 第174-179行
[
    'field' => 'purchase_cost',
    'searchable' => true,
    'sortable' => true,
    'title' => trans('general.purchase_cost'),
    'footerFormatter' => 'sumFormatter',
    'class' => 'text-right',
    // 【关键】未设置 visible 属性
],

// 净值列 - 第181-186行
[
    'field' => 'book_value',
    'searchable' => false,
    'sortable' => false,
    'title' => trans('admin/hardware/table.book_value'),
    'footerFormatter' => 'sumFormatter',
    'class' => 'text-right',
    // 【关键】未设置 visible 属性
],
```

### 5.2 配置项对比表

| 字段 | 配置项 | purchase_cost | book_value |
|------|--------|---------------|------------|
| `field` | | `'purchase_cost'` | `'book_value'` |
| `searchable` | | `true` | `false` |
| `sortable` | | `true` | `false` |
| `title` | | `trans('general.purchase_cost')` | `trans('admin/hardware/table.book_value')` |
| `footerFormatter` | | `'sumFormatter'` | `'sumFormatter'` |
| `class` | | `'text-right'` | `'text-right'` |
| **`visible`** | | **未设置** | **未设置** |

### 5.3 真实行为分析

1. **Bootstrap Table 默认行为**: `visible` 属性默认值为 `true`
2. **两列均未显式设置 `visible` 属性**，因此**默认都显示**
3. **功能差异**:
   - `purchase_cost`: 可搜索、可排序
   - `book_value`: 不可搜索、不可排序
4. **共同点**:
   - 都支持页脚求和 (`footerFormatter: 'sumFormatter'`)
   - 都右对齐 (`class: 'text-right'`)
   - 都由 API 格式化为货币字符串（如 "$1,299.00"）

### 5.4 对比其他明确隐藏的列

```php
// model_number - 第82-87行
[
    'field' => 'model_number',
    'searchable' => true,
    'sortable' => true,
    'title' => trans('admin/models/table.modelnumber'),
    'visible' => false,  // 【明确】显式隐藏
],

// company - 第52-58行
[
    'field' => 'company',
    'searchable' => true,
    'sortable' => true,
    'switchable' => true,
    'title' => trans('general.company'),
    'visible' => false,  // 【明确】显式隐藏
    'formatter' => 'companiesLinkObjFormatter',
],
```

> **结论**: `purchase_cost` 和 `book_value` 由于未设置 `visible: false`，在网页表格中默认显示。这是代码事实，而非文档或直觉声称的「默认隐藏」。

---

## 6. 分页外层结构在表格消费端的作用

### 6.1 分页外层结构定义

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

**实际调用位置** (`app/Http/Transformers/AssetsTransformer.php:27`):
```php
public function transformAssets(Collection $assets, $total)
{
    $array = [];
    foreach ($assets as $asset) {
        $array[] = self::transformAsset($asset);
    }

    // 【注意】只有两个参数，没有第三个 $request 参数
    return (new DatatablesTransformer)->transformDatatables($array, $total);
}
```

### 6.2 完整响应结构示例

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

### 6.3 Bootstrap Table 服务端分页配置

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
    
    sidePagination: 'server',    // 【关键】服务端分页模式
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

### 6.4 各字段在表格消费端的具体作用

| 字段 | Bootstrap Table 使用情况 | 说明 |
|------|-------------------------|------|
| **`total`** | ✅ 核心字段 | Bootstrap Table 内部 `options.totalField = 'total'`（默认值），用于显示总记录数、计算总页数、控制分页按钮状态 |
| **`rows`** | ✅ 核心字段 | Bootstrap Table 内部 `options.dataField = 'rows'`（默认值），表格行数据数组 |
| **`current_page`** | ❌ 不直接使用 | Bootstrap Table 通过 `offset`/`limit` 自行维护当前页状态，此字段主要供纯 API 消费者使用 |
| **`per_page`** | ⚠️ 间接对应 | 对应 Bootstrap Table 的 `pageSize` 选项，但由表格自己维护，不读取响应中的此字段 |
| **`total_pages`** | ⚠️ 间接对应 | 可用于 `options.totalPages`，但 Bootstrap Table 通常通过 `total` / `pageSize` 自行计算 |
| **`prev_page_url`** | ❌ 不使用 | 供 API 消费者直接调用，Bootstrap Table 自行构造请求参数 |
| **`next_page_url`** | ❌ 不使用 | 供 API 消费者直接调用，Bootstrap Table 自行构造请求参数 |

### 6.5 分页交互完整流程

```
Bootstrap Table 初始化
    ↓
用户点击「下一页」或选择每页条数
    ↓
Bootstrap Table 自动构造请求参数:
    ?offset=50&limit=50&sort=asset_tag&order=asc
    ↓
API 控制器接收参数:
    $offset = ($request->input('offset') > $assets->count()) ? $assets->count() : app('api_offset_value');  [第448行]
    $limit = app('api_limit_value');  [第449行]
    ↓
数据库查询:
    $total = $assets->count();               [第451行] - 先查总数
    $assets = $assets->skip($offset)->take($limit)->get();  [第452行] - 再查当前页
    ↓
AssetsTransformer 转换每行数据
    ↓
DatatablesTransformer 包装分页结构
    ↓
返回 JSON 响应
    ↓
Bootstrap Table 解析响应:
    1. 从 response.total 更新分页控件显示（如 "共 156 条"）
    2. 从 response.rows 渲染表格行
    3. 根据 total / pageSize 计算总页数，更新页码按钮
    4. 高亮当前页按钮，禁用首尾页按钮
```

### 6.6 分页字段冗余分析

对于 Bootstrap Table 消费端：
- **必选**: `total`, `rows`
- **可选但不用**: `current_page`, `per_page`, `total_pages`
- **完全不用**: `prev_page_url`, `next_page_url`

对于纯 API 消费者（如脚本、第三方集成）：
- **全部有用**: 所有字段都提供了便利的导航信息

> **设计权衡**: 分页结构同时服务两类消费者，导致对网页表格存在约 40% 的字段冗余，但换来了 API 接口的统一性和易用性。

---

## 7. 三层核心差异点（最终修正版）

### 7.1 Transformer 层 - API 输出规范

**文件位置**: `app/Http/Transformers/AssetsTransformer.php`

**核心方法**: `transformAsset(Asset $asset)` (第30行)

**真实输出特性**:

1. **完整字段输出** (第35-127行):
   - 基础字段: `id`, `name`, `asset_tag`, `serial`, `model`, `model_number`, `byod`, `requestable`
   - 关联对象: `status_label`(兼容), `status`, `category`, `manufacturer`, `depreciation`, `supplier`, `company`, `location`, `rtd_location`, `assigned_to`
   - 日期字段: 统一格式化为 `{ datetime: "Y-m-d H:i:s", formatted: "Y/m/d" }` 结构
   - 数值字段: `purchase_cost`, `book_value`, `checkin_counter`, `checkout_counter`, `requests_counter`
   - 其他: `image`, `qr`, `alt_barcode`, `warranty_months`, `warranty_expires`, `notes`, `order_number`, `age`, `eol`, `asset_eol_date`, `deleted_at`

2. **自定义字段处理** (第129-171行):
   ```php
   // 加密字段权限检查 (第135行)
   $value = (Gate::allows('assets.view.encrypted_custom_fields')) 
       ? $decrypted 
       : strtoupper(trans('admin/custom_fields/general.encrypted'));
   
   // 真实输出结构 - field 是 db_column
   $fields_array[$field->name] = [
       'field' => e($field->db_column),
       'value' => e($value),
       'field_format' => $field->format,
       'element' => $field->element,
   ];
   ```

3. **可用动作权限** (第173-181行) - 注意每个条件都有多余的 `? true : false`:
   ```php
   $permissions_array['available_actions'] = [
       'checkout' => ($asset->deleted_at == '' && Gate::allows('checkout', Asset::class)) ? true : false,
       'checkin' => ($asset->deleted_at == '' && Gate::allows('checkin', Asset::class)) ? true : false,
       'clone' => Gate::allows('create', Asset::class) ? true : false,
       'restore' => ($asset->deleted_at != '' && Gate::allows('create', Asset::class)) ? true : false,
       'update' => ($asset->deleted_at == '' && Gate::allows('update', Asset::class)) ? true : false,
       'audit' => Gate::allows('audit', Asset::class) ? true : false,
       'delete' => ($asset->deleted_at == '' && $asset->assigned_to == '' && Gate::allows('delete', Asset::class) && ($asset->deleted_at == '')) ? true : false,
   ];
   ```

4. **方法签名** (第20行) - 只有两个参数:
   ```php
   public function transformAssets(Collection $assets, $total)  // 没有第三个 $request 参数
   ```

### 7.2 Presenter 层 - 表格列配置

**文件位置**: `app/Presenters/AssetPresenter.php`

**核心方法**: `dataTableLayout()` (第20行)

**关键列配置事实**:

| 字段 | visible 值 | 默认状态 |
|------|-----------|---------|
| `checkbox` | 无（checkbox 列） | 显示 |
| `id` | `false` | 隐藏 |
| `asset_tag` | `true` | 显示 |
| `name` | `true` | 显示 |
| `image` | `true` | 显示 |
| `serial` | `true` | 显示 |
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

### 7.3 前端 Formatter 层 - 显示格式化

**文件位置**: `resources/views/partials/bootstrap-table.blade.php`

**核心格式化器函数（真实代码）**:

1. **`customFieldsFormatter(value, row)`** (第1928行): 按 URL/BOOLEAN/EMAIL 类型格式化
2. **`dateDisplayFormatter(value)`** (第2113行): 提取日期对象的 `formatted` 字段
3. **`polymorphicItemFormatter(value)`** (第1710行): 处理多态关联对象，用户类型追加用户名，已删除项显示删除线
4. **`imageFormatter(value, row)`** (第2191行): 渲染图片标签
5. **`hardwareActionsFormatter(value, row)`**: 根据 `available_actions` 渲染操作按钮

---

## 8. 完整字段差异对照表（最终修正版）

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
| `assigned_to` (用户) | 对象: `{id, username, name, first_name, last_name, email, employee_number, jobtitle, type}` | 仅显示: 图标 + "姓名 (用户名)" 链接 | 1. Presenter `visible: true`<br>2. Formatter `polymorphicItemFormatter` 仅使用 `id/type/name/username` 4 个字段，其余 5 个字段浪费 |
| `assigned_to` (资产/位置) | 对象: `{id, name, type}` | 图标 + 名称链接 | Formatter `polymorphicItemFormatter` |
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
| `user_can_checkout` | 布尔值 | 不直接显示，用于控制借出按钮状态 | Formatter 内部逻辑 |
| **特殊字段** | | | |
| `checkbox` | API 不输出 | 表格第一列复选框 | Presenter 新增配置列: `{checkbox: true}` |
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
| `deleted_at` | 日期对象或 null | 不单独显示列，Formatter 内部检查 | 用于 `polymorphicItemFormatter` 中已删除项的删除线显示 |

---

## 9. 关键代码位置索引（精确版）

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 网页入口控制器 | `app/Http/Controllers/Assets/AssetsController.php` | 66 (`index`), 68 (`authorize`) |
| API 控制器 | `app/Http/Controllers/Api/AssetsController.php` | 65 (`index`), 91 (`authorize`), 96-145 (`$allowed_columns`), 377-445 (排序逻辑), 451-452 (分页) |
| 资产权限策略 | `app/Policies/AssetPolicy.php` | 8 (类定义), 10-13 (`columnName()`) |
| 借出权限基类 | `app/Policies/CheckoutablePermissionsPolicy.php` | 7 (类定义), 15 (`checkout`), 26 (`checkin`), 37 (`manage`) |
| 通用权限基类 | `app/Policies/SnipePermissionsPolicy.php` | 25 (类定义), 38 (`before`), 80 (`index`), 90 (`view`), 115 (`create`), 125 (`update`), 145 (`delete`) |
| 资产 Transformer | `app/Http/Transformers/AssetsTransformer.php` | 20 (`transformAssets` - 只有2个参数), 30 (`transformAsset`), 103 (`assigned_to` 调用), 129 (自定义字段), 135 (加密检查), 173 (可用动作), 215 (`transformAssignedTo`) |
| 分页 Transformer | `app/Http/Transformers/DatatablesTransformer.php` | 10 (`transformDatatables`) |
| 表格列配置 | `app/Presenters/AssetPresenter.php` | 20 (`dataTableLayout`), 24-28 (checkbox 列), 103-108 (`assigned_to`), 174-179 (`purchase_cost`), 181-186 (`book_value`) |
| 多态格式化器 | `resources/views/partials/bootstrap-table.blade.php` | 1710 (`polymorphicItemFormatter`), 1747-1749 (追加用户名), 1752-1753 (删除线) |
| 自定义字段格式化器 | `resources/views/partials/bootstrap-table.blade.php` | 1928 (`customFieldsFormatter`) |
| Bootstrap Table 初始化 | `resources/views/partials/bootstrap-table.blade.php` | 478 (destroy + each), 518 (bootstrapTable init), 557 (pagination) |
| 公司 Scope | `app/Models/CompanyableScope.php` | 16 |
| 公司 Scope 逻辑 | `app/Models/Company.php` | 289 (`scopeCompanyables`), 305 (`scopeCompanyablesDirectly`) |
| 资产列表视图 | `resources/views/hardware/index.blade.php` | 58 (`<x-table.assets>` 组件) |

---

## 10. 已纠正的错误清单

| 原错误内容 | 正确事实 |
|-----------|---------|
| `AssetPolicy` 使用 `protected $section = 'assets';` | 实际使用 `protected function columnName() { return 'assets'; }` |
| `AssetPolicy` 包含所有权限方法定义 | 实际只有 `columnName()`、`viewRequestable()`、`audit()`、`files()`，其余继承自父类 |
| `transformAssets` 方法有三个参数 `($assets, $total, $request)` | 实际只有两个参数 `(Collection $assets, $total)` |
| `transformAssignedTo` 使用 `getFullNameAttribute()` | 实际使用 `display_name` 属性 |
| `polymorphicItemFormatter` 只显示名称 | 实际用户类型会追加用户名：`"张三 (zhangsan)"`，已删除项显示删除线 |
| `book_value` 列默认隐藏 | 实际未设置 `visible`，Bootstrap Table 默认 `true`，因此默认显示 |
| 自定义字段输出 `field` 是 `db_column_name()` | 实际是 `e($field->db_column)` |
| `available_actions` 使用简洁布尔表达式 | 实际每个条件都有多余的 `? true : false` |
| `deleted_at` 字段完全不使用 | 实际 `polymorphicItemFormatter` 会检查此状态显示删除线 |
| `assigned_to` 所有字段都会被使用 | 实际用户类型 9 个字段中只有 4 个被使用，其余 5 个浪费 |

---

## 11. 设计意图与架构思考

### 11.1 分层设计的权衡

| 层次 | 设计目标 | 优点 | 缺点 |
|------|---------|------|------|
| **API 层** | 完整性、机器可读性、向后兼容 | 第三方集成方便，字段齐全 | 数据量大，部分字段前端不用（如 assigned_to 的多余字段） |
| **Presenter 层** | 可配置性、列权限控制 | 后端统一控制列显示，用户可自定义 | visible 属性存在隐式默认值（未设置即为显示），易产生误解 |
| **Formatter 层** | 用户体验、交互友好 | 数据格式化为最易读形式，减轻后端压力 | 部分逻辑与 Transformer 重复，用户名字段重复拼接 |
| **Scope 层** | 数据安全、深度防御 | 查询自动过滤，无法绕过 | 调试困难，SQL 变得复杂 |
| **Policy 层** | 动作权限、灵活授权 | 统一权限入口，支持细粒度控制 | 多层检查有性能开销，继承链较深不易理解 |

### 11.2 已确认的潜在问题

1. **Presenter `visible` 隐式默认值**: `purchase_cost` 和 `book_value` 未显式设置 `visible` 属性，依赖 Bootstrap Table 默认值 `true`。如果后续 Bootstrap Table 版本变更默认行为，这两列可能意外隐藏。

2. **白名单维护成本**: `$allowed_columns` 需要手动维护，新增字段时容易遗漏，导致排序功能失效。

3. **分页字段冗余**: `current_page`、`prev_page_url`、`next_page_url` 对 Bootstrap Table 没有实际作用，仅对纯 API 消费者有用。

4. **`assigned_to` 字段浪费**: 用户类型分配时输出 9 个字段，但前端只使用 4 个，浪费约 55% 的带宽。

5. **排序关联字段**: `model`、`status` 等关联字段排序需要 `join` 多张表，性能开销较大。

6. **`available_actions` 冗余写法**: 每个条件都有多余的 `? true : false`，代码不够简洁。

### 11.3 优化建议

1. **显式设置所有列的 `visible` 属性**: 避免依赖框架默认值，提高代码可读性。

2. **白名单自动生成**: 从模型 `$fillable` 属性或数据库 schema 自动生成 `$allowed_columns`，减少手动维护。

3. **分页结构可选**: 通过请求参数控制是否返回 `prev_page_url` 等冗余字段。

4. **精简 `assigned_to` 输出**: 根据请求来源（API vs 网页）输出不同数量的字段，或提供 `?fields=` 参数让调用方选择。

5. **关联排序优化**: 考虑使用子查询或预先计算的排序列优化关联字段排序性能。

6. **清理 `available_actions` 代码**: 移除多余的 `? true : false` 写法。
