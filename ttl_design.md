# Building Energy Metadata TTL Design

## 1. 设计目的

本 `.ttl` 文件用于验证 Building Energy Data Product 的 metadata 是否符合预定的 SHACL 约束。

设计思路是：

```text
JSON-LD metadata 的结构
        ↓
用户可能出现的错误
        ↓
将错误类型转化为 SHACL 规则
        ↓
Validator 输出 conformance result
```

重点覆盖字段完整性、单值约束、值域、时间逻辑、License、API endpoint、Dataset 数量和额外字段等问题。

---

## 2. 输入 Metadata 的结构

### 2.1 文件格式与语言

给定的 `valid.jsonld` 和 `invalid.jsonld` 都使用 **JSON-LD（JSON for Linked Data）**。

JSON-LD 的表面语法是 JSON，但它通过 `@context` 把字段映射到 RDF vocabulary，因此可以被 RDF/SHACL Validator 解析成 RDF Graph。

当前 metadata 使用的主要 vocabulary 包括：

| Prefix | 用途 |
|---|---|
| `dcat:` | 描述 Dataset 和 API endpoint |
| `dct:` | 描述 title、description、format、license、spatial、frequency |
| `ex:` | 项目自定义字段，如 datasetId、providerName、unit、temporalStart、temporalEnd |
| `xsd:` | 定义 string、date 等 datatype |

JSON-LD 中：

```json
"@type": "dcat:Dataset"
```

表示该对象是一个 `dcat:Dataset`。

`@id` 则给 Dataset 一个唯一的 RDF IRI。

---

### 2.2 Metadata 中包含的内容

当前 Dataset 采用扁平结构，metadata 直接挂在一个 `dcat:Dataset` 上：

```text
dcat:Dataset
│
├── datasetId
├── title
├── description
├── providerName
├── format
├── frequency
├── unit
├── spatialCoverage
├── temporalStart
├── temporalEnd
├── license
└── endpointUrl
```

可以将这些信息分为三类。

#### A. 基础 Metadata 字段

| 字段 | 含义 |
|---|---|
| `datasetId` | Dataset 标识 |
| `title` | 数据产品名称 |
| `description` | 数据产品描述 |
| `providerName` | Provider 名称 |
| `format` | 数据格式 |
| `frequency` | 更新频率 |
| `unit` | 数据单位 |
| `spatialCoverage` | 空间覆盖范围 |
| `temporalStart` | 时间范围起点 |
| `temporalEnd` | 时间范围终点 |

#### B. License

JSON-LD 中：

```json
"license": {
  "@id": "dct:license",
  "@type": "@id"
}
```

说明 License 的值按 **IRI** 处理，而不是普通字符串。

例如：

```text
https://creativecommons.org/licenses/by/4.0/
```

表示 Dataset 指向一个许可证资源。

#### C. API Endpoint

JSON-LD 中：

```json
"endpointUrl": {
  "@id": "dcat:endpointURL",
  "@type": "@id"
}
```

因此 API endpoint 同样作为 **IRI** 表达。

当前 metadata 只描述 API 的 endpoint：

```text
https://api.example.org/energy/buildings/hourly
```

它并不是完整的 OpenAPI description，因此 TTL 验证的是 **API endpoint metadata**，而不是 API 的运行状态。

---

## 3. 用户上传时可能出现的问题

TTL 的设计不能只针对一份预先构造好的 invalid sample，而需要考虑用户实际提交 metadata 时可能出现的不同错误。

这些问题可以分为四类。

---

### 3.1 字段内容不符合要求

字段存在，但值本身错误。

例如：

```text
unit = MWh
```

而 profile 只允许：

```text
unit = kWh
```

类似情况还包括：

```text
format = text/csv
frequency = daily
```

以及日期虽然格式正确，但逻辑错误：

```text
temporalStart = 2026-05-10
temporalEnd   = 2026-05-01
```

两个字段单独看都是合法的 `xsd:date`，但：

```text
temporalStart > temporalEnd
```

说明时间范围倒置。

因此 TTL 不仅要检查 datatype，还要检查：

```text
允许值
+
字段之间的逻辑关系
```

---

### 3.2 字段少了

用户可能漏掉本 profile 要求的字段，例如：

```text
providerName
temporalEnd
endpointUrl
unit
```

也可能提交：

```json
"providerName": null
```

或：

```json
"providerName": []
```

在 JSON-LD 转换为 RDF 后，这些情况可能不会形成对应的有效 property value。

因此必填字段需要：

```turtle
sh:minCount 1
```

保证至少存在一个值。

---

### 3.3 字段多了

“字段多了”有两种不同情况。

#### 情况 1：同一个字段出现多个值

例如：

```text
unit = ["kWh", "MWh"]
```

或者：

```text
endpointUrl = [endpointA, endpointB]
```

对于本 profile 中定义为单值的字段，应通过：

```turtle
sh:maxCount 1
```

拒绝多个值。

#### 情况 2：出现 profile 没有定义的字段

例如用户增加：

```text
temperature
ownerPhone
unknownField
```

普通 SHACL Shape 默认不会自动拒绝这些字段。

因此最终 TTL 使用：

```turtle
sh:closed true
```

限制 Dataset 只能包含 profile 明确声明的 properties。

---

## 4. License 可能出现的问题

License 当前设计为 **可选字段**。

因此：

```text
没有 License
```

本身不会导致验证失败。

但如果用户提供了 License，则可能出现以下问题：

| 情况 | 预期结果 |
|---|---|
| License 缺失 | PASS |
| 一个合法 HTTPS IRI | PASS |
| 多个 License | FAIL |
| License 是普通字符串 | FAIL |
| License 使用 `http://` | FAIL |

对应规则：

```turtle
sh:maxCount 1 ;
sh:nodeKind sh:IRI ;
sh:pattern "^https://" ;
```

因此 License 的设计逻辑是：

```text
可以不提供
        ↓
如果提供
        ↓
只能有一个
        ↓
必须是 IRI
        ↓
必须使用 HTTPS
```

当前 TTL **没有规定 License 必须是 CC-BY-4.0**。

所以一个其他的 HTTPS License IRI 仍可能通过。若未来需要限定许可证集合，可以增加：

```turtle
sh:in (...)
```

作为 whitelist。

---

## 5. API Endpoint 可能出现的问题

API endpoint 是必填字段。

可能出现的问题包括：

| 情况 | 预期结果 |
|---|---|
| endpoint 缺失 | FAIL |
| 出现多个 endpoint | FAIL |
| endpoint 是普通字符串 Literal | FAIL |
| endpoint 使用 HTTP | FAIL |
| 一个 HTTPS IRI | PASS |

对应规则：

```turtle
sh:minCount 1 ;
sh:maxCount 1 ;
sh:nodeKind sh:IRI ;
sh:pattern "^https://" ;
```

因此 API endpoint 的规则同时检查：

```text
存在性
+
唯一性
+
IRI 类型
+
HTTPS
```

需要注意，TTL 只能检查 endpoint metadata 是否符合规则，不能判断：

```text
URL 是否真的能访问
HTTP status 是否为 200
TLS certificate 是否有效
API 是否需要认证
返回的数据是否真的是 JSON
返回内容是否符合 OpenAPI schema
```

这些属于运行时测试，应由 ITB 的 HTTP / Messaging Test Step 处理，而不是由 SHACL TTL 负责。

---

## 6. 第一版规则的不足

第一版已经能够完成基础验证，包括：

- 必填字段存在性；
- 基本 datatype；
- endpoint 必须是 IRI；
- `format` 和 `unit` 的基本值检查。

但它还不能完整覆盖前面定义的用户输入风险。

主要不足包括：

| 风险 | 第一版不足 |
|---|---|
| 空字符串 / 纯空白 | `minCount` 不能判断内容 |
| 单值字段出现多个值 | `maxCount` 使用不完整 |
| 枚举字段出现额外值 | `hasValue` 不能严格限定值域 |
| frequency 错误 | 未覆盖 |
| License 错误 | 未覆盖 |
| HTTP endpoint | IRI 合法但仍可能是 HTTP |
| 时间倒流 | datatype 无法判断 start/end 关系 |
| Dataset 数量异常 | 未检查 |
| 未声明字段 | 默认不会拒绝 |

因此最终版是在原有 metadata 结构上扩大验证范围，而不是改变 Dataset 的数据模型。

---

## 7. 最终 TTL 规则设计

### 7.1 Dataset 数量与身份

一次 submission 必须包含且只包含一个：

```text
dcat:Dataset
```

通过 SHACL-SPARQL 检查：

```text
0 个 Dataset      → FAIL
1 个 Dataset      → PASS
2 个以上 Dataset  → FAIL
```

Dataset 自身还必须满足：

```turtle
sh:nodeKind sh:IRI
```

即 Dataset 必须具有明确的 IRI identity。

---

### 7.2 必填字符串字段

以下字段：

```text
datasetId
title
providerName
spatialCoverage
```

统一使用：

```turtle
sh:minCount 1 ;
sh:maxCount 1 ;
sh:datatype xsd:string ;
sh:minLength 1 ;
sh:pattern "\\S" ;
```

分别解决：

```text
缺失
多值
datatype 错误
空字符串
纯空白字符串
```

---

### 7.3 严格枚举字段

当前 profile 规定：

```text
format    = application/json
frequency = hourly
unit      = kWh
```

因此使用：

```turtle
sh:in (...)
```

同时配合：

```turtle
sh:maxCount 1
```

保证字段只能存在一个值，而且该值必须属于允许集合。

---

### 7.4 日期与时间顺序

`temporalStart` 和 `temporalEnd` 都要求：

```turtle
sh:minCount 1 ;
sh:maxCount 1 ;
sh:datatype xsd:date ;
```

此外通过 SHACL-SPARQL 增加：

```text
temporalStart ≤ temporalEnd
```

从而同时检查：

```text
日期格式
+
时间逻辑
```

---

### 7.5 License

License 为可选字段：

```text
0 或 1 个
```

如果提供，则：

```text
必须是 IRI
必须使用 HTTPS
```

---

### 7.6 API Endpoint

Endpoint 为必填字段：

```text
exactly one
+
IRI
+
HTTPS
```

---

### 7.7 可选 Description

`description` 可以缺失。

如果存在，则要求：

```text
最多一个
+
xsd:string
```

---

### 7.8 未声明字段

最终 TTL 使用：

```turtle
sh:closed true
```

只允许当前 metadata profile 明确声明的 Dataset properties。

因此 profile 之外的额外字段会被拒绝。

---

## 8. 最终规则总表

| 对象 / 字段 | 必填 | 最大数量 | 类型 | 其他规则 |
|---|---:|---:|---|---|
| Dataset | 是 | exactly 1 | IRI | 必须为 `dcat:Dataset` |
| `datasetId` | 是 | 1 | `xsd:string` | 非空、非纯空白 |
| `title` | 是 | 1 | `xsd:string` | 非空、非纯空白 |
| `providerName` | 是 | 1 | `xsd:string` | 非空、非纯空白 |
| `spatialCoverage` | 是 | 1 | `xsd:string` | 非空、非纯空白 |
| `format` | 是 | 1 | `xsd:string` | `application/json` |
| `frequency` | 是 | 1 | `xsd:string` | `hourly` |
| `unit` | 是 | 1 | `xsd:string` | `kWh` |
| `temporalStart` | 是 | 1 | `xsd:date` | start ≤ end |
| `temporalEnd` | 是 | 1 | `xsd:date` | start ≤ end |
| `endpointUrl` | 是 | 1 | IRI | HTTPS |
| `description` | 否 | 1 | `xsd:string` | — |
| `license` | 否 | 1 | IRI | HTTPS |
| Profile 外字段 | 不允许 | — | — | Closed Shape |

---

## 9. TTL 的验证边界

最终 TTL 负责：

```text
metadata conformance
```

可以检查：

```text
字段缺失
字段多值
datatype
空字符串
纯空白
枚举值
IRI / Literal
HTTPS
时间倒流
Dataset 数量
额外字段
```

不能检查：

```text
API 是否真的在线
HTTP response 是否正常
API 返回内容是否正确
License URL 是否真实有效
Provider 是否真实存在
```

这些运行时或外部语义检查，应作为 ITB 中独立的 supplemental test。

---

## 10. 在 ITB 中的作用

最终执行关系为：

```text
User / SUT
    ↓
JSON-LD metadata
    ↓
ITB Test Case
    ↓
SHACL Validator
    ↓
TTL Shapes
    ↓
Validation Report
    ↓
PASS / FAIL / INAPPLICABLE / UNTESTABLE
```

TTL 的职责是定义 metadata 的可执行约束；ITB 负责组织测试流程、调用 Validator，并对最终验证结果进行汇总和映射。
