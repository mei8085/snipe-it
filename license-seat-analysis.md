# Snipe-IT License Seat 管理机制深度分析

## 一、核心数据模型与关系

### 1. License 模型 (`app/Models/License.php`)

**核心字段定义**：
```php
protected $fillable = [
    'seats',              // 总席位数
    'expiration_date',    // 过期日期
    'termination_date',   // 终止日期
    'reassignable',       // 是否可重新分配
    'name', 'category_id', 'company_id', ...
];
```

**关键计数方法**：
| 方法 | 用途 | 计算逻辑 |
|------|------|---------|
| `remaincount()` | 可用席位数 | `totalSeats - assignedSeats - unreassignableSeats` |
| `freeSeat()` | 获取第一个可用席位 | `whereNull(assigned_to) + whereNull(asset_id) + unreassignable_seat=false` |
| `availCount()` | 可用席位计数 | 同上，仅计数 |
| `assignedCount()` | 已分配席位计数 | `whereNotNull(assigned_to) OR whereNotNull(asset_id)` |

### 2. LicenseSeat 模型 (`app/Models/LicenseSeat.php`)

**核心字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `license_id` | int | 关联的许可证ID |
| `assigned_to` | int|null | 分配给的用户ID |
| `asset_id` | int|null | 分配给的资产ID |
| `unreassignable_seat` | boolean | 是否标记为不可重新分配 |
| `notes` | text|null | 备注 |

**关键约束**：
- `assigned_to` 和 `asset_id` 互斥（通过 API 验证 `prohibits` 规则实现）
- 无数据库层面的唯一索引约束重复分配

---

## 二、Seat 分配主链路

### 2.1 UI 单席分配流程 (`LicenseCheckoutController::store()`)

**完整执行链路**：
```
1. 权限检查: authorize('checkout', $license)
   ↓
2. 可用席位检查: $license->availCount()->count() < 1 → 错误返回
   [LicenseCheckoutController.php:88]
   ↓
3. 许可证状态检查: $license->isInactive() → 错误返回
   [LicenseCheckoutController.php:93]
   ↓
4. 席位定位: findLicenseSeatToCheckout($license, $seatId)
   - 指定 seatId 时直接查找
   - 未指定时调用 $license->freeSeat() 获取第一个可用
   [LicenseCheckoutController.php:159-176]
   ↓
5. 分配目标处理:
   - checkoutToUser(): 设置 assigned_to
   - checkoutToAsset(): 设置 asset_id，若资产已分配给用户则同步 assigned_to
   [LicenseCheckoutController.php:178-213]
   ↓
6. 席位保存 + 事件触发
   event(new CheckoutableCheckedOut($licenseSeat, $target, ...))
   ↓
7. 监听器处理 (CheckoutableListener):
   - 发送邮件通知 (如配置)
   - 发送 Webhook 通知 (如配置)
   - 创建 CheckoutAcceptance (如需用户确认)
```

### 2.2 API 席位分配流程 (`LicenseSeatsController::update()`)

**关键差异**：
- API 层面使用 **数据库事务** 确保席位更新与日志记录的原子性
- 支持直接更新 `assigned_to` 或 `asset_id` 字段
- 显式检查 `unreassignable_seat` 标记

```php
// app/Http/Controllers/Api/LicenseSeatsController.php:217-241
$updated = DB::transaction(function () use ($licenseSeat, ...) {
    if (! $licenseSeat->save()) return false;
    
    if ($assignmentTouched) {
        if ($is_checkin) {
            // 回收逻辑
        } else {
            $licenseSeat->logCheckout($request->input('notes'), $target);
        }
    }
    return true;
});
```

### 2.3 批量分配流程 (`LicenseCheckoutController::bulkCheckout()`)

**执行流程**：
```
1. 获取所有开启 autoassign_licenses 的用户
   User::where('autoassign_licenses', '=', 1)->with('licenses')->get()
   ↓
2. 遍历用户，跳过已拥有该许可证的用户
   if ($user->licenses->where('id', '=', $licenseId)->count()) continue;
   [LicenseCheckoutController.php:252-256]
   ↓
3. 获取可用席位并分配
   $licenseSeat = $license->freeSeat();
   $licenseSeat->assigned_to = $user->id;
   $licenseSeat->save();
   ↓
4. 可用席位为 0 时提前终止
   if ($avail_count == 0) return;
```

---

## 三、Seat 回收主链路

### 3.1 UI 单席回收流程 (`LicenseCheckinController::store()`)

```
1. 验证席位存在性
   ↓
2. 验证席位确实已分配
   if (is_null($licenseSeat->assigned_to) && is_null($licenseSeat->asset_id)) → 错误
   [LicenseCheckinController.php:68-70]
   ↓
3. 权限检查: authorize('checkout', $license)
   ↓
4. 清空分配字段
   $licenseSeat->assigned_to = null;
   $licenseSeat->asset_id = null;
   ↓
5. 不可重新分配许可证处理
   if (! $licenseSeat->license->reassignable) {
       $licenseSeat->unreassignable_seat = true;
   }
   [LicenseCheckinController.php:101-103]
   ↓
6. 保存 + 事件触发
   event(new CheckoutableCheckedIn($licenseSeat, $return_to, ...))
```

### 3.2 API 席位回收流程

```php
// app/Http/Controllers/Api/LicenseSeatsController.php:224-233
if ($is_checkin) {
    if (! $licenseSeat->license->reassignable) {
        $licenseSeat->unreassignable_seat = true;
        if (! $licenseSeat->save()) return false;
    }
    $licenseSeat->logCheckin($target, $licenseSeat->notes);
}
```

### 3.3 批量回收流程 (`LicenseCheckinController::bulkCheckin()`)

```
1. 分别查询分配给用户和资产的席位
   ↓
2. 遍历回收，同样应用 unreassignable_seat 标记
   ↓
3. 记录操作日志
```

---

## 四、过期通知主链路

### 4.1 过期状态判断 (`app/Models/License.php:355-382`)

```php
public function isInactive(): bool {
    return $this->isExpired() || $this->isTerminated();
}

public function isExpired(): bool {
    return $this->expiration_date && 
           $this->asDateTime($this->expiration_date)->startofDay() <= now();
}

public function isTerminated(): bool {
    return $this->termination_date && 
           $this->asDateTime($this->termination_date)->startofDay() <= now();
}
```

### 4.2 即将过期查询范围

```php
// app/Models/License.php:887-910
public function scopeExpiringLicenses($query, $days = 60, $includeExpired = false)
{
    return $query
        ->where(function ($query) use ($days) {
            // termination_date 为空或在 N 天内到期
            $query->whereNull('termination_date')
                  ->orWhereBetween('termination_date', [now(), now()->addDays($days)]);
        })
        ->where(function ($query) use ($days, $includeExpired) {
            // expiration_date 不为空
            $query->whereNotNull('expiration_date')
                ->where(function ($query) use ($days, $includeExpired) {
                    // 无终止日期时，检查过期日期
                    $query->whereNull('termination_date')
                          ->whereBetween('expiration_date', [now(), now()->addDays($days)])
                          ->when($includeExpired, function ($query) {
                              $query->orwhereDate('expiration_date', '<=', now());
                          });
                })
                // 或终止日期在 N 天内
                ->orWhere(function ($query) use ($days) {
                    $query->whereBetween('termination_date', [now(), now()->addDays($days)]);
                });
        });
}
```

### 4.3 通知发送流程 (`SendExpirationAlerts`)

**执行时机**：通过 Artisan 命令 `snipeit:expiring-alerts` 触发，通常由 cron 每日执行

```
1. 检查系统设置: alerts_enabled == 1 且 alert_email 已配置
   ↓
2. 查询即将过期许可证
   License::query()->ExpiringLicenses($alert_interval, $includeExpired)
   ↓
3. 发送邮件通知
   Mail::to($recipients)->send(new ExpiringLicenseMail($licenses, $alert_interval))
   ↓
4. 命令行输出结果表格
```

### 4.4 过期许可证的分配限制

```php
// LicenseCheckoutController::create() 和 store() 均检查
if ($license->isInactive()) {
    return redirect()->route('licenses.index')
        ->with('error', trans('admin/licenses/message.checkout.license_is_inactive'));
}
```

> **关键点**：过期许可证 **禁止新分配**，但 **不会自动回收** 已分配的席位。

---

## 五、重复分配与超额分配防护边界

### 5.1 重复分配防护

| 场景 | 防护机制 | 代码位置 | 防护强度 |
|------|---------|---------|---------|
| **批量分配** | 检查 `$user->licenses->where('id', '=', $licenseId)->count()` | LicenseCheckoutController.php:252 | ✅ 强防护 |
| **UI 单席分配** | **无显式检查** | - | ❌ 无防护 |
| **API 单席更新** | **无显式检查** | - | ❌ 无防护 |
| **席位状态** | `freeSeat()` 排除已分配席位 | License.php:806-817 | ✅ 强防护 |
| **不可重新分配** | `unreassignable_seat = true` 时拒绝分配 | LicenseSeatsController.php:190-192 | ✅ 强防护 |

**重复分配风险说明**：
- 单席分配（UI/API）未检查目标用户是否已拥有同一许可证的其他席位
- 理论上同一用户可被分配多个席位
- 批量分配是唯一具有重复分配检查的场景

### 5.2 超额分配防护

#### 5.2.1 分配时检查

```php
// LicenseCheckoutController.php:88
if ($license->availCount()->count() < 1) {
    return redirect()->route('licenses.index')
        ->with('error', trans('admin/licenses/message.checkout.not_enough_seats'));
}
```

#### 5.2.2 席位数量调整限制

```php
// License.php:226-236 减少席位时
$seatsAvailableForDelete = $license->licenseseats()
    ->whereNull('assigned_to')
    ->whereNull('asset_id')
    ->limit($change);

if ($change > $seatsAvailableForDelete->count()) {
    Session::flash('error', trans('admin/licenses/message.assoc_users'));
    return false;
}
```

> **关键点**：减少总席位数时，**只能删除未分配的席位**，已分配的席位不能被删除。

#### 5.2.3 许可证删除限制

```php
// app/Http/Controllers/Api/LicensesController.php:247
if ($license->assigned_seats_count == 0) {
    // 允许删除
} else {
    return response()->json(['error' => trans('admin/licenses/message.assoc_users')]);
}
```

#### 5.2.4 跨公司分配限制 (FMCS)

```php
// LicenseSeatsController.php:155-157
if ((Setting::getSettings()->full_multiple_companies_support == '1') && 
    ($license->company_id !== $targetUser->company_id)) {
    return response()->json(Helper::formatStandardApiResponse('error', null, 
        trans('general.error_user_company')));
}
```

### 5.3 互斥分配约束

```php
// LicenseSeatsController.php:108, 121
'assigned_to' => ['prohibits:asset_id', ...],
'asset_id' => ['prohibits:assigned_to', ...],
```

> `assigned_to` 和 `asset_id` 不能同时设置（API 层面验证）。

---

## 六、并发下可能失效的场景与触发条件

### 6.1 超额分配竞态条件

**场景描述**：两个请求同时分配最后一个可用席位

```
时序图:
T0: 许可证有 1 个可用席位
T1: 请求A 执行 availCount()->count() → 返回 1
T2: 请求B 执行 availCount()->count() → 返回 1
T3: 请求A 执行 freeSeat() → 获取席位 #5
T4: 请求B 执行 freeSeat() → 也获取席位 #5 (数据库读取无锁)
T5: 请求A 保存席位 #5 assigned_to = 用户A ✓
T6: 请求B 保存席位 #5 assigned_to = 用户B ✓ (覆盖!)

结果: 席位 #5 被分配给用户B，用户A的分配丢失
```

**触发条件**：
1. 可用席位数量 ≤ 并发请求数
2. 请求时间差小于数据库事务执行时间
3. UI 控制器分配（无事务保护）或 API 分配（事务但无行锁）

**代码依据**：
- UI 控制器 `LicenseCheckoutController::store()` **无数据库事务**
- API 控制器 `LicenseSeatsController::update()` 有事务，但 **无行锁 (`lockForUpdate()`)**
- `freeSeat()` 查询使用普通 `SELECT`，无共享锁或排他锁

**风险等级**：高

### 6.2 批量分配重复分配

**场景描述**：两个批量分配请求同时执行

```
T0: 用户X 未被分配许可证
T1: 批量请求A 加载用户列表 (with('licenses')) → 用户X 无此许可证
T2: 批量请求B 加载用户列表 (with('licenses')) → 用户X 无此许可证
T3: 请求A 分配席位给用户X ✓
T4: 请求B 分配席位给用户X ✓ (重复分配!)
```

**触发条件**：
1. 两个批量分配请求同时执行
2. 用户列表加载在对方分配完成之前

**代码依据**：
- `bulkCheckout()` 预加载用户许可证关系，但该查询不锁定
- 遍历过程中无重新检查机制

**风险等级**：中

### 6.3 席位数量调整竞态

**场景描述**：分配请求与减少席位请求同时执行

```
T0: 许可证有 5 个席位，2 个已分配，3 个可用
T1: 请求A (减少席位到 3) 计算可删除数 = 3 (可用席位)
T2: 请求B (分配席位) 检查可用 = 3 ✓
T3: 请求A 删除 2 个可用席位 → 总席位 3，已分配 2，可用 1
T4: 请求B 分配席位 → 成功，总席位 3，已分配 3，可用 0
结果: 操作均成功，但如果请求A尝试删除 3 个席位，则会失败
```

**代码依据**：
- `adjustSeatCount()` 删除席位时使用 `limit($change)`，但查询不锁定
- 删除和分配操作可能交叉执行

**风险等级**：中

### 6.4 过期许可证分配窗口

**场景描述**：许可证即将过期时的分配

```
T0: 许可证将于今天 23:59:59 过期
T1: 09:00 请求检查 isInactive() → false (未过期)
T2: 10:00 用户开始填写分配表单
T3: 23:59:58 提交分配请求 → 检查 isInactive() → false
T4: 23:59:59 许可证过期
T5: 00:00:01 席位分配保存完成 ✓
结果: 席位在许可证过期后 1 秒分配成功
```

**代码依据**：
- `isInactive()` 检查在请求开始时执行
- 保存操作不重新检查许可证状态
- 无数据库层面的过期约束

**风险等级**：低（时间窗口极小）

---

## 七、事务与锁机制分析

### 7.1 当前事务使用情况

| 操作 | 是否使用事务 | 代码位置 |
|------|-------------|---------|
| UI 单席分配 | ❌ 无 | LicenseCheckoutController::store() |
| UI 单席回收 | ❌ 无 | LicenseCheckinController::store() |
| API 席位更新 | ✅ 有 | LicenseSeatsController.php:217 |
| 批量分配 | ❌ 无 | LicenseCheckoutController::bulkCheckout() |
| 批量回收 | ❌ 无 | LicenseCheckinController::bulkCheckin() |
| 席位数量调整 | ✅ 有 (删除时) | License.php:265-270 |

### 7.2 锁机制缺失

**全局搜索确认**：代码库中 **未使用** `lockForUpdate()` 或 `sharedLock()` 进行行级锁定。

### 7.3 数据库约束缺失

**全局搜索确认**：`license_seats` 表无唯一索引约束 `(license_id, assigned_to)` 或 `(license_id, asset_id)`，允许同一用户被多次分配。

---

## 八、改进建议

### 8.1 增强并发安全性

**建议 1：API 层面添加行锁**
```php
// 在 freeSeat() 查询中添加锁
public function freeSeat()
{
    return $this->licenseseats()
        ->whereNull('deleted_at')
        ->where('unreassignable_seat', '=', false)
        ->where(function ($query) {
            $query->whereNull('assigned_to')
                  ->whereNull('asset_id');
        })
        ->lockForUpdate()  // 添加行锁
        ->orderBy('id', 'asc')
        ->first();
}
```

**建议 2：UI 控制器添加事务保护**
```php
DB::transaction(function () use ($license, $licenseSeat, $target) {
    $licenseSeat->assigned_to = $target->id;
    $licenseSeat->save();
    event(new CheckoutableCheckedOut(...));
});
```

### 8.2 增强重复分配防护

**建议 3：单席分配添加重复检查**
```php
// 在 LicenseCheckoutController::checkoutToUser() 中添加
$existingSeat = LicenseSeat::where('license_id', $licenseSeat->license_id)
    ->where('assigned_to', $target->id)
    ->whereNull('deleted_at')
    ->first();

if ($existingSeat) {
    return redirect()->back()->with('error', '该用户已拥有此许可证席位');
}
```

**建议 4：添加数据库唯一索引**
```sql
CREATE UNIQUE INDEX idx_license_seats_license_user 
ON license_seats (license_id, assigned_to) 
WHERE assigned_to IS NOT NULL AND deleted_at IS NULL;

CREATE UNIQUE INDEX idx_license_seats_license_asset 
ON license_seats (license_id, asset_id) 
WHERE asset_id IS NOT NULL AND deleted_at IS NULL;
```

### 8.3 增强过期处理

**建议 5：保存时重新验证许可证状态**
```php
// 在保存前重新检查
$license->refresh();
if ($license->isInactive()) {
    throw new \Exception('许可证已过期或终止');
}
```

---

## 九、总结

| 维度 | 现状 | 风险等级 |
|------|------|---------|
| **超额分配防护** | 分配前检查可用席位，但查询无锁 | 高（并发场景） |
| **重复分配防护** | 仅批量分配有检查，单席分配无 | 中 |
| **事务保护** | API 有事务，UI 无 | 中 |
| **过期通知** | 定时任务扫描，邮件通知 | 低 |
| **不可重新分配处理** | 回收时标记席位，分配时检查 | 低 |
| **并发安全性** | 无行锁，存在竞态条件 | 高 |

**核心风险点**：
1. 并发分配可能导致超额或覆盖
2. 单席分配可能重复分配给同一用户
3. UI 控制器无事务保护，部分失败可能导致数据不一致

**建议优先级**：
1. ✅ 高优先级：为 `freeSeat()` 查询添加行锁
2. ✅ 高优先级：为 UI 控制器分配/回收添加事务保护
3. ⚠️ 中优先级：单席分配添加重复分配检查
4. ⚠️ 中优先级：添加数据库唯一索引
5. 📝 低优先级：保存时重新验证许可证状态
