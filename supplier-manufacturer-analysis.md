# 资产采购记录中供应商与制造商关系 - 代码理解材料

> **可复核声明**：本材料所有结论均对应具体代码位置，点击链接可直接跳转至源代码行号进行验证。

---

## 一、领域模型文件清单

涉及的核心领域模型文件位于 `app/Models/` 目录下：

| 模型 | 文件路径 | 职责 |
|-----|---------|-----|
| Supplier | [Supplier.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php) | 供应商（采购卖方） |
| Manufacturer | [Manufacturer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php) | 制造商（产品品牌方） |
| Asset | [Asset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php) | 资产（采购主体记录） |
| AssetModel | [AssetModel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/AssetModel.php) | 资产型号（连接资产与制造商的中间层） |
| License | [License.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/License.php) | 许可证（也关联供应商和制造商） |
| Accessory | [Accessory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Accessory.php) | 配件 |
| Consumable | [Consumable.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Consumable.php) | 耗材 |
| Component | [Component.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Component.php) | 组件 |
| Maintenance | [Maintenance.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Maintenance.php) | 维护记录（仅关联供应商） |

所有模型均继承自 [SnipeModel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/SnipeModel.php)。

---

## 二、角色差异详细阐述

### 2.1 供应商（Supplier）：采购交易的卖方

**业务定义**：供应商是各类资产、许可证、配件、耗材、组件以及维护服务的**提供方/卖方**，直接参与采购交易流程。

**代码依据**：供应商模型 [Supplier.php#L17-L18](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L17-L18) 定义：

```php
class Supplier extends SnipeModel
{
    // ...
}
```

供应商通过 `hasMany` 关系直接作为6类采购记录的外键关联方：

| 关联方法 | 关系类型 | 被关联模型 | 代码位置 |
|---------|---------|----------|---------|
| `assets()` | hasMany | Asset | [Supplier.php#L117-L120](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L117-L120) |
| `licenses()` | hasMany | License | [Supplier.php#L215-L218](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L215-L218) |
| `accessories()` | hasMany | Accessory | [Supplier.php#L131-L134](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L131-L134) |
| `consumables()` | hasMany | Consumable | [Supplier.php#L159-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L159-L162) |
| `components()` | hasMany | Component | [Supplier.php#L145-L148](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L145-L148) |
| `maintenances()` | hasMany | Maintenance | [Supplier.php#L183-L186](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L183-L186) |

以资产关联为例，代码实现如下 [Supplier.php#L117-L120](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L117-L120)：

```php
public function assets()
{
    return $this->hasMany(Asset::class, 'supplier_id');
}
```

### 2.2 制造商（Manufacturer）：产品品牌/生产方

**业务定义**：制造商是产品的**生产方/品牌持有方**，定义产品的技术属性、支持渠道和保修信息，不直接参与采购交易。

**代码依据**：制造商模型 [Manufacturer.php#L13-L14](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L13-L14) 定义：

```php
class Manufacturer extends SnipeModel
{
    // ...
}
```

制造商通过 `hasMany` 关联资产型号，再通过 `hasManyThrough` 穿透关联资产：

| 关联方法 | 关系类型 | 被关联模型 | 代码位置 |
|---------|---------|----------|---------|
| `models()` | hasMany | AssetModel | [Manufacturer.php#L101-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L101-L104) |
| `assets()` | hasManyThrough | Asset | [Manufacturer.php#L96-L99](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L96-L99) |
| `licenses()` | hasMany | License | [Manufacturer.php#L106-L109](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L106-L109) |
| `accessories()` | hasMany | Accessory | [Manufacturer.php#L111-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L111-L114) |
| `consumables()` | hasMany | Consumable | [Manufacturer.php#L116-L119](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L116-L119) |
| `components()` | hasMany | Component | [Manufacturer.php#L121-L124](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L121-L124) |

穿透关联资产的实现 [Manufacturer.php#L96-L99](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L96-L99)：

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

在 [Supplier.php#L28-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L28-L42) 中通过 `$rules` 属性定义：

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

**唯一约束实现机制**：`unique_undeleted` 规则由 [UniqueUndeletedTrait](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Http/Traits/UniqueUndeletedTrait.php) 提供，确保名称在未软删除的记录中唯一。模型顶部引入了该 Trait [Supplier.php#L55](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L55)：

```php
use UniqueUndeletedTrait;
```

**可填充字段** (`$fillable`) 在 [Supplier.php#L77](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L77) 定义：

```php
protected $fillable = ['name', 'address', 'address2', 'city', 'state', 'country', 'zip', 'phone', 'fax', 'email', 'contact', 'url', 'tag_color', 'notes'];
```

### 3.2 制造商字段约束

在 [Manufacturer.php#L25-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L25-L31) 中通过 `$rules` 属性定义：

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

**可填充字段** (`$fillable`) 在 [Manufacturer.php#L51-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L51-L61) 定义：

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

资产模型是采购记录的核心，其关联校验规则定义在 [Asset.php#L111-L140](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php#L111-L140)：

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

1. **资产 → 供应商（直接关联）**，定义于 [Asset.php#L1115-L1118](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php#L1115-L1118)：

```php
public function supplier()
{
    return $this->belongsTo(Supplier::class, 'supplier_id');
}
```

2. **资产 → 制造商（穿透关联，通过 AssetModel）**，定义于 [Asset.php#L643-L646](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php#L643-L646)：

```php
public function manufacturer()
{
    return $this->hasOneThrough(Manufacturer::class, AssetModel::class, 'id', 'id', 'model_id', 'manufacturer_id');
}
```

**`hasOneThrough` 参数解析**：
- 第1参数：最终目标模型 `Manufacturer::class`
- 第2参数：中间模型 `AssetModel::class`
- 第3参数：中间模型上的外键（`AssetModel.id`）
- 第4参数：目标模型上的外键（`Manufacturer.id`）
- 第5参数：本模型上的本地键（`Asset.model_id`）
- 第6参数：中间模型上的本地键（`AssetModel.manufacturer_id`）

3. **资产 → 资产型号（直接关联）**，定义于 [Asset.php#L1028-L1031](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php#L1028-L1031)：

```php
public function model()
{
    return $this->belongsTo(AssetModel::class, 'model_id')->withTrashed();
}
```

### 4.2 资产型号（AssetModel）中的制造商关联校验

资产型号作为连接资产与制造商的中间层，其校验规则定义于 [AssetModel.php#L51-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/AssetModel.php#L51-L58)：

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

**资产型号 → 制造商关联方法**，定义于 [AssetModel.php#L207-L210](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/AssetModel.php#L207-L210)：

```php
public function manufacturer()
{
    return $this->belongsTo(Manufacturer::class, 'manufacturer_id');
}
```

### 4.3 其他采购实体的关联校验汇总

| 模型 | supplier_id 校验规则 | manufacturer_id 校验规则 | 代码位置 |
|-----|---------------------|-------------------------|---------|
| License | 无显式 exists 规则（仅在 fillable） | 无显式 exists 规则（仅在 fillable） | [License.php#L55-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/License.php#L55-L68) |
| Accessory | 无显式 exists 规则（仅在 fillable） | 无显式 exists 规则（仅在 fillable） | [Accessory.php#L79-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Accessory.php#L79-L88) |
| Consumable | 无显式 exists 规则（仅在 fillable） | 无显式 exists 规则（仅在 fillable） | [Consumable.php#L47-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Consumable.php#L47-L56) |
| Component | `nullable\|integer\|exists:suppliers,id` | `integer\|exists:manufacturers,id\|nullable` | [Component.php#L45-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Component.php#L45-L56) |
| Maintenance | `nullable\|integer` | 不适用（无此字段） | [Maintenance.php#L39-L52](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Maintenance.php#L39-L52) |

> **注意**：Component 模型是唯一对 supplier_id 和 manufacturer_id 同时显式使用 `exists` 校验规则的模型。其他模型依赖数据库层外键约束或仅通过 fillable 允许赋值。

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
│  • Asset → AssetModel: belongsTo (model_id)                          │
│  • AssetModel → Manufacturer: belongsTo (manufacturer_id)            │
│  • Asset → Manufacturer: hasOneThrough (穿透关联)                     │
│  • Asset → Supplier: belongsTo (supplier_id, 直接外键)                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 三层关联代码详解

#### 第1层：Asset → AssetModel（资产 → 资产型号）

定义于 [Asset.php#L1028-L1031](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php#L1028-L1031)：

```php
public function model()
{
    return $this->belongsTo(AssetModel::class, 'model_id')->withTrashed();
}
```

- 外键字段：`assets.model_id`
- 目标主键：`models.id`
- 特点：使用 `withTrashed()` 允许关联已软删除的型号

#### 第2层：AssetModel → Manufacturer（资产型号 → 制造商）

定义于 [AssetModel.php#L207-L210](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/AssetModel.php#L207-L210)：

```php
public function manufacturer()
{
    return $this->belongsTo(Manufacturer::class, 'manufacturer_id');
}
```

- 外键字段：`models.manufacturer_id`
- 目标主键：`manufacturers.id`
- 特点：标准 belongsTo，不包含软删除型号

#### 第3层：Asset → Manufacturer（穿透快捷关联）

定义于 [Asset.php#L643-L646](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php#L643-L646)：

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

### 5.3 实际使用示例

在代码中获取资产的制造商有两种方式：

```php
// 方式1：穿透关联（直接调用）
$asset = Asset::find(1);
$manufacturer = $asset->manufacturer;  // 通过 hasOneThrough

// 方式2：通过型号逐层访问
$asset = Asset::find(1);
$model = $asset->model;                 // belongsTo AssetModel
$manufacturer = $model?->manufacturer;  // belongsTo Manufacturer
```

在搜索功能中，资产搜索也包含制造商名称，定义于 [Asset.php#L207-L217](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php#L207-L217)：

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

定义于 [Supplier.php#L79-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L79-L89)：

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

定义于 [Manufacturer.php#L85-L94](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L85-L94)：

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
| `SoftDeletes` | ✅ | ✅ | [Supplier.php#L22](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L22) / [Manufacturer.php#L20](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L20) | 软删除支持 |
| `HasFactory` | ✅ | ✅ | [Supplier.php#L19](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L19) / [Manufacturer.php#L15](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L15) | 模型工厂（测试用） |
| `ValidatingTrait` | ✅ | ✅ | [Supplier.php#L56](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L56) / [Manufacturer.php#L44](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L44) | 模型级自动验证（watson/validating） |
| `Presentable` | ✅ | ✅ | [Supplier.php#L21](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L21) / [Manufacturer.php#L19](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L19) | 数据展示器（Presenter 模式） |
| `Searchable` | ✅ | ✅ | [Supplier.php#L54](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L54) / [Manufacturer.php#L63](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L63) | 全文搜索支持 |
| `Loggable` | ✅ | ❌ | [Supplier.php#L53](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L53) | 操作日志记录 |
| `HasUploads` | ✅ | ❌ | [Supplier.php#L20](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L20) | 附件上传支持 |
| `UniqueUndeletedTrait` | ✅ | ❌ | [Supplier.php#L55](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L55) | 未删除记录唯一性校验规则 |

---

## 八、数据库迁移历史（字段演进参考）

### 8.1 供应商表初始创建

迁移文件 [2014_06_24_003011_add_suppliers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2014_06_24_003011_add_suppliers.php)：

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
- 地址长度扩展：[2021_12_27_151849_change_supplier_address_length.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2021_12_27_151849_change_supplier_address_length.php)
- 电话长度扩展：[2016_12_19_133936_extend_phone_lengths_in_supplier_and_elsewhere.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2016_12_19_133936_extend_phone_lengths_in_supplier_and_elsewhere.php)

### 8.2 制造商表初始创建

迁移文件 [2013_11_13_075347_create_manufacturers_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2013_11_13_075347_create_manufacturers_table.php)：

```php
Schema::create('manufacturers', function ($table) {
    $table->increments('id');
    $table->string('name');
    $table->timestamps();
    $table->engine = 'InnoDB';
});
```

后续通过迁移文件逐步增加支持字段：
- 用户ID和软删除：[2013_11_15_112701_add_user_id_to_manufacturers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2013_11_15_112701_add_user_id_to_manufacturers.php)、[2013_11_20_123137_add_soft_delete_on_manufacturers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2013_11_20_123137_add_soft_delete_on_manufacturers.php)
- 支持字段：[2017_03_10_210807_add_fields_to_manufacturer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2017_03_10_210807_add_fields_to_manufacturer.php)
- 保修查询URL：[2023_04_26_160235_add_warranty_url_to_manufacturers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2023_04_26_160235_add_warranty_url_to_manufacturers.php)

### 8.3 资产表关联字段

资产表中 `supplier_id` 通过迁移 [2014_06_24_010742_add_supplier_id_to_asset.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2014_06_24_010742_add_supplier_id_to_asset.php) 添加，后续通过 [2025_06_06_155058_make_supplier_id_nullable.php](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/database/migrations/2025_06_06_155058_make_supplier_id_nullable.php) 改为可空。

---

## 九、总结与复核清单

### 9.1 核心结论

| 结论 | 代码依据 |
|-----|---------|
| 供应商是采购交易卖方，直接关联6类采购记录（含维护） | [Supplier.php#L117-L186](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L117-L186) |
| 制造商是产品品牌方，通过 AssetModel 间接关联资产 | [Manufacturer.php#L96-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L96-L104) |
| 资产通过 `hasOneThrough` 穿透关联制造商 | [Asset.php#L643-L646](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Asset.php#L643-L646) |
| 供应商字段侧重商务联系信息（地址/电话/联系人） | [Supplier.php#L28-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L28-L42) |
| 制造商字段侧重技术支持信息（support_*/warranty_*） | [Manufacturer.php#L25-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L25-L31) |
| 两者均实现严格的引用完整性删除校验 | [Supplier.php#L79-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Supplier.php#L79-L89)、[Manufacturer.php#L85-L94](file:///d:/fz/0601-1/solo-dogfeeding/code/16-snipe-it/app/Models/Manufacturer.php#L85-L94) |

### 9.2 复核清单

- [ ] 检查 Supplier 模型的 `$rules` 是否包含地址、电话等商务字段
- [ ] 检查 Manufacturer 模型的 `$rules` 是否包含 support_*、warranty_* 等技术字段
- [ ] 确认 Asset 模型存在 `supplier()` 方法且为 belongsTo
- [ ] 确认 Asset 模型存在 `manufacturer()` 方法且为 hasOneThrough
- [ ] 确认 AssetModel 模型存在 `manufacturer()` 方法且为 belongsTo
- [ ] 确认 Supplier 的 `isDeletable()` 检查 6 类关联记录
- [ ] 确认 Manufacturer 的 `isDeletable()` 检查 5 类关联记录（不含 maintenance）
- [ ] 确认两者名称唯一性均限定在未删除范围内
