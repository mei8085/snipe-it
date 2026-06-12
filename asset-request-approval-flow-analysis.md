# Snipe-IT 资产申请流程代码分析报告

## 1. 整体流程概览

资产申请从用户提交到完成的完整流程包含以下阶段：

```
用户提交申请 → [等待审批] → 管理员审批(分配/拒绝) → 用户验收(可选) → 完成入库
```

核心涉及的数据库表：
- `checkout_requests` - 资产申请表
- `checkout_acceptances` - 资产验收表
- `assets` - 资产表
- `action_logs` - 操作日志表

---

## 2. 状态定义与流转

### 2.1 CheckoutRequest（资产申请）状态

CheckoutRequest 没有使用枚举状态字段，而是通过**时间戳**隐式表达状态：

| 字段 | 含义 | 状态说明 |
|------|------|---------|
| `created_at` | 申请创建时间 | 申请提交 |
| `canceled_at` | 取消时间 | NULL=未取消；非NULL=已取消 |
| `fulfilled_at` | 完成时间 | 字段存在但代码中**未使用** |
| `deleted_at` | 软删除时间 | NULL=存在；非NULL=已删除 |

代码参考：[2018_03_29_053618_add_canceled_at_and_fulfilled_at_in_requests.php](database/migrations/2018_03_29_053618_add_canceled_at_and_fulfilled_at_in_requests.php#L14-L20)

**状态流转：**

```
pending (canceled_at=NULL)
    ├─→ 用户取消 → canceled (canceled_at=NOW)
    └─→ 管理员执行checkout → （无状态变化，直接通过checkout完成申请）
```

注意：当管理员执行 checkout 操作后，CheckoutRequest 记录的状态**不会自动更新** `fulfilled_at`，申请记录保留在表中但通过业务逻辑过滤（`whereNull('canceled_at')`）不再显示为待处理。

### 2.2 CheckoutAcceptance（资产验收）状态

CheckoutAcceptance 通过两个时间戳字段组合表达三种状态：

| accepted_at | declined_at | 状态 | 说明 |
|-------------|-------------|------|------|
| NULL | NULL | pending | 待验收 |
| NOT NULL | NULL | accepted | 已接受 |
| NULL | NOT NULL | declined | 已拒绝 |

代码参考：[CheckoutAcceptance.php#L103-L106](app/Models/CheckoutAcceptance.php#L103-L106)

查询作用域定义：[CheckoutAcceptance.php#L173-L181](app/Models/CheckoutAcceptance.php#L173-L181)

### 2.3 Asset（资产）状态

资产是否可申请通过 `scopeRequestableAssets` 定义：

```php
// 必须满足：
1. requestable = 1           // 资产标记为可申请
2. status.archived = 0       // 状态未归档
3. status.deployable = 1     // 状态为可部署  OR  status.pending = 1  // 状态为待处理
```

代码参考：[Asset.php#L1788-L1803](app/Models/Asset.php#L1788-L1803)

资产是否可分配通过 `availableForCheckout()` 定义：

```php
// 必须满足：
1. assigned_to = NULL        // 未分配给任何人
2. deleted_at = NULL         // 未软删除
3. status.archived = 0       // 未归档
4. status.deployable = 1     // 可部署
```

代码参考：[Asset.php#L468-L486](app/Models/Asset.php#L468-L486)

---

## 3. 阶段一：用户提交申请

### 3.1 入口路由

| 路由 | 方法 | 说明 |
|------|------|------|
| `POST /request-asset/{asset}` | ViewAssetsController::store | Web端单资产申请 |
| `POST /account/request-item/{itemType}/{itemId}` | ViewAssetsController::getRequestItem | Web端通用申请（资产/模型/配件等） |
| `POST /api/v1/assets/request/{asset}` | Api\CheckoutRequest::store | API端申请 |

代码参考：[web.php#L406-L413](routes/web.php#L406-L413), [api.php#L65-L66](routes/api.php#L65-L66)

### 3.2 核心执行逻辑：CreateCheckoutRequestAction

代码参考：[CreateCheckoutRequestAction.php](app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php)

**执行步骤：**

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

### 3.3 权限校验详情

| 校验点 | 代码位置 | 校验内容 |
|--------|---------|---------|
| 资产可申请 | [CreateCheckoutRequestAction.php#L23-L25](app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php#L23-L25) | 资产是否为 RequestableAssets |
| 公司权限 | [CreateCheckoutRequestAction.php#L26-L28](app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php#L26-L28) | 当前用户是否有该资产所属公司的访问权限 |

---

## 4. 阶段二：管理员审批（查看待处理列表）

### 4.1 待审批列表

代码参考：[AssetsController::getRequestedIndex](app/Http/Controllers/Assets/AssetsController.php#L1102-L1114)

```php
// 查询条件：
CheckoutRequest::with('user', 'requestedItem')
    ->whereNull('canceled_at')   // 未取消的
    ->orderBy('created_at', 'desc')
```

**权限校验：**
```php
$this->authorize('index', Asset::class);
// 需要 assets.view 权限
```

参考：[AssetsController.php#L1104](app/Http/Controllers/Assets/AssetsController.php#L1104)

注意：Snipe-IT **没有显式的"批准/拒绝"按钮**。审批通过的方式是管理员直接对资产执行 **Checkout（分配）** 操作。待处理列表仅用于展示哪些资产被申请了。

---

## 5. 阶段三：管理员执行分配（Checkout）

### 5.1 入口路由

| 路由 | 方法 | 说明 |
|------|------|------|
| `GET /hardware/{asset}/checkout` | AssetCheckoutController::create | 显示分配表单 |
| `POST /hardware/{asset}/checkout` | AssetCheckoutController::store | 执行分配 |

代码参考：[hardware.php 路由文件](routes/web/hardware.php)

### 5.2 权限校验

```php
$this->authorize('checkout', $asset);
// 需要 assets.checkout 权限
```

参考：[AssetCheckoutController.php#L37](app/Http/Controllers/Assets/AssetCheckoutController.php#L37), [CheckoutablePermissionsPolicy.php#L15-L18](app/Policies/CheckoutablePermissionsPolicy.php#L15-L18)

### 5.3 前置校验

| 校验 | 代码位置 | 说明 |
|------|---------|------|
| 资产存在 | [AssetCheckoutController.php#L74-L75](app/Http/Controllers/Assets/AssetCheckoutController.php#L74-L75) | findOrFail |
| 可分配 | [AssetCheckoutController.php#L76-L78](app/Http/Controllers/Assets/AssetCheckoutController.php#L76-L78) | availableForCheckout() |
| 模型关联 | [AssetCheckoutController.php#L81-L83](app/Http/Controllers/Assets/AssetCheckoutController.php#L81-L83) | 资产必须关联模型 |
| 多公司匹配 | [AssetCheckoutController.php#L125-L129](app/Http/Controllers/Assets/AssetCheckoutController.php#L125-L129) | 目标用户/资产公司一致 |

### 5.4 核心执行逻辑：Asset::checkOut()

代码参考：[Asset.php#L521-L573](app/Models/Asset.php#L521-L573)

**执行步骤：**

```
Step 1: 基础校验
  ├─ 目标不为空
  └─ 不能将资产分配给自己

Step 2: 设置属性
  ├─ expected_checkin = 预期归还日期
  ├─ last_checkout = 分配时间
  ├─ name = 资产名称(可选)
  ├─ assignedTo = 目标（用户/位置/资产）
  └─ location_id = 继承目标位置或指定位置

Step 3: 保存资产记录
  └─ $this->save()

Step 4: 触发 CheckoutableCheckedOut 事件
  └─ event(new CheckoutableCheckedOut(...))

Step 5: 更新计数
  └─ increment('checkout_counter', 1)
```

### 5.5 事件监听：CheckoutableListener::onCheckedOut

代码参考：[CheckoutableListener.php#L73-L165](app/Listeners/CheckoutableListener.php#L73-L165)

**关键副作用：创建 CheckoutAcceptance（验收记录）**

```php
private function getCheckoutAcceptance($event)
{
    // 只有分配给用户 且 资产/分类需要验收时才创建
    if (目标类型不是 User) → return null
    if (! $event->checkoutable->requireAcceptance()) → return null

    // 创建验收记录
    CheckoutAcceptance::create([
        checkoutable_type → Asset::class
        checkoutable_id   → asset.id
        assigned_to_id    → user.id
        qty               → 1
        alert_on_response_id → auth()->id()  // 如果分类设置了响应提醒
    ])
}
```

参考：[CheckoutableListener.php#L278-L308](app/Listeners/CheckoutableListener.php#L278-L308)

---

## 6. 阶段四：用户验收（可选流程）

### 6.1 入口路由

| 路由 | 方法 | 说明 |
|------|------|------|
| `GET /account/accept` | AcceptanceController::index | 验收列表 |
| `GET /account/accept/{acceptance}` | AcceptanceController::create | 验收表单 |
| `POST /account/accept/{acceptance}` | AcceptanceController::store | 提交验收 |

### 6.2 权限校验

代码参考：[AcceptanceController.php#L49-L86](app/Http/Controllers/Account/AcceptanceController.php#L49-L86)

| 校验点 | 说明 |
|--------|------|
| 用户类型 | 必须是 User 模型实例（非其他可分配类型） |
| 验收存在 | CheckoutAcceptance 记录存在 |
| 验收状态 | 必须是 pending（未接受未拒绝） |
| 归属校验 | 必须分配给当前用户（或管理员代签流程） |
| 公司权限 | `Company::isCurrentUserHasAccess()` |
| 签名（可选） | 若设置要求签名，必须提交签名图片 |

### 6.3 验收提交逻辑

代码参考：[AcceptanceController.php#L238-L280](app/Http/Controllers/Account/AcceptanceController.php#L238-L280)

**接受（accepted）分支：**
```
1. 生成验收 PDF（含EULA、签名、资产信息）
2. 调用 CheckoutAcceptance::accept()
   ├─ accepted_at = NOW
   ├─ signature_filename = 签名文件
   ├─ stored_eula = EULA文本
   ├─ stored_eula_file = PDF文件名
   └─ 触发 checkoutable->acceptedCheckout()
3. 发送用户确认邮件（可选）
4. 发送管理员通知
5. 触发 CheckoutAccepted 事件
```

参考：[CheckoutAcceptance.php#L125-L138](app/Models/CheckoutAcceptance.php#L125-L138)

**拒绝（declined）分支：**
```
1. 调用 CheckoutAcceptance::decline()
   ├─ declined_at = NOW
   ├─ signature_filename = 签名文件
   └─ 触发 checkoutable->declinedCheckout() → 关键！解除分配
2. 发送拒绝通知邮件
3. 触发 CheckoutDeclined 事件
```

参考：[CheckoutAcceptance.php#L145-L156](app/Models/CheckoutAcceptance.php#L145-L156)

### 6.4 拒绝后的自动归还（declinedCheckout）

代码参考：[Asset.php#L63-L69](app/Models/Asset.php#L63-L69)

```php
public function declinedCheckout(User $declinedBy, $signature)
{
    $this->assigned_to = null;     // 解除分配
    $this->assigned_type = null;   // 清除分配类型
    $this->accepted = null;        // 清除验收标记
    $this->save();
}
```

这是**状态回滚**的关键：当用户拒绝验收时，资产自动变为"未分配"状态，可重新分配。

---

## 7. 阶段五：取消申请（用户主动取消）

### 7.1 入口路由

| 路由 | 方法 |
|------|------|
| `POST /request-asset/{asset}/cancel` | ViewAssetsController::destroy |
| `POST /api/v1/assets/request/{asset}/cancel` | Api\CheckoutRequest::destroy |

### 7.2 核心逻辑：CancelCheckoutRequestAction

代码参考：[CancelCheckoutRequestAction.php](app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php)

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

---

## 8. 权限体系总览

### 8.1 权限继承链

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

参考：[SnipePermissionsPolicy.php](app/Policies/SnipePermissionsPolicy.php), [AssetPolicy.php](app/Policies/AssetPolicy.php)

### 8.2 各阶段所需权限矩阵

| 操作 | 所需权限 | 代码位置 |
|------|---------|---------|
| 查看可申请资产 | `assets.view.requestable` | [AssetPolicy.php#L15-L18](app/Policies/AssetPolicy.php#L15-L18) |
| 提交申请 | 登录用户 + 公司权限 + 资产可申请 | [CreateCheckoutRequestAction.php#L23-L28](app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php#L23-L28) |
| 查看待审批列表 | `assets.view` | [AssetsController.php#L1104](app/Http/Controllers/Assets/AssetsController.php#L1104) |
| 执行分配（审批通过） | `assets.checkout` | [AssetCheckoutController.php#L79](app/Http/Controllers/Assets/AssetCheckoutController.php#L79) |
| 取消申请 | 登录用户 + 公司权限 | [CancelCheckoutRequestAction.php#L17-L19](app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php#L17-L19) |
| 用户验收 | 分配给本人 + 公司权限 | [AcceptanceController.php#L72-L80](app/Http/Controllers/Account/AcceptanceController.php#L72-L80) |

---

## 9. 操作日志（Actionlog）记录点

整个流程在关键节点都会写入 `action_logs` 表：

| 节点 | action_type 值 | 触发代码 |
|------|--------------|---------|
| 用户提交申请 | `requested` | [CreateCheckoutRequestAction.php#L42](app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php#L42) |
| 用户取消申请 | `request canceled` | [CancelCheckoutRequestAction.php#L37](app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php#L37) |
| 管理员分配资产 | `checkout` | (通过 Loggable trait 自动记录) |
| 用户验收接受 | `accepted` | (acceptedCheckout 中记录) |
| 用户验收拒绝 | `declined` + `checkin from declined` | (declinedCheckout 中记录) |

---

## 10. 关键设计特点与潜在问题

### 10.1 设计特点

1. **隐式状态机**：通过时间戳而非枚举字段表达状态，简化设计但降低可读性
2. **多态关联（Polymorphic）**：CheckoutRequest 通过 `requestable_type/id` 支持资产、配件、耗材等多种类型
3. **事件驱动架构**：Checkout 完成后通过事件解耦验收创建、邮件通知、Webhook 等副作用
4. **公司级权限隔离**：所有操作通过 `Company::isCurrentUserHasAccess()` 进行多租户检查
5. **验收可选**：根据分类/模型配置决定是否需要用户验收步骤

### 10.2 潜在问题点

1. **CheckoutRequest 状态未闭环**：`fulfilled_at` 字段定义但从未被设置，申请被满足后记录仍保持 `canceled_at=NULL` 状态，仅通过逻辑过滤
2. **审批与分配耦合**：没有独立的"批准"动作，管理员必须执行实际分配才能表示同意申请，无法"批准但暂不分配"
3. **取消申请的用户身份校验缺失**：`cancelRequest()` 只校验公司权限，未校验是否为申请人本人（理论上任何同公司用户可取消他人申请）
4. **并发申请处理**：数据库唯一索引 `(user_id, requestable_id, requestable_type)` 防止同一用户重复申请，但未防止多人同时申请同一资产

---

## 11. 完整流程时序图

```
用户                          系统                         管理员
 |                            |                             |
 |-- 查看可申请资产列表 ------>|                             |
 |<------- 返回列表 ----------|                             |
 |                            |                             |
 |-- 提交资产申请 ----------->| 1. 校验资产可申请性         |
 |                            | 2. 校验公司权限             |
 |                            | 3. 创建 CheckoutRequest     |
 |                            |    (状态: 待处理)           |
 |                            | 4. 记录 requested 日志      |
 |                            | 5. +1 requests_counter      |
 |<--- 申请成功 --------------| 6. 发送申请通知邮件 ------->|
 |                            |                             |-- 查看待审批列表
 |                            |                             |-- 执行资产分配(Checkout)
 |                            | 1. 校验 checkout 权限 <------|
 |                            | 2. 校验资产可分配           |
 |                            | 3. 设置 assigned_to 用户    |
 |                            | 4. +1 checkout_counter      |
 |                            | 5. 触发 CheckedOut 事件     |
 |                            |    → 创建验收记录(如需要)   |
 |<-- 分配通知邮件(验收链接) --|                             |
 |                            |                             |-- 分配完成
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

## 核心文件索引

| 类型 | 文件 | 说明 |
|------|------|------|
| 模型 | [CheckoutRequest.php](app/Models/CheckoutRequest.php) | 申请模型 |
| 模型 | [CheckoutAcceptance.php](app/Models/CheckoutAcceptance.php) | 验收模型 |
| 模型 | [Asset.php](app/Models/Asset.php) | 资产模型（含checkOut方法） |
| Trait | [Requestable.php](app/Models/Traits/Requestable.php) | 可申请特性 |
| Action | [CreateCheckoutRequestAction.php](app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php) | 创建申请动作 |
| Action | [CancelCheckoutRequestAction.php](app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php) | 取消申请动作 |
| 控制器 | [ViewAssetsController.php](app/Http/Controllers/ViewAssetsController.php) | 用户端申请控制器 |
| 控制器 | [AssetCheckoutController.php](app/Http/Controllers/Assets/AssetCheckoutController.php) | 分配控制器 |
| 控制器 | [AcceptanceController.php](app/Http/Controllers/Account/AcceptanceController.php) | 验收控制器 |
| 控制器 | [AssetsController.php](app/Http/Controllers/Assets/AssetsController.php) | 资产控制器(待审批列表) |
| 监听器 | [CheckoutableListener.php](app/Listeners/CheckoutableListener.php) | checkout事件监听（创建验收记录） |
| 策略 | [AssetPolicy.php](app/Policies/AssetPolicy.php) | 资产权限策略 |
| 策略 | [CheckoutablePermissionsPolicy.php](app/Policies/CheckoutablePermissionsPolicy.php) | 分配权限策略基类 |
| 策略 | [SnipePermissionsPolicy.php](app/Policies/SnipePermissionsPolicy.php) | 通用权限策略基类 |
| API控制器 | [Api/CheckoutRequest.php](app/Http/Controllers/Api/CheckoutRequest.php) | API申请接口 |

        
            
toolName: todo_write
            
status: success
          
Todos updated: 6 items