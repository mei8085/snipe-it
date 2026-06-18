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
- 仅导出**当前页面**已加载的数据（服务端分页时不是全部数据），项目未配置 `exportDataType` 参数，默认值为 `basic`（当前页）
- 导出内容受当前列可见性影响（隐藏列不会被导出，除非配置不同）
- 不经过服务器，浏览器端直接生成下载

### 1.7 前端导出数据范围核对

**核心发现：前端导出默认只导出当前页数据**

项目代码中**未配置** `exportDataType` 参数（Bootstrap Table 导出插件的核心参数），因此使用默认值 `basic`，即只导出当前分页显示的数据。

| 配置项 | 值 | 说明 |
|-------|----|------|
| `exportDataType` | 未设置（默认 `basic`） | 只导出当前页数据 |
| `pageSize` | 20（默认，可配置） | 每页显示行数 |
| `pageList` | ['10','20','30','50','100','150','200'...] | 可选每页行数 |

**如果需要导出全部数据**，需要配置 `exportDataType: 'all'`，但这会触发全部数据加载，可能影响性能。

**8 种格式导出范围一致性**：所有 8 种格式（xlsx、csv、pdf、json、xml、txt、sql、doc）都使用相同的数据范围，即当前页已加载数据，区别仅在于输出格式不同。

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

#### 2.2.1 前端列选择 UI 与列配置数量核准

位于 [custom.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L92-L345)。

**列配置数量精确定义**：

前端页面共有 **49 个 checkbox**，按功能可分为 6 类：

| 类别 | 数量 | 字段名示例 | 后端是否读取 |
|------|------|-----------|-------------|
| 全选交互按钮 | 1 | `checkAll` (无 name) | ❌ 纯前端 JS |
| 资产基础字段 | 30 | `id`, `asset_tag`, `model`, `purchase_date` 等 | ✅ 全部读取 |
| 用户相关字段 | 15 | `assigned_to`, `username`, `email`, `department` 等 | ✅ 全部读取 |
| 自定义字段 | 动态 N | `_snipeit_*` | ✅ 动态读取 |
| 模板共享选项 | 1 | `is_shared` | ⚠️ 表头读取，数据行未实现 |
| 筛选控制选项 | 1 | `exclude_archived` | ✅ 用于查询，不导出 |
| CSV 编码选项 | 1 | `use_bom` | ❌ **完全不读取** |

**46 个后端读取字段（30+15+1）完整清单与导出映射**：

| 序号 | 前端 checkbox (name) | 后端表头代码位置 | 数据行代码位置 | 导出门列数 |
|------|---------------------|-------------|---------------|-----------|
| **1** | `is_shared` | [L521-L523](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L521-L523) | **未实现** [L730-L733](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L730-L733) | 1 |
| **2** | `id` | [L525-L527](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L525-L527) | [L877-L879](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L877-L879) | 1 |
| **3** | `company` | [L529-L531](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L529-L531) | [L881-L883](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L881-L883) | 1 |
| **4** | `asset_name` | [L533-L535](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L533-L535) | [L885-L887](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L885-L887) | 1 |
| **5** | `asset_tag` | [L537-L539](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L537-L539) | [L889-L891](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L889-L891) | 1 |
| **6** | `model` | [L541-L544](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L541-L544) | [L893-L896](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L893-L896) | **2** |
| **7** | `category` | [L546-L548](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L546-L548) | [L898-L900](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L898-L900) | 1 |
| **8** | `manufacturer` | [L550-L552](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L550-L552) | [L902-L904](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L902-L904) | 1 |
| **9** | `serial` | [L554-L556](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L554-L556) | [L906-L908](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L906-L908) | 1 |
| **10** | `purchase_date` | [L557-L559](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L557-L559) | [L910-L912](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L910-L912) | 1 |
| **11** | `purchase_cost` | [L561-L563](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L561-L563) | [L914-L916](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L914-L916) | 1 |
| **12** | `eol` | [L565-L567](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L565-L567) | [L918-L920](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L918-L920) | 1 |
| **13** | `warranty` | [L569-L572](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L569-L572) | [L922-L925](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L922-L925) | **2** |
| **14** | `depreciation` | [L574-L578](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L574-L578) | [L927-L933](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L927-L933) | **3** |
| **15** | `order` | [L580-L582](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L580-L582) | [L935-L937](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L935-L937) | 1 |
| **16** | `supplier` | [L584-L586](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L584-L586) | [L939-L941](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L939-L941) | 1 |
| **17** | `location` | [L588-L590](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L588-L590) | [L943-L945](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L943-L945) | 1 |
| **18** | `location_address` | [L591-L598](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L591-L598) | [L947-L954](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L947-L954) | **6** |
| **19** | `rtd_location` | [L600-L602](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L600-L602) | [L956-L958](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L956-L958) | 1 |
| **20** | `rtd_location_address` | [L604-L611](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L604-L611) | [L960-L967](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L960-L967) | **6** |
| **21** | `assigned_to` | [L613-L616](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L613-L616) | [L969-L972](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L969-L972) | **2** |
| **22** | `username` | [L618-L620](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L618-L620) | [L974-L981](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L974-L981) | 1 |
| **23** | `user_company` | [L622-L624](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L622-L624) | [L983-L989](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L983-L989) | 1 |
| **24** | `email` | [L626-L628](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L626-L628) | [L991-L998](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L991-L998) | 1 |
| **25** | `employee_num` | [L630-L632](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L630-L632) | [L1000-L1007](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1000-L1007) | 1 |
| **26** | `manager` | [L634-L636](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L634-L636) | [L1009-L1015](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1009-L1015) | 1 |
| **27** | `department` | [L638-L640](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L638-L640) | [L1017-L1023](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1017-L1023) | 1 |
| **28** | `title` | [L642-L644](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L642-L644) | [L1025-L1031](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1025-L1031) | 1 |
| **29** | `phone` | [L646-L648](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L646-L648) | [L1033-L1039](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1033-L1039) | 1 |
| **30** | `user_address` | [L650-L652](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L650-L652) | [L1041-L1047](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1041-L1047) | 1 |
| **31** | `user_city` | [L654-L656](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L654-L656) | [L1049-L1055](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1049-L1055) | 1 |
| **32** | `user_state` | [L658-L660](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L658-L660) | [L1057-L1063](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1057-L1063) | 1 |
| **33** | `user_country` | [L662-L664](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L662-L664) | [L1065-L1071](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1065-L1071) | 1 |
| **34** | `user_zip` | [L666-L668](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L666-L668) | [L1073-L1079](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1073-L1079) | 1 |
| **35** | `target_notes` | [L670-L672](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L670-L672) | [L1081-L1087](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1081-L1087) | 1 |
| **36** | `status` | [L674-L676](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L674-L676) | [L1089-L1091](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1089-L1091) | 1 |
| **37** | `checkout_date` | [L678-L680](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L678-L680) | [L1093-L1095](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1093-L1095) | 1 |
| **38** | `checkin_date` | [L682-L684](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L682-L684) | [L1097-L1101](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1097-L1101) | 1 |
| **39** | `expected_checkin` | [L686-L688](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L686-L688) | [L1103-L1105](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1103-L1105) | 1 |
| **40** | `created_at` | [L690-L692](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L690-L692) | [L1107-L1109](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1107-L1109) | 1 |
| **41** | `updated_at` | [L694-L696](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L694-L696) | [L1111-L1113](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1111-L1113) | 1 |
| **42** | `deleted_at` | [L698-L700](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L698-L700) | [L1115-L1117](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1115-L1117) | 1 |
| **43** | `last_audit_date` | [L702-L704](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L702-L704) | [L1119-L1121](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1119-L1121) | 1 |
| **44** | `next_audit_date` | [L706-L708](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L706-L708) | [L1123-L1125](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1123-L1125) | 1 |
| **45** | `notes` | [L710-L712](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L710-L712) | [L1127-L1129](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1127-L1129) | 1 |
| **46** | `url` | [L714-L716](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L714-L716) | [L1131-L1133](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1131-L1133) | 1 |
| **47** | 自定义字段 | [L718-L722](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L718-L722) | [L1135-L1146](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1135-L1146) | N |

> ⚠️ **列配置数量澄清**：
> - 前端 checkbox 总数：49 个（含交互、筛选、编码选项）
> - 后端实际读取的字段：46 个（is_shared + 45 个数据列）+ 自定义字段
> - 全部勾选时的最大导出列数：约 **75+ 列**（因为存在一对多映射：model→2, warranty→2, depreciation→3, location_address→6, rtd_location_address→6, assigned_to→2）

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

    <!-- ... 共 30 个基础字段 ... -->

    <!-- 用户相关字段组 -->
    <h2>{{ trans('general.checked_out_to_fields') }}:</h2>
    <label class="form-control">
        <input type="checkbox" name="assigned_to" value="1" @checked(...) />
        {{ trans('admin/licenses/table.assigned_to') }}
    </label>
    <!-- username, email, manager, department 等 15 个用户字段 -->

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


### 2.5 共享模板(is_shared) 与地址类字段映射核准

#### 2.5.1 共享模板勾选(is_shared) 映射分析

**问题描述**：`is_shared` 列存在表头数据不一致的严重 Bug。

| 环节 | 处理逻辑 | 代码位置 |
|------|---------|---------|
| 前端 UI | 独立 checkbox（不在 included_fields_wrapper 内） | [custom.blade.php:80](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L80) |
| 表头生成 | 添加 "Share template" 列 | [L521-L523](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L521-L523) |
| 数据行生成 | 未实现（只有 TODO 注释） | [L730-L733](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L730-L733) |

**代码证据**：

表头（L521-L523）：
```php
if ($request->filled('is_shared')) {
    $header[] = trans('admin/reports/general.share_template');
}
```

数据行（L730-L733）：
```php
if ($request->filled('is_shared')) {
    // to fill with logic for the report template and NOT the assets retrieved by the query
    // do we scope here or??
}
```

**后果**：当 `is_shared` 被勾选时，所有列向右偏移 1 列：
- 表头第 1 列："Share template"
- 数据第 1 列：资产 ID（本应是 is_shared 的值）
- 表头第 2 列："ID"
- 数据第 2 列：公司名称（本应是 ID）
- 依此类推，全部错位

#### 2.5.2 地址类字段映射分析

地址类字段包括 4 个 checkbox，对应不同的映射关系：

| 前端 checkbox | 导出门列数 | 表头文本 | 对应数据字段 |
|--------------|-----------|---------|-------------|
| `location` | 1 | "Location" | `$asset->location->display_name` |
| `location_address` | 6 | "Address", "Address", "City", "State", "Country", "Zip" | `address`, `address2`, `city`, `state`, `country`, `zip` |
| `rtd_location` | 1 | "Default Location" | `$asset->defaultLoc->display_name` |
| `rtd_location_address` | 6 | "Address", "Address", "City", "State", "Country", "Zip" | 同上 |

**`location_address` 映射详情**：

表头代码 [L591-L598](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L591-L598)：
```php
if ($request->filled('location_address')) {
    $header[] = trans('general.address');   // 列 1: Address
    $header[] = trans('general.address');   // 列 2: Address - 重复
    $header[] = trans('general.city');      // 列 3: City
    $header[] = trans('general.state');     // 列 4: State
    $header[] = trans('general.country');   // 列 5: Country
    $header[] = trans('general.zip');       // 列 6: Zip
}
```

数据行代码 [L947-L954](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L947-L954)：
```php
if ($request->filled('location_address')) {
    $row[] = ($asset->location) ? $asset->location->address : '';   // 对应列 1
    $row[] = ($asset->location) ? $asset->location->address2 : '';  // 对应列 2
    $row[] = ($asset->location) ? $asset->location->city : '';      // 对应列 3
    $row[] = ($asset->location) ? $asset->location->state : '';     // 对应列 4
    $row[] = ($asset->location) ? $asset->location->country : '';   // 对应列 5
    $row[] = ($asset->location) ? $asset->location->zip : '';       // 对应列 6
}
```

**问题**：前两列表头都是 "Address"，用户无法区分哪列是 `address`（地址行 1），哪列是 `address2`（地址行 2）。

`rtd_location_address` 存在完全相同的问题 [L604-L611](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L604-L611)。

#### 2.5.3 所有一对多列映射完整清单

| 前端 checkbox | 导出门列数 | 导出列明细 |
|--------------|-----------|-----------|
| `model` | 2 | 型号名称、型号编号 |
| `warranty` | 2 | 保修月数、保修到期日 |
| `depreciation` | 3 | 账面价值、差额、完全折旧日期 |
| `location_address` | 6 | Address, Address, City, State, Country, Zip |
| `rtd_location_address` | 6 | Address, Address, City, State, Country, Zip |
| `assigned_to` | 2 | 借用人名称、借用人类型 |

**全部勾选时的列数计算**：
- 49 个前端 checkbox / 46 个后端读取字段（含 is_shared）
- + 1（model 多 1 列）
- + 1（warranty 多 1 列）
- + 2（depreciation 多 2 列）
- + 5（location_address 多 5 列）
- + 5（rtd_location_address 多 5 列）
- + 1（assigned_to 多 1 列）
- + N（自定义字段）
- 总计：约 62 + N 列

---


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

## 四、已知代码缺陷

按严重程度排序：

### 4.1 is_shared 列错位（严重）

- **问题**：表头有 is_shared 列但数据行未实现，导致勾选时所有列向右偏移 1 列
- **代码位置**：[L521-L523](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L521-L523)（表头）, [L730-L733](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L730-L733)（数据行）
- **影响**：勾选 is_shared 时，所有导出数据列错位 1 列

### 4.2 location_address 表头重复

- **问题**：两个 "Address" 列标题相同，无法区分 address（地址行1）和 address2（地址行2）
- **代码位置**：[L591-L598](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L591-L598)
- **影响**：导出 CSV 中两列标题相同，用户无法区分

### 4.3 rtd_location_address 表头重复

- **问题**：同上
- **代码位置**：[L604-L611](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L604-L611)

### 4.4 use_bom 复选框无效

- **问题**：UI 存在 use_bom 复选框，但后端完全不读取该参数，BOM 始终硬编码输出
- **代码位置**：[custom.blade.php:627](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L627)（UI）, [L517](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L517)（硬编码）
- **影响**：用户勾选无效，功能名存实亡

---

## 五、导出数量统计：代码级精确核准

### 5.1 前端 checkbox 精确统计

[custom.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php) 中共有 **49 个固定 checkbox** + N 个动态自定义字段 checkbox。

**按位置与用途分类**：

| 分类 | 数量 | 位置 | 用途 |
|------|------|------|------|
| 资产字段 checkbox | 31 | `#included_fields_wrapper` 内 [L100-L247](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L100-L247) | 控制资产数据列导出 |
| 用户字段 checkbox | 15 | `#included_fields_wrapper` 内 [L255-L328](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L255-L328) | 控制借用人数据列导出 |
| 模板共享 checkbox | 1 | `#included_fields_wrapper` 外 [L80](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L80) | `is_shared`（仅在编辑模板时显示） |
| 排除归档 checkbox | 1 | `#included_fields_wrapper` 外 [L621](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L621) | `exclude_archived`（筛选参数，不产生导出列） |
| BOM 选项 checkbox | 1 | `#included_fields_wrapper` 外 [L627](file:///d:/fz/0601-2/solo-dogfeeding/code/26/snipe-it/resources/views/reports/custom.blade.php#L627) | `use_bom`（后端不读取，硬编码输出） |
| 全选控制 checkbox | 1 | `#included_fields_wrapper` 内 [L95](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L95) | `checkAll`（JS 控制，不提交数据） |
| 动态自定义字段 | N | `#included_fields_wrapper` 内 [L336-L343](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L336-L343) | `@foreach $customfields` 循环生成 |

**49 个固定 checkbox 的完整清单**：

| 序号 | name 属性 | 显示标签 | 所属分类 | 是否产生导出列 |
|------|----------|---------|---------|--------------|
| 1 | `is_shared` | Share template | 模板选项 | 有表头无数据（Bug） |
| 2 | `id` | ID | 资产字段 | 1 列 |
| 3 | `company` | Company | 资产字段 | 1 列 |
| 4 | `asset_tag` | Asset Tag | 资产字段 | 1 列 |
| 5 | `asset_name` | Name | 资产字段 | 1 列 |
| 6 | `manufacturer` | Manufacturer | 资产字段 | 1 列 |
| 7 | `model` | Model | 资产字段 | **2 列**（名称+编号） |
| 8 | `category` | Category | 资产字段 | 1 列 |
| 9 | `serial` | Serial | 资产字段 | 1 列 |
| 10 | `purchase_date` | Purchase Date | 资产字段 | 1 列 |
| 11 | `purchase_cost` | Purchase Cost | 资产字段 | 1 列 |
| 12 | `eol` | EOL Date | 资产字段 | 1 列 |
| 13 | `warranty` | Warranty | 资产字段 | **2 列**（月数+到期日） |
| 14 | `depreciation` | Depreciation | 资产字段 | **3 列**（账面价值+差额+折旧日期） |
| 15 | `order` | Order | 资产字段 | 1 列 |
| 16 | `supplier` | Supplier | 资产字段 | 1 列 |
| 17 | `location` | Location | 资产字段 | 1 列 |
| 18 | `location_address` | Address | 资产字段 | **6 列**（address,address2,city,state,country,zip） |
| 19 | `rtd_location` | Default Location | 资产字段 | 1 列 |
| 20 | `rtd_location_address` | Address | 资产字段 | **6 列**（同上） |
| 21 | `status` | Status | 资产字段 | 1 列 |
| 22 | `checkout_date` | Checkout Date | 资产字段 | 1 列 |
| 23 | `checkin_date` | Last Checkin Date | 资产字段 | 1 列 |
| 24 | `expected_checkin` | Expected Checkin | 资产字段 | 1 列 |
| 25 | `created_at` | Created At | 资产字段 | 1 列 |
| 26 | `updated_at` | Updated At | 资产字段 | 1 列 |
| 27 | `deleted_at` | Deleted | 资产字段 | 1 列 |
| 28 | `last_audit_date` | Last Audit | 资产字段 | 1 列 |
| 29 | `next_audit_date` | Next Audit Date | 资产字段 | 1 列 |
| 30 | `notes` | Notes | 资产字段 | 1 列 |
| 31 | `url` | URL | 资产字段 | 1 列 |
| 32 | `assigned_to` | Assigned To | 用户字段 | **2 列**（名称+类型） |
| 33 | `username` | Username | 用户字段 | 1 列 |
| 34 | `user_company` | User Company | 用户字段 | 1 列 |
| 35 | `email` | Email | 用户字段 | 1 列 |
| 36 | `employee_num` | Employee No. | 用户字段 | 1 列 |
| 37 | `manager` | Manager | 用户字段 | 1 列 |
| 38 | `department` | Department | 用户字段 | 1 列 |
| 39 | `title` | Job Title | 用户字段 | 1 列 |
| 40 | `phone` | Phone | 用户字段 | 1 列 |
| 41 | `user_address` | Address | 用户字段 | 1 列 |
| 42 | `user_city` | City | 用户字段 | 1 列 |
| 43 | `user_state` | State | 用户字段 | 1 列 |
| 44 | `user_country` | Country | 用户字段 | 1 列 |
| 45 | `user_zip` | Zip | 用户字段 | 1 列 |
| 46 | `target_notes` | Target Notes | 用户字段 | 1 列 |
| 47 | `exclude_archived` | Exclude Archived | 筛选选项 | **0 列**（不产生导出列） |
| 48 | `use_bom` | BOM | 选项 | **0 列**（后端不读取） |
| 49 | `checkAll` | Select All | 控制按钮 | **0 列**（JS 控制） |

### 5.2 后端 `$request->filled()` 字段精确统计

[ReportsController.php::postCustom()](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L501-L1172) 中用于**表头生成**的 `$request->filled()` 调用共 **46 次**（L521-L714）。

这 46 个字段分为两大类：

| 分类 | 数量 | 行号范围 | 说明 |
|------|------|---------|------|
| 资产列字段 | 30 | L521-L714 | 含 is_shared（1列）、1:1 映射（24列）、1:N 映射（5个 → 15列） |
| 用户列字段 | 16 | L613-L672 | 含 assigned_to（1:2 映射）、1:1 映射（15列） |

**46 个字段的精确清单（按代码顺序）**：

| # | 字段名 | 代码行 | 一对多 | 导出列数 | 表头文本 |
|---|--------|-------|--------|---------|---------|
| 1 | `is_shared` | L521 | - | 1 | Share template |
| 2 | `id` | L525 | - | 1 | ID |
| 3 | `company` | L529 | - | 1 | Company |
| 4 | `asset_name` | L533 | - | 1 | Name |
| 5 | `asset_tag` | L537 | - | 1 | Asset Tag |
| 6 | `model` | L541 | 1:2 | 2 | Model, Model No. |
| 7 | `category` | L546 | - | 1 | Category |
| 8 | `manufacturer` | L550 | - | 1 | Manufacturer |
| 9 | `serial` | L554 | - | 1 | Serial |
| 10 | `purchase_date` | L557 | - | 1 | Purchase Date |
| 11 | `purchase_cost` | L561 | - | 1 | Purchase Cost |
| 12 | `eol` | L565 | - | 1 | EOL |
| 13 | `warranty` | L569 | 1:2 | 2 | Warranty, Warranty Expires |
| 14 | `depreciation` | L574 | 1:3 | 3 | Book Value, Diff, Fully Depreciated |
| 15 | `order` | L580 | - | 1 | Order |
| 16 | `supplier` | L584 | - | 1 | Supplier |
| 17 | `location` | L588 | - | 1 | Location |
| 18 | `location_address` | L591 | 1:6 | 6 | Address, Address, City, State, Country, Zip |
| 19 | `rtd_location` | L600 | - | 1 | Default Location |
| 20 | `rtd_location_address` | L604 | 1:6 | 6 | Address, Address, City, State, Country, Zip |
| 21 | `assigned_to` | L613 | 1:2 | 2 | Checked Out To, Type |
| 22 | `username` | L618 | - | 1 | Username |
| 23 | `user_company` | L622 | - | 1 | User Company |
| 24 | `email` | L626 | - | 1 | Email |
| 25 | `employee_num` | L630 | - | 1 | Employee No. |
| 26 | `manager` | L634 | - | 1 | Manager |
| 27 | `department` | L638 | - | 1 | Department |
| 28 | `title` | L642 | - | 1 | Job Title |
| 29 | `phone` | L646 | - | 1 | Phone |
| 30 | `user_address` | L650 | - | 1 | User Address |
| 31 | `user_city` | L654 | - | 1 | User City |
| 32 | `user_state` | L658 | - | 1 | User State |
| 33 | `user_country` | L662 | - | 1 | User Country |
| 34 | `user_zip` | L666 | - | 1 | User Zip |
| 35 | `target_notes` | L670 | - | 1 | Target Notes |
| 36 | `status` | L674 | - | 1 | Status |
| 37 | `checkout_date` | L678 | - | 1 | Checkout Date |
| 38 | `checkin_date` | L682 | - | 1 | Last Checkin Date |
| 39 | `expected_checkin` | L686 | - | 1 | Expected Checkin |
| 40 | `created_at` | L690 | - | 1 | Created At |
| 41 | `updated_at` | L694 | - | 1 | Updated At |
| 42 | `deleted_at` | L698 | - | 1 | Deleted |
| 43 | `last_audit_date` | L702 | - | 1 | Last Audit |
| 44 | `next_audit_date` | L706 | - | 1 | Next Audit Date |
| 45 | `notes` | L710 | - | 1 | Notes |
| 46 | `url` | L714 | - | 1 | URL |

**后端不读取的 3 个前端 checkbox**：

| 前端 checkbox | 不读取原因 | 实际作用 |
|--------------|-----------|---------|
| `checkAll` | 不是数据字段 | JS 控制 `#included_fields_wrapper` 内所有 checkbox |
| `exclude_archived` | 用作筛选参数 | L846: `$assets->notArchived()` |
| `use_bom` | 后端完全忽略 | BOM 硬编码输出，不读取请求参数 |

### 5.3 动态自定义字段的代码逻辑与数量来源

自定义字段的数量是**运行时动态**决定的，无法在代码中硬编码一个数字。

**数据来源链路**：

```
数据库 custom_fields 表
        ↓
CustomField::get() [L508]
        ↓
$customfields 变量传递给 StreamedResponse 闭包
        ↓
表头：foreach ($customfields as $customfield) [L718-L722]
  → $request->input($customfield->db_column_name()) == '1' 判断
  → $header[] = $customfield->name
        ↓
数据行：foreach ($customfields as $customfield) [L1135-L1146]
  → $request->filled($customfield->db_column_name()) 判断
  → $row[] = $asset->$column_name（加密字段需 Gate 权限 + 解密）
```

**关键代码**：

表头生成 [L718-L722](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L718-L722)：
```php
foreach ($customfields as $customfield) {
    if ($request->input($customfield->db_column_name()) == '1') {
        $header[] = $customfield->name;
    }
}
```

数据行生成 [L1135-L1146](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L1135-L1146)：
```php
foreach ($customfields as $customfield) {
    $column_name = $customfield->db_column_name();
    if ($request->filled($customfield->db_column_name())) {
        $value = $asset->$column_name;
        if (($customfield->field_encrypted == '1') && Gate::allows('assets.view.encrypted_custom_fields')) {
            $value = Helper::gracefulDecrypt($customfield, $value);
        }
        $row[] = $value;
    }
}
```

**前端动态生成** [L332-L343](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php#L332-L343)：
```blade
@if ($customfields->count() > 0)
    <h2>{{ trans('admin/custom_fields/general.custom_fields') }}</h2>
    @foreach ($customfields as $customfield)
        <label class="form-control">
            <input type="checkbox" name="{{ $customfield->db_column_name() }}" value="1"
                   @checked($template->checkmarkValue($customfield->db_column_name())) />
            {{ $customfield->name }}
        </label>
    @endforeach
@endif
```

**注意差异**：前端 checkbox 使用 `name="{{ $customfield->db_column_name() }}"`，其中 `db_column_name()` 返回的是 `db_column` 属性值（如 `_snipeit_mac_address_1`），而非字段 ID。

### 5.4 Presenter 定义文件的列配置数量

Presenter 列配置用于**前端 Bootstrap Table 导出**，与后端 CSV 导出是两套独立体系。

**AssetPresenter（资产列表）** — 共 **30 个固定列 + N 个动态自定义字段列 + 2 个操作列**：

| 列类型 | 数量 | field 名称 |
|--------|------|-----------|
| 功能列 | 2 | checkbox, checkincheckout |
| 操作列 | 1 | actions |
| 资产固定列 | 27 | id, asset_tag, name, company, image, serial, model, model_number, category, status, assigned_to, employee_number, jobtitle, location, rtd_location, manufacturer, supplier, purchase_date, age, purchase_cost, book_value, order_number, eol, asset_eol_date, warranty_months, warranty_expires, requestable |
| 扩展列 | 4 | notes, checkout_counter, checkin_counter, requests_counter |
| 管理列 | 2 | created_by, created_at |
| 时间列 | 5 | updated_at, deleted_at(条件), last_checkout, last_checkin, expected_checkin |
| 审计列 | 2 | last_audit_date, next_audit_date |
| 标志列 | 1 | byod |
| 自定义字段 | N | 动态（CustomField::whereHas('fieldset', ...)->get()） |

**DepreciationReportPresenter（折旧报表）** — 共 **22 个固定列**：

field: company, category, name, asset_tag, model, model_number, serial, depreciation, number_of_months, status, checked_out_to, location, manufacturer, supplier, purchase_date, currency, purchase_cost, order_number, eol, book_value, monthly_depreciation, diff, warranty_expires

**其他 Presenter 文件列表**（29 个，每个含 `dataTableLayout()` 方法定义各自的列配置）：

| Presenter 文件 | 对应功能 |
|---------------|---------|
| [AssetPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/AssetPresenter.php) | 资产列表 |
| [DepreciationReportPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/DepreciationReportPresenter.php) | 折旧报表 |
| [AccessoryPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/AccessoryPresenter.php) | 配件列表 |
| [AssetAuditPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/AssetAuditPresenter.php) | 资产审计 |
| [AssetModelPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/AssetModelPresenter.php) | 资产型号 |
| [CategoryPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/CategoryPresenter.php) | 分类 |
| [CompanyPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/CompanyPresenter.php) | 公司 |
| [ComponentPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/ComponentPresenter.php) | 组件 |
| [ConsumablePresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/ConsumablePresenter.php) | 耗材 |
| [CustomFieldPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/CustomFieldPresenter.php) | 自定义字段 |
| [CustomFieldsetPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/CustomFieldsetPresenter.php) | 自定义字段集 |
| [DepartmentPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/DepartmentPresenter.php) | 部门 |
| [DepreciationPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/DepreciationPresenter.php) | 折旧 |
| [GroupPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/GroupPresenter.php) | 用户组 |
| [HistoryPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/HistoryPresenter.php) | 活动日志 |
| [LabelPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/LabelPresenter.php) | 标签 |
| [LicensePresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/LicensePresenter.php) | 许可证 |
| [LicenseSeatPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/LicenseSeatPresenter.php) | 许可证席位 |
| [LocationPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/LocationPresenter.php) | 位置 |
| [MaintenancesPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/MaintenancesPresenter.php) | 维护 |
| [MaintenanceTypePresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/MaintenanceTypePresenter.php) | 维护类型 |
| [ManufacturerPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/ManufacturerPresenter.php) | 制造商 |
| [PredefinedKitPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/PredefinedKitPresenter.php) | 预定义套件 |
| [StatusLabelPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/StatusLabelPresenter.php) | 状态标签 |
| [SupplierPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/SupplierPresenter.php) | 供应商 |
| [UserPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/UserPresenter.php) | 用户 |
| [UploadedFilesPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/UploadedFilesPresenter.php) | 上传文件 |
| [ActionlogPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/ActionlogPresenter.php) | 操作日志 |

### 5.5 全部勾选时的最终导出列数计算

**后端 CSV 导出（自定义报表）全部勾选时的列数**：

| 计算项 | 数量 | 说明 |
|--------|------|------|
| 一对一映射字段 | 39 | 46 - 6（一对多）- 1（is_shared 有表头无数据） |
| `model` 多出的列 | +1 | model_number |
| `warranty` 多出的列 | +1 | warranty_expires |
| `depreciation` 多出的列 | +2 | diff + fully_depreciated |
| `location_address` 多出的列 | +5 | address2 + city + state + country + zip |
| `rtd_location_address` 多出的列 | +5 | 同上 |
| `assigned_to` 多出的列 | +1 | type |
| `is_shared` | +1 | 有表头列但无数据（Bug：导致错位） |
| 动态自定义字段 | +N | 由 `CustomField::get()` 数量决定 |

**精确计算公式**：

```
导出列数 = 39(1:1) + 1(model额外) + 1(warranty额外) + 2(depreciation额外)
         + 5(location_address额外) + 5(rtd_location_address额外) 
         + 1(assigned_to额外) + 1(is_shared表头) + N(自定义字段)
         = 39 + 15 + 1(is_shared) + N
         = 55 + N
```

**但需注意 is_shared Bug**：由于 `is_shared` 在数据行中未实现（[L730-L733](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L730-L733)），实际数据行只有 54 + N 列，表头有 55 + N 列，**差 1 列导致全部数据错位**。

**不勾选 is_shared 时的正确列数**：

```
导出列数 = 54 + N（表头与数据行完全对齐）
```

### 5.6 数量统计总结对照表

| 统计维度 | 数量 | 来源代码 |
|---------|------|---------|
| 前端固定 checkbox | 49 | [custom.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/resources/views/reports/custom.blade.php) |
| 前端数据列 checkbox | 46 | 49 - 1(checkAll) - 1(exclude_archived) - 1(use_bom) |
| 前端资产字段 checkbox | 31 | L100-L247 |
| 前端用户字段 checkbox | 15 | L255-L328 |
| 前端动态自定义字段 checkbox | N | L336-L343（@foreach） |
| 后端表头 `$request->filled()` | 46 | [ReportsController.php L521-L714](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Http/Controllers/ReportsController.php#L521-L714) |
| 后端数据行 `$request->filled()` | 45 | 同上，排除 is_shared（L730-L733 空实现） |
| 后端筛选条件 `$request->filled()` | 17+ | L739-L846（by_location_id 等，不产生导出列） |
| 一对多映射字段 | 6 | model, warranty, depreciation, location_address, rtd_location_address, assigned_to |
| 一对多额外列数合计 | 15 | 1+1+2+5+5+1 |
| 全部勾选导出列数（不含 is_shared） | 54 + N | 39(1:1) + 15(1:N额外) + N |
| 全部勾选导出列数（含 is_shared） | 55 + N（表头错位） | 表头 55+N 列，数据 54+N 列 |
| AssetPresenter 固定列 | 30+2 操作 | [AssetPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/AssetPresenter.php) |
| DepreciationReportPresenter 列 | 22 | [DepreciationReportPresenter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/DepreciationReportPresenter.php) |
| Presenter 文件总数 | 29 | [app/Presenters/](file:///d:/fz/0601-2/solo-dogfeeding/code/26-snipe-it/app/Presenters/) |

## 六、关键文件索引

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

