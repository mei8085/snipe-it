# Snipe-IT 跨公司/跨地点数据可见性机制分析

## 一、核心结论：三层协作的切片架构

Snipe-IT 的数据可见性切片并非由单一层负责，而是 **模型全局作用域（Model Global Scope）+ 用户上下文（User Context）+ 控制器查询层（Query Layer）** 三层协作的结果，并辅以 **Policy 授权层** 作为单对象访问的第二道防线。

| 层 | 职责 | 机制 |
|---|---|---|
| 模型全局作用域 | 自动为所有 Eloquent 查询注入 `WHERE company_id IN (...)` | `CompanyableTrait` → `CompanyableScope` → `Company::scopeCompanyables()` |
| 用户上下文 | 提供当前用户所属的公司 ID 列表，作为作用域的参数 | `Company::getCurrentUserCompanyIds()` 读取 `company_user` 中间表 |
| 控制器查询层 | 对下拉选择等场景做额外约束 | 控制器中显式调用 `Company::scopeCompanyables()` / `Company::getIdsForCurrentUser()` |
| Policy 授权层 | 对单对象操作（view/edit/delete）做二次校验 | `SnipePermissionsPolicy::before()` → `Company::isCurrentUserHasAccess()` |

---

## 二、全局开关：FMCS

所有公司切片逻辑的前提是系统设置中的 **Full Multiple Company Support（FMCS）** 开关，存储在 `settings` 表的 `full_multiple_companies_support` 字段。

- **FMCS 关闭**：所有公司切片逻辑均为 no-op，任何用户可看到全部数据。
- **FMCS 开启**：切片逻辑生效，非超级用户只能看到自己所属公司的数据。

判断方法位于 [Company.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L118-L128)：

```php
public static function isFullMultipleCompanySupportEnabled()
{
    $settings = Setting::getSettings();
    if (is_null($settings)) {
        return false;
    } else {
        return $settings->full_multiple_companies_support == 1;
    }
}
```

另一个关键设置是 `null_company_is_floater`（浮动模式）——决定没有公司的对象是否对所有用户可见。

---

## 三、模型全局作用域：自动注入 WHERE 条件

### 3.1 CompanyableTrait —— 直接拥有 company_id 的模型

[CompanyableTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Traits/CompanyableTrait.php#L10-L61) 在模型 boot 时注册一个 Eloquent Global Scope：

```php
public static function bootCompanyableTrait()
{
    static::addGlobalScope(new CompanyableScope);
}
```

使用此 Trait 的模型（即**直接拥有 `company_id` 列**的模型）：

| 模型 | 文件 |
|---|---|
| Asset | [Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Asset.php#L39) |
| User | [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/User.php#L37) |
| License | [License.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/License.php) |
| Accessory | [Accessory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Accessory.php) |
| Component | [Component.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Component.php) |
| Consumable | [Consumable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Consumable.php) |
| ConsumableAssignment | [ConsumableAssignment.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/ConsumableAssignment.php) |
| Location | [Location.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Location.php#L22) |
| Department | [Department.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Department.php) |
| Company | [Company.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L28) |
| Actionlog | [Actionlog.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Actionlog.php#L27) |

### 3.2 CompanyableChildTrait —— 没有自己的 company_id，通过父模型继承

[CompanyableChildTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Traits/CompanyableChildTrait.php#L8-L17) 注册 `CompanyableChildScope`，通过 `whereHas` 对父模型做公司过滤：

```php
public static function bootCompanyableChildTrait()
{
    static::addGlobalScope(new CompanyableChildScope);
}
```

使用此 Trait 的模型需实现 `ICompanyableChild` 接口的 `getCompanyableParents()` 方法：

| 模型 | 父模型关系 | 文件 |
|---|---|---|
| LicenseSeat | `['asset', 'license']` | [LicenseSeat.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/LicenseSeat.php#L67-L70) |
| Maintenance | `['asset']` | [Maintenance.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Maintenance.php#L110-L113) |

---

## 四、核心切片引擎：Company::scopeCompanyables()

### 4.1 入口分发

[CompanyableScope::apply()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/CompanyableScope.php#L23-L26) 只是简单委托：

```php
public function apply(Builder $builder, Model $model)
{
    return Company::scopeCompanyables($builder);
}
```

### 4.2 scopeCompanyables() —— 三路分发

[Company::scopeCompanyables()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L372-L380) 首先判断是否需要切片：

```php
public static function scopeCompanyables($query, $column = 'company_id', $table_name = null)
{
    if (!self::isFullMultipleCompanySupportEnabled()
        || (Auth::hasUser() && auth()->user()->isSuperUser())
        || (!Auth::hasUser())) {
        return $query;  // 不切片：FMCS 关闭 / 超级用户 / 命令行无用户
    } else {
        return self::scopeCompanyablesDirectly($query, $column, $table_name);
    }
}
```

**跳过切片的三种情况：**
1. FMCS 未开启
2. 当前用户是超级用户（SuperUser）
3. 没有登录用户（CLI 场景）

### 4.3 scopeCompanyablesDirectly() —— 实际的 SQL 注入

[Company::scopeCompanyablesDirectly()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L388-L466) 是真正的切片实现，按目标表分三种策略：

#### 策略一：companies 表自身

```php
if ($query->getModel()->getTable() == 'companies') {
    if (empty($companyIds)) {
        return $query->whereNull('companies.id');  // 无公司 → 什么也看不到
    }
    return $query->whereIn('companies.id', $companyIds);
}
```

#### 策略二：users 表（通过 company_user 中间表过滤）

用户与公司是多对多关系（`company_user` 中间表），不能简单用 `company_id` 过滤：

```php
if ($query->getModel()->getTable() == 'users') {
    if (empty($companyIds)) {
        if ($floater) return $query;  // 浮动模式：看到所有人
        return $query->whereNotIn('users.id', function ($sub) {
            $sub->select('user_id')->from('company_user');
        });  // 非浮动：只看到没有公司关联的用户
    }

    if ($floater) {
        return $query->where(function ($q) use ($companyIds) {
            $q->whereIn('users.id', function ($sub) use ($companyIds) {
                $sub->select('user_id')->from('company_user')->whereIn('company_id', $companyIds);
            })->orWhereDoesntHave('companies');  // 本公司用户 + 无公司用户
        });
    }

    return $query->whereIn('users.id', function ($sub) use ($companyIds) {
        $sub->select('user_id')->from('company_user')->whereIn('company_id', $companyIds);
    });  // 只看到本公司的用户
}
```

#### 策略三：普通 companyable 表（通过 company_id 列过滤）

```php
if (Schema::hasColumn($query->getModel()->getTable(), $column)) {
    if (empty($companyIds)) {
        if ($floater) return $query;              // 浮动：看全部
        return $query->whereNull($table.$column);  // 非浮动：只看无公司的
    }

    // action_logs 特殊：NULL company_id 表示全局对象日志，不隐藏
    if ($query->getModel()->getTable() === 'action_logs') {
        return $query->where(function ($q) use ($table, $column, $companyIds) {
            $q->whereIn($table.$column, $companyIds)
              ->orWhereNull($table.$column);
        });
    }

    // 浮动模式：本公司 + 无公司
    if ($floater) {
        return $query->where(function ($q) use ($table, $column, $companyIds) {
            $q->whereIn($table.$column, $companyIds)
              ->orWhereNull($table.$column);
        });
    }

    return $query->whereIn($table.$column, $companyIds);  // 严格：只看本公司
}
```

### 4.4 用户公司 ID 的获取 —— 避免无限递归

[Company::getCurrentUserCompanyIds()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L106-L116) 直接查询 `company_user` 中间表，**刻意绕过 Eloquent 关系**：

```php
private static function getCurrentUserCompanyIds(): array
{
    if (!Auth::hasUser()) return [];
    return DB::table('company_user')
        ->where('user_id', auth()->id())
        ->pluck('company_id')
        ->toArray();
}
```

原因：如果用 `$user->companies`，会触发 Company 模型的 `CompanyableScope`，又调回 `getCurrentUserCompanyIds()`，形成无限递归。

---

## 五、CompanyableChildScope：间接继承的公司过滤

[CompanyableChildScope::apply()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/CompanyableChildScope.php#L23-L28) 通过 `whereHas` 沿父模型关系做切片：

```php
public function apply(Builder $builder, Model $model)
{
    $model = $builder->getModel();
    return Company::scopeCompanyableChildren($model->getCompanyableParents(), $builder);
}
```

[Company::scopeCompanyableChildren()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L495-L519) 对所有父关系做 `orWhereHas`：

```php
$q = $query->where(function ($q) use ($companyable_names, $f) {
    $q2 = $q->whereHas($companyable_names[0], $f);
    for ($i = 1; $i < count($companyable_names); $i++) {
        $q2 = $q2->orWhereHas($companyable_names[$i], $f);
    }
});
```

例如 `LicenseSeat` 的 `getCompanyableParents()` 返回 `['asset', 'license']`，意味着：只要 license seat 关联的 asset 或 license 属于当前用户的公司，该 seat 就可见。

---

## 六、Policy 授权层：单对象访问的第二道防线

### 6.1 SnipePermissionsPolicy::before()

[SnipePermissionsPolicy::before()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Policies/SnipePermissionsPolicy.php#L38-L73) 在每次 Policy 检查前执行：

```php
public function before(User $user, $ability, $item)
{
    // 1. admin 权限 → 允许（但仍受 FMCS 作用域约束查询结果）
    if ($user->hasAccess('admin')) return true;

    // 2. 如果 $item 不是模型实例，不在此拦截
    if (!$item instanceof Model) return;

    // 3. 公司访问检查
    if (!Company::isCurrentUserHasAccess($item)) return false;
}
```

### 6.2 Company::isCurrentUserHasAccess()

[Company::isCurrentUserHasAccess()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L190-L256) 对单个对象做访问检查：

- **FMCS 关闭** → 直接放行
- **超级用户** → 直接放行
- **目标表无 company_id 列**（如 locations） → 放行
- **User 对象** → 通过 `company_user` 中间表做交集判断
- **Company 对象** → 检查 company.id 是否在用户公司列表中
- **其他对象** → 检查 `company_id` 是否在用户公司列表中

```php
$companyable_company_id = ($companyable instanceof Company)
    ? $companyable->id
    : $companyable->company_id;
return in_array($companyable_company_id, $userCompanyIds);
```

### 6.3 Gate::before() —— 超级用户终极放行

[AuthServiceProvider](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Providers/AuthServiceProvider.php#L112-L122) 中注册的全局 Gate 优先级最高：

```php
Gate::before(function ($user, $ability) {
    if ($user->isSuperUser()) return true;
});
```

超级用户在 **查询层** 和 **Policy 层** 都被放行，不受 FMCS 限制。

---

## 七、控制器查询层：手动切片场景

控制器中有些场景需要手动调用 `Company` 的切片方法：

### 7.1 下拉选择列表

[Api\AssetsController](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/AssetsController.php) 在 select2 列表中显式调用：

```php
$assets = Company::scopeCompanyables($assets);
```

[Api\UsersController](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/UsersController.php) 对用户选择列表：

```php
$users = Company::scopeCompanyables($users, 'company_id', 'users');
```

并支持前端传 `companyId` 做进一步收窄：

```php
if ((Setting::getSettings()->full_multiple_companies_support == '1') && $request->filled('companyId')) {
    $companyIds = array_values(array_filter(array_map('intval', explode(',', $request->input('companyId')))));
    if (!empty($companyIds)) {
        $assets->whereIn('assets.company_id', $companyIds);
    }
}
```

### 7.2 创建/更新时约束 company_id 赋值

[Company::getIdForCurrentUser()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L148-L182) 确保非超级用户不能把对象分配到自己不属的公司：

```php
$asset->company_id = Company::getIdForCurrentUser($request->input('company_id'));
```

逻辑：
1. 超级用户：可以设置任意 company_id
2. 普通用户：只能设置自己所属公司的 ID，否则抛 `ValidationException`

### 7.3 用户公司同步

[Company::getIdsForCurrentUser()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Company.php#L263-L278) 过滤请求的公司 ID 列表，只保留当前用户所属的：

```php
$user->syncCompaniesWithLogging(Company::getIdsForCurrentUser($companyIds));
```

---

## 八、Actionlog 的特殊处理

[Actionlog](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Actionlog.php#L126-L143) 使用 `CompanyableTrait`，但有特殊逻辑：

### 8.1 创建时自动填充 company_id

```php
static::creating(function (self $actionlog): void {
    if (!array_key_exists('company_id', $actionlog->getAttributes())) {
        $actionlog->company_id = static::resolveCompanyIdFromAttributes(...);
    }
});
```

优先从 item（资产/许可证）取 `company_id`，因为 FMCS 可见性基于物品归属方，而非 checkout 目标。如果 item 无 `company_id`，再回退到 target。

### 8.2 查询时 NULL company_id 不隐藏

在 `scopeCompanyablesDirectly()` 中，`action_logs` 表的 NULL `company_id` 日志条目不隐藏——因为它们记录的是全局对象（如 AssetModel、Company）的操作日志。

---

## 九、Location 的公司归属

[Location](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Location.php) 同样使用 `CompanyableTrait` 并拥有 `company_id` 列。但 `isCurrentUserHasAccess()` 中有特殊处理：

```php
if (!Schema::hasColumn($company_table, 'company_id')) {
    return true;  // 无 company_id 列的表 → 放行
}
```

这主要是为了 `SnipePermissionsPolicy::before()` 在 `location->isDeletable()` 中的兼容——Location 有 `company_id`，所以 FMCS 开启时 Location 也受公司约束。

---

## 十、完整数据流图

```
请求进入
  │
  ├─ Gate::before() ──────────── 超级用户？ → 全部放行
  │
  ├─ Controller 查询（如 Asset::all()）
  │    │
  │    └─ Eloquent Builder 触发 Global Scope
  │         │
  │         └─ CompanyableScope::apply()
  │              │
  │              └─ Company::scopeCompanyables($query)
  │                   │
  │                   ├─ FMCS 关闭？ → 不切片，返回原查询
  │                   ├─ 超级用户？ → 不切片
  │                   ├─ 无登录用户？ → 不切片（CLI）
  │                   │
  │                   └─ Company::scopeCompanyablesDirectly()
  │                        │
  │                        ├─ 获取用户公司 IDs（getCurrentUserCompanyIds）
  │                        │     └─ DB::table('company_user')->where('user_id', ...)
  │                        │
  │                        ├─ companies 表 → whereIn('id', $companyIds)
  │                        ├─ users 表 → whereIn via company_user pivot
  │                        └─ 其他表 → whereIn('company_id', $companyIds)
  │                             └─ 浮动模式追加 orWhereNull('company_id')
  │
  ├─ Controller 显式调用
  │    ├─ Company::scopeCompanyables()  —— select2 列表
  │    ├─ Company::getIdForCurrentUser()  —— 创建/更新时约束
  │    └─ Company::getIdsForCurrentUser()  —— 同步用户公司
  │
  └─ Policy 检查（view/edit/delete 单对象）
       │
       └─ SnipePermissionsPolicy::before()
            │
            └─ Company::isCurrentUserHasAccess($item)
                 ├─ 超级用户 → true
                 ├─ FMCS 关闭 → true
                 ├─ User 对象 → pivot 交集判断
                 └─ 其他 → company_id ∈ 用户公司列表？
```

---

## 十一、关键发现与注意事项

1. **查询层切片是隐式的**：只要模型用了 `CompanyableTrait`，所有查询自动被作用域约束，开发者无需手动加条件。这是最主要的切片入口。

2. **Policy 层是显式的第二道防线**：防止通过 ID 直接访问不属于自己公司的对象（如 `/hardware/123`），即使该 ID 不在列表查询结果中。

3. **User 模型有无限递归风险**：User 使用 `CompanyableTrait` 但公司关系通过 `company_user` 中间表解析，`getCurrentUserCompanyIds()` 必须绕过 Eloquent 直接查 DB 以避免递归。

4. **`null_company_is_floater` 设置影响巨大**：开启时，无公司的对象对所有用户可见；关闭时，无公司对象只对无公司用户可见。

5. **Actionlog 的 NULL company_id 是有意的**：全局对象（如 AssetModel、Category）的日志记录 company_id 为 NULL，查询时不隐藏这些记录。

6. **Location 有 company_id 但不影响物理位置层级**：Location 的 `parent_id` 层级关系不受公司切片影响，但 Location 本身的可见性受 FMCS 约束。

7. **超级用户完全不受 FMCS 限制**：在查询层和 Policy 层都被放行，可以跨公司看到所有数据。
