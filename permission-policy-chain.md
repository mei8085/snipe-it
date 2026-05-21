# Snipe-IT 权限判定三层协作机制分析

## 核心结论

**最终"拍板"的是策略类（Policy），但三层各司其职，形成完整的权限链：**

1. **权限声明层**（配置 + 注册）：定义权限有哪些、哪个模型对应哪个策略
2. **策略判定层**（Policy 类）：核心判定逻辑，真正决定"允许/拒绝"
3. **请求拦截层**（中间件 + 控制器）：在合适的时机触发权限检查

---

## 三层架构详解

### 第一层：权限声明层 —— "规则的定义者"

#### 1.1 权限键名定义：`config/permissions.php`

这是系统所有权限的"花名册"，定义了所有可用的权限标识：

```php
'Assets' => [
    ['permission' => 'assets.view',    'display' => true],
    ['permission' => 'assets.create',  'display' => true],
    ['permission' => 'assets.edit',    'display' => true],
    ['permission' => 'assets.delete',  'display' => true],
    ['permission' => 'assets.checkout','display' => true],
    ['permission' => 'assets.checkin', 'display' => true],
    ['permission' => 'assets.audit',   'display' => true],
    // ...
],
```

> **作用**：只是权限的"字典"，本身不做判定，供UI展示和代码引用。

#### 1.2 策略注册：`app/Providers/AuthServiceProvider.php:62-83`

将**模型类**与**策略类**绑定，告诉 Laravel 检查某个模型的权限时该用哪个策略：

```php
protected $policies = [
    Asset::class       => AssetPolicy::class,
    Accessory::class   => AccessoryPolicy::class,
    User::class        => UserPolicy::class,
    // ... 20+ 个模型的策略映射
];
```

#### 1.3 全局 Gate 定义：`AuthServiceProvider.php:112-275`

定义全局 Gate 规则，包括**超级管理员豁免**的关键逻辑：

```php
// 最优先执行的全局前置钩子
Gate::before(function ($user, $ability) {
    // 演示模式下的特殊限制
    if (($ability == 'editableOnDemo') && (config('app.lock_passwords'))) {
        return false;
    }
    // 超级管理员直接放行 ALL
    if ($user->isSuperUser()) {
        return true;
    }
});
```

> **关键**：这里是整个权限系统的"第一道闸门"，超级管理员在进入任何策略前就已放行。

还定义了一些全局通用 Gate：
```php
Gate::define('admin', function ($user) {
    return $user->hasAccess('admin');
});
Gate::define('import', function ($user) {
    return $user->hasAccess('import');
});
```

---

### 第二层：策略判定层 —— "核心决策者"（真正拍板的地方）

这是权限判定的**核心层**，所有复杂逻辑都在这里。

#### 2.1 继承关系链

```
SnipePermissionsPolicy (抽象基类)
        ↓
CheckoutablePermissionsPolicy (可借出资产基类)
        ↓
AssetPolicy (资产专属策略)
```

#### 2.2 策略基类：`app/Policies/SnipePermissionsPolicy.php:38-73`

**策略前置钩子 `before()`** —— 每个策略方法执行前都会先调用：

```php
public function before(User $user, $ability, $item)
{
    // 1. 管理员放行（但受公司权限约束）
    if ($user->hasAccess('admin')) {
        return true;
    }

    // 2. 如果传的是类名（而非实例），跳过公司检查，继续往下走
    if (! $item instanceof Model) {
        return;
    }

    // 3. 多公司权限检查（FMCSA 模式）
    if (! Company::isCurrentUserHasAccess($item)) {
        return false;  // 不在用户可访问的公司范围内，直接拒绝
    }
}
```

> **关键点**：
> - `before()` 返回 `true` = 放行，返回 `false` = 拒绝，返回 `null` = 继续执行具体方法
> - 超级管理员已在 `Gate::before()` 层放行，到不了这里
> - `admin` 角色在这里放行，但受公司范围约束

#### 2.3 通用权限方法：`SnipePermissionsPolicy.php:80-163`

```php
// 列表权限
public function index(User $user) {
    return $user->hasAccess($this->columnName().'.view');
}

// 查看权限
public function view(User $user, $item = null) {
    return $user->hasAccess($this->columnName().'.view');
}

// 创建权限
public function create(User $user) {
    return $user->hasAccess($this->columnName().'.create');
}

// 编辑权限
public function update(User $user, $item = null) {
    return $user->hasAccess($this->columnName().'.edit');
}

// 删除权限（附加软删除检查）
public function delete(User $user, $item = null) {
    $itemConditional = true;
    if ($item) {
        $itemConditional = empty($item->deleted_at);
    }
    return $itemConditional && $user->hasAccess($this->columnName().'.delete');
}
```

#### 2.4 可借出资产基类：`app/Policies/CheckoutablePermissionsPolicy.php:15-42`

在基类基础上增加借出/归还相关权限：

```php
public function checkout(User $user, $item = null) {
    return $user->hasAccess($this->columnName().'.checkout');
}

public function checkin(User $user, $item = null) {
    return $user->hasAccess($this->columnName().'.checkin');
}

// manage 是"或"逻辑：有任一操作权限就算可以管理
public function manage(User $user, $item = null) {
    return $user->hasAccess($this->columnName().'.checkin')
         || $user->hasAccess($this->columnName().'.edit')
         || $user->hasAccess($this->columnName().'.checkout');
}
```

#### 2.5 资产专属策略：`app/Policies/AssetPolicy.php:10-28`

资产特有的权限：

```php
class AssetPolicy extends CheckoutablePermissionsPolicy
{
    // 告诉基类权限前缀是 'assets'
    protected function columnName() {
        return 'assets';
    }

    // 资产特有：查看可申请的资产
    public function viewRequestable(User $user, ?Asset $asset = null) {
        return $user->hasAccess('assets.view.requestable');
    }

    // 资产特有：资产盘点
    public function audit(User $user, ?Asset $asset = null) {
        return $user->hasAccess('assets.audit');
    }
}
```

#### 2.6 底层权限检查：`app/Models/User.php:514-500`

所有 `hasAccess()` 最终调用 `checkPermissionSection()`：

```php
public function hasAccess($section)
{
    if ($this->isSuperUser()) {
        return true;  // 超级用户再次兜底（虽然 Gate::before 已处理）
    }
    return $this->checkPermissionSection($section);
}

protected function checkPermissionSection($section)
{
    // 1. 检查用户直接权限（JSON 存储）
    $user_permissions = json_decode($this->permissions, true);
    if (isset($user_permissions[$section])) {
        if ($user_permissions[$section] == '1')  return true;   // 明确允许
        if ($user_permissions[$section] == '-1') return false;  // 明确拒绝
    }

    // 2. 检查用户所属用户组的权限
    foreach ($this->groups as $user_group) {
        $group_permissions = json_decode($user_group->permissions, true);
        if (isset($group_permissions[$section]) && $group_permissions[$section] == '1') {
            return true;
        }
    }

    return false;  // 都没找到，默认拒绝
}
```

> **权限优先级**：用户个人权限 > 用户组权限（用户明确拒绝 `-1` 优先级最高）

---

### 第三层：请求拦截层 —— "时机的把控者"

在正确的时机调用权限检查，有两种调用方式：

#### 3.1 路由中间件：`app/Http/Middleware/CheckPermissions.php:21-30`

在路由定义时直接拦截，适合粗粒度权限：

```php
public function handle($request, Closure $next, $section = null)
{
    if (Gate::allows($section)) {
        return $next($request);
    }
    return response()->view('layouts/basic', ['content' => view('errors/403')], 403);
}
```

**使用方式**（在路由中）：
```php
// routes/web.php:155
Route::group(['prefix' => 'admin', 'middleware' => ['auth', 'authorize:superuser']], function () {
    // 只有 superuser 能进的管理页面
});

// routes/api.php:935
Route::group(['middleware' => ['auth', 'authorize:superuser'], 'prefix' => 'settings'], function () {
    // 系统设置 API
});
```

> 中间件注册在 `app/Http/Kernel.php:104`：`'authorize' => CheckPermissions::class`

#### 3.2 控制器内授权：`app/Http/Controllers/Assets/AssetsController.php`

这是**最常用**的方式，在控制器动作内精细控制：

```php
// 列表页 - 检查 index 权限
public function index(Request $request): View
{
    $this->authorize('index', Asset::class);  // 传类名
    // ...
}

// 创建页 - 检查 create 权限
public function create(Request $request): View
{
    $this->authorize('create', Asset::class);
    // ...
}

// 编辑页 - 检查具体资产实例（会触发公司范围检查）
public function edit(Asset $asset): View|RedirectResponse
{
    $this->authorize($asset);  // 传实例，自动匹配 update 权限
    // ...
}

// 查看详情 - 显式指定 view 权限
public function show(Asset $asset): View|RedirectResponse
{
    $this->authorize('view', $asset);
    // ...
}

// 删除资产
public function destroy(Request $request, Asset $asset): RedirectResponse
{
    $this->authorize('delete', $asset);
    // ...
}
```

> **小技巧**：`$this->authorize($asset)` 不传动作名时，Laravel 会根据当前路由动作自动推断（如 `edit` 路由 → `update` 权限）。

---

## 完整调用链路（以资产编辑为例）

```
用户请求 GET /hardware/123/edit
        ↓
[中间件层] auth 中间件 → 认证通过
        ↓
[控制器层] AssetsController::edit() 调用 $this->authorize($asset)
        ↓
[Laravel 内核] 查找 Asset::class 对应的 AssetPolicy
        ↓
[全局 Gate] Gate::before() 检查
    ├─ 是超级管理员？→ 返回 true → ✅ 放行
    └─ 否 → 继续
        ↓
[策略层] AssetPolicy::before() 检查
    ├─ 是 admin 角色？→ 返回 true → ✅ 放行
    ├─ 否 → 检查公司范围
    │   ├─ 不在可访问公司？→ 返回 false → ❌ 403
    │   └─ 在范围内 → 继续
        ↓
[策略层] AssetPolicy::update($user, $asset)
    ↓ 调用
    $user->hasAccess('assets.edit')
        ↓
[模型层] User::checkPermissionSection('assets.edit')
    ├─ 用户权限有 assets.edit = 1？→ ✅
    ├─ 用户权限有 assets.edit = -1？→ ❌
    └─ 遍历用户组，任一有 assets.edit = 1？→ ✅ / ❌
        ↓
✅ 通过 → 执行控制器逻辑 / ❌ 不通过 → 抛出 AuthorizationException → 403 页面
```

---

## 深度解析：三个关键协作机制

### 一、全局 Gate 与策略判定的完整执行顺序

这是最容易混淆的地方。Laravel 的授权系统有**明确的层级检查顺序**，Snipe-IT 在各层都植入了钩子，形成了一个 7 层的判定漏斗：

```
调用 Gate::allows($ability, $arguments)
        ↓
【第1层】 Gate::before() 全局前置钩子
    在 AuthServiceProvider.php:112-122 定义
    ├─ 演示模式检查（editableOnDemo）
    ├─ 超级管理员检查 → true 直接放行，终止后续所有检查
    └─ 返回 null → 继续
        ↓
【第2层】 检查是否有直接的 Gate::define() 定义
    在 AuthServiceProvider.php:130-275 定义了 'admin', 'import', 'superuser', 'reports.view' 等
    ├─ 有匹配的 Gate → 执行该闭包 → 返回 true/false → 终止
    └─ 无匹配 → 继续向下查找 Policy
        ↓
【第3层】 解析 $arguments 找对应模型 → 查找绑定的 Policy 类
    通过 AuthServiceProvider 的 $policies 数组映射
    如 Asset::class → AssetPolicy::class
        ↓
【第4层】 Policy::before() 策略级前置钩子
    在 SnipePermissionsPolicy.php:38-73 定义
    ├─ admin 角色检查 → true 放行（但注意：这一层还没有检查公司范围？不，看代码顺序）
    ├─ 检查是否为 Model 实例 → 不是实例则 return null 跳过公司检查
    ├─ 公司范围检查 → false 直接拒绝
    └─ 返回 null → 继续
        ↓
【第5层】 检查 Policy 类中是否有对应方法
    如 view(), create(), update(), delete(), audit() 等
    ├─ 有方法 → 调用该方法
    └─ 无方法 → 检查 HandlesAuthorization trait 的兜底逻辑
        ↓
【第6层】 Policy 具体方法执行
    如 AssetPolicy::update() → 调用 $user->hasAccess('assets.edit')
        ↓
【第7层】 底层权限位检查
    User::hasAccess() → User::checkPermissionSection()
    ├─ 用户个人权限检查（1=允许, -1=拒绝）
    └─ 用户组权限遍历检查
```

**关键代码证据**：

1. `Gate::before()` 是最外层，代码位置 `AuthServiceProvider.php:112`：
```php
Gate::before(function ($user, $ability) {
    // 这里是整个系统的第一道闸门
    if ($user->isSuperUser()) {
        return true;  // 超级管理员直接返回true，后面所有层都走不到
    }
});
```

2. 直接 Gate 定义次之，代码位置 `AuthServiceProvider.php:170-174`：
```php
Gate::define('admin', function ($user) {
    return $user->hasAccess('admin');
});
```
> 当调用 `Gate::allows('admin')` 时，匹配到这个定义后直接返回，不会去找 Policy。

3. Policy 的 `before()` 是策略级的，代码位置 `SnipePermissionsPolicy.php:38`：
```php
public function before(User $user, $ability, $item)
{
    if ($user->hasAccess('admin')) {
        return true;  // admin在这里放行，但要注意：只有走到Policy层才会执行这里
    }
    // ... 公司范围检查
}
```

> **致命区别**：`Gate::before()` 是全局的，对 ALL Gate 检查生效；`Policy::before()` 只对该模型的 Policy 检查生效。
>
> 例如：`Gate::allows('import')` 只会走前两层，不会走任何 Policy 层。

**判定顺序总结表**：

| 层级 | 执行位置 | 返回true | 返回false | 返回null | 适用场景 |
|------|---------|---------|-----------|----------|---------|
| 1. Gate::before() | 全局 | 终止，✅通过 | 终止，❌拒绝 | 继续 | 超级管理员豁免、演示模式 |
| 2. Gate::define() | 全局 | 终止，✅通过 | 终止，❌拒绝 | 继续 | 通用权限（admin/import/reports） |
| 3. Policy 查找 | 内核 | - | - | - | 根据模型找对应Policy |
| 4. Policy::before() | 策略类 | 终止，✅通过 | 终止，❌拒绝 | 继续 | admin放行、公司范围检查 |
| 5. Policy 方法 | 策略类 | ✅通过 | ❌拒绝 | - | 具体动作权限（view/create/edit） |

---

### 二、控制器授权 vs 路由中间件：触发路径分流

**同一个权限检查，从不同入口调用，走的代码路径完全不同**。这是理解系统的核心。

#### 路径A：路由中间件 `authorize:xxx`

**代码路径**：`routes/web.php` → `Kernel.php` → `CheckPermissions.php` → `Gate::allows($section)`

```
HTTP Request
    ↓
[路由匹配] 匹配到路由定义：middleware => ['auth', 'authorize:superuser']
    ↓
[中间件栈] 按顺序执行
    ├─ auth 中间件 → 认证
    └─ authorize 中间件（CheckPermissions.php:21）
            ↓
            Gate::allows('superuser')
                ↓
                【只走第1-2层】Gate::before() → Gate::define('superuser')
                （不会走到任何 Policy！因为没有传模型参数）
            ↓
            ✅ 放行 → 进入控制器
            ❌ 拒绝 → 返回403页面
```

**典型用法**（代码位置 `routes/web.php:155`）：
```php
Route::group(['prefix' => 'admin', 'middleware' => ['auth', 'authorize:superuser']], function () {
    Route::get('settings', [SettingsController::class, 'getSettings']);
    // ...
});
```

**特点**：
- ✅ 粗粒度过滤，适合整个路由组
- ❌ 不传模型实例，**不会触发 Policy 层**，更不会触发公司范围检查
- ❌ 只检查简单的 Gate（如 'superuser', 'admin', 'import'）
- ⚠️ 在控制器方法执行**之前**就触发

#### 路径B：控制器内 `$this->authorize()`

**代码路径**：控制器方法 → `AuthorizesRequests trait` → `Gate::authorize()` → 完整7层检查

```
HTTP Request
    ↓
[中间件栈] auth 等基础中间件通过
    ↓
[控制器方法] AssetsController::edit($asset)
    ↓
    $this->authorize('update', $asset)  /  $this->authorize($asset)
            ↓
            Gate::authorize('update', $asset)
                ↓
                【完整走7层】
                Gate::before() → Gate::define() → 找Policy → Policy::before() → Policy::update()
                （因为传了$asset实例，所以会触发公司范围检查！）
            ↓
            ✅ 放行 → 继续执行控制器逻辑
            ❌ 拒绝 → 抛出 AuthorizationException → 渲染403
```

**典型用法对比**：

| 调用方式 | 代码示例 | 触发公司检查 | 走Policy层 | 适用场景 |
|---------|---------|------------|-----------|---------|
| 传类名 | `$this->authorize('index', Asset::class)` | ❌ 不会 | ✅ 会 | 列表页、创建页 |
| 传实例 | `$this->authorize('view', $asset)` | ✅ 会 | ✅ 会 | 查看、编辑、删除具体资产 |
| 只传实例 | `$this->authorize($asset)` | ✅ 会 | ✅ 会 | 自动推断动作（edit→update） |

**关键证据**在 `SnipePermissionsPolicy.php:61-63`：
```php
if (! $item instanceof Model) {
    return;  // 传类名时，这里返回null，跳过下面的公司检查
}
// 只有传实例时才会走到这里
if (! Company::isCurrentUserHasAccess($item)) {
    return false;
}
```

**执行时间线对比**：

```
请求到达
    ↓
[0ms]  全局中间件（StartSession, EncryptCookies...）
    ↓
[10ms] 路由中间件 auth → 认证通过
    ↓
[12ms] 路由中间件 authorize:xxx → 路径A检查（如果定义了的话）
    ↓
[15ms] 控制器构造函数执行
    ↓
[20ms] 控制器方法执行
    ↓
[22ms] 控制器内 $this->authorize() → 路径B检查 ⭐
    ↓
[30ms] 执行业务逻辑
```

> **分流结论**：路由中间件是"门卫"，在门口就拦住粗粒度权限；控制器授权是"安检"，在具体操作前做精细检查。两者可以叠加使用。

---

### 三、同角色不同入口结果不一致：定位排查指南

这是开发中最容易踩的坑。当"同一个用户、同一个权限，在A页面可以，B页面不行"时，按以下步骤排查：

#### 【第一步】确认调用路径：是中间件还是控制器？

检查路由定义和控制器代码：

```bash
# 先看路由有没有 authorize 中间件
grep -n "authorize:" routes/web.php routes/api.php

# 再看控制器有没有 $this->authorize()
grep -n "authorize(" app/Http/Controllers/Assets/AssetsController.php
```

**典型不一致场景1：中间件检查 vs 控制器检查**

- `/admin/settings` 路由有 `authorize:superuser` 中间件 → 走 Gate 检查
- `/hardware/1/edit` 控制器调用 `$this->authorize($asset)` → 走完整 Policy 检查

如果一个用户有 `superuser` 权限但公司范围不匹配：
- ✅ 访问 `/admin/settings` 通过（中间件只检查 superuser Gate）
- ❌ 访问 `/hardware/1/edit` 拒绝（控制器检查触发了公司范围）

#### 【第二步】确认参数类型：传的是类名还是实例？

检查 `authorize()` 的第二个参数：

```php
// ✅ 传类名 - 不会触发公司范围检查
$this->authorize('index', Asset::class);

// ✅ 传实例 - 会触发公司范围检查
$this->authorize('view', $asset);
```

**典型不一致场景2：列表页 vs 详情页**

- 资产列表页 `index()` 调用 `$this->authorize('index', Asset::class)` → 传类名，跳过公司检查
- 资产详情页 `show()` 调用 `$this->authorize('view', $asset)` → 传实例，触发公司检查

同用户有 `assets.view` 权限，但资产不在其公司范围内：
- ✅ 列表页能看到资产（SQL查询已经做了公司范围过滤）
- ❌ 点进去看详情被拒绝（Policy 层再次检查公司范围）

> 代码证据 `SnipePermissionsPolicy.php:61-71`：传类名时 `!$item instanceof Model` 为 true，直接 return，跳过公司检查。

#### 【第三步】确认权限节点：检查的是同一个 ability 吗？

看似都是"编辑权限"，实际检查的可能完全不同：

| 调用代码 | 实际检查的 ability |
|---------|-------------------|
| `Gate::allows('admin')` | `'admin'` |
| `$this->authorize('update', $asset)` | `'update'`（Policy方法）→ 内部检查 `'assets.edit'` |
| `$this->authorize('manage', $asset)` | `'manage'`（Policy方法）→ 内部检查 `'assets.checkin' OR 'assets.edit' OR 'assets.checkout'` |
| `$user->hasAccess('assets.edit')` | `'assets.edit'`（直接查权限位） |

**典型不一致场景3：ability 不匹配**

- 中间件 `authorize:admin` 检查的是 `'admin' 这个 Gate`
- 控制器 `$this->authorize('update', $asset)` 检查的是 Policy 的 `update()` 方法 → 内部检查 `'assets.edit'`

一个用户有 `admin` 全局权限但没有 `assets.edit` 权限位：
- ✅ 通过中间件检查
- ❌ 被控制器授权拒绝（因为 Policy 层检查的是具体的 assets.edit 权限位）

> 注意：`SnipePermissionsPolicy::before()` 中 `$user->hasAccess('admin')` 会放行 admin，但这只在走到 Policy 层时才生效！

#### 【第四步】定位工具：权限检查追踪

当上述步骤还无法定位时，可以在关键位置临时加入调试代码：

```php
// 在 User::checkPermissionSection() 开头加入：
\Log::debug("[PERMISSION CHECK] section=$section, user={$this->id}", [
    'user_permissions' => $this->permissions,
    'groups' => $this->groups->pluck('id', 'name')->toArray(),
]);

// 在 SnipePermissionsPolicy::before() 开头加入：
\Log::debug("[POLICY BEFORE] ability=$ability, is_model=" . ($item instanceof Model ? 'yes' : 'no'), [
    'item_class' => is_object($item) ? get_class($item) : $item,
    'item_id' => is_object($item) && isset($item->id) ? $item->id : null,
    'user_is_admin' => $user->hasAccess('admin'),
]);
```

**排查清单（按优先级）**：

| 检查项 | 检查方法 | 常见坑 |
|-------|---------|-------|
| 1. 调用入口 | 看路由和控制器 | 是中间件调用还是控制器调用？ |
| 2. 参数类型 | `$this->authorize(?, ?)` 第二个参数 | 是 `Asset::class` 还是 `$asset`？ |
| 3. Ability | 看第一个参数 | 是 `'view'`/`'update'` 还是 `'admin'`/`'superuser'`？ |
| 4. 公司范围 | 检查 `$asset->company_id` 和用户可访问公司 | 多公司模式下最容易出问题 |
| 5. 软删除状态 | 检查 `$asset->deleted_at` | `delete()` 方法会拒绝已删除的资产 |
| 6. 演示模式 | `config('app.lock_passwords')` | 演示模式下 `editableOnDemo` Gate 会返回 false |
| 7. 权限优先级 | 用户权限 vs 用户组权限 | 用户 `-1` 拒绝优先级最高，会覆盖组权限 |

---

## 各层职责对比表

| 层级 | 核心文件 | 主要职责 | 能否最终拍板 |
|------|---------|---------|------------|
| **权限声明层** | `config/permissions.php`<br>`AuthServiceProvider.php` | 定义权限键名、注册策略映射、全局 Gate | ❌ 只是定义规则，不做判定 |
| **策略判定层** | `SnipePermissionsPolicy.php`<br>`AssetPolicy.php`<br>`User::hasAccess()` | 核心判定逻辑：超级管理员豁免、管理员豁免、公司范围检查、权限位检查 | ✅ **真正拍板的地方** |
| **请求拦截层** | `CheckPermissions.php`<br>控制器 `$this->authorize()` | 在请求流程的合适时机触发权限检查，返回 403 | ❌ 只是"调用者"，判定结果来自策略层 |

---

## 关键设计洞察

1. **双重 `before` 钩子**：`Gate::before()`（全局）+ `Policy::before()`（策略级），形成灵活的豁免机制
2. **权限前缀约定**：通过 `columnName()` 方法约定权限前缀（如 `assets` + `.view`），大量减少重复代码
3. **"或"逻辑权限**：如 `manage` 权限是多个权限的 OR 组合，体现了策略层的灵活性
4. **软删除感知**：`delete` 方法中检查 `$item->deleted_at`，已删除的资产不能再删除
5. **公司范围隔离**：多公司模式下，即使有权限，也只能操作本公司的资产
6. **权限优先级**：用户个人权限（尤其是 `-1` 拒绝）优先级高于用户组权限

---

## 常见疑问解答

**Q: 为什么管理员能看到所有资产，但普通用户只能看到本公司的？**
A: 在 `SnipePermissionsPolicy::before()` 中，`admin` 角色直接 `return true` 跳过了公司检查。

**Q: 超级管理员和管理员有什么区别？**
A: 超级管理员（`superuser`）在最外层 `Gate::before()` 就放行，不受任何约束；管理员（`admin`）在策略层放行，但仍受多公司范围约束。

**Q: 中间件 `authorize:superuser` 和控制器 `$this->authorize()` 是什么关系？**
A: 两者是互补的。中间件适合粗粒度的路由组过滤（如整个后台都需要 superuser），控制器内授权适合精细到动作的检查（如查看、编辑、删除各自的权限）。

**Q: 如果我想新增一个资产权限 `assets.archive` 该怎么做？**
A: 三层各改一处：
1. 声明层：`config/permissions.php` 增加 `assets.archive` 项
2. 策略层：`AssetPolicy.php` 增加 `archive()` 方法调用 `hasAccess('assets.archive')`
3. 拦截层：在控制器对应方法调用 `$this->authorize('archive', $asset)`

---

**Q: Gate::before() 和 Policy::before() 有什么本质区别？**
A: 三个关键区别：
1. **作用范围不同**：`Gate::before()` 是全局的，对 ALL 权限检查生效；`Policy::before()` 只对特定模型的 Policy 生效
2. **执行顺序不同**：`Gate::before()` 是第1层，`Policy::before()` 是第4层，中间还隔了 `Gate::define()` 匹配
3. **豁免机制不同**：超级管理员在 `Gate::before()` 就被放行，所有后续检查（包括公司范围）都跳过；普通 admin 在 `Policy::before()` 放行，受公司范围约束

例如 `Gate::allows('import')` 只会走 `Gate::before()`，不会走任何 Policy 层的 `before()`。

---

**Q: 为什么我在路由中间件加了 `authorize:admin`，但控制器里还要再 authorize 一次？**
A: 两者检查的维度完全不同，是互补关系而非重复：
1. **中间件 `authorize:admin`**：检查用户是否有全局 admin 角色，粗粒度过滤，不传模型，不检查公司范围，在控制器执行前就触发
2. **控制器 `$this->authorize('update', $asset)`**：检查具体动作权限（如 assets.edit），传模型实例，会检查公司范围，在控制器方法内触发

典型组合使用场景：
- 路由中间件 `authorize:admin` 拦住非管理员
- 控制器内 `$this->authorize('delete', $asset)` 拦住越权操作其他公司资产的管理员

---

**Q: 同一个用户，列表页能看到资产，但点详情就 403，这是什么原因？**
A: 90% 概率是**传参类型导致的公司范围检查差异**：
- 列表页 `index()`：`$this->authorize('index', Asset::class)` → 传类名，`Policy::before()` 中 `!$item instanceof Model` 为 true，跳过公司检查
- 详情页 `show()`：`$this->authorize('view', $asset)` → 传实例，触发 `Company::isCurrentUserHasAccess($item)` 检查

资产虽然在 SQL 查询层被公司范围过滤了，但 Policy 层会做第二次校验。如果数据不一致（如资产被转移了公司但缓存没清），就会出现列表能看到但详情打不开的情况。

另一个可能：列表页走的是中间件检查，详情页走的是控制器检查，两者检查的 ability 不同。

---

**Q: 我给用户加了 admin 角色，为什么还是被拒绝？**
A: 检查调用路径：
1. 如果调用路径是 `Gate::allows('assets.edit')` → 只走前两层，`Gate::before()` 检查 superuser，然后找 `Gate::define('assets.edit')`（没定义），所以返回 false
2. 如果调用路径是 `$this->authorize('update', $asset)` → 走完整7层，`Policy::before()` 会检查 admin 并放行

**关键**：admin 角色的放行逻辑在 `Policy::before()` 中，只有走到 Policy 层才会生效。如果直接调用 `Gate::allows('assets.edit')` 而不传模型，是走不到 Policy 层的。

正确写法：`Gate::allows('update', $asset)` 或 `$user->can('update', $asset)`，传模型参数才能触发完整检查链。
