# 资产申请-审批-通知 状态流分析

## 一、核心数据模型

### CheckoutRequest 模型
**文件**: `app/Models/CheckoutRequest.php:9-56`

- 多态关联模型 (`morphTo('requestable')`)，支持资产、配件等多种类型的申请
- 关键字段：
  - `user_id` - 申请人ID
  - `requestable_id` / `requestable_type` - 申请目标（多态关联）
  - `qty` - 申请数量
  - `canceled_at` - 取消时间（NULL表示待处理）
  - `fulfilled_at` - 完成时间（**代码中未实际使用**）
  - `deleted_at` - 软删除时间

### Requestable Trait
**文件**: `app/Models/Traits/Requestable.php:12-53`

为资产模型添加申请相关方法：
- `requests()` - 获取该资产的所有申请
- `isRequestedBy(User $user)` - 检查某用户是否已申请该资产
- `request($qty = 1)` - 创建申请
- `cancelRequest($user_id = null)` - 取消申请（设置 `canceled_at`）
- `deleteRequest()` - 删除申请记录

---

## 二、申请入口

### 1. 可申请资产列表页
**路由**: `GET /requestable-assets`  
**控制器**: `ViewAssetsController@getRequestableIndex` (`app/Http/Controllers/ViewAssetsController.php:146-162`)

- 展示可申请的资产和资产型号列表
- 查询条件 (`scopeRequestableAssets`):
  - `assets.requestable = 1` - 资产标记为可申请
  - 状态：`deployable = 1 AND archived = 0` 或 `pending = 1`
  - 公司权限过滤

### 2. 申请提交（Web方式）
**路由**: `POST /request-asset/{asset}`  
**控制器**: `ViewAssetsController@store` (`app/Http/Controllers/ViewAssetsController.php:228-243`)

调用 `CreateCheckoutRequestAction::run($asset, auth()->user())`

### 3. 申请提交（API方式）
**路由**: `POST /api/v1/account/request/{asset}`  
**控制器**: `Api\CheckoutRequest@store` (`app/Http/Controllers/Api/CheckoutRequest.php:17-32`)

同样调用 `CreateCheckoutRequestAction::run()`

### 4. 通用申请/取消入口
**路由**: `POST /request/{itemType}/{itemId}/{cancel_by_admin?}/{requestingUser?}`  
**控制器**: `ViewAssetsController@getRequestItem` (`app/Http/Controllers/ViewAssetsController.php:164-221`)

- 支持资产和资产型号两种类型的申请
- 如果用户已申请，则切换为取消操作
- 支持管理员代用户取消（`cancel_by_admin = true`）

### CreateCheckoutRequestAction 执行流程
**文件**: `app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php:15-54`

```
1. 权限检查
   ├─ 检查资产是否在可申请列表中 (scopeRequestableAssets)
   └─ 检查公司权限
2. 记录操作日志 (Actionlog: 'requested')
3. 创建申请记录 ($asset->request())
4. 递增 requests_counter
5. 发送邮件通知 (RequestAssetNotification)
```

---

## 三、审批动作

> **注意**: 本系统**没有显式的"批准/拒绝"按钮**，审批通过管理员执行 checkout 操作来完成。

### 审批列表页
**路由**: `GET /hardware/requested`  
**控制器**: `AssetsController@getRequestedIndex` (`app/Http\Controllers\Assets\AssetsController.php:1102-1114`)

- 查询所有 `canceled_at IS NULL` 的申请记录
- 按创建时间倒序排列
- 每条申请显示：资产信息、申请人、申请时间、操作按钮

### 审批操作（隐式）
**文件**: `resources/views/hardware/requested.blade.php:107-115`

审批页面提供两个操作：

1. **取消申请** (Cancel)
   - 表单提交到 `POST /request/{itemType}/{itemId}/true/{requestingUser}`
   - 调用 `CancelCheckoutRequestAction`
   - 记录日志 'request canceled'
   - 递减 requests_counter
   - 发送取消通知邮件

2. **借出资产** (Checkout) - **即审批通过**
   - 链接跳转到 `GET /hardware/{asset}/checkout`
   - 由管理员执行 checkout 操作完成审批
   - 如果资产已被占用 (`assigned_to != ''`)，按钮变为 "Checkin"

### 取消申请动作
**文件**: `app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php:13-47`

```
1. 权限检查（公司权限）
2. 取消申请 ($asset->cancelRequest() → 设置 canceled_at)
3. 递减 requests_counter
4. 记录操作日志 (Actionlog: 'request canceled')
5. 发送取消通知邮件 (RequestAssetCancelation)
```

---

## 四、邮件通知触发机制

### 通知类型与触发时机

| 通知类 | 触发时机 | 接收者 | 邮件主题 |
|--------|---------|--------|---------|
| `RequestAssetNotification` | 用户提交申请时 | 系统设置中的 alert_email | 👀 Item Requested |
| `RequestAssetCancelation` | 申请被取消时 | 系统设置中的 alert_email | ⚠️ Request Canceled |
| `CheckoutAssetNotification` | 资产 checkout 成功时 | 借用人 / alert_email | Asset Checkout Notification |
| `CheckinAssetNotification` | 资产 checkin 成功时 | 借用人 / alert_email | Asset Checkin Notification |

### 事件驱动机制

资产借出/归还通过事件系统触发通知：

1. **资产借出**: `Asset::checkOut()` → `event(new CheckoutableCheckedOut(...))`
   - **文件**: `app/Models/Asset.php:565`

2. **事件订阅**: `CheckoutableListener` 订阅 `CheckoutableCheckedOut` 和 `CheckoutableCheckedIn` 事件
   - **文件**: `app/Providers/EventServiceProvider.php:34-38`
   - **文件**: `app/Listeners/CheckoutableListener.php:56-67`

3. **通知发送**: `CheckoutableListener@onCheckedOut`
   - **文件**: `app/Listeners/CheckoutableListener.php:73-165`
   - 发送邮件给借用人（如果启用了 checkout_email、需要验收或有EULA）
   - 发送邮件给 alert_email（如果设置了 admin_cc_email）
   - 发送 Webhook 通知（Slack / Microsoft Teams）

### 申请通知的特殊逻辑
**文件**: `app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php:47-50`

申请和取消的通知**不通过事件系统**，而是直接调用：
```php
$settings->notify((new RequestAssetNotification($data))->locale($settings->locale));
```

发送条件：
- `$settings->alert_email != ''`
- `$settings->alerts_enabled == '1'`
- `! config('app.lock_passwords')`

---

## 五、资产被占用时的回退路径

### 1. 借出前检查（前置保护）
**文件**: `app/Models/Asset.php:468-486`

`availableForCheckout()` 方法在 checkout 前检查：
```php
public function availableForCheckout()
{
    // 资产未分配给任何人 且 未被删除
    if ((! $this->assigned_to) && (! $this->deleted_at)) {
        // 状态未归档 且 可部署
        if (($this->status) && ($this->status->archived == '0')
            && ($this->status->deployable == '1')
        ) {
            return true;
        }
    }
    return false;
}
```

如果检查不通过，checkout 操作会被拒绝，错误信息：
> `trans('admin/hardware/message.checkout.not_available')`

### 2. 无自动回退机制

**重要发现**: 当资产被 checkout 占用时，**系统不会自动取消相关的 checkout_request 记录**。

证据：
- `fulfilled_at` 字段在数据库迁移中定义，但代码中**从未被使用**
- `CheckoutableListener@onCheckedOut` 中没有处理 checkout_request 的逻辑
- 没有任何地方在 checkout 成功后调用 `cancelRequest()` 或 `deleteRequest()`

### 3. 手动回退路径

管理员需要在申请列表页**手动点击 "Cancel" 按钮**来取消已借出资产的申请。

### 4. 过期/无效申请清理
**命令**: `snipeit:clean-old-checkout-requests`  
**文件**: `app/Console/Commands/CleanOldCheckoutRequests.php:8-77`

定时清理无效的申请记录：

```
1. 强制删除 (forceDelete)
   ├─ 申请目标 (requestable) 不存在
   └─ 申请人 (user) 不存在

2. 软删除 (delete)
   ├─ 申请目标已被软删除 (trashed)
   └─ 申请人已被软删除 (trashed)
```

---

## 六、完整状态流转图

```
                     ┌─────────────────┐
                     │  可申请资产列表  │
                     │ /requestable-assets │
                     └────────┬────────┘
                              │ 点击 "Request"
                              ▼
                     ┌─────────────────┐
                     │  提交申请        │
                     │  CreateCheckoutRequestAction
                     └────────┬────────┘
                              │
                              ├─ 记录 actionlog: 'requested'
                              ├─ 创建 checkout_request 记录
                              ├─ 递增 requests_counter
                              └─ 发送 RequestAssetNotification
                              │
                              ▼
                     ┌─────────────────┐
                     │  待审批状态      │
                     │  canceled_at = NULL │
                     └────────┬────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │ 点击 "Cancel"     │ 点击 "Checkout"    │
          ▼                   ▼                   ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  取消申请        │  │  资产借出        │  │  资产已被占用    │
│  CancelCheckout- │  │  AssetCheckout-  │  │  availableFor-   │
│  RequestAction   │  │  Controller@store│  │  Checkout()=false│
└────────┬─────────┘  └────────┬─────────┘  └──────────────────┘
         │                     │
         ├─ 设置 canceled_at   ├─ 检查 availableForCheckout()
         ├─ 递减 requests_counter ├─ 检查权限
         ├─ 记录 'request canceled' ├─ 设置 assigned_to
         └─ 发送 RequestAssetCancelation ├─ 触发 CheckoutableCheckedOut 事件
                               │
                               ├─ 发送 CheckoutAssetNotification
                               └─ 状态：资产已分配
                                    checkout_request 仍为待处理状态
                                    （需要手动取消）
```

---

## 七、关键代码位置汇总

| 功能 | 文件 | 行号 |
|------|------|------|
| 申请模型 | `app/Models/CheckoutRequest.php` | 9-56 |
| 申请 Trait | `app/Models/Traits/Requestable.php` | 12-53 |
| 创建申请 Action | `app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php` | 15-54 |
| 取消申请 Action | `app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php` | 13-47 |
| 可借出检查 | `app/Models/Asset.php` | 468-486 |
| 借出方法 | `app/Models/Asset.php` | 521-573 |
| 借出控制器 | `app/Http/Controllers/Assets/AssetCheckoutController.php` | 69-179 |
| 审批列表 | `app/Http/Controllers/Assets/AssetsController.php` | 1102-1114 |
| 申请/取消控制器 | `app/Http/Controllers/ViewAssetsController.php` | 164-256 |
| 借出事件监听器 | `app/Listeners/CheckoutableListener.php` | 73-165 |
| 申请通知 | `app/Notifications/RequestAssetNotification.php` | 13-127 |
| 取消通知 | `app/Notifications/RequestAssetCancelation.php` | 14-133 |
| 清理命令 | `app/Console/Commands/CleanOldCheckoutRequests.php` | 8-77 |
| 可申请范围 | `app/Models/Asset.php` | 1788-1803 |

---

## 八、潜在问题与建议

### 问题 1：缺少自动回退机制
当资产被借出后，相关的 checkout_request 记录仍然保持待处理状态，需要管理员手动取消。

**建议**: 在 `CheckoutableListener@onCheckedOut` 中添加逻辑，当资产被借出时自动取消该资产的所有待处理申请：
```php
// 在 onCheckedOut 方法中添加
if ($event->checkoutable instanceof Asset) {
    $event->checkoutable->requests()
        ->whereNull('canceled_at')
        ->update(['canceled_at' => now()]);
}
```

### 问题 2：fulfilled_at 字段未使用
数据库中有 `fulfilled_at` 字段但代码中从未使用，可能是遗留功能或未完成功能。

### 问题 3：审批状态不明确
系统没有明确的"已批准"、"已拒绝"状态，只有"已取消"和"待处理"。借出操作与审批操作耦合在一起，不利于审计追踪。
