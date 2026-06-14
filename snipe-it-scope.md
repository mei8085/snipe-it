# Snipe-IT 资产可见性范围过滤机制分析

## 概述

Snipe-IT 使用 **FMCS (Full Multiple Company Support)** 机制实现资产按公司和位置进行可见性隔离。整个系统通过一套由 **Trait → Global Scope → 静态方法** 组成的链式架构，配合 **策略层(Gate Policy)** 和 **验证层(Validation Rules)** 构成多维度的数据隔离防线。

---

## 一、用户公司/位置信息的来源

### 1.1 数据模型定义

用户的公司和位置信息直接存储在 `users` 表中，由 [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/User.php) 模型管理：

```php
// User.php $fillable 中定义的关键字段
'company_id',   // 用户所属公司ID
'location_id',  // 用户所属位置ID
```

关系定义（[User.php#L601-L812](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/User.php#L601-L812)）：

```php
public function company() {
    return $this->belongsTo(Company::class, 'company_id');
}

public function userloc() {  // 历史遗留别名
    return $this->belongsTo(Location::class, 'location_id')->withTrashed();
}

public function location() {
    return $this->belongsTo(Location::class, 'location_id')->withTrashed();
}
```

### 1.2 FMCS 功能开关

FMCS 由 [Setting.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Setting.php) 中的两个全局配置控制：

| 配置字段 | 作用 |
|---|---|
| `full_multiple_companies_support` | 总开关，开启后所有公司级别的可见性过滤才会生效 |
| `scope_locations_fmcs` | 位置范围开关，开启后 Location 模型也会被公司范围约束，且会启用 `fmcs_location` 验证规则 |

判断函数位于 [Company.php#L97-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L97-L107)：

```php
private static function isFullMultipleCompanySupportEnabled() {
    $settings = Setting::getSettings();
    return $settings?->full_multiple_companies_support == 1;
}
```

### 1.3 当前用户公司ID获取

通过 `auth()->user()->company_id` 直接获取当前登录用户的公司ID。这是所有范围过滤的源头。

---

## 二、过滤条件的注入机制

### 2.1 整体架构调用链

```
模型使用 CompanyableTrait
        ↓
bootCompanyableTrait() → addGlobalScope(new CompanyableScope)
        ↓
CompanyableScope::apply() → 调用 Company::scopeCompanyables($builder)
        ↓
检查: FMCS关闭? / 用户是SuperUser? / 未登录?
  ├─ 是 → 直接返回 $query (不过滤)
  └─ 否 → 调用 Company::scopeCompanyablesDirectly($query, ...)
        ↓
从 auth()->user()->company_id 取用户公司ID
        ↓
where(表名.company_id = 用户公司ID)
```

### 2.2 CompanyableTrait — 触发入口

[CompanyableTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/CompanyableTrait.php) 是所有需要被公司范围过滤的模型都必须 `use` 的 Trait：

```php
trait CompanyableTrait {
    public static function bootCompanyableTrait() {
        // Location 模型需要额外检查 scope_locations_fmcs 开关
        if (__CLASS__ != 'App\Models\Location') {
            static::addGlobalScope(new CompanyableScope);
        } else {
            if (Setting::getSettings()?->scope_locations_fmcs == 1) {
                static::addGlobalScope(new CompanyableScope);
            }
        }
    }
}
```

**使用 CompanyableTrait 的核心模型**（通过 grep `use CompanyableTrait` 可得）：
- `User`, `Asset`, `Accessory`, `Component`, `Consumable`, `License`, `LicenseSeat`
- `Location`, `Department`, `Supplier`, `Manufacturer`, `Category`, `AssetModel`
- `Actionlog`, `Company`, `Maintenance`, `PredefinedKit`, `Statuslabel`, `Group` 等

### 2.3 CompanyableScope — Global Scope 层

[CompanyableScope.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/CompanyableScope.php) 是 Laravel 的 Global Scope 实现：

```php
final class CompanyableScope implements Scope {
    public function apply(Builder $builder, Model $model) {
        return Company::scopeCompanyables($builder);
    }
}
```

它只是一个薄包装层，真正的逻辑被委托给 `Company` 模型的静态方法。这样做的好处是所有模型的范围逻辑集中在一处维护。

### 2.4 Company::scopeCompanyables() — 核心调度器

[Company.php#L289-L297](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L289-L297)：

```php
public static function scopeCompanyables($query, $column = 'company_id', $table_name = null) {
    // 三种情况跳过过滤:
    // 1. FMCS 未开启
    // 2. 已登录用户是 SuperUser (超级用户)
    // 3. 没有登录用户（CLI/命令行场景）
    if (!self::isFullMultipleCompanySupportEnabled() 
        || (Auth::hasUser() && auth()->user()->isSuperUser()) 
        || (!Auth::hasUser())) {
        return $query;
    }
    return self::scopeCompanyablesDirectly($query, $column, $table_name);
}
```

### 2.5 Company::scopeCompanyablesDirectly() — 实际过滤执行

[Company.php#L305-L328](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L305-L328)：

```php
private static function scopeCompanyablesDirectly($query, $column = 'company_id', $table_name = null) {
    $company_id = null;
    if (Auth::hasUser()) {
        $company_id = auth()->user()->company_id;
    }

    // 特殊处理: 对 companies 表自身用 id 字段过滤
    if ($query->getModel()->getTable() == 'companies') {
        return $query->where('companies.id', '=', $company_id);
    }

    // 通用处理: 检查模型表是否存在 company_id 列
    if ((($query) && ($query->getModel()) 
         && (Schema::hasColumn($query->getModel()->getTable(), $column)))) {
        $table = ($table_name) ? $table_name.'.' : $query->getModel()->getTable().'.';
        return $query->where($table.$column, '=', $company_id);
    }
}
```

**关键点**：
- 使用 `Schema::hasColumn()` 动态检查表是否有 `company_id` 列，避免 SQL 错误
- 支持通过 `$table_name` 参数指定别名表名（用于复杂 join 查询）

### 2.6 CompanyableChildScope — 子关联范围

[CompanyableChildScope.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/CompanyableChildScope.php) 用于模型自身**没有** `company_id` 列，但可以通过父级关联（如 `asset`、`license` 等）来推断公司范围的场景。

调用链：`CompanyableChildScope::apply()` → `Company::scopeCompanyableChildren()` ([Company.php#L339-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L339-L363))

```php
public static function scopeCompanyableChildren(array $companyable_names, $query) {
    // 对每个父级关联名使用 whereHas / orWhereHas 过滤
    // 只要任意一个父级关联属于用户公司, 该记录就可见
    $f = fn($q) => static::scopeCompanyablesDirectly($q);
    $q = $query->where(function ($q) use ($companyable_names, $f) {
        $q2 = $q->whereHas($companyable_names[0], $f);
        for ($i = 1; $i < count($companyable_names); $i++) {
            $q2 = $q2->orWhereHas($companyable_names[$i], $f);
        }
    });
    return $q;
}
```

### 2.7 Searchable Trait — 高级搜索中的位置/公司过滤

[Searchable.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/Searchable.php) 中的结构化过滤器（JSON payload）支持对位置和公司进行高级搜索：

- **属性层过滤**：如果过滤键是模型的直接属性（如 `company_id`），直接对表列使用 LIKE / = 查询
- **关系层过滤**：对配置在 `$searchableRelations` 中的关联（如 `'company' => ['name']`、`'location' => ['name']`），使用 `whereHas()` 子查询
- **否定关系过滤的特殊处理**：使用 `doesntHave() + orWhereHas(NOT LIKE)` 的复合查询，确保"没有关联记录 + 关联记录不匹配"都能被包含

例如 User 模型中：
```php
protected $searchableRelations = [
    'userloc' => ['name', 'address', ...],  // 位置
    'company' => ['name'],                   // 公司
];
protected $searchableRelationAliases = [
    'location' => 'userloc',  // UI/API 用 "location" 映射到实际关联名 "userloc"
];
protected $searchableRelationFilterColumns = [
    'location' => ['name'],   // 位置高级搜索只匹配 name 列
];
```

---

## 三、跨范围引用的处理

跨范围引用指的是：当 FMCS 开启时，用户 A（公司 1）需要访问/引用用户 B 或资产 B（公司 2）的场景。系统采用 **先解除范围获取对象 → 再显式校验权限** 的两段式策略。

### 3.1 withoutGlobalScopes() — 解除范围约束的标准方法

当需要跨范围查询时（如 checkout 时查找可能属于其他公司的目标对象），必须使用 `withoutGlobalScopes()` 绕过 CompanyableScope。

典型应用位于 [Api/AssetsController.php#L878-L893](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L878-L893)：

```php
private function resolveCheckoutTarget($assetId) {
    if ($request->filled('assigned_user')) {
        return User::withoutGlobalScopes()->find($request->input('assigned_user'));
    }
    if ($request->filled('assigned_asset')) {
        return Asset::withoutGlobalScopes()->where('id', '!=', $assetId)
            ->find($request->input('assigned_asset'));
    }
    if ($request->filled('assigned_location')) {
        return Location::withoutGlobalScopes()->find($request->input('assigned_location'));
    }
}
```

### 3.2 Checkout 时的显式公司匹配校验

解除范围获取目标对象后，**必须**立即进行公司一致性校验。代码位于 [Api/AssetsController.php#L895-L906](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L895-L906) 和 [L1054-L1062](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L1054-L1062)：

```php
private function checkoutCompanyMismatchResponse(Asset $asset, $target): ?JsonResponse {
    if ((Setting::getSettings()->full_multiple_companies_support == '1')
        && (!is_null($asset->company_id))
        && (!is_null($target->company_id))
        && ((int)$asset->company_id !== (int)$target->company_id)
    ) {
        return response()->json(Helper::formatStandardApiResponse(
            'error', null, trans('general.error_user_company')
        ));
    }
    return null;
}
```

**校验逻辑**：当 FMCS 开启且资产和目标都有明确的 company_id 时，两者必须完全相同，否则拒绝 checkout。注意：任一方 company_id 为 NULL（表示未分配公司）时视为允许跨范围。

### 3.3 Actionlog 公司ID回填的跨范围查询

Actionlog 作为审计日志，其 `company_id` 必须忠实地记录被操作对象所属的公司——即使用户本身看不到该对象（不是自己公司的）。因此在 [Actionlog.php#L160-L201](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Actionlog.php#L160-L201) 中必须使用 `withoutGlobalScopes()`：

```php
protected static function resolveCompanyIdFromModelClass(?string $modelClass, ?int $id): ?int {
    // ... 前置检查 ...
    return $modelClass::withoutGlobalScopes()  // 关键: 解除范围
        ->whereKey($id)
        ->value('company_id');
}
```

**优先级规则**：优先取 `item`（被操作的资产/许可证等）的 company_id，取不到时回退到 `target`（用户/位置）的 company_id。这确保了 FMCS 归属以被操作资产的所有权为准。

### 3.4 SnipePermissionsPolicy — 策略层的范围守卫

[SnipePermissionsPolicy.php#L38-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Policies/SnipePermissionsPolicy.php#L38-L73) 的 `before()` 方法在所有具体权限方法之前执行：

```php
public function before(User $user, $ability, $item) {
    // Admin 权限仍需受 FMCS 约束 (SuperUser 在 AuthServiceProvider 另行处理)
    if ($user->hasAccess('admin')) {
        return true;
    }
    // $item 是类名字符串不是实例时, 交给后续方法处理
    if (!$item instanceof Model) {
        return;
    }
    // 核心: 检查当前用户是否有权访问该模型实例
    if (!Company::isCurrentUserHasAccess($item)) {
        return false;
    }
}
```

而 `Company::isCurrentUserHasAccess()` ([Company.php#L153-L196](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L153-L196)) 实现了如下访问矩阵：

| 条件 | 结果 |
|---|---|
| FMCS 未开启 | ✅ 允许 |
| 模型表没有 company_id 列 | ✅ 允许 |
| 用户 company_id 为 NULL | ✅ 允许（未分配公司的用户可以看到所有） |
| 用户 company_id == 资产 company_id | ✅ 允许 |
| 用户是 SuperUser | ✅ 允许 |
| 其他情况 | ❌ 拒绝 |

### 3.5 fmcs_location 验证规则 — 位置与公司的绑定约束

当同时开启 FMCS 和位置范围时，[ValidationServiceProvider.php#L352-L365](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Providers/ValidationServiceProvider.php#L352-L365) 中的自定义验证规则 `fmcs_location` 会生效：

```php
Validator::extend('fmcs_location', function ($attribute, $value, $parameters, $validator) {
    $settings = Setting::getSettings();
    if ($settings->full_multiple_companies_support == '1' 
        && $settings->scope_locations_fmcs == '1') {
        $company_id = array_get($validator->getData(), 'company_id');
        $location = Location::find($value);
        if (($location) && ($company_id != $location->company_id)) {
            return false;  // 位置所属公司与模型所属公司不一致
        }
    }
    return true;
});
```

该规则在 Asset、User、Accessory、Component、Consumable 等模型的 `$rules` 中被用于 `location_id` 和 `rtd_location_id` 字段。

---

## 四、报表导出时的同步约束

### 4.1 隐式约束 — 通过 CompanyableTrait 自动生效

大部分报表导出查询直接使用 `Model::...->get()` 或 `->chunk()`，由于模型已经 use 了 CompanyableTrait，Global Scope 会自动注入范围条件，无需额外代码。

以下是 [ReportsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php) 中各报表的约束方式：

| 报表 | 查询代码 | 约束来源 |
|---|---|---|
| 折旧报表 | `Asset::with(...)->orderBy()->get()` ([L170](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L170)) | Asset 的 CompanyableTrait |
| 活动日志 | `Actionlog::with(...)->chunk(500, ...)` ([L309-L311](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L309-L311)) | Actionlog 的 CompanyableTrait |
| 许可证报表 | `License::with(...)->get()` ([L382-L384](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L382-L384)) | License 的 CompanyableTrait |
| 维护报表 | `Maintenance::with(...)->get()` ([L1179-L1181](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L1179-L1181)) | Maintenance 的 CompanyableTrait |
| 自定义报表 | `Asset::select(...)->with(...)->chunk(500, ...)` ([L714-L716](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L714-L716)) | Asset 的 CompanyableTrait |

### 4.2 显式约束 — CheckoutAcceptance 的特殊处理

**关键问题**：`CheckoutAcceptance` 模型（待验收记录表）**没有** `company_id` 列，也**不使用** CompanyableTrait。如果直接查询会导致跨公司数据泄露。

解决方案：使用 `whereHasMorph()` 通过多态关联 `checkoutable` 间接过滤，让关联的具体模型（Asset、LicenseSeat 等）的 CompanyableTrait 自动生效。

#### 报表首页计数 — ReportsController::index()
[L66-L70](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L66-L70)（代码中带注释说明了设计意图）：

```php
// CheckoutAcceptance has no company_id column; scope through the checkoutable
// relationship so each type's CompanyableTrait global scope is applied.
$pending_acceptance_count = CheckoutAcceptance::pending()
    ->whereHasMorph('checkoutable', [
        Asset::class, LicenseSeat::class, Accessory::class, 
        Component::class, Consumable::class
    ])
    ->count();
```

**原理**：`whereHasMorph()` 的子查询会走各具体模型的 Query Builder，此时 CompanyableScope 会被自动应用，从而只统计当前用户可见公司内的待验收记录。

#### 待验收报表视图 — ReportsController::getAssetAcceptanceReport()
[L1244-L1258](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L1244-L1258) 使用 `with([...])` 预加载关联，虽然使用了 `withTrashed()` 和 `morphWith()`，但底层的 Global Scope 仍然生效，保证公司范围约束不被破坏。

### 4.3 自定义报表中的显式过滤条件

自定义报表（`postCustom` 方法）允许用户通过表单提交额外的过滤条件，这些条件**叠加**在 Global Scope 之上（两者是 AND 关系，不会冲突）：

```php
$assets = Asset::select('assets.*')->with(...);  // 先自动应用公司范围

// 然后是用户选择的条件 (都用 where/whereIn, 与公司范围 AND 合并)
if ($request->filled('by_company_id')) {
    $assets->whereIn('assets.company_id', $request->input('by_company_id'));
}
if ($request->filled('by_location_id')) {
    $assets->whereIn('assets.location_id', $request->input('by_location_id'));
}
if ($request->filled('by_rtd_location_id')) {
    $assets->whereIn('assets.rtd_location_id', $request->input('by_rtd_location_id'));
}
// ... 还有 category/manufacturer/department/status/日期范围等
```

### 4.4 导出数据的权限检查

所有报表方法的第一行都是：
```php
$this->authorize('reports.view');
```
该权限在 Policy 层与 `before()` 中的 FMCS 检查串联（参见 3.4 节），确保即使有用户直接访问导出 URL，也会先被权限系统拦截。

---

## 附录：关键文件索引

| 功能 | 文件位置 |
|---|---|
| FMCS 开关判断 | [Company.php#L97-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L97-L107) |
| 公司范围入口 Trait | [CompanyableTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/CompanyableTrait.php) |
| Global Scope 层 | [CompanyableScope.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/CompanyableScope.php) |
| 核心范围调度 | [Company.php#L289-L328](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L289-L328) |
| 子关联范围 | [Company.php#L339-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L339-L363) |
| 访问权限判断 | [Company.php#L153-L196](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L153-L196) |
| 创建时公司ID注入 | [Company.php#L127-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L127-L145) |
| Gate Policy 守卫 | [SnipePermissionsPolicy.php#L38-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Policies/SnipePermissionsPolicy.php#L38-L73) |
| fmcs_location 验证 | [ValidationServiceProvider.php#L352-L365](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Providers/ValidationServiceProvider.php#L352-L365) |
| Checkout 跨范围解析 | [Api/AssetsController.php#L878-L906](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L878-L906) |
| Checkout 公司不匹配校验 | [Api/AssetsController.php#L1054-L1062](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L1054-L1062) |
| Actionlog 公司ID回填 | [Actionlog.php#L125-L201](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Actionlog.php#L125-L201) |
| CheckoutAcceptance 范围约束 | [ReportsController.php#L66-L70](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L66-L70) |
| 自定义报表导出查询 | [ReportsController.php#L714-L845](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L714-L845) |
| 搜索 Trait 过滤逻辑 | [Searchable.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/Searchable.php) |
| 用户模型(公司/位置字段) | [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/User.php) |
| 资产模型(CompanyableTrait) | [Asset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Asset.php) |
| 位置模型(条件性 Scope) | [Location.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Location.php) |
