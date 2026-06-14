# Snipe-IT 资产可见性范围过滤机制分析

## 概述

Snipe-IT 使用 **FMCS (Full Multiple Company Support)** 机制实现资产按公司和位置进行可见性隔离。整个系统通过一套由 **Trait → Global Scope → 静态方法** 组成的链式架构，配合 **策略层(Gate Policy)** 和 **验证层(Validation Rules)** 构成多维度的数据隔离防线。

范围隔离的完整生命周期包含三个阶段：
1. **写入时**：创建/更新对象时，通过 `Company::getIdForCurrentUser()` 将公司归属覆盖到对象上
2. **读取时**：查询列表时，通过 Global Scope 自动注入 `WHERE company_id = ?` 过滤
3. **访问时**：对单个对象操作时，通过 Policy 的 `before()` 钩子进行权限校验

其中读取层存在两条并行的链路：**直接范围**（模型自身有 company_id 列）和 **子关联范围**（模型无 company_id 列，通过父级关联推断）。

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

## 二、写入时的公司归属覆盖

### 2.1 Company::getIdForCurrentUser() — 创建时公司ID注入

[Company.php#L127-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L127-L145)：

```php
public static function getIdForCurrentUser($unescaped_input) {
    if (! self::isFullMultipleCompanySupportEnabled()) {
        return self::getIdFromInput($unescaped_input);  // FMCS关闭: 使用前端提交值
    } else {
        $current_user = auth()->user();
        if ($current_user->isSuperUser()) {
            return self::getIdFromInput($unescaped_input);  // SuperUser: 允许自由指定
        } else {
            if ($current_user->company_id != null) {
                return $current_user->company_id;  // 普通用户: 强制覆盖为用户自身公司
            } else {
                return null;  // 无公司用户: 允许为null
            }
        }
    }
}
```

**关键行为**：当 FMCS 开启时，普通用户（非 SuperUser）创建或更新对象的 `company_id` 字段会被**强制覆盖**为用户自身的 `company_id`，无论前端提交了什么值。这是写入层的核心保护机制。

### 2.2 Company::getIdForUser() — 用户模型的特殊方法

[Company.php#L234-L241](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L234-L241)：

```php
public static function getIdForUser($unescaped_input) {
    if (! self::isFullMultipleCompanySupportEnabled() || auth()->user()->isSuperUser()) {
        return self::getIdFromInput($unescaped_input);  // FMCS关闭 或 SuperUser: 使用提交值
    } else {
        return self::getIdForCurrentUser($unescaped_input);  // 否则: 走强制覆盖逻辑
    }
}
```

`getIdForUser()` 与 `getIdForCurrentUser()` 的区别在于：`getIdForUser()` 专门用于 **User 模型**的创建/更新场景（[UsersController.php#L126](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Users/UsersController.php#L126)、[L278](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Users/UsersController.php#L278)）。它对 SuperUser 的处理更宽松——SuperUser 管理用户时可以自由指定公司归属，而 `getIdForCurrentUser()` 对 SuperUser 也允许自由指定但用于非用户对象。

### 2.3 各控制器中的写入覆盖调用

以下是各控制器在 create/update 时调用 `Company::getIdForCurrentUser()` 的位置：

| 控制器 | 操作 | 代码位置 |
|---|---|---|
| AssetsController (API) | create | [L767](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L767) |
| LicensesController | create | [L92](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Licenses/LicensesController.php#L92) |
| LicensesController | update | [L178](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Licenses/LicensesController.php#L178) |
| LocationsController | create | [L92-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/LocationsController.php#L92-L96) |
| LocationsController | update | [L175-L176](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/LocationsController.php#L175-L176) |
| ComponentsController | create | [L87](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Components/ComponentsController.php#L87) |
| ComponentsController | update | [L162](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Components/ComponentsController.php#L162) |
| ConsumablesController | create | [L85](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Consumables/ConsumablesController.php#L85) |
| ConsumablesController | update | [L179](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Consumables/ConsumablesController.php#L179) |
| AccessoriesController | create | [L70](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Accessories/AccessoriesController.php#L70) |
| AccessoriesController | update | [L177](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Accessories/AccessoriesController.php#L177) |
| DepartmentsController | create | [L57](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/DepartmentsController.php#L57) |
| DepartmentsController | update | [L167](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/DepartmentsController.php#L167) |
| UsersController (Web) | create | [L126](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Users/UsersController.php#L126) (使用 `getIdForUser`) |
| UsersController (Web) | update | [L278](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Users/UsersController.php#L278) (使用 `getIdForUser`) |
| UsersController (API) | create | [L444](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/UsersController.php#L444) |
| UsersController (API) | update | [L579](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/UsersController.php#L579) |
| ComponentsController (API) | create | [L170](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/ComponentsController.php#L170) |
| ComponentsController (API) | update | [L211](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/ComponentsController.php#L211) |
| ConsumablesController (API) | create | [L160](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/ConsumablesController.php#L160) |
| LocationsController (API) | create | [L204](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/LocationsController.php#L204) |
| LocationsController (API) | update | [L284](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/LocationsController.php#L284) |

### 2.4 Location 模型的写入时特殊逻辑

Location 模型在写入时有双重逻辑（[LocationsController.php#L92-L103](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/LocationsController.php#L92-L103)）：

```php
$location->company_id = Company::getIdForCurrentUser($request->input('company_id'));  // 先设默认值

if (Setting::getSettings()->scope_locations_fmcs) {
    $location->company_id = Company::getIdForCurrentUser($request->input('company_id'));  // 再次覆盖（与上相同）
    if ($location->parent_id && Location::find($location->parent_id)->company_id != $location->company_id) {
        return redirect()->back()->withInput()->with('error', 'different company than parent');
    }
} else {
    $location->company_id = $request->input('company_id');  // 位置范围关闭时: 使用前端提交值
}
```

**行为分析**：
- 当 `scope_locations_fmcs` **开启**时：公司ID被 `getIdForCurrentUser()` 强制覆盖，并额外检查父位置的公司归属是否一致
- 当 `scope_locations_fmcs` **关闭**时：第92行的赋值被第102行**回退**为前端提交值，即位置的 company_id 完全由操作者指定

### 2.5 Actionlog 写入时的公司ID自动回填

Actionlog 的 `company_id` 不由控制器显式赋值，而是在模型的 `boot()` 钩子中自动解析（[Actionlog.php#L125-L139](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Actionlog.php#L125-L139)）：

```php
static::creating(function (self $actionlog): void {
    if (! array_key_exists('company_id', $actionlog->getAttributes())) {
        $actionlog->company_id = static::resolveCompanyIdFromAttributes(
            $actionlog->target_type, $actionlog->target_id,
            $actionlog->item_type,   $actionlog->item_id,
        );
    }
});
```

`resolveCompanyIdFromAttributes()` 内部调用 `resolveCompanyIdFromModelClass()`，使用 `withoutGlobalScopes()` 获取被操作对象的真实 company_id，优先级为 item > target。

此外，Loggable Trait 中的 `logable()` 和 `logCheckout()` 方法也通过 `resolveLoggableCompanyId()` 在写入时设置 company_id（[Loggable.php#L233-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/Loggable.php#L233-L255)）：

```php
private function resolveLoggableCompanyId(): ?int {
    if (static::class === LicenseSeat::class) {
        return $this->license?->company_id;  // LicenseSeat: 从父级 License 取
    }
    if (isset($this->company_id)) {
        return $this->company_id;  // 普通模型: 直接取自身
    }
    if ($this instanceof ICompanyableChild) {
        foreach ((array) $this->getCompanyableParents() as $parentRelation) {
            $parent = $this->{$parentRelation} ?? null;
            if (isset($parent?->company_id)) {
                return $parent->company_id;  // 子关联模型: 从父级取
            }
        }
    }
    return null;
}
```

---

## 三、读取时的过滤条件注入机制

### 3.1 整体架构调用链

```
模型使用 CompanyableTrait 或 CompanyableChildTrait
        ↓
    ┌───────────────────────────────┬──────────────────────────────────────┐
    │  直接范围 (有 company_id 列)    │  子关联范围 (无 company_id 列)          │
    │                               │                                      │
    │  CompanyableTrait             │  CompanyableChildTrait               │
    │       ↓                       │       ↓                              │
    │  addGlobalScope(              │  addGlobalScope(                     │
    │    CompanyableScope)          │    CompanyableChildScope)            │
    │       ↓                       │       ↓                              │
    │  CompanyableScope::apply()    │  CompanyableChildScope::apply()      │
    │       ↓                       │       ↓                              │
    │  Company::scopeCompanyables() │  model->getCompanyableParents()      │
    │       ↓                       │       ↓                              │
    │  scopeCompanyablesDirectly()  │  Company::scopeCompanyableChildren() │
    │       ↓                       │       ↓                              │
    │  where(表.company_id = ?)     │  whereHas(父关联, scopeCompanyables)  │
    └───────────────────────────────┴──────────────────────────────────────┘
```

### 3.2 CompanyableTrait — 直接范围的触发入口

[CompanyableTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/CompanyableTrait.php) 是所有自身拥有 `company_id` 列、需要被公司范围直接过滤的模型都必须 `use` 的 Trait：

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

### 3.3 CompanyableScope — Global Scope 层

[CompanyableScope.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/CompanyableScope.php) 是 Laravel 的 Global Scope 实现：

```php
final class CompanyableScope implements Scope {
    public function apply(Builder $builder, Model $model) {
        return Company::scopeCompanyables($builder);
    }
}
```

它只是一个薄包装层，真正的逻辑被委托给 `Company` 模型的静态方法。这样做的好处是所有模型的范围逻辑集中在一处维护。

### 3.4 Company::scopeCompanyables() — 核心调度器

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

### 3.5 Company::scopeCompanyablesDirectly() — 实际过滤执行

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

### 3.6 CompanyableChildTrait + CompanyableChildScope — 子关联范围链路

这条链路与直接范围链路平行，用于模型自身**没有** `company_id` 列，但可以通过父级关联来推断公司范围的场景。

#### ICompanyableChild 接口

[ICompanyableChild.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/ICompanyableChild.php) 定义了子关联模型的契约：

```php
interface ICompanyableChild {
    public function getCompanyableParents();
}
```

`getCompanyableParents()` 返回一个关联名数组，声明该模型通过哪些父级关联来确定公司归属。

#### CompanyableChildTrait — 触发入口

[CompanyableChildTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/CompanyableChildTrait.php)：

```php
trait CompanyableChildTrait {
    public static function bootCompanyableChildTrait() {
        static::addGlobalScope(new CompanyableChildScope);
    }
}
```

与 `CompanyableTrait` 不同，这里**无条件**注册 Global Scope，没有 FMCS 开关判断——因为子关联范围的跳过逻辑在 `scopeCompanyableChildren()` 内部处理。

#### CompanyableChildScope — Global Scope 层

[CompanyableChildScope.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/CompanyableChildScope.php)：

```php
final class CompanyableChildScope implements Scope {
    public function apply(Builder $builder, Model $model) {
        $model = $builder->getModel();
        return Company::scopeCompanyableChildren($model->getCompanyableParents(), $builder);
    }
}
```

#### Company::scopeCompanyableChildren() — 子关联过滤执行

[Company.php#L339-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L339-L363)：

```php
public static function scopeCompanyableChildren(array $companyable_names, $query) {
    if (count($companyable_names) == 0) {
        throw new Exception('No Companyable Children to scope');
    } elseif (! self::isFullMultipleCompanySupportEnabled()
               || (Auth::hasUser() && auth()->user()->isSuperUser())) {
        return $query;  // FMCS关闭 或 SuperUser: 跳过
    } else {
        $f = function ($q) {
            static::scopeCompanyablesDirectly($q);  // 复用直接范围的 WHERE 逻辑
        };
        $q = $query->where(function ($q) use ($companyable_names, $f) {
            $q2 = $q->whereHas($companyable_names[0], $f);
            for ($i = 1; $i < count($companyable_names); $i++) {
                $q2 = $q2->orWhereHas($companyable_names[$i], $f);
            }
        });
        return $q;
    }
}
```

**核心逻辑**：对每个父级关联名使用 `whereHas` / `orWhereHas`，只要**任意一个**父级关联属于用户公司，该子记录就可见。内部复用了 `scopeCompanyablesDirectly()` 来执行父表上的 `WHERE company_id = ?` 条件。

#### 实际使用 ICompanyableChild 的模型

| 模型 | getCompanyableParents() 返回 | 含义 |
|---|---|---|
| [LicenseSeat](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/LicenseSeat.php#L67-L70) | `['asset', 'license']` | 许可证席位的可见性由其关联的资产或许可证决定 |
| [Maintenance](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Maintenance.php#L110-L113) | `['asset']` | 维护记录的可见性由其关联的资产决定 |

LicenseSeat 返回两个父级（asset 和 license），使用 OR 语义：只要资产或许可证任一属于当前用户的公司，该席位就可见。这处理了"席位已分配给其他公司的资产，但许可证本身属于当前公司"的场景。

### 3.7 Searchable Trait — 高级搜索中的位置/公司过滤

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

## 四、公司归属为空时的边界场景

### 4.1 列表过滤中的 NULL 行为

在 `scopeCompanyablesDirectly()` 中（[Company.php#L305-L328](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L305-L328)），过滤条件为：

```php
return $query->where($table.$column, '=', $company_id);
```

当用户的 `company_id` 为 NULL 时，生成的 SQL 是 `WHERE company_id = NULL`。在 SQL 标准中，`= NULL` 永远返回 false（必须用 `IS NULL`），因此：

| 用户 company_id | 对象 company_id | 列表过滤结果 |
|---|---|---|
| NULL | NULL | ❌ 不可见（`NULL = NULL` → false） |
| NULL | 1 | ❌ 不可见（`NULL = 1` → false） |
| 1 | NULL | ❌ 不可见（`1 = NULL` → false） |
| 1 | 1 | ✅ 可见 |

**结论**：FMCS 开启时，`company_id` 为 NULL 的用户在列表中**看不到任何记录**（包括未分配公司的对象）；`company_id` 为 NULL 的对象也**不会出现在任何用户的列表中**。

### 4.2 权限校验中的 NULL 行为

在 `isCurrentUserHasAccess()` 中（[Company.php#L153-L196](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L153-L196)），核心判断为：

```php
return ($current_user_company_id == null)
    || ($current_user_company_id == $companyable_company_id)
    || auth()->user()->isSuperUser();
```

这是 PHP 的 `==` 比较，不是 SQL：

| 用户 company_id | 对象 company_id | 权限校验结果 | 原因 |
|---|---|---|---|
| NULL | NULL | ✅ 允许 | 第一个条件 `$current_user_company_id == null` 为 true |
| NULL | 1 | ✅ 允许 | 同上：用户无公司则允许访问所有 |
| 1 | NULL | ❌ 拒绝 | 用户有公司，对象无公司，两者不相等 |
| 1 | 1 | ✅ 允许 | 直接匹配 |

### 4.3 边界矛盾：列表可见 ≠ 权限允许

由于列表过滤（SQL `= NULL`）和权限校验（PHP `== null`）对 NULL 的处理逻辑不同，存在以下矛盾场景：

**场景 A：无公司用户访问有公司对象**
- 列表过滤：❌ 不可见（`NULL = 1` → false，列表为空）
- 权限校验：✅ 允许（`$current_user_company_id == null` 为 true）
- **实际效果**：无公司用户在列表中看不到有公司的对象，但如果通过 URL 直接访问某个有公司对象的详情页，Policy 的 `before()` 会放行。不过由于列表无法发现该对象，实际上这种情况极少触发。

**场景 B：有公司用户访问无公司对象**
- 列表过滤：❌ 不可见（`1 = NULL` → false）
- 权限校验：❌ 拒绝（用户有公司但对象没有，不匹配）
- **实际效果**：双阻，无公司对象对有公司用户完全不可达。

**场景 C：无公司用户访问无公司对象**
- 列表过滤：❌ 不可见（`NULL = NULL` → false）
- 权限校验：✅ 允许
- **实际效果**：理论上允许但列表不可见，形成"幽灵对象"——只能通过 API 或直接输入 ID 才能访问。

### 4.4 Checkout 时的 NULL 豁免

在 checkout 公司匹配校验中（[AssetsController.php#L895-L906](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L895-L906)）：

```php
if ((!is_null($asset->company_id))
    && (!is_null($target->company_id))
    && ((int)$asset->company_id !== (int)$target->company_id))
```

当资产或目标的 `company_id` 为 NULL 时，整个校验被跳过（视为允许）。这是有意设计：**未分配公司的对象可以 checkout 给任何公司的目标**，反之亦然。这与列表过滤的严格行为形成了反差——列表中 NULL 对象不可见，但一旦获得引用则允许操作。

### 4.5 `canManageUsersCompanies()` — 无公司用户的特权

[Company.php#L203-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L203-L207)：

```php
public static function canManageUsersCompanies() {
    return ! self::isFullMultipleCompanySupportEnabled()
        || auth()->user()->isSuperUser()
        || auth()->user()->company_id == null;
}
```

无公司用户在 FMCS 开启时拥有与 SuperUser 等同的用户管理特权——可以创建/编辑任何公司的用户。这与他们在列表中看不到任何对象的限制形成了又一个矛盾。

---

## 五、跨范围引用的处理

跨范围引用指的是：当 FMCS 开启时，用户 A（公司 1）需要访问/引用用户 B 或资产 B（公司 2）的场景。系统采用 **先解除范围获取对象 → 再显式校验权限** 的两段式策略。

### 5.1 withoutGlobalScopes() — 解除范围约束的标准方法

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

### 5.2 Checkout 时的显式公司匹配校验

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

### 5.3 Actionlog 公司ID回填的跨范围查询

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

### 5.4 SnipePermissionsPolicy — 策略层的范围守卫

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

### 5.5 fmcs_location 验证规则 — 位置与公司的绑定约束

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

## 六、报表导出时的同步约束

### 6.1 隐式约束 — 通过 CompanyableTrait/CompanyableChildTrait 自动生效

大部分报表导出查询直接使用 `Model::...->get()` 或 `->chunk()`，由于模型已使用范围 Trait，Global Scope 会自动注入范围条件，无需额外代码。

以下是 [ReportsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php) 中各报表的约束方式：

| 报表 | 查询代码 | 约束来源 |
|---|---|---|
| 折旧报表 | `Asset::with(...)->orderBy()->get()` ([L170](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L170)) | Asset 的 CompanyableTrait |
| 活动日志 | `Actionlog::with(...)->chunk(500, ...)` ([L309-L311](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L309-L311)) | Actionlog 的 CompanyableTrait |
| 许可证报表 | `License::with(...)->get()` ([L382-L384](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L382-L384)) | License 的 CompanyableTrait |
| 维护报表 | `Maintenance::with(...)->get()` ([L1179-L1181](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L1179-L1181)) | Maintenance 的 CompanyableChildTrait（通过 asset 父关联推断） |
| 自定义报表 | `Asset::select(...)->with(...)->chunk(500, ...)` ([L714-L716](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L714-L716)) | Asset 的 CompanyableTrait |

### 6.2 显式约束 — CheckoutAcceptance 的特殊处理

**关键问题**：`CheckoutAcceptance` 模型（待验收记录表）**没有** `company_id` 列，也**不使用** CompanyableTrait 或 CompanyableChildTrait。如果直接查询会导致跨公司数据泄露。

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

### 6.3 自定义报表中的显式过滤条件

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

### 6.4 导出数据的权限检查

所有报表方法的第一行都是：
```php
$this->authorize('reports.view');
```
该权限在 Policy 层与 `before()` 中的 FMCS 检查串联（参见 5.4 节），确保即使有用户直接访问导出 URL，也会先被权限系统拦截。

---

## 七、范围隔离的模型清单（勘误）

### 7.1 使用 CompanyableTrait 的模型（直接范围）

以下 11 个模型通过 `use CompanyableTrait` 注册 `CompanyableScope`，在读取时自动注入 `WHERE company_id = ?`：

| 模型 | 数据表 | 有 company_id 列 | 备注 |
|---|---|---|---|
| [Asset](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Asset.php#L39) | assets | ✅ | 核心业务对象 |
| [Accessory](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Accessory.php#L27) | accessories | ✅ | |
| [Component](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Component.php#L31) | components | ✅ | |
| [Consumable](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Consumable.php#L27) | consumables | ✅ | |
| [ConsumableAssignment](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/ConsumableAssignment.php#L11) | consumables_users | ⚠️ 可能没有 | 使用 CompanyableTrait 但表可能无 company_id 列，Schema::hasColumn 检查会跳过 |
| [License](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/License.php#L30) | licenses | ✅ | |
| [Location](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Location.php#L22) | locations | ✅ | **条件性 Scope**：仅在 scope_locations_fmcs=1 时注册 |
| [Department](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Department.php#L21) | departments | ✅ | |
| [Company](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L26) | companies | ✅ (即 id 列) | scopeCompanyablesDirectly 对 companies 表特殊处理，用 id 代替 company_id |
| [Actionlog](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Actionlog.php#L27) | action_logs | ✅ | company_id 在 creating 时自动回填 |
| [User](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/User.php#L35) | users | ✅ | ⚠️ CompanyableTrait 注释中说明：不能在 User 上使用 CompanyableScope，会导致认证时的无限循环（参见 Company.php L280-L283 注释） |

### 7.2 使用 CompanyableChildTrait 的模型（子关联范围）

以下 2 个模型通过 `implements ICompanyableChild` + `use CompanyableChildTrait` 注册 `CompanyableChildScope`：

| 模型 | 数据表 | getCompanyableParents() | 含义 |
|---|---|---|---|
| [LicenseSeat](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/LicenseSeat.php#L17) | license_seats | `['asset', 'license']` | OR 语义：资产或许可证任一属于当前公司即可见 |
| [Maintenance](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Maintenance.php#L24) | maintenances | `['asset']` | 单一父关联：仅通过资产推断 |

### 7.3 不使用范围隔离的模型

以下模型虽然有 `company_id` 列或与公司有关联，但**不使用**任何范围 Trait，**不受 FMCS 列表过滤约束**：

| 模型 | 数据表 | 说明 |
|---|---|---|
| Supplier | suppliers | 无 CompanyableTrait，无 company_id 列，跨公司共享 |
| Manufacturer | manufacturers | 无 CompanyableTrait，无 company_id 列，跨公司共享 |
| Category | categories | 无 CompanyableTrait，无 company_id 列，跨公司共享 |
| AssetModel | models | 无 CompanyableTrait，无 company_id 列，跨公司共享 |
| Statuslabel | statuslabels | 无 CompanyableTrait，无 company_id 列，跨公司共享 |
| Group | groups | 无 CompanyableTrait，跨公司共享 |
| PredefinedKit | kits | 无 CompanyableTrait，跨公司共享 |
| AccessoryCheckout | accessories_checkout | 无 CompanyableTrait，无 company_id 列 |
| CheckoutAcceptance | checkout_acceptances | 无 CompanyableTrait，无 company_id 列，需要 whereHasMorph 间接过滤 |

**之前文档中的错误**：原 2.2 节将 `Supplier`、`Manufacturer`、`Category`、`AssetModel`、`Statuslabel`、`Group`、`PredefinedKit`、`LicenseSeat`、`Maintenance` 列为使用 CompanyableTrait 的模型。其中前7个根本不使用范围隔离；`LicenseSeat` 和 `Maintenance` 使用的是 CompanyableChildTrait（子关联范围），不是 CompanyableTrait（直接范围）。

---

## 八、按位置过滤的实际含义

### 8.1 位置不是独立的过滤维度

在 Snipe-IT 的 FMCS 机制中，**位置（Location）不是与公司平行的独立隔离维度**。所谓的"按位置过滤"有两种完全不同的含义：

### 8.2 含义一：位置被公司范围间接约束（scope_locations_fmcs）

当 `scope_locations_fmcs` 开关启用时，Location 模型本身被纳入公司范围隔离：

- **读取时**：Location 模型的 CompanyableTrait 注册 CompanyableScope（仅当 `scope_locations_fmcs=1`），列表查询自动注入 `WHERE locations.company_id = 用户公司ID`
- **写入时**：`Company::getIdForCurrentUser()` 覆盖位置的 company_id，并检查父位置的公司一致性
- **效果**：公司A的用户只能看到公司A的位置，公司B的位置完全不可见

这意味着位置的 `company_id` 列构成了位置的**归属标记**——一个位置属于某个公司。当 `scope_locations_fmcs` 关闭时，位置跨公司共享；开启时，位置被公司隔离。

### 8.3 含义二：自定义报表中的位置属性过滤

自定义报表中的 `by_location_id` 和 `by_rtd_location_id` 过滤是**纯粹的属性匹配**：

```php
if ($request->filled('by_location_id')) {
    $assets->whereIn('assets.location_id', $request->input('by_location_id'));
}
```

这是对资产 `location_id` 外键的 `whereIn` 过滤，与公司范围是 AND 叠加关系。用户选择"位置=北京"时，实际条件是 `WHERE assets.company_id = 用户公司ID AND assets.location_id IN (北京ID)`。

**关键区别**：这里的"位置过滤"不是隔离机制，而是业务维度的查询条件。它只在公司范围约束的基础上进一步缩小结果集。

### 8.4 fmcs_location 验证：位置与公司的绑定约束

当 FMCS 和位置范围同时开启时，`fmcs_location` 验证规则确保模型的位置与其公司归属一致：

```php
if ($company_id != $location->company_id) {
    return false;  // 位置所属公司与模型所属公司不一致
}
```

这防止了"公司A的资产指定了公司B的位置"的逻辑错误。在此机制下，位置本质上是公司的一个**从属属性**——它不能独立于公司存在，而是公司的细分。

### 8.5 位置在 API 列表中的双重约束

[Api/LocationsController.php#L107-L109](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/LocationsController.php#L107-L109)：

```php
if (Setting::getSettings()->scope_locations_fmcs) {
    $locations = Company::scopeCompanyables($locations);
}
```

位置列表的过滤是**手动调用** `Company::scopeCompanyables()` 而非依赖 Global Scope。这是因为 Location 模型的 CompanyableScope 注册是条件性的（取决于 scope_locations_fmcs），而控制器中又做了一次显式检查——双重保障。

---

## 附录：关键文件索引

| 功能 | 文件位置 |
|---|---|
| FMCS 开关判断 | [Company.php#L97-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L97-L107) |
| 创建时公司ID注入 (getIdForCurrentUser) | [Company.php#L127-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L127-L145) |
| 用户模型公司ID注入 (getIdForUser) | [Company.php#L234-L241](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L234-L241) |
| 输入值清理 (getIdFromInput) | [Company.php#L109-L118](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L109-L118) |
| 无公司用户管理特权 | [Company.php#L203-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L203-L207) |
| 直接范围入口 Trait | [CompanyableTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/CompanyableTrait.php) |
| 子关联范围入口 Trait | [CompanyableChildTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/CompanyableChildTrait.php) |
| 子关联接口定义 | [ICompanyableChild.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/ICompanyableChild.php) |
| 直接范围 Global Scope | [CompanyableScope.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/CompanyableScope.php) |
| 子关联范围 Global Scope | [CompanyableChildScope.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/CompanyableChildScope.php) |
| 核心范围调度 (直接) | [Company.php#L289-L328](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L289-L328) |
| 子关联范围调度 | [Company.php#L339-L363](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L339-L363) |
| 访问权限判断 | [Company.php#L153-L196](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Company.php#L153-L196) |
| Gate Policy 守卫 | [SnipePermissionsPolicy.php#L38-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Policies/SnipePermissionsPolicy.php#L38-L73) |
| fmcs_location 验证 | [ValidationServiceProvider.php#L352-L365](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Providers/ValidationServiceProvider.php#L352-L365) |
| Actionlog 公司ID自动回填 | [Actionlog.php#L125-L201](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Actionlog.php#L125-L201) |
| Loggable 公司ID解析 | [Loggable.php#L233-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/Loggable.php#L233-L255) |
| Checkout 跨范围解析 | [Api/AssetsController.php#L878-L906](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L878-L906) |
| Checkout 公司不匹配校验 | [Api/AssetsController.php#L1054-L1062](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/AssetsController.php#L1054-L1062) |
| Location 写入时双重逻辑 | [LocationsController.php#L92-L103](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/LocationsController.php#L92-L103) |
| Location API 列表显式过滤 | [Api/LocationsController.php#L107-L109](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/Api/LocationsController.php#L107-L109) |
| CheckoutAcceptance 范围约束 | [ReportsController.php#L66-L70](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L66-L70) |
| 自定义报表导出查询 | [ReportsController.php#L714-L845](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Http/Controllers/ReportsController.php#L714-L845) |
| 搜索 Trait 过滤逻辑 | [Searchable.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Traits/Searchable.php) |
| LicenseSeat 子关联定义 | [LicenseSeat.php#L67-L70](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/LicenseSeat.php#L67-L70) |
| Maintenance 子关联定义 | [Maintenance.php#L110-L113](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Maintenance.php#L110-L113) |
| 用户模型(公司/位置字段) | [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/User.php) |
| 资产模型(CompanyableTrait) | [Asset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Asset.php) |
| 位置模型(条件性 Scope) | [Location.php](file:///d:/fz/0601-1/solo-dogfeeding/code/95-snipe-it/app/Models/Location.php) |
