# Snipe-IT License Seat 管理机制深度分析

> **版本说明**：本分析基于代码精确核对，修正了之前关于并发行为和风险分类的偏差。

---

## 一、核心数据模型与计数恒等式

### 1. License 模型 (`app/Models/License.php`)

**核心字段定义**：
```php
protected $fillable = [
    'seats',              // 总席位数（由模型事件严格维护）
    'expiration_date',    // 过期日期
    'termination_date',   // 终止日期
    'reassignable',       // 是否可重新分配
];
```

**席位计数恒等式**（代码严格保证）：
```
license_seats 表记录数 = license.seats 字段值
已分配席位数 + 可用席位数 + 不可重新分配席位数 = 总席位数
```

**关键计数方法**：
| 方法 | 计算逻辑 | 代码位置 |
|------|---------|---------|
| `remaincount()` | `license_seats_count - assigned_seats_count - unreassignable_count` | License.php:740-748 |
| `freeSeat()` | `whereNull(assigned_to) + whereNull(asset_id) + unreassignable_seat=false` | License.php:806-817 |
| `availCount()` | 同上，仅计数 | License.php:631-638 |
| `assignedCount()` | `whereNotNull(assigned_to) OR whereNotNull(asset_id)` | License.php:686-694 |

### 2. LicenseSeat 模型 (`app/Models/LicenseSeat.php`)

**核心字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `license_id` | int | 关联的许可证ID |
| `assigned_to` | int/null | 分配给的用户ID |
| `asset_id` | int/null | 分配给的资产ID |
| `unreassignable_seat` | boolean | 不可重新分配标记 |

**数据库约束现状**：
- ❌ 无 `UNIQUE(license_id, assigned_to)` 索引
- ❌ 无 `UNIQUE(license_id, asset_id)` 索引
- ✅ 主键 `id` 唯一约束

---

## 二、Seat 分配主链路（精确流程）

### 2.1 UI 单席分配流程 (`LicenseCheckoutController::store()`)

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
   - 指定 seatId: LicenseSeat::find($seatId)
   - 未指定: $license->freeSeat() → 按 ID 升序取第一个可用
   [LicenseCheckoutController.php:159-176]
   ↓
5. 席位归属检查: $licenseSeat->license->is($license) → 错误返回
   [LicenseCheckoutController.php:171-173]
   ↓
6. 分配目标处理:
   - checkoutToUser(): 设置 assigned_to
   - checkoutToAsset(): 设置 asset_id，若资产已分配给用户则同步 assigned_to
   [LicenseCheckoutController.php:178-213]
   ↓
7. 席位保存 + 事件触发（无事务保护！）
   $licenseSeat->save();
   event(new CheckoutableCheckedOut($licenseSeat, $target, ...));
```

**关键事实**：UI 控制器分配 **没有数据库事务保护**，如果事件监听器失败，席位已分配但通知未发送。

### 2.2 API 席位分配流程 (`LicenseSeatsController::update()`)

**关键差异**：
- ✅ 使用数据库事务确保席位更新与日志记录的原子性
- ✅ 显式检查 `unreassignable_seat` 标记
- ❌ 不检查可用席位数（直接操作指定 seatId）

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

**唯一具有重复分配检查的场景**：
```php
// LicenseCheckoutController.php:252-256
foreach ($users as $user) {
    // 检查用户是否已拥有该许可证
    if ($user->licenses->where('id', '=', $licenseId)->count()) {
        continue; // 跳过已分配用户
    }
    // ... 分配逻辑
}
```

---

## 三、Seat 回收主链路（精确流程）

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
       $licenseSeat->unreassignable_seat = true;  // 标记为不可再分配
   }
   [LicenseCheckinController.php:101-103]
   ↓
6. 保存 + 事件触发（无事务保护！）
```

### 3.2 不可重新分配席位的生命周期

```
许可证 reassignable = false
    ↓
用户分配席位（正常）
    ↓
用户归还席位
    ↓
unreassignable_seat = true （标记）
    ↓
该席位永久不可再分配（freeSeat() 会排除它）
    ↓
可用席位数 = 总席位数 - 已分配 - 已标记不可重新分配
```

**代码依据**：
```php
// freeSeat() 排除不可重新分配的席位 [License.php:810]
->where('unreassignable_seat', '=', false)

// API 分配时显式检查 [LicenseSeatsController.php:190-192]
if ($assignmentTouched && $licenseSeat->unreassignable_seat) {
    return error('该席位不可重新分配');
}
```

---

## 四、过期通知与过期判定（精确时间点）

### 4.1 过期判定的精确生效时间

```php
// License.php:366-373
public function isExpired(): bool {
    $day = now()->startOfDay();  // 今天 00:00:00
    
    // 过期日期 <= 今天 00:00:00 → 已过期
    return $this->expiration_date && 
           $this->asDateTime($this->expiration_date)->startofDay()
               ->lessThanOrEqualTo($day);
}
```

**关键结论**：
> ✅ **过期日期当天 00:00:00 即判定为过期**
> 
> 例如：`expiration_date = 2026-05-20`
> - 2026-05-19 23:59:59 → 未过期
> - 2026-05-20 00:00:00 → 已过期
> - 2026-05-20 任何时间 → 已过期

### 4.2 即将过期查询范围

```php
// License.php:887-910
public function scopeExpiringLicenses($query, $days = 60, $includeExpired = false)
{
    return $query
        ->where(function ($query) use ($days) {
            // termination_date 为空 或 在 [今天, 今天+days] 范围内
            $query->whereNull('termination_date')
                  ->orWhereBetween('termination_date', [now(), now()->addDays($days)]);
        })
        ->where(function ($query) use ($days, $includeExpired) {
            $query->whereNotNull('expiration_date')
                ->where(function ($query) use ($days, $includeExpired) {
                    // 无终止日期时：expiration_date 在 [今天, 今天+days]
                    $query->whereNull('termination_date')
                          ->whereBetween('expiration_date', [now(), now()->addDays($days)])
                          ->when($includeExpired, function ($query) {
                              $query->orWhereDate('expiration_date', '<=', now());
                          });
                });
        });
}
```

### 4.3 过期许可证的分配限制

```php
// LicenseCheckoutController::create() 和 store() 均检查 [L44, L93]
if ($license->isInactive()) {
    return redirect()->route('licenses.index')
        ->with('error', trans('admin/licenses/message.checkout.license_is_inactive'));
}
```

> **事实**：过期/终止的许可证 **禁止新分配**，但 **不会自动回收** 已分配的席位。

### 4.4 通知发送流程

**触发方式**：Artisan 命令 `snipeit:expiring-alerts`（通常由 cron 每日执行）

```
1. 检查系统设置: alerts_enabled == 1 && alert_email 已配置
   ↓
2. 查询即将过期许可证
   License::query()->ExpiringLicenses($alert_interval, $includeExpired)
   ↓
3. 发送邮件通知（包含许可证列表、到期时间、剩余天数）
   Mail::to($recipients)->send(new ExpiringLicenseMail($licenses, $alert_interval))
```

---

## 五、重复分配与超额分配的真实边界

### 5.1 重复分配防护边界

| 场景 | 是否检查 | 检查逻辑 | 代码位置 |
|------|---------|---------|---------|
| **批量分配** | ✅ 检查 | `$user->licenses->where('id', '=', $licenseId)->count()` | LicenseCheckoutController.php:252 |
| **UI 单席分配** | ❌ 不检查 | 无 | - |
| **API 单席更新** | ❌ 不检查 | 无 | - |
| **席位状态过滤** | ✅ 隐式防护 | `freeSeat()` 只返回未分配席位 | License.php:806-817 |

**重要事实**：
> ⚠️ **单席分配允许同一用户被分配多个席位**
> 
> 这不是 bug，而是设计允许的行为（某些许可证允许多设备/多用户使用）。
> 只有批量分配场景为了"快速批量分配给不同用户"的使用场景，才做了重复检查。

### 5.2 超额分配防护边界

#### 5.2.1 核心防护机制（代码严格保证）

```php
// 模型事件保证 license_seats 记录数 = license.seats [License.php:142-170]
static::created(function ($license) {
    static::adjustSeatCount($license, 0, $license->seats);  // 创建时生成对应数量席位
});

static::updating(function ($license) {
    static::adjustSeatCount($license, $oldCount, $newCount);  // 更新时调整席位数量
});
```

**席位数量调整约束**：
```php
// License.php:226-236 减少席位时
$seatsAvailableForDelete = $license->licenseseats()
    ->whereNull('assigned_to')   // 只能删除未分配的席位
    ->whereNull('asset_id')
    ->limit($change);

if ($change > $seatsAvailableForDelete->count()) {
    return false;  // 已分配的席位不能被删除
}
```

#### 5.2.2 恒等式证明

由于：
1. `license_seats` 记录数 **始终等于** `license.seats`（模型事件保证）
2. `freeSeat()` 只返回 `assigned_to` 和 `asset_id` 都为 null 的记录
3. 每个 `license_seats` 记录只能被分配一次（单条记录）

因此：
> ✅ **已分配席位数 + 可用席位数 = 总席位数 恒成立**
> 
> ✅ **超额分配（已分配 > 总席位数）在数学上不可能发生**

### 5.3 其他防护边界

| 防护类型 | 代码位置 |
|---------|---------|
| 互斥分配（assigned_to 和 asset_id 不能同时设置） | LicenseSeatsController.php:108, 121 |
| 跨公司分配限制（FMCS） | LicenseSeatsController.php:155-157 |
| 席位归属检查（seat 必须属于指定 license） | LicenseCheckoutController.php:171-173 |
| 不可重新分配检查 | LicenseSeatsController.php:190-192 |
| 许可证删除限制（必须无已分配席位） | LicensesController.php:247 |

---

## 六、并发场景下的真实行为分析

### 6.1 并发分配：不是超额，而是覆盖

**场景**：两个请求同时分配最后一个可用席位

```
数据库初始状态:
  license_id=1, seats=1
  license_seat #5: assigned_to=null, asset_id=null

时序图:
T0: 请求A 到达
T1: 请求A availCount()->count() → 1 ✓
T2: 请求A freeSeat() → SELECT * FROM license_seats WHERE ... → 返回 seat #5
T3: 请求B 到达
T4: 请求B availCount()->count() → 1 ✓ (seat #5 尚未被更新)
T5: 请求B freeSeat() → SELECT * FROM license_seats WHERE ... → 也返回 seat #5 (无锁!)
T6: 请求A $licenseSeat->assigned_to = 101; save() → UPDATE 成功 ✓
T7: 请求B $licenseSeat->assigned_to = 102; save() → UPDATE 成功 ✓ (覆盖!)

最终状态:
  license_seat #5: assigned_to=102 (用户B)
  
结果:
  ✅ 总席位数仍为 1（无超额）
  ❌ 用户 A 的分配被覆盖丢失
  ❌ 操作日志只记录用户 B 的分配
```

**代码依据**：
- `freeSeat()` 使用普通 `SELECT`，**无 `lockForUpdate()` 行锁**
- 两个请求获取到同一个 seat ID，后保存的覆盖先保存的
- **这是覆盖分配，不是超额分配**

**风险等级**：高（并发场景下数据一致性问题）

### 6.2 并发批量分配：可能重复分配

**场景**：两个批量分配请求同时执行

```
T0: 用户X 未被分配许可证
T1: 批量请求A User::with('licenses')->get() → 用户X 无此许可证
T2: 批量请求B User::with('licenses')->get() → 用户X 无此许可证
T3: 请求A 分配席位给用户X ✓
T4: 请求B 分配席位给用户X ✓ (用户X 获得两个席位)

结果:
  ✅ 总席位数正确（用了两个席位）
  ⚠️ 用户X 被分配了两个席位（重复分配）
```

**代码依据**：
- `bulkCheckout()` 预加载用户许可证，但查询不锁定
- 遍历过程中不重新检查数据库状态

**风险等级**：中（依赖批量分配的使用频率）

### 6.3 席位数量调整并发：操作可能失败但数据一致

**场景**：分配请求与减少席位请求同时执行

```
T0: 许可证有 5 个席位，2 个已分配，3 个可用
T1: 请求A (减少到 3 席) SELECT 可用席位 → 3 个
T2: 请求B (分配) SELECT 可用席位 → 3 个 ✓
T3: 请求A DELETE 2 个可用席位 → 成功
T4: 请求B UPDATE 席位 #5 SET assigned_to=... → 成功

最终状态:
  总席位 3，已分配 3，可用 0
  
结果:
  ✅ 数据一致，无超额
  ✅ 两个请求都成功（如果删除的是其他可用席位）
  或
  ❌ 请求A 失败（如果尝试删除的席位包含 #5）
```

**风险等级**：低（最多操作失败，不会数据不一致）

### 6.4 过期临界分配：极小概率窗口

```
T0: 许可证 expiration_date = 今天（当天 00:00:00 已过期）
T1: 00:00:01 请求检查 isInactive() → true，拒绝分配

结果:
  ✅ 过期当天就被拦截，窗口极小
```

**风险等级**：极低

---

## 七、事务与锁机制现状

### 7.1 事务使用情况

| 操作 | 事务保护 | 代码位置 |
|------|---------|---------|
| UI 单席分配 | ❌ 无 | LicenseCheckoutController::store() |
| UI 单席回收 | ❌ 无 | LicenseCheckinController::store() |
| API 席位更新 | ✅ 有 | LicenseSeatsController.php:217 |
| 批量分配 | ❌ 无 | LicenseCheckoutController::bulkCheckout() |
| 批量回收 | ❌ 无 | LicenseCheckinController::bulkCheckin() |
| 新增席位 (批量插入) | ✅ 有 (按 1000 条分块) | License.php:263-271 |

### 7.2 锁机制现状

**全局搜索确认**：代码库中 **未使用** `lockForUpdate()` 或 `sharedLock()` 进行行级锁定。

**关键缺失点**：`freeSeat()` 查询没有行锁，这是并发覆盖问题的根源。

---

## 八、风险分类：哪些被覆盖，哪些会导致问题

| 风险类型 | 是否被覆盖 | 可能后果 | 风险等级 |
|---------|-----------|---------|---------|
| **超额分配（数量失衡）** | ✅ 完全覆盖 | 不可能发生 | 无 |
| **覆盖分配（并发）** | ❌ 未覆盖 | 前一个用户的分配被后一个覆盖，数据不一致 | 高 |
| **重复分配（单席）** | ❌ 未覆盖 | 同一用户被分配多个席位（设计允许，但可能非预期） | 中 |
| **UI 部分失败** | ❌ 未覆盖 | 席位已保存但事件/通知未发送 | 中 |
| **过期许可证分配** | ✅ 基本覆盖 | 过期当天即拦截，窗口极小 | 低 |
| **不可重新分配绕过** | ✅ 完全覆盖 | 分配时检查标记，不可能绕过 | 无 |
| **席位归属错误** | ✅ 完全覆盖 | UI 分配时检查 seat 归属 | 无 |

---

## 九、改进建议（按优先级）

### 🔴 高优先级：解决并发覆盖问题

**建议 1：为 `freeSeat()` 查询添加行锁**
```php
// License.php:806-817
public function freeSeat()
{
    return $this->licenseseats()
        ->whereNull('deleted_at')
        ->where('unreassignable_seat', '=', false)
        ->where(function ($query) {
            $query->whereNull('assigned_to')
                  ->whereNull('asset_id');
        })
        ->lockForUpdate()  // 🔴 添加行锁，防止并发覆盖
        ->orderBy('id', 'asc')
        ->first();
}
```

> **注意**：`lockForUpdate()` 需要在事务中调用才生效。

**建议 2：为 UI 控制器分配添加事务保护**
```php
// LicenseCheckoutController.php:101-120
DB::transaction(function () use ($licenseSeat, $request, $checkoutTarget) {
    if ($licenseSeat->save()) {
        event(new CheckoutableCheckedOut(...));
    }
});
```

### 🟡 中优先级：增强重复分配防护（可选，根据业务需求）

**建议 3：单席分配添加重复分配检查（如果业务不允许多席位）**
```php
// 在 checkoutToUser() 中添加
$existingCount = LicenseSeat::where('license_id', $licenseSeat->license_id)
    ->where('assigned_to', $target->id)
    ->whereNull('deleted_at')
    ->count();

if ($existingCount > 0) {
    return redirect()->back()->with('error', '该用户已拥有此许可证席位');
}
```

**建议 4：添加数据库唯一索引（如果业务不允许多席位）**
```sql
CREATE UNIQUE INDEX idx_license_seats_license_user 
ON license_seats (license_id, assigned_to) 
WHERE assigned_to IS NOT NULL AND deleted_at IS NULL;

CREATE UNIQUE INDEX idx_license_seats_license_asset 
ON license_seats (license_id, asset_id) 
WHERE asset_id IS NOT NULL AND deleted_at IS NULL;
```

### 🟢 低优先级：增强健壮性

**建议 5：保存时重新验证许可证状态**
```php
// 在 save() 前重新检查
$license->refresh();
if ($license->isInactive()) {
    throw new \Exception('许可证已过期或终止');
}
```

---

## 十、总结

### 核心事实确认

| 结论 | 代码依据 |
|------|---------|
| 过期日期当天即失效 | License.php:366-373 `startofDay()->lessThanOrEqualTo($day)` |
| 超额分配不可能发生 | License.php:142-170 模型事件严格维护席位记录数 |
| 并发会导致覆盖分配 | License.php:806-817 `freeSeat()` 无行锁 |
| 单席分配可能重复 | LicenseCheckoutController 无重复检查 |
| UI 控制器无事务 | 代码检查确认 |
| 数量不会失衡 | `license_seats` 记录数 = `license.seats` 恒成立 |

### 风险矩阵

| 风险 | 现状 | 后果 | 优先级 |
|------|------|------|-------|
| 并发覆盖分配 | 无行锁 | 数据不一致，分配记录丢失 | 🔴 高 |
| UI 无事务 | 部分操作无事务 | 部分失败导致不一致 | 🔴 高 |
| 单席重复分配 | 无检查 | 同一用户多席位（设计允许但可能非预期） | 🟡 中 |
| 批量并发重复 | 无锁 | 同一用户多席位 | 🟡 中 |
| 过期临界分配 | 请求开始时检查 | 极小概率窗口 | 🟢 低 |

> **最终结论**：当前实现的数量一致性是有保障的（不会超额），但并发场景下的数据一致性（覆盖分配）和业务一致性（重复分配）存在风险，需要通过添加行锁和事务来解决。
