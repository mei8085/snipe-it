# Snipe-IT 软件许可席位分配与回收——可用余量与归属关系同步维护链路分析

## 1. 数据模型概览

### 1.1 核心表结构

| 表名 | 关键字段 | 作用 |
|------|----------|------|
| `licenses` | `id`, `seats`(总量), `reassignable`(bool), `company_id` | 许可证主表，`seats` 字段记录声明的席位总数 |
| `license_seats` | `id`, `license_id`, `assigned_to`(nullable), `asset_id`(nullable), `unreassignable_seat`(bool), `notes`, `created_by`, `deleted_at` | 席位实例表，每条记录对应一个席位 |

> 原始建表迁移：[2013_11_25_031458_create_license_seats_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/database/migrations/2013_11_25_031458_create_license_seats_table.php)  
> `asset_id` 列追加：[2014_07_17_085822_add_asset_to_software.php](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/database/migrations/2014_07_17_085822_add_asset_to_software.php)  
> `unreassignable_seat` 列追加：[2025_01_15_190348_adds_unavailable_to_license_seats_tables.php](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/database/migrations/2025_01_15_190348_adds_unavailable_to_license_seats_tables.php)

### 1.2 模型关系

```
License (1) ──hasMany──▶ LicenseSeat (N)
LicenseSeat ──belongsTo──▶ License      (license_id)
LicenseSeat ──belongsTo──▶ User         (assigned_to)   ← 归属关系：用户
LicenseSeat ──belongsTo──▶ Asset        (asset_id)      ← 归属关系：资产
License ──belongsToMany──▶ User         (通过 license_seats.assigned_to → license_seats.license_id)
User   ──belongsToMany──▶ License       (通过 license_seats.assigned_to → license_seats.license_id)
```

> 模型定义分别位于 [License.php](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php) 和 [LicenseSeat.php](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/LicenseSeat.php)

---

## 2. 可用余量的计算机制——"无冗余字段，纯查询推导"

Snipe-IT **没有**在 `licenses` 表上维护一个 `remaining_seats` 计数字段。所有余量数据均通过 **实时查询 `license_seats` 表的状态** 来推导。这意味着余量永远是 `license_seats` 记录当前归属状态的投影，不存在"缓存不同步"问题。

### 2.1 核心计算方法

| 方法 | 位置 | 逻辑 | 返回类型 |
|------|------|------|----------|
| `remaincount()` | [License.php#L740-L748](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L740-L748) | `总席位数 - 已分配席位数 - 不可再分配席位数` | `int` |
| `availCount()` | [License.php#L631-L638](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L631-L638) | 查询 `assigned_to IS NULL AND asset_id IS NULL AND unreassignable_seat = false AND deleted_at IS NULL` | `Relation`（可 count） |
| `getAvailSeatsCountAttribute()` | [License.php#L668-L675](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L668-L675) | 对 `availCount()` 取聚合 count | `int` |
| `freeSeat()` | [License.php#L806-L817](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L806-L817) | `availCount()` 条件 + `orderBy('id','asc')->first()` | `LicenseSeat\|null` |
| `freeSeats()`（关系） | [License.php#L828-L831](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L828-L831) | `whereNull('assigned_to')->whereNull('deleted_at')->whereNull('asset_id')` **⚠ 不检查 `unreassignable_seat`** | `Relation`（hasMany） |
| `getFreeSeatCountAttribute()` | [License.php#L393-L396](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L393-L396) | 委托给 `remaincount()`，作为 `$appends` 属性暴露 | `int` |
| `percentRemaining()` | [License.php#L465-L476](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L465-L476) | `(可用席位数 / 总席位数) * 100` | `float` |

### 2.2 `freeSeat()` 与 `freeSeats()` 的关键差异

这是容易混淆的两个方法，也是边界问题的根源之一：

| 维度 | `freeSeat()`（方法） | `freeSeats()`（关系） |
|------|---------------------|-----------------------|
| 返回值 | 单条 `LicenseSeat\|null` | `HasMany` 关系（多条） |
| `assigned_to IS NULL` | ✅ 是 | ✅ 是 |
| `asset_id IS NULL` | ✅ 是 | ✅ 是 |
| `deleted_at IS NULL` | ✅ 是 | ✅ 是 |
| `unreassignable_seat = false` | ✅ **是** | ❌ **否（缺漏）** |
| 排序 | `ORDER BY id ASC` | 无排序 |
| 主要使用方 | Web checkout、批量 checkout、CLI checkout | **预定义套件 checkout** |

**结论**：`freeSeats()` 关系缺漏了 `unreassignable_seat = false` 条件，因此通过它获取的"空闲"席位可能包含已被标记为不可再分配的席位。这会导致预定义套件分配路径与其他路径在边界判断上不一致。

**`remaincount()` 的核心公式：**

```
可用余量 = license_seats_count(总记录数) - assigned_seats_count(已分配数) - unReassignableCount(不可再分配数)
```

其中：
- `license_seats_count` 通过 `licenseSeatsRelation` 聚合查询得到（[License.php#L573-L594](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L573-L594)）
- `assigned_seats_count` 通过 `assignedCount` 查询 `assigned_to IS NOT NULL OR asset_id IS NOT NULL` 得到（[License.php#L686-L712](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L686-L712)）
- `unReassignableCount` 查询 `unreassignable_seat = true` 的席位数量（[License.php#L721-L731](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L721-L731)）

---

## 3. 席位分配（Checkout）链路

### 3.1 Web UI 单席分配

**入口路由：** `POST /licenses/{licenseId}/checkout/{seatId?}`  
**控制器：** [LicenseCheckoutController::store()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L79-L157)

```
请求进入
  │
  ├─ 1. License::find($licenseId)        查找许可证
  ├─ 2. authorize('checkout', $license)  权限校验
  ├─ 3. availCount()->count() < 1 ?      余量守卫 ← 【关键同步点】
  ├─ 4. isInactive() ?                   有效性守卫（过期/终止）
  ├─ 5. findLicenseSeatToCheckout()      确定目标席位
  │     ├─ LicenseSeat::find($seatId)    指定席位
  │     └─ $license->freeSeat()          或取第一个空闲席位 ← 【关键同步点】
  ├─ 6. checkoutToUser() / checkoutToAsset()  写入归属
  │     ├─ $licenseSeat->assigned_to = $user->id   (分配给用户)
  │     ├─ $licenseSeat->asset_id = $asset->id     (分配给资产)
  │     ├─ $licenseSeat->save()                     持久化 ← 【余量自动减少】
  │     └─ event(CheckoutableCheckedOut)            触发事件
  └─ 7. 返回重定向
```

**归属关系写入后的余量自动变化原理：** 当 `assigned_to` 或 `asset_id` 从 `NULL` 变为非空值并 `save()` 后，该席位不再满足 `availCount()` 的查询条件（`whereNull('assigned_to')` / `whereNull('asset_id')`），因此下一次查询余量时自动减少。**无需手动维护计数器。**

### 3.2 API 席位更新（分配/回收统一入口）

**入口路由：** `PUT /api/licenses/{licenseId}/seats/{seatId}`  
**控制器：** [LicenseSeatsController::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Api/LicenseSeatsController.php#L101-L248)

```
请求进入
  │
  ├─ 1. 校验 assigned_to / asset_id（互斥，prohibits 规则）
  ├─ 2. authorize('checkout', License::class)
  ├─ 3. 查找 licenseSeat，校验 license 归属
  ├─ 4. FMCS 公司隔离校验（full_multiple_companies_support）
  ├─ 5. 记录 oldUser / oldAsset（用于日志）
  ├─ 6. $licenseSeat->fill($validated)
  ├─ 7. 判断操作类型：
  │     ├─ isDirty('assigned_to') || isDirty('asset_id')  →  归属变更
  │     ├─ assigned_to === null && asset_id === null       →  回收(checkin)
  │     └─ 否则                                             →  分配(checkout)
  ├─ 8. unreassignable_seat 守卫：不可再分配的席位拒绝变更
  └─ 9. DB::transaction {                    ← 【事务原子性保障】
         $licenseSeat->save()
         if (checkin && !reassignable)
           $licenseSeat->unreassignable_seat = true; save()
         logCheckin / logCheckout
       }
```

**事务保障要点：** `save()` 与 `logCheckin/logCheckout` 被包裹在同一 `DB::transaction` 中，确保席位状态变更与审计日志的原子性——要么全部成功，要么全部回滚。

### 3.3 批量分配

**入口路由：** `POST /licenses/{licenseId}/bulkcheckout`  
**控制器：** [LicenseCheckoutController::bulkCheckout()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L227-L281)

```
获取 avail_count = getAvailSeatsCountAttribute()
获取开启了 autoassign_licenses 的用户列表
foreach user:
  ├─ 跳过已有此许可证的用户
  ├─ $license->freeSeat()           取空闲席位
  ├─ $licenseSeat->assigned_to = user->id
  ├─ $licenseSeat->save()            ← 持久化，余量自动减少
  ├─ $avail_count--                  ← 内存计数器递减（提前退出判断用）
  ├─ logCheckout()
  └─ if $avail_count == 0 → break
```

### 3.4 CLI 批量分配

**命令：** `php artisan snipeit:checkout-to-all --license_id=X`  
**文件：** [CheckoutLicenseToAllUsers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Console/Commands/CheckoutLicenseToAllUsers.php#L80-L103)

逻辑与 Web 批量分配一致：循环用户 → `freeSeat()` → 设 `assigned_to` → `save()` → `logCheckout()`。

### 3.5 预定义套件分配

**服务：** [PredefinedKitCheckoutService](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Services/PredefinedKitCheckoutService.php#L103-L120)

```
getLicenseSeatsToAdd():
  foreach kit.licenses:
    ├─ 检查 freeSeats 数量是否 ≥ pivot.quantity
    ├─ 取前 N 个 freeSeats 作为待分配席位
  saveToDb() [DB::transaction]:
    foreach seats:
      ├─ assigned_to = user->id
      ├─ save()
      └─ event(CheckoutableCheckedOut)
```

---

## 4. 席位回收（Checkin）链路

### 4.1 Web UI 单席回收

**入口路由：** `POST /licenses/{licenseId}/checkin/{backTo?}`  
**控制器：** [LicenseCheckinController::store()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckinController.php#L57-L120)

```
请求进入
  │
  ├─ 1. LicenseSeat::find($seatId)
  ├─ 2. 守卫：assigned_to 和 asset_id 都为 null → 不可回收
  ├─ 3. authorize('checkout', $license)
  ├─ 4. 记录 return_to（原归属用户/资产，用于日志）
  ├─ 5. 清除归属：
  │     ├─ $licenseSeat->assigned_to = null
  │     └─ $licenseSeat->asset_id = null
  ├─ 6. 不可再分配标记：
  │     if (!license->reassignable)
  │       $licenseSeat->unreassignable_seat = true   ← 【关键：余量不会恢复】
  ├─ 7. $licenseSeat->save()                          ← 持久化
  └─ 8. event(CheckoutableCheckedIn)                  触发事件
```

**`reassignable` 与 `unreassignable_seat` 的交互：**

| `license.reassignable` | 回收后 `unreassignable_seat` | 余量是否恢复 |
|:---:|:---:|:---:|
| `true` | 保持 `false` | ✅ 恢复（席位重新进入可用池） |
| `false` | 设置为 `true` | ❌ 不恢复（席位被永久"冻结"） |

这意味着：当一个许可证标记为"不可再分配"时，即使席位被回收，该席位的 `unreassignable_seat` 会被置为 `true`，导致 `availCount()` 查询（条件含 `unreassignable_seat = false`）不会将其计入可用余量。

### 4.2 API 回收

与 API 分配共享同一个 `update()` 方法（见 3.2），通过 `is_checkin` 标志位区分。关键差异：回收时额外执行：

```php
if (!$licenseSeat->license->reassignable) {
    $licenseSeat->unreassignable_seat = true;
    $licenseSeat->save();
}
$licenseSeat->logCheckin($target, $notes);
```

### 4.3 批量回收

**入口路由：** `POST /licenses/{licenseId}/bulkcheckin`  
**控制器：** [LicenseCheckinController::bulkCheckin()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckinController.php#L134-L178)

分两轮处理：
1. **按用户回收**：`whereNotNull('assigned_to')` → `assigned_to = null` + 不可再分配标记 + `save()` + `logCheckin()`
2. **按资产回收**：`whereNotNull('asset_id')` → `asset_id = null` + 不可再分配标记 + `save()` + `logCheckin()`

---

## 5. 席位总量变更时的同步调整

当管理员修改 `licenses.seats` 字段时，需要同步增减 `license_seats` 记录数。

**触发点：** [License::boot()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L142-L170) 中的模型事件监听

```
License::created  → adjustSeatCount($license, 0, $newSeats)     // 新建时从 0 创建
License::updating → adjustSeatCount($license, $oldCount, $new)  // 更新时按实际记录数调整
```

**[adjustSeatCount()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L218-L287) 逻辑：**

```
oldSeats == newSeats → 无操作，返回 true

newSeats > oldSeats（增加席位）:
  ├─ 循环创建 LicenseSeat 记录（license_id, created_by, timestamps）
  ├─ 分块(1000) + DB::transaction 批量插入
  └─ 记录 Actionlog（"added N seats"）

newSeats < oldSeats（减少席位）:
  ├─ 查找可用待删除席位：whereNull('assigned_to')->whereNull('asset_id')->limit($change)
  ├─ if 待删除数 < 需删除数 → 拒绝操作（Session::flash error） ← 【保护已分配席位】
  ├─ 删除找到的席位
  └─ 记录 Actionlog（"deleted N seats"）
```

**关键安全机制：** 减少席位时，只会删除 `assigned_to IS NULL AND asset_id IS NULL` 的空闲席位。如果已分配席位数使得无法删除足够的席位，操作会被拒绝。这保证了已建立的归属关系不会被强制破坏。

---

## 6. 事件驱动的通知与日志同步

### 6.1 事件体系

```
CheckoutableCheckedOut  ──▶  LogListener::onCheckoutableCheckedOut()  ──▶  logCheckout()
                     └──▶  CheckoutableListener::onCheckedOut()       ──▶  邮件/Webhook通知 + CheckoutAcceptance

CheckoutableCheckedIn   ──▶  LogListener::onCheckoutableCheckedIn()   ──▶  logCheckin()
                     └──▶  CheckoutableListener::onCheckedIn()        ──▶  邮件/Webhook通知 + 删除待处理Acceptance
```

### 6.2 审计日志的归属映射

在 [Loggable::determineLogItemType()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/Traits/Loggable.php#L212-L224) 中，LicenseSeat 的日志特殊处理：

```php
if (static::class == LicenseSeat::class) {
    $log->item_type = License::class;   // 日志关联到 License 而非 LicenseSeat
    $log->item_id = $this->license_id;
}
```

这意味着所有席位操作的审计日志在 `action_logs` 表中关联的是 `License` 类，而非 `LicenseSeat` 类，使得在许可证维度查看操作历史更为直观。

### 6.3 事件监听注册

| 监听器 | 注册方式 | 文件 |
|--------|---------|------|
| [LogListener](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Listeners/LogListener.php) | `subscribe()` 方法批量注册 | 监听 5 类事件 |
| [CheckoutableListener](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Listeners/CheckoutableListener.php) | `subscribe()` 方法 | 监听 CheckIn/CheckOut |

---

## 7. 完整同步维护链路总结图

```
┌──────────────────────────────────────────────────────────────────────┐
│                      席位分配 (Checkout)                              │
│                                                                      │
│  请求 ──▶ 权限校验 ──▶ 余量检查(availCount) ──▶ 取空闲席位(freeSeat) │
│                                              │                       │
│                                    ┌─────────▼──────────┐           │
│                                    │ LicenseSeat.save()  │           │
│                                    │ assigned_to = X     │           │
│                                    │ asset_id = Y        │           │
│                                    └─────────┬──────────┘           │
│                                              │                       │
│                         ┌────────────────────┼────────────────┐     │
│                         ▼                    ▼                ▼     │
│                  DB 记录已变更      CheckoutableCheckedOut   Log    │
│                  余量自动减少       → 邮件通知                记录   │
│                  (下次查询时        → Webhook通知             写入   │
│                   自然体现)         → Acceptance创建          action │
│                                                              _logs  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                      席位回收 (Checkin)                               │
│                                                                      │
│  请求 ──▶ 权限校验 ──▶ 已分配守卫 ──▶ 记录原归属 ──▶ 清除归属字段   │
│                                              │                       │
│                                    ┌─────────▼──────────┐           │
│                                    │ LicenseSeat.save()  │           │
│                                    │ assigned_to = null  │           │
│                                    │ asset_id = null     │           │
│                                    └─────────┬──────────┘           │
│                                              │                       │
│                              ┌───────────────┼───────────────┐      │
│                              ▼                               ▼      │
│                   reassignable=true              reassignable=false │
│                   余量自动恢复                    unreassignable_    │
│                   (席位重回可用池)                 seat = true       │
│                                                  余量不恢复         │
│                                                  (席位被冻结)       │
│                                              │                       │
│                         ┌────────────────────┼────────────────┐     │
│                         ▼                    ▼                ▼     │
│                  DB 记录已变更      CheckoutableCheckedIn    Log    │
│                  余量按规则变化     → 邮件通知                记录   │
│                                    → Webhook通知             写入   │
│                                    → 删除待处理Acceptance    action │
│                                                              _logs  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                   席位总量调整 (adjustSeatCount)                       │
│                                                                      │
│  License.seats 变更 ──▶ boot() 事件监听 ──▶ adjustSeatCount()       │
│                                              │                       │
│                              ┌───────────────┼───────────────┐      │
│                              ▼                               ▼      │
│                     增加席位                          减少席位        │
│                     LicenseSeat::insert()        查找空闲席位        │
│                     (分块+事务)                   (assigned_to=null   │
│                                                    asset_id=null)   │
│                                              不足则拒绝操作         │
│                                              够则 delete()          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 8. 设计要点与权衡

### 8.1 一致性保障策略

| 策略 | 实现方式 | 涉及代码 |
|------|----------|----------|
| **查询时推导** | 可用余量不存储，通过 `availCount()`/`remaincount()` 实时查询 | License 模型多处 |
| **事务原子性** | API 操作使用 `DB::transaction` 包裹 save + log | LicenseSeatsController::update() |
| **模型事件** | `License::boot()` 监听 created/updating 自动调整席位数 | License.php#L142-L170 |
| **归属字段即状态** | `assigned_to`/`asset_id` 的 NULL/非NULL 直接决定席位是否可用 | 全链路 |

### 8.2 潜在风险点

1. **Web UI 非事务性**：[LicenseCheckoutController::store()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L79-L157) 和 [LicenseCheckinController::store()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckinController.php#L57-L120) 中的 `save()` 和 `event()` 不在事务中，如果 `save()` 成功但事件处理失败，可能出现席位已变更但通知/日志丢失的情况。相比之下，API 端的 `LicenseSeatsController::update()` 使用了 `DB::transaction`。

2. **批量操作的竞态**：[bulkCheckout()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L227-L281) 在循环中逐个 `freeSeat()` → `save()`，在并发场景下可能出现 `freeSeat()` 返回同一席位给不同请求的情况。没有使用数据库级锁（如 `LOCK FOR UPDATE`）。

3. **`remaincount()` 与 `availCount()` 的语义差异**：`remaincount()` 使用聚合计算（总数 - 已分配 - 不可再分配），而 `availCount()` 使用条件查询（NULL + false + 未删除），两者在极端数据不一致时可能产生不同结果。

### 8.3 `unreassignable_seat` 的作用

该字段是"不可再分配许可证"业务规则的实现关键：

- **场景**：企业购买了 100 席位许可证，标记为 `reassignable = false`。当员工 A 离职归还席位时，该席位不应再次分配给员工 B（许可证条款限制）。
- **实现**：回收时将 `unreassignable_seat = true`，使该席位从 `availCount()` 查询中永久排除。
- **效果**：`remaincount()` = 总数 - 已分配 - 不可再分配，准确反映真正可分配的余量。

---

## 9. 各操作入口一览

| 操作 | 路由/入口 | 控制器/命令 | 事务保护 |
|------|-----------|-------------|----------|
| 单席分配(Web) | `POST /licenses/{id}/checkout/{seatId?}` | LicenseCheckoutController::store() | ❌ 无 |
| 单席回收(Web) | `POST /licenses/{id}/checkin/{backTo?}` | LicenseCheckinController::store() | ❌ 无 |
| 席位更新(API) | `PUT /api/licenses/{id}/seats/{seatId}` | LicenseSeatsController::update() | ✅ DB::transaction |
| 批量分配 | `POST /licenses/{id}/bulkcheckout` | LicenseCheckoutController::bulkCheckout() | ❌ 无 |
| 批量回收 | `POST /licenses/{id}/bulkcheckin` | LicenseCheckinController::bulkCheckin() | ❌ 无 |
| CLI 批量分配 | `php artisan snipeit:checkout-to-all` | CheckoutLicenseToAllUsers | ❌ 无 |
| 套件分配 | Kit checkout flow | PredefinedKitCheckoutService::checkout() | ✅ DB::transaction |
| 席位总量调整 | License 创建/更新 | License::boot() → adjustSeatCount() | ✅ 分块事务 |
| 批量删除许可证 | `POST /licenses/bulk/delete` | BulkLicensesController::destroy() | ❌ 无（但有守卫） |

---

## 10. 指定席位与预定义套件的边界深度分析

本章深入分析两条特殊分配路径的边界判断逻辑，并对比 `unreassignable_seat` 在各路径中的阻碍作用差异。

### 10.1 指定席位分配路径的边界判断

**入口路由**：`POST /licenses/{licenseId}/checkout/{seatId}`  
**控制器方法**：[LicenseCheckoutController::store()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L79-L157)  
**辅助方法**：[findLicenseSeatToCheckout()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L159-L176)

#### 10.1.1 执行流程

```
用户传入 seatId（可选，不传则走自由分配）
  │
  ├─ 1. License::find($licenseId)        查找许可证
  ├─ 2. authorize('checkout', $license)  权限校验
  ├─ 3. availCount()->count() < 1 ?      ← 【许可证整体余量守卫】
  │     └─ 是 → 拒绝分配
  ├─ 4. isInactive() ?                   ← 过期/终止守卫
  │     └─ 是 → 拒绝分配
  ├─ 5. findLicenseSeatToCheckout($license, $seatId)
  │     ├─ seatId 传入：LicenseSeat::find($seatId)  → 校验是否存在、是否属于该 license
  │     └─ seatId 未传：$license->freeSeat()        → 走自由分配（过滤已分配 + 不可再分配）
  ├─ 6. checkoutToUser() / checkoutToAsset()
  │     ├─ 直接写入 assigned_to / asset_id
  │     └─ save()
  └─ 7. event(CheckoutableCheckedOut)
```

#### 10.1.2 条件判断详单（指定 seatId 场景）

当 `seatId` 被显式传入时，各检查项的实际执行情况：

| 检查项 | 是否检查 | 代码位置 | 说明 |
|--------|---------|----------|------|
| 席位是否存在 | ✅ 是 | `LicenseSeat::find($seatId)` | findLicenseSeatToCheckout 内 |
| 席位是否属于该许可证 | ✅ 是 | `$licenseSeat->license->is($license)` | findLicenseSeatToCheckout 内 |
| 许可证整体余量是否 > 0 | ✅ 是 | `availCount()->count() < 1`（第 88 行） | **store 入口守卫，在获取席位之前就执行** |
| 许可证是否过期/终止 | ✅ 是 | `isInactive()` | store 内 |
| **指定席位本身是否空闲** | ❌ **否** | — | 不检查 `assigned_to`/`asset_id` 是否为 NULL |
| **指定席位是否不可再分配** | ❌ **否** | — | 不检查 `unreassignable_seat` 字段 |
| 用户权限（checkout） | ✅ 是 | `authorize('checkout', $license)` | store 内 |

**关键发现 1：许可证整体余量守卫在席位获取之前执行。** `availCount()->count() < 1` 检查发生在 `findLicenseSeatToCheckout()` 之前（第 88 行 vs 第 97 行）。这意味着：即使你传入的指定席位本身是空闲的，只要许可证整体余量为 0，也会被拒绝。

**关键发现 2：传入已分配的指定席位时，归属会被静默覆盖。** 控制器不检查该席位当前的 `assigned_to`/`asset_id` 值，直接写入新值。原有归属关系被直接替换。

**关键发现 3：传入 `unreassignable_seat = true` 的冻结席位时，依然可以成功分配。** Web 端指定席位路径完全不检查该字段，与自由分配路径（`freeSeat()` 过滤了该字段）行为不一致。

### 10.2 指定席位分配时归属关系的修改机制

当指定 `seatId` 进行分配时，归属字段的修改方式如下：

#### 10.2.1 分配给用户（checkoutToUser）

**方法**：[checkoutToUser()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L198-L213)

```php
$licenseSeat->assigned_to = request('assigned_to');
$licenseSeat->save();
```

- 直接覆盖 `assigned_to` 字段。
- `asset_id` 字段保持不变（如果之前有值则保留）。
- 不做任何"是否已分配"的校验。

#### 10.2.2 分配给资产（checkoutToAsset）

**方法**：[checkoutToAsset()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L178-L196)

```php
$licenseSeat->asset_id = request('asset_id');

// Override asset's assigned user if available
if ($target->checkedOutToUser()) {
    $licenseSeat->assigned_to = $target->assigned_to;
}
$licenseSeat->save();
```

- 设置 `asset_id` 为目标资产 ID。
- **特殊覆盖逻辑**：如果目标资产当前已被分配给某个用户（`checkedOutToUser()`），则自动将 `assigned_to` 也设为该资产的当前持有人。
- 这意味着"分配给资产"可能同时写入两个归属字段。

#### 10.2.3 归属修改的行为对比

| 分配目标 | `assigned_to` 变化 | `asset_id` 变化 | 备注 |
|----------|-------------------|-----------------|------|
| 用户（自由分配） | 设为目标用户 ID | 保持不变 | 席位原先是空闲的 |
| 用户（指定席位） | **直接覆盖**为目标用户 ID | **保留原值** | 原归属用户被静默替换 |
| 资产（自由分配） | 可能设为资产持有人 | 设为目标资产 ID | 席位原先是空闲的 |
| 资产（指定席位） | 可能被**覆盖**为资产持有人 | **直接覆盖**为目标资产 ID | 原归属被静默替换 |

> ⚠️ **注意**：Web UI 的指定席位分配路径不检查 `unreassignable_seat`，也不检查席位是否已分配。这与 API 端的 `LicenseSeatsController::update()` 不同——API 端有明确的 `unreassignable_seat` 守卫。

### 10.3 API 端指定席位更新的边界判断

**入口**：`PUT /api/licenses/{licenseId}/seats/{seatId}`  
**控制器**：[LicenseSeatsController::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Api/LicenseSeatsController.php#L101-L248)

API 端的判断更严格：

```php
// 不可再分配的席位，禁止任何归属变更
if ($assignmentTouched && $licenseSeat->unreassignable_seat) {
    return response()->json(Helper::formatStandardApiResponse('error', null, 
        trans('admin/licenses/message.checkout.unavailable')));
}
```

| 检查项 | Web UI 指定席位 | API 指定席位更新 |
|--------|----------------|-----------------|
| 席位存在 | ✅ | ✅ |
| 席位属于该 license | ✅ | ✅ |
| 席位已被分配 | ❌ 不检查 | ✅ （通过 isDirty 判断是否为变更操作，但允许覆盖） |
| 不可再分配标记 | ❌ 不检查 | ✅ **有守卫，拒绝变更** |
| 事务保护 | ❌ | ✅ DB::transaction |
| FMCS 公司隔离 | ❌ （Web 端另有全局中间件） | ✅ 显式校验 |

### 10.4 预定义套件分配路径的边界判断

**服务类**：[PredefinedKitCheckoutService](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Services/PredefinedKitCheckoutService.php)

#### 10.4.1 席位选取逻辑

**[getLicenseSeatsToAdd()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Services/PredefinedKitCheckoutService.php#L103-L120)**

```php
$licenses = $kit->licenses()->with('freeSeats')->get();
foreach ($licenses as $license) {
    $quantity = $license->pivot->quantity;
    if ($quantity > count($license->freeSeats)) {
        $errors[] = trans('admin/kits/general.none_licenses', [...]);
    }
    for ($i = 0; $i < $quantity; $i++) {
        $seats_to_add[] = $license->freeSeats[$i];
    }
}
```

**关键要点**：
1. 使用 `with('freeSeats')` 预加载关联关系。
2. 使用 `count($license->freeSeats)` 判断余量是否充足。
3. 取前 N 个 `freeSeats` 作为待分配席位。
4. `freeSeats` 关系**不检查** `unreassignable_seat = false`。

#### 10.4.2 实际分配逻辑

**[saveToDb()](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Services/PredefinedKitCheckoutService.php#L146-L194)**

```php
foreach ($license_seats_to_add as $licenseSeat) {
    $licenseSeat->created_by = $admin->id;
    $licenseSeat->assigned_to = $user->id;
    if ($licenseSeat->save()) {
        event(new CheckoutableCheckedOut($licenseSeat, $user, $admin, $note));
    } else {
        $errors[] = 'Something went wrong saving a license seat';
    }
}
```

整个 saveToDb 被包裹在 `DB::transaction` 中，具有事务原子性。

#### 10.4.3 边界判断详单

| 检查项 | 预定义套件路径 |
|--------|---------------|
| 套件中 license 的 freeSeats 数量 ≥ pivot.quantity | ✅ 是 |
| 席位 `assigned_to IS NULL` | ✅ 是（通过 freeSeats 关系） |
| 席位 `asset_id IS NULL` | ✅ 是（通过 freeSeats 关系） |
| 席位 `deleted_at IS NULL` | ✅ 是（通过 freeSeats 关系） |
| **席位 `unreassignable_seat = false`** | ❌ **否（freeSeats 关系缺漏）** |
| 事务保护 | ✅ 是（DB::transaction） |
| 用户权限 | ✅ 是（AuthorizesRequests） |

**边界问题**：如果一个许可证的某些席位被标记为 `unreassignable_seat = true`（因回收不可再分配许可证而冻结），这些席位仍然会出现在 `freeSeats` 结果集中。预定义套件在做余量判断和席位选取时，会把这些"冻结"席位当作可用席位。

**实际影响**：
- 假设某许可证有 10 个席位，其中 5 个已分配，3 个被冻结（`unreassignable_seat = true`），2 个真正可用。
- 通过 `freeSeats` 计数会得到 5（3 冻结 + 2 可用）。
- 套件请求 4 个席位：`count($freeSeats) = 5 ≥ 4`，余量判断通过。
- 但实际分配时，取到的前 4 个席位可能包含被冻结的席位。
- 冻结席位被分配后，"是否可分配"的语义上出现不一致——该席位本应永久不可再分配。

### 10.5 `unreassignable_seat` 在各路径中的阻碍作用总览

`unreassignable_seat` 是席位级别的冻结标记，当一个 `reassignable = false` 的许可证席位被回收后，该字段会被设为 `true`。它在各操作路径中的实际阻碍效果如下：

| 操作路径 | `unreassignable_seat = true` 是否阻碍分配 | 守卫位置 | 守卫触发条件 |
|----------|-------------------------------------------|----------|-------------|
| Web UI 自由分配（`freeSeat()`） | ✅ **阻碍** | [License.php#L806-L817](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L806-L817) | SQL 查询条件：`where('unreassignable_seat', '=', false')` |
| Web UI 指定席位（`seatId`） | ❌ **不阻碍** | 无 | — |
| Web 批量分配（`freeSeat()`） | ✅ **阻碍** | [LicenseCheckoutController.php#L255](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Licenses/LicenseCheckoutController.php#L255) | 同上，通过 `freeSeat()` 过滤 |
| CLI 批量分配（`freeSeat()`） | ✅ **阻碍** | [CheckoutLicenseToAllUsers.php#L94](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Console/Commands/CheckoutLicenseToAllUsers.php#L94) | 同上，通过 `freeSeat()` 过滤 |
| API 席位更新 | ✅ **阻碍（但有例外）** | [LicenseSeatsController.php#L190-L192](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Http/Controllers/Api/LicenseSeatsController.php#L190-L192) | **仅当归属字段被变更时（`$assignmentTouched = true`）；只改 `notes` 不阻碍** |
| 预定义套件分配（`freeSeats`） | ❌ **不阻碍** | 无 | `freeSeats` 关系不包含该条件 |
| 席位总量减少（`adjustSeatCount`） | ❌ **不阻碍** | [License.php#L255-L269](file:///d:/fz/0601-1/solo-dogfeeding/code/42-snipe-it/app/Models/License.php#L255-L269) | 删除候选条件只看 `assigned_to IS NULL AND asset_id IS NULL` |

**重要修正**：之前对 API 端的描述不够精确。API 端的 `unreassignable_seat` 守卫**不是无条件触发**的：
- 如果请求只修改 `notes`（`$assignmentTouched = false`），即使席位已冻结也会成功更新。
- 只有当 `assigned_to` 或 `asset_id` 发生变更时，该守卫才会拒绝操作。

### 10.6 一致性差异汇总

1. **`freeSeat()` vs `freeSeats()` 语义不一致**：
   - `freeSeat()`（查询构造器方法）：包含完整可用性检查 `assigned_to=null AND asset_id=null AND unreassignable_seat=false AND deleted_at=null`。
   - `freeSeats()`（Eloquent 关系）：缺漏了 `unreassignable_seat = false` 条件。
   - 两者名称高度相近但行为不同，是预定义套件路径边界问题的根源。

2. **Web 端 vs API 端守卫不对称**：
   - Web 端指定席位路径：有**整体余量守卫**（`availCount() < 1` 拒绝），但无**席位级 `unreassignable_seat` 守卫**。
   - API 端更新路径：无**整体余量守卫**，但有**席位级 `unreassignable_seat` 守卫**（仅归属变更时）。
   - 这导致两种接口在特定边界条件下会给出相反的通过/拒绝结果。

3. **两者都允许静默覆盖既有归属**：
   - 无论是 Web 端指定席位还是 API 端更新，**都不检查席位原先是否已被分配**。
   - 直接用新值覆盖 `assigned_to` / `asset_id`，原有归属关系被静默替换。
   - 差异仅在于：Web 端通过 `event(CheckoutableCheckedOut)` 触发日志，API 端在事务内直接调用 `logCheckout()`。

4. **Web 端整体余量守卫的时序问题**：
   - Web 端 `availCount() < 1` 检查发生在获取席位**之前**（store 第 88 行 vs 第 97 行）。
   - 当用户指定的席位本身是空闲的，但许可证整体余量为 0 时（如所有其他席位都被分配或冻结），请求依然会被拒绝。
   - 这是一个略显保守的守卫——指定一个空闲席位本应是合法操作，却被整体余量拦住了。

5. **分配给资产的隐式双归属（仅 Web 端）**：
   - Web 端 `checkoutToAsset()` 中，如果目标资产当前有分配用户，会自动把 `assigned_to` 也设为该用户。
   - API 端无此逻辑，调用方需要自行决定是否同时设置两个字段。
   - 这意味着"分配给同一个资产"在两个接口上的最终数据状态可能不同。
