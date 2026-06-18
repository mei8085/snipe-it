# 资产折旧与报废生命周期处理逻辑文档

## 一、整体架构概览

折旧与报废的处理链路涉及以下核心模块，层层递进：

```
折旧配置入口 → 折旧规则模型 → 计算Trait层 → 资产模型继承 → 状态标签体系 → 报废软删除 → API查询分流 → 账面价值展示
   │              │              │            │              │              │            │             │
   ▼              ▼              ▼            ▼              ▼              ▼            ▼             ▼
 Settings    Depreciation    Depreciable     Asset        Statuslabel   SoftDeletes   Api/Assets   Presenter
Controller     Model          Trait         Model          Model      (Eloquent)    Controller   /Transformer
```

---

## 二、状态标签体系（Statuslabel）

状态标签是资产生命周期的业务层核心标识，与报废（软删除）共同构成双层过滤机制。

### 2.1 状态标签的四象限分类

[Statuslabel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Statuslabel.php) L99-L110 定义了 `getStatuslabelType()` 方法，通过三个布尔字段的组合，将所有状态标签划分为**互斥的四类**：

| 状态类型 | `deployable` | `pending` | `archived` | 说明 |
|---------|-------------|-----------|------------|------|
| `pending` | 0 | 1 | 0 | 待处理：资产尚未就绪，等待审批/验收 |
| `archived` | 0 | 0 | 1 | 归档：资产已退出日常使用但保留记录 |
| `undeployable` | 0 | 0 | 0 | 不可部署：资产损坏、维修中、丢失等 |
| `deployable` | 1 | 0 | 0 | 可部署：资产正常可用（默认情况） |

**核心字段验证规则**（L32-L38）：
```php
protected $rules = [
    'name'       => 'required|max:255|string|unique_undeleted',
    'deployable' => 'required',
    'pending'    => 'required',
    'archived'   => 'required',
];
```

这三个字段**必须同时设置**且组合唯一，确保系统中不会出现模糊的状态定义。

### 2.2 状态标签对资产操作的影响

#### 自动签回触发（资产更新时）

当资产状态变更为 `archived` 或 `undeployable` 时，如果资产正被借出，系统会**强制自动签回**。

[AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Assets/AssetsController.php) L435-L441：

```php
$status = Statuslabel::find($request->input('status_id'));

// 状态变为 archived 或 undeployable 且资产已借出 → 自动签回
if (($status) 
    && (($status->getStatuslabelType() != 'pending') 
        && ($status->getStatuslabelType() != 'deployable'))  // 排除 pending 和 deployable
    && ($target = $asset->assignedTo)) {
    
    $originalValues = $asset->getRawOriginal();
    $asset->assigned_to = null;
    $asset->assigned_type = null;
    $asset->accepted = null;
    $asset->last_checkin = now();
    
    event(new CheckoutableCheckedIn(
        $asset, $target, auth()->user(), 
        'Checkin on asset update with '.$status->getStatuslabelType().' status',
        date('Y-m-d H:i:s'), $originalValues
    ));
}
```

**关键理解：** `pending` 和 `deployable` 状态不触发自动签回。这意味着待处理资产和可部署资产允许被持续借出，只有归档和不可部署状态会强制收回。

#### 签出/签入权限控制

[Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) L468-L502：

```php
// 可签出条件：未分配 + 未删除 + 状态非归档 + 状态可部署
public function availableForCheckout()
{
    if ((! $this->assigned_to) && (! $this->deleted_at)) {
        if (($this->status) && ($this->status->archived == '0')
            && ($this->status->deployable == '1')) {
            return true;
        }
    }
    return false;
}

// 可签入条件：已分配 或 已删除（特殊情况）
public function availableForCheckIn()
{
    if ($this->assigned_to == '') { return false; }
    if ($this->deleted_at != '') { return true; }  // 已删除资产始终允许签回
    
    return $this->status
        && ($this->status->archived == '0')
        && ($this->status->deployable == '1');
}
```

### 2.3 状态标签 vs 软删除（deleted_at）

这是两套**独立但互补**的机制：

| 维度 | `status_id` + `archived` | `deleted_at`（SoftDeletes） |
|------|-------------------------|-----------------------------|
| 层面 | 业务状态 | 数据存在状态 |
| 管理 | 自定义状态标签，用户可增删 | Laravel 内置机制，透明管理 |
| 查询排除 | 需要手动 JOIN 过滤 | 全局 Scope 自动排除 |
| 恢复方式 | 改回其他状态标签 | `restore()` 方法 |
| 对折旧影响 | 仅影响查询范围，不影响计算逻辑 | 仅影响查询范围，不影响计算逻辑 |
| 可同时存在 | ✅ 是的，可以同时设置 | ✅ 是的，可以同时设置 |

**双重过滤的典型场景**（`availableForCheckout()`）：
```
可签出 = (deleted_at 为空) AND (status.archived = 0) AND (status.deployable = 1)
```

---

## 三、折旧曲线参数体系

### 3.1 折旧规则定义（Depreciation 模型）

折旧规则定义在 [Depreciation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciation.php)，这是整个折旧体系的参数源头。

**核心字段：**

| 字段 | 类型 | 说明 | 验证规则 |
|------|------|------|----------|
| `name` | string | 折旧规则名称（如："电子产品3年折旧"） | 必填，最长255，唯一 |
| `months` | int | 折旧总月数 | 必填，最大3600（300年） |
| `depreciation_type` | enum | 残值类型：`amount`（固定金额）/ `percent`（百分比） | 必填，二选一 |
| `depreciation_min` | numeric | 残值值。当 type=percent 时取值范围 0-100 | 必填，数值型 |

**创建与管理入口：** [DepreciationsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/DepreciationsController.php) 的 `store()` / `update()` 方法（L58-L91, L123-L152）。

### 3.2 全局折旧方法选择（Setting）

系统级折旧计算策略由 `Settings` 表的 `depreciation_method` 字段控制，在 [SettingsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/SettingsController.php) L129 处写入。

三种折旧算法：

| 方法名 | 说明 | 调用函数 |
|--------|------|----------|
| `linear`（默认） | 直线法/平均年限法：每月折旧额固定 | `getLinearDepreciatedValue()` |
| `half_1` | 半年惯例（强制半年首年）：首年统一只计半年折旧 | `getHalfYearDepreciatedValue(true)` |
| `half_2` | 半年惯例（按购入时点）：年中购入才只计半年 | `getHalfYearDepreciatedValue(false)` |

方法调度入口在 [Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) L52-L63 的 `getDepreciatedValue()`。

---

## 四、折旧计算详解（Depreciable Trait）

### 4.1 残值计算（calculateDepreciation）

[Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) L231-L243：

```php
private function calculateDepreciation()
{
    if ($this->get_depreciation()->depreciation_type === 'percent') {
        $depreciation_percent = $this->get_depreciation()->depreciation_min / 100;
        return $this->purchase_cost * $depreciation_percent;   // 百分比残值：成本 × 比例
    }
    return $this->get_depreciation()->depreciation_min;        // 固定残值：直接取配置值
}
```

### 4.2 直线法（getLinearDepreciatedValue）

**情形 A：已超过折旧年限**
- 若配置了残值 → 账面价值 = 残值
- 若无残值配置 → 账面价值 = 0

**情形 B：仍在折旧年限内**
```
账面价值 = 购入成本 − (购入成本 − 残值) × (已过月数 / 总折旧月数)
```

**每月折旧额：**
```php
getMonthlyDepreciation() = (purchase_cost - calculateDepreciation()) / months
```

### 4.3 半年惯例法（getHalfYearDepreciatedValue）

适用于税务折旧场景（如美国 MACRS 半年惯例），核心是**按财年**而非自然月计算：
1. 财年规则：12-31 视为当年，其他日期视为上一年
2. 首年调整：`half_1` 强制半年，`half_2` 仅年中购入才算半年
3. 当年调整：当前日期在下半年时加 0.5 年
4. 年限被夹在 `[0, 折旧年限]` 之间

### 4.4 资产如何获取折旧配置

[Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) L695-L700 的 `get_depreciation()` 方法：

```php
public function get_depreciation()
{
    if (($this->model) && ($this->model->depreciation)) {
        return $this->model->depreciation;
    }
}
```

**关键理解：资产自身不持有折旧ID，而是通过 AssetModel 间接获取折旧规则。** 同一型号的所有资产共享同一条折旧曲线参数。

---

## 五、API 查询分流与列表过滤

资产列表的查询分流逻辑集中在 [Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) 的 `index()` 方法（L65-L305），这是整个系统最核心的查询入口。

### 5.1 状态类型查询分流（status_type）

L227-L300 通过 `status_type`（或兼容旧版的 `status`）参数，将查询导向不同分支：

```php
$status_type_key = $request->filled('status_type') 
    ? $request->input('status_type') 
    : $request->input('status');

switch ($status_type_key) {
    case 'Deleted':
        $assets->onlyTrashed();   // 仅已删除（软删）资产
        break;
    case 'Pending':
        $assets->join('status_labels AS status_alias', function ($join) {
            $join->on('status_alias.id', '=', 'assets.status_id')
                ->where('status_alias.deployable', '=', 0)
                ->where('status_alias.pending', '=', 1)
                ->where('status_alias.archived', '=', 0);
        });
        break;
    case 'RTD':  // Ready To Deploy
        $assets->whereNull('assets.assigned_to')
            ->join('status_labels AS status_alias', function ($join) {
                $join->on('status_alias.id', '=', 'assets.status_id')
                    ->where('status_alias.deployable', '=', 1)
                    ->where('status_alias.pending', '=', 0)
                    ->where('status_alias.archived', '=', 0);
            });
        break;
    case 'Undeployable':
        $assets->Undeployable();   // 调用模型 scope
        break;
    case 'Archived':
        $assets->join('status_labels AS status_alias', function ($join) {
            $join->on('status_alias.id', '=', 'assets.status_id')
                ->where('status_alias.deployable', '=', 0)
                ->where('status_alias.pending', '=', 0)
                ->where('status_alias.archived', '=', 1);
        });
        break;
    case 'Deployed':
        $assets->whereNotNull('assets.assigned_to');  // 只要已分配就算 Deployed
        break;
    default:
        // 默认情况：根据 show_archived_in_list 设置决定是否排除归档
        if ((! $request->filled('status_id')) && ($settings->show_archived_in_list != '1')) {
            $assets->join('status_labels AS status_alias', function ($join) {
                $join->on('status_alias.id', '=', 'assets.status_id')
                    ->where('status_alias.archived', '=', 0);  // 排除归档
            });
        } else {
            $assets->join('status_labels AS status_alias', function ($join) {
                $join->on('status_alias.id', '=', 'assets.status_id');  // 纯 JOIN，不过滤
            });
        }
}
```

### 5.2 查询分流全景图

| status_type 值 | 查询条件 | 包含 deleted_at != '' 的资产？ |
|---------------|---------|-------------------------------|
| `Deleted` | `onlyTrashed()` | ✅ **仅包含**（已软删的） |
| `Pending` | status.pending = 1 | ❌ 不包含（SoftDeletes 全局排除） |
| `RTD` | status.deployable = 1 AND assigned_to IS NULL | ❌ 不包含 |
| `Undeployable` | 调用 scopeUndeployable() | ❌ 不包含 |
| `Archived` | status.archived = 1 | ❌ 不包含 |
| `Deployed` | assigned_to IS NOT NULL | ❌ 不包含 |
| 默认（无参数） | 根据 `show_archived_in_list` 决定是否排除 archived=1 | ❌ 不包含 |

### 5.3 归档显示开关（show_archived_in_list）

[SettingsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/SettingsController.php) L115：

```php
$setting->show_archived_in_list = $request->input('show_archived_in_list', '0');
```

**行为：**
- `show_archived_in_list = 0`（默认）：默认列表不显示归档资产（status.archived = 1）
- `show_archived_in_list = 1`：默认列表显示归档资产
- 仅当**未指定** `status_id` 时生效（L287），如果明确指定了状态 ID 则不过滤

### 5.4 模型层查询范围（Scopes）

[Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) 中定义了一系列 scope：

| Scope 方法 | 位置 | 作用 |
|-----------|------|------|
| `scopePending()` | L1483-L1488 | 状态为 pending 的资产 |
| `scopeRTD()` | L1536-L1542 | 就绪待部署资产 |
| `scopeUndeployable()` | L1551-L1556 | 不可部署资产 |
| `scopeNotArchived()` | L1565-L1570 | 非归档资产 |
| `scopeArchived()` | L1751-L1756 | 已归档资产 |
| `scopeDeployed()` | L1765-L1767 | 已分配资产 |
| `scopeAssetsForShow()` | L1731-L1742 | 受 `show_archived_in_list` 控制的显示范围 |
| `scopeByDepreciationId()` | L2153-L2157 | 按折旧规则 ID 过滤 |

**这些 scopes 同样受 SoftDeletes 全局 Scope 影响**，除非显式调用 `withTrashed()`，否则自动排除已软删资产。

### 5.5 折旧报告的专用查询路径

当路由为 `api.depreciation-report.index` 时，查询路径发生特殊分支（L85-L88, L173-L176）：

```php
if (Route::currentRouteName() == 'api.depreciation-report.index') {
    $filter_non_deprecable_assets = true;
    $transformer = 'App\Http\Transformers\DepreciationReportTransformer';
    $this->authorize('reports.view');
}
```

后续 L173-L176 自动过滤没有折旧配置的资产型号：
```php
if ($filter_non_deprecable_assets) {
    $non_deprecable_models = AssetModel::select('id')->whereNotNull('depreciation_id')->get();
    $assets->InModelList($non_deprecable_models->toArray());
}
```

**这意味着折旧报告中**不会**出现未配置折旧规则的资产。**

同时支持按折旧规则过滤（L362-L363）：
```php
if ($request->filled('depreciation_id')) {
    $assets->ByDepreciationId($request->input('depreciation_id'));
}
```

### 5.6 折旧报告归档筛选三路径对比

折旧报告的前端入口在 [depreciation.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/resources/views/reports/depreciation.blade.php)，默认数据 URL 为：
```
data-url="{{ route('api.depreciation-report.index') }}"
```

路由定义在 [api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/routes/api.php) L359-L364：
```php
Route::get('reports/depreciation',
    [Api\AssetsController::class, 'index']
)->name('api.depreciation-report.index');
```

**关键洞察：折旧报告复用了 Api/AssetsController 的同一个 `index()` 方法，通过路由名识别上下文切换行为。**

#### 关于"无折旧配置资产"的精确定义

"无折旧配置"在代码中有两层过滤，必须同时满足才会计算并显示账面价值：

**第一层（查询层，L173-L176）：**
```php
if ($filter_non_deprecable_assets) {
    $non_deprecable_models = AssetModel::select('id')->whereNotNull('depreciation_id')->get();
    $assets->InModelList($non_deprecable_models->toArray());
}
```
过滤掉资产型号中 `depreciation_id IS NULL` 的所有资产。这层在 switch 之前执行，对所有三条 API 路径都生效。

**第二层（Transformer 层，DepreciationReportTransformer.php L66）：**
```php
if (($asset->model) && ($asset->model->depreciation) && ($asset->model->depreciation->months !== 0)) {
    // 计算账面价值
} elseif ($asset->model->eol !== null) {
    // 仅计算 monthly_depreciation（无账面价值和 diff）
}
```
即使通过了第一层过滤，如果折旧规则的 `months = 0`，也不会计算账面价值；此时若型号有 EOL（使用年限）字段，则回退到只计算 monthly_depreciation。

---

#### 路径一：默认报告（无 status_type 参数）

对应前端页面直接访问 `GET /reports/depreciation`。

[Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) 执行流程：
1. L85 路由名匹配 → `$filter_non_deprecable_assets = true`
2. L173-L176 过滤无折旧配置的资产型号（第一层过滤）
3. L227-L232 未传入 `status_type` → `$status_type_key = null`
4. L234 switch 进入 `default` 分支
5. L287-L299 根据 `show_archived_in_list` 决定归档资产是否显示：

```php
default:
    if ((! $request->filled('status_id')) && ($settings->show_archived_in_list != '1')) {
        $assets->join('status_labels AS status_alias', function ($join) {
            $join->on('status_alias.id', '=', 'assets.status_id')
                ->where('status_alias.archived', '=', 0);  // 排除归档
        });
    } else {
        $assets->join('status_labels AS status_alias', function ($join) {
            $join->on('status_alias.id', '=', 'assets.status_id');  // 纯 JOIN，不过滤
        });
    }
```

**show_archived_in_list 开关的生效条件（L287）：**
```php
if ((! $request->filled('status_id')) && ($settings->show_archived_in_list != '1'))
```
两个条件必须同时满足：
1. **未显式指定 `status_id`**（指定了具体状态 ID 则不过滤）
2. **`show_archived_in_list != '1'`**（开关关闭）

开关在 [SettingsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/SettingsController.php) L115 写入，默认值为 `'0'`。

**默认报告的完整行为：**
- `show_archived_in_list = 0`（默认）→ **不显示归档资产**
- `show_archived_in_list = 1` → **显示归档资产**
- 无论开关如何，**不显示已软删资产**（无 `withTrashed()` 调用，受 SoftDeletes 全局 Scope 限制）
- 经过第一层和第二层过滤后，才计算并显示账面价值

#### 路径二：显式归档筛选（?status_type=Archived）

对应 API 调用 `GET /api/reports/depreciation?status_type=Archived`。

[Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) L258-L264：
```php
case 'Archived':
    $assets->join('status_labels AS status_alias', function ($join) {
        $join->on('status_alias.id', '=', 'assets.status_id')
            ->where('status_alias.deployable', '=', 0)
            ->where('status_alias.pending', '=', 0)
            ->where('status_alias.archived', '=', 1);  // 仅归档
    });
    break;
```

**显式归档筛选的行为：**
- **仅显示归档资产**（`archived = 1`，且 `deployable = 0` 且 `pending = 0`，确保"纯归档"状态）
- **不受 `show_archived_in_list` 影响**（switch 已匹配 `case 'Archived'`，不进入 default 分支，因此开关逻辑不执行）
- **不显示已软删资产**（无 `withTrashed()` 调用）
- 经过第一层和第二层折旧过滤后，才计算并显示账面价值

#### 路径三：已删除资产筛选（?status_type=Deleted）

对应 API 调用 `GET /api/reports/depreciation?status_type=Deleted`。

[Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) L235-L237：
```php
case 'Deleted':
    $assets->onlyTrashed();  // 仅已软删
    break;
```

**已删除资产筛选的行为：**
- **仅显示已软删资产**（`deleted_at IS NOT NULL`）
- **不 JOIN status_labels**（代码中没有 JOIN），因此**完全不考虑 archived/pending/deployable 状态**——已软删资产无论是否归档、是否可部署都会显示
- **不受 `show_archived_in_list` 影响**（不进入 default 分支）
- 经过第一层和第二层折旧过滤后，才计算并显示账面价值

---

#### 路径四（独立）：CSV 导出

CSV 导出不走 API 路径，而是走 [ReportsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/ReportsController.php) 的 `exportDeprecationReport()` 方法（L174-L245）。

**核心查询（L195-L197）：**
```php
Asset::with('model', 'assignedTo', 'status', 'defaultLoc', 'assetlog')
    ->orderBy('created_at', 'DESC')
    ->chunk(500, function ($assets) use ($handle, $formatter) {
```

**CSV 导出的关键差异：**
- ❌ **不过滤无折旧配置资产**（无 L173-L176 的 AssetModel 过滤）
- ❌ **不按 show_archived_in_list 过滤归档**（无 switch 和 default 分支）
- ❌ **不接受 status_type 参数**（硬编码查询）
- ❌ **不显示已软删资产**（无 `withTrashed()` 调用，受 SoftDeletes 全局 Scope 限制）
- ✅ 账面价值计算（L229）：`$asset->getDepreciatedValue()`，与 API 路径使用完全相同的方法

**但注意：** 即使 CSV 导出不过滤无折旧配置的资产，`getDepreciatedValue()` 内部仍会检查折旧规则——若折旧规则不存在或 months=0，将返回 `purchase_cost`（见 Depreciable.php L43-L48），而非 null。

---

#### 四路径对比总表（统一版）

| 路径 | 触发方式 | 归档资产 | 已软删资产 | 无折旧配置资产 | 受 show_archived_in_list 影响？ |
|------|---------|---------|-----------|--------------|--------------------------------|
| 默认报告 | `GET /api/reports/depreciation`（无参数） | ✅/❌ 取决于开关 | ❌ 不显示 | ❌ 两层过滤后排除 | ✅ 是 |
| 显式归档 | `GET /api/reports/depreciation?status_type=Archived` | ✅ 仅显示归档 | ❌ 不显示 | ❌ 两层过滤后排除 | ❌ 否 |
| 已删除 | `GET /api/reports/depreciation?status_type=Deleted` | ⚠️ 完全忽略归档状态 | ✅ 仅显示已删 | ❌ 两层过滤后排除 | ❌ 否 |
| CSV 导出 | `GET /reports/depreciation/export` | ✅ 全部显示 | ❌ 不显示 | ✅ 显示（返回 purchase_cost） | ❌ 否 |

---

#### 账面价值显示的一致性（统一结论）

**计算层：** 无论哪条路径，一旦资产被查询出来，`getDepreciatedValue()` 的计算逻辑完全相同（无 deleted_at/status_id/archived 判断）。

**Transformer 层（仅 API 路径）：** 在计算前额外检查 `$asset->model->depreciation->months !== 0`，若不满足则：
- `book_value` 和 `diff` 为 null（或空值）
- 若型号有 EOL 字段，`monthly_depreciation` 回退为 `purchase_cost / eol`

**CSV 导出层：** 无 Transformer 层检查，直接调用 `getDepreciatedValue()`，无折旧配置时返回 `purchase_cost`。

**各路径的账面价值显示差异仅在于：**
1. **哪些资产被包含在结果集中**（由查询过滤决定）
2. **无折旧配置资产的处理方式**（API 路径：不显示账面价值；CSV 路径：显示为 purchase_cost）
3. **账面价值的汇总（sumFormatter）会因为包含/排除归档/已删除资产而不同**
4. **计算逻辑本身没有任何差异**

#### 前端表格列配置

[DepreciationReportPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Presenters/DepreciationReportPresenter.php) L142-L149 定义了账面价值列：
```php
[
    'field'           => 'book_value',
    'searchable'      => true,
    'sortable'        => false,
    'visible'         => true,
    'title'           => trans('admin/hardware/table.book_value'),
    'footerFormatter' => 'sumFormatter',   // 底部自动汇总
    'class'           => 'text-right',
],
```

注意：`searchable = true` 意味着前端可以搜索账面价值，但**后端排序被禁用**（`sortable = false`），因为它是计算属性而非数据库字段。

---

## 六、报废（删除/恢复）处理流程

### 6.1 报废触发（destroy 方法）

入口在 [AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Assets/AssetsController.php) L536-L561：

```php
public function destroy(Request $request, Asset $asset): RedirectResponse
{
    $this->authorize('delete', $asset);
    
    // 步骤1：若资产正被借出 → 自动签回
    if ($asset->assignedTo) {
        event(new CheckoutableCheckedIn(...));
        DB::table('assets')
            ->where('id', $asset->id)
            ->update(['assigned_to' => null, 'assigned_type' => null]);
    }
    
    // 步骤2：删除资产图片文件
    if ($asset->image) { Storage::disk('public')->delete('assets/'.basename($asset->image)); }
    
    // 步骤3：软删除（设置 deleted_at 时间戳）
    $asset->delete();
    
    return redirect()->route('hardware.index');
}
```

**模型事件：** [Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) L235-L244 注册了软删除监听：
```php
static::softDeleted(function (Asset $asset) {
    $asset->requests()->delete();  // 连带删除该资产的领用请求
});
```

### 6.2 报废资产的恢复（restore）

L942-L966 的 `getRestore()` 方法：

```php
public function getRestore($assetId = null)
{
    // 必须用 withTrashed() 查询，否则查不到软删除记录
    if ($asset = Asset::withTrashed()->find($assetId)) {
        $this->authorize('delete', $asset);
        
        if ($asset->deleted_at == '') { /* 防止重复恢复 */ }
        
        // 恢复（清空 deleted_at），若资产标签冲突会触发 UniqueUndeletedTrait 校验失败
        if ($asset->restore()) { ... }
    }
}
```

---

## 七、报废/归档对账面价值展示的影响链路

### 7.1 核心原则：计算与展示分离

**账面价值的计算逻辑完全不关心资产状态。**

[Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) L41-L66 的 `getDepreciatedValue()` 方法中：
- ❌ 没有任何 `deleted_at` 判断
- ❌ 没有任何 `status_id` 判断
- ❌ 没有任何 `archived` 判断
- ✅ 只依赖 `purchase_date`、`purchase_cost`、`get_depreciation()` 返回的折旧规则

### 7.2 状态影响展示的唯一途径：查询过滤

状态标签和 `deleted_at` **唯一影响账面价值展示的方式是决定哪些资产会被查询出来**，进入 Transformer 层。

**完整链路：**
```
前端请求（带 status_type 参数）
    ↓
Api/AssetsController::index()  → 应用查询过滤（JOIN status_labels + withTrashed/onlyTrashed）
    ↓
查询结果集合（只包含符合过滤条件的资产）
    ↓
AssetsTransformer::transformAsset()  → 对每个资产调用 $asset->getDepreciatedValue()
    ↓
JSON 响应中的 book_value 字段
```

### 7.3 各状态下的账面价值可见性

| 资产状态 | 常规列表（无参数） | ?status_type=Deleted | ?status_type=Archived | 折旧报告 |
|---------|-------------------|---------------------|------------------------|---------|
| 正常，deployable | ✅ 可见 | ❌ 不可见 | ❌ 不可见 | ✅ 可见（有折旧配置） |
| 正常，archived | ✅/❌ 取决于 show_archived_in_list | ❌ 不可见 | ✅ 可见 | ✅ 可见（有折旧配置） |
| 正常，undeployable | ✅ 可见 | ❌ 不可见 | ❌ 不可见 | ✅ 可见（有折旧配置） |
| 已软删（deleted_at != ''） | ❌ 不可见 | ✅ 可见 | ❌ 不可见 | ❌ 不可见（无 withTrashed） |
| 已软删 + archived | ❌ 不可见 | ✅ 可见 | ❌ 不可见 | ❌ 不可见 |

### 7.4 特殊场景说明

#### 场景 A：已软删资产的账面价值
```
GET /api/hardware?status_type=Deleted
    → onlyTrashed() → 查询出已软删资产
    → 每个资产依然正常调用 getDepreciatedValue()
    → 折旧继续按时间推进，不因为 deleted_at 而停止
    → 响应中包含正常计算的 book_value
```

#### 场景 B：归档资产的账面价值
```
GET /api/hardware?status_type=Archived
    → JOIN status_labels where archived = 1
    → 资产账面价值正常计算
    → 状态变更为归档不影响折旧计算，只影响是否显示
```

#### 场景 C：资产状态变更为 archived 时的连锁反应
```
1. 更新资产 status_id 为 archived 状态
2. AssetsController::update() 检测到状态变更
3. 若资产已借出 → 自动签回（assigned_to 清空）
4. 资产保存成功
5. 下次查询时，除非 show_archived_in_list = 1 或指定 ?status_type=Archived，否则不显示
6. 但只要能查询到，账面价值依然按原规则计算
```

#### 场景 D：折旧报告中的已删除资产
折旧报告查询路径**没有**调用 `withTrashed()`，因此**已软删资产不会出现在折旧报告中**。

但**已归档资产会正常出现在折旧报告中**（只要有折旧配置），因为归档只是业务状态，不是数据删除。

### 7.5 无折旧配置的资产

当资产型号未关联折旧规则时：
- `get_depreciation()` 返回 `null`
- `getDepreciatedValue()` 直接返回 `purchase_cost`（L43-L45）
- 即：无折旧配置的资产，账面价值永远等于购入成本
- **折旧报告自动排除这类资产**（通过 L173-L176 过滤）

---

## 八、典型场景数据流

### 场景1：资产报废 → 在已删除列表查看账面价值 → 恢复

```
1. 资产正常使用中，账面价值按月递减
2. 点击"删除" → AssetsController::destroy()
   → 若已借出则自动签回
   → SoftDeletes 设置 deleted_at
   → 资产从默认列表中消失
3. 导航到"已删除资产" → GET /api/hardware?status_type=Deleted
   → Api/AssetsController 调用 onlyTrashed()
   → 查询出所有已软删资产
   → 每个资产正常调用 getDepreciatedValue()
   → 账面价值继续按时间递减（未因为报废而冻结）
4. 点击"恢复" → AssetsController::getRestore()
   → restore() 清空 deleted_at
   → 资产重新出现在常规列表
   → 账面价值无缝衔接，无任何"中断续接"问题
```

### 场景2：资产状态变更为归档

```
1. 资产状态更新为"已报废"（标签类型为 archived）
   → AssetsController::update() 检测到状态变更为非 deployable
   → 若已借出则自动签回
2. 默认列表中（show_archived_in_list = 0）
   → 查询自动加上 status_alias.archived = 0 条件
   → 该资产不再显示
3. 切换到"已归档"视图 → GET /api/hardware?status_type=Archived
   → JOIN status_labels where archived = 1
   → 资产正常显示，账面价值正常计算
4. 折旧报告中该资产依然存在（有折旧配置的话）
   → 账面价值继续按月计算
   → 归档是业务状态，不影响折旧曲线
```

### 场景3：折旧报告查看特定折旧规则的资产

```
1. GET /api/reports/depreciation?depreciation_id=5
   → 路由命中 api.depreciation-report.index
   → filter_non_deprecable_assets = true
   → 过滤掉无 depreciation_id 的资产型号
   → 按 depreciation_id = 5 过滤
   → 使用 DepreciationReportTransformer
2. Transformer 输出更详细的折旧信息：
   - book_value（当前账面价值）
   - monthly_depreciation（每月折旧额）
   - diff（累计折旧 = purchase_cost - book_value）
   - number_of_months（折旧总月数）
3. 已软删资产不出现在报告中（无 withTrashed）
4. 已归档资产正常出现在报告中
```

---

## 九、关键代码速查表（统一版）

| 功能 | 文件 | 方法/位置 |
|------|------|-----------|
| 状态标签四象限分类 | [Statuslabel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Statuslabel.php) | `getStatuslabelType()` L99 |
| 资产更新时自动签回 | [AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Assets/AssetsController.php) | L435-L441 |
| 可签出检查 | [Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) | `availableForCheckout()` L468 |
| API 查询分流（switch 分支） | [Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) | `index()` L227-L300 |
| 折旧报告路由名识别 | [Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) | L85-L88 |
| 折旧报告第1层过滤（depreciation_id IS NOT NULL） | [Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) | L173-L176 |
| 折旧报告第2层过滤（months !== 0 + EOL fallback） | [DepreciationReportTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Transformers/DepreciationReportTransformer.php) | L66-L72 |
| 折旧报告 CSV 导出（独立查询，无 status_type/折旧过滤） | [ReportsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/ReportsController.php) | `exportDeprecationReport()` L174-L245 |
| CSV 导出账面价值计算（直接调用 getDepreciatedValue） | [ReportsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/ReportsController.php) | L229-L230 |
| 默认报告归档过滤（show_archived_in_list 生效条件） | [Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) | L287-L299 |
| 显式归档筛选（case 'Archived'） | [Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) | L258-L264 |
| 已删除筛选（case 'Deleted'，不 JOIN status_labels） | [Api/AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Api/AssetsController.php) | L235-L237 |
| 折旧规则模型 | [Depreciation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciation.php) | 全文 |
| 折旧计算核心（无状态判断） | [Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) | `getDepreciatedValue()` L41-L66 |
| 无折旧规则时回退为 purchase_cost | [Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) | L43-L48 |
| 报废（软删） | [AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Assets/AssetsController.php) | `destroy()` L536 |
| 恢复资产 | [AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Assets/AssetsController.php) | `getRestore()` L942 |
| 软删模型事件 | [Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) | `booted()` L235-L244 |
| 资产列表账面价值输出 | [AssetsTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Transformers/AssetsTransformer.php) | L127 |
| 折旧报告账面价值输出 | [DepreciationReportTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Transformers/DepreciationReportTransformer.php) | `transformAsset()` L33, L66-L72, L103 |
| 折旧报告表格列配置 | [DepreciationReportPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Presenters/DepreciationReportPresenter.php) | book_value L142-L149 |
| 资产列表表格列配置 | [AssetPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Presenters/AssetPresenter.php) | book_value L181 |
| 归档显示开关写入（默认 '0'） | [SettingsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/SettingsController.php) | L115 |
| 折旧方法全局设置 | [SettingsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/SettingsController.php) | L129 |
| 折旧报告路由定义 | [api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/routes/api.php) | L359-L364 |
| 折旧报告前端入口 | [depreciation.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/resources/views/reports/depreciation.blade.php) | L28 |

---

## 十、核心设计洞察（统一结论）

### 1. 计算与查询的关注点分离
账面价值计算（Depreciable Trait）与资产状态（Statuslabel + SoftDeletes）完全解耦。计算层只关心时间和折旧规则（无 deleted_at/status_id/archived 判断），查询层负责根据状态过滤。这种设计使得：
- 折旧算法可以独立演进
- 新状态类型的添加不影响折旧逻辑
- 已删除/归档资产的历史账面价值可追溯

### 2. 双层过滤机制的设计意图
- `deleted_at` 处理"数据存在性"：资产是否还在系统中"活着"
- `status_id` + `archived` 处理"业务可用性"：资产是否还在正常使用

两者结合提供了比单一状态字段更丰富的生命周期表达能力。但两者**互不干扰**——`case 'Deleted'` 路径甚至不 JOIN status_labels，完全忽略 archived 状态。

### 3. 折旧报告的三层过滤设计（API 路径）
折旧报告 API 路径的账面价值显示经过三层控制，每层的职责明确：
1. **查询层第1层**（L173-L176）：过滤 `depreciation_id IS NULL` 的资产型号，确保只有关联了折旧规则的资产才进入结果集
2. **查询层第2层**（switch 分支）：根据 `status_type` 决定归档/已删除资产的可见性
3. **Transformer 层**（L66-L72）：检查 `months !== 0`，若不满足则不计算账面价值（但有 EOL 时回退计算 monthly_depreciation）

### 4. CSV 导出与 API 路径的有意分离
CSV 导出采用独立查询路径，完全绕过 API 路径的三层过滤设计：
- 不过滤无折旧配置的资产
- 不按 archived 过滤
- 不接受 status_type 参数
- 无折旧配置时账面价值显示为 purchase_cost

这种设计使得 CSV 导出可以作为"全量数据快照"，而 API 路径专注于"折旧相关数据视图"，两者服务于不同的使用场景。

### 5. 四路径筛选的设计权衡
折旧报告的四路径筛选（默认/显式归档/已删除/CSV 导出）体现了清晰的权责分离：
- **默认路径**：面向日常运营，按 show_archived_in_list 开关决定是否显示归档，排除已删除
- **显式归档路径**：面向档案管理，专门查看已退出使用的资产（archived=1, deployable=0, pending=0）
- **已删除路径**：面向审计和资产处置，追踪已报废资产的剩余价值（完全忽略 archived 状态）
- **CSV 导出**：面向财务全量快照，包含所有非删除资产的完整数据

所有路径共享同一个 `getDepreciatedValue()` 计算逻辑，确保账面价值计算口径的一致性。

### 6. 汇总值差异的设计意图
账面价值汇总值在不同路径下存在差异并非设计缺陷，而是有意为之。差异来源于查询层对以下四类资产的包含/排除：
1. 已软删资产（deleted_at IS NOT NULL）
2. 归档资产（status_labels.archived = 1）
3. 无折旧配置资产（AssetModel.depreciation_id IS NULL）
4. 折旧月数为0的资产（depreciation.months = 0）

不同路径包含不同的资产集合，因此汇总值不同：
- 运营视角（默认路径）：只包含在用、有折旧配置且 archvied=0 的资产
- 财务视角（CSV 导出）：包含全部非删除资产（含归档、含无折旧配置）
- 档案视角（显式归档）：只包含 archived=1 且有折旧配置的资产
- 审计视角（已删除路径）：只包含 deleted_at IS NOT NULL 且有折旧配置的资产
- 不同汇总值服务于不同决策场景，通过路径切换实现数据的多维度呈现
