# 资产申请-审批-通知 状态流分析

---

## 零、事实校准

本章节对之前分析中的关键偏差进行校正，详细列出更正前后的对比。

### 0.1 审批页面 Cancel 按钮的实际跳转路径

**更正前**：
> 取消申请表单提交到 `POST /request/{itemType}/{itemId}/true/{requestingUser}`

**更正后**：
> 取消申请表单提交到路由 `route('account/request-item')`，实际路径为：
> `POST /request/{itemType}/{itemId}/{cancel_by_admin?}/{requestingUser?}`
>
> 路由名称：`account/request-item`（注意是斜杠分隔，不是点分隔）
> 控制器方法：`ViewAssetsController::getRequestItem()`

**完整链路**：
```
1. requested.blade.php:93-105
   <form method="POST" action="{{ route('account/request-item', [
       $request->itemType(),
       $request->requestable->id,
       true,                          // cancel_by_admin = true
       $request->requestingUser()->id  // 指定要取消的申请人
   ]) }}">

2. routes/web.php:412-413
   Route::post('request/{itemType}/{itemId}/{cancel_by_admin?}/{requestingUser?}',
       [ViewAssetsController::class, 'getRequestItem'])
       ->name('account/request-item');

3. ViewAssetsController::getRequestItem():202-211
   判断 $cancel_by_admin = true → 执行取消逻辑
```

---

### 0.2 审批页 Cancel 按钮 vs CancelCheckoutRequestAction 差异

| 对比项 | 审批页 Cancel 按钮 (`getRequestItem`) | CancelCheckoutRequestAction (API) |
|--------|---------------------------------------|------------------------------------|
| **调用方式** | Web 表单提交，路由 `account/request-item` | API 调用，`DELETE /api/v1/account/request/{asset}` |
| **计数器处理** | ❌ **不处理** `requests_counter` | ✅ `$asset->decrement('requests_counter', 1)` |
| **权限检查** | ❌ 无公司权限检查 | ✅ `Company::isCurrentUserHasAccess($asset)` |
| **通知触发条件** | ✅ 三重检查：<br>`alert_email != '' && alerts_enabled == '1' && !lock_passwords` | ❌ 无条件发送（try-catch 包裹） |
| **通知 locale** | ✅ `->locale($settings->locale)` | ❌ 未设置 locale |
| **日志行为** | ✅ `ActionType::RequestCanceled`（枚举） | ✅ `'request canceled'`（字符串） |
| **支持类型** | ✅ Asset / AssetModel | ❌ 仅 Asset |
| **指定申请人** | ✅ `$item->cancelRequest($requestingUser)`<br>可指定取消任意用户的申请 | ❌ `$asset->cancelRequest()`<br>无参数，取消当前用户的申请 |
| **申请数量** | ✅ 从 `$item_request->qty` 获取实际数量 | ❌ 固定为 1 |
| **异常处理** | ❌ 无 try-catch 包裹通知发送 | ✅ 用 try-catch 包裹，异常记录日志 |

**代码位置对比**：
- 审批页 Cancel：`app/Http/Controllers/ViewAssetsController.php:202-211`
- API CancelAction：`app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php:15-46`

---

### 0.3 checkout/checkin 邮件链路：Mailable vs Notification

**更正前**：
> `CheckoutAssetNotification` 用于发送邮件给借用人和管理员

**更正后**：
> checkout/checkin 的**邮件发送**和 **webhook 通知**使用**完全独立**的两套类：
>
> - **邮件链路**：使用 `Mailable` 类（`CheckoutAssetMail` 等）
> - **Webhook 链路**：使用 `Notification` 类（`CheckoutAssetNotification` 等）
>
> 且 `CheckoutAssetNotification` 的 `via()` 方法**不包含 'mail' 渠道**，只用于 webhook 通知。

**完整链路对比**：

| 维度 | 邮件链路 (Mailable) | Webhook 链路 (Notification) |
|------|---------------------|------------------------------|
| **触发点** | `CheckoutableListener@onCheckedOut`:94-124 | `CheckoutableListener@onCheckedOut`:127-164 |
| **使用类** | `CheckoutAssetMail`, `CheckinAssetMail` | `CheckoutAssetNotification`, `CheckinAssetNotification` |
| **类位置** | `app/Mail/` 目录 | `app/Notifications/` 目录 |
| **基类** | 继承 `BaseMailable` | 继承 `Notification` |
| **发送方式** | `Mail::to()->send($mailable)` | `Notification::route()->notify($notification)` |
| **获取方法** | `getCheckoutMailType()` → `CheckoutAssetMail::class` | `getCheckoutNotification()` → `CheckoutAssetNotification::class` |
| **通知渠道** | 仅邮件 (SMTP/Mailgun/Sendmail 等) | Slack / Microsoft Teams / Google Chat |
| **toMail 方法** | 有 `content()` / `envelope()` 方法 | ❌ **没有** `toMail()` 方法 |
| **via() 返回** | 不适用 (Mailable) | 只返回 webhook 渠道，**不含 'mail'** |

**关键代码证据**：

`CheckoutAssetNotification::via()` (`app/Notifications/CheckoutAssetNotification.php:59-80`)：
```php
public function via()
{
    $notifyBy = [];
    // 只添加 webhook 渠道，不包含 'mail'
    if (Setting::getSettings()->webhook_selected === 'google' && Setting::getSettings()->webhook_endpoint) {
        $notifyBy[] = GoogleChatChannel::class;
    }
    if (Setting::getSettings()->webhook_selected === 'microsoft' && Setting::getSettings()->webhook_endpoint) {
        $notifyBy[] = MicrosoftTeamsChannel::class;
    }
    if (Setting::getSettings()->webhook_selected === 'slack' || Setting::getSettings()->webhook_selected === 'general') {
        $notifyBy[] = SlackWebhookChannel::class;
    }
    return $notifyBy; // 不包含 'mail'
}
```

`CheckoutableListener@onCheckedOut()` 中的双链路：
```php
// 链路 1：邮件发送 - 使用 Mailable
if ($shouldSendEmailToUser || $shouldSendEmailToAlertAddress) {
    $mailable = $this->getCheckoutMailType($event, $acceptance); // 返回 CheckoutAssetMail
    // ...
    Mail::to(array_flatten($to))->send($toMail);
    Mail::cc(array_flatten($cc))->send($ccMail);
}

// 链路 2：Webhook 通知 - 使用 Notification
if ($shouldSendWebhookNotification) {
    $notification = $this->getCheckoutNotification($event, $acceptance, true); // 返回 CheckoutAssetNotification
    // ...
    Notification::route(...)->notify($notification);
}
```

---

### 0.4 更正后的通知类型与触发时机

| 通知类 | 类型 | 触发时机 | 渠道 | 接收者 |
|--------|------|---------|------|--------|
| `RequestAssetNotification` | Notification | 用户提交申请时 | 邮件 + Webhook | reply_to 地址 |
| `RequestAssetCancelation` | Notification | 申请被取消时 | 邮件 + Webhook | reply_to 地址 |
| `CheckoutAssetMail` | Mailable | 资产 checkout 成功时 | 邮件 | 借用人 + admin_cc_email |
| `CheckinAssetMail` | Mailable | 资产 checkin 成功时 | 邮件 | 借用人 + admin_cc_email |
| `CheckoutAssetNotification` | Notification | 资产 checkout 成功时 | Webhook | Slack/Teams/Google Chat |
| `CheckinAssetNotification` | Notification | 资产 checkin 成功时 | Webhook | Slack/Teams/Google Chat |

---

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
   - 表单提交到路由 `account/request-item` → `POST /request/{itemType}/{itemId}/true/{requestingUser}`
   - 调用 `ViewAssetsController@getRequestItem` 内部逻辑（不使用 Action 类）
   - 记录日志 'request canceled'
   - ❌ **不处理** requests_counter（存在数据不一致风险）
   - 发送取消通知邮件（需满足三重条件检查）

2. **借出资产** (Checkout) - **即审批通过**
   - 链接跳转到 `GET /hardware/{asset}/checkout`
   - 由管理员执行 checkout 操作完成审批
   - 如果资产已被占用 (`assigned_to != ''`)，按钮变为 "Checkin"

### API 取消申请动作（CancelCheckoutRequestAction）
**文件**: `app/Actions/CheckoutRequests/CancelCheckoutRequestAction.php:13-47`

> ⚠️ 此为 API 取消路径，**审批页 Cancel 按钮不使用此 Action**。

```
1. 权限检查（公司权限）
2. 取消申请 ($asset->cancelRequest() → 设置 canceled_at)
3. 递减 requests_counter
4. 记录操作日志 (Actionlog: 'request canceled')
5. 发送取消通知邮件 (RequestAssetCancelation) - 无条件（try-catch包裹）
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

> ⚠️ **重要区分**：checkout/checkin 的邮件和 Webhook 使用**完全独立**的两套类。

| 类名 | 类型 | 触发时机 | 渠道 | 接收者 | 主题 |
|------|------|---------|------|--------|------|
| `RequestAssetNotification` | Notification | 用户提交申请时 | 邮件 + Webhook | reply_to 地址 | 👀 Item Requested |
| `RequestAssetCancelation` | Notification | 申请被取消时 | 邮件 + Webhook | reply_to 地址 | ⚠️ Request Canceled |
| `CheckoutAssetMail` | Mailable | 资产 checkout 成功时 | 邮件 | 借用人 + admin_cc_email | Asset Checkout Notification |
| `CheckinAssetMail` | Mailable | 资产 checkin 成功时 | 邮件 | 借用人 + admin_cc_email | Asset Checkin Notification |
| `CheckoutAssetNotification` | Notification | 资产 checkout 成功时 | Webhook | Slack/Teams/Google Chat | :arrow_up: :computer: Asset Checkout |
| `CheckinAssetNotification` | Notification | 资产 checkin 成功时 | Webhook | Slack/Teams/Google Chat | :arrow_down: :computer: Asset Checkin |

### 事件驱动机制

资产借出/归还通过事件系统触发通知，**邮件和 Webhook 是两条独立链路**：

1. **资产借出**: `Asset::checkOut()` → `event(new CheckoutableCheckedOut(...))`
   - **文件**: `app/Models/Asset.php:565`

2. **事件订阅**: `CheckoutableListener` 订阅 `CheckoutableCheckedOut` 和 `CheckoutableCheckedIn` 事件
   - **文件**: `app/Providers/EventServiceProvider.php:34-38`
   - **文件**: `app/Listeners/CheckoutableListener.php:56-67`

3. **通知发送**: `CheckoutableListener@onCheckedOut`
   - **文件**: `app/Listeners/CheckoutableListener.php:73-165`
   - **邮件链路**（94-124行）：使用 `CheckoutAssetMail` (Mailable)
     - 发送给借用人：需要验收 或 有EULA 或 checkin_email 启用
     - 抄送给 admin_cc_email：有 acceptance 或 admin_cc_always 启用
   - **Webhook 链路**（127-164行）：使用 `CheckoutAssetNotification` (Notification)
     - 发送到 Slack / Microsoft Teams / Google Chat
     - `CheckoutAssetNotification::via()` 只返回 webhook 渠道，**不含 'mail'**

### 申请通知的特殊逻辑
**文件**: `app/Actions/CheckoutRequests/CreateCheckoutRequestAction.php:47-50`

申请和取消的通知**不通过事件系统**，而是直接调用：
```php
$settings->notify((new RequestAssetNotification($data))->locale($settings->locale));
```

### 申请取消的两条路径

| 路径 | 调用代码 | 计数器 | 权限检查 | 通知条件 |
|------|---------|--------|---------|---------|
| **审批页 Cancel 按钮** | `ViewAssetsController@getRequestItem` | ❌ 不处理 | ❌ 无 | ✅ 三重条件检查 |
| **API 取消** | `CancelCheckoutRequestAction::run()` | ✅ 递减 | ✅ 公司权限 | ❌ 无条件 |

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
        ├─ 🔗 邮件链路（使用 Mailable）
        │   ├─ 是否发送给用户？
        │   │   ├─ 非批量 checkout
        │   │   ├─ 资产 requireAcceptance() 或有 EULA 或 checkin_email 启用
        │   │   └─ 收件人：借用人 (User) → Mail::to()
        │   │
        │   ├─ 是否发送给管理员？
        │   │   ├─ 非批量 checkout
        │   │   ├─ 有 acceptance 或 admin_cc_always 启用
        │   │   ├─ admin_cc_email 已设置
        │   │   └─ 收件人：admin_cc_email → Mail::cc()
        │   │
        │   └─ 使用类：CheckoutAssetMail (Mailable)
        │
        └─ 🔗 Webhook 链路（使用 Notification）
            ├─ webhook_endpoint 已配置
            ├─ 使用类：CheckoutAssetNotification (Notification)
            └─ 渠道：Slack / Microsoft Teams / Google Chat
```

**代码位置**: `app/Listeners/CheckoutableListener.php:73-165`

### 5.4 通知发送条件汇总表

| 通知类型 | 类名 | 类型 | 触发条件 | 收件人来源 | 检查的设置项 |
|---------|------|------|---------|-----------|-------------|
| **申请通知** | `RequestAssetNotification` | Notification | 用户提交申请 | `config('mail.reply_to.address')` | `alert_email != '' && alerts_enabled == '1'` |
| **取消通知（审批页）** | `RequestAssetCancelation` | Notification | 申请被取消（审批页） | `config('mail.reply_to.address')` | `alert_email != '' && alerts_enabled == '1' && !lock_passwords` |
| **取消通知（API）** | `RequestAssetCancelation` | Notification | 申请被取消（API） | `config('mail.reply_to.address')` | 无条件（try-catch） |
| **借出邮件（用户）** | `CheckoutAssetMail` | Mailable | 资产 checkout 成功 | 借用人的 email | `requireAcceptance() 有 EULA checkin_email 启用` |
| **借出邮件（管理员）** | `CheckoutAssetMail` | Mailable | 资产 checkout 成功 | `admin_cc_email` | `admin_cc_email 已设置 (有 acceptance 或 admin_cc_always)` |
| **借出 Webhook** | `CheckoutAssetNotification` | Notification | 资产 checkout 成功 | webhook_endpoint | `webhook_endpoint 已配置` |
| **归还邮件（用户）** | `CheckinAssetMail` | Mailable | 资产 checkin 成功 | 借用人的 email | `checkin_email 启用` |
| **归还邮件（管理员）** | `CheckinAssetMail` | Mailable | 资产 checkin 成功 | `admin_cc_email` | `admin_cc_email 已设置` |
| **归还 Webhook** | `CheckinAssetNotification` | Notification | 资产 checkin 成功 | webhook_endpoint | `webhook_endpoint 已配置` |

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
    │   │       └─ 执行 checkout 流程
    │   │           ├─ 🔗 邮件链路：CheckoutAssetMail
    │   │           └─ 🔗 Webhook 链路：CheckoutAssetNotification
    │   │
    │   └─ 分支 1b：点击 "Cancel"（审批拒绝）
    │       └─ 提交到 /request/{itemType}/{itemId}/true/{requestingUser}
    │           └─ 执行 ViewAssetsController@getRequestItem（内部逻辑）
    │               └─ 发送 RequestAssetCancelation
    │
    └─ 情况 2：资产已被占用 (assigned_to != '')
        │
        ├─ 分支 2a：点击 "Checkin"（先归还再借出）
        │   ├─ 跳转到 /hardware/{asset}/checkin
        │   ├─ 执行 checkin 流程
        │   │   ├─ 触发 CheckoutableCheckedIn 事件
        │   │   ├─ 🔗 邮件链路：CheckinAssetMail
        │   │   └─ 🔗 Webhook 链路：CheckinAssetNotification
        │   └─ 归还后资产变为可用状态
        │       └─ 管理员需再次进入审批列表，点击 "Checkout"
        │
        └─ 分支 2b：点击 "Cancel"（直接取消申请）
            └─ 提交到 /request/{itemType}/{itemId}/true/{requestingUser}
                └─ 执行 ViewAssetsController@getRequestItem（内部逻辑）
                    └─ 发送 RequestAssetCancelation
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
│  getRequestItem  │  │  AssetCheckout-  │  │  2a: Checkin     │
│  (内部逻辑)      │  │  Controller@store│  │  2b: Cancel      │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
         ├─ 设置 canceled_at   ├─ 前置检查          ├─ 2a: 先归还资产
         ├─ ❌ 不处理计数器    ├─ availableForCheckout() ├─ 2b: 直接取消
         ├─ 记录 'request canceled' ├─ 设置 assigned_to └───────────┬───────────┘
         └─ 发送 RequestAssetCancelation ├─ 触发 CheckoutableCheckedOut 事件
                               │                     │
                               ├─ 🔗 邮件链路：CheckoutAssetMail
                               │   ├─ Mail::to() → 借用人
                               │   └─ Mail::cc() → admin_cc_email
                               │
                               ├─ 🔗 Webhook 链路：CheckoutAssetNotification
                               │   └─ Slack / Teams / Google Chat
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
| **借出邮件 (Mailable)** | `app/Mail/CheckoutAssetMail.php` | 17-179 |
| **归还邮件 (Mailable)** | `app/Mail/CheckinAssetMail.php` | - |
| **借出 Webhook (Notification)** | `app/Notifications/CheckoutAssetNotification.php` | 24-176 |
| **归还 Webhook (Notification)** | `app/Notifications/CheckinAssetNotification.php` | - |
| 审批页视图 | `resources/views/hardware/requested.blade.php` | 93-115 |
| 清理命令 | `app/Console/Commands/CleanOldCheckoutRequests.php` | 8-77 |
| 可申请范围 | `app/Models/Asset.php` | 1788-1803 |
| 邮件配置 | `config/mail.php` | 237-256 |
| 路由定义 | `routes/web.php` | 412-413 |

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

### 问题 3：两条取消路径逻辑不一致（⚠️ 高风险）

审批页 Cancel 按钮与 API 取消使用完全不同的实现，存在数据不一致风险：

| 对比项 | 审批页 Cancel | API CancelAction |
|--------|-------------|-----------------|
| `requests_counter` | ❌ 不处理 | ✅ 递减 |
| 权限检查 | ❌ 无公司权限检查 | ✅ 有 |
| 通知条件 | ✅ 三重检查 | ❌ 无条件 |
| 通知 locale | ✅ 设置 | ❌ 未设置 |
| 支持类型 | ✅ Asset + AssetModel | ❌ 仅 Asset |

**风险**：审批页取消申请后，`requests_counter` 不会递减，导致计数器数据不一致。

**建议**: 审批页取消操作也调用 `CancelCheckoutRequestAction::run()`，统一逻辑。

### 问题 4：checkout/checkin 邮件与 Webhook 类命名易混淆

- `CheckoutAssetMail` (Mailable) - 用于邮件发送
- `CheckoutAssetNotification` (Notification) - 仅用于 Webhook，不含 mail 渠道

**风险**: 开发者容易误以为 `CheckoutAssetNotification` 会发送邮件，但实际上它的 `via()` 方法只返回 webhook 渠道。

**建议**: 重命名 `CheckoutAssetNotification` 为 `CheckoutAssetWebhookNotification`，明确其用途。

### 问题 5：三个申请入口逻辑不统一

- `/request/{itemType}/{itemId}` 入口没有权限检查
- 通用入口不处理 `requests_counter`
- 通知触发条件不一致

**建议**: 统一使用 `CreateCheckoutRequestAction` 和 `CancelCheckoutRequestAction`，消除重复代码。

### 问题 6：fulfilled_at 字段未使用

数据库中有 `fulfilled_at` 字段但代码中从未使用，可能是遗留功能或未完成功能。

### 问题 7：审批状态不明确

系统没有明确的"已批准"、"已拒绝"状态，只有"已取消"和"待处理"。借出操作与审批操作耦合在一起，不利于审计追踪。

### 问题 8：alert_email 与 admin_cc_email 职责混淆

- `alert_email` 用于申请/取消通知的**判断条件**（但实际发送到 `reply_to`）
- `admin_cc_email` 用于借出/归还通知的**抄送收件人**
- 两个设置项功能相似但使用场景不同，容易造成混淆

**建议**: 考虑合并这两个设置项，或在界面上明确说明它们的区别。

---

## 十、一致性核对清单

本清单列出所有已修正的冲突点，确保全文同一事实只保留一种说法。

---

### 冲突点 1：审批页 Cancel 按钮调用链

| 项目 | 内容 |
|------|------|
| **更正前** | 审批页 Cancel 按钮调用 `CancelCheckoutRequestAction` 并递减 `requests_counter` |
| **更正后** | 审批页 Cancel 按钮走 `ViewAssetsController@getRequestItem` 内部逻辑，不使用 Action 类，**不处理** `requests_counter` |
| **代码证据** | `resources/views/hardware/requested.blade.php:93-105` → `route('account/request-item')` → `ViewAssetsController::getRequestItem()` `app/Http/Controllers/ViewAssetsController.php:202-211`（无 `decrement` 调用） |
| **文档位置** | 第三章"审批操作"小节（第246-251行）、第六章"回退分支"（第548-550、564-567行） |

---

### 冲突点 2：回退分支图 Cancel 分支链路

| 项目 | 内容 |
|------|------|
| **更正前** | 回退分支图中 Cancel 分支标注为 `执行 CancelCheckoutRequestAction` |
| **更正后** | 回退分支图中 Cancel 分支标注为 `执行 ViewAssetsController@getRequestItem（内部逻辑）` |
| **代码证据** | `resources/views/hardware/requested.blade.php:93-105` 表单提交到 `route('account/request-item')` |
| **文档位置** | 第六章"完整回退分支"（第548-550、564-567行） |

---

### 冲突点 3：Checkout 通知类混写

| 项目 | 内容 |
|------|------|
| **更正前** | Checkout 流程描述为"发送 `CheckoutAssetNotification`"，混淆邮件与 Webhook 类 |
| **更正后** | Checkout 流程为双链路：🔗 邮件链路使用 `CheckoutAssetMail` (Mailable)，🔗 Webhook 链路使用 `CheckoutAssetNotification` (Notification) |
| **代码证据** | `app/Listeners/CheckoutableListener.php:94-165` <br> 邮件：`getCheckoutMailType()` → `CheckoutAssetMail::class` → `Mail::to()->send()` <br> Webhook：`getCheckoutNotification()` → `CheckoutAssetNotification::class` → `Notification::route()->notify()` <br> `app/Notifications/CheckoutAssetNotification.php:59-80` → `via()` 只返回 webhook 渠道，**不含 'mail'** |
| **文档位置** | 第六章"完整回退分支"（第544-546行） |

---

### 冲突点 4：Checkin 通知类混写

| 项目 | 内容 |
|------|------|
| **更正前** | Checkin 流程描述为"发送 `CheckinAssetNotification`"，混淆邮件与 Webhook 类 |
| **更正后** | Checkin 流程为双链路：🔗 邮件链路使用 `CheckinAssetMail` (Mailable)，🔗 Webhook 链路使用 `CheckinAssetNotification` (Notification) |
| **代码证据** | `app/Listeners/CheckoutableListener.php` → `onCheckedIn()` 方法逻辑与 `onCheckedOut()` 对称，同样使用双链路 |
| **文档位置** | 第六章"完整回退分支"（第556-560行） |

---

### 冲突点 5：取消申请动作小节标题

| 项目 | 内容 |
|------|------|
| **更正前** | 小节标题为"取消申请动作"，易误解为所有取消路径 |
| **更正后** | 小节标题为"API 取消申请动作（CancelCheckoutRequestAction）"，并添加警告说明审批页不使用此 Action |
| **代码证据** | 审批页取消使用内部逻辑（见冲突点1），只有 API 取消使用 `CancelCheckoutRequestAction` |
| **文档位置** | 第三章（第258-269行） |

---

### 冲突点 6：审批页 Cancel 按钮路径描述

| 项目 | 内容 |
|------|------|
| **更正前** | 审批页 Cancel 按钮路径描述为 `POST /request/{itemType}/{itemId}/true/{requestingUser}`（仅 URL） |
| **更正后** | 审批页 Cancel 按钮路径描述为 `路由 account/request-item → POST /request/{itemType}/{itemId}/true/{requestingUser}`（包含路由名称） |
| **代码证据** | `routes/web.php:412-413` → `Route::post('request/...')->name('account/request-item')` |
| **文档位置** | 第三章"审批操作"小节（第247行） |

---

### 全文统一事实汇总

| 事实 | 统一说法 | 出现章节 |
|------|---------|---------|
| 审批页 Cancel 调用链 | `ViewAssetsController@getRequestItem` 内部逻辑，不使用 Action 类 | 第三章、第六章、第七章 |
| 审批页 Cancel 计数器 | ❌ 不处理 `requests_counter` | 第三章、第五章、第七章 |
| API 取消调用链 | `CancelCheckoutRequestAction::run()` | 第三章、第五章 |
| API 取消计数器 | ✅ 递减 `requests_counter` | 第三章、第五章 |
| Checkout 邮件类 | `CheckoutAssetMail` (Mailable) | 第四章、第五章、第六章、第七章 |
| Checkout Webhook 类 | `CheckoutAssetNotification` (Notification，不含 mail 渠道) | 第四章、第五章、第六章、第七章 |
| Checkin 邮件类 | `CheckinAssetMail` (Mailable) | 第四章、第五章、第六章 |
| Checkin Webhook 类 | `CheckinAssetNotification` (Notification，不含 mail 渠道) | 第四章、第五章、第六章 |
| 申请/取消通知收件人 | `config('mail.reply_to.address')`（通过 `Setting::routeNotificationForMail()`） | 第五章 |
| 借出/归还邮件收件人 | 借用人 + `admin_cc_email`（抄送） | 第四章、第五章 |
