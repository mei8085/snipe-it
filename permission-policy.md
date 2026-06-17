# 资产盘点系统权限判定机制分析

## 概述

Snipe-IT 资产盘点系统的权限判定由**权限注册表**、**策略类**、**控制器中间件**、**API 鉴权**、**Form Request 授权**与**公司隔离 Scope** 六部分协同完成，形成了一个完整的多维度权限判定链路。系统基于 Laravel 的 Authorization 系统构建，通过 Gate 门面对象进行统一调度。

除了基础的权限检查外，系统还通过：
- **Passport/PAT** 实现 API 令牌鉴权
- **Form Request authorize()** 在验证前进行权限拦截
- **CompanyableTrait + Global Scope** 实现跨公司数据隔离

## 一、权限定义：权限注册表

### 1.1 配置文件结构

所有权限定义在 [config/permissions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/config/permissions.php) 中，按业务模块分组：

```
├── Superuser     # 超级管理员
├── Admin         # 管理员
├── Import        # 导入权限
├── Reports       # 报表
├── Assets        # 资产
├── Accessories   # 配件
├── Consumables   # 消耗品
├── Licenses      # 许可证
├── Components    # 部件
├── Kits          # 套件
├── Users         # 用户
├── Models        # 资产型号
└── ...
```

### 1.2 权限命名规范

权限采用 `{模块}.{动作}` 的命名格式，例如：
- `assets.view` - 查看资产
- `assets.create` - 创建资产
- `assets.edit` - 编辑资产
- `assets.delete` - 删除资产
- `assets.checkout` - 资产借出
- `assets.checkin` - 资产归还
- `assets.audit` - 资产盘点

### 1.3 权限值含义

在 [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L482-L518) 的 `checkPermissionSection()` 方法中定义：
- `1` - 明确允许
- `-1` - 明确拒绝
- 未设置 - 继承用户组权限

## 二、策略类体系与匹配表

### 2.1 策略类继承层级

```
HandlesAuthorization (Laravel Trait)
        │
        ▼
SnipePermissionsPolicy (抽象基类)
        │
        ├── 通用方法: index(), view(), create(), update(), delete(), checkout(), files()
        │
        ▼
CheckoutablePermissionsPolicy (可借出资产基类)
        │
        ├── 扩展方法: checkout(), checkin(), manage()
        │
        ▼
AssetPolicy / AccessoryPolicy / LicensePolicy / ComponentPolicy / ConsumablePolicy
```

### 2.2 基类策略详解

#### SnipePermissionsPolicy 抽象基类

位置：[app/Policies/SnipePermissionsPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Policies/SnipePermissionsPolicy.php)

**核心方法 `before()` 钩子（[L38-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Policies/SnipePermissionsPolicy.php#L38-L73)）：

```php
public function before(User $user, $ability, $item)
{
    // 1. admin 权限全局放行（除 FMCSA 公司访问限制）
    if ($user->hasAccess('admin')) {
        return true;
    }
    
    // 2. 如果传入的是类名而非实例，跳过公司检查
    if (! $item instanceof Model) {
        return;
    }
    
    // 3. FMCSA 多公司访问控制检查
    if (! Company::isCurrentUserHasAccess($item)) {
        return false;
    }
}
```

**通用权限方法（[L80-L163](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Policies/SnipePermissionsPolicy.php#L80-L163)）：

| 策略方法 | 检查的权限 | 说明 |
|---------|-----------|------|
| `index()` | `{column}.view` | 列表访问 |
| `view()` | `{column}.view` | 详情查看 |
| `create()` | `{column}.create` | 创建 |
| `update()` | `{column}.edit` | 更新 |
| `delete()` | `{column}.delete` | 删除（含软删除检查） |
| `checkout()` | `{column}.checkout` | 借出 |
| `files()` | `{column}.files` | 文件管理 |
| `manage()` | `{column}.edit` | 管理（别名编辑） |

#### CheckoutablePermissionsPolicy 可借出基类

位置：[app/Policies/CheckoutablePermissionsPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Policies/CheckoutablePermissionsPolicy.php)

**扩展方法：

| 策略方法 | 检查的权限 | 说明 |
|---------|-----------|------|
| `checkout()` | `{column}.checkout` | 借出 |
| `checkin()` | `{column}.checkin` | 归还 |
| `manage()` | `{column}.checkin` \|\| `{column}.edit` \|\| `{column}.checkout` | 管理（三选一即可） |

### 2.3 具体策略类示例

#### AssetPolicy 资产策略

位置：[app/Policies/AssetPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Policies/AssetPolicy.php)

```php
class AssetPolicy extends CheckoutablePermissionsPolicy
{
    protected function columnName() { return 'assets'; }
    
    public function viewRequestable(User $user, ?Asset $asset = null)
    {
        return $user->hasAccess('assets.view.requestable');
    }
    
    public function audit(User $user, ?Asset $asset = null)
    {
        return $user->hasAccess('assets.audit');
    }
    
    public function files(User $user, $item = null)
    {
        return $user->hasAccess($this->columnName().'.files');
    }
}
```

#### UserPolicy 用户策略

位置：[app/Policies/UserPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Policies/UserPolicy.php)

```php
class UserPolicy extends SnipePermissionsPolicy
{
    protected function columnName() { return 'users'; }
}
```

### 2.4 策略类与模型绑定

在 [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Providers/AuthServiceProvider.php#L62-L83) 中注册策略映射：

```php
protected $policies = [
    Asset::class => AssetPolicy::class,
    User::class => UserPolicy::class,
    Accessory::class => AccessoryPolicy::class,
    // ... 其他模型与策略的映射
];
```

## 三、Gate 注册与全局权限检查

### 3.1 AuthServiceProvider 启动流程

位置：[app/Providers/AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Providers/AuthServiceProvider.php)

#### 全局 `before()` 钩子（[L112-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Providers/AuthServiceProvider.php#L112-L122)）：

```php
Gate::before(function ($user, $ability) {
    // Demo 模式锁定检查
    if (($ability == 'editableOnDemo') && (config('app.lock_passwords')) {
        return false;
    }
    // 超级用户全局放行
    if ($user->isSuperUser()) {
        return true;
    }
});
```

> **关键点**：这个 `before()` 钩子优先级最高，超级用户在此处被授予所有权限。

#### 通用 Gate 定义（[L130-L275](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Providers/AuthServiceProvider.php#L130-L275)）：

| Gate 名称 | 检查逻辑 | 用途 |
|---------|---------|------|
| `admin` | `$user->hasAccess('admin')` | 管理员权限 |
| `import` | `$user->hasAccess('import')` | CSV 导入 |
| `reports.view` | `$user->hasAccess('reports.view')` | 报表查看 |
| `activity.view` | `reports.view` \|\| `admin` | 活动日志 |
| `self.two_factor` | `self.two_factor` \|\| `admin` | 双因素认证 |
| `backend.interact` | 多个 view 权限的或运算 | 左侧导航显示 |
| `view.selectlists` | 多权限组合 | API 下拉列表 |

### 3.2 User 模型权限检查方法

位置：[app/Models/User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php)

#### `hasAccess()` 方法（[L532-L539](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L532-L539)）：

```php
public function hasAccess($section)
{
    // 超级用户直接放行
    if ($this->isSuperUser()) {
        return true;
    }
    // 检查具体权限
    return $this->checkPermissionSection($section);
}
```

#### `checkPermissionSection()` 方法（[L482-L518](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L482-L518)）：

权限判定优先级：
1. **用户个人权限（`1` 允许，`-1` 拒绝）
2. **用户所属组权限**（任一用户组中任一允许即允许）
3. **默认拒绝**

## 四、中间件链路

### 4.1 中间件注册

位置：[app/Http/Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Kernel.php#L102-L111)

```php
protected $routeMiddleware = [
    'auth' => Authenticate::class,
    'authorize' => CheckPermissions::class,  // 权限检查中间件
    'can' => Authorize::class,             // Laravel 原生权限中间件
    // ...
];
```

### 4.2 CheckPermissions 中间件实现

位置：[app/Http/Middleware/CheckPermissions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Middleware/CheckPermissions.php)

```php
public function handle($request, Closure $next, $section = null)
{
    // 直接调用 Gate::allows()
    if (Gate::allows($section)) {
        return $next($request);
    }
    // 权限不足返回 403
    return response()->view('layouts/basic', [
        'content' => view('errors/403'),
    ], 403);
}
```

### 4.3 路由中使用中间件

**Web 路由示例（[routes/web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/routes/web.php#L156)）：

```php
Route::group(['prefix' => 'admin', 'middleware' => ['auth', 'authorize:superuser']], function () {
    // 仅超级用户可访问
    Route::get('settings', ...);
});
```

**API 路由示例（[routes/api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/routes/api.php#L949)）：

```php
Route::group(['middleware' => ['auth', 'authorize:superuser'], 'prefix' => 'settings'], function () {
    // 设置相关 API
});
```

**SCIM 路由示例（[routes/scim.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/routes/scim.php#L19)）：

```php
Route::middleware(['auth:api','authorize:superadmin'])->group(function () {
    // SCIM 接口
});
```

## 五、控制器中的权限检查

### 5.1 控制器构造函数中间件

位置：[app/Http/Controllers/Assets/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Controllers/Assets/AssetsController.php#L51-L55)

```php
public function __construct()
{
    $this->middleware('auth');
    parent::__construct();
}
```

### 5.2 控制器方法内权限检查

**方式一：`$this->authorize()` 配合策略类

位置：[app/Http/Controllers/Assets/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Controllers/Assets/AssetsController.php)

```php
// 检查列表权限
public function index(Request $request): View
{
    $this->authorize('index', Asset::class);  // 调用 AssetPolicy@index
    // ...
}

// 检查创建权限
public function create(Request $request): View
{
    $this->authorize('create', Asset::class);  // 调用 AssetPolicy@create
    // ...
}

// 检查更新权限（传入具体资产实例）
public function edit(Asset $asset): View|RedirectResponse
{
    $this->authorize($asset);  // 自动推断为 'update' 能力
    // ...
}

// 检查查看权限
public function show(Asset $asset): View|RedirectResponse
{
    $this->authorize('view', $asset);  // 调用 AssetPolicy@view
    // ...
}
```

**方式二：checkout/checkin 权限检查

位置：[app/Http/Controllers/Assets/AssetCheckoutController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Controllers/Assets/AssetCheckoutController.php#L36)

```php
public function create(Asset $asset): View|RedirectResponse
{
    $this->authorize('checkout', $asset);  // 调用 CheckoutablePermissionsPolicy@checkout
    // ...
}
```

位置：[app/Http/Controllers/Assets/AssetCheckinController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Controllers/Assets/AssetCheckinController.php#L36)

```php
public function create(Asset $asset, $backto = null): View|RedirectResponse
{
    $this->authorize('checkin', $asset);  // 调用 CheckoutablePermissionsPolicy@checkin
    // ...
}
```

**方式三：Gate 门面直接调用

```php
use Illuminate\Support\Facades\Gate;

if (Gate::allows('admin')) {
    // 管理员逻辑
}

if (Gate::denies('assets.audit')) {
    abort(403);
}
```

## 六、完整调用链路

### 6.1 路由中间件链路（`authorize:superuser`）

```
HTTP Request
    │
    ▼
[Kernel 中间件堆栈
    │
    ▼
CheckPermissions::handle($request, $next, 'superuser')
    │
    └── Gate::allows('superuser')
            │
            ├── AuthServiceProvider@before
            │     └── $user->isSuperUser() ── true/false
            │
            └── Gate 定义匹配 'superuser'
                  └── User::hasAccess('superuser')
                        └── User::checkPermissionSection('superuser')
    │
    ├── 权限通过 → $next($request)
    │
    └── 权限拒绝 → 返回 403 视图
```

### 6.2 控制器 `$this->authorize('checkout', $asset) 链路

```
控制器方法调用 $this->authorize('checkout', $asset)
    │
    ▼
Laravel AuthorizesRequests trait
    │
    ▼
Gate::authorize('checkout', $asset)
    │
    ├── AuthServiceProvider@before 全局钩子
    │     ├── Demo 模式检查
    │     └── $user->isSuperUser() → true 直接放行
    │
    ├── 策略类解析（通过 $policies 映射找到 AssetPolicy
    │
    ├── AssetPolicy@before 钩子
    │     ├── $user->hasAccess('admin') → true 直接放行
    │     ├── 检查 $item 是否为 Model 实例
    │     └── Company::isCurrentUserHasAccess($asset) → false 直接拒绝
    │
    └── AssetPolicy@checkout() 方法
          └── $user->hasAccess('assets.checkout')
                ├── $user->isSuperUser()
                └── User::checkPermissionSection('assets.checkout')
                      ├── 用户个人权限检查（1/-1）
                      └── 用户组权限遍历
```

### 6.3 权限判定优先级总览

```
Gate::allows($ability, $arguments)
    │
    ├─ AuthServiceProvider@before
    │     └─ isSuperUser() → true
    │
    ├─ Policy@before
    │     ├─ hasAccess('admin') → true
    │     ├─ 公司访问检查 → false
    │
    └─ Policy@method()
          └─ hasAccess('{module}.{action}')
                ├─ isSuperUser() → true
                ├─ 用户权限 = '1' → true
                ├─ 用户权限 = '-1' → false
                └─ 遍历用户组
                      └─ 组权限 = '1' → true
```

## 七、关键代码速查表

### 7.1 资产相关权限与策略方法对应

| 权限字符串 | 策略方法 | 所在类 |
|-----------|---------|--------|
| `assets.view` | `view()`, `index()` | SnipePermissionsPolicy |
| `assets.create` | `create()` | SnipePermissionsPolicy |
| `assets.edit` | `update()`, `manage()` | SnipePermissionsPolicy |
| `assets.delete` | `delete()` | SnipePermissionsPolicy |
| `assets.checkout` | `checkout()` | CheckoutablePermissionsPolicy |
| `assets.checkin` | `checkin()` | CheckoutablePermissionsPolicy |
| `assets.audit` | `audit()` | AssetPolicy |
| `assets.files` | `files()` | AssetPolicy |
| `assets.view.requestable` | `viewRequestable()` | AssetPolicy |

### 7.2 权限层级

| 层级 | 检查点 | 位置 |
|-----|--------|------|
| 1 | 超级用户全局放行 | AuthServiceProvider@before |
| 2 | admin 全局放行（受公司限制） | SnipePermissionsPolicy@before |
| 3 | 多公司访问控制 | SnipePermissionsPolicy@before |
| 4 | 具体权限检查 | 各 Policy 方法 |
| 5 | 用户个人权限 | User::checkPermissionSection |
| 6 | 用户组权限继承 | User::checkPermissionSection |

## 八、设计特点

1. **分层授权**：通过全局 Gate 层层递进，超级用户 → admin → 具体权限，结构清晰
2. **策略复用**：抽象基类提供通用实现，具体策略类极简，符合开闭原则
3. **灵活扩展**：通过 `columnName()` 方法实现权限前缀动态绑定
4. **双重检查**：用户个人权限优先级高于组权限，支持精确授权
5. **多维度控制**：同时支持路由中间件和控制器内 authorize 两种保护
