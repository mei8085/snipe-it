# 资产采购记录中供应商与制造商关系 - 代码理解材料

> **可复核声明**：本材料所有结论均对应具体代码位置，引用格式为 `项目根目录下的相对路径#行号范围`，可在代码仓库中直接定位验证。项目根目录为 `16-snipe-it/`。

---

## 一、领域模型文件清单

涉及的核心领域模型文件位于 `app/Models/` 目录下：

| 模型 | 文件路径 | 职责 |
|-----|---------|-----|
| Supplier | `app/Models/Supplier.php` | 供应商（采购卖方） |
| Manufacturer | `app/Models/Manufacturer.php` | 制造商（产品品牌方） |
| Asset | `app/Models/Asset.php` | 资产（采购主体记录） |
| AssetModel | `app/Models/AssetModel.php` | 资产型号（连接资产与制造商的中间层） |
| License | `app/Models/License.php` | 许可证（也关联供应商和制造商） |
| Accessory | `app/Models/Accessory.php` | 配件 |
| Consumable | `app/Models/Consumable.php` | 耗材 |
| Component | `app/Models/Component.php` | 组件 |
| Maintenance | `app/Models/Maintenance.php` | 维护记录（仅关联供应商） |

所有模型均继承自 `app/Models/SnipeModel.php`。

---

## 二、角色差异详细阐述

### 2.1 供应商（Supplier）：采购交易的卖方

**业务定义**：供应商是各类资产、许可证、配件、耗材、组件以及维护服务的**提供方/卖方**，直接参与采购交易流程。

**代码依据**：供应商模型 `app/Models/Supplier.php#L17-L18` 定义：

```php
class Supplier extends SnipeModel
{
    // ...
}
```

供应商通过 `hasMany` 关系直接作为6类采购记录的外键关联方：

| 关联方法 | 关系类型 | 被关联模型 | 代码位置 |
|---------|---------|----------|---------|
| `assets()` | hasMany | Asset | `app/Models/Supplier.php#L117-L120` |
| `licenses()` | hasMany | License | `app/Models/Supplier.php#L215-L218` |
| `accessories()` | hasMany | Accessory | `app/Models/Supplier.php#L131-L134` |
| `consumables()` | hasMany | Consumable | `app/Models/Supplier.php#L159-L162` |
| `components()` | hasMany | Component | `app/Models/Supplier.php#L145-L148` |
| `maintenances()` | hasMany | Maintenance | `app/Models/Supplier.php#L183-L186` |

以资产关联为例，代码实现如下（`app/Models/Supplier.php#L117-L120`）：

```php
public function assets()
{
    return $this->hasMany(Asset::class, 'supplier_id');
}
```

### 2.2 制造商（Manufacturer）：产品品牌/生产方

**业务定义**：制造商是产品的**生产方/品牌持有方**，定义产品的技术属性、支持渠道和保修信息，不直接参与采购交易。

**代码依据**：制造商模型 `app/Models/Manufacturer.php#L13-L14` 定义：

```php
class Manufacturer extends SnipeModel
{
    // ...
}
```

制造商通过 `hasMany` 关联资产型号，再通过 `hasManyThrough` 穿透关联资产：

| 关联方法 | 关系类型 | 被关联模型 | 代码位置 |
|---------|---------|----------|---------|
| `models()` | hasMany | AssetModel | `app/Models/Manufacturer.php#L101-L104` |
| `assets()` | hasManyThrough | Asset | `app/Models/Manufacturer.php#L96-L99` |
| `licenses()` | hasMany | License | `app/Models/Manufacturer.php#L106-L109` |
| `accessories()` | hasMany | Accessory | `app/Models/Manufacturer.php#L111-L114` |
| `consumables()` | hasMany | Consumable | `app/Models/Manufacturer.php#L116-L119` |
| `components()` | hasMany | Component | `app/Models/Manufacturer.php#L121-L124` |

穿透关联资产的实现（`app/Models/Manufacturer.php#L96-L99`）：

```php
public function assets()
{
    return $this->hasManyThrough(Asset::class, AssetModel::class, 'manufacturer_id', 'model_id');
}
```

### 2.3 角色差异对比总览

| 对比维度 | 供应商 (Supplier) | 制造商 (Manufacturer) |
|---------|------------------|----------------------|
| **业务本质** | 采购交易卖方/服务提供方 | 产品品牌/生产方 |
| **与资产关联** | 直接外键 `asset.supplier_id` | 间接穿透 `asset → model → manufacturer` |
| **关联维护记录** | ✅ 是，作为维护服务提供商 | ❌ 否 |
| **联系信息侧重** | 邮寄地址、电话、传真、联系人 | 技术支持邮箱、支持网址、保修查询 |
| **核心关联对象** | 所有采购记录（6类） | 资产型号、产品定义类（5类） |

---

## 三、字段约束详细定义

### 3.1 供应商字段约束

在 `app/Models/Supplier.php#L28-L42` 中通过 `$rules` 属性定义：

```php
protected $rules = [
    'name' => 'required|max:255|unique_undeleted',
    'fax' => 'min:7|max:35|nullable',
    'phone' => 'min:7|max:35|nullable',
    'contact' => 'max:100|nullable',
    'notes' => 'max:191|nullable',
    'email' => 'email|max:150|nullable',
    'address' => 'max:250|nullable',
    'address2' => 'max:250|nullable',
    'city' => 'max:191|nullable',
    'state' => 'min:2|max:191|nullable',
    'country' => 'min:2|max:191|nullable',
    'zip' => 'max:10|nullable',
    'url' => 'sometimes|url|nullable|string|max:250',
];
```

**字段约束逐条解析**：

| 字段 | 验证规则 | 必填 | 约束说明 |
|-----|---------|-----|---------|
| `name` | `required\|max:255\|unique_undeleted` | ✅ 是 | 名称必填，最长255字符，**未删除记录范围内唯一** |
| `fax` | `min:7\|max:35\|nullable` | ❌ 否 | 传真号，长度7-35字符，可空 |
| `phone` | `min:7\|max:35\|nullable` | ❌ 否 | 电话，长度7-35字符，可空 |
| `contact` | `max:100\|nullable` | ❌ 否 | 联系人姓名，最长100字符，可空 |
| `notes` | `max:191\|nullable` | ❌ 否 | 备注，最长191字符，可空 |
| `email` | `email\|max:150\|nullable` | ❌ 否 | 邮箱格式校验，最长150字符，可空 |
| `address` | `max:250\|nullable` | ❌ 否 | 地址，最长250字符，可空 |
| `address2` | `max:250\|nullable` | ❌ 否 | 地址2，最长250字符，可空 |
| `city` | `max:191\|nullable` | ❌ 否 | 城市，最长191字符，可空 |
| `state` | `min:2\|max:191\|nullable` | ❌ 否 | 州/省，长度2-191字符，可空 |
| `country` | `min:2\|max:191\|nullable` | ❌ 否 | 国家，长度2-191字符，可空 |
| `zip` | `max:10\|nullable` | ❌ 否 | 邮编，最长10字符，可空 |
| `url` | `sometimes\|url\|nullable\|string\|max:250` | ❌ 否 | 仅当字段存在时校验URL格式，最长250字符 |

**唯一约束实现机制**：`unique_undeleted` 规则由 `app/Http/Traits/UniqueUndeletedTrait.php` 提供，确保名称在未软删除的记录中唯一。模型引入了该 Trait（`app/Models/Supplier.php#L55`）：

```php
use UniqueUndeletedTrait;
```

**可填充字段** (`$fillable`) 在 `app/Models/Supplier.php#L77` 定义：

```php
protected $fillable = ['name', 'address', 'address2', 'city', 'state', 'country', 'zip', 'phone', 'fax', 'email', 'contact', 'url', 'tag_color', 'notes'];
```

### 3.2 制造商字段约束

在 `app/Models/Manufacturer.php#L25-L31` 中通过 `$rules` 属性定义：

```php
protected $rules = [
    'name' => 'required|max:255|unique:manufacturers,name,NULL,id,deleted_at,NULL',
    'url' => 'nullable|starts_with:http://,https://,afp://,facetime://,file://,irc://',
    'support_email' => 'email|nullable',
    'support_url' => 'nullable|starts_with:http://,https://,afp://,facetime://,file://,irc://',
    'warranty_lookup_url' => 'nullable|starts_with:http://,https://,afp://,facetime://,file://,irc://',
];
```

**字段约束逐条解析**：

| 字段 | 验证规则 | 必填 | 约束说明 |
|-----|---------|-----|---------|
| `name` | `required\|max:255\|unique:manufacturers,name,NULL,id,deleted_at,NULL` | ✅ 是 | 名称必填，最长255字符，**deleted_at 为 NULL 时唯一**（即未删除记录唯一） |
| `url` | `nullable\|starts_with:http://,https://,...` | ❌ 否 | 官网 URL，必须以指定协议开头，可空 |
| `support_email` | `email\|nullable` | ❌ 否 | 技术支持邮箱，邮箱格式校验，可空 |
| `support_url` | `nullable\|starts_with:http://,https://,...` | ❌ 否 | 技术支持网址，必须以指定协议开头，可空 |
| `warranty_lookup_url` | `nullable\|starts_with:http://,https://,...` | ❌ 否 | 保修查询网址，必须以指定协议开头，可空 |

**唯一约束实现机制**：使用 Laravel 原生 `unique` 规则的扩展语法 `unique:table,column,except,idColumn,whereColumn,whereValue`，实现 `deleted_at IS NULL` 条件下的唯一性。

**可填充字段** (`$fillable`) 在 `app/Models/Manufacturer.php#L51-L61` 定义：

```php
protected $fillable = [
    'name',
    'image',
    'support_email',
    'support_phone',
    'support_url',
    'url',
    'warranty_lookup_url',
    'tag_color',
    'notes',
];
```

### 3.3 字段约束差异对比表

| 字段分类 | 供应商 (Supplier) | 制造商 (Manufacturer) |
|---------|------------------|----------------------|
| **基础信息** | `name`, `notes`, `tag_color` | `name`, `notes`, `tag_color`, `image` |
| **地址信息** | `address`, `address2`, `city`, `state`, `country`, `zip` | ❌ 无 |
| **联系电话** | `phone`, `fax` | `support_phone` |
| **邮箱** | `email`（商务联系） | `support_email`（技术支持） |
| **URL** | `url`（官网，`sometimes\|url`） | `url`, `support_url`, `warranty_lookup_url`（3个URL，均用 `starts_with` 校验协议前缀） |
| **联系人** | `contact` | ❌ 无 |
| **唯一规则实现** | `unique_undeleted` (自定义 Trait) | `unique:...,deleted_at,NULL` (Laravel 原生) |

---

## 四、关联校验详细分析

### 4.1 资产（Asset）中的关联校验

资产模型是采购记录的核心，其关联校验规则定义在 `app/Models/Asset.php#L111-L140`：

```php
protected $rules = [
    'model_id' => ['required', 'integer', 'exists:models,id,deleted_at,NULL', 'not_array'],
    'status_id' => ['required', 'integer', 'exists:status_labels,id'],
    'asset_tag' => ['required', 'min:1', 'max:255', 'unique_undeleted:assets,asset_tag', 'not_array'],
    'name' => ['nullable', 'max:255'],
    'company_id' => ['nullable', 'integer', 'exists:companies,id'],
    'warranty_months' => ['nullable', 'numeric', 'digits_between:0,240'],
    'location_id' => ['nullable', 'exists:locations,id', 'fmcs_location'],
    'rtd_location_id' => ['nullable', 'exists:locations,id', 'fmcs_location'],
    'purchase_date' => ['nullable', 'date', 'date_format:Y-m-d'],
    'serial' => ['nullable', 'string', 'unique_undeleted:assets,serial'],
    'purchase_cost' => ['nullable', 'numeric', 'gte:0', 'max:99999999999999999.99'],
    'supplier_id' => ['nullable', 'exists:suppliers,id'],
    'order_number' => ['nullable', 'string', 'max:191'],
    // ...
];
```

**采购相关关联字段校验**：

| 字段 | 校验规则 | 说明 |
|-----|---------|-----|
| `supplier_id` | `nullable\|exists:suppliers,id` | **可选**，但填写时必须在 suppliers 表中存在对应 ID |
| `model_id` | `required\|integer\|exists:models,id,deleted_at,NULL` | **必填**，必须在 models 表中存在且未软删除 |
| `purchase_date` | `nullable\|date\|date_format:Y-m-d` | 采购日期，格式 Y-m-d |
| `purchase_cost` | `nullable\|numeric\|gte:0\|max:99999999999999999.99` | 采购金额，非负数，最大 17 位整数 2 位小数 |
| `order_number` | `nullable\|string\|max:191` | 订单号，最长 191 字符 |
| `warranty_months` | `nullable\|numeric\|digits_between:0,240` | 保修月数，0-240 个月 |

**关键关联方法**：

1. **资产 → 供应商（直接关联）**，定义于 `app/Models/Asset.php#L1115-L1118`：

```php
public function supplier()
{
    return $this->belongsTo(Supplier::class, 'supplier_id');
}
```

> ⚠️ 注意：此关联**未使用 `withTrashed()`**，因此当供应商被软删除后，`$asset->supplier` 将返回 `null`。

2. **资产 → 制造商（穿透关联，通过 AssetModel）**，定义于 `app/Models/Asset.php#L643-L646`：

```php
public function manufacturer()
{
    return $this->hasOneThrough(Manufacturer::class, AssetModel::class, 'id', 'id', 'model_id', 'manufacturer_id');
}
```

> ⚠️ 注意：此关联**未使用 `withTrashed()`**，因此当制造商被软删除后，`$asset->manufacturer` 将返回 `null`。详见本文第五节软删除行为分析。

3. **资产 → 资产型号（直接关联）**，定义于 `app/Models/Asset.php#L1028-L1031`：

```php
public function model()
{
    return $this->belongsTo(AssetModel::class, 'model_id')->withTrashed();
}
```

> 注意：此关联**使用了 `withTrashed()`**，即使型号被软删除也能查询到。

### 4.2 资产型号（AssetModel）中的制造商关联校验

资产型号作为连接资产与制造商的中间层，其校验规则定义于 `app/Models/AssetModel.php#L51-L58`：

```php
protected $rules = [
    'name' => 'string|required|min:1|max:255|two_column_unique_undeleted:model_number',
    'model_number' => 'string|max:255|nullable|two_column_unique_undeleted:name',
    'min_amt' => 'integer|min:0|nullable',
    'category_id' => 'required|integer|exists:categories,id',
    'manufacturer_id' => 'integer|exists:manufacturers,id|nullable',
    'eol' => 'integer:min:0|max:240|nullable',
];
```

**制造商关联字段校验**：

| 字段 | 校验规则 | 说明 |
|-----|---------|-----|
| `manufacturer_id` | `integer\|exists:manufacturers,id\|nullable` | **可选**，但填写时必须为整数且在 manufacturers 表中存在 |
| `category_id` | `required\|integer\|exists:categories,id` | **必填**，类别必须存在 |
| `name` + `model_number` | `two_column_unique_undeleted` | 型号名称与型号号联合唯一（未删除范围内） |

**资产型号 → 制造商关联方法**，定义于 `app/Models/AssetModel.php#L207-L210`：

```php
public function manufacturer()
{
    return $this->belongsTo(Manufacturer::class, 'manufacturer_id');
}
```

> ⚠️ 注意：此关联**未使用 `withTrashed()`**，当制造商被软删除后，`$assetModel->manufacturer` 将返回 `null`。

### 4.3 其他采购实体的关联校验汇总

| 模型 | supplier_id 校验规则 | manufacturer_id 校验规则 | 代码位置 |
|-----|---------------------|-------------------------|---------|
| License | ❌ **无显式 exists 规则** | ❌ **无显式 exists 规则** | `app/Models/License.php#L55-L68` |
| Accessory | ❌ **无显式 exists 规则** | ❌ **无显式 exists 规则** | `app/Models/Accessory.php#L79-L88` |
| Consumable | ❌ **无显式 exists 规则** | ❌ **无显式 exists 规则** | `app/Models/Consumable.php#L47-L56` |
| Component | `nullable\|integer\|exists:suppliers,id` | `integer\|exists:manufacturers,id\|nullable` | `app/Models/Component.php#L45-L56` |
| Maintenance | `nullable\|integer`（无 exists 校验） | 不适用（无此字段） | `app/Models/Maintenance.php#L39-L52` |

### 4.4 无显式 exists 校验的采购实体约束条件分析 ⚠️

上一节标注为"无显式 exists 规则"的采购实体（License、Accessory、Consumable），其 `supplier_id` 和 `manufacturer_id` 的存在性**没有**在模型 `$rules` 中校验。以下逐层核查它们实际依赖的约束条件：

#### 数据库层：无外键约束

核查所有相关迁移文件后发现，项目中**所有**外键约束均被注释掉。以 `company_id` 为典型例证（`database/migrations/2015_11_05_061115_add_company_id_to_consumables_table.php#L17`）：

```php
// $table->foreign('company_id')->references('id')->on('companies');
```

供应商和制造商相关字段的迁移文件同样**未声明任何外键约束**：

| 表 | 字段 | 迁移文件 | 外键约束 |
|---|-----|---------|---------|
| licenses | supplier_id | `database/migrations/2014_11_05_212408_add_fields_to_licenses.php#L15` | ❌ 仅 `$table->integer('supplier_id')->nullable()` |
| licenses | manufacturer_id | `database/migrations/2016_08_09_002225_add_manufacturer_to_licenses.php#L17` | ❌ 仅 `$table->integer('manufacturer_id')->nullable()` |
| accessories | supplier_id | `database/migrations/2017_10_19_130406_add_image_and_supplier_to_accessories.php#L18` | ❌ 仅 `$table->integer('supplier_id')->nullable()` |
| accessories | manufacturer_id | `database/migrations/2016_08_12_121613_add_manufacturer_to_accessories_table.php#L16` | ❌ 仅 `$table->integer('manufacturer_id')->nullable()` |
| consumables | supplier_id | `database/migrations/2023_04_12_135822_add_supplier_to_components.php#L23-L24` | ❌ 仅 `$table->integer('supplier_id')->nullable()`（与 components 共用迁移） |
| consumables | manufacturer_id | `database/migrations/2024_10_23_162301_add_manufacturer_id_model_number_to_consumables.php#L15` | ❌ 仅 `$table->integer('manufacturer_id')->nullable()` |
| maintenances | supplier_id | （随表创建） | ❌ 无外键 |

#### Controller 层/FormRequest 层：无补充校验

核查了 `app/Http/Controllers/` 和 `app/Http/Requests/` 目录：
- `StoreAccessoryRequest`（`app/Http/Requests/StoreAccessoryRequest.php`）继承 `ImageUploadRequest`，仅校验 `category_type` 和图片格式
- `StoreConsumableRequest`（`app/Http/Requests/StoreConsumableRequest.php`）同样仅校验 `category_type` 和图片格式
- 在 Controller 层全局搜索 `supplier_id.*exists` 和 `manufacturer_id.*exists`，**无任何结果**

#### 实际依赖的约束条件总结

对于 License、Accessory、Consumable 三个模型，`supplier_id` 和 `manufacturer_id` 字段**仅依赖以下条件**：

| 约束层次 | 是否存在 | 说明 |
|---------|---------|-----|
| 数据库外键约束 | ❌ 不存在 | 迁移中所有外键均被注释 |
| 模型 `$rules` 中的 `exists` 规则 | ❌ 不存在 | License/Accessory/Consumable 的 `$rules` 中未包含此两字段的校验 |
| FormRequest 补充校验 | ❌ 不存在 | 相关 Request 类未添加 exists 校验 |
| 模型 `$fillable` 允许赋值 | ✅ 存在 | 字段出现在 `$fillable` 数组中，允许批量赋值 |
| 关联方法定义 | ✅ 存在 | 定义了 `belongsTo` 关系，但**仅用于读取关联数据**，不阻止写入无效 ID |

> **风险提示**：这意味着 License、Accessory、Consumable 的 `supplier_id` 和 `manufacturer_id` 可以被赋值为 suppliers/manufacturers 表中不存在的 ID，且不会被应用层或数据库层拒绝。只有 Component 模型在 `$rules` 中对两个字段同时声明了 `exists` 校验。Maintenance 模型的 `supplier_id` 仅有 `nullable\|integer` 校验，同样缺少 `exists` 校验。

---

## 五、资产通过型号关联到制造商的完整路径

### 5.1 关联路径图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        资产关联制造商完整路径                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐          ┌──────────────┐          ┌──────────────┐ │
│  │    Asset     │          │  AssetModel  │          │ Manufacturer │ │
│  │   (资产)     │          │  (资产型号)   │          │  (制造商)     │ │
│  │              │          │              │          │              │ │
│  │  model_id ───┼─────────▶│  id          │          │              │ │
│  │              │          │              │          │              │ │
│  │              │          │  manufacturer_id ───────┼▶ id          │ │
│  │              │          │              │          │              │ │
│  │ supplier_id  │          │              │          │   name       │ │
│  │     │        │          │   name       │          │   url        │ │
│  │     ▼        │          │ model_number │          │ support_*    │ │
│  │ ┌────────┐   │          │              │          │ warranty_*   │ │
│  │ │Supplier│   │          │              │          │              │ │
│  │ └────────┘   │          │              │          │              │ │
│  └──────────────┘          └──────────────┘          └──────────────┘ │
│                                                                      │
│  关联方式：                                                           │
│  • Asset → AssetModel: belongsTo (model_id), ✅ withTrashed          │
│  • AssetModel → Manufacturer: belongsTo (manufacturer_id), ❌ 无     │
│  • Asset → Manufacturer: hasOneThrough (穿透关联), ❌ 无 withTrashed  │
│  • Asset → Supplier: belongsTo (supplier_id), ❌ 无 withTrashed      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 三层关联代码详解

#### 第1层：Asset → AssetModel（资产 → 资产型号）

定义于 `app/Models/Asset.php#L1028-L1031`：

```php
public function model()
{
    return $this->belongsTo(AssetModel::class, 'model_id')->withTrashed();
}
```

- 外键字段：`assets.model_id`
- 目标主键：`models.id`
- 特点：**使用了 `withTrashed()`**，允许关联已软删除的型号

#### 第2层：AssetModel → Manufacturer（资产型号 → 制造商）

定义于 `app/Models/AssetModel.php#L207-L210`：

```php
public function manufacturer()
{
    return $this->belongsTo(Manufacturer::class, 'manufacturer_id');
}
```

- 外键字段：`models.manufacturer_id`
- 目标主键：`manufacturers.id`
- 特点：**未使用 `withTrashed()`**，软删除的制造商会从查询中排除

#### 第3层：Asset → Manufacturer（穿透快捷关联）

定义于 `app/Models/Asset.php#L643-L646`：

```php
public function manufacturer()
{
    return $this->hasOneThrough(Manufacturer::class, AssetModel::class, 'id', 'id', 'model_id', 'manufacturer_id');
}
```

`hasOneThrough` 6个参数的含义：

| 参数位置 | 参数值 | 含义 |
|---------|-------|-----|
| 1 | `Manufacturer::class` | 最终要到达的目标模型 |
| 2 | `AssetModel::class` | 中间模型（跳板） |
| 3 | `'id'` | 中间模型表上与本地模型关联的字段（`models.id`） |
| 4 | `'id'` | 目标模型表上与中间模型关联的字段（`manufacturers.id`） |
| 5 | `'model_id'` | 本地模型表上的外键（`assets.model_id`） |
| 6 | `'manufacturer_id'` | 中间模型表上的外键（`models.manufacturer_id`） |

### 5.3 软删除对关联路径的影响分析 ⚠️

软删除行为在关联路径的**不同层级产生不对称影响**，这是此前版本的分析中未精确区分的关键事实：

#### 各关联方法的 `withTrashed()` 使用情况

| 关联方法 | 所属模型 | 是否 withTrashed | 代码位置 |
|---------|---------|-----------------|---------|
| `Asset.model()` | Asset | ✅ 是 | `app/Models/Asset.php#L1030` |
| `AssetModel.manufacturer()` | AssetModel | ❌ 否 | `app/Models/AssetModel.php#L207-L210` |
| `Asset.manufacturer()` (hasOneThrough) | Asset | ❌ 否 | `app/Models/Asset.php#L643-L646` |
| `Asset.supplier()` | Asset | ❌ 否 | `app/Models/Asset.php#L1115-L1118` |
| `License.manufacturer()` | License | ✅ 是 | `app/Models/License.php#L435-L438` |
| `License.supplier()` | License | ❌ 否 | `app/Models/License.php#L791-L794` |
| `Maintenance.supplier()` | Maintenance | ✅ 是 | `app/Models/Maintenance.php#L209-L213` |
| `Accessory.manufacturer()` | Accessory | ❌ 否 | `app/Models/Accessory.php#L334-L337` |
| `Accessory.supplier()` | Accessory | ❌ 否 | `app/Models/Accessory.php#L131-L134` |
| `Consumable.manufacturer()` | Consumable | ❌ 否 | `app/Models/Consumable.php#L203-L206` |
| `Consumable.supplier()` | Consumable | ❌ 否 | `app/Models/Consumable.php#L294-L297` |
| `Component.manufacturer()` | Component | ❌ 否 | `app/Models/Component.php#L240-L243` |
| `Component.supplier()` | Component | ❌ 否 | `app/Models/Component.php#L226-L229` |

#### 资产 → 制造商路径的软删除影响矩阵

| 场景 | Asset.model() 结果 | Asset.manufacturer() 结果 | 逐层访问结果 |
|-----|--------------------|--------------------------|------------|
| 型号正常 + 制造商正常 | ✅ 返回型号 | ✅ 返回制造商 | ✅ 正常 |
| **型号被软删除** + 制造商正常 | ✅ 返回型号（`withTrashed`） | ❌ **返回 null** | ✅ `$asset->model->manufacturer` 可正常访问（因第一步用了 withTrashed） |
| 型号正常 + **制造商被软删除** | ✅ 返回型号 | ❌ **返回 null** | ❌ `$asset->model->manufacturer` 也返回 null（第二步无 withTrashed） |
| **型号被软删除** + **制造商被软删除** | ✅ 返回型号 | ❌ **返回 null** | ❌ 逐层访问同样在第二步断裂 |

> **关键发现**：`Asset.manufacturer()` 的 `hasOneThrough` 关联受两个软删除全局作用域的影响：
>
> 1. **中间模型 AssetModel 的软删除**：`hasOneThrough` 查询中，中间表（`models`）**未**应用 `withTrashed()`，因此当 AssetModel 被软删除时，`$asset->manufacturer` 直接返回 `null`——尽管 `$asset->model` 因 `withTrashed()` 仍可访问到该型号。
>
> 2. **目标模型 Manufacturer 的软删除**：目标表（`manufacturers`）同样**未**应用 `withTrashed()`，因此当制造商被软删除时，无论通过哪种方式访问，结果均为 `null`。
>
> 这意味着：**软删除行为在关联路径的第1层（Asset→AssetModel）被 `withTrashed()` 突破，但在第2层（AssetModel→Manufacturer）和穿透关联中生效**，两侧影响不对称。

#### 对比：使用 `withTrashed()` 的例外

仅以下两个关联方法对软删除对象做了包容处理：

- **License.manufacturer()**（`app/Models/License.php#L435-L438`）：使用了 `withTrashed()`，即使制造商被软删除也能查询到
- **Maintenance.supplier()**（`app/Models/Maintenance.php#L209-L213`）：使用了 `withTrashed()`，即使供应商被软删除也能查询到

### 5.4 实际使用示例

在代码中获取资产的制造商有两种方式：

```php
// 方式1：穿透关联（直接调用）—— 受软删除影响，制造商或型号软删除后返回 null
$asset = Asset::find(1);
$manufacturer = $asset->manufacturer;  // 通过 hasOneThrough

// 方式2：通过型号逐层访问 —— 第1步不受软删除影响，第2步受影响
$asset = Asset::find(1);
$model = $asset->model;                 // belongsTo + withTrashed → 即使型号软删除也能获取
$manufacturer = $model?->manufacturer;  // belongsTo 无 withTrashed → 制造商软删除则返回 null
```

在搜索功能中，资产搜索也包含制造商名称，定义于 `app/Models/Asset.php#L207-L217`：

```php
protected $searchableRelations = [
    'status' => ['name'],
    'supplier' => ['name'],
    'company' => ['name'],
    'defaultLoc' => ['name'],
    'location' => ['name'],
    'model' => ['name', 'model_number', 'eol'],
    'category' => ['name'],
    'manufacturer' => ['name'],      // <-- 穿透关联的制造商名称
    'assigned_to' => ['name'],
];
```

---

## 六、删除约束与引用完整性校验

两个模型均实现了 `isDeletable()` 方法，确保在存在关联数据时阻止删除操作。

### 6.1 供应商删除校验

定义于 `app/Models/Supplier.php#L79-L89`：

```php
public function isDeletable()
{
    return Gate::allows('delete', $this)
        && (($this->assets_count ?? $this->assets()->count()) === 0)
        && (($this->licenses_count ?? $this->licenses()->count()) === 0)
        && (($this->consumables_count ?? $this->consumables()->count()) === 0)
        && (($this->accessories_count ?? $this->accessories()->count()) === 0)
        && (($this->components_count ?? $this->components()->count()) === 0)
        && (($this->maintenances_count ?? $this->maintenances()->count()) === 0)
        && ($this->deleted_at == '');
}
```

**删除条件（必须全部满足）**：
1. 当前用户具有 `delete` 权限（Gate 授权）
2. 关联资产数为 0
3. 关联许可证数为 0
4. 关联耗材数为 0
5. 关联配件数为 0
6. 关联组件数为 0
7. 关联维护记录数为 0
8. 当前记录未被软删除

**性能优化**：使用 `$this->assets_count ?? $this->assets()->count()` 模式，优先使用预加载的计数字段（`withCount`），避免 N+1 查询。

### 6.2 制造商删除校验

定义于 `app/Models/Manufacturer.php#L85-L94`：

```php
public function isDeletable()
{
    return Gate::allows('delete', $this)
        && (($this->assets_count ?? $this->assets()->count()) === 0)
        && (($this->licenses_count ?? $this->licenses()->count()) === 0)
        && (($this->consumables_count ?? $this->consumables()->count()) === 0)
        && (($this->accessories_count ?? $this->accessories()->count()) === 0)
        && (($this->components_count ?? $this->components()->count()) === 0)
        && ($this->deleted_at == '');
}
```

**删除条件（必须全部满足）**：
1. 当前用户具有 `delete` 权限
2. 关联资产数为 0（通过 hasManyThrough 统计）
3. 关联许可证数为 0
4. 关联耗材数为 0
5. 关联配件数为 0
6. 关联组件数为 0
7. 当前记录未被软删除

> **关键差异**：制造商不关联维护记录（Maintenance），因此删除校验中少了 `maintenances_count` 检查项。

---

## 七、模型使用的 Traits 对比

| Trait | Supplier | Manufacturer | 引入位置 | 功能说明 |
|-------|----------|-------------|---------|---------|
| `SoftDeletes` | ✅ | ✅ | `Supplier.php#L22` / `Manufacturer.php#L20` | 软删除支持 |
| `HasFactory` | ✅ | ✅ | `Supplier.php#L19` / `Manufacturer.php#L15` | 模型工厂（测试用） |
| `ValidatingTrait` | ✅ | ✅ | `Supplier.php#L56` / `Manufacturer.php#L44` | 模型级自动验证（watson/validating） |
| `Presentable` | ✅ | ✅ | `Supplier.php#L21` / `Manufacturer.php#L19` | 数据展示器（Presenter 模式） |
| `Searchable` | ✅ | ✅ | `Supplier.php#L54` / `Manufacturer.php#L63` | 全文搜索支持 |
| `Loggable` | ✅ | ❌ | `Supplier.php#L53` | 操作日志记录 |
| `HasUploads` | ✅ | ❌ | `Supplier.php#L20` | 附件上传支持 |
| `UniqueUndeletedTrait` | ✅ | ❌ | `Supplier.php#L55` | 未删除记录唯一性校验规则 |

---

## 八、数据库迁移历史（字段演进参考）

### 8.1 供应商表初始创建

迁移文件 `database/migrations/2014_06_24_003011_add_suppliers.php`：

```php
Schema::create('suppliers', function ($table) {
    $table->increments('id');
    $table->string('name');
    $table->string('address', 50)->nullable()->default(null);
    $table->string('address2', 50)->nullable()->default(null);
    $table->string('city')->nullable()->default(null);
    $table->string('state', 2)->nullable()->default(null);
    $table->string('country', 2)->nullable()->default(null);
    $table->string('phone', 20)->nullable()->default(null);
    $table->string('fax', 20)->nullable()->default(null);
    $table->string('email', 150)->nullable()->default(null);
    $table->string('contact', 100)->nullable()->default(null);
    $table->string('notes')->nullable()->default(null);
    $table->timestamps();
    $table->integer('user_id')->nullable();
    $table->softDeletes();
    $table->engine = 'InnoDB';
});
```

后续通过多个迁移文件扩展字段，例如：
- 地址长度扩展：`database/migrations/2021_12_27_151849_change_supplier_address_length.php`
- 电话长度扩展：`database/migrations/2016_12_19_133936_extend_phone_lengths_in_supplier_and_elsewhere.php`
- Notes 字段类型变更：`database/migrations/2025_11_10_205136_change_suppliers_notes_to_text.php`、`database/migrations/2025_11_10_211219_change_suppliers_notes_to_nullable.php`

### 8.2 制造商表初始创建

迁移文件 `database/migrations/2013_11_13_075347_create_manufacturers_table.php`：

```php
Schema::create('manufacturers', function ($table) {
    $table->increments('id');
    $table->string('name');
    $table->timestamps();
    $table->engine = 'InnoDB';
});
```

后续通过迁移文件逐步增加支持字段：
- 用户ID和软删除：`database/migrations/2013_11_15_112701_add_user_id_to_manufacturers.php`、`database/migrations/2013_11_20_123137_add_soft_delete_on_manufacturers.php`
- 支持字段：`database/migrations/2017_03_10_210807_add_fields_to_manufacturer.php`
- 保修查询URL：`database/migrations/2023_04_26_160235_add_warranty_url_to_manufacturers.php`
- Notes 字段：`database/migrations/2025_02_10_230155_add_notes_to_locations_companies_categories_manufacturers_groups.php`

### 8.3 资产表关联字段

- `supplier_id`：通过 `database/migrations/2014_06_24_010742_add_supplier_id_to_asset.php` 添加，后续通过 `database/migrations/2025_06_06_155058_make_supplier_id_nullable.php` 改为可空
- `model_id`：随资产表创建时即存在（`database/migrations/2013_11_15_190327_create_assets_table.php`）

---

## 九、总结与复核清单

### 9.1 核心结论

| 结论 | 代码依据 |
|-----|---------|
| 供应商是采购交易卖方，直接关联6类采购记录（含维护） | `app/Models/Supplier.php#L117-L186` |
| 制造商是产品品牌方，通过 AssetModel 间接关联资产 | `app/Models/Manufacturer.php#L96-L104` |
| 资产通过 `hasOneThrough` 穿透关联制造商 | `app/Models/Asset.php#L643-L646` |
| 供应商字段侧重商务联系信息（地址/电话/联系人） | `app/Models/Supplier.php#L28-L42` |
| 制造商字段侧重技术支持信息（support_*/warranty_*） | `app/Models/Manufacturer.php#L25-L31` |
| 两者均实现严格的引用完整性删除校验 | `app/Models/Supplier.php#L79-L89`、`app/Models/Manufacturer.php#L85-L94` |
| License/Accessory/Consumable 的 supplier_id 和 manufacturer_id **无任何存在性校验**（模型层、数据库层、Controller层均无） | 详见第四节 4.4 分析 |
| 软删除在资产→制造商路径中产生不对称影响：第1层（Asset→AssetModel）被 `withTrashed()` 突破，第2层（AssetModel→Manufacturer）和穿透关联中生效 | 详见第五节 5.3 分析 |
| 数据库层所有外键约束均被注释掉，全系统无 DB 级引用完整性保障 | 详见第四节 4.4 迁移文件核查 |

### 9.2 复核清单

- [ ] 检查 Supplier 模型的 `$rules` 是否包含地址、电话等商务字段（`app/Models/Supplier.php#L28-L42`）
- [ ] 检查 Manufacturer 模型的 `$rules` 是否包含 support_*、warranty_* 等技术字段（`app/Models/Manufacturer.php#L25-L31`）
- [ ] 确认 Asset 模型存在 `supplier()` 方法且为 belongsTo，**未使用** `withTrashed()`（`app/Models/Asset.php#L1115-L1118`）
- [ ] 确认 Asset 模型存在 `manufacturer()` 方法且为 hasOneThrough，**未使用** `withTrashed()`（`app/Models/Asset.php#L643-L646`）
- [ ] 确认 Asset 模型的 `model()` 方法使用了 `withTrashed()`（`app/Models/Asset.php#L1028-L1031`）
- [ ] 确认 AssetModel 模型的 `manufacturer()` 方法**未使用** `withTrashed()`（`app/Models/AssetModel.php#L207-L210`）
- [ ] 确认 License.manufacturer() 使用了 `withTrashed()`（`app/Models/License.php#L435-L438`）
- [ ] 确认 Maintenance.supplier() 使用了 `withTrashed()`（`app/Models/Maintenance.php#L209-L213`）
- [ ] 确认 License/Accessory/Consumable 的 `$rules` 中不包含 supplier_id/manufacturer_id 的 `exists` 校验
- [ ] 确认 Supplier 的 `isDeletable()` 检查 6 类关联记录（`app/Models/Supplier.php#L79-L89`）
- [ ] 确认 Manufacturer 的 `isDeletable()` 检查 5 类关联记录（不含 maintenance）（`app/Models/Manufacturer.php#L85-L94`）
- [ ] 确认两者名称唯一性均限定在未删除范围内
- [ ] 在迁移文件中确认所有外键声明均被注释掉（如 `database/migrations/2015_11_05_061115_add_company_id_to_consumables_table.php#L17`）
