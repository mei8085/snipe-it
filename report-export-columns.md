# 报表导出功能与列选择实现分析

## 概述

Snipe-IT 项目中存在**两种独立的报表导出体系**：

1. **前端客户端导出**（Bootstrap Table 导出插件）：在浏览器端基于已加载的表格数据导出，支持 xlsx、csv、pdf、json、xml、txt、sql、doc 等 8 种格式
2. **后端服务器端导出**：在 PHP 后端生成 CSV 文件流下载，主要用于大数据量导出和自定义报表

两种体系的列选择机制完全不同，下文分别详细分析。

---

## 一、前端客户端导出（Bootstrap Table 方式）

### 1.1 整体架构

```
列配置定义 (Presenter::dataTableLayout)
        ↓
HTML data-columns 属性（JSON 格式）
        ↓
Bootstrap Table 初始化（partials/bootstrap-table.blade.php）
        ↓
showColumns 列选择 UI（勾选/取消勾选）
        ↓
exportTypes/exportOptions 配置导出
        ↓
前端插件 tableExport.jquery.plugin 生成文件下载
```

### 1.2 列配置定义

列配置通过各个 Presenter 类的 `dataTableLayout()` 静态方法定义，返回 JSON 格式的列配置数组。

**关键代码位置**：
- [DepreciationReportPresenter::dataTableLayout()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/DepreciationReportPresenter.php#L18-L174)

**每个列的配置属性**：

| 属性 | 说明 | 对导出的影响 |
|------|------|-------------|
| `field` | 字段唯一标识（对应后端数据字段） | 导出时作为列数据的 key |
| `title` | 列显示标题（国际化翻译） | 导出时作为列标题 |
| `visible` | 是否默认显示 | 控制初始可见性，但不影响导出（导出可选择隐藏列） |
| `switchable` | 是否允许用户切换显示/隐藏 | `false` 时该列不能被用户隐藏 |
| `searchable` | 是否可搜索 | 与导出无关 |
| `sortable` | 是否可排序 | 与导出无关 |
| `formatter` | 前端格式化函数名 | **重要**：导出时会应用格式化函数 |
| `class` | CSS 类名 | 导出时部分格式（如 xlsx）会保留样式 |

**典型列定义示例**（来自 [DepreciationReportPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/DepreciationReportPresenter.php#L20-L171)）：

```php
$layout = [
    [
        'field' => 'asset_tag',
        'searchable' => true,
        'sortable' => true,
        'title' => trans('general.asset_tag'),
        'visible' => true,
    ],
    [
        'field' => 'purchase_date',
        'searchable' => true,
        'sortable' => true,
        'visible' => true,
        'title' => trans('general.purchase_date'),
        'formatter' => 'dateDisplayFormatter',  // 前端格式化
    ],
    // ...
];
return json_encode($layout);
```

### 1.3 列配置的使用方式

在 Blade 视图中，通过 `data-columns` 属性将列配置传给表格元素：

**方式一：使用 Presenter::dataTableLayout()（推荐方式）**

[activity.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/activity.blade.php#L24-L39)：
```blade
<table
    data-columns="{{ \App\Presenters\HistoryPresenter::dataTableLayout() }}"
    data-cookie-id-table="activityReport"
    data-url="{{ route('api.activity.index') }}"
    ...
    data-export-options='{
        "fileName": "activity-report-{{ date('Y-m-d') }}",
        "ignoreColumn": ["actions","image","change","checkbox","checkincheckout","icon"]
    }'>
</table>
```

**方式二：在 `<th>` 标签上使用 data-* 属性（传统方式）**

[asset.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/asset.blade.php#L36-L76)：
```blade
<thead>
    <tr>
        <th data-sortable="true" data-field="id" data-visible="false">{{ trans('general.id') }}</th>
        <th data-field="company" data-switchable="true" data-visible="false">{{ trans('general.company') }}</th>
        <th data-sortable="true" data-field="asset_tag">{{ trans('admin/hardware/table.asset_tag') }}</th>
        <!-- 自定义字段动态生成 -->
        @foreach(\App\Models\CustomField::all() AS $field)
            <th data-field="{{$field->db_column_name()}}" data-visible="false">{{$field->name}}</th>
        @endforeach
    </tr>
</thead>
```

### 1.4 Bootstrap Table 初始化与列选择 UI

核心初始化代码位于 [bootstrap-table.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/partials/bootstrap-table.blade.php)。

**关键配置项**（[bootstrap-table.blade.php:547-639](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/partials/bootstrap-table.blade.php#L547-L639)）：

```javascript
$(this).bootstrapTable({
    showColumns: true,              // 显示列选择按钮
    showColumnsToggleAll: true,     // 显示"全选/取消全选"
    showExport: true,               // 显示导出按钮
    minimumCountColumns: 2,         // 最少保留显示的列数
    cookie: true,                   // 记住用户的列选择
    cookieExpire: '2y',             // Cookie 有效期 2 年
    cookieStorage: 'localStorage' or 'cookie',  // 存储位置
    exportTypes: ['xlsx', 'csv', 'pdf', 'json', 'xml', 'txt', 'sql', 'doc'],  // 支持的导出格式
    exportOptions: export_options,  // 导出选项配置
    // ...
});
```

**列选择工作原理**：
- 点击工具栏的 "列" 按钮（fa-columns 图标），Bootstrap Table 会弹出下拉菜单列出所有 `switchable: true` 的列
- 用户勾选/取消勾选会即时更新表格显示
- 列状态通过 Cookie 或 localStorage 持久化保存（2 年有效期）

### 1.5 前端导出配置详解

导出选项配置在 [bootstrap-table.blade.php:480-516](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/partials/bootstrap-table.blade.php#L480-L516)：

```javascript
data_export_options = $(this).attr('data-export-options');
export_options = data_export_options ? JSON.parse(data_export_options) : {};
export_options['htmlContent'] = false;  // 不导出 HTML 内容，导出纯文本

// PDF 导出配置
export_options['jspdf'] = {
    "orientation": "l",           // 横向
    "autotable": {
        "styles": { overflow: 'linebreak' },
        tableWidth: 'wrap'
    }
};

// XSS 防护：导出时处理 HTML 数据
export_options['onCellHtmlData'] = function (cell, rowIndex, colIndex, htmlData) {
    if (cell.is('th')) {
        return cell.find('.th-inner').text()
    }
    // <br> 转换行符
    return htmlData.replace(/<br\s*\/?>/gi, '\n');
}

// Excel 超链接处理：防止双引号破坏 HYPERLINK 公式
export_options['mso'] = {
    xlsx: {
        onHyperlink: function (cell, row, col, href, cellText, formula) {
            var escapedHref = href.replace(/"/g, '""');
            var escapedText = cellText.replace(/"/g, '""');
            return '=HYPERLINK("' + escapedHref + '","' + escapedText + '")';
        }
    }
};
```

**视图级别的导出选项**（通过 `data-export-options` 属性覆盖）：

```json
{
    "fileName": "activity-report-2026-06-18",
    "ignoreColumn": ["actions","image","change","checkbox","checkincheckout","icon"]
}
```

- `fileName`: 导出文件名（不带扩展名）
- `ignoreColumn`: 导出时忽略的列（通常是操作列、复选框列等非数据列）

### 1.6 前端 8 种导出格式对比

| 格式 | 实现方式 | 特点 | 样式保留 | 适用场景 |
|------|---------|------|---------|---------|
| **xlsx** | tableExport + SheetJS | Excel 原生格式，超链接转 HYPERLINK 公式 | 部分保留 | 正式报表、数据分析 |
| **csv** | tableExport 原生 | 纯文本逗号分隔 | 不保留 | 数据导入、简单导出 |
| **pdf** | jsPDF + AutoTable | 横向布局，自动换行 | 保留表格样式 | 打印、存档 |
| **json** | tableExport 原生 | JSON 数组格式 | 不保留 | 数据交换、API 对接 |
| **xml** | tableExport 原生 | XML 表格格式 | 不保留 | 企业系统数据交换 |
| **txt** | tableExport 原生 | Tab 分隔纯文本 | 不保留 | 简单文本查看 |
| **sql** | tableExport 原生 | INSERT INTO SQL 语句 | 不保留 | 数据库迁移 |
| **doc** | tableExport 原生 | Word HTML 格式 | 部分保留 | 文档编辑 |

**所有前端导出的共同特点**：
- 仅导出当前页面已加载的数据（服务端分页时不是全部数据）
- 导出内容受当前列可见性影响（隐藏列不会被导出，除非配置不同）
- 不经过服务器，浏览器端直接生成下载

---

## 二、后端服务器端导出

### 2.1 后端导出示例总览

后端导出全部为 **CSV 格式**，使用流式响应 `StreamedResponse` 输出，支持大数据量导出。

| 报表类型 | 控制器方法 | 路由 | 是否支持列选择 |
|---------|-----------|------|---------------|
| 自定义资产报表 | [ReportsController::postCustom()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L501-L1172) | `POST /reports/custom` | ✅ 完整支持 |
| 用户列表 | [UsersController::getExportUserCsv()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/Users/UsersController.php#L536-L695) | `GET /users/export` | ❌ 固定列 |
| 活动日志 | [ReportsController::postActivityReport()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L285-L384) | `POST /reports/activity` | ❌ 固定列 |
| 配件报表 | [ReportsController::exportAccessoryReport()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L106-L148) | `GET /reports/export/accessories` | ❌ 固定列（v2 已废弃） |
| 折旧报表 | [ReportsController::exportDeprecationReport()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L174-L248) | `GET /reports/export/depreciation` | ❌ 固定列（v2 已废弃） |
| 许可证报表 | [ReportsController::exportLicenseReport()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L412-L460) | `GET /reports/export/licenses` | ❌ 固定列（v2 已废弃） |
| 维护报表 | [ReportsController::exportMaintenancesReport()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1195-L1251) | `GET /reports/export/maintenances` | ❌ 固定列（v2 已废弃） |
| 待接受资产 | [ReportsController::postAssetAcceptanceReport()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1454-L1524) | `POST /reports/unaccepted_assets` | ❌ 固定列 |

> ⚠️ **注意**：大部分后端导出方法已标记为 `@deprecated`，注释说明 "Server-side exports have been replaced by datatables export since v2."（自 v2 起已被数据表前端导出替代）。但**自定义资产报表（postCustom）是唯一的例外，仍在活跃使用并支持完整列选择**。

### 2.2 自定义资产报表：列选择完整流程

这是后端导出中唯一支持列选择的功能，也是本分析的重点。

#### 2.2.1 前端列选择 UI

位于 [custom.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L92-L345)。

**列选择区域结构**：

```blade
<div class="col-md-3" id="included_fields_wrapper">

    <!-- 全选/取消全选 -->
    <label class="form-control">
        <input type="checkbox" id="checkAll" checked="checked">
        {{ trans('general.select_all') }}
    </label>

    <!-- 基础字段列（每个列都是一个独立的 checkbox，value="1" 表示选中） -->
    <label class="form-control">
        <input type="checkbox" name="id" value="1" @checked($template->checkmarkValue('id')) />
        {{ trans('general.id') }}
    </label>

    <label class="form-control">
        <input type="checkbox" name="asset_tag" value="1" @checked($template->checkmarkValue('asset_tag')) />
        {{ trans('general.asset_tag') }}
    </label>

    <!-- ... 约 50+ 个基础字段 ... -->

    <!-- 用户相关字段组 -->
    <h2>{{ trans('general.checked_out_to_fields') }}:</h2>
    <label class="form-control">
        <input type="checkbox" name="assigned_to" value="1" @checked(...) />
        {{ trans('admin/licenses/table.assigned_to') }}
    </label>
    <!-- username, email, manager, department 等 -->

    <!-- 自定义字段动态生成 -->
    @if ($customfields->count() > 0)
        <h2>{{ trans('admin/custom_fields/general.custom_fields') }}</h2>
        @foreach ($customfields as $customfield)
            <label class="form-control">
                <input type="checkbox" name="{{ $customfield->db_column_name() }}" value="1" @checked(...) />
                {{ $customfield->name }}
            </label>
        @endforeach
    @endif
</div>
```

**全选/取消全选 JS**（[custom.blade.php:901-903](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L901-L903)）：
```javascript
$("#checkAll").change(function () {
    $("#included_fields_wrapper input:checkbox").prop('checked', $(this).prop("checked"));
});
```

#### 2.2.2 表单提交（列选择数据传递到后端）

表单配置（[custom.blade.php:38-45](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L38-L45)）：

```blade
<form
    method="POST"
    action="{{ route('reports.post-custom') }}"
    class="form-horizontal"
    id="custom-report-form"
>
    {{csrf_field()}}
    <!-- 列选择 checkboxes、筛选条件、提交按钮 -->
</form>
```

**数据流向**：
- 用户勾选列 → 每个 checkbox 的 `name` 属性作为字段名，`value="1"` 作为值
- 未勾选的 checkbox **不会被提交**（HTML 标准行为）
- 后端通过 `$request->filled('字段名')` 判断是否选中该列

#### 2.2.3 后端接收与验证

请求通过 [CustomAssetReportRequest](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Requests/CustomAssetReportRequest.php) 进行验证，验证规则主要针对日期范围等筛选条件，**不验证列选择参数**（列选择参数通过 `$request->filled()` 动态判断）。

#### 2.2.4 列参数到导出文件的映射（核心逻辑）

这是最关键的部分，位于 [ReportsController::postCustom()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L501-L1172)。

**阶段一：生成 CSV 表头**（[L519-L728](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L519-L728)）

```php
$header = [];

// 每个列都通过 $request->filled() 判断是否被勾选
if ($request->filled('id')) {
    $header[] = trans('general.id');
}
if ($request->filled('company')) {
    $header[] = trans('general.company');
}
if ($request->filled('asset_name')) {
    $header[] = trans('admin/hardware/form.name');
}
if ($request->filled('asset_tag')) {
    $header[] = trans('admin/hardware/table.asset_tag');
}

// 特殊：某些列对应多个导出列
if ($request->filled('model')) {
    $header[] = trans('admin/hardware/form.model');       // 型号名称
    $header[] = trans('general.model_no');                 // 型号编号
}
if ($request->filled('warranty')) {
    $header[] = trans('admin/hardware/form.warranty');     // 保修月数
    $header[] = trans('admin/hardware/form.warranty_expires'); // 保修到期日
}
// ... 约 50+ 个列判断 ...

// 自定义字段动态处理
foreach ($customfields as $customfield) {
    if ($request->input($customfield->db_column_name()) == '1') {
        $header[] = $customfield->name;
    }
}

fputcsv($handle, $header);
```

**阶段二：构建查询条件**（[L735-L864](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L735-L864)）

```php
$assets = Asset::select('assets.*')->with(
    'location', 'status', 'company', 'defaultLoc', 'assignedTo',
    'model.category', 'model.manufacturer', 'supplier');

// 根据筛选条件动态添加 where 子句
if ($request->filled('by_location_id')) {
    $assets->whereIn('assets.location_id', $request->input('by_location_id'));
}
// ... 众多筛选条件（公司、部门、状态、日期范围等）
```

**阶段三：逐行生成数据**（[L865-L1160](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L865-L1160)）

使用 `chunk(500)` 分批处理，防止内存溢出：

```php
$assets->orderBy('assets.id', 'ASC')->chunk(500, function ($assets) use ($handle, $customfields, $request) {
    $formatter = new EscapeFormula('`');

    foreach ($assets as $asset) {
        $row = [];

        // ⚠️ 顺序必须与表头生成顺序完全一致！
        if ($request->filled('id')) {
            $row[] = ($asset->id) ? $asset->id : '';
        }
        if ($request->filled('company')) {
            $row[] = ($asset->company) ? $asset->company->name : '';
        }
        if ($request->filled('asset_tag')) {
            $row[] = ($asset->asset_tag) ? $asset->asset_tag : '';
        }
        if ($request->filled('model')) {
            $row[] = ($asset->model) ? $asset->model->name : '';
            $row[] = ($asset->model) ? $asset->model->model_number : '';
        }
        // ... 约 50+ 个字段，顺序与表头一一对应 ...

        // 自定义字段处理（含加密字段解密）
        foreach ($customfields as $customfield) {
            $column_name = $customfield->db_column_name();
            if ($request->filled($column_name)) {
                $value = $asset->$column_name;
                // 加密字段解密（需权限）
                if (($customfield->field_encrypted == '1') 
                    && Gate::allows('assets.view.encrypted_custom_fields')) {
                    $value = Helper::gracefulDecrypt($customfield, $value);
                }
                $row[] = $value;
            }
        }

        // CSV 公式注入防护
        if (config('app.escape_formulas') === false) {
            fputcsv($handle, $row);
        } else {
            fputcsv($handle, $formatter->escapeRecord($row));
        }
    }
});
```

**⚠️ 关键设计要点**：
- 表头生成和数据行生成的列判断顺序必须**完全一致**，这是通过代码顺序隐式保证的（非显式映射）
- 某些前端列对应导出中的**多个数据列**（如 `model` → 型号名称 + 型号编号，`warranty` → 保修月数 + 保修到期日）
- 自定义字段通过动态循环处理，自动适配系统中创建的自定义字段

### 2.3 报告模板存储与加载

用户可以将列选择配置保存为报告模板，通过 [ReportTemplate](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Models/ReportTemplate.php) 模型持久化。

**存储逻辑**（[ReportTemplatesController::store()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportTemplatesController.php#L13-L29)）：

```php
$report = $request->user()->reportTemplates()->create([
    'name' => $validated['name'],
    'options' => $request->except(['_token', 'name', 'is_shared']), // 所有表单字段（含列选择）
    'is_shared' => $request->has('is_shared'),
]);
```

**数据结构**：
- `options` 字段以 JSON 格式存储整个表单数据（包括列勾选状态和所有筛选条件）
- 列勾选状态：选中的列值为 `"1"`，未选中的列不存在于数组中
- 全局作用域：用户只能看到自己创建的或共享的模板

**模板回显**（[ReportTemplate::checkmarkValue()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Models/ReportTemplate.php#L76-L91)）：

```php
public function checkmarkValue(string $fieldName, string $fallbackValue = '1'): string
{
    // 新建模板（空对象）：默认全部勾选（返回 fallbackValue='1'）
    if (is_null($this->id)) {
        return $fallbackValue;
    }
    // 已保存模板：读取存储的值，未勾选的列不存在，返回 '0'
    return $this->options[$fieldName] ?? '0';
}
```

### 2.4 后端 CSV 导出安全机制

**CSV 公式注入防护**（所有后端导出统一实现）：

```php
use League\Csv\EscapeFormula;

$formatter = new EscapeFormula('`');
// ...
if (config('app.escape_formulas') === false) {
    fputcsv($handle, $row);
} else {
    fputcsv($handle, $formatter->escapeRecord($row));
}
```

- 配置项 `CSV_ESCAPE_FORMULAS`（默认 true）控制是否启用
- 以 `=`, `+`, `-`, `@` 开头的单元格内容会被添加反引号前缀
- 防止恶意数据通过 Excel 公式执行攻击

**BOM 头**（仅自定义报表）：
```php
fprintf($handle, chr(0xEF).chr(0xBB).chr(0xBF));  // UTF-8 BOM，解决 Excel 中文乱码
```

---

## 三、两种导出体系对比总结

| 维度 | 前端 Bootstrap Table 导出 | 后端服务器端 CSV 导出 |
|------|------------------------|---------------------|
| **数据范围** | 当前页面已加载数据（受分页影响） | 全量数据（所有符合条件的记录） |
| **列选择方式** | Bootstrap Table showColumns 下拉菜单 | 独立页面 checkbox 列表 |
| **列选择持久化** | Cookie/localStorage（2 年） | ReportTemplate 模型（数据库） |
| **支持格式** | xlsx, csv, pdf, json, xml, txt, sql, doc | 仅 csv |
| **大数据量性能** | 受浏览器内存限制 | 流式响应，支持 10 万+ 行 |
| **自定义字段支持** | 动态生成 `<th>` 列 | 动态循环处理 |
| **加密字段** | API 返回已解密数据（权限控制） | 后端实时解密（权限控制） |
| **公式注入防护** | 依赖前端插件 | `League\Csv\EscapeFormula` |
| **超链接处理** | 转 Excel HYPERLINK 公式 | 纯文本 URL |
| **筛选条件** | 前端搜索 + 后端 API 过滤 | 丰富的后端筛选条件 |
| **当前状态** | 推荐方式（v2 默认） | 仅自定义报表活跃，其余已废弃 |

---

## 四、关键文件索引

| 文件 | 作用 |
|------|------|
| [app/Http/Controllers/ReportsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php) | 所有报表后端导出控制器 |
| [app/Http/Controllers/ReportTemplatesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportTemplatesController.php) | 报告模板 CRUD |
| [app/Http/Controllers/Users/UsersController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/Users/UsersController.php) | 用户列表 CSV 导出 |
| [app/Http/Requests/CustomAssetReportRequest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Requests/CustomAssetReportRequest.php) | 自定义报表请求验证 |
| [app/Models/ReportTemplate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Models/ReportTemplate.php) | 报告模板 Eloquent 模型 |
| [app/Presenters/DepreciationReportPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/DepreciationReportPresenter.php) | 折旧报表列配置示例 |
| [app/Presenters/*.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/) | 60+ 个 Presenter，每个定义对应表格的列配置 |
| [resources/views/partials/bootstrap-table.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/partials/bootstrap-table.blade.php) | Bootstrap Table 初始化与导出配置核心 JS |
| [resources/views/reports/custom.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php) | 自定义报表前端页面（列选择 UI） |
| [resources/views/reports/activity.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/activity.blade.php) | 活动报表视图（前端导出示例） |
| [resources/views/reports/asset.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/asset.blade.php) | 资产报表视图（传统 th 列配置示例） |
| [routes/web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/routes/web.php#L465-L587) | 报表相关路由定义 |
| [routes/web/users.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/routes/web/users.php#L29-L35) | 用户导出路由 |
