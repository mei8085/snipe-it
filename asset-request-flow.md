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

审批页面根据资产状态动态显示操作按钮：

```blade
@if ($request->requestable->assigned_to=='')
    <!-- 资产未被占用：显示 Checkout 按钮 -->
    <a href="/hardware/{asset}/checkout">Checkout</a>
@else
    <!-- 资产已被占用：显示 Checkin 按钮 -->
    <a href="/hardware/{asset}/checkin">Checkin</a>
@endif
```

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

### Checkout 审批流程
**文件**: `app/Http/Controllers/Assets/AssetCheckoutController.php:69-179`

```
1. 检查资产是否存在
2. 前置检查：availableForCheckout()
   ├─ 资产未分配 (assigned_to 为空)
   ├─ 资产未删除
   ├─ 状态可部署 (deployable=1) 且未归档 (archived=0)
3. 权限检查 (authorize checkout)
4. 处理表单数据
5. 执行 $asset->checkOut()
   ├─ 设置 assigned_to, assigned_type
   ├─ 触发 CheckoutableCheckedOut 事件
   └─ 递增 checkout_counter
6. 触发 CheckoutableListener 发送通知
```

---

## 四、邮件通知触发机制

### 通知类型与触发时机

| 通知类 | 触发时机 | 接收者 | 邮件主题 |
|--------|---------|--------|---------|
| `RequestAssetNotification` | 用户提交申请时 | alert_email | 👀 Item Requested |
| `RequestAssetCancelation` | 申请被取消时 | alert_email | ⚠️ Request Canceled |
| `CheckoutAssetNotification` | 资产 checkout 成功时 | 借用人 / admin_cc_email | Asset Checkout Notification |
| `CheckinAssetNotification` | 资产 checkin 成功时 | 借用人 / admin_cc_email | Asset Checkin Notification |

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
   - 发送邮件给 admin_cc_email（如果设置了 admin_cc_email）
   - 发送 Webhook 通知（Slack / Microsoft Teams）

### 申请通知的特殊逻辑
**文件**: `app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php:47-50`

申请和取消的通知**不通过事件系统**，而是直接调用：
```php
$settings->notify((new RequestAssetNotification($data))->locale($settings->locale));
```

---

## 五、通知差异对照

### 5.1 三个申请入口的通知差异

| 对比项 | `POST /request-asset/{asset}` | `POST /request/{itemType}/{itemId}` | `POST /api/v1/account/request/{asset}` |
|--------|------------------------------|-----------------------------------|--------------------------------------|
| **调用链** | `ViewAssetsController@store` → `CreateCheckoutRequestAction` | `ViewAssetsController@getRequestItem` (内部逻辑) | `Api\CheckoutRequest@store` → `CreateCheckoutRequestAction` |
| **支持类型** | 仅 Asset | Asset / AssetModel | 仅 Asset |
| **申请数量** | 固定为 1 | 支持 `request-quantity` 参数 | 固定为 1 |
| **通知触发条件** | 无条件发送（try-catch包裹） | 需满足 `alert_email != '' && alerts_enabled == '1' && !lock_passwords` | 无条件发送（try-catch包裹） |
| **通知类** | `RequestAssetNotification` | `RequestAssetNotification` / `RequestAssetCancelation` | `RequestAssetNotification` |
| **通知设置locale** | ✅ `->locale($settings->locale)` | ✅ `->locale($settings->locale)` | ✅ `->locale($settings->locale)` |
| **日志记录时机** | 通知前记录 'requested' | 通知后记录 'requested' | 通知前记录 'requested' |
| **requests_counter** | ✅ 递增 | ❌ 未处理 | ✅ 递增 |
| **权限检查** | ✅ `scopeRequestableAssets` + 公司权限 | ❌ 无前置权限检查 | ✅ `scopeRequestableAssets` + 公司权限 |
| **返回类型** | RedirectResponse | RedirectResponse | JsonResponse |

**关键差异说明**：

1. **`/request-asset/{asset}` 和 API 入口**：
   - 使用 `CreateCheckoutRequestAction`，有完整的权限检查
   - 递增 `requests_counter`
   - 通知用 try-catch 包裹，失败不影响主流程
   - 代码位置：`app/Http/Controllers/ViewAssetsController.php:231` 和 `app/Http/Controllers/Api/CheckoutRequest.php:20`

2. **`/request/{itemType}/{itemId}` 入口**：
   - 内部实现，不使用 Action 类
   - **没有前置权限检查**，直接调用 `$item->request()`
   - **不处理 `requests_counter`**（存在数据不一致风险）
   - 支持资产型号（AssetModel）的申请
   - 支持取消操作（如果用户已申请）
   - 通知有额外的开关条件检查
   - 代码位置：`app/Http/Controllers/ViewAssetsController.php:202-220`

### 5.2 收件人决定机制：alert_email vs reply_to vs admin_cc_email

#### 邮件配置分层
**文件**: `config/mail.php:237-256`

```php
'from' => [
    'address' => env('MAIL_FROM_ADDR', null),    // 发件人地址
    'name' => env('MAIL_FROM_NAME', null),        // 发件人名称
],
'reply_to' => [
    'address' => env('MAIL_REPLYTO_ADDR', null),  // 回复地址
    'name' => env('MAIL_REPLYTO_NAME', null),      // 回复名称
],
```

#### 三个邮箱配置的职责区分

| 配置项 | 用途 | 代码位置 | 通知类型 |
|--------|------|---------|---------|
| `MAIL_FROM_ADDR` | 邮件发件人（显示在 "发件人"） | Laravel 框架自动处理 | 所有邮件 |
| `MAIL_REPLYTO_ADDR` | 邮件回复地址（用户点击 "回复" 时使用） | `Setting::routeNotificationForMail()` → `config('mail.reply_to.address')` | 申请/取消通知 |
| `settings.alert_email` | 申请/取消事件的**收件人** | `ViewAssetsController@getRequestItem` 判断条件 | 申请/取消通知 |
| `settings.admin_cc_email` | checkout/checkin 事件的**抄送收件人** | `CheckoutableListener::getFormattedAlertAddresses()` | 借出/归还通知 |

#### Setting 的通知路由
**文件**: `app/Models/Setting.php:249-254`

```php
public function routeNotificationForMail(): ?string
{
    // 返回 reply_to 地址，而非 alert_email
    return config('mail.reply_to.address');
}
```

> **⚠️ 重要发现**: `Setting::routeNotificationForMail()` 返回的是 `mail.reply_to.address`，这意味着：
> - 当调用 `$settings->notify()` 时，邮件会发送到 `reply_to` 配置的地址
> - 但代码中检查的是 `$settings->alert_email != ''` 作为发送条件
> - 这可能导致：配置了 `alert_email` 但 `reply_to` 为空时，通知发送失败

#### CheckoutAcceptance 的通知路由（对比）
**文件**: `app/Models/CheckoutAcceptance.php:31-39`

```php
public function routeNotificationForMail()
{
    // 正确地使用 alert_email 作为收件人
    $recipients_string = explode(',', Setting::getSettings()->alert_email);
    $recipients = array_map('trim', $recipients_string);
    return array_filter($recipients);
}
```

> **对比**: `CheckoutAcceptance` 正确地将 `alert_email` 作为收件人，而 `Setting` 模型则返回 `reply_to` 地址。这是一个不一致的设计。

### 5.3 审批阶段的通知路径

#### 路径 A：取消申请（审批拒绝）
**触发点**: 审批页点击 "Cancel" 按钮 → `POST /request/{itemType}/{itemId}/true/{requestingUser}`

```
ViewAssetsController@getRequestItem
    │
    ├─ 检查：$item_request 存在 或 $cancel_by_admin = true
    │
    ├─ $item->cancelRequest($requestingUser)
    │   └─ 更新 checkout_requests.canceled_at = NOW()
    │
    ├─ 记录 Actionlog: 'request canceled'
    │
    └─ 通知条件检查
        ├─ alert_email != ''
        ├─ alerts_enabled == '1'
        └─ ! config('app.lock_passwords')
            │
            └─ $settings->notify(new RequestAssetCancelation($data))
                └─ 收件人：config('mail.reply_to.address')
```

**代码位置**: `app/Http/Controllers/ViewAssetsController.php:202-211`

#### 路径 B：资产借出（审批通过）
**触发点**: 审批页点击 "Checkout" 按钮 → `GET /hardware/{asset}/checkout` → `POST /hardware/{assetId}/checkout`

```
AssetCheckoutController@store
    │
    ├─ 检查 availableForCheckout()
    │   ├─ assigned_to 为空？
    │   ├─ 资产未删除？
    │   └─ 状态可部署？
    │
    ├─ $asset->checkOut($target, $admin, ...)
    │   ├─ 设置 assigned_to, assigned_type
    │   ├─ 保存资产
    │   └─ event(new CheckoutableCheckedOut(...))
    │
    └─ CheckoutableListener@onCheckedOut
        │
        ├─ 是否发送给用户？
        │   ├─ 非批量 checkout
        │   ├─ 资产 requireAcceptance() 或有 EULA 或 checkin_email 启用
        │   └─ 收件人：借用人 (User)
        │
        ├─ 是否发送给管理员？
        │   ├─ 非批量 checkout
        │   ├─ 有 acceptance 或 admin_cc_always 启用
        │   ├─ admin_cc_email 已设置
        │   └─ 收件人：admin_cc_email (抄送)
        │
        └─ 通知类：CheckoutAssetNotification
```

**代码位置**: `app/Listeners/CheckoutableListener.php:73-165`

### 5.4 通知发送条件汇总表

| 通知类型 | 触发条件 | 收件人来源 | 检查的设置项 |
|---------|---------|-----------|-------------|
| **申请通知** | 用户提交申请 | `config('mail.reply_to.address')` | `alert_email != '' && alerts_enabled == '1'` |
| **取消通知** | 申请被取消 | `config('mail.reply_to.address')` | `alert_email != '' && alerts_enabled == '1'` |
| **借出通知（用户）** | 资产 checkout 成功 | 借用人的 email | `requireAcceptance() 有 EULA checkin_email 启用` |
| **借出通知（管理员）** | 资产 checkout 成功 | `admin_cc_email` | `admin_cc_email 已设置 (有 acceptance 或 admin_cc_always)` |
| **归还通知（用户）** | 资产 checkin 成功 | 借用人的 email | `checkin_email 启用` |
| **归还通知（管理员）** | 资产 checkin 成功 | `admin_cc_email` | `admin_cc_email 已设置` |

---

## 六、资产被占用时的回退路径

### 6.1 审批页的分支逻辑
**文件**: `resources/views/hardware/requested.blade.php:109-113`

```
审批列表页面渲染每条申请时：
    │
    ├─ 判断 $request->requestable->assigned_to == ''
    │   │
    │   ├─ 为空 → 资产未被占用
    │   │   └─ 显示 "Checkout" 按钮 → /hardware/{asset}/checkout
    │   │
    │   └─ 非空 → 资产已被占用
    │       └─ 显示 "Checkin" 按钮 → /hardware/{asset}/checkin
    │
    └─ 无论资产状态如何，始终显示 "Cancel" 按钮
        └─ 提交到 /request/{itemType}/{itemId}/true/{requestingUser}
```

### 6.2 完整回退分支

```
管理员访问 /hardware/requested（审批列表）
    │
    ├─ 查看申请列表，每条申请显示资产状态
    │
    ├─ 情况 1：资产未被占用 (assigned_to == '')
    │   │
    │   ├─ 分支 1a：点击 "Checkout"（审批通过）
    │   │   └─ 跳转到 /hardware/{asset}/checkout
    │   │       └─ 执行 checkout 流程 → 发送 CheckoutAssetNotification
    │   │
    │   └─ 分支 1b：点击 "Cancel"（审批拒绝）
    │       └─ 提交到 /request/{itemType}/{itemId}/true/{requestingUser}
    │           └─ 执行 CancelCheckoutRequestAction → 发送 RequestAssetCancelation
    │
    └─ 情况 2：资产已被占用 (assigned_to != '')
        │
        ├─ 分支 2a：点击 "Checkin"（先归还再借出）
        │   ├─ 跳转到 /hardware/{asset}/checkin
        │   ├─ 执行 checkin 流程
        │   │   ├─ 触发 CheckoutableCheckedIn 事件
        │   │   └─ 发送 CheckinAssetNotification
        │   └─ 归还后资产变为可用状态
        │       └─ 管理员需再次进入审批列表，点击 "Checkout"
        │
        └─ 分支 2b：点击 "Cancel"（直接取消申请）
            └─ 提交到 /request/{itemType}/{itemId}/true/{requestingUser}
                └─ 执行 CancelCheckoutRequestAction → 发送 RequestAssetCancelation
```

### 6.3 借出前的前置保护
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

**代码位置**: `app/Http/Controllers/Assets/AssetCheckoutController.php:76-78`

### 6.4 无自动回退机制

**重要发现**: 当资产被 checkout 占用时，**系统不会自动取消相关的 checkout_request 记录**。

证据：
- `fulfilled_at` 字段在数据库迁移中定义，但代码中**从未被使用**
- `CheckoutableListener@onCheckedOut` 中没有处理 checkout_request 的逻辑
- 没有任何地方在 checkout 成功后调用 `cancelRequest()` 或 `deleteRequest()`

### 6.5 过期/无效申请清理
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

## 七、完整状态流转图

```
                     ┌─────────────────┐
                     │  可申请资产列表  │
                     │ /requestable-assets │
                     └────────┬────────┘
                              │ 点击 "Request"
                              ▼
        ┌───────────────────────────────────────────┐
        │  选择申请入口                             │
        │  ┌─────────────────────────────────────┐ │
        │  │ /request-asset/{asset} (Web)        │ │
        │  │ /api/v1/account/request/{asset}(API)│ │→ 使用 Action 类
        │  │ /request/{itemType}/{itemId} (通用) │ │→ 内部实现
        │  └─────────────────────────────────────┘ │
        └───────────────────┬───────────────────────┘
                              │
                              ├─ 记录 actionlog: 'requested'
                              ├─ 创建 checkout_request 记录
                              ├─ 递增 requests_counter（通用入口除外）
                              └─ 发送 RequestAssetNotification → reply_to
                              │
                              ▼
                     ┌─────────────────┐
                     │  待审批状态      │
                     │  canceled_at = NULL │
                     └────────┬────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │ 审批页：Cancel    │ 审批页：Checkout  │ 资产已被占用
          ▼                   ▼                   ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  取消申请        │  │  资产借出        │  │  分支选择        │
│  CancelCheckout- │  │  AssetCheckout-  │  │  2a: Checkin     │
│  RequestAction   │  │  Controller@store│  │  2b: Cancel      │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
         ├─ 设置 canceled_at   ├─ 前置检查          ├─ 2a: 先归还资产
         ├─ 递减 requests_counter ├─ availableForCheckout() ├─ 2b: 直接取消
         ├─ 记录 'request canceled' ├─ 设置 assigned_to └───────────┬───────────┘
         └─ 发送 RequestAssetCancelation ├─ 触发 CheckoutableCheckedOut 事件
                               │                     │
                               ├─ 发送 CheckoutAssetNotification
                               │   ├─ 用户：借用人
                               │   └─ 管理员：admin_cc_email
                               │
                               └─ 状态：资产已分配
                                    checkout_request 仍为待处理状态
                                    （需要手动取消）
```

---

## 八、关键代码位置汇总

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
| 通用申请/取消 | `app/Http/Controllers/ViewAssetsController.php` | 164-221 |
| Web申请入口 | `app/Http/Controllers/ViewAssetsController.php` | 228-243 |
| API申请入口 | `app/Http/Controllers/Api/CheckoutRequest.php` | 17-32 |
| 借出事件监听器 | `app/Listeners/CheckoutableListener.php` | 73-165 |
| 通知收件人逻辑 | `app/Listeners/CheckoutableListener.php` | 472-564 |
| Setting 邮件路由 | `app/Models/Setting.php` | 249-254 |
| CheckoutAcceptance 邮件路由 | `app/Models/CheckoutAcceptance.php` | 31-39 |
| 申请通知 | `app/Notifications/RequestAssetNotification.php` | 13-127 |
| 取消通知 | `app/Notifications/RequestAssetCancelation.php` | 14-133 |
| 审批页视图 | `resources/views/hardware/requested.blade.php` | 107-115 |
| 清理命令 | `app/Console/Commands/CleanOldCheckoutRequests.php` | 8-77 |
| 可申请范围 | `app/Models/Asset.php` | 1788-1803 |
| 邮件配置 | `config/mail.php` | 237-256 |

---

## 九、潜在问题与建议

### 问题 1：Setting 模型的邮件路由不一致

**现象**: `Setting::routeNotificationForMail()` 返回 `config('mail.reply_to.address')`，但通知发送条件检查的是 `$settings->alert_email`。

**风险**: 当管理员配置了 `alert_email` 但 `MAIL_REPLYTO_ADDR` 为空时，通知会发送失败但条件检查通过。

**建议**: 修改 `Setting::routeNotificationForMail()` 使其返回 `alert_email`，与 `CheckoutAcceptance` 保持一致：
```php
public function routeNotificationForMail()
{
    $recipients_string = explode(',', self::getSettings()->alert_email);
    $recipients = array_map('trim', $recipients_string);
    return array_filter($recipients);
}
```

### 问题 2：缺少自动回退机制

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

### 问题 3：三个申请入口逻辑不统一

- `/request/{itemType}/{itemId}` 入口没有权限检查
- 通用入口不处理 `requests_counter`
- 通知触发条件不一致

**建议**: 统一使用 `CreateCheckoutRequestAction` 和 `CancelCheckoutRequestAction`，消除重复代码。

### 问题 4：fulfilled_at 字段未使用

数据库中有 `fulfilled_at` 字段但代码中从未使用，可能是遗留功能或未完成功能。

### 问题 5：审批状态不明确

系统没有明确的"已批准"、"已拒绝"状态，只有"已取消"和"待处理"。借出操作与审批操作耦合在一起，不利于审计追踪。

### 问题 6：alert_email 与 admin_cc_email 职责混淆

- `alert_email` 用于申请/取消通知的收件人
- `admin_cc_email` 用于借出/归还通知的抄送收件人
- 两个设置项功能相似但使用场景不同，容易造成混淆

**建议**: 考虑合并这两个设置项，或在界面上明确说明它们的区别。
