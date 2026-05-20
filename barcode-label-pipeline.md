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

**核心逻辑**: `Label.php:132-177`

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

### 4.3 二维码内容拼接规则总结

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
| `plain_manufacturer_name` | 前缀 + 厂商名称 | `prefixApple` |
| `plain_location_name` | 前缀 + 位置名称 | `prefix北京办公室` |

### 4.4 旧版标签的条形码生成 (Legacy)

当 `label2_enable = 0` 时，使用旧版 HTML 标签模板：

**核心文件**: `resources/views/hardware/labels.blade.php:115, 161`

```html
<!-- QR Code 图片 -->
<img src="{{ config('app.url') }}/hardware/{{ $asset->id }}/qr_code" class="qr_img">

<!-- 一维条形码图片 -->
<img src="{{ config('app.url') }}/hardware/{{ $asset->id }}/barcode" class="barcode">
```

这些路由指向独立的图片生成控制器，通过 HTTP 请求动态生成 PNG 图片。

---

## 五、图片输出目录与 Logo 路径解析

### 5.1 Logo 来源优先级

**核心逻辑**: `Label.php:105-120`

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

### 5.2 存储磁盘与路径

| Logo 类型 | 磁盘 | 路径格式 | 完整路径示例 |
|-----------|------|----------|-------------|
| 公司 Logo | `public` | `companies/{filename}` | `storage/app/public/companies/acme.png` |
| 全局标签 Logo | `public` | `/{basename(filename)}` | `storage/app/public/logo.png` |
| 预览占位图 | - | `public_path()` | `public/img/label-preview-logo.png` |

> **注意**: TCPDF 的 `Image()` 方法需要 **绝对文件系统路径**，而非 URL。代码中使用 `Storage::disk('public')->path()` 获取实际文件路径。

### 5.3 图片输出机制

在模板的 `write()` 方法中，通过 `writeImage()` 绘制：

**核心方法**: `Label.php:316-411`

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
| `app/Models/Labels/Label.php` | 标签抽象基类，定义通用绘制方法 |
| `app/Models/Labels/Sheet.php` | 多标签/页纸张模板抽象基类 |
| `app/Models/Labels/Field.php` | 字段行解析器 |
| `app/Models/Labels/FieldOption.php` | 字段选项解析与数据提取 |
| `app/Models/Labels/DefaultLabel.php` | 默认可配置尺寸的标签模板 |
| `app/Http/Controllers/LabelsController.php` | 标签预览控制器 |
| `resources/views/hardware/labels.blade.php` | 旧版 HTML 标签模板 |
