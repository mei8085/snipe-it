# 资产标签与条形码生成管道分析

本文档深入解析 Snipe-IT 中资产标签（Label）和条形码（Barcode）的生成机制，重点阐述**标签模板**、**字段映射**、**图片输出目录**三者的协作流程。

---

## 一、整体架构概览

标签生成系统采用 **分层设计**，核心类继承关系如下：

```
Label (抽象基类)
├── Sheet (抽象类 - 支持多标签/页的纸张模板)
│   ├── RectangleSheet
│   │   └── DefaultLabel (默认自定义模板)
│   └── Sheets/Avery/* (各种 Avery 标签纸模板)
└── Tapes
    ├── Generic/GenericTape (通用连续标签)
    ├── Brother/TZe_* (Brother 标签机模板)
    └── Dymo/LabelWriter_* (Dymo 标签机模板)
```

### 核心协作流程图

```
用户请求生成标签
        │
        ▼
LabelsController::show() ── 构造测试数据 / 接收资产数据
        │
        ▼
LabelView (app/View/Label.php) ────────────────┐
        │                                      │
        ├─► 从 settings 读取配置               │
        │    (label2_template, label2_fields,  │
        │     label2_1d_type, label2_2d_type,  │
        │     label2_2d_target, 等)            │
        │                                      │
        ├─► 解析字段映射 (Field::fromString)   │
        │                                      │
        ├─► 条形码内容生成 (1D/2D)             ◄─── 本模块核心
        │                                      │
        ├─► Logo 路径解析                      │
        │                                      │
        └─► 组装每条资产的 record 数据         │
               │                               │
               ▼                               │
        Label 模板的 writeAll()                │
               │                               │
               ▼                               │
        Label 模板的 write()                   │
               │                               │
               ▼                               │
        TCPDF 输出 PDF                         │
                                               │
        (旧版标签)                             │
        view('hardware/labels')  ◄─────────────┘
               │
               ▼
        HTML + <img src="/barcode"> / <img src="/qr_code">
```

---

## 二、标签模板系统 (Label Templates)

### 2.1 模板发现机制

**核心文件**: `app/Models/Labels/Label.php:675-727`

```php
public static function find($name = null)
{
    // 扫描 app/Models/Labels 目录下所有非抽象子类
    return collect(File::allFiles(__DIR__))
        ->map(function ($file) {
            // 转换为命名空间类名
            return __NAMESPACE__.'\\'.implode('\\', $matches[1]);
        })
        ->filter(function ($name) {
            // 过滤：类存在、非抽象、继承自 Label
            $refClass = new \ReflectionClass($name);
            return !$refClass->isAbstract() 
                && $refClass->isSubclassOf(Label::class);
        })
        ->map(function ($name) {
            return new $name;
        });
}
```

> **设计意图**: 通过文件系统扫描自动发现模板，无需手动注册。新增模板只需在对应目录添加 PHP 类即可。

### 2.2 模板能力声明 (Support Flags)

每个模板通过以下抽象方法声明其支持的元素：

| 方法 | 返回类型 | 含义 |
|------|----------|------|
| `getSupportAssetTag()` | bool | 是否支持资产编号文本 |
| `getSupport1DBarcode()` | bool | 是否支持一维条形码 |
| `getSupport2DBarcode()` | bool | 是否支持二维条形码 (QR Code 等) |
| `getSupportFields()` | int | 支持显示的自定义字段数量 (如 4) |
| `getSupportLogo()` | bool | 是否支持 Logo 图片 |
| `getSupportTitle()` | bool | 是否支持标题文本 |

**示例** (`DefaultLabel.php:166-194`):
```php
public function getSupport1DBarcode() { return true; }
public function getSupport2DBarcode() { return true; }
public function getSupportFields() { return 4; }
public function getSupportLogo() { return true; }
public function getSupportTitle() { return true; }
```

### 2.3 模板布局绘制

每个模板实现 `write(TCPDF $pdf, Collection $record)` 方法定义具体布局。

**示例** (`TZe_24mm_A.php:65-125`):
```php
public function write($pdf, $record)
{
    $pa = $this->getPrintableArea();  // 获取可打印区域
    
    // 1. 左侧绘制二维码 + 底部资产编号
    if ($record->has('barcode2d')) {
        static::writeText($pdf, $record->get('tag'), ...);
        static::write2DBarcode(
            $pdf, 
            $record->get('barcode2d')->content,  // 内容
            $record->get('barcode2d')->type,     // 类型 (QRCODE,L 等)
            $currentX, $currentY,
            $barcodeSize, $barcodeSize
        );
    }
    
    // 2. 右侧绘制标题
    if ($record->has('title')) {
        static::writeText($pdf, $record->get('title'), ...);
    }
    
    // 3. 遍历绘制自定义字段
    foreach ($record->get('fields') as $field) {
        static::writeText($pdf, $field['label'], ...);  // 字段名
        static::writeText($pdf, $field['value'], ...);  // 字段值
    }
}
```

---

## 三、字段映射系统 (Field Mapping)

### 3.1 存储格式

字段映射配置存储在 `settings` 表的 `label2_fields` 字段中，格式为 **分号分隔的键值对字符串**：

```
格式: 标签1=数据源1|标签2=数据源2;标签3=数据源3;...
示例: 资产=name|型号=model.name;序列号=serial;购买日期=purchase_date
```

- `;` 分隔 **不同行** (显示在标签的不同行)
- `|` 分隔 **同一行内的多个字段** (用 `|` 连接显示)
- `=` 分隔 **显示标签** 和 **数据源路径**

### 3.2 解析流程

**核心文件**: `app/Models/Labels/Field.php` + `FieldOption.php`

**解析入口** (`Label.php:86-88`):
```php
$fieldDefinitions = collect(explode(';', $settings->label2_fields))
    ->filter()
    ->map(fn ($fieldString) => Field::fromString($fieldString));
```

**Field::fromString()** (`Field.php:46-54`):
```php
public static function fromString(string $theString)
{
    $field = new Field;
    $field->options = collect(explode('|', $theString))
        ->filter()
        ->map(fn ($optionString) => FieldOption::fromString($optionString));
    return $field;
}
```

**FieldOption::fromString()** (`FieldOption.php:82-92`):
```php
public static function fromString(string $theString)
{
    $parts = explode('=', $theString);  // "资产=name" => ["资产", "name"]
    if (count($parts) == 2) {
        $option = new FieldOption;
        $option->label = $parts[0];      // 显示标签: "资产"
        $option->dataSource = $parts[1]; // 数据源路径: "name"
        return $option;
    }
}
```

### 3.3 数据提取机制

**核心方法**: `FieldOption::getValue(Asset $asset)` (`FieldOption.php:23-54`)

```php
public function getValue(Asset $asset)
{
    $dataPath = collect(explode('.', $this->dataSource));
    
    // 特殊处理: assignedTo 关系
    if ($dataPath[0] === 'assignedTo') {
        return $asset->assigned ? $asset->assigned->display_name : null;
    }
    
    // 特殊处理: purchase_date 日期格式化
    if ($dataPath[0] === 'purchase_date') {
        return $asset->purchase_date ? $asset->purchase_date->format('Y-m-d') : null;
    }
    
    // 通用: 使用 reduce 按点路径逐级提取
    return $dataPath->reduce(function ($myValue, $path) {
        try {
            return $myValue ? $myValue->{$path} : ${$myValue};
        } catch (\Exception $e) {
            return $myValue;
        }
    }, $asset);
}
```

> **支持的点路径示例**:
> - `name` → `$asset->name`
> - `model.name` → `$asset->model->name`
> - `model.manufacturer.name` → `$asset->model->manufacturer->name`
> - `location.name` → `$asset->location->name`

### 3.4 数据组装 (多字段同行合并)

**核心逻辑**: `Label.php:179-214`

```php
$fields = $fieldDefinitions
    ->map(fn ($field) => $field->toArray($asset))
    ->reduce(function ($myFields, $field) {
        // 过滤重复字段
        $toAdd = $field
            ->filter(fn ($o) => !$myFields->contains('dataSource', $o['dataSource']))
            // 同一行多个字段用 " | " 连接
            ->reduce(function ($previous, $current) {
                if (is_null($previous)) {
                    return $current;
                }
                // 合并: "{标签1} {值1} | {标签2} {值2}"
                $previous['value'] = trim(implode(' | ', [
                    implode(' ', [$previous['label'], $previous['value']]),
                    implode(' ', [$current['label'], $current['value']]),
                ]));
                $previous['label'] = '';  // 标签已合并到值中
                return $previous;
            });
        
        return $toAdd ? $myFields->push($toAdd) : $myFields;
    }, new Collection);
```

---

## 四、条形码内容生成 (Barcode Content)

### 4.1 一维条形码 (1D Barcode)

**核心逻辑**: `Label.php:122-130`

```php
if ($template->getSupport1DBarcode()) {
    $barcode1DType = $settings->label2_1d_type;
    if ($barcode1DType != 'none') {
        $assetData->put('barcode1d', (object)[
            'type' => $barcode1DType,      // 如: C128, C39, EAN13 等
            'content' => $asset->asset_tag, // ⚠️ 内容固定为 asset_tag
        ]);
    }
}
```

> **关键点**: 一维条形码内容 **固定为资产编号 (asset_tag)**，不可配置。

### 4.2 二维条形码 (2D Barcode / QR Code)

**核心逻辑**: `app/View/Label.php:132-177`

```php
if ($template->getSupport2DBarcode()) {
    $barcode2DType = $settings->label2_2d_type;  // QRCODE,L, QRCODE,M 等
    
    $label2_2d_prefix = $settings->label2_2d_prefix ? e($settings->label2_2d_prefix) : '';
    
    switch ($settings->label2_2d_target) {
        case 'ht_tag':
            $barcode2DTarget = route('ht/assetTag', $asset->asset_tag);
            break;
        case 'plain_asset_id':
            $barcode2DTarget = $label2_2d_prefix . (string)$asset->id;
            break;
        case 'plain_asset_tag':
            $barcode2DTarget = $label2_2d_prefix . $asset->asset_tag;
            break;
        case 'plain_serial_number':
            $barcode2DTarget = $label2_2d_prefix . $asset->serial;
            break;
        case 'plain_model_number':
            $barcode2DTarget = $label2_2d_prefix . ($asset->model->model_number ?? '');
            break;
        case 'plain_model_name':
            $barcode2DTarget = $label2_2d_prefix . ($asset->model->display_name ?? '');
            break;
        case 'plain_manufacturer_name':
            $barcode2DTarget = $label2_2d_prefix . $asset->model->display_name;
            break;
        case 'plain_location_name':
            $barcode2DTarget = $label2_2d_prefix . $asset->location->name;
            break;
        case 'location':
            $barcode2DTarget = $asset->location_id 
                ? route('locations.show', $asset->location_id) 
                : null;
            break;
        case 'hardware_id':
        default:
            $barcode2DTarget = route('hardware.show', $asset);
            break;
    }
    
    $assetData->put('barcode2d', (object)[
        'type' => $barcode2DType,
        'content' => $barcode2DTarget,
    ]);
}
```

### 4.3 display_name 的解析机制

在理解具体取值之前，需要先了解 `display_name` 的魔术方法解析机制：

**核心逻辑**: `app/Presenters/Presenter.php:128-135`

```php
public function __get($property)
{
    if (method_exists($this, $property)) {
        return $this->{$property}();
    }
    return $this->model->{$property};
}
```

当访问 `$model->display_name` 时：
1. 首先检查 Presenter 类中是否存在 `display_name()` 方法
2. 如果存在，调用该方法并返回结果
3. 如果不存在，直接返回底层模型的 `display_name` 属性

**重要发现**: 
- `AssetModelPresenter` 中没有定义 `display_name()` 方法
- 因此 `$asset->model->display_name` 实际返回的是 `$asset->model->name`（即型号名称）

### 4.4 二维码目标字段取值详解与容易混淆的地方

下表详细列出每个 `label2_2d_target` 的**实际代码取值**、**预期含义**以及**容易混淆的问题**：

| label2_2d_target | 实际代码路径 | 真实取值 | 预期含义 | 混淆风险 |
|------------------|-------------|----------|----------|---------|
| `hardware_id` (默认) | `route('hardware.show', $asset)` | 资产详情页 URL | 资产详情页 URL | ✓ 无混淆 |
| `ht_tag` | `route('ht/assetTag', $asset->asset_tag)` | 自助查询 URL | 自助查询 URL | ✓ 无混淆 |
| `location` | `route('locations.show', $asset->location_id)` | 位置详情 URL | 位置详情 URL | ✓ 无混淆 |
| `plain_asset_id` | `$asset->id` | 资产数据库 ID | 资产数据库 ID | ✓ 无混淆 |
| `plain_asset_tag` | `$asset->asset_tag` | 资产编号 | 资产编号 | ✓ 无混淆 |
| `plain_serial_number` | `$asset->serial` | 序列号 | 序列号 | ✓ 无混淆 |
| `plain_model_number` | `$asset->model->model_number` | 型号编号 | 型号编号 | ✓ 无混淆 |
| `plain_model_name` | `$asset->model->display_name` | 型号名称 (通过Presenter) | 型号名称 | ✓ 正确，但需注意 `display_name` 解析机制 |
| `plain_manufacturer_name` | `$asset->model->display_name` | **型号名称** | **制造商名称** | ⚠️ **严重BUG！代码错误地返回了型号名称，而非制造商名称。正确代码应为 `$asset->model->manufacturer->name`** |
| `plain_location_name` | `$asset->location->name` | 位置名称 | 位置名称 | ✓ 无混淆 |

> **关键BUG警告**: `plain_manufacturer_name` 选项存在代码错误！它返回的是 `$asset->model->display_name`（型号名称），而不是预期的 `$asset->model->manufacturer->name`（制造商名称）。这意味着 `plain_model_name` 和 `plain_manufacturer_name` 当前返回完全相同的值。

### 4.5 二维码内容拼接规则总结

| label2_2d_target | 内容格式 | 示例 |
|------------------|----------|------|
| `hardware_id` (默认) | 资产详情 URL | `https://snipe.example.com/hardware/123` |
| `ht_tag` | 资产自助查询 URL | `https://snipe.example.com/ht/assetTag/ASS-001` |
| `location` | 位置详情 URL | `https://snipe.example.com/locations/5` |
| `plain_asset_id` | 前缀 + 资产ID | `prefix456` |
| `plain_asset_tag` | 前缀 + 资产编号 | `prefixASS-001` |
| `plain_serial_number` | 前缀 + 序列号 | `prefixSN123456` |
| `plain_model_number` | 前缀 + 型号编号 | `prefixMDL-789` |
| `plain_model_name` | 前缀 + 型号名称 | `prefixMacBook Pro` |
| `plain_manufacturer_name` | 前缀 + **型号名称** (BUG) | `prefixMacBook Pro` (应为 `prefixApple`) |
| `plain_location_name` | 前缀 + 位置名称 | `prefix北京办公室` |

### 4.6 旧版标签的条形码生成 (Legacy)

当 `label2_enable = 0` 时，使用旧版 HTML 标签模板：

**核心文件**: `resources/views/hardware/labels.blade.php:115, 161`

```html
<!-- QR Code 图片 -->
<img src="{{ config('app.url') }}/hardware/{{ $asset->id }}/qr_code" class="qr_img">

<!-- 一维条形码图片 -->
<img src="{{ config('app.url') }}/hardware/{{ $asset->id }}/barcode" class="barcode">
```

这些路由指向独立的图片生成控制器，通过 HTTP 请求动态生成 PNG 图片。

#### 旧版条码生成流程

**核心文件**: `app/Http/Controllers/Assets/AssetsController.php:618-691`

```
浏览器请求 <img src="/hardware/123/qr_code">
        │
        ▼
AssetsController::getQrCode(Asset $asset)
        │
        ├─► 检查缓存文件是否存在
        │    $qr_file = public_path().'/uploads/barcodes/qr-{asset_tag}-{id}.png'
        │
        ├─► 存在 → 直接返回文件 response()->file($qr_file)
        │
        └─► 不存在 → 使用 Barcode 库生成 PNG
              ├─► $barcode = new Barcode;
              ├─► $barcode->getBarcodeObj($type, $content, ...)
              ├─► file_put_contents($qr_file, $barcode_obj->getPngData())
              └─► 返回 PNG 数据
```

**关键注意**: 
- 旧版二维码内容**固定**为 `route('hardware.show', $asset->id)`，不支持多种 target 选项
- 旧版一维码内容固定为 `$asset->asset_tag`（与新版相同）

#### 旧版条码缓存目录

| 条码类型 | 文件路径格式 | 完整路径示例 |
|---------|------------|-------------|
| 二维码 | `public/uploads/barcodes/qr-{asset_tag}-{id}.png` | `public/uploads/barcodes/qr-ass-001-42.png` |
| 一维码 | `public/uploads/barcodes/{type}-{asset_tag}.png` | `public/uploads/barcodes/c128-ass-001.png` |
| 无效条码占位图 | `public/uploads/barcodes/invalid_barcode.gif` | `public/uploads/barcodes/invalid_barcode.gif` |

> **磁盘映射**: `public/uploads/barcodes/` 实际对应 `storage/app/public/barcodes/`（通过 Laravel 的 storage:link 软链接）

#### 缓存清理机制

**核心文件**: `app/Http/Controllers/Api/SettingsController.php:196-222`

```php
public function purgeBarcodes(): JsonResponse
{
    $files = Storage::disk('public')->files('barcodes');
    
    foreach ($files as $file) {
        $extension = end(explode('.', $file));
        if ($extension == 'png') {  // 只删除 .png 缓存文件，保留 .gif 占位图
            Storage::disk('public')->delete($file);
            $file_count++;
        }
    }
    
    return response()->json(['message' => 'Deleted '.$file_count.' barcodes'], 200);
}
```

> **设计意图**: 当条码类型设置变更或系统 URL 变更时，需要清除缓存以重新生成条码。清除后下次访问时会自动重新生成。

### 4.7 新旧版条码生成的差异与协作关系

#### 核心差异对比

| 维度 | 旧版 (label2_enable = 0) | 新版 (label2_enable = 1) |
|------|------------------------|------------------------|
| 输出格式 | HTML 页面 + PNG 图片 | PDF 文档 (矢量图形) |
| 条码生成方式 | 独立 HTTP 请求动态生成 PNG 图片 | TCPDF 直接矢量绘制到 PDF |
| 缓存机制 | 生成 PNG 文件缓存到 `storage/app/public/barcodes/` | **无缓存**，内存中直接绘制 |
| 二维码内容 | 固定为资产详情 URL | 根据 `label2_2d_target` 配置，支持 10 种目标 |
| 一维码内容 | `$asset->asset_tag` | `$asset->asset_tag` (相同) |
| 条码类型配置 | 使用 `label2_1d_type` / `label2_2d_type` | 使用相同的配置 |
| 缓存清理 | `purgeBarcodes()` API 可清除缓存 | 无需清理，无缓存文件 |
| 性能特点 | 首次访问生成 PNG，后续命中缓存；多一次 HTTP 请求开销 | 直接生成 PDF，无额外 HTTP 请求；批量生成效率更高 |
| 打印质量 | PNG 位图，质量取决于生成时的 DPI | PDF 矢量图形，无限缩放不失真 |
| 布局灵活性 | HTML/CSS 控制，易于自定义 | PHP 代码控制布局，需编写模板类 |

#### 协作关系

```
┌─────────────────────────────────────────────────────────────────┐
│                         配置层 (settings 表)                     │
│  label2_enable, label2_1d_type, label2_2d_type, label2_2d_target │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          ▼                       ▼                       ▼
┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
│  label2_enable=0  │   │  label2_enable=1  │   │  独立路由          │
│  旧版标签系统      │   │  新版标签系统      │   │  /qr_code /barcode │
└─────────┬─────────┘   └─────────┬─────────┘   └─────────┬─────────┘
          │                       │                       │
          ▼                       ▼                       ▼
┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
│ HTML 模板渲染      │   │ LabelView 组装数据 │   │ AssetsController  │
│ labels.blade.php  │   │  直接生成条码内容  │   │  生成 PNG 图片    │
└─────────┬─────────┘   └─────────┬─────────┘   └─────────┬─────────┘
          │                       │                       │
          ▼                       ▼                       ▼
┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
│ <img src=...>     │   │ TCPDF 矢量绘制    │   │  写入缓存目录      │
│ 触发 HTTP 请求    │   │  输出 PDF         │   │  返回 PNG 响应    │
└─────────┬─────────┘   └───────────────────┘   └─────────┬─────────┘
          │                                               │
          └───────────────────┬───────────────────────────┘
                              ▼
                    ┌───────────────────┐
                    │ 缓存目录           │
                    │ storage/app/public │
                    │   /barcodes/       │
                    └───────────────────┘
```

**协作要点说明**:

1. **配置共享**: 两套系统共用 `settings` 表中的条码类型配置（`label2_1d_type`, `label2_2d_type`），但只有新版使用 `label2_2d_target`

2. **切换开关**: 通过 `label2_enable` 设置切换标签系统：
   - `0` → 使用旧版 HTML 标签系统
   - `1` → 使用新版 PDF 标签系统

3. **独立路由共存**: `/hardware/{id}/qr_code` 和 `/hardware/{id}/barcode` 路由始终可用，与 `label2_enable` 无关。它们服务于：
   - 旧版标签的 `<img>` 标签
   - 其他可能需要独立条码图片的场景

4. **缓存目录共存**: 旧版缓存目录 `storage/app/public/barcodes/` 始终存在：
   - 新版不使用缓存目录，也不修改其中的文件
   - `purgeBarcodes()` API 只清除旧版缓存的 `.png` 文件
   - `invalid_barcode.gif` 占位图不会被清除

5. **条码内容差异**: 
   - 旧版二维码内容固定为 `route('hardware.show', $asset->id)`
   - 新版二维码内容根据 `label2_2d_target` 配置动态生成
   - 因此即使条码类型配置相同，新旧版生成的二维码内容可能不同

---

## 五、图片输出目录与 Logo 路径解析

### 5.1 Logo 来源优先级

**核心逻辑**: `app/View/Label.php:105-120`

```php
if ($template->getSupportLogo()) {
    $logo = null;
    
    // 优先级 1: 资产所属公司的 Logo (如果启用 label2_asset_logo)
    if ($settings->label2_asset_logo && $asset->company && $asset->company->image != '') {
        $logo = Storage::disk('public')->path('companies/'.e($asset->company->image));
    }
    // 优先级 2: 全局设置的标签 Logo
    elseif (!empty($settings->label_logo)) {
        $logo = Storage::disk('public')->path('/'.e(basename($settings->label_logo)));
    }
    // 优先级 3: 预览模式下的占位 Logo
    elseif (!empty($asset->is_label_preview)) {
        $logo = public_path('img/label-preview-logo.png');
    }
    
    if (!empty($logo)) {
        $assetData->put('logo', $logo);
    }
}
```

### 5.2 Logo 存储路径详解

所有 Logo 图片都存储在 `public` 磁盘（即 `storage/app/public/`）下，通过 Laravel 的 `storage:link` 软链接到 `public/storage/` 目录。

| Logo 类型 | 存储路径格式 | 实际文件路径示例 | 数据库字段值示例 |
|-----------|-------------|-----------------|-----------------|
| 公司 Logo | `companies/{filename}` | `storage/app/public/companies/acme-corp-abc123.png` | `companies/acme-corp-abc123.png` |
| 全局标签 Logo | `/{filename}` (存储在 public 磁盘根目录) | `storage/app/public/setting-label_logo-1-xyz789.png` | `setting-label_logo-1-xyz789.png` |
| 预览占位图 | `public_path()` 目录 | `public/img/label-preview-logo.png` | (硬编码，不存数据库) |

> **关键注意**:
> 1. 全局标签 Logo 的文件名格式：`setting-label_logo-{setting_id}-{random_str}.{ext}`，由 `ImageUploadRequest::handleImages()` 方法生成
> 2. 代码中使用 `basename($settings->label_logo)` 提取文件名，确保只取文件名部分
> 3. TCPDF 的 `Image()` 方法需要 **绝对文件系统路径**，而非 URL。代码中使用 `Storage::disk('public')->path()` 获取实际文件路径

### 5.3 条码缓存目录（旧版）

**核心文件**: `app/Http/Controllers/Assets/AssetsController.php:626, 664`

| 条码类型 | 存储路径格式 | 实际文件路径示例 |
|---------|-------------|-----------------|
| 二维码 | `barcodes/qr-{asset_tag}-{id}.png` | `storage/app/public/barcodes/qr-ass-001-42.png` |
| 一维码 | `barcodes/{type}-{asset_tag}.png` | `storage/app/public/barcodes/c128-ass-001.png` |
| 无效条码占位图 | `barcodes/invalid_barcode.gif` | `storage/app/public/barcodes/invalid_barcode.gif` |

> **存储逻辑**:
> - 首次访问 `/hardware/{id}/qr_code` 或 `/hardware/{id}/barcode` 时生成图片并写入缓存
> - 后续访问直接读取缓存文件
> - `purgeBarcodes()` API 只删除 `.png` 文件，保留 `.gif` 占位图

### 5.4 新旧版图片输出机制对比

#### 新版标签系统（PDF 直接嵌入）

**核心方法**: `app/Models/Labels/Label.php:316-411`

```php
final public function writeImage(TCPDF $pdf, $image, $x, $y, $width = null, $height = null, ...)
{
    $imageInfo = getimagesize($image);  // 读取图片信息
    $imageWidthPx = $imageInfo[0];
    $imageHeightPx = $imageInfo[1];
    $imageType = image_type_to_extension($imageInfo[2], false);
    
    // 按 DPI 转换像素到物理单位
    $dpu = Helper::convertUnit($dpi, $this->getUnit(), 'in');
    $imageWidth = $imageWidthPx / $dpu;
    $imageHeight = $imageHeightPx / $dpu;
    
    // 计算缩放、对齐后实际绘制到 PDF
    $pdf->Image($image, $originX, $originY, $outputWidth, $outputHeight, $imageType, ...);
}
```

**特点**:
- 直接读取图片文件，嵌入 PDF 文档
- 不生成额外的缓存文件
- 输出为 PDF 矢量/位图混合格式
- Logo 图片在 PDF 生成时一次性读取嵌入

#### 旧版标签系统（HTML + 图片 URL）

**核心文件**: `resources/views/hardware/labels.blade.php:113-163`

```html
<!-- QR Code 图片 -->
<div class="qr_img">
    <img src="{{ config('app.url') }}/hardware/{{ $asset->id }}/qr_code" class="qr_img">
</div>

<!-- Logo 图片 -->
@if ($settings->label_logo)
    <div class="label-logo">
        <img class="label-logo" src="{{ Storage::disk('public')->url('').e($snipeSettings->label_logo) }}">
    </div>
@endif

<!-- 一维条形码图片 -->
@if ((($settings->alt_barcode_enabled=='1') && $settings->label2_1d_type!=''))
    <div class="barcode_container">
        <img src="{{ config('app.url') }}/hardware/{{ $asset->id }}/barcode" class="barcode">
    </div>
@endif
```

**特点**:
- 使用 URL 引用图片，浏览器通过 HTTP 请求获取
- 条码图片通过独立路由生成并缓存
- Logo 图片通过 `Storage::url()` 生成公开 URL
- 输出为 HTML 页面，由浏览器渲染后打印

#### 图片路径对比总结

| 图片类型 | 新版 (PDF) | 旧版 (HTML) |
|---------|-----------|------------|
| Logo 路径 | `Storage::disk('public')->path($file)` → 绝对文件路径 | `Storage::disk('public')->url($file)` → HTTP URL |
| 条码路径 | 直接绘制到 PDF，无文件 | `/hardware/{id}/qr_code` → HTTP URL → 缓存文件 |
| 图片格式 | 直接嵌入 PDF | PNG/GIF 图片文件 |
| 缓存机制 | 无缓存 | 条码缓存到 `storage/app/public/barcodes/` |

> **容易混淆的地方**:
> - `path()` 方法返回**文件系统绝对路径**（用于 TCPDF 读取）
> - `url()` 方法返回**公开访问 URL**（用于 HTML 引用）
> - 两者指向同一个物理文件，但用途完全不同
> - 条码缓存目录 `barcodes/` 只被旧版系统使用，新版系统不生成也不依赖这些缓存文件

---

## 六、完整数据流示例

假设配置如下：
- `label2_template = Tapes\Brother\TZe_24mm_A`
- `label2_2d_type = QRCODE,L`
- `label2_2d_target = hardware_id`
- `label2_1d_type = none`
- `label2_fields = 名称=name|型号=model.name;序列号=serial;使用人=assignedTo`
- `label2_title = {COMPANY} 资产标签`
- `label2_asset_logo = 1`

### 数据组装结果 (单条 record):

```php
Collection {
    'asset' => Asset {#123},
    'id' => 42,
    'tag' => 'ASS-0042',
    'title' => 'ACME Corp 资产标签',
    'logo' => '/var/www/storage/app/public/companies/acme.png',
    'barcode2d' => {
        'type' => 'QRCODE,L',
        'content' => 'https://snipe.example.com/hardware/42'
    },
    'fields' => [
        [
            'label' => '',
            'dataSource' => 'name',
            'value' => '名称 MacBook Pro 16 | 型号 Apple MacBook Pro 16'
        ],
        [
            'label' => '序列号',
            'dataSource' => 'serial',
            'value' => 'C02XL0GTJGH7'
        ],
        [
            'label' => '使用人',
            'dataSource' => 'assignedTo',
            'value' => '张三'
        ]
    ]
}
```

---

## 七、关键配置项速查表

| 配置字段 | 存储位置 | 含义 |
|---------|----------|------|
| `label2_enable` | settings | 是否启用新版标签系统 (0=旧版, 1=新版) |
| `label2_template` | settings | 选中的标签模板类名 |
| `label2_1d_type` | settings | 一维码类型 (C128, C39, none 等) |
| `label2_2d_type` | settings | 二维码类型 (QRCODE,L, QRCODE,M 等) |
| `label2_2d_target` | settings | 二维码内容目标类型 |
| `label2_2d_prefix` | settings | 二维码内容前缀 |
| `label2_fields` | settings | 字段映射配置字符串 |
| `label2_title` | settings | 标签标题 (支持 `{COMPANY}` 占位符) |
| `label2_asset_logo` | settings | 是否优先使用资产所属公司 Logo |
| `label_logo` | settings | 全局标签 Logo 文件名 |
| `label2_empty_row_count` | settings | 字段上方空行数 |

---

## 八、核心文件索引

| 文件路径 | 职责 |
|---------|------|
| `app/View/Label.php` | **核心调度器** - 数据组装、条形码生成、PDF 输出 |
| `app/Models/Labels/Label.php` | 标签抽象基类，定义通用绘制方法 (writeText/writeImage/write1DBarcode/write2DBarcode) |
| `app/Models/Labels/Sheet.php` | 多标签/页纸张模板抽象基类 |
| `app/Models/Labels/Field.php` | 字段行解析器 |
| `app/Models/Labels/FieldOption.php` | 字段选项解析与数据提取 |
| `app/Models/Labels/DefaultLabel.php` | 默认可配置尺寸的标签模板 |
| `app/Http/Controllers/LabelsController.php` | 标签预览控制器 |
| `app/Http/Controllers/Assets/AssetsController.php` | **旧版条码生成控制器** - getQrCode(), getBarCode() 方法 |
| `app/Http/Controllers/Api/SettingsController.php` | **缓存清理控制器** - purgeBarcodes() 方法 |
| `app/Http/Requests/ImageUploadRequest.php` | 图片上传处理，生成 label_logo 文件名 |
| `app/Presenters/Presenter.php` | Presenter 基类，定义 `__get()` 魔术方法解析 display_name |
| `resources/views/hardware/labels.blade.php` | 旧版 HTML 标签模板 |

---

## 九、关键问题总结

### 9.1 已确认的代码 BUG

**`plain_manufacturer_name` 返回错误值** (`app/View/Label.php:156-157`)

```php
case 'plain_manufacturer_name':
    $barcode2DTarget = $label2_2d_prefix.$asset->model->display_name;
    break;
```

- **问题**: 代码返回 `$asset->model->display_name`（型号名称），而非预期的 `$asset->model->manufacturer->name`（制造商名称）
- **影响**: `plain_model_name` 和 `plain_manufacturer_name` 返回完全相同的值
- **修复方案**: 将代码改为 `$asset->model->manufacturer->name ?? ''`

### 9.2 容易混淆的概念

| 概念 | 说明 |
|------|------|
| `path()` vs `url()` | `path()` 返回文件系统绝对路径（供 TCPDF 读取），`url()` 返回公开访问 URL（供 HTML 引用） |
| `display_name` 解析 | 通过 Presenter 的 `__get()` 魔术方法，先检查是否有同名方法，再返回模型属性 |
| `label2_2d_target` 选项 | 10 个选项中 `plain_manufacturer_name` 存在 BUG，其余正常 |
| 条码缓存目录 | `storage/app/public/barcodes/` 只被旧版系统使用，新版 PDF 系统不生成缓存文件 |
| 新旧版二维码内容 | 旧版固定为资产详情 URL，新版根据 `label2_2d_target` 配置生成 |

### 9.3 新旧版协作要点

1. 两套系统共享条码类型配置，但只有新版使用 `label2_2d_target`
2. 独立路由 `/hardware/{id}/qr_code` 和 `/hardware/{id}/barcode` 始终可用，与 `label2_enable` 无关
3. `purgeBarcodes()` API 只清除旧版缓存的 `.png` 文件，不影响新版
4. `barcodes/` 缓存目录始终存在，新版不使用但也不清理
5. 切换 `label2_enable` 即可在两套系统间切换，无需其他配置变更
