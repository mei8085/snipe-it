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

---

## 十二、Asset 的三个 Location 字段与可见性的关系

Asset 模型涉及三个不同的 location 相关字段，各自语义不同，但**对数据可见性本身均不直接影响——它们都受同一个公司切片机制约束**。三个字段的定义见 [Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Asset.php)。

### 12.1 三个字段的语义对比

| 字段 | 关系方法 | 语义 | 何时被赋值 |
|---|---|---|---|
| `location_id` | `location()` (belongsTo Location) | **资产当前所在地**：资产被 checkout 后，从 checkout 目标（User/Location/Asset）继承 location_id | 每次 checkout/checkin 时更新；`checkOut()` 方法第 3 参数就是 location |
| `rtd_location_id` | `defaultLoc()` (belongsTo Location) | **默认部署位置（Ready-To-Deploy）**：资产未分配时的"仓库"位置 | 创建设置或更新时手动指定 |
| `assignedTo` → `location_id` | 沿 `assignedTo()` morphTo 关系访问 | **间接派生位置**：资产被借给 User 时，User 自身的 `location_id` 代表用户办公地点；借给 Location 时就是该 Location 本身 | 不直接存储，通过关联对象读取 |

关键代码：

```php
// location_id - 当前所在地
public function location()
{
    return $this->belongsTo(Location::class, 'location_id');
}

// rtd_location_id - 默认部署位置
public function defaultLoc()
{
    return $this->belongsTo(Location::class, 'rtd_location_id');
}

// assignedTo - 借出目标（多态：User/Location/Asset）
public function assignedTo()
{
    return $this->morphTo('assigned', 'assigned_type', 'assigned_to')->withTrashed();
}
```

### 12.2 Checkout 时 location_id 的传播逻辑

在 [Api\AssetsController::resolveCheckoutTargetForAssetMutation()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/AssetsController.php#L897-L912) 中解析 checkout 目标后，在 [update()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/AssetsController.php#L854-L864) 中传播 location_id：

```php
if ($request->filled('assigned_user')) {
    $location = $target->location_id;       // 从 User 的 location_id 继承
} elseif ($request->filled('assigned_asset')) {
    $location = $target->location_id;       // 从宿主 Asset 的 location_id 继承
} elseif ($request->filled('assigned_location')) {
    $location = $target->id;                 // 直接就是 Location 的 id
}
// 传入 checkOut()，最终写入 assets.location_id
$asset->checkOut($target, auth()->user(), ..., $location);
```

### 12.3 三个字段对可见性的实际影响

**结论：三个 location 字段都不是可见性切片的直接维度。可见性始终以 `assets.company_id` 为唯一依据。**

但 location 字段会通过 **`fmcs_location` 验证规则** 和 **`scope_locations_fmcs` 设置** 间接地约束 location 与 company 的一致性：

1. **Location API 列表切片的独立开关**：[Api\LocationsController::index()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/LocationsController.php#L117-L122) 中，Location 列表是否受 FMCS 切片由 `scope_locations_fmcs` 单独控制：

```php
// scope_locations_fmcs 未开启时，所有 Location 对所有用户可见（不切片）
if (Setting::getSettings()->scope_locations_fmcs) {
    $locations = Company::scopeCompanyables($locations);
}
```

这意味着：**Location 模型虽然使用了 `CompanyableTrait`，但只有当 `scope_locations_fmcs = 1` 时，Location 列表 API 才会被显式二次切片**。不过，由于 Location 用了 `CompanyableTrait`，其他地方（如 `Location::find()` 单个查询、`Asset::with('location')` 预加载）实际上仍然会触发 Global Scope 切片。

2. **`fmcs_location` 验证规则**：在 [ValidationServiceProvider](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Providers/ValidationServiceProvider.php#L366-L384) 中定义，用于在创建/更新 Asset 时验证所选 location 是否属于所选 company：

```php
Validator::extend('fmcs_location', function ($attribute, $value, $parameters, $validator) {
    $settings = Setting::getSettings();
    if ($settings->full_multiple_companies_support == '1' && $settings->scope_locations_fmcs == '1') {
        $data = $validator->getData();
        $companyIds = ...; // 收集 company_id 和 company_ids[]
        $location = Location::find($value);
        if ($location && ! in_array($location->company_id, $companyIds)) {
            return false;  // 校验失败：location 的公司与 asset 的公司不匹配
        }
    }
    return true;
});
```

在 Asset 模型的 `$rules` 中使用了此规则：
```php
'location_id' => ['nullable', 'exists:locations,id', 'fmcs_location'],
'rtd_location_id' => ['nullable', 'exists:locations,id', 'fmcs_location'],
'assigned_location' => ['integer', 'nullable', 'exists:locations,id,deleted_at,NULL', 'fmcs_location'],
```

**即：`location_id`、`rtd_location_id`、`assigned_location` 这三个字段本身不做可见性切片，但当 FMCS + scope_locations_fmcs 同时开启时，它们必须属于与 asset 相同的公司（校验级约束，非查询级约束）。**

### 12.4 Checkout 时的跨公司检查：canCheckoutTo()

[CompanyableTrait::canCheckoutTo()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Traits/CompanyableTrait.php#L35-L60) 决定资产是否可以 checkout 给某目标：

```php
public function canCheckoutTo(Model $target): bool
{
    // FMCS 关闭 → 允许
    if (! $settings->full_multiple_companies_support) return true;

    // 资产本身无公司 → 目标无公司则允许；有公司则取决于浮动模式
    if (! $this->company_id) {
        if (is_null($target->company_id)) return true;
        return (bool) $settings->null_company_is_floater;
    }

    // 目标是 User → 检查 User 的公司是否包含资产的公司
    if ($target instanceof User) {
        return $target->canReceiveFromCompany((int) $this->company_id);
    }

    // 目标无公司 → 取决于浮动模式
    if (is_null($target->company_id)) {
        return (bool) $settings->null_company_is_floater;
    }

    // 目标（Location/Asset）：company_id 必须精确匹配
    return (int) $target->company_id === (int) $this->company_id;
}
```

此方法在 AssetsController 的 checkout 流程中通过 `checkoutCompanyMismatchResponse()` 调用，确保不会把 A 公司的资产借给 B 公司的 User/Location。

---

## 十三、withoutGlobalScope / removeGlobalScope 全仓盘点与绕过场景

代码中显式移除 Global Scope（即绕过公司切片）的场景分为两大类：**功能性绕过（有理由的）** 和 **测试类用途**。以下按类别列出非测试代码中的调用：

### 13.1 功能性绕过：有明确业务理由

| 文件 | 行号 | 调用 | 绕过理由 |
|---|---|---|---|
| [HasUploads.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Traits/HasUploads.php#L16) | L16 | `Actionlog::...->withoutGlobalScope(CompanyableScope::class)` | 父对象（如 AssetModel、Company）无 company_id，其上传日志的 company_id 恒为 null，不绕过会被过滤掉；访问权已由父对象 Policy 把关 |
| [Loggable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Traits/Loggable.php#L49) | L49 | `Actionlog::...->withoutGlobalScope(CompanyableScope::class)` | 同上：对象历史日志中全局对象的记录会被 Global Scope 隐藏 |
| [Actionlog.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Actionlog.php#L199) | L199 | `$modelClass::withoutGlobalScopes()->whereKey($id)` | 创建日志时反向解析 item/target 的 company_id，需要全局视野避免循环依赖 |
| [Component.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Models/Component.php#L322) | L322 | `Asset::...->withoutGlobalScope(new CompanyableScope)` | `unconstrainedAssets()` 关系用于计算组件分配数量（如库存计算），不能因 FMCS 丢失统计准确性 |
| [UserImporter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Importer/UserImporter.php#L172) | L172 | `User::withoutGlobalScopes()->where('username', ...)` | 导入时判断 username 是否已被其他公司占用（用户名全局唯一），否则会误判为新建导致唯一键冲突 |
| [ActionlogsTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Transformers/ActionlogsTransformer.php#L308) | L308 | `Company::withoutGlobalScopes()->withTrashed()->find($id)` | 展示 actionlog 的变更 diff（如公司字段从 A 改到 B）时需要跨公司解析公司名称 |
| [ReportsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/ReportsController.php#L1380-L1387) | L1380-L1387 | `$checkoutableType::withoutGlobalScopes()->find(...)` 及 `License::withoutGlobalScopes()` | 验收报告页按单条 CheckoutAcceptance 关联物品时，需跨公司获取目标以正确渲染；报告本身有 `reports.view` 权限门控 |
| [Api\LicensesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/LicensesController.php#L322) | L322 | `User::withoutGlobalScopes()->find(...)` | License checkout 目标解析：先全局查找目标再显式进行 FMCS 检查，避免"目标不存在"和"目标跨公司"两种错误混同 |
| [Api\LicensesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/LicensesController.php#L338) | L338 | `Asset::withoutGlobalScopes()->find(...)` | 同上：License 分配给 Asset 时的目标解析 |
| [Api\LicenseSeatsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/LicenseSeatsController.php#L112) | L112 | `User::withoutGlobalScopes()->exists()` | 校验 assigned_to 存在性，后续有显式 FMCS 校验 |
| [Api\LicenseSeatsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/LicenseSeatsController.php#L125) | L125 | `Asset::withoutGlobalScopes()->exists()` | 校验 asset_id 存在性 |
| [Api\LicenseSeatsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/LicenseSeatsController.php#L161) | L161 | `User::withoutGlobalScopes()->find(...)` | 跨公司解析目标，便于返回清晰错误信息 |
| [Api\LicenseSeatsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/LicenseSeatsController.php#L179) | L179 | `Asset::withoutGlobalScopes()->find(...)` | 同上 |
| [Api\ComponentsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/ComponentsController.php#L320) | L320 | `Asset::withoutGlobalScopes()->find(...)` | 组件 checkout 目标解析，后续用 Policy + 显式检查 |
| [Api\ConsumablesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/ConsumablesController.php#L313) | L313 | `User::withoutGlobalScopes()->find(...)` | 耗材 checkout 目标解析 |
| [Api\AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/AssetsController.php#L900) | L900 | `User::withoutGlobalScopes()->find(...)` | `resolveCheckoutTargetForAssetMutation`：asset checkout 目标解析，用 `canCheckoutTo()` 做二次校验 |
| [Api\AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/AssetsController.php#L904) | L904 | `Asset::withoutGlobalScopes()->find(...)` | 同上：借给另一个 Asset 的目标解析 |
| [Api\AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/Api/AssetsController.php#L908) | L908 | `Location::withoutGlobalScopes()->find(...)` | 同上：借给 Location 的目标解析 |

**功能性绕过的共同模式：**
- 先 `withoutGlobalScopes()` 全局解析目标对象
- 然后用显式的 FMCS 检查（`canCheckoutTo()` / `Company::isCurrentUserHasAccess()` / 直接比较 `company_id`）确保操作合法
- 目的：区分"对象不存在"与"对象存在但跨公司"两种错误场景，给用户清晰反馈

### 13.2 测试类绕过（4处）

| 文件 | 用途 |
|---|---|
| [StoreAssetWithFullMultipleCompanySupportTest (UI)](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/tests/Feature/Assets/Ui/StoreAssetWithFullMultipleCompanySupportTest.php#L67) | 创建后断言 asset.company_id 被正确设置，需绕开作用域看到被创建对象 |
| [StoreAssetWithFullMultipleCompanySupportTest (API)](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/tests/Feature/Assets/Api/StoreAssetWithFullMultipleCompanySupportTest.php#L34) | 同上 |
| [StoreAssetWithFullMultipleCompanySupportTest (API)](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/tests/Feature/Assets/Api/StoreAssetWithFullMultipleCompanySupportTest.php#L54) | 同上，测试字符串形式的 company_id |
| [CreateAccessoryWithFullMultipleCompanySupportTest (UI)](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/tests/Feature/Accessories/Ui/CreateAccessoryWithFullMultipleCompanySupportTest.php#L31) | 同上，创建 Accessory 后断言 |

---

## 十四、Reports、Console 命令等"隐式绕过"入口

### 14.1 Reports 控制器：隐式依赖 Global Scope，但统计口径仍有风险

[ReportsController](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/ReportsController.php) 中的所有报表查询**都没有显式调用 `withoutGlobalScopes()`**，因此默认依赖模型 Global Scope 进行切片。但是有几点需要注意：

1. **权限门控**：所有报表方法开头都有 `$this->authorize('reports.view')`，本身不区分公司。
2. **Dashboard 统计的特殊性**：[DashboardController](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Http/Controllers/DashboardController.php#L35-L59) 中 Dashboard 只对 `admin` 权限可见（非超级用户被重定向到个人资产页），而 admin 用户在 Policy 层被放行，但查询层仍受 Global Scope 约束——所以 Dashboard 的统计数字实际上是**按当前 admin 用户所属公司切片后的数据**。

Dashboard 代码中唯一显式处理：
```php
$counts['user'] = Company::scopeCompanyables(auth()->user())->count();
```
这里显式调用 `scopeCompanyables` 对当前用户（即 admin 自己）做切片后再 count，确保只看自己公司的用户数。但 Asset 等其他计数直接用 `Asset::count()`，**隐式依赖 Global Scope**。

3. **活动报告（Activity Report）**：`postActivityReport()` 中用 `Actionlog::with(...)->chunk(...)`，由于 Actionlog Global Scope 中 NULL company_id 不隐藏，全局对象的操作日志会被所有 admin 看到。

4. **验收报告的显式绕过**：`assetAcceptanceCsv()` 方法中为了正确渲染 CheckoutAcceptance 关联的物品，用了 `withoutGlobalScopes()`（见第十三节），但受 `reports.view` 权限门控。

### 14.2 Console 命令：天然绕过（无 Auth 用户）

所有 Artisan 命令运行时都没有登录用户（`Auth::hasUser() === false`），根据 `scopeCompanyables()` 的判断逻辑：

```php
if (!Auth::hasUser()) {
    return $query;  // 不切片
}
```

因此 **所有 Console 命令的查询都天然不受 FMCS 公司切片约束**。关键命令清单：

| 命令类 | 命令签名 | 查询操作 | 影响 |
|---|---|---|---|
| [SendCurrentInventoryToUsers](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/SendCurrentInventoryToUsers.php) | `snipeit:user-inventory` | `User::with('assets', ...)->get()` 遍历所有用户并发送邮件 | 邮件按用户实际 checkout 的物品生成，Asset 虽然全局查询但最终只发给所属用户 |
| [SendInventoryAlerts](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/SendInventoryAlerts.php) | `snipeit:inventory-alerts` | `Helper::checkLowInventory()` 查询 Consumable/Accessory/Component/AssetModel/License 的低库存 | 邮件发给全局配置的 `alert_email`，不区分公司，**跨公司汇总** |
| [SendUpcomingAuditReport](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/SendUpcomingAuditReport.php) | `snipeit:upcoming-audits` | `Asset::dueOrOverdueForAudit()->limit(30)` 找待审计资产 | 邮件发给全局配置的 `alert_email`，**跨公司汇总** |
| [SendExpectedCheckinAlerts](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/SendExpectedCheckinAlerts.php) | `snipeit:expected-checkin` | 查询资产到期情况 | 跨公司查询 |
| [SendExpirationAlerts](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/SendExpirationAlerts.php) | `snipeit:expiring-alerts` | 查询许可证到期 | 跨公司查询 |
| [SendAcceptanceReminder](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/SendAcceptanceReminder.php) | `snipeit:acceptance-reminder` | 遍历待验收记录，发给对应用户 | 按记录发给用户本身 |
| [ValidateAssets](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/ValidateAssets.php) | `snipeit:validate-assets` | 全量 Asset 校验 | 跨公司 |
| [SyncAssetLocations](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/SyncAssetLocations.php) | `snipeit:sync-asset-locations` | 全量 Asset 更新 location_id | 跨公司修改 |
| [FixMismatchedAssetsAndLogs](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/FixMismatchedAssetsAndLogs.php) | `snipeit:fix-mismatched-asset-logs` | 修复 asset-log 不一致 | 跨公司 |
| [SyncAssetCounters](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/SyncAssetCounters.php) | `snipeit:recount-assets` | 重新计数 | 跨公司 |
| [LdapSync](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/LdapSync.php) | `snipeit:ldap-sync` | `User::where('username', ...)` 查找用户用于 LDAP 同步 | 跨公司查找（避免用户名重复时重复创建） |
| [CheckoutLicenseToAllUsers](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/CheckoutLicenseToAllUsers.php) | `snipeit:checkout-license-to-all-users` | 给所有用户分配 license | 跨公司操作 |
| [TestLocationsFMCS](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Console/Commands/TestLocationsFMCS.php) | `snipeit:test-locations-fmcs` | `Location::all()` 遍历所有 location 做 FMCS 一致性检查 | 跨公司，用于迁移前检查 |

### 14.3 Console 命令绕过 FMCS 的原理图示

```
php artisan snipeit:inventory-alerts
  │
  └─ Console Kernel 启动 → 无 HTTP Request → Auth::hasUser() === false
       │
       └─ scopeCompanyables() 判断：
            if (!Auth::hasUser()) return $query;  ← 直接返回，不加 WHERE
       │
       └─ Asset::count() → SELECT COUNT(*) FROM assets  (无 company_id 条件)
       │
       └─ 邮件发送至 settings.alert_email (全局配置的管理员邮箱)
```

**Console 命令都是天然跨公司的**，这在设计上是合理的——它们处理的是系统级的维护任务和告警通知。但需要注意：`alert_email` 是全局设置，收到的告警邮件是全公司汇总的。

---

## 十五、独立按地点进行访问控制？—— 不存在

代码中**没有独立于 FMCS 的按地点（Location.id）进行访问控制的机制**。地点相关的"切片"只有两层：

1. **FMCS 级**：Location 模型的 `company_id` 必须与用户所属公司匹配（通过 `CompanyableScope` + `scope_locations_fmcs` 开关控制列表 API）。
2. **验证级**：Asset 的 `location_id` / `rtd_location_id` 必须属于 Asset 自身的 `company_id`（通过 `fmcs_location` 验证规则）。

不存在"某用户只能看到某个 Location.id 下的资产"这种地点级 RBAC。所有可见性最终都归约到 `company_id` 这一个维度。

另外，[Helper::test_locations_fmcs()](file:///d:/fz/0601-2/solo-dogfeeding/code/10-snipe-it/app/Helpers/Helper.php#L1686) 提供了一个 FMCS 一致性检查工具，用于启用 `scope_locations_fmcs` 前排查：
- 检查已 checkout 的 Asset：其 `location_id` 对应的 Location.company_id 是否与 Asset.company_id 匹配
- 检查 User：其 `location_id` 对应的 Location.company_id 是否与 User 所属公司匹配
- 不匹配的项需要管理员修正后才能安全启用位置范围限定

---

## 十六、最终总结：各层在可见性切片中的职责澄清

| 层/机制 | 是否直接决定可见性？ | 作用对象 | 依据 |
|---|---|---|---|
| **模型 Global Scope（CompanyableScope）** | ✅ **最主要、最隐式** | 所有使用 `CompanyableTrait` / `CompanyableChildTrait` 的模型的**所有查询** | `company_id` 列 + `company_user` 中间表 |
| **用户上下文（Auth user + company_user pivot）** | ✅ **参数来源** | 提供 `scopeCompanyables()` 的输入值（用户所属公司 ID 列表） | `company_user` 中间表 |
| **scope_locations_fmcs 开关** | ⚠️ **仅控制 Location 列表 API** | Locations 列表是否附加公司切片 | 不开启时，所有 Location 对所有已认证用户可见 |
| **fmcs_location 验证规则** | ❌ **只校验赋值一致性** | 创建/更新 Asset 时 location 与 company 是否匹配 | 不做查询切片，只做写入前校验 |
| **canCheckoutTo()** | ❌ **只限制出借方向** | Asset/License/Consumable 等能否借给某目标 | 写入时的跨公司保护 |
| **Policy（SnipePermissionsPolicy + isCurrentUserHasAccess）** | ✅ **单对象第二道防线** | view/update/delete 单对象操作 | `company_id` 精确匹配 |
| **Gate::before（SuperUser）** | ✅ **最高权限放行** | 超级用户所有操作 | 绕过所有切片 |
| **控制器显式 scopeCompanyables()** | ⚠️ **补充性（非必须）** | 下拉列表、Dashboard 用户计数等 | 用于 Global Scope 不生效或需要明确语义的场景 |
| **withoutGlobalScopes 绕过** | ✅ **功能性旁路** | checkout 目标解析、日志展示、导入、报表关联等 | 绕过之后必有显式 FMCS 二次检查 |
| **Console 命令（无 Auth 用户）** | ✅ **天然跨公司** | 所有 Artisan 命令中的查询 | `Auth::hasUser() === false` 导致切片 no-op |
| **独立按 Location.id 切片** | ❌ **不存在** | — | — |
