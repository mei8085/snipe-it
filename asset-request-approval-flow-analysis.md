# Snipe-IT 资产申请流程代码分析报告

## 文档说明

本文档基于 Snipe-IT 源代码深入分析资产申请从用户提交到审批完成的完整状态流转和权限校验机制，特别补充了**审批通过后申请记录退出待处理视图的流转机制**和**Web/API 双端门禁差异对比**。

---

## 1. 整体流程概览

```
用户提交申请 → [等待审批] → 管理员执行Checkout分配 → [用户验收(可选)] → 完成入库
         ↓                     ↓
   申请记录写入         申请记录状态不自动更新（设计特点）
checkout_requests      仍保留在 checkout_requests 表
```

核心数据表：
- `checkout_requests` - 资产申请表
- `checkout_acceptances` - 资产验收表
- `assets` - 资产主表
- `action_logs` - 操作审计日志表

---

## 2. 状态定义与流转机制

### 2.1 CheckoutRequest（资产申请）状态模型

CheckoutRequest 不使用枚举状态字段，而是通过**时间戳字段的组合**隐式表达状态：

| 字段 | 类型 | 含义 | 状态表达 |
|------|------|------|---------|
| `created_at` | timestamp | 申请创建时间 | 申请提交时间点 |
| `canceled_at` | timestamp NULL | 取消时间 | NULL=待处理/未取消；非NULL=已取消 |
| `fulfilled_at` | timestamp NULL | 完成时间 | **字段存在但代码中从未被设置** |
| `deleted_at` | timestamp NULL | 软删除时间 | NULL=存在；非NULL=已删除 |

数据库迁移定义：
- 初始表结构：[2016_09_02_001448_create_checkout_requests_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/database/migrations/2016_09_02_001448_create_checkout_requests_table.php#L13-L24)
- 状态字段补充：[2018_03_29_053618_add_canceled_at_and_fulfilled_at_in_requests.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/database/migrations/2018_03_29_053618_add_canceled_at_and_fulfilled_at_in_requests.php#L14-L20)

**完整状态流转图：**

```
创建（created_at=NOW, canceled_at=NULL）
    │
    ├─→ 用户主动取消 → canceled_at=NOW → [已取消]
    │
    ├─→ 用户/资产被删除 → 级联删除 → [已删除]
    │
    ├─→ 定时任务清理 → forceDelete/delete → [已清理]
    │
    └─→ 管理员执行Checkout分配 → 【状态字段不变化】
                                canceled_at 仍为 NULL
                                fulfilled_at 仍为 NULL
                                仅通过业务逻辑过滤退出视图
```

### 2.2 关键发现：审批通过后的状态流转机制

**审批通过（管理员执行 Checkout）后，CheckoutRequest 记录不会被自动更新或删除！**

这是 Snipe-IT 一个重要的设计特点（或设计缺陷）：

1. **没有代码在 checkout 时更新 CheckoutRequest**
   - 全代码库搜索结果：checkout 流程中没有任何 `CheckoutRequest::update()`、`CheckoutRequest::delete()` 或 `fulfilled_at = NOW` 的调用
   - 唯一的修改动作是用户取消、用户删除、资产删除、定时任务清理

2. **退出待处理视图的方式**
   
   不同视图通过不同的查询逻辑隐式过滤：

   | 视图 | 查询过滤条件 | checkout后是否显示 | 代码位置 |
   |------|-------------|-------------------|---------|
   | 管理员待审批列表 | `whereNull('canceled_at')` | **仍然显示！**（缺陷） | [AssetsController.php#L1105](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetsController.php#L1105) |
   | 用户申请列表(API) | `where('user_id', auth()->id())` | **仍然显示！** 无canceled过滤 | [ProfileController.php#L50](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/ProfileController.php#L50) |
   | 资产是否被某用户申请 | `whereNull('canceled_at') + where('user_id', X)` | 仍返回true（缺陷） | [Requestable.php#L19-L22](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Traits/Requestable.php#L19-L22) |

3. **唯一会改变 CheckoutRequest 状态的场景**

   | 场景 | 操作 | 代码位置 |
   |------|------|---------|
   | 用户主动取消 | `canceled_at = NOW` | [Requestable.php#L45-L52](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Traits/Requestable.php#L45-L52) |
   | 用户被删除 | 关联申请级联删除 | [User.php#L248-L253](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/User.php#L248-L253) |
   | 资产被删除 | 关联申请级联删除 | [Asset.php#L236-L243](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php#L236-L243) |
   | 定时任务清理 | 清理引用已删除用户/资产的申请 | [CleanOldCheckoutRequests.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Console/Commands/CleanOldCheckoutRequests.php) |

   定时任务清理逻辑：
   ```php
   // 强制删除：关联对象不存在（外键断裂）
   !$request->requestable || !$request->user → forceDelete()
   
   // 软删除：关联对象已软删除
   $request->requestable->trashed() || $request->user->trashed() → delete()
   ```

4. **设计缺陷总结**
   - `fulfilled_at` 字段定义但从未使用，状态机不完整
   - 资产被 checkout 后申请记录仍显示在待审批列表，造成管理混乱
   - 理论上已分配的资产，`isRequestedBy()` 仍返回 true，可能导致重复申请判断错误

### 2.3 CheckoutAcceptance（资产验收）状态模型

验收记录通过两个时间戳字段组合表达三种互斥状态：

| `accepted_at` | `declined_at` | 状态 | 说明 |
|---------------|---------------|------|------|
| NULL | NULL | pending | 待用户验收 |
| NOT NULL | NULL | accepted | 用户已接受 |
| NULL | NOT NULL | declined | 用户已拒绝 |

状态判断方法：
- [CheckoutAcceptance.php#L103-L106](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/CheckoutAcceptance.php#L103-L106)

查询作用域：
- [CheckoutAcceptance.php#L173-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/CheckoutAcceptance.php#L173-L181)

**验收状态流转：**

```
pending (两字段均为NULL)
    ├─→ 用户接受 → accepted_at=NOW → accepted（资产保持分配状态）
    └─→ 用户拒绝 → declined_at=NOW → declined（自动解除分配）
```

拒绝后的自动归还逻辑：
```php
// Asset::declinedCheckout()
$this->assigned_to = null;      // 解除分配
$this->assigned_type = null;    // 清除分配类型
$this->accepted = null;         // 清除验收标记
$this->save();
```
代码位置：[Asset.php#L63-L69](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php#L63-L69)

### 2.4 Asset（资产）可申请性定义

`scopeRequestableAssets` 定义了哪些资产可以被申请：

```php
// 必须同时满足：
1. requestable = 1                       // 资产标记为可申请
2. status.archived = 0                   // 状态未归档
3. (status.deployable = 1                // 状态为可部署
   OR status.pending = 1)                //  或状态为待处理
4. 公司权限过滤（多公司支持）             // Company::scopeCompanyables()
```

代码位置：[Asset.php#L1788-L1803](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php#L1788-L1803)

资产可分配性定义 `availableForCheckout()`：
```php
// 必须同时满足：
1. assigned_to = NULL        // 未被分配
2. deleted_at = NULL         // 未软删除
3. status.archived = 0       // 未归档
4. status.deployable = 1     // 可部署
```

代码位置：[Asset.php#L468-L486](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php#L468-L486)

---

## 3. 三大入口：Web 与 API 门禁权限详细对比

### 3.1 入口一：可申请资产列表

#### Web 端
- **路由**：`GET /requestable-assets`
- **控制器**：[ViewAssetsController::getRequestableIndex](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L146-L162)
- **中间件**：`web` 组（Session、CSRF、Auth）

**权限校验链：**
```
1. Auth 中间件 → 必须是已登录用户
2. 无显式 $this->authorize() 调用
3. 查询级别过滤：
   ├─ Asset::RequestableAssets() → 资产状态+可申请标记过滤
   └─ Company::scopeCompanyables() → 多公司权限自动过滤
```

**公司校验**：通过 `scopeRequestableAssets` 内部调用 `Company::scopeCompanyables()` 自动完成。

#### API 端
- **路由**：`GET /api/v1/account/requestable/hardware`
- **控制器**：[Api\AssetsController::requestable](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/AssetsController.php#L1359-L1436)
- **中间件**：`api` 组（Token 认证：`auth:api`）

**权限校验链：**
```
1. auth:api 中间件 → Token 认证
2. 显式权限校验：$this->authorize('viewRequestable', Asset::class)
   → 需要 assets.view.requestable 权限
3. 查询级别过滤：
   ├─ $assets->requestableAssets() → 同 Web 端
   └─ Company::scopeCompanyables() → 多公司过滤
```

**公司校验**：
1. 策略层 `before()` 钩子：`Company::isCurrentUserHasAccess($item)`（但传参为类名，跳过）
2. 查询层：`scopeRequestableAssets` 内部 `Company::scopeCompanyables()`

#### Web vs API 差异对比

| 对比项 | Web 端 | API 端 |
|--------|--------|--------|
| 认证方式 | Session Cookie | Bearer Token |
| CSRF 保护 | 有 | 无 |
| 显式权限校验 | 无 | 有（`assets.view.requestable`） |
| 权限策略调用 | 不触发 Policy | 触发 `AssetPolicy::viewRequestable()` |
| 查询过滤 | 完全相同 | 完全相同 |
| 响应格式 | HTML 视图 | JSON |
| 代码位置 | [ViewAssetsController.php#L146-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L146-L162) | [Api/AssetsController.php#L1359-L1436](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/AssetsController.php#L1359-L1436) |

**⚠️ 不一致问题**：Web 端缺少显式权限校验，理论上任何登录用户都能访问可申请列表，绕过了 `assets.view.requestable` 权限控制。

---

### 3.2 入口二：提交申请动作（Web 双路径深度对比）

Web 端存在**两条完全独立的提交路径**，使用不同的代码逻辑和权限校验，**绝对不是一套门禁**！

---

#### Web 路径 A：单资产申请入口（store 方法）
- **路由**：`POST /request-asset/{asset}`
- **路由名**：`account.request-asset`
- **控制器方法**：[ViewAssetsController::store](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L228-L243)
- **参数注入**：通过 Laravel 路由模型绑定自动注入 `Asset $asset`
- **支持类型**：仅支持单个 `Asset`（资产实例）

**完整执行流程：**
```php
public function store(Asset $asset): RedirectResponse
{
    try {
        CreateCheckoutRequestAction::run($asset, auth()->user());
        return redirect()->route('requestable-assets')->with('success', ...);
    } catch (AssetNotRequestable $e) {
        return redirect()->back()->with('error', 'Asset is not requestable');
    } catch (AuthorizationException $e) {
        return redirect()->back()->with('error', trans('admin/hardware/message.requests.error'));
    } catch (Exception $e) {
        report($e);
        return redirect()->back()->with('error', trans('general.something_went_wrong'));
    }
}
```

**权限校验链（严格模式）：**
```
1. Auth 中间件 → 必须登录
2. 路由模型绑定 → 资产必须存在（404 if not found）
3. CreateCheckoutRequestAction 内部校验：
   ├─ 3.1 可申请性校验
   │    is_null(Asset::RequestableAssets()->find($asset->id))
   │    → 不满足 → 抛出 AssetNotRequestable 异常
   │    
   └─ 3.2 公司权限校验
        Company::isCurrentUserHasAccess($asset)
        → 不满足 → 抛出 AuthorizationException 异常

4. 创建 CheckoutRequest 记录
5. 记录操作日志
6. 更新 requests_counter
7. 发送通知邮件
```

代码位置：[CreateCheckoutRequestAction.php#L21-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php#L21-L53)

**可申请范围定义（Asset）：**
```php
scopeRequestableAssets($query) {
    return Company::scopeCompanyables(
        $query->where('assets.requestable', '=', 1)
    )->join('status_labels AS status_alias', function ($join) {
        $join->on('status_alias.id', '=', 'assets.status_id')
            ->where('status_alias.archived', '=', 0)
            ->where(function ($s) {
                $s->where('status_alias.deployable', '=', 1)
                  ->orWhere('status_alias.pending', '=', 1);
            });
    });
}
```
代码位置：[Asset.php#L1788-L1803](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php#L1788-L1803)

---

#### Web 路径 B：通用申请入口（getRequestItem 方法）
- **路由**：`POST /account/request/{itemType}/{itemId}/{cancel_by_admin?}/{requestingUser?}`
- **路由名**：`account/request-item`
- **控制器方法**：[ViewAssetsController::getRequestItem](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L164-L221)
- **参数注入**：无模型绑定，手动 `call_user_func([$fullItemType, 'find'], $itemId)`
- **支持类型**：仅支持使用 `Requestable` trait 的模型
  - `asset` → `App\Models\Asset`
  - `asset_model` → `App\Models\AssetModel`（内部转换显示名为 `model`）

**完整执行流程：**
```php
public function getRequestItem(Request $request, $itemType, $itemId = null, 
                                $cancel_by_admin = false, $requestingUser = null): RedirectResponse
{
    // 1. 动态解析模型类
    $fullItemType = 'App\\Models\\'.studly_case($itemType);
    if ($itemType == 'asset_model') { $itemType = 'model'; }
    
    // 2. 手动查找实例（无 404 异常！）
    $item = call_user_func([$fullItemType, 'find'], $itemId);
    
    // 3. 记录日志（注意：此时 $item 可能为 null！）
    $logaction = new Actionlog;
    $logaction->item_id = $item->id;  // ⚠️ 如果 $item 为 null 会报错！
    
    // 4. 检查是否已申请（toggle 逻辑）
    if (($item_request = $item->isRequestedBy($user)) || $cancel_by_admin) {
        // 4.1 取消申请路径
        $item->cancelRequest($requestingUser);  // 无权限校验！
        $logaction->logaction(ActionType::RequestCanceled);
        // 发送取消通知...
        return redirect()->back()->with('success', ...);
    } else {
        // 4.2 提交申请路径
        $item->request();  // 直接调用，无任何校验！
        $logaction->logaction('requested');
        // 发送申请通知...
        return redirect()->route('requestable-assets')->with('success', ...);
    }
}
```

**权限校验链（宽松模式 - 几乎没有校验！）：**
```
1. Auth 中间件 → 必须登录
2. 手动查找 → $item = Model::find($itemId)
   ⚠️ 找不到不会抛 404，后面访问 $item->id 时会报错
   
3. ⚠️ 无显式可申请性校验
   - 不检查 $item->requestable 字段
   - 不检查关联状态是否可部署/归档
   
4. ⚠️ 无显式公司权限校验
   - 不调用 Company::isCurrentUserHasAccess()
   - 依赖 Model 自身是否有全局 CompanyScope
   
5. 检查是否已申请（isRequestedBy）
   → 是 → 执行取消（cancelRequest）
   → 否 → 执行申请（request）
   
6. 直接调用 $item->request() / $item->cancelRequest()
   ⚠️ Requestable trait 中的方法无任何校验！
```

**可申请范围对比（按类型）：**

| 类型 | 可申请性定义 | 公司隔离 | 代码位置 |
|------|-------------|---------|---------|
| Asset | `scopeRequestableAssets` 严格校验<br>requestable=1 + 状态可部署/待处理 + 未归档 | 有（`Company::scopeCompanyables`） | [Asset.php#L1788-L1803](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php#L1788-L1803) |
| AssetModel | `scopeRequestableModels` 宽松校验<br>仅检查 `requestable = 1` | 无（直接 `where('requestable', '1')`） | [AssetModel.php#L314-L317](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/AssetModel.php#L314-L317) |

**⚠️ 关键差异**：通用入口申请 AssetModel 时，`scopeRequestableModels` **没有调用 `Company::scopeCompanyables`**，理论上可以跨公司申请其他公司的资产模型！

---

#### 两条 Web 路径核心差异对比表

| 对比维度 | 路径 A：单资产申请（store） | 路径 B：通用申请（getRequestItem） |
|---------|---------------------------|---------------------------------|
| 路由 | `POST /request-asset/{asset}` | `POST /account/request/{itemType}/{itemId}` |
| 模型查找 | 路由模型绑定（自动 404） | 手动 `find()`（无 404） |
| 支持类型 | 仅 Asset | Asset + AssetModel |
| 业务逻辑类 | `CreateCheckoutRequestAction` | 直接调用 `$item->request()` |
| 可申请性校验 | ✅ 严格（RequestableAssets scope） | ❌ 无（直接写入） |
| 公司权限校验 | ✅ 显式 `isCurrentUserHasAccess` | ❌ 无（依赖模型全局 scope） |
| 异常处理 | 分类捕获三种异常 | 无 try-catch（直接报错） |
| 重复申请检查 | Action 内通过 DB 唯一索引 | 方法内 `isRequestedBy()` toggle |
| 计数更新 | ✅ `increment('requests_counter')` | ❌ 无 |
| 代码位置 | [ViewAssetsController.php#L228-L243](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L228-L243) | [ViewAssetsController.php#L164-L221](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L164-L221) |

---

#### API 端
- **路由**：`POST /api/v1/account/request/{asset}`
- **控制器**：[Api\CheckoutRequest::store](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/CheckoutRequest.php#L17-L32)
- **中间件**：`api` 组

**权限校验链**：与 Web 路径 A **完全相同**（复用 `CreateCheckoutRequestAction`）

```
1. auth:api 中间件 → Token 认证
2. 调用 CreateCheckoutRequestAction::run()
   → 可申请性校验 + 公司权限校验
```

#### Web 路径 A vs API 一致性对比

| 对比项 | Web 路径 A | API 端 |
|--------|-----------|--------|
| 认证方式 | Session + CSRF Token | Bearer Token |
| 核心业务逻辑 | 完全相同（复用 CreateCheckoutRequestAction） | 完全相同（复用 CreateCheckoutRequestAction） |
| 可申请性校验 | 有 | 有 |
| 公司权限校验 | 有 | 有 |
| 计数更新 | 有 | 有 |
| 响应格式 | RedirectResponse | JsonResponse |

---

### 3.3 入口三：待审批列表

这是**差异最大**的入口，面向完全不同的用户角色！

#### Web 端（管理员视角）
- **路由**：`GET /hardware/requested`
- **控制器**：[AssetsController::getRequestedIndex](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetsController.php#L1102-L1114)
- **目标用户**：管理员 / 具有资产查看权限的用户
- **显示内容**：**所有用户**提交的待处理申请

**权限校验链：**
```
1. Auth 中间件 → 登录
2. 显式权限校验：$this->authorize('index', Asset::class)
   → 注意：传递的是类名 Asset::class，不是实例！
3. 查询过滤：
   CheckoutRequest::with('user', 'requestedItem')
       ->whereNull('canceled_at')    // 只看未取消的
       ->orderBy('created_at', 'desc')
   → ⚠️ 无任何 company_id 过滤条件！
```

---

#### 深度分析：为什么 `authorize('index', Asset::class)` 跳过了公司校验？

这是理解管理端待审批列表无公司隔离的**关键**。让我们从 `SnipePermissionsPolicy::before()` 钩子的源码和注释讲起。

**关键源码**：[SnipePermissionsPolicy.php#L38-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Policies/SnipePermissionsPolicy.php#L38-L73)

```php
public function before(User $user, $ability, $item)
{
    // 1. 超级管理员 admin 直接放行
    if ($user->hasAccess('admin')) {
        return true;
    }

    // 2. ⚠️ 关键判断：传类名 vs 传实例
    // 官方注释原文：
    // "If we got here by $this->authorize('something', $actualModel) then we can continue on,
    //  but if we got here via $this->authorize('something', Model::class) then calling
    //  Company::isCurrentUserHasAccess($item) gets weird."
    if (! $item instanceof Model) {
        return;  // ← 返回 null！表示"不做判断，让后续策略方法处理"
    }

    // 3. 实例级公司权限校验（只有传实例时才会走到这里）
    if (! Company::isCurrentUserHasAccess($item)) {
        return false;
    }
}
```

**Laravel Policy `before()` 的返回值语义**：

| 返回值 | 含义 | 后续行为 |
|--------|------|---------|
| `true` | 允许 | 直接返回 true，不再执行后续策略方法 |
| `false` | 拒绝 | 直接返回 false，不再执行后续策略方法 |
| `null`（无 return 或 return;） | **不做判断** | **继续执行后续的 index()/view()/update() 等方法** |

**完整执行链路分析**：

```
$this->authorize('index', Asset::class)
        │
        ▼
AssetPolicy::before() [继承自 SnipePermissionsPolicy]
        │
        ├─ 检查 user->hasAccess('admin') → false（非超级管理员）
        │
        ├─ 检查 $item instanceof Model
        │     Asset::class 是字符串类名，不是 Model 实例
        │     → 条件成立！
        │     → return null（不做判断）
        │
        ▼
AssetPolicy::index() [继承自 SnipePermissionsPolicy]
        │
        ▼
return $user->hasAccess('assets.view');
        │
        ▼
   仅检查权限字符串，不检查公司！
```

**架构设计意图（从源码注释推断）**：

Snipe-IT 的公司权限采用**两层架构**，每层职责不同：

| 层级 | 机制 | 适用场景 | 说明 |
|------|------|---------|------|
| **L1 模型层** | `CompanyableTrait` + `CompanyableScope` 全局 Scope | 列表查询（index/view） | 注释原文："That scoping happens on the model level (except for the Users model) via the Companyable trait." |
| **L2 策略层** | `SnipePermissionsPolicy::before()` 中的 `Company::isCurrentUserHasAccess()` | 单个实例操作（update/delete/checkout） | 对具体 Model 实例做实例级公司校验 |

设计预期：
- 对于 `Asset::all()`、`Asset::find()` 等列表查询，靠 **L1 全局 Scope** 自动加 `where company_id = X` 过滤
- 对于 `authorize('update', $assetInstance)` 等单实例操作，靠 **L2 Policy before** 做实例级校验

**问题出在哪里？—— CheckoutRequest 缺少 L1 全局 Scope**：

CheckoutRequest 模型 **没有使用 `CompanyableTrait`**，因此：
1. CheckoutRequest 表查询不会自动加 `where company_id = ?`
2. `CheckoutRequest::with('user', 'requestedItem')` 中的主查询无公司过滤
3. 仅关联的 `requestedItem` 如果是 Asset 类型，会被其自身的全局 Scope 过滤（可能导致关联加载为 null）

**最终效果**：
- 非超级管理员但有 `assets.view` 权限的用户，可以看到**所有公司**用户提交的 CheckoutRequest 申请记录
- 但查看关联的具体资产时，跨公司的资产可能因全局 Scope 过滤而显示为 null（数据不一致）

---

#### API 端（普通用户视角）
- **路由**：`GET /api/v1/account/requests`
- **控制器**：[Api\ProfileController::requestedAssets](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/ProfileController.php#L48-L89)
- **目标用户**：普通登录用户
- **显示内容**：**当前用户自己**提交的所有申请

**权限校验链：**
```
1. auth:api 中间件 → Token 认证
2. 无显式 $this->authorize() 调用
3. 查询过滤：
   CheckoutRequest::where('user_id', '=', auth()->id())->get()
   → ⚠️ 注意：没有 whereNull('canceled_at') 过滤！
4. 存在性检查：if ($checkoutRequest && $checkoutRequest->itemRequested())
```

#### Web vs API 差异对比

| 对比项 | Web 端（管理员） | API 端（普通用户） |
|--------|------------------|-------------------|
| 使用角色 | 管理员 | 普通用户 |
| 数据范围 | 所有用户的申请 | 仅当前用户的申请 |
| 所需权限 | `assets.view` | 仅需登录 |
| 显式 Policy 校验 | 有（`AssetPolicy::index`，传类名 `Asset::class`） | 无 |
| 公司权限校验 | ❌ **无**（类级权限 `authorize('index', Asset::class)` 只检查 `assets.view`，不触发实例级公司校验；且 CheckoutRequest 无全局 Scope） | 无（`where user_id = auth()->id()` 天然隔离） |
| canceled_at 过滤 | 有（`whereNull('canceled_at')`） | **无**（已取消的也显示） |
| 查询过滤条件 | 无用户过滤，无公司过滤 | `where('user_id', auth()->id())` |
| 代码位置 | [AssetsController.php#L1102-L1114](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetsController.php#L1102-L1114) | [Api/ProfileController.php#L48-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/ProfileController.php#L48-L89) |

**⚠️ 不一致问题**：API 端缺少 `canceled_at` 过滤，用户已取消的申请仍然会显示在"我的申请"列表中。

---

## 4. 各阶段详细流程分析

### 4.1 阶段一：用户提交申请

**核心执行逻辑**：[CreateCheckoutRequestAction::run()](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php#L21-L53)

```
Step 1: 资产可申请性校验
  └─ 检查资产是否在 RequestableAssets() 范围内
     失败 → 抛出 AssetNotRequestable 异常

Step 2: 公司权限校验（多公司支持 FMCS）
  └─ Company::isCurrentUserHasAccess($asset)
     失败 → 抛出 AuthorizationException 异常

Step 3: 记录操作日志（Actionlog）
  └─ action_type = 'requested'
     item_id = asset.id
     target_id = 当前登录用户ID
     location_id = 用户所在位置

Step 4: 创建 CheckoutRequest 记录
  └─ $asset->request() → 保存 morphMany 关系
     user_id = auth()->id()
     qty = 1 (默认)

Step 5: 更新资产请求计数
  └─ $asset->increment('requests_counter', 1)

Step 6: 发送管理员通知邮件
  └─ RequestAssetNotification → 发送至 settings.alert_email
```

### 4.2 阶段二：管理员审批与分配

**设计特点**：Snipe-IT **没有独立的"批准"动作**。管理员查看待处理列表后，直接执行 Checkout（资产分配）表示审批通过。

**分配入口**：
- Web: `POST /hardware/{asset}/checkout` → [AssetCheckoutController::store](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetCheckoutController.php#L69-L179)
- API: `POST /api/v1/assets/{asset_id}/checkout` → [Api\AssetsController::checkout](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/AssetsController.php#L1001-L1064)

**权限校验**：
```php
$this->authorize('checkout', $asset);
// → 需要 assets.checkout 权限
```
代码位置：[CheckoutablePermissionsPolicy.php#L15-L18](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Policies/CheckoutablePermissionsPolicy.php#L15-L18)

**核心分配逻辑**：[Asset::checkOut()](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php#L521-L573)

```
Step 1: 基础校验
  ├─ 目标不为空
  └─ 不能将资产分配给自己

Step 2: 设置资产属性
  ├─ expected_checkin = 预期归还日期
  ├─ last_checkout = 分配时间
  ├─ assignedTo() = 目标（用户/位置/资产）
  └─ location_id = 继承目标位置

Step 3: 保存资产记录
  └─ $this->save()

Step 4: 触发 CheckoutableCheckedOut 事件
  └─ event(new CheckoutableCheckedOut(...))

Step 5: 更新计数
  └─ increment('checkout_counter', 1)
```

**事件监听副作用**：[CheckoutableListener::onCheckedOut](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Listeners/CheckoutableListener.php#L73-L165)

自动创建验收记录（如需要）：
```php
private function getCheckoutAcceptance($event)
{
    // 只有分配给用户 且 资产/分类需要验收时才创建
    if (目标类型不是 User) → return null
    if (! $event->checkoutable->requireAcceptance()) → return null

    CheckoutAcceptance::create([
        checkoutable_type → Asset::class
        checkoutable_id   → asset.id
        assigned_to_id    → user.id
        qty               → 1
        alert_on_response_id → auth()->id()  // 如果分类设置了响应提醒
    ])
}
```

### 4.3 阶段三：用户验收（可选流程）

**入口**：
- 验收列表：`GET /account/accept` → [AcceptanceController::index](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Account/AcceptanceController.php#L37-L42)
- 提交验收：`POST /account/accept/{acceptance}` → [AcceptanceController::store](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Account/AcceptanceController.php#L93-L322)

**权限校验链**：
```
1. 用户类型校验：必须是 User 模型实例（非其他可分配类型）
2. 验收存在性：CheckoutAcceptance 记录存在
3. 状态校验：必须是 pending（accepted_at 和 declined_at 均为 NULL）
4. 归属校验：必须分配给当前用户（或管理员代签流程）
5. 公司权限：Company::isCurrentUserHasAccess($acceptance->checkoutable)
6. 签名校验：如果设置要求签名，必须提交签名图片
```

**接受（accepted）分支**：
```
1. 生成验收 PDF（含EULA、签名、资产信息）
2. CheckoutAcceptance::accept()
   ├─ accepted_at = NOW
   ├─ signature_filename = 签名文件
   ├─ stored_eula = EULA文本
   ├─ stored_eula_file = PDF文件名
   └─ 触发 checkoutable->acceptedCheckout()
3. 发送用户确认邮件（可选）
4. 发送管理员通知
5. 触发 CheckoutAccepted 事件
```

**拒绝（declined）分支**：
```
1. CheckoutAcceptance::decline()
   ├─ declined_at = NOW
   ├─ signature_filename = 签名文件
   └─ 触发 checkoutable->declinedCheckout() → 关键！解除分配
2. 发送拒绝通知邮件
3. 触发 CheckoutDeclined 事件
```

拒绝后的状态回滚：[Asset::declinedCheckout](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php#L63-L69)
```php
$this->assigned_to = null;     // 解除分配
$this->assigned_type = null;   // 清除分配类型
$this->accepted = null;        // 清除验收标记
$this->save();
```

### 4.4 阶段四：用户取消申请

**核心逻辑**：[CancelCheckoutRequestAction::run()](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php#L15-L46)

```
Step 1: 公司权限校验
  └─ Company::isCurrentUserHasAccess($asset)

Step 2: 标记取消
  └─ $asset->cancelRequest() → 更新 canceled_at = NOW

Step 3: 更新计数
  └─ decrement('requests_counter', 1)

Step 4: 记录日志
  └─ action_type = 'request canceled'

Step 5: 发送取消通知
  └─ RequestAssetCancelation → 管理员邮箱
```

**⚠️ 权限缺陷**：`cancelRequest()` 只校验公司权限，未校验是否为申请人本人。理论上任何同公司用户可以取消他人的申请。

---

## 5. 权限体系详解

### 5.1 权限策略继承链

```
SnipePermissionsPolicy (抽象基类)
  ├─ before() → 超级管理员豁免 + 公司权限检查
  ├─ index()  → *.view
  ├─ view()   → *.view
  ├─ create() → *.create
  ├─ edit()   → *.edit
  └─ delete() → *.delete

CheckoutablePermissionsPolicy (抽象中间类)
  ├─ checkout() → *.checkout
  ├─ checkin()  → *.checkin
  └─ manage()   → *.checkin || *.edit || *.checkout

AssetPolicy (具体策略类)
  └─ viewRequestable() → assets.view.requestable
```

### 5.2 公司权限检查（SnipePermissionsPolicy::before）

所有 Policy 继承的全局钩子：[SnipePermissionsPolicy.php#L38-L71](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Policies/SnipePermissionsPolicy.php#L38-L71)

```php
public function before(User $user, $ability, $item)
{
    // 超级管理员全局豁免
    if ($user->hasAccess('admin')) {
        return true;
    }

    // 如果传参是类名而非实例，跳过公司检查
    if (! $item instanceof Model) {
        return;
    }

    // 多公司权限校验
    if (! Company::isCurrentUserHasAccess($item)) {
        return false;
    }
}
```

### 5.3 权限矩阵汇总表

| 操作 | 所需权限 | 公司校验 | 代码位置 |
|------|---------|---------|---------|
| 查看可申请资产（API） | `assets.view.requestable` | 有 | [AssetPolicy.php#L15-L18](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Policies/AssetPolicy.php#L15-L18) |
| 查看可申请资产（Web） | 仅需登录（缺陷） | 查询层隐式 | [ViewAssetsController.php#L146-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L146-L162) |
| 提交申请（Web路径A：单资产） | 登录 + 资产可申请 + 同公司 | ✅ 有 | [CreateCheckoutRequestAction.php#L23-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php#L23-L28) |
| 提交申请（Web路径B：通用-Asset） | 登录 | ❌ 无（依赖模型全局scope） | [ViewAssetsController.php#L164-L221](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L164-L221) |
| 提交申请（Web路径B：通用-AssetModel） | 登录 | ❌ 无（无公司过滤） | [AssetModel.php#L314-L317](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/AssetModel.php#L314-L317) |
| 查看待审批列表（管理端） | `assets.view` | ❌ 无（类级权限，架构缺陷） | [AssetsController.php#L1102-L1114](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetsController.php#L1102-L1114) |
| 查看我的申请（用户端API） | 仅需登录 | ✅ 数据天然隔离 | [Api/ProfileController.php#L48-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/ProfileController.php#L48-L89) |
| 执行分配（审批通过） | `assets.checkout` | ✅ 有 | [AssetCheckoutController.php#L79](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetCheckoutController.php#L79) |
| 取消申请 | 登录 + 同公司（缺陷：不校验本人） | ✅ 有 | [CancelCheckoutRequestAction.php#L17-L19](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php#L17-L19) |
| 用户验收 | 分配给本人 + 同公司 | ✅ 有 | [AcceptanceController.php#L72-L80](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Account/AcceptanceController.php#L72-L80) |

---

## 6. 操作日志记录点

| 节点 | action_type 值 | 代码位置 |
|------|--------------|---------|
| 用户提交申请 | `requested` | [CreateCheckoutRequestAction.php#L42](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php#L42) |
| 用户取消申请 | `request canceled` | [CancelCheckoutRequestAction.php#L37](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php#L37) |
| 管理员分配资产 | `checkout` | Loggable trait 自动记录 |
| 用户验收接受 | `accepted` | acceptedCheckout 中记录 |
| 用户验收拒绝 | `declined` + `checkin from declined` | declinedCheckout 中记录 |

---

## 7. 设计特点总结

### 7.1 设计优点

1. **隐式状态机**：通过时间戳字段表达状态，简化数据库设计
2. **多态关联**：CheckoutRequest 通过 `requestable_type/id` 支持资产、配件、耗材、模型等多种类型申请
3. **事件驱动**：Checkout 完成后通过事件解耦验收创建、邮件通知、Webhook 等副作用
4. **公司级隔离**：所有操作叠加 `Company::isCurrentUserHasAccess()` 检查，支持多租户
5. **验收可选**：根据分类/模型配置灵活决定是否需要用户验收步骤
6. **Action 模式**：申请创建/取消逻辑封装为 Action 类，Web/API 复用

### 7.2 已确认的设计缺陷

| # | 缺陷描述 | 影响程度 | 影响范围 | 绕过的防护层级 | 相关代码 |
|---|---------|---------|---------|--------------|---------|
| D1 | checkout 后 CheckoutRequest 状态不更新 | 高 | 管理端待审批列表仍显示已分配的申请 | N/A（状态流转逻辑缺失） | 全代码库无 checkout 后更新逻辑 |
| D2 | `fulfilled_at` 字段定义但未使用 | 高 | 状态机不完整，无法区分"已完成"和"待处理" | N/A（状态流转逻辑缺失） | [迁移文件](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/database/migrations/2018_03_29_053618_add_canceled_at_and_fulfilled_at_in_requests.php) |
| D3 | **通用入口申请 AssetModel 三层防护全缺失** | 高 | 可跨公司申请其他公司的资产模型；可申请 `requestable=0` 的模型 | L1（无 Policy）+ L2（无业务校验）+ **L3（无全局 Scope）** | [AssetModel.php#L28-L30](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/AssetModel.php#L28-L30) |
| D4 | **管理端待审批列表无实例级公司校验** | 高 | 多公司环境下有 `assets.view` 权限的管理员可看到其他公司的申请 | L2（类级传参触发 `before()` 返回 null）+ **L3（CheckoutRequest 无全局 Scope）** | [AssetsController.php#L1102-L1114](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetsController.php#L1102-L1114) |
| D5 | 通用入口申请 Asset 绕过 L1/L2 业务校验 | 中 | 可申请 `requestable=0`、已归档、不可部署的资产；不更新计数 | L1（无 Policy）+ L2（无业务校验） | [ViewAssetsController.php#L164-L221](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L164-L221) |
| D6 | Web 端可申请列表无权限校验 | 中 | 绕过 `assets.view.requestable` 权限控制 | L1（无 Policy 调用） | [ViewAssetsController.php#L146-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L146-L162) |
| D7 | 取消申请不校验是否为本人 | 中 | 同公司任意用户可取消他人申请 | L2（缺少申请人身份校验） | [CancelCheckoutRequestAction.php#L17-L19](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php#L17-L19) |
| D8 | 通用入口不更新 requests_counter | 中 | 申请计数不准确，影响统计 | L2（缺少计数更新逻辑） | [Requestable.php#L33-L38](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Traits/Requestable.php#L33-L38) |
| D9 | 审批与分配强耦合 | 中 | 无法"批准申请但暂不分配资产" | N/A（架构设计缺陷） | 无独立 approve 动作 |
| D10 | API 端我的申请无 canceled_at 过滤 | 低 | 已取消的申请仍显示在用户"我的申请"列表 | L2（缺少状态过滤） | [ProfileController.php#L50](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/ProfileController.php#L50) |
| D11 | 通用入口无 404 异常处理 | 低 | 请求不存在的 itemId 会直接报 TypeError | L2（缺少存在性校验） | [ViewAssetsController.php#L172](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php#L172) |

**高风险缺陷解读（按风险严重度排序）**：

| 缺陷 | 为什么是高危 | 根因 |
|------|------------|------|
| **D3：AssetModel 三层全缺** | 公司隔离完全失效，任何登录用户可跨公司申请其他公司的资产模型 | AssetModel 模型 `use` 语句中缺失 `CompanyableTrait` |
| **D4：管理端列表无公司过滤** | 多公司环境数据泄露，有 `assets.view` 权限即可看全公司申请 | ① CheckoutRequest 表无 `company_id` 字段 ② 模型无 `CompanyableTrait` ③ `authorize('index', Asset::class)` 传类名触发 `before()` 返回 null |
| **D1/D2：状态流转缺失** | 已分配的申请仍然出现在待审批列表，管理员重复操作 | 架构上 Checkout 与 CheckoutRequest 完全解耦，无事件监听更新申请状态 |

### 7.3 并发与一致性问题

1. **数据库唯一索引**：`(user_id, requestable_id, requestable_type)` 防止同一用户重复申请同一资产
2. **但无资产级锁**：多人可同时申请同一资产，由管理员分配时竞争
3. **申请计数竞态**：`increment/decrement` 使用数据库原子操作，避免竞态

---

## 8. 完整时序图

```
用户                          系统                         管理员
 |                            |                             |
 |-- 查看可申请资产列表 ------>|                             |
 |   (Web: 仅登录, API: 需权限)|                             |
 |<------- 返回列表 ----------|                             |
 |                            |                             |
 |-- 提交资产申请 ----------->| 1. 校验资产可申请性         |
 |                            | 2. 校验公司权限             |
 |                            | 3. 创建 CheckoutRequest     |
 |                            |    (状态: canceled_at=NULL) |
 |                            | 4. 记录 requested 日志      |
 |                            | 5. +1 requests_counter      |
 |<--- 申请成功 --------------| 6. 发送申请通知邮件 ------->|
 |                            |                             |
 |                            |                             |-- GET /hardware/requested
 |                            |                             |   (需 assets.view 权限)
 |                            |<--- 返回所有待处理申请 ----|
 |                            |                             |
 |                            |                             |-- 执行 Checkout 分配
 |                            | 1. 校验 checkout 权限 <------|
 |                            | 2. 校验资产可分配           |
 |                            | 3. 设置 assigned_to = 用户   |
 |                            | 4. +1 checkout_counter      |
 |                            | 5. 触发 CheckedOut 事件     |
 |                            |    → 创建验收记录(如需要)   |
 |                            |    ⚠️ CheckoutRequest 不变  |
 |                            |       canceled_at 仍为 NULL  |
 |<-- 分配通知邮件(含验收链接)-|                             |-- 分配完成
 |                            |                             |
 |-- 点击验收链接 ----------->|                             |
 |-- 接受/拒绝 -------------->| 1. 校验待验收状态           |
 |                            | 2. 校验归属&公司权限        |
 |                            | 3. 保存签名(如需要)         |
 |                            | 4. 设置 accepted/declined   |
 |                            | 5. 如拒绝: 归还资产         |
 |                            |    → assigned_to = null     |
 |<--- 验收完成 --------------| 6. 发送响应通知邮件 ------->|
 |                            |                             |
```

---

## 9. 核心文件索引

| 类型 | 文件路径 | 核心功能 |
|------|---------|---------|
| 模型 | [CheckoutRequest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/CheckoutRequest.php) | 申请模型（多态关联） |
| 模型 | [CheckoutAcceptance.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/CheckoutAcceptance.php) | 验收模型 |
| 模型 | [Asset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Asset.php) | 资产模型（checkOut/declinedCheckout） |
| Trait | [Requestable.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Models/Traits/Requestable.php) | 可申请特性 |
| Action | [CreateCheckoutRequestAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php) | 创建申请动作 |
| Action | [CancelCheckoutRequestAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php) | 取消申请动作 |
| 控制器 | [ViewAssetsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/ViewAssetsController.php) | 用户端Web申请控制器 |
| 控制器 | [AssetCheckoutController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetCheckoutController.php) | 资产分配控制器 |
| 控制器 | [AcceptanceController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Account/AcceptanceController.php) | 验收控制器 |
| 控制器 | [AssetsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Assets/AssetsController.php) | 资产控制器（待审批列表） |
| 控制器 | [Api/AssetsController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/AssetsController.php) | API资产控制器（可申请列表） |
| 控制器 | [Api/CheckoutRequest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/CheckoutRequest.php) | API申请接口 |
| 控制器 | [Api/ProfileController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Http/Controllers/Api/ProfileController.php) | API个人中心（我的申请） |
| 监听器 | [CheckoutableListener.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Listeners/CheckoutableListener.php) | Checkout事件监听 |
| 策略 | [AssetPolicy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Policies/AssetPolicy.php) | 资产权限策略 |
| 策略 | [CheckoutablePermissionsPolicy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Policies/CheckoutablePermissionsPolicy.php) | 分配权限策略基类 |
| 策略 | [SnipePermissionsPolicy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Policies/SnipePermissionsPolicy.php) | 通用权限策略基类（公司校验） |
| 命令 | [CleanOldCheckoutRequests.php](file:///d:/fz/0601-1/solo-dogfeeding/code/41-snipe-it/app/Console/Commands/CleanOldCheckoutRequests.php) | 清理无效申请定时任务 |
