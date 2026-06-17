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

### 7.2 API 鉴权速查

| 组件 | 配置/代码 | 位置 |
|------|----------|------|
| API Guard 驱动 | `'driver' => 'passport'` | [config/auth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/config/auth.php#L47) |
| User Trait | `use HasApiTokens` | [app/Models/User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L43) |
| API 中间件组 | `'auth:api'` + 其他 | [app/Http/Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Kernel.php#L83) |
| v1 路由前缀 | `'prefix' => 'v1'` | [routes/api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/routes/api.php#L18) |
| Passport 过期时间 | `Passport::tokensExpireIn()` | [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Providers/AuthServiceProvider.php#L100) |

### 7.3 Form Request 授权速查

| 操作 | Request 类 | authorize() 逻辑 |
|------|-----------|-----------------|
| 创建资产 | `StoreAssetRequest` | `Gate::allows('create', Asset::class)` |
| 更新资产 | `UpdateAssetRequest` | `Gate::allows('update', $this->asset)` |
| 创建配件 | `StoreAccessoryRequest` | `Gate::allows('create', Accessory::class)` |
| 创建部件 | `StoreComponentRequest` | `Gate::allows('create', Component::class)` |
| 更新部件 | `UpdateComponentRequest` | `Gate::allows('update', $this->component)` |
| 资产借出 | `AssetCheckoutRequest` | `return true`（控制器内检查） |
| 资产归还 | `AssetCheckinRequest` | `return true`（控制器内检查） |
| 安全设置 | `StoreSecuritySettings` | `Gate::allows('superuser')` |

### 7.4 公司隔离速查

| 组件 | 位置 | 作用 |
|------|------|------|
| `CompanyableTrait` | [app/Models/Traits/CompanyableTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Traits/CompanyableTrait.php) | 自动注册全局 Scope + 提供 `canCheckoutTo()` |
| `CompanyableScope` | [app/Models/CompanyableScope.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/CompanyableScope.php) | 全局查询 Scope，委托给 Company::scopeCompanyables |
| `scopeCompanyables()` | [app/Models/Company.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Company.php#L372) | 核心过滤逻辑入口 |
| `isCurrentUserHasAccess()` | [app/Models/Company.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Company.php#L190) | 策略 before 钩子中的访问检查 |
| `null_company_is_floater` | 设置项 | 无公司数据是否对所有用户可见 |
| `full_multiple_companies_support` | 设置项 | FMCS 总开关 |

### 7.5 权限层级（扩展版）

| 层级 | 检查点 | 位置 |
|-----|--------|------|
| 1 | API 令牌认证 (Passport PAT) | 'auth:api' 中间件 |
| 2 | 超级用户全局放行 | AuthServiceProvider@before |
| 3 | 路由级权限拦截 | CheckPermissions 中间件 |
| 4 | Form Request 授权 | 各 Request 类 authorize() |
| 5 | admin 全局放行（受公司限制） | SnipePermissionsPolicy@before |
| 6 | 多公司访问控制 | SnipePermissionsPolicy@before + CompanyableScope |
| 7 | 控制器方法级权限检查 | $this->authorize() |
| 8 | 具体权限检查 | 各 Policy 方法 |
| 9 | 用户个人权限 | User::checkPermissionSection |
| 10 | 用户组权限继承 | User::checkPermissionSection |
| 11 | 数据查询自动过滤 | CompanyableScope |

## 八、设计特点

1. **分层授权**：通过全局 Gate 层层递进，超级用户 → admin → 具体权限，结构清晰
2. **策略复用**：抽象基类提供通用实现，具体策略类极简，符合开闭原则
3. **灵活扩展**：通过 `columnName()` 方法实现权限前缀动态绑定
4. **双重检查**：用户个人权限优先级高于组权限，支持精确授权
5. **多维度控制**：同时支持路由中间件和控制器内 authorize 两种保护
6. **API 与 Web 统一**：API 控制器与 Web 控制器复用相同的策略类，保持权限逻辑一致
7. **安全前置**：Form Request 的 authorize() 在验证前执行，未授权用户无法触碰验证逻辑
8. **数据层隔离**：通过 Global Scope 自动应用公司过滤，无需手动编写查询条件，从根本上防止数据泄漏
9. **配置驱动**：FMCS 功能可通过设置项开关，灵活适配单公司和多公司场景
10. **令牌管理**：完整的 PAT 生命周期管理（创建/查看/撤销），支持 API 安全访问

---

## 九、API v1 鉴权：Passport PAT 全链路

### 9.1 认证驱动与守卫配置

[config/auth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/config/auth.php#L46-L50) 定义了两个守卫：

| 守卫 | 驱动 | 用途 |
|------|------|------|
| `web` | `session` | Web 界面会话认证 |
| `api` | `passport` | API 请求令牌认证 |

当路由使用 `auth:api` 中间件时，Laravel 解析到 `passport` 驱动，由 Passport 的 `TokenGuard` 拦截请求，从 `Authorization: Bearer {token}` 头部提取令牌并验证。

### 9.2 Passport 令牌配置

[config/passport.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/config/passport.php) 定义了关键参数：

```php
return [
    'private_key' => env('PASSPORT_PRIVATE_KEY'),
    'public_key' => env('PASSPORT_PUBLIC_KEY'),
    'expiration_years' => env('API_TOKEN_EXPIRATION_YEARS', 20),
    'cookie_name' => env('PASSPORT_COOKIE_NAME', 'snipeit_passport_token'),
];
```

[AuthServiceProvider](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Providers/AuthServiceProvider.php#L99-L104) 中的过期时间注册：

```php
$expirationYears = (int) config('passport.expiration_years');
Passport::tokensExpireIn(CarbonInterval::years($expirationYears));
Passport::refreshTokensExpireIn(CarbonInterval::years($expirationYears));
Passport::personalAccessTokensExpireIn(CarbonInterval::years($expirationYears));
Passport::cookie(config('passport.cookie_name'));
```

> 默认令牌有效期为 20 年，通过 `API_TOKEN_EXPIRATION_YEARS` 环境变量可调整。

### 9.3 User 模型与 HasApiTokens

[User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L32-L43) 引入 `HasApiTokens` Trait：

```php
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use Authenticatable, Authorizable, CanResetPassword, HasApiTokens;
}
```

`HasApiTokens` 为 User 提供以下能力：
- `createToken($name, $scopes = [])` — 创建 PAT
- `token()` — 获取当前请求使用的令牌实例
- `tokens()` — 令牌关联查询

### 9.4 API 中间件组执行链路

[Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Kernel.php#L82-L88) 中 `api` 中间件组定义：

```php
'api' => [
    'auth:api',                    // 1. Passport TokenGuard 验证 Bearer Token
    CheckLocale::class,            // 2. 语言/区域检查
    LogAuthedUserHeader::class,    // 3. 记录认证用户头信息
    SetPaginationDefaults::class,  // 4. 分页默认值（继承自 ThrottleRequests）
    SubstituteBindings::class,     // 5. 路由模型绑定
],
```

[routes/api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/routes/api.php#L18) 中所有 v1 路由自动应用此中间件组：

```php
Route::group(['prefix' => 'v1', 'middleware' => ['api', 'api-throttle:api']], function () {
    // 全部 v1 API 路由
});
```

> `api-throttle` 对应 [SetAPIResponseHeaders](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Middleware/SetAPIResponseHeaders.php)，继承自 Laravel 的 `ThrottleRequests`，增加了 `Retry-After` 和 `X-RateLimit-Reset` 响应头。

### 9.5 PAT 生命周期：创建、查看、撤销

#### 创建 PAT

[ProfileController::createApiToken()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Controllers/Api/ProfileController.php#L98-L120)：

```php
public function createApiToken(Request $request): JsonResponse
{
    // 权限检查：需要 self.api 权限
    if (! Gate::allows('self.api')) {
        abort(403);
    }

    $accessTokenName = $request->input('name', 'Auth Token');

    // 调用 HasApiTokens::createToken() 生成 PAT
    if ($accessToken = auth()->user()->createToken($accessTokenName)->accessToken) {
        $token = DB::table('oauth_access_tokens')
            ->where('user_id', '=', auth()->id())
            ->where('name', '=', $accessTokenName)
            ->orderBy('created_at', 'desc')->first();

        return response()->json(Helper::formatStandardApiResponse('success', [
            'id'    => $token->id,
            'token' => $accessToken,        // 明文令牌，仅此一次返回
            'name'  => $accessTokenName,
        ], ...));
    }
}
```

> **安全要点**：明文令牌仅在创建时返回一次，之后无法再获取。

#### 查看 PAT 列表

[ProfileController::showApiTokens()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Controllers/Api/ProfileController.php#L157-L171)：

```php
public function showApiTokens(): JsonResponse
{
    if (! Gate::allows('self.api')) {
        abort(403);
    }

    $tokens = $this->tokenRepository->forUser(auth()->user()->getAuthIdentifier());
    $token_values = $tokens->load('client')->filter(function ($token) {
        return $token->client->personal_access_client && ! $token->revoked;
    })->values();

    return response()->json(Helper::formatStandardApiResponse('success', $token_values, null));
}
```

> 仅返回 `personal_access_client` 类型且未撤销的令牌。

#### 撤销 PAT

[ProfileController::deleteApiToken()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Controllers/Api/ProfileController.php#L129-L148)：

```php
public function deleteApiToken($tokenId): Response
{
    if (! Gate::allows('self.api')) {
        abort(403);
    }

    $token = $this->tokenRepository->findForUser(
        $tokenId, auth()->user()->getAuthIdentifier()
    );

    if (is_null($token)) {
        return new Response('', 404);
    }

    $token->revoke();    // 软撤销，令牌标记为 revoked=true

    return new Response('', Response::HTTP_NO_CONTENT);
}
```

### 9.6 用户删除时的令牌批量处理

[User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L279-L316) 提供两种令牌清理方式：

**撤销（软删除）** — [revokeAssociatedPassportTokens()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L279-L295)：

```php
private function revokeAssociatedPassportTokens(): void
{
    $accessTokenIds = DB::table('oauth_access_tokens')
        ->where('user_id', $this->id)
        ->pluck('id');

    DB::table('oauth_access_tokens')
        ->whereIn('id', $accessTokenIds)
        ->update(['revoked' => true]);           // 标记 revoked

    DB::table('oauth_refresh_tokens')
        ->whereIn('access_token_id', $accessTokenIds)
        ->update(['revoked' => true]);
}
```

**清除（硬删除）** — [purgeAssociatedPassportTokens()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L301-L316)：

```php
private function purgeAssociatedPassportTokens(): void
{
    $accessTokenIds = DB::table('oauth_access_tokens')
        ->where('user_id', $this->id)
        ->pluck('id');

    if ($accessTokenIds->isNotEmpty()) {
        DB::table('oauth_refresh_tokens')
            ->whereIn('access_token_id', $accessTokenIds)
            ->delete();                          // 物理删除 refresh_token
    }

    DB::table('oauth_access_tokens')
        ->where('user_id', $this->id)
        ->delete();                              // 物理删除 access_token
}
```

### 9.7 API v1 完整鉴权链路

```
GET /api/v1/hardware
    │
    ├── HTTP Header: Authorization: Bearer eyJ0eXAiOiJKV1...
    │
    ▼
[Kernel] api 中间件组
    │
    ├── auth:api
    │     └── Passport\TokenGuard
    │           ├── 解析 Bearer Token
    │           ├── 查询 oauth_access_tokens 表验证有效性
    │           ├── 检查 revoked = false
    │           ├── 检查未过期
    │           └── 加载关联 User 模型 → auth()->user()
    │                 ├── 令牌无效 → 返回 401 Unauthorized
    │                 └── 令牌有效 → 继续
    │
    ├── CheckLocale → 设置语言
    ├── LogAuthedUserHeader → 日志
    ├── SetPaginationDefaults → 限流头
    └── SubstituteBindings → 路由模型绑定
    │
    ▼
路由匹配 → 控制器方法 → FormRequest authorize() → $this->authorize()
```

### 9.8 Web 与 API 认证对比

| 特征 | Web 路由 | API v1 路由 |
|------|---------|------------|
| 认证驱动 | `session` | `passport` |
| 中间件 | `auth` (session) | `auth:api` (Bearer Token) |
| 令牌类型 | Session Cookie | PAT (Personal Access Token) |
| 权限检查 | 控制器 authorize() | FormRequest authorize() + 控制器 authorize() |
| 公司隔离 | CompanyableScope | CompanyableScope（同一套） |
| 策略类 | 相同 | 相同 |
| 特殊路由 | `authorize:superuser` | Settings 路由使用 `auth` + `authorize:superuser` |

---

## 十、Form Request authorize 通道

### 10.1 Form Request 继承体系

```
Illuminate\Foundation\Http\FormRequest  (Laravel 基类)
        │
        ▼
App\Http\Requests\Request  (抽象基类)
        │   ├── json() — JSON 解析容错
        │   ├── rules() — 默认空规则
        │   └── response() — 统一错误格式（API 返回 JSON）
        │
        ▼
App\Http\Requests\ImageUploadRequest
        │   ├── authorize() → return true  (默认放行)
        │   ├── rules() → 图片 MIME 验证
        │   ├── handleImages() → 图片存储处理
        │   └── base64FileKeys() → Base64 转文件
        │
        ├── StoreAssetRequest         authorize() → Gate::allows('create', Asset::class)
        ├── UpdateAssetRequest         authorize() → Gate::allows('update', $this->asset)
        ├── StoreAccessoryRequest      authorize() → Gate::allows('create', Accessory::class)
        ├── StoreConsumableRequest     authorize() → Gate::allows('create', Consumable::class)
        ├── StoreComponentRequest      authorize() → Gate::allows('create', Component::class)
        ├── UpdateComponentRequest     authorize() → Gate::allows('update', $this->component)
        │
        ▼
App\Http\Requests\Request (非 ImageUploadRequest 子类)
        │
        ├── AssetCheckoutRequest       authorize() → return true
        ├── AssetCheckinRequest        authorize() → return true
        └── StoreSecuritySettings      authorize() → Gate::allows('superuser')
```

### 10.2 Form Request 执行时序

Laravel 在解析控制器方法参数时，如果参数类型是 FormRequest 子类，会自动执行以下流程：

```
控制器方法签名: public function store(StoreAssetRequest $request)
    │
    ▼
1. Laravel 容器解析 StoreAssetRequest
    │
    ├── 2. 调用 $request->authenticate()
    │     └── 检查用户是否已登录（auth 中间件已保证）
    │
    ├── 3. 调用 $request->authorize()
    │     └── Gate::allows('create', Asset::class)
    │           ├── AuthServiceProvider@before → isSuperUser()
    │           ├── 策略类 AssetPolicy@before → hasAccess('admin') + 公司检查
    │           └── AssetPolicy@create() → hasAccess('assets.create')
    │     ├── 返回 false → 抛出 AuthorizationException → 403
    │     └── 返回 true → 继续
    │
    ├── 4. 调用 $request->prepareForValidation()
    │     └── 数据预处理（公司 ID 覆盖、金额解析等）
    │
    ├── 5. 调用 $request->withValidator($validator)  [MayContainCustomFields Trait]
    │     └── 添加自定义字段验证后钩子
    │
    └── 6. 调用 $request->validate()
          └── rules() 验证规则执行
                ├── 验证失败 → ValidationException → 422
                └── 验证通过 → 注入控制器方法
```

### 10.3 FormRequest authorize() 与控制器 authorize() 的协作关系

系统中存在**双重授权检查**模式。以资产创建为例：

**FormRequest 层** — [StoreAssetRequest](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Requests/StoreAssetRequest.php#L21-L24)：

```php
public function authorize(): bool
{
    return Gate::allows('create', Asset::class);
}
```

**控制器层** — [AssetsController](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Controllers/Assets/AssetsController.php#L84-L98)：

```php
public function create(Request $request): View
{
    $this->authorize('create', Asset::class);  // 同样检查 assets.create
    // ...
}
```

两种模式的协作：

| 场景 | FormRequest | 控制器 | 说明 |
|------|-------------|--------|------|
| Web `store` | ✅ `Gate::allows('create', Asset::class)` | ✅ `$this->authorize(Asset::class)` | 双重检查，FormRequest 先拦截 |
| Web `create` | ❌ 无 FormRequest | ✅ `$this->authorize('create', Asset::class)` | 仅控制器检查 |
| API `store` | ✅ `Gate::allows('create', Asset::class)` | ❌ API 控制器通常不做二次检查 | 仅 FormRequest 检查 |
| Web `checkout` | ❌ `AssetCheckoutRequest` 返回 true | ✅ `$this->authorize('checkout', $asset)` | 权限检查在控制器中 |

> **注意**：Web 控制器的 `create` 方法使用普通 `Request`，不经过 FormRequest 授权；而 `store` 方法使用 `StoreAssetRequest`，先经过 FormRequest 授权再进入控制器。checkout/checkin 的 FormRequest 均返回 `true`，将权限检查完全交给控制器——因为此时需要传入具体资产实例进行策略判断（公司隔离检查）。

### 10.4 prepareForValidation 中的公司 ID 写入隔离

[StoreAssetRequest::prepareForValidation()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Requests/StoreAssetRequest.php#L26-L46) 中执行了关键的公司 ID 安全覆盖：

```php
public function prepareForValidation(): void
{
    parent::prepareForValidation();

    // 关键：将用户提交的 company_id 替换为当前用户允许的值
    $idForCurrentUser = is_numeric($this->company_id)
        ? Company::getIdForCurrentUser($this->company_id)
        : $this->company_id;

    $this->merge([
        'asset_tag' => $this->asset_tag ?? Asset::autoincrement_asset(),
        'company_id' => $idForCurrentUser,   // 覆盖为安全值
        'purchase_cost' => ...,
    ]);
}
```

[Company::getIdForCurrentUser()](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Company.php#L148-L182) 的安全覆盖逻辑：

```php
public static function getIdForCurrentUser($unescaped_input)
{
    // FMCS 未启用 → 直接使用用户提交值
    if (! self::isFullMultipleCompanySupportEnabled()) {
        return self::getIdFromInput($unescaped_input);
    }

    // 超级用户 → 可以设置任意公司
    if ($current_user->isSuperUser()) {
        return self::getIdFromInput($unescaped_input);
    }

    $userCompanyIds = self::getCurrentUserCompanyIds();
    $submittedId = (int) self::getIdFromInput($unescaped_input);

    // 提交的公司在用户所属公司列表中 → 允许
    if ($submittedId && in_array($submittedId, $userCompanyIds)) {
        return $submittedId;
    }

    // 提交了不属于自己的公司 → 抛出验证异常（阻止跨公司写入）
    if ($submittedId && ! empty($userCompanyIds)) {
        throw ValidationException::withMessages([
            'company_id' => [trans('validation.in', ['attribute' => 'company_id'])],
        ]);
    }

    // 未提交公司 → 自动填充用户唯一公司或 null
    return count($userCompanyIds) === 1 ? $userCompanyIds[0] : null;
}
```

> 这是**写入隔离**的核心：用户无法通过 API 提交 `company_id` 将资产分配给无权访问的公司。如果不属于用户的公司 ID 被提交，直接抛出 422 验证异常。

### 10.5 MayContainCustomFields Trait

[app/Http/Requests/Traits/MayContainCustomFields.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Http/Requests/Traits/MayContainCustomFields.php) 在验证后检查自定义字段合法性：

```php
trait MayContainCustomFields
{
    public function withValidator($validator)
    {
        // 解析资产型号
        if (request()->has('model_id') != '') {
            $asset_model = AssetModel::find(request()->input('model_id'));
        } elseif ((request()->route('asset') && (request()->route('asset')->model_id))) {
            $asset_model = AssetModel::find(request()->route('asset')->model_id);
        } else {
            // POST/PATCH/PUT 不同解析方式
        }

        // 验证后钩子：检查自定义字段是否属于该型号的 fieldset
        $validator->after(function ($validator) use ($asset_model) {
            $request_fields = $this->collect()->keys()
                ->filter(fn ($attr) => str_starts_with($attr, '_snipeit_'));

            if (count($request_fields) > 0 && $validator->errors()->isEmpty()) {
                $request_fields->diff($asset_model?->fieldset?->fields?->pluck('db_column'))
                    ->each(function ($request_field_name) use ($validator) {
                        // 字段不属于该型号 → 添加验证错误
                        $validator->errors()->add($request_field_name, ...);
                    });
            }
        });
    }
}
```

---

## 十一、CompanyableTrait 公司隔离 Scope 全链路

### 11.1 两级隔离 Trait 体系

```
CompanyableTrait                适用于有 company_id 列的独立模型
    │   ├── bootCompanyableTrait() → 注册 CompanyableScope
    │   └── canCheckoutTo()        → FMCS 借出合法性检查
    │
CompanyableChildTrait           适用于无 company_id 列的子模型
    │   └── bootCompanyableChildTrait() → 注册 CompanyableChildScope
    │       └── 通过父模型关系间接隔离
```

### 11.2 CompanyableTrait 详解

位置：[app/Models/Traits/CompanyableTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Traits/CompanyableTrait.php)

**自动注册全局 Scope**：

```php
public static function bootCompanyableTrait()
{
    static::addGlobalScope(new CompanyableScope);
}
```

`bootXxxTrait()` 是 Laravel 的约定方法——当模型 `use CompanyableTrait` 时，Laravel 在启动模型时自动调用此方法，将 `CompanyableScope` 注册为全局 Scope。

**canCheckoutTo 借出合法性判断**：

```php
public function canCheckoutTo(Model $target): bool
{
    $settings = Setting::getSettings();

    // FMCS 未启用 → 允许任何借出
    if (! $settings->full_multiple_companies_support) {
        return true;
    }

    // 资产无公司 → 根据目标公司决定
    if (! $this->company_id) {
        if (is_null($target->company_id)) {
            return true;
        }
        return (bool) $settings->null_company_is_floater;
    }

    // 目标是 User → 检查用户是否属于资产所在公司
    if ($target instanceof User) {
        return $target->canReceiveFromCompany((int) $this->company_id);
    }

    // 目标无公司 → floater 配置决定
    if (is_null($target->company_id)) {
        return (bool) $settings->null_company_is_floater;
    }

    // 通用：公司 ID 必须匹配
    return (int) $target->company_id === (int) $this->company_id;
}
```

### 11.3 CompanyableScope 读取隔离

位置：[app/Models/CompanyableScope.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/CompanyableScope.php)

```php
final class CompanyableScope implements Scope
{
    public function apply(Builder $builder, Model $model)
    {
        return Company::scopeCompanyables($builder);
    }
}
```

**scopeCompanyables** 入口 — [Company.php#L372](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Company.php#L372)：

```php
public static function scopeCompanyables($query, $column = 'company_id', $table_name = null)
{
    // 三种跳过条件
    if (! self::isFullMultipleCompanySupportEnabled()   // FMCS 未启用
        || (Auth::hasUser() && auth()->user()->isSuperUser())  // 超级用户
        || (! Auth::hasUser())) {                        // CLI / 未登录
        return $query;
    }

    return self::scopeCompanyablesDirectly($query, $column, $table_name);
}
```

**scopeCompanyablesDirectly** 核心过滤 — [Company.php#L388](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Company.php#L388)：

| 表类型 | 过滤逻辑 | 说明 |
|--------|---------|------|
| `companies` | `whereIn('companies.id', $companyIds)` 或 `whereNull` | 直接按 ID 过滤 |
| `users` | `whereIn('users.id', 子查询 company_user)` | 通过 pivot 表多对多过滤 |
| `action_logs` | `whereIn(company_id) or WhereNull(company_id)` | 全局对象日志对所有公司可见 |
| 通用表 | `whereIn(company_id, $companyIds)` | 标准 company_id 列过滤 |
| 通用表 + floater | `whereIn(company_id) or WhereNull(company_id)` | 无公司数据对所有人可见 |

### 11.4 CompanyableChildScope 子模型隔离

位置：[app/Models/CompanyableChildScope.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/CompanyableChildScope.php)

某些模型（如 `Maintenance`、`LicenseSeat`）自身没有 `company_id` 列，但通过父模型（如 `Asset`、`License`）间接属于某个公司：

```php
final class CompanyableChildScope implements Scope
{
    public function apply(Builder $builder, Model $model)
    {
        $model = $builder->getModel();
        return Company::scopeCompanyableChildren($model->getCompanyableParents(), $builder);
    }
}
```

**scopeCompanyableChildren** — [Company.php#L495](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Company.php#L495)：

```php
public static function scopeCompanyableChildren(array $companyable_names, $query)
{
    if (! self::isFullMultipleCompanySupportEnabled()
        || (Auth::hasUser() && auth()->user()->isSuperUser())) {
        return $query;
    }

    $f = function ($q) {
        static::scopeCompanyablesDirectly($q);   // 复用父模型的过滤逻辑
    };

    // whereHas(parent1, filter) or whereHas(parent2, filter)
    $q = $query->where(function ($q) use ($companyable_names, $f) {
        $q2 = $q->whereHas($companyable_names[0], $f);
        for ($i = 1; $i < count($companyable_names); $i++) {
            $q2 = $q2->orWhereHas($companyable_names[$i], $f);
        }
    });

    return $q;
}
```

**子模型 getCompanyableParents() 定义**：

| 子模型 | 父模型关系 | 位置 |
|--------|-----------|------|
| `Maintenance` | `['asset']` | [Maintenance.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Maintenance.php#L110-L113) |
| `LicenseSeat` | `['asset', 'license']` | [LicenseSeat.php](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/LicenseSeat.php#L67-L70) |

> `LicenseSeat` 同时关联 `asset` 和 `license` 两个父模型，任一父模型属于用户可见公司即允许访问。

### 11.5 getCurrentUserCompanyIds — 公司 ID 获取

[Company.php#L106](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Company.php#L106)：

```php
private static function getCurrentUserCompanyIds(): array
{
    if (! Auth::hasUser()) {
        return [];
    }

    return DB::table('company_user')
        ->where('user_id', auth()->id())
        ->pluck('company_id')
        ->toArray();
}
```

> 用户的可见公司完全由 `company_user` pivot 表决定，不再依赖 User 模型的 `company_id` 标量字段。

### 11.6 User 多公司关联

[User.php#L623](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L623)：

```php
public function companies(): BelongsToMany
{
    return $this->belongsToMany(Company::class, 'company_user');
}
```

**canReceiveFromCompany** — [User.php#L635](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L635)：

```php
public function canReceiveFromCompany(int $companyId): bool
{
    if (! $companyId) {
        return true;
    }

    // 直接查 pivot 表，绕过 Company 模型的 FMCS 全局 Scope
    $userCompanyIds = DB::table('company_user')
        ->where('user_id', $this->id)
        ->pluck('company_id');

    if ($userCompanyIds->isEmpty()) {
        return (bool) Setting::getSettings()->null_company_is_floater;
    }

    return $userCompanyIds->contains($companyId);
}
```

> 故意绕过 Eloquent 关联，直接查 `company_user` 表，避免 CompanyableScope 对查询自身的递归过滤。

**syncCompaniesWithLogging** — [User.php#L673](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/User.php#L673)：

```php
public function syncCompaniesWithLogging(array $companyIds): void
{
    $oldIds = $this->companies()->orderBy('companies.id')->pluck('companies.id')->toArray();
    $this->companies()->sync($companyIds);
    $newIds = $this->companies()->orderBy('companies.id')->pluck('companies.id')->toArray();

    if ($oldIds === $newIds) {
        return;
    }

    // 将公司变更合并到已有的 Actionlog 记录中
    $companyChange = ['companies' => ['old' => $oldIds, 'new' => $newIds]];
    if ($this->currentUpdateLogId && ($existing = Actionlog::find($this->currentUpdateLogId))) {
        $meta = json_decode($existing->log_meta ?? '{}', true) ?: [];
        $existing->log_meta = json_encode(array_merge($meta, $companyChange));
        $existing->save();
    }
}
```

### 11.7 isCurrentUserHasAccess — 策略层公司访问检查

[Company.php#L190](file:///d:/fz/0601-2/solo-dogfeeding/code/9-snipe-it/app/Models/Company.php#L190)：

```
Company::isCurrentUserHasAccess($companyable)
    │
    ├── $companyable 为 null → false
    ├── FMCS 未启用 → true
    ├── 模型表无 company_id 列 → true（全局对象如 Location）
    ├── 当前用户为 SuperUser → true
    ├── 用户无 pivot 且无标量 company_id → true（传统无公司用户）
    │
    ├── $companyable 是 User 实例
    │     └── 检查 company_user pivot 交集
    │           └── array_intersect($userCompanyIds, $companyableCompanyIds) 非空 → true
    │
    └── 通用模型
          └── $companyable->company_id in $userCompanyIds → true
```

> 此方法在 `SnipePermissionsPolicy@before` 钩子中被调用，是策略层公司访问控制的核心。

### 11.8 读取隔离 vs 写入隔离

| 隔离类型 | 机制 | 触发点 | 作用 |
|---------|------|--------|------|
| **读取隔离** | CompanyableScope 全局 Scope | Eloquent 查询时自动应用 | 用户只能查到自己公司的数据 |
| **写入隔离** | Company::getIdForCurrentUser() | FormRequest prepareForValidation() | 用户无法将资产写入非授权公司 |
| **策略隔离** | Company::isCurrentUserHasAccess() | SnipePermissionsPolicy@before() | 策略检查时拒绝跨公司操作 |
| **借出隔离** | canCheckoutTo() / canReceiveFromCompany() | 控制器借出逻辑 | 资产只能借出给同公司用户 |

### 11.9 FMCS 配置项影响表

| 配置项 | 值 | 影响 |
|--------|---|------|
| `full_multiple_companies_support` | `0` | 所有公司数据互通，无隔离 |
| `full_multiple_companies_support` | `1` | 启用公司数据隔离 |
| `null_company_is_floater` | `0` | 无公司数据仅对无公司用户可见 |
| `null_company_is_floater` | `1` | 无公司数据对所有用户可见（浮动） |

### 11.10 完整公司隔离调用链路

```
┌─ 读取隔离 ────────────────────────────────────────────────┐
│                                                            │
│  Asset::all()                                              │
│      │                                                     │
│      └── CompanyableScope::apply()                         │
│            └── Company::scopeCompanyables($builder)        │
│                  ├── FMCS off / SuperUser / CLI → 不过滤    │
│                  └── scopeCompanyablesDirectly()            │
│                        ├── getCurrentUserCompanyIds()       │
│                        │     └── DB::table('company_user')  │
│                        ├── users 表 → pivot 子查询          │
│                        ├── action_logs → whereIn + orNull   │
│                        ├── floater → whereIn + orNull       │
│                        └── 标准 → whereIn(company_id)       │
│                                                            │
├─ 写入隔离 ────────────────────────────────────────────────┤
│                                                            │
│  StoreAssetRequest::prepareForValidation()                 │
│      │                                                     │
│      └── Company::getIdForCurrentUser($this->company_id)   │
│            ├── FMCS off → 直接使用提交值                     │
│            ├── SuperUser → 直接使用提交值                     │
│            ├── 提交值 ∈ 用户公司列表 → 使用提交值             │
│            ├── 提交值 ∉ 用户公司列表 → 抛出 422              │
│            └── 无提交值 → 自动填充唯一公司或 null             │
│                                                            │
├─ 策略隔离 ────────────────────────────────────────────────┤
│                                                            │
│  $this->authorize('checkout', $asset)                      │
│      │                                                     │
│      └── AssetPolicy@before()                              │
│            └── Company::isCurrentUserHasAccess($asset)      │
│                  ├── FMCS off → true                       │
│                  ├── SuperUser → true                      │
│                  ├── User 实例 → pivot 交集检查              │
│                  └── 通用 → company_id ∈ userCompanyIds     │
│                                                            │
├─ 借出隔离 ────────────────────────────────────────────────┤
│                                                            │
│  $asset->canCheckoutTo($target)                            │
│      │                                                     │
│      ├── FMCS off → true                                   │
│      ├── 资产无公司 → floater 配置决定                       │
│      ├── target 是 User → canReceiveFromCompany()          │
│      │     └── DB::table('company_user') 直接查 pivot       │
│      └── 通用 → company_id 匹配                            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```
