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
