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

这是最容易混淆的地方。**代码实际情况与之前的分析有重大差异**，让我们通过代码逐行验证：

#### ⚠️ 重要修正：代码中根本没有 `Gate::define('superuser')`！

```bash
# 实际搜索结果
grep -rn "Gate::define.*superuser" app/
# → 无任何结果！
```

`superuser` 权限完全依赖 `Gate::before()` 钩子中的 `$user->isSuperUser()` 检查，这是理解整个系统的关键。

---

#### Laravel 授权系统的真实执行顺序（6 层判定漏斗 + 短路机制）

```
调用 Gate::allows($ability, $arguments)
        ↓
【第1层】 Gate::before() 全局前置钩子 ⭐
    在 AuthServiceProvider.php:112-122 定义
    ├─ 演示模式检查（editableOnDemo）
    │   ├─ true → 返回 false ❌ 直接拒绝，短路！后面所有层都不执行
    │   └─ false → 继续
    ├─ 超级管理员检查 $user->isSuperUser()
    │   ├─ true → 返回 true ✅ 直接放行，短路！后面所有层都不执行
    │   └─ false → 继续
    └─ 返回 null → 继续
        ↓
【第2层】 解析 $arguments，查找绑定的 Policy 类
    检查第一个参数是否为模型类名/实例
    ├─ 有模型 & 有 Policy 注册 → 进入 Policy 流程（第3-4层）
    └─ 无模型 / 无 Policy → 跳过 Policy，直接去第5层
        ↓
【第3层】 Policy::before() 策略级前置钩子 ⚡
    在 SnipePermissionsPolicy.php:38-73 定义
    ├─ admin 角色检查 $user->hasAccess('admin')
    │   ├─ true → 返回 true ✅ 短路！跳过公司检查和 Policy 具体方法
    │   └─ false → 继续
    ├─ 检查是否为 Model 实例
    │   ├─ 不是实例 → 返回 null（跳过公司检查），继续
    │   └─ 是实例 → 继续
    ├─ 公司范围检查 Company::isCurrentUserHasAccess()
    │   ├─ false → 返回 false ❌ 短路！拒绝访问，Policy 具体方法不执行
    │   └─ true → 继续
    └─ 返回 null → 继续
        ↓
【第4层】 Policy 具体方法执行
    如 view(), create(), update(), delete(), audit() 等
    ├─ 有方法 → 调用该方法 → 通常返回 $user->hasAccess('xxx.yyy')
    │   ├─ true → ✅ 通过
    │   └─ false/null → 继续
    └─ 无方法 → 返回 null，继续
        ↓
【第5层】 检查是否有直接的 Gate::define() 定义
    在 AuthServiceProvider.php:130-275 定义了 'admin', 'import', 'reports.view' 等
    ├─ 有匹配的 Gate → 执行该闭包 → 返回 true/false → 终止
    └─ 无匹配 → 继续
        ↓
【第6层】 兜底
    → 如果以上所有层都没返回 true，返回 false ❌
```

> **关键顺序修正**：Policy 检查（第3-4层）在 Gate::define 检查（第5层）**之前**！不是之后！
>
> **关键短路修正**：任何一层返回 `true` 或 `false`（非 null），都会**立即终止**后续所有检查！

---

#### ⚡ 短路行为详解：Policy::before 返回 true 后，后续策略方法不会执行！

这是最关键的一点。`Policy::before()` 返回 `true` 后，**不仅跳过公司检查，连 Policy 的具体方法（如 view/update/delete）也不会执行**。

**Snipe-IT 代码中的证据**（`SnipePermissionsPolicy.php:57-60` 注释）：
```php
/**
 * If we got here by $this->authorize('something', $actualModel) then we can continue on, but if we got here
 * via $this->authorize('something', Model::class) then calling Company::isCurrentUserHasAccess($item) gets weird.
 * Bail out here by returning "nothing" and allow the relevant method lower in this class to be called and handle authorization.
 */
if (! $item instanceof Model) {
    return;  // 返回 null，继续执行下面的方法
}
```

注释明确说明：**返回 "nothing"（即 null）才会继续执行下面的方法**。反过来，如果返回 `true` 或 `false`，下面的方法就不会被调用。

**Laravel 官方文档证据**：
> "If the `before` callback returns a non-null result that result will be considered the result of the check."
>
> "If `null` is returned, the authorization will fall through to the policy method."

---

### 🔥 Gate::before vs Policy::before：返回值语义完全对齐

| 返回值 | Gate::before() 后果 | Policy::before() 后果 | 行为一致性 |
|-------|--------------------|----------------------|-----------|
| **`true`** | ✅ 通过，**短路**<br>所有后续层（包括 Policy 所有方法、Gate::define）都不执行 | ✅ 通过，**短路**<br>公司检查、Policy 具体方法、Gate::define 都不执行 | 💯 完全一致 |
| **`false`** | ❌ 拒绝，**短路**<br>所有后续层都不执行 | ❌ 拒绝，**短路**<br>Policy 具体方法、Gate::define 都不执行 | 💯 完全一致 |
| **`null`** | ➡️ 继续执行下一层 | ➡️ 继续执行 Policy 具体方法 | 💯 完全一致 |

---

### 经典场景分析：admin 角色的完整短路路径

```
$this->authorize('update', $asset)
    ↓
【第1层】Gate::before()
    ├─ 演示模式？→ 否
    ├─ isSuperUser()？→ 否（是 admin 不是 superuser）
    └─ 返回 null → 继续
        ↓
【第2层】有 $asset 模型，有 AssetPolicy 注册 → 进入 Policy
        ↓
【第3层】Policy::before()
    ├─ $user->hasAccess('admin')？→ true
    │   └─ 返回 true ✅ **短路！**
    ├─ （公司范围检查被跳过）
    └─ （Policy::update() 方法永不执行）
        ↓
✅ 授权通过，执行业务逻辑
```

**重要结论**：admin 角色在 `Policy::before()` 返回 `true` 后，**后面的一切检查（包括公司范围检查、assets.edit 权限位检查）全部被跳过**。这就是为什么 admin 可以操作所有公司的资产——不是跳过了公司检查，而是**公司检查根本没机会执行**。

---

### 经典场景分析：公司范围不匹配的拒绝路径

```
$this->authorize('view', $asset)  // 普通用户，非 superuser，非 admin
    ↓
【第1层】Gate::before() → 返回 null
        ↓
【第2层】有模型，有 Policy → 继续
        ↓
【第3层】Policy::before()
    ├─ isSuperUser()？→ 否
    ├─ isAdmin()？→ 否
    ├─ 是 Model 实例？→ 是
    ├─ Company::isCurrentUserHasAccess($item)？→ false
    │   └─ 返回 false ❌ **短路！**
    └─ （Policy::view() 方法永不执行）
        ↓
❌ 授权失败，403
```

**重要结论**：公司范围检查在 `Policy::before()` 中返回 `false` 后，**具体的 `assets.view` 权限位检查根本没机会执行**。即使该用户有 `assets.view` 权限，只要公司不匹配，就会被拒绝。

---

**关键代码证据**：

1. **第1层：Gate::before() 全局钩子**，代码位置 `AuthServiceProvider.php:112-122`：
```php
Gate::before(function ($user, $ability) {
    // 演示模式特殊限制（优先级最高，连 superuser 都受约束）
    if (($ability == 'editableOnDemo') && (config('app.lock_passwords'))) {
        return false;
    }
    // superuser 直接放行 ALL
    if ($user->isSuperUser()) {
        return true;
    }
    // 返回 null，继续后续检查
});
```
> **核心机制**：`Gate::allows('superuser')` 完全靠这里工作！因为没有 `Gate::define('superuser')`，整个检查流程在这一层就被处理了。

2. **第3层：Policy::before() 策略级钩子**，代码位置 `SnipePermissionsPolicy.php:38-73`：
```php
public function before(User $user, $ability, $item)
{
    // admin 角色放行（注意：只有传了模型参数，走到 Policy 层才会执行这里！）
    if ($user->hasAccess('admin')) {
        return true;
    }
    // 传类名（非实例）时，跳过公司检查
    if (! $item instanceof Model) {
        return;
    }
    // 多公司范围检查
    if (! Company::isCurrentUserHasAccess($item)) {
        return false;
    }
}
```

3. **第5层：Gate::define() 明确定义**，代码位置 `AuthServiceProvider.php:170-174`：
```php
Gate::define('admin', function ($user) {
    return $user->hasAccess('admin');
});

Gate::define('import', function ($user) {
    return $user->hasAccess('import');
});
// ... 还有 'reports.view', 'activity.view', 'self.xxx' 等约 15 个
```
> 注意：**没有** 'superuser' / 'superadmin' 的 define！

---

**三大钩子的触发条件对比表**：

| 钩子 | 触发条件 | 不触发条件 | 返回值语义 |
|------|---------|-----------|-----------|
| **Gate::before()** | ✅ 总是触发，对 ALL Gate 检查生效 | ❌ 从不（除非抛异常） | `true`=通过，**短路**<br>`false`=拒绝，**短路**<br>`null`=继续 |
| **Policy::before()** | ✅ 传了模型参数（类名/实例）<br>✅ 该模型有 Policy 注册<br>✅ Gate::before() 返回了 null | ❌ 没传模型参数<br>❌ 没有 Policy 注册<br>❌ Gate::before() 已返回 true/false | `true`=通过，**短路**<br>`false`=拒绝，**短路**<br>`null`=继续 |
| **Gate::define()** | ✅ ability 字符串精确匹配<br>✅ 前面所有层都返回了 null | ❌ ability 不匹配<br>❌ 前面已有层返回 true/false | `true`=通过，**短路**<br>`false`=拒绝，**短路**<br>`null`=继续 |

---

**判定顺序总结表（含短路标记）**：

| 层级 | 执行位置 | 执行时机 | 返回true | 返回false | 返回null | 典型检查 |
|------|---------|---------|---------|-----------|----------|---------|
| 1. **Gate::before()** | 全局 | 最先 | 🔴 终止，✅ | 🔴 终止，❌ | ➡️ 继续 | superuser 豁免、演示模式 |
| 2. Policy 查找 | 内核 | Gate::before 之后 | - | - | - | 解析模型参数 |
| 3. **Policy::before()** | 策略类 | 有模型参数时 | 🔴 终止，✅ | 🔴 终止，❌ | ➡️ 继续 | admin 放行、公司范围 |
| 4. **Policy 方法** | 策略类 | Policy::before 之后 | 🔴 终止，✅ | 🔴 终止，❌ | ➡️ 继续 | assets.edit, assets.view |
| 5. **Gate::define()** | 全局 | 最后 | 🔴 终止，✅ | 🔴 终止，❌ | ➡️ 继续 | admin, import, reports.view |
| 6. 兜底 | 内核 | 所有层都没返回 | - | ❌ | - | 默认拒绝 |

> 🔴 标记表示该返回值会**短路**（short-circuit），后续所有层都不再执行。

> **致命区别**：`Gate::before()` 对 ALL 检查生效；`Policy::before()` 只对传了模型参数的检查生效。
>
> 例如：`Gate::allows('import')` 只会走第 1、5、6 层，永远不会走到 Policy 层。

---

### 二、控制器授权 vs 路由中间件：触发路径分流

**同一个权限检查，从不同入口调用，走的代码路径完全不同**。这是理解系统的核心。

---

#### ⚠️ 重要修正：`authorize:superuser` vs `authorize:superadmin` 为何效果相同，但原因不是"ability 被忽略"

代码中同时存在两种写法，效果相同，但之前"ability 参数被忽略"的说法是**错误**的。正确的理解是：

**Gate::before() 对不同 ability 的处理**（`AuthServiceProvider.php:112-122`）：
```php
Gate::before(function ($user, $ability) {
    // 只有 editableOnDemo 做了特殊处理
    if (($ability == 'editableOnDemo') && (config('app.lock_passwords'))) {
        return false;
    }
    // 对所有其他 ability：superuser 返回 true，非 superuser 返回 null（继续）
    if ($user->isSuperUser()) {
        return true;
    }
    // 非 superuser 的非 editableOnDemo：返回 null，继续后续检查
});
```

**`authorize:superuser` 和 `authorize:superadmin` 效果相同的真正原因**：
- 两者都**没有**对应的 `Gate::define()`
- superuser 在 `Gate::before()` 中通过（返回 true）
- 非 superuser 在 `Gate::before()` 返回 null → 继续查找 `Gate::define()` → 找不到 → 兜底返回 false
- 所以两者对非 superuser 都被拒绝，效果相同

**但 ability 参数并没有被忽略！** 如果用 `authorize:admin`（虽然代码中没这样用）：
- 有 `Gate::define('admin')`
- superuser 在 `Gate::before()` 通过
- 非 superuser 但有 admin 权限位的用户 → `Gate::define('admin')` 返回 true → **可以通过！

这与 `authorize:superuser` 的效果完全不同。

---

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
            Gate::allows($section)  //$section = 'superuser'
                ↓
                【只走第1、5、6层】
                第1层：Gate::before()
                    ├─ $ability == 'editableOnDemo' 且 演示模式 → return false ❌
                    ├─ $user->isSuperUser() → return true ✅
                    └─ 其他 → return null（继续）
                第5层：查找 Gate::define($section)
                    ├─ 有 define → 执行闭包 → 返回 true/false
                    └─ 无 define → 继续
                第6层：兜底 → return false ❌
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

---

### 🔥 中间件传入不同 ability 的完整行为矩阵

| 中间件写法 | Gate::define 存在？ | superuser | 非 superuser 有权限位 | 最终效果 |
|-----------|-----------------|----------|---------------------|---------|
| `authorize:superuser` | ❌ 无 | ✅ 通过 | ❌ 拒绝（无 define） | 只有 superuser 通过 |
| `authorize:superadmin` | ❌ 无 | ✅ 通过 | ❌ 拒绝（无 define） | 只有 superuser 通过 |
| `authorize:admin` | ✅ 有 | ✅ 通过 | 有 `admin` 权限位 → ✅ 通过 | superuser + admin 用户通过 |
| `authorize:import` | ✅ 有 | ✅ 通过 | 有 `import` 权限位 → ✅ 通过 | superuser + import 用户通过 |
| `authorize:reports.view` | ✅ 有 | ✅ 通过 | 有 `reports.view` 权限位 → ✅ 通过 | superuser + reports 用户通过 |
| `authorize:任意未定义字符串` | ❌ 无 | ✅ 通过 | ❌ 拒绝（无 define） | 只有 superuser 通过 |

**关键结论**：ability 参数**没有**被忽略，它决定了非 superuser 是否能通过。`superuser` 和 `superadmin` 效果相同，是因为两者都没有对应的 `Gate::define()`，而非 ability 被忽略。

---

**代码中实际使用的中间件只有两种**（都属于"无 define"类）：
- `authorize:superuser`（web.php, api.php）
- `authorize:superadmin`（scim.php）

**特点**：
- ✅ 粗粒度过滤，适合整个路由组
- ❌ 不传模型实例，**不会触发 Policy 层**，更不会触发公司范围检查
- ❌ 对于"有 define"类中间件（如 `authorize:admin`），非 superuser 但有对应权限位的用户可以通过
- ⚠️ 在控制器方法执行**之前**就触发

#### 路径B：控制器内 `$this->authorize()`

**代码路径**：控制器方法 → `AuthorizesRequests trait` → `Gate::authorize()` → 完整 6 层检查

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
                【完整走6层】
                Gate::before() → 找Policy → Policy::before() → Policy::update() → Gate::define()
                （因为传了$asset实例，所以会触发公司范围检查！）
            ↓
            ✅ 放行 → 继续执行控制器逻辑
            ❌ 拒绝 → 抛出 AuthorizationException → 渲染403
```

**典型用法对比**：

| 调用方式 | 代码示例 | 触发公司检查 | 走Policy层 | 走Gate::define | 适用场景 |
|---------|---------|------------|-----------|---------------|---------|
| 传类名 | `$this->authorize('index', Asset::class)` | ❌ 不会 | ✅ 会 | ✅ 最后兜底 | 列表页、创建页 |
| 传实例 | `$this->authorize('view', $asset)` | ✅ 会 | ✅ 会 | ✅ 最后兜底 | 查看、编辑、删除具体资产 |
| 只传实例 | `$this->authorize($asset)` | ✅ 会 | ✅ 会 | ✅ 最后兜底 | 自动推断动作（edit→update） |

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
>
> **admin 角色的特殊注意**：admin 角色的放行逻辑在 `Policy::before()` 中，所以**只有路径B（控制器授权）才会放行 admin 角色**，路径A（中间件）不会放行 admin 角色。

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

- `/admin/settings` 路由有 `authorize:superuser` 中间件 → 走路径A（第1→5→6层）
- `/hardware/1/edit` 控制器调用 `$this->authorize($asset)` → 走路径B（完整6层）

**真实案例**：一个用户有 `admin` 角色但没有 `superuser` 权限位：
- ❌ 访问 `/admin/settings` 被拒绝（中间件 `authorize:superuser` 只走第1层，检查 `isSuperUser()`，不检查 `admin` 角色）
- ✅ 访问 `/hardware/1/edit` 通过（控制器授权走到第3层 `Policy::before()`，检查 `$user->hasAccess('admin')` 放行）

> **关键**：admin 角色的放行逻辑在 `Policy::before()` 中，只有传模型参数时才会触发！中间件不传模型，走不到 Policy 层，所以不会放行 admin 角色。

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

- 中间件 `authorize:admin` 检查的是 `'admin'` 这个 Gate
- 控制器 `$this->authorize('update', $asset)` 检查的是 Policy 的 `update()` 方法 → 内部检查 `'assets.edit'`

**真实案例**：一个用户有 `admin` 全局权限但被在用户级别设置了 `assets.edit = -1`（明确拒绝）：
- ✅ 通过中间件 `authorize:admin` 检查（走第5层 `Gate::define('admin')`）
- ❌ 被控制器授权拒绝（走到第3层 `Policy::before()` 时 admin 放行，但第4层 `Policy::update()` 调用 `$user->hasAccess('assets.edit')` 时，用户级别 `-1` 拒绝优先级更高）

> **关键**：`Policy::before()` 中的 admin 放行只是跳过了公司范围检查，但具体权限方法仍会执行，用户级别的 `-1` 明确拒绝优先级最高！

**典型不一致场景4：superuser vs admin 命名混乱**

代码中存在三种写法，实际效果不同：
| 写法 | 检查内容 | 谁能通过 |
|------|---------|---------|
| `authorize:superuser` | 检查 `isSuperUser()` → `'superuser'` 权限位 | 只有 superuser |
| `authorize:superadmin` | 同上（ability 被忽略） | 只有 superuser |
| `authorize:admin` | 检查 `Gate::define('admin')` → `'admin'` 权限位 | superuser + 有 admin 权限位的用户 |
| `$this->authorize('update', $asset)` | 完整 Policy 检查 | superuser + admin + 有 assets.edit 权限位的用户 |

> **深坑预警**：`routes/scim.php:19` 写的是 `authorize:superadmin`，但实际检查的还是 `'superuser'` 权限位，与 `authorize:superuser` 完全等价。

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

1. **双重 `before` 钩子**：`Gate::before()`（全局）+ `Policy::before()`（策略级），形成灵活的豁免机制，但两者触发条件完全不同
2. **统一的短路语义**：`Gate::before()` 和 `Policy::before()` 的返回值语义 100% 对齐——`true`/`false` 短路，`null` 继续
3. **权限前缀约定**：通过 `columnName()` 方法约定权限前缀（如 `assets` + `.view`），大量减少重复代码
4. **"或"逻辑权限**：如 `manage` 权限是多个权限的 OR 组合，体现了策略层的灵活性
5. **软删除感知**：`delete` 方法中检查 `$item->deleted_at`，已删除的资产不能再删除
6. **公司范围隔离**：多公司模式下，即使有权限，也只能操作本公司的资产，但 admin 角色在 `Policy::before()` 中 `return true` 短路，**公司检查根本没机会执行**
7. **权限优先级**：用户个人权限（尤其是 `-1` 拒绝）优先级高于用户组权限
8. **巧妙的 "无定义" 设计**：`superuser` 没有 `Gate::define()`，完全靠 `Gate::before()` 处理，使得 superuser 豁免逻辑集中在一处
9. **执行顺序反直觉**：Policy 检查在 Gate::define 之前，而不是之后，这是 Laravel 内核的默认行为
10. **命名不统一陷阱**：`authorize:superuser` 和 `authorize:superadmin` 实际效果完全相同，都是检查 `'superuser'` 权限位
11. **admin 角色的双重性**：admin 角色的放行逻辑只在 Policy 层生效，中间件检查 `authorize:admin` 看的是 `'admin'` 权限位，两者不是一回事
12. **传参决定一切**：传 `Asset::class` 还是 `$asset` 实例，决定了是否触发公司范围检查，也决定了 admin 角色豁免是否生效

---

## 常见疑问解答

---

### 🔌 短路行为专题

**Q: Policy::before 返回 true 后，还会执行后续的策略方法吗？**
A: **绝对不会！** 这是最关键的短路行为。

根据 Laravel 官方文档和代码注释双重验证：
- `Policy::before()` 返回 **`true`** → ✅ 通过，**短路**，后续的公司范围检查、Policy 具体方法（view/update/delete）、Gate::define 全部不执行
- `Policy::before()` 返回 **`false`** → ❌ 拒绝，**短路**，后续所有检查都不执行
- `Policy::before()` 返回 **`null`**（或 `return;` 无值）→ ➡️ 继续执行后续检查

**Snipe-IT 代码中的直接证据**（`SnipePermissionsPolicy.php:57-60` 注释）：
> "Bail out here by returning 'nothing' and allow the relevant method lower in this class to be called and handle authorization."

翻译：只有返回"nothing"（即 null）时，才允许调用下面的方法。反过来，返回 `true` 或 `false` 时，下面的方法不会被调用。

**实际影响**：
- admin 角色在 `Policy::before()` 返回 `true` → 公司范围检查、`assets.edit` 权限位检查，全部跳过
- 公司范围不匹配在 `Policy::before()` 返回 `false` → `assets.view` 权限位检查，跳过

---

**Q: Gate::before 和 Policy::before 的短路行为完全一样吗？**
A: **是的，语义 100% 对齐**，只是作用范围不同：

| 返回值 | Gate::before() 后果 | Policy::before() 后果 |
|-------|--------------------|----------------------|
| **`true`** | ✅ 通过，短路，所有后续层都不执行 | ✅ 通过，短路，Policy 方法 + 后续层都不执行 |
| **`false`** | ❌ 拒绝，短路，所有后续层都不执行 | ❌ 拒绝，短路，Policy 方法 + 后续层都不执行 |
| **`null`** | ➡️ 继续下一层 | ➡️ 继续执行 Policy 方法 |

---

**Q: 为什么管理员能看到所有资产，但普通用户只能看到本公司的？**
A: 在 `SnipePermissionsPolicy::before()` 中，`admin` 角色直接 `return true` 跳过了公司检查。

**Q: 超级管理员和管理员有什么区别？**
A: 两者的放行层面不同，短路时机不同：
1. **超级管理员（superuser）**：在第1层 `Gate::before()` 就返回 `true` 短路，**所有后续层都不执行**，不受任何约束
2. **管理员（admin）**：在第3层 `Policy::before()` 返回 `true` 短路，**跳过公司范围检查和具体权限位检查**，但只对传了模型参数的检查生效

> **修正之前的错误**：admin 不是"仍受多公司范围约束"，恰恰相反——admin 在 `Policy::before()` 返回 `true` 后，**公司范围检查根本没机会执行**，所以 admin 可以操作所有公司的资产。

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
1. **作用范围不同**：`Gate::before()` 是全局的，对 ALL 权限检查生效；`Policy::before()` 只对传了模型参数的检查生效
2. **执行顺序不同**：`Gate::before()` 是第1层（最先），`Policy::before()` 是第3层，中间隔了 Policy 查找（第2层），而 `Gate::define()` 在第5层（最后）
3. **豁免机制不同**：超级管理员在 `Gate::before()` 就被放行，所有后续检查（包括公司范围）都跳过；普通 admin 在 `Policy::before()` 放行，不受公司范围约束

> **经典反例**：`Gate::allows('import')` 只会走第 1→5→6 层，永远不会走到 Policy 层，`Policy::before()` 完全不执行。

---

**Q: 为什么我在路由中间件加了 `authorize:admin`，但控制器里还要再 authorize 一次？**
A: 两者检查的维度完全不同，是互补关系而非重复：
1. **中间件 `authorize:admin`**：检查用户是否有 `'admin'` 权限位，粗粒度过滤，**不传模型**，走第 1→5→6 层，**不会触发 admin 角色在 Policy 层的放行逻辑**（因为走不到 Policy）
2. **控制器 `$this->authorize('update', $asset)`**：检查具体动作权限（如 assets.edit），传模型实例，走完整 6 层，会检查公司范围

**注意一个反直觉的事实**：
- 中间件 `authorize:admin` → 只看 `'admin'` 权限位，不看 admin 角色的 Policy 放行
- 控制器 `$this->authorize('update', $asset)` → 先看 superuser，再看 admin 角色（Policy::before()），最后看 `'assets.edit'` 权限位

所以一个有 admin 角色但没有 `'admin'` 权限位的用户：
- ❌ 通不过中间件 `authorize:admin`（因为没有 'admin' 权限位）
- ✅ 能通过控制器 `$this->authorize('update', $asset)`（因为 Policy::before() 中 admin 角色放行）

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
1. 如果调用路径是 `Gate::allows('assets.edit')` → 只走第 1→5→6 层，`Gate::before()` 检查 superuser，然后找 `Gate::define('assets.edit')`（没定义），所以返回 false
2. 如果调用路径是 `$this->authorize('update', $asset)` → 走完整 6 层，`Policy::before()` 会检查 admin 并放行

**关键**：admin 角色的放行逻辑在 `Policy::before()` 中，只有**传了模型参数**，走到 Policy 层才会生效。如果直接调用 `Gate::allows('assets.edit')` 而不传模型，是走不到 Policy 层的。

正确写法：`Gate::allows('update', $asset)` 或 `$user->can('update', $asset)`，传模型参数才能触发完整检查链。

---

**Q: `authorize:superuser` 和 `authorize:superadmin` 有什么区别？**
A: **完全没有区别！** 这是历史遗留的命名混乱：
- 两者都调用 `Gate::allows($section)`，其中 `$section` 分别是 `'superuser'` 和 `'superadmin'`
- 但两者都**没有**对应的 `Gate::define()`
- 两者都在 `Gate::before()` 中被处理，调用 `$user->isSuperUser()`，该方法检查的是 `'superuser'` 权限位
- `Gate::before()` 中除了 `editableOnDemo` 特殊处理外，**完全不关心 `$ability` 参数是什么**，只检查 `$user->isSuperUser()`

所以无论写 `authorize:superuser`、`authorize:superadmin`、甚至 `authorize:whatever`，只要不是 `editableOnDemo`，实际效果都完全一样——检查 `'superuser'` 权限位。
