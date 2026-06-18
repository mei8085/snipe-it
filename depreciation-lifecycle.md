# 资产折旧与报废生命周期处理逻辑文档

## 一、整体架构概览

折旧与报废的处理链路涉及以下核心模块，层层递进：

```
折旧配置入口 → 折旧规则模型 → 计算Trait层 → 资产模型继承 → 报废软删除 → 账面价值展示
   │              │              │            │              │           │
   ▼              ▼              ▼            ▼              ▼           ▼
 Settings    Depreciation    Depreciable     Asset     SoftDeletes     Presenter
Controller     Model          Trait         Model      (Eloquent)    /Transformer
```

---

## 二、折旧曲线参数体系

### 2.1 折旧规则定义（Depreciation 模型）

折旧规则定义在 [Depreciation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciation.php)，这是整个折旧体系的参数源头。

**核心字段：**

| 字段 | 类型 | 说明 | 验证规则 |
|------|------|------|----------|
| `name` | string | 折旧规则名称（如："电子产品3年折旧"） | 必填，最长255，唯一 |
| `months` | int | 折旧总月数 | 必填，最大3600（300年） |
| `depreciation_type` | enum | 残值类型：`amount`（固定金额）/ `percent`（百分比） | 必填，二选一 |
| `depreciation_min` | numeric | 残值值。当 type=percent 时取值范围 0-100 | 必填，数值型 |

**关系链：**
- `Depreciation` → `hasMany(AssetModel::class)`：折旧规则被多个资产型号引用
- `Depreciation` → `hasManyThrough(Asset::class, AssetModel::class)`：通过型号间接关联实际资产

**创建与管理入口：** [DepreciationsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/DepreciationsController.php) 的 `store()` / `update()` 方法（L58-L91, L123-L152）。

### 2.2 全局折旧方法选择（Setting）

系统级折旧计算策略由 `Settings` 表的 `depreciation_method` 字段控制，在 [SettingsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/SettingsController.php) L129 处写入。

三种折旧算法：

| 方法名 | 说明 | 调用函数 |
|--------|------|----------|
| `linear`（默认） | 直线法/平均年限法：每月折旧额固定 | `getLinearDepreciatedValue()` |
| `half_1` | 半年惯例（强制半年首年）：首年统一只计半年折旧 | `getHalfYearDepreciatedValue(true)` |
| `half_2` | 半年惯例（按购入时点）：年中购入才只计半年 | `getHalfYearDepreciatedValue(false)` |

方法调度入口在 [Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) L52-L63 的 `getDepreciatedValue()`：

```php
switch ($setting->depreciation_method) {
    case 'half_1':  $depreciation = $this->getHalfYearDepreciatedValue(true);  break;
    case 'half_2':  $depreciation = $this->getHalfYearDepreciatedValue(false); break;
    default:        $depreciation = $this->getLinearDepreciatedValue();        break;
}
```

---

## 三、折旧计算详解（Depreciable Trait）

### 3.1 残值计算（calculateDepreciation）

在 [Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) L231-L243：

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

### 3.2 直线法（getLinearDepreciatedValue）

L71-L95，核心公式：

**情形 A：已超过折旧年限**
- 若配置了残值（`depreciation_min` 非空）→ 账面价值 = 残值（通过 `calculateDepreciation()` 计算）
- 若无残值配置 → 账面价值 = 0

**情形 B：仍在折旧年限内**

```
账面价值 = 购入成本 − (购入成本 − 残值) × (已过月数 / 总折旧月数)
```

对应代码 L90：
```php
$current_value = round(
    ($this->purchase_cost - ($this->purchase_cost - $this->calculateDepreciation())) 
    * ($months_passed / $this->get_depreciation()->months),
    2
);
```

注意：此处公式展开后等价于 `purchase_cost - (cost - floor) * (elapsed/total)`，符合标准直线折旧法。

**每月折旧额** 在 L97-L102：
```php
getMonthlyDepreciation() = (purchase_cost - calculateDepreciation()) / months
```

### 3.3 半年惯例法（getHalfYearDepreciatedValue）

L108-L133，适用于税务折旧场景（如美国 MACRS 半年惯例）。

**核心逻辑：**
1. 将折旧月数转换为年度：`ceil(months / 12)`
2. 以财年（fiscal year）为计算单位：`12-31` 视为当年，其他日期视为上一年（见 L139-L148 `get_fiscal_year()`）
3. 首年调整：
   - `half_1`（强制）：首年永远只算 0.5 年
   - `half_2`（时点）：仅当年中购入（6月1日之后，L154-L159 `is_first_half_of_year()`）才只算 0.5 年
4. 当年调整：当前日期在下半年时，再加 0.5 年
5. 最终年限被夹在 `[0, 折旧年限]` 之间

**账面价值公式：**
```
账面价值 = 购入成本 − round(已过年数 / 折旧年限 × 购入成本, 2)
```

### 3.4 折旧完成日期与进度

- **`depreciated_date()`**（L180-L190）：`purchase_date + months`
- **`depreciationProgressPercent()`**（L196-L206）：已过月数 / 总月数 × 100%，限制在 0-100%

### 3.5 资产模型如何获取折旧配置

[Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) L695-L700 的 `get_depreciation()` 方法：

```php
public function get_depreciation()
{
    if (($this->model) && ($this->model->depreciation)) {
        return $this->model->depreciation;
    }
}
```

**关键理解：资产自身不持有折旧ID，而是通过 AssetModel 间接获取折旧规则。** 这意味着同一型号的所有资产共享同一条折旧曲线参数。关系定义在 L664-L667（`hasOneThrough`）。

---

## 四、报废（删除/恢复）处理流程

Snipe-IT 中资产"报废"对应 Eloquent 的 **SoftDeletes** 软删除机制，而非物理删除。

### 4.1 报废触发（destroy 方法）

入口在 [AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Assets/AssetsController.php) L536-L561：

```php
public function destroy(Request $request, Asset $asset): RedirectResponse
{
    $this->authorize('delete', $asset);
    
    // 步骤1：若资产正被借出 → 自动签回
    if ($asset->assignedTo) {
        event(new CheckoutableCheckedIn(...)); // 触发签入事件
        DB::table('assets')
            ->where('id', $asset->id)
            ->update(['assigned_to' => null, 'assigned_type' => null]); // 清空分配字段
    }
    
    // 步骤2：删除资产图片文件
    if ($asset->image) { Storage::disk('public')->delete('assets/'.basename($asset->image)); }
    
    // 步骤3：软删除（设置 deleted_at 时间戳）
    $asset->delete();
    
    return redirect()->route('hardware.index');
}
```

**前置条件：**
- `Gate::allows('delete', $asset)` 权限检查
- `isDeletable()`（Asset.php L435-L440）：要求 `deleted_at == ''`（未被删除过）

**模型事件：** Asset.php L235-L244 注册了软删除监听
```php
static::softDeleted(function (Asset $asset) {
    $asset->requests()->delete();  // 连带删除该资产的领用请求
});
```

### 4.2 报废资产的恢复（restore）

L942-L966 的 `getRestore()` 方法：

```php
public function getRestore($assetId = null)
{
    // 必须用 withTrashed() 查询，否则查不到软删除记录
    if ($asset = Asset::withTrashed()->find($assetId)) {
        $this->authorize('delete', $asset);
        
        // 防止重复恢复
        if ($asset->deleted_at == '') { return redirect()->back()->withError(...); }
        
        // 恢复（清空 deleted_at），若资产标签冲突会触发 UniqueUndeletedTrait 校验失败
        if ($asset->restore()) { ... }
        
        return redirect()->back()->withError(...);
    }
}
```

### 4.3 已报废资产的查询范围

Asset 模型中通过 `scopeArchived()`（L1751-L1757）、`scopeNotArchived()`（L1565-L1571）等方法配合 `status_id` 过滤资产。关键是 Laravel SoftDeletes 自动注入的全局 scope：
- 常规查询（`Asset::where(...)`）自动排除 `deleted_at IS NOT NULL` 的记录
- 需使用 `Asset::withTrashed()` 才能看到已报废资产
- `Asset::onlyTrashed()` 仅查看已报废资产

**报废 vs 状态标签（Statuslabel）的区别：**
- `deleted_at`：物理层面的"报废/已删除"，由 Laravel SoftDeletes 管理
- `status_id` 配合 `archived` 字段：业务层面的"归档"状态标签
- 两者互不冲突，但查询范围往往同时检查二者（如 `availableForCheckout()` L472 检查了 `deleted_at` + `status->archived` + `status->deployable`）

---

## 五、账面价值（Book Value）展示链路

从计算到前端展示，账面价值的传播路径如下：

### 5.1 计算层

`Asset` → 继承 `Depreciable` → 调用 `getDepreciatedValue()` → 根据全局折旧方法分派。

这是**实时动态计算**，不是数据库存值。每次访问都会根据当前时间重新计算。

### 5.2 数据转换层（Transformer）

#### 资产列表 API：[AssetsTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Transformers/AssetsTransformer.php) L127

```php
'book_value' => Helper::formatCurrencyOutput($asset->getDepreciatedValue()),
```

同时在 L71-L77 会附带返回折旧规则的元信息：
```php
'depreciation' => [
    'id'       => (int) $asset->model->depreciation->id,
    'name'     => e($asset->model->depreciation->name),
    'months'   => (int) $asset->model->depreciation->months,
    'type'     => e($asset->model->depreciation->depreciation_type),
    'minimum'  => ($asset->model->depreciation->depreciation_min) 
                  ? (int) $asset->model->depreciation->depreciation_min : null,
],
```

#### 折旧报告 API：[DepreciationReportTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Transformers/DepreciationReportTransformer.php) L66-L72, L103

```php
if (($asset->model) && ($asset->model->depreciation) && ($asset->model->depreciation->months !== 0)) {
    $depreciated_value      = Helper::formatCurrencyOutput($asset->getDepreciatedValue());
    $monthly_depreciation   = Helper::formatCurrencyOutput($asset->getMonthlyDepreciation());
    $diff                   = Helper::formatCurrencyOutput(($asset->purchase_cost - $asset->getDepreciatedValue()));
}
```

折旧报告额外提供了：
- **monthly_depreciation**：每月折旧额
- **diff**：累计折旧（购入成本 - 当前账面价值）
- **number_of_months**：折旧总月数
- **depreciation**：折旧规则名称

### 5.3 前端表格列定义（Presenter）

#### 资产列表：[AssetPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Presenters/AssetPresenter.php) L181-L186

```php
[
    'field'           => 'book_value',
    'searchable'      => false,
    'sortable'        => false,
    'title'           => trans('admin/hardware/table.book_value'),
    'footerFormatter' => 'sumFormatter',   // 表格底部自动求和
    'class'           => 'text-right',
],
```

注意：账面价值列**不可搜索、不可排序**，因为它是运行时计算属性，不是数据库字段。

#### 折旧报告：[DepreciationReportPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Presenters/DepreciationReportPresenter.php) L142-L146 同样定义了 book_value 列。

### 5.4 报废对账面价值展示的影响

**关键点：账面价值计算不区分资产是否已报废。**

`getDepreciatedValue()` 的代码（Depreciable.php L41-L66）中没有任何 `deleted_at` 判断。这意味着：

1. **报废后的资产仍然可以计算账面价值** — 只要你能通过 `withTrashed()` 或 `onlyTrashed()` 查询到它
2. **报废本身不改变折旧逻辑** — 折旧继续按时间推进，不会因为资产被标记删除而停止
3. **在常规资产列表中看不到报废资产的账面价值** — 因为 SoftDeletes 全局 scope 已将其排除，但在"已删除资产"视图中（使用 `withTrashed`）仍可正常计算和展示
4. **恢复资产后，账面价值自然延续** — 因为是实时计算，恢复后直接用当前时间重新算即可，无需补录

### 5.5 无折旧配置的资产

当资产型号未关联折旧规则时：
- `get_depreciation()` 返回 `null`
- `getDepreciatedValue()` 直接返回 `purchase_cost`（L43-L45）
- 即：无折旧配置的资产，账面价值永远等于购入成本

---

## 六、典型场景数据流

### 场景1：新增资产型号并配置折旧

```
1. 管理员创建折旧规则（/depreciations）
   → DepreciationsController::store()
   → 写入 depreciations 表（name, months, type, min）

2. 管理员创建资产型号，选择上述折旧规则
   → 写入 models 表的 depreciation_id 字段

3. 资产入库时选择该型号
   → 写入 assets 表的 model_id、purchase_cost、purchase_date
   → 资产通过 model→depreciation 隐式关联折旧规则
```

### 场景2：浏览资产列表查看账面价值

```
1. GET /hardware → 页面加载
2. 前端 AJAX 请求 /api/hardware（Datatables）
3. AssetsTransformer::transformAsset()
   → $asset->getDepreciatedValue() 计算
   → Helper::formatCurrencyOutput() 格式化货币
   → book_value 字段输出
4. Bootstrap Table 通过 AssetPresenter 定义的列配置渲染
5. 底部通过 sumFormatter 汇总全部账面价值
```

### 场景3：资产报废 → 恢复

```
1. 点击"删除" → AssetsController::destroy()
   → 若已借出则触发签入事件
   → SoftDeletes 设置 deleted_at
   → 资产从默认列表中消失

2. "已删除资产"页面（/hardware?status=deleted）
   → 使用 withTrashed 查询
   → 账面价值依然按 getDepreciatedValue() 正常计算

3. 点击"恢复" → AssetsController::getRestore()
   → restore() 清空 deleted_at
   → 资产重新出现在常规列表
   → 账面价值无任何"中断续接"问题，因为始终是实时计算
```

---

## 七、关键代码速查表

| 功能 | 文件 | 方法/位置 |
|------|------|-----------|
| 折旧规则模型 | [Depreciation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciation.php) | 全文 |
| 折旧计算核心 | [Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) | `getDepreciatedValue()` L41, `getLinearDepreciatedValue()` L71, `getHalfYearDepreciatedValue()` L108 |
| 残值计算 | [Depreciable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Depreciable.php) | `calculateDepreciation()` L231 |
| 资产获取折旧 | [Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) | `get_depreciation()` L695 |
| 报废（软删） | [AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Assets/AssetsController.php) | `destroy()` L536 |
| 恢复资产 | [AssetsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/Assets/AssetsController.php) | `getRestore()` L942 |
| 软删模型事件 | [Asset.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Models/Asset.php) | `booted()` L235-L244 |
| 账面价值输出 | [AssetsTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Transformers/AssetsTransformer.php) | L127 |
| 折旧报告输出 | [DepreciationReportTransformer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Transformers/DepreciationReportTransformer.php) | `transformAsset()` L33 |
| 表格列配置 | [AssetPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Presenters/AssetPresenter.php) | book_value L181 |
| 折旧方法全局设置 | [SettingsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/27-snipe-it/app/Http/Controllers/SettingsController.php) | L129 |
