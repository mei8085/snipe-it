# 维护记录、成本字段、附件文件与资产生命周期关联链路分析

## 一、整体架构概览

维护记录系统涉及三个核心链路的协作：
1. **维护单创建链路**：表单提交 → 数据验证 → 模型保存 → 观察者事件触发
2. **附件落盘链路**：文件上传 → 存储目录 → 文件命名 → 操作日志记录
3. **资产状态联动链路**：维护单创建时快照 → 维护完成时记录 → 资产状态只读关联

---

## 二、维护单创建流程详解

### 2.1 入口控制器

**Web 控制器**：`app/Http/Controllers/MaintenancesController.php`

```php
// 单资产或批量资产创建
public function store(ImageUploadRequest $request): RedirectResponse
{
    $assets = Asset::whereIn('id', $request->input('selected_assets'))->get();
    
    foreach ($assets as $asset) {
        $maintenance = new Maintenance;
        // 字段赋值...
        $maintenance->cost = $request->input('cost');
        $request->handleImages($maintenance);  // 处理图片上传
        
        if (! $maintenance->save()) {
            return redirect()->back()->withInput()->withErrors($maintenance->getErrors());
        }
        
        $this->storeUploadedFiles($request, $maintenance);  // 处理附件上传
    }
}
```

**API 控制器**：`app/Http/Controllers/Api/MaintenancesController.php`

```php
public function store(ImageUploadRequest $request): JsonResponse|array
{
    $isBulk = $request->has('asset_ids');
    $assetIds = $isBulk ? $request->input('asset_ids') : [$request->input('asset_id')];
    
    foreach ($assetIds as $assetId) {
        $maintenance = new Maintenance;
        $maintenance->fill($request->except(['asset_id', 'asset_ids']));
        $maintenance->asset_id = $assetId;
        $maintenance->created_by = auth()->id();
        $request->handleImages($maintenance);
        $maintenance->save();
    }
}
```

### 2.1.1 Web 与 API 入口差异对比

维护单创建有两条独立的入口路径，功能差异显著：

| 功能点 | Web 入口 <br>`MaintenancesController@store` | API 入口 <br>`Api/MaintenancesController@store` |
|--------|---------------------------------------------|-------------------------------------------------|
| 路由 | `POST /maintenances` | `POST /api/v1/maintenances` |
| 表单请求类 | `ImageUploadRequest` | `ImageUploadRequest` |
| 成本字段处理 | ✅ `setCostAttribute` Mutator | ✅ `setCostAttribute` Mutator |
| 图片上传 (`handleImages`) | ✅ 支持，存 `public` 目录 | ✅ 支持，存 `public` 目录 |
| 附件验证 (`validateUploadedFiles`) | ✅ 调用 | ❌ **不调用** |
| 附件落盘 (`storeUploadedFiles`) | ✅ 调用 | ❌ **不调用** |
| `uploaded` 日志写入 | ✅ 写入 | ❌ **不写入** |
| 批量创建参数 | `selected_assets[]` | `asset_ids[]` |
| 响应格式 | `RedirectResponse` | `JsonResponse` |

> **关键结论**：
> - **API 入口创建维护单时不处理附件上传**，只处理图片上传和主记录保存
> - 附件必须通过专门的上传接口单独提交（见下文 2.1.2）
> - 两条路径都会触发 `MaintenanceObserver` 的 `creating` / `created` 事件

### 2.1.2 附件单独上传入口（Web + API）

维护单创建完成后，可通过以下两个独立接口上传附件：

| 特性 | Web 上传入口 <br>`UploadedFilesController@store` | API 上传入口 <br>`Api/UploadedFilesController@store` |
|------|-------------------------------------------------|-----------------------------------------------------|
| 路由 | `POST /files/maintenances/{id}` | `POST /api/v1/files/maintenances/{id}` |
| 请求类 | `UploadFileRequest` | `UploadFileRequest` |
| `handleFile` 调用 | ✅ | ✅ |
| `logUpload` 调用 | ✅ | ✅ |
| 事务处理 | ❌ 无 | ❌ 无 |
| 响应格式 | `RedirectResponse` | `JsonResponse` |

这两个接口是唯一会执行**附件落盘 + `uploaded` 日志写入**完整流程的路径。

### 2.2 成本字段处理逻辑

**模型**：`app/Models/Maintenance.php:151-158`

```php
public function setCostAttribute($value)
{
    $value = Helper::ParseCurrency($value);
    if ($value == 0) {
        $value = null;
    }
    $this->attributes['cost'] = $value;
}
```

**货币解析辅助函数**：`app/Helpers/Helper.php:579-592`

```php
public static function ParseCurrency($currencyString)
{
    // 移除货币符号
    $without_currency = str_replace(Setting::getSettings()->default_currency, '', $currencyString);
    
    // 根据千位分隔符格式处理
    if (Setting::getSettings()->digit_separator == '1.234,56') {
        // EU 格式：1.234,56
        $without_thousands = str_replace('.', '', $without_currency);
        $corrected_decimal = str_replace(',', '.', $without_thousands);
    } else {
        // US 格式：1,234.56
        $without_thousands = str_replace(',', '', $without_currency);
        $corrected_decimal = $without_thousands;
    }
    
    return floatval($corrected_decimal);
}
```

**验证规则**：`app/Models/Maintenance.php:48`
```php
'cost' => 'numeric|nullable|gte:0|max:99999999999999999.99',
```

> **关键点**：
> - 成本字段支持本地化货币格式输入（EU/US 格式）
> - 0 值会被转换为 null 存储
> - 最大值约为 10^17，支持大规模资产管理

---

## 三、附件落盘流程详解

### 3.1 存储路径配置

**基类控制器**：`app/Http/Controllers/Controller.php:66-98`

```php
public static $map_storage_path = [
    'maintenances' => 'private_uploads/maintenances/',
    // ... 其他类型
];

public static $map_file_prefix = [
    'maintenances' => 'maintenance',
    // ... 其他类型
];
```

### 3.2 附件上传处理

**Web 控制器内联方法**：`app/Http/Controllers/MaintenancesController.php:195-223`

```php
private function storeUploadedFiles(ImageUploadRequest $request, Maintenance $maintenance): void
{
    if (! $request->hasFile('file')) {
        return;
    }

    $objectType = 'maintenances';
    $storagePath = self::$map_storage_path[$objectType];
    
    // 确保目录存在
    if (! Storage::exists($storagePath)) {
        Storage::makeDirectory($storagePath, 775);
    }

    $uploadFileRequest = app(UploadFileRequest::class);
    
    foreach ((array) $request->file('file') as $file) {
        $fileName = $uploadFileRequest->handleFile(
            $storagePath,
            self::$map_file_prefix[$objectType].'-'.$maintenance->id,
            $file
        );
        // 记录上传日志
        $maintenance->logUpload($fileName, $request->input('file_notes'));
    }
}
```

**文件处理核心方法**：`app/Http/Requests/UploadFileRequest.php:43-63`

```php
public function handleFile(string $dirname, string $name_prefix, $file): string
{
    $extension = $file->getClientOriginalExtension();
    // 文件名格式：{prefix}-{id}-{random}-{original_name}.{ext}
    $file_name = $name_prefix.'-'.str_random(8).'-'.str_slug(basename($file->getClientOriginalName(), '.'.$extension)).'.'.$file->guessExtension();

    // SVG 文件特殊处理：消毒
    if ($file->getMimeType() === 'image/svg+xml') {
        $uploaded_file = $this->handleSVG($file);
    } else {
        $uploaded_file = file_get_contents($file);
    }

    Storage::put($dirname.$file_name, $uploaded_file);
    
    return $file_name;
}
```

### 3.3 附件上传日志记录

**Loggable Trait**：`app/Models/Traits/Loggable.php:521-541`

```php
public function logUpload($filename, $note)
{
    $log = new Actionlog;
    $log->item_type = static::class;
    $log->item_id = $this->id;
    $log->created_by = auth()->id();
    $log->note = $note;
    $log->company_id = $this->resolveLoggableCompanyId();
    $log->filename = $filename;
    $log->logaction('uploaded');  // ActionType::Uploaded

    return $log;
}
```

### 3.4 附件查询关联

**HasUploads Trait**：`app/Models/Traits/HasUploads.php:9-22`

```php
public function uploads()
{
    return $this->hasMany(Actionlog::class, 'item_id')
        ->where('item_type', self::class)
        ->where('action_type', '=', 'uploaded')
        ->whereNotNull('filename')
        // 排除已删除的文件
        ->whereNotIn('filename', function ($query) {
            $query->select('filename')
                ->from('action_logs')
                ->where('item_type', '=', self::class)
                ->where('action_type', '=', 'upload deleted')
                ->where('item_id', $this->id);
        });
}
```

> **关键点**：
> - 附件存储在 `private_uploads/maintenances/` 目录（非公开）
> - 文件名格式：`maintenance-{id}-{random8}-{original_slug}.{ext}`
> - 附件与维护记录通过 `action_logs` 表关联，而非直接外键
> - 支持批量上传，每个文件独立记录日志
> - SVG 文件会经过安全消毒处理

### 3.5 上传写盘失败异常链路分析

#### 3.5.1 问题代码位置

**UploadFileRequest**：`app/Http/Requests/UploadFileRequest.php:56-62`

```php
public function handleFile(string $dirname, string $name_prefix, $file): string
{
    // ... 文件名生成、SVG 消毒 ...

    try {
        Storage::put($dirname.$file_name, $uploaded_file);
    } catch (\Exception $e) {
        Log::debug($e);  // ⚠️  仅记录 debug 级别日志，不抛出异常
    }

    return $file_name;  // ⚠️  无论写盘成功失败，都返回文件名
}
```

#### 3.5.2 完整异常链路时序

```
用户提交附件上传 (Web 或 API)
    ↓
UploadedFilesController::store()
    ├─ 权限检查 ✅
    ├─ 检查并创建存储目录 ✅
    ├─ 循环处理文件
    │   ├─ handleFile($storagePath, 'maintenance-'.$id, $file)
    │   │   ├─ 生成文件名: maintenance-123-abc12345-invoice.pdf
    │   │   ├─ 读取文件内容: file_get_contents($file) ✅
    │   │   ├─ Storage::put($path, $content)
    │   │   │   └─ 写盘失败触发异常 (权限/磁盘满/S3断开等)
    │   │   ├─ catch (\Exception $e) { Log::debug($e); }
    │   │   │   └─ 异常被静默吞掉，调用方无感知
    │   │   └─ return 'maintenance-123-abc12345-invoice.pdf'
    │   │       └─ 返回文件名，假装一切正常 ⚠️
    │   ├─ $files[] = $file_name  // 收集文件名
    │   └─ $object->logUpload($file_name, $notes)
    │       ├─ new Actionlog()
    │       ├─ item_type = 'App\Models\Maintenance'
    │       ├─ item_id = 123
    │       ├─ action_type = 'uploaded'
    │       ├─ filename = 'maintenance-123-abc12345-invoice.pdf'
    │       └─ logaction() -> $this->save()  ✅ 数据库写入成功
    └─ 返回成功响应 (success + 上传文件列表)
```

#### 3.5.3 触发条件

| 场景 | 具体原因 |
|------|----------|
| **存储层异常** | 目录权限不足（`private_uploads/` 无写入权限）<br>磁盘空间耗尽<br>远程存储（S3/MinIO）连接失败或认证失效<br>网络中断（云存储场景）<br>文件系统只读挂载 |
| **文件内容异常** | `file_get_contents()` 成功但 `Storage::put()` 失败<br>SVG 消毒后内容为空但仍尝试写入 |
| **并发竞争** | 目录被其他进程删除<br>磁盘配额瞬间超限 |

#### 3.5.4 数据不一致表现

| 数据层面 | 状态 |
|---------|------|
| `action_logs` 表记录 | ✅ 存在，`action_type='uploaded'`，`filename` 有值 |
| `action_logs.action_source` | `'api'` 或 `'gui'` |
| 磁盘物理文件 | ❌ 不存在 |
| `Storage::exists($filename)` | 返回 `false` |
| 应用日志 | 仅 `debug` 级别有异常记录，`error` 级别无痕迹 |
| 用户反馈 | 显示"上传成功"，但点击下载失败 |

#### 3.5.5 后续查询结果

##### 场景 1：附件列表查询 (`$maintenance->uploads()`)

**SQL 逻辑**：
```sql
SELECT * FROM action_logs 
WHERE item_type = 'App\Models\Maintenance' 
  AND item_id = 123
  AND action_type = 'uploaded' 
  AND filename IS NOT NULL
  AND filename NOT IN (
    SELECT filename FROM action_logs 
    WHERE item_type = 'App\Models\Maintenance' 
      AND item_id = 123
      AND action_type = 'upload deleted'
  )
```

**用户看到的结果**：
- ✅ 该文件**正常显示**在附件列表中
- 显示正确的文件名、上传时间、上传者、备注
- ❌ 点击下载或预览会失败

##### 场景 2：下载/预览文件 (`show()` 方法)

**代码逻辑**：
```php
// Api/UploadedFilesController.php:156-158
if (! Storage::exists(self::$map_storage_path[$object_type].$log->filename)) {
    return response()->json(Helper::formatStandardApiResponse(
        'error', null, trans('general.file_upload_status.file_not_found')
    ), 200);  // ⚠️  注意：HTTP 状态码是 200，不是 404
}
```

**用户看到的结果**：
- 返回 `File not found` 错误消息
- HTTP 状态码却是 `200 OK`（容易被前端误认为成功）

##### 场景 3：删除文件 (`destroy()` 方法)

**代码逻辑**：
```php
// UploadedFilesController.php:143-150
if (Storage::exists(self::$map_storage_path[$object_type].$log->filename)) {
    Storage::delete(...);  // 跳过，因为文件不存在
}
// 无论文件是否存在，都记录 upload deleted 日志
if ($log->logUploadDelete($object, $log->filename)) {
    return 'success';
}
```

**用户看到的结果**：
- ✅ 显示"删除成功"
- 写入一条 `action_type='upload deleted'` 的日志
- 下次列表查询时，该文件会被 `NOT IN` 子查询过滤，不再显示
- 整个过程用户感知不到文件原本就不存在

---

## 四、资产状态联动逻辑

### 4.1 维护单创建时的资产状态快照

**MaintenanceObserver**：`app/Observers/MaintenanceObserver.php:15-23`

```php
public function creating(Maintenance $maintenance): void
{
    // 记录创建时资产的分配状态快照
    if ($maintenance->asset_id && $asset = Asset::find($maintenance->asset_id)) {
        $maintenance->checked_out_to_id = $asset->assigned_to;
        $maintenance->checked_out_to_type = $asset->assigned_type;
    }

    $this->syncLegacyMaintenanceType($maintenance);
}
```

### 4.2 维护完成处理

**Web 控制器**：`app/Http/Controllers/MaintenancesController.php:245-270`

```php
public function complete(Request $request, Maintenance $maintenance): RedirectResponse
{
    // 防止重复完成
    if ($maintenance->completed_at) {
        return redirect()->back()
            ->with('warning', trans('admin/maintenances/form.already_complete'));
    }

    $maintenance->completed_at = now();
    $maintenance->completed_by = auth()->id();
    // 计算维护持续时间（天数）
    $maintenance->asset_maintenance_time = (int) $maintenance->created_at->diffInDays(now(), true);
    $maintenance->saveQuietly();  // 静默保存，不触发 update 事件

    // 记录完成日志
    $logAction = new Actionlog;
    $logAction->item_type = Maintenance::class;
    $logAction->item_id = $maintenance->id;
    $logAction->target_type = Asset::class;
    $logAction->target_id = $maintenance->asset_id;
    $logAction->created_by = auth()->id();
    $logAction->note = $request->input('note');
    $logAction->logaction(ActionType::MaintenanceComplete);  // 'completed'
}
```

### 4.3 资产状态只读关联

维护记录与资产状态之间是**只读关联**，不自动修改资产状态。资产状态通过 `asset_id` 外键关联读取：

**Transformer 中读取资产状态**：`app/Http/Transformers/MaintenancesTransformer.php:43-48`

```php
'status_label' => (($assetmaintenance->asset) && ($assetmaintenance->asset->status)) ? [
    'id' => (int) $assetmaintenance->asset->status->id,
    'name' => e($assetmaintenance->asset->status->name),
    'status_type' => e($assetmaintenance->asset->status->getStatuslabelType()),
    'status_meta' => e($assetmaintenance->asset->present()->statusMeta),
] : null,
```

### 4.4 维护类型与资产生命周期

维护类型通过 `maintenance_type_id` 关联到 `maintenance_types` 表：

**同步旧维护类型**：`app/Observers/MaintenanceObserver.php:66-74`

```php
private function syncLegacyMaintenanceType(Maintenance $maintenance): void
{
    if ($maintenance->maintenance_type_id && ! $maintenance->asset_maintenance_type) {
        $type = MaintenanceType::find($maintenance->maintenance_type_id);
        if ($type) {
            $maintenance->asset_maintenance_type = $type->name;
        }
    }
}
```

内置维护类型（8 种）：
- Maintenance（日常维护）
- Repair（维修）
- Upgrade（升级）
- PAT Test（电气安全测试）
- Calibration（校准）
- Software Support（软件支持）
- Hardware Support（硬件支持）
- Configuration Change（配置变更）

> **关键点**：
> - 维护单创建时**快照**资产的分配状态（`checked_out_to_id`、`checked_out_to_type`），用于历史追溯
> - 维护单完成时**不自动修改**资产的实际状态，需用户手动在资产页面调整
> - 维护单与资产状态是**只读关联**，通过 `$maintenance->asset->status` 实时读取当前状态
> - `saveQuietly()` 用于完成操作，避免触发 `updating` 事件产生重复日志
> - 维护持续时间通过 `created_at` 到 `completed_at` 的天数差计算

---

## 五、完整调用链路时序图

```
用户提交维护表单
    ↓
MaintenancesController::store()
    ├─ 验证权限
    ├─ 验证文件上传 validateUploadedFiles()
    ├─ 循环处理选中资产
    │   ├─ 创建 Maintenance 模型
    │   ├─ 设置字段（cost 经过 setCostAttribute 处理）
    │   ├─ handleImages() 处理图片上传到 public 目录
    │   ├─ $maintenance->save()
    │   │   └─ MaintenanceObserver::creating()
    │   │       ├─ 快照资产分配状态
    │   │       └─ 同步维护类型名称
    │   ├─ MaintenanceObserver::created() 记录 create 日志
    │   └─ storeUploadedFiles() 处理附件到 private 目录
    │       ├─ handleFile() 生成文件名并存储
    │       └─ logUpload() 记录 uploaded 日志
    └─ 返回成功响应

用户标记维护完成
    ↓
MaintenancesController::complete()
    ├─ 检查是否已完成
    ├─ 设置 completed_at、completed_by
    ├─ 计算 asset_maintenance_time（天数）
    ├─ saveQuietly() 静默保存
    └─ 记录 MaintenanceComplete 日志
```

---

## 六、数据模型关系图

```
maintenances 表
├─ id (PK)
├─ asset_id (FK → assets.id)
├─ maintenance_type_id (FK → maintenance_types.id)
├─ supplier_id (FK → suppliers.id)
├─ name
├─ cost (decimal, 经过 ParseCurrency 处理)
├─ start_date
├─ completion_date
├─ notes
├─ image (主图，存储在 public 目录)
├─ checked_out_to_id (创建时快照)
├─ checked_out_to_type (创建时快照)
├─ responsible_party_id (FK → users.id)
├─ completed_at
├─ completed_by (FK → users.id)
├─ asset_maintenance_time (int, 天数)
└─ created_by (FK → users.id)

action_logs 表（附件关联）
├─ id (PK)
├─ item_type = 'App\Models\Maintenance'
├─ item_id = maintenances.id
├─ action_type = 'uploaded' | 'upload deleted' | 'create' | 'update' | 'delete' | 'completed'
├─ filename (附件文件名)
├─ note
├─ created_by
└─ company_id

maintenance_types 表
├─ id (PK)
├─ name (维护类型名称)
└─ created_by
```

---

## 七、附件落盘与 uploaded 日志执行路径矩阵

| 操作场景 | 入口路径 | 附件落盘 | `uploaded` 日志 |
|---------|---------|----------|----------------|
| Web 创建维护单 + 上传附件 | `MaintenancesController@store` | ✅ 执行 | ✅ 写入 |
| API 创建维护单 + 上传附件 | `Api/MaintenancesController@store` | ❌ **不执行** | ❌ **不写入** |
| Web 单独上传附件 | `UploadedFilesController@store` | ✅ 执行 | ✅ 写入 |
| API 单独上传附件 | `Api/UploadedFilesController@store` | ✅ 执行 | ✅ 写入 |
| Web 更新维护单 + 上传附件 | `MaintenancesController@update` | ✅ 执行 | ✅ 写入 |
| API 更新维护单 + 上传附件 | `Api/MaintenancesController@update` | ❌ **不执行** | ❌ **不写入** |

> **重要提示**：通过 API 创建/更新维护单时，附件必须单独调用 `/api/v1/files/maintenances/{id}` 接口上传，否则附件会被静默丢弃。

---

## 八、关键技术点总结

| 功能点 | 实现方式 | 关键代码位置 |
|--------|----------|--------------|
| 成本字段处理 | Mutator + Helper 货币解析 | `Maintenance.php:151-158`, `Helper.php:579-592` |
| 图片上传 | `ImageUploadRequest::handleImages()` | `ImageUploadRequest.php:73-154` |
| 附件上传 | `UploadFileRequest::handleFile()` | `UploadFileRequest.php:43-63` |
| 附件存储路径 | 基类静态数组配置 | `Controller.php:66-98` |
| 附件关联 | `action_logs` 表软关联 | `HasUploads.php:9-22` |
| 上传日志记录 | `Loggable::logUpload()` | `Loggable.php:521-541` |
| Web 创建时附件处理 | `storeUploadedFiles()` 内联方法 | `MaintenancesController.php:195-223` |
| 上传失败静默吞异常 | `try-catch` + `Log::debug()` | `UploadFileRequest.php:56-60` |
| 资产状态快照 | `creating` 观察者事件 | `MaintenanceObserver.php:15-23` |
| 维护完成处理 | `complete()` 方法 + `saveQuietly()` | `MaintenancesController.php:245-270` |
| 资产状态读取 | 只读关联 `$maintenance->asset->status` | `MaintenancesTransformer.php:43-48` |

---

## 九、设计特点与潜在优化点

### 设计特点
1. **附件与主记录解耦**：通过 `action_logs` 表关联，支持灵活的多文件管理和删除追踪
2. **公私目录分离**：图片存 `public`，附件存 `private_uploads`，权限控制更精细
3. **状态快照而非状态变更**：维护记录只记录创建时的资产分配状态，不干预资产生命周期
4. **批量操作支持**：支持一次为多个资产创建相同的维护记录
5. **Web/API 双轨设计**：Web 入口功能更完整，API 入口更轻量化

### 已知缺陷：上传写盘失败数据不一致

**根本原因**：`UploadFileRequest::handleFile()` 中 `Storage::put()` 异常被静默捕获（仅打 debug 日志），但调用方继续执行 `logUpload()` 写入数据库。

**风险等级**：中高

**影响范围**：所有使用 `UploadFileRequest::handleFile()` 的模块（不仅限于维护记录）

**修复建议**：
```php
// 原代码（有缺陷）
try {
    Storage::put($dirname.$file_name, $uploaded_file);
} catch (\Exception $e) {
    Log::debug($e);  // 级别太低
}
return $file_name;

// 建议修复方案
try {
    Storage::put($dirname.$file_name, $uploaded_file);
} catch (\Exception $e) {
    Log::error('File upload failed: '.$e->getMessage(), [
        'file_name' => $file_name,
        'dirname' => $dirname,
    ]);
    throw $e;  // 抛出异常让调用方处理
}
return $file_name;
```

### 关联不直观的原因
1. **附件无直接外键**：附件通过 `action_logs` 的多态关联，而非 `maintenances` 表的直接字段
2. **资产状态只读**：维护完成不自动变更资产状态，容易让人误解两者没有关联
3. **双轨维护类型**：同时存在 `maintenance_type_id`（新）和 `asset_maintenance_type`（旧）两个字段
4. **快照字段隐藏**：`checked_out_to_id` 和 `checked_out_to_type` 是后台自动填充，用户界面不明显
5. **Web/API 功能差异**：API 创建维护单不处理附件，容易造成 API 调用方误解
6. **异常静默处理**：上传失败不提示用户，导致用户以为上传成功实际失败

