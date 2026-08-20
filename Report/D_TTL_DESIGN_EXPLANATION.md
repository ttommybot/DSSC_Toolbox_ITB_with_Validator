# Building Energy Metadata SHACL TTL 设计详解

## 1. 文件目标

`building-energy-shapes.ttl` 用于验证一个 Building Energy Dataset 的 RDF/JSON-LD metadata 是否符合当前项目定义的 metadata profile。

它主要完成四类工作：

1. 检查提交文件中是否恰好存在一个 `dcat:Dataset`；
2. 检查 Dataset 的必填字段、可选字段、类型、数量和允许值；
3. 检查 Dataset 对应的 Distribution 和 DataService 访问结构；
4. 发现当前 profile 未声明的多余字段，以 `sh:Violation` 拒绝提交，并在报告中标识为 `INAPPLICABLE`。

TTL 定义“数据满足什么约束”并直接拒绝 `FAIL` 与 `INAPPLICABLE`。ITB 原生会把两者都显示为 `FAILURE`，具体业务类别通过 validation report 区分；`UNTESTABLE` 仍需由 Test Case 或额外服务判断：

| Test Case 结果 | 含义 |
|---|---|
| `PASS` | 适用规则全部满足；选填字段未写也属于 PASS。 |
| `FAIL` | 已声明字段缺失、类型错误、数量错误或值不符合约束。 |
| `INAPPLICABLE` | SUT 提交了本 profile 未声明的字段。 |
| `UNTESTABLE` | 文件无法解析、Validator 超时或基础设施故障，无法得到可靠结论。 |

> `UNTESTABLE` 不能由 TTL 本身产生。它必须由 Test Case 根据解析器、Validator 和基础设施状态判断。

---

## 2. Prefix 设计

文件开头定义了六个 namespace：

```turtle
@prefix sh:   <http://www.w3.org/ns/shacl#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix dcat: <http://www.w3.org/ns/dcat#> .
@prefix dct:  <http://purl.org/dc/terms/> .
@prefix ex:   <https://example.org/dssc-energy#> .
```

它们分别负责：

- `sh:`：SHACL 约束词汇，例如 `sh:minCount`、`sh:datatype`；
- `xsd:`：数据类型，例如 `xsd:string`、`xsd:date`；
- `rdf:`：RDF 基础词汇，closed shape 中用于忽略 `rdf:type`；
- `dcat:`：Dataset、Distribution、DataService 和访问地址；
- `dct:`：identifier、title、description、license、spatial、format 等 metadata；
- `ex:`：项目自定义字段，例如 `providerName`、`unit`、`temporalStart`。

`ex:resultId` 已被删除。Test Case 不再依靠自定义 ID 映射字段，而是使用 validation report 中的：

- `sh:sourceShape`；
- `sh:resultPath`；
- `sh:sourceConstraintComponent`；
- `sh:focusNode`。

这样 TTL 保持为纯粹的约束文件，不混入额外的结果编号逻辑。

---

## 3. 总体数据结构

本 profile 的核心结构是：

```text
一个提交文件
└── 恰好一个 dcat:Dataset
    ├── Dataset metadata 字段
    └── 一个或多个 dcat:Distribution
        ├── dct:format = application/json
        ├── dcat:accessURL
        └── 或 dcat:accessService
            └── dcat:DataService
                └── dcat:endpointURL
```

访问方式允许两种：

### 直接访问

```text
Dataset
└── Distribution
    └── dcat:accessURL
```

### 通过服务访问

```text
Dataset
└── Distribution
    └── dcat:accessService
        └── DataService
            └── dcat:endpointURL
```

两种方式也可以同时存在。

---

## 4. Dataset 数量约束

### 4.1 设计内容

```turtle
ex:DatasetCardinalityShape
    a sh:NodeShape ;
    sh:targetNode ex:ValidationSubmission ;
    sh:sparql [
        sh:select """
            PREFIX dcat: <http://www.w3.org/ns/dcat#>
            SELECT $this
            WHERE {
                {
                    SELECT (COUNT(DISTINCT ?dataset) AS ?count)
                    WHERE { ?dataset a dcat:Dataset . }
                }
                FILTER (?count != 1)
            }
        """ ;
    ] .
```

### 4.2 为什么需要它

普通 Dataset shape 使用：

```turtle
sh:targetClass dcat:Dataset
```

如果提交文件中根本没有 `dcat:Dataset`，普通字段约束可能一个 focus node 都选不到，从而没有产生字段 violation。这样容易出现“实际什么都没测，却看起来通过”的零目标假 PASS。

因此增加一个提交级 SPARQL 约束，直接统计整个 RDF graph 中的 Dataset 数量。

### 4.3 判定结果

| Dataset 数量 | 结果 |
|---:|---|
| 0 | FAIL |
| 1 | PASS |
| 2 个或以上 | FAIL |
| RDF 无法解析，无法统计 | UNTESTABLE，由 Test Case 判断 |

这里的 `ex:ValidationSubmission` 只是一个稳定的 target node，用来触发这条全图查询，并不要求上传数据中真的声明这个节点。

---

## 5. Dataset 主 Shape

```turtle
ex:BuildingEnergyDatasetShape
    a sh:NodeShape ;
    sh:targetClass dcat:Dataset ;
    sh:nodeKind sh:IRI ;
    sh:property ex:IdentifierShape ;
    ...
```

它负责把全部 Dataset 字段 shape 组合起来。

### 5.1 `sh:targetClass dcat:Dataset`

所有具有：

```turtle
a dcat:Dataset
```

的节点都会被选为 focus node。

### 5.2 `sh:nodeKind sh:IRI`

Dataset 本身必须是 IRI，不能是 blank node 或 literal。例如：

```turtle
<https://example.org/datasets/energy-001> a dcat:Dataset .
```

可以通过，而匿名 Dataset 节点会失败。

这使 Dataset 具有稳定、可引用的标识。

### 5.3 `sh:property`

每一项：

```turtle
sh:property ex:IdentifierShape ;
```

表示 Dataset 必须接受对应 PropertyShape 的验证。

---

## 6. 必填字符串字段的统一设计

`identifier`、`title`、`providerName` 和 `spatial` 采用相同的约束模板：

```turtle
sh:minCount 1 ;
sh:maxCount 1 ;
sh:datatype xsd:string ;
sh:minLength 1 ;
sh:pattern "\\S" ;
```

### 6.1 `sh:minCount 1`

要求至少有一个值。

因此以下情况都会被视为未提供：

- 字段不存在；
- JSON-LD 中值为 `null`，展开后没有 RDF triple；
- 空数组 `[]`，展开后没有 RDF value；
- 必填字段在预处理阶段因只有空白而被删除。

结果均为 `FAIL`。

### 6.2 `sh:maxCount 1`

要求最多一个不同的 RDF 值，防止同一字段同时出现多个互相冲突的值。

例如：

```json
"identifier": ["dataset-a", "dataset-b"]
```

会失败。

需要注意：两个完全相同的 RDF triple 可能在 RDF graph 中被合并。因此 SHACL 不一定能发现原始 JSON-LD 中重复写入两个完全相同值的情况。这属于序列化层问题，不是当前 TTL 能可靠检测的内容。

### 6.3 `sh:datatype xsd:string`

字段必须是字符串 literal，不能是数字、布尔值或 IRI。

例如：

```json
"providerName": 123
```

会产生 datatype violation。

### 6.4 `sh:minLength 1`

阻止真正的空字符串：

```json
"title": ""
```

### 6.5 `sh:pattern "\\S"`

要求字符串中至少存在一个非空白字符，防止：

```json
"title": "   "
```

通过。

其中 `\S` 能识别普通空格、Tab、换行等常见空白字符。

### 6.6 各字段含义

- `dct:identifier`：必填，但不再固定为 `building-energy-hourly-v1`；只要求一个非空字符串；
- `dct:title`：Dataset 标题；
- `ex:providerName`：数据提供方名称；
- `dct:spatial`：空间覆盖范围，目前设计为一个非空字符串。

---

## 7. 可选字段设计

### 7.1 Description

```turtle
ex:DescriptionShape
    sh:path dct:description ;
    sh:maxCount 1 ;
    sh:datatype xsd:string .
```

这里没有 `sh:minCount 1`，所以 description 可以不提供。

结果逻辑：

| 输入情况 | 结果 |
|---|---|
| 没有 description | PASS |
| 一个正常字符串 | PASS |
| 多个不同 description | FAIL |
| 数字等非字符串 | FAIL |
| 只有空格、Tab、换行 | 预处理后删除，视为未提供，PASS |

需要特别说明：

> “空白选填字段算 PASS”不是单靠 TTL 完成的，而是依赖 Test Case 在 JSON-LD 展开前进行空白归一化。

推荐预处理判断：

```regex
^[\s\u00A0]*$
```

它覆盖：

- 空字符串；
- 一个或多个普通空格；
- Tab；
- 回车；
- 换行；
- 不换行空格 `U+00A0`；
- 上述字符的混合。

匹配后删除该可选属性，再进行 JSON-LD/RDF 验证。

### 7.2 License

```turtle
ex:LicenseShape
    sh:path dct:license ;
    sh:maxCount 1 ;
    sh:nodeKind sh:IRI ;
    sh:pattern "^https://" .
```

License 是可选字段：

- 不写：PASS；
- 写了：必须是一个 HTTPS IRI；
- 写成普通字符串：FAIL；
- 写成 `http://...`：FAIL；
- 写多个 license：FAIL；
- 只写空格或 Tab：预处理后删除，PASS。

`sh:nodeKind sh:IRI` 检查 RDF 节点类型，`sh:pattern "^https://"` 检查 IRI 的字符串形式是否以 HTTPS 开头。

---

## 8. 严格枚举字段

### 8.1 Frequency

```turtle
sh:path dct:accrualPeriodicity ;
sh:minCount 1 ;
sh:maxCount 1 ;
sh:datatype xsd:string ;
sh:in ( "hourly" ) .
```

该字段：

- 必须存在；
- 只能有一个值；
- 必须是字符串；
- 值必须精确等于 `hourly`。

以下都会 FAIL：

```text
Hourly
HOURLY
hourly 
daily
```

Validator 不应自动 trim、改大小写或纠正拼写。

### 8.2 Unit

```turtle
sh:path ex:unit ;
sh:in ( "kWh" ) .
```

要求精确等于 `kWh`。

使用 `sh:in` 而不是 `sh:hasValue` 的原因是：

```json
"unit": ["kWh", "MWh"]
```

如果只使用 `sh:hasValue "kWh"`，因为值集合中确实包含 `kWh`，可能仍会通过。现在使用：

```turtle
sh:maxCount 1 ;
sh:in ( "kWh" ) ;
```

能够同时禁止多值和非法值。

### 8.3 Distribution format

```turtle
sh:path dct:format ;
sh:in ( "application/json" ) .
```

每个 Distribution 必须有且只有一个 format，并且精确等于 `application/json`。

---

## 9. 时间字段和时间关系

### 9.1 单个日期字段

```turtle
ex:TemporalStartShape
    sh:path ex:temporalStart ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:datatype xsd:date .
```

`temporalEnd` 使用相同设计。

要求：

- 两个字段都必填；
- 每个字段只能有一个值；
- RDF datatype 必须是 `xsd:date`。

一个看起来像日期的普通字符串不一定满足 `xsd:date`。JSON-LD context 必须正确把它转换为带 `xsd:date` datatype 的 RDF literal。

### 9.2 时间顺序

```turtle
FILTER (
    datatype(?start) = xsd:date &&
    datatype(?end) = xsd:date &&
    ?start > ?end
)
```

当 start 晚于 end 时产生 violation。

设计中特意加入 datatype guard：

```text
datatype(?start) = xsd:date
datatype(?end) = xsd:date
```

原因是：如果日期本身已经类型错误，就由各自的 PropertyShape 报 datatype FAIL；时间关系 Shape 不再重复产生一个误导性的“日期顺序错误”。

因此：

| 情况 | 日期字段 | 时间关系 |
|---|---|---|
| 两个日期类型正确且顺序正确 | PASS | PASS |
| 两个日期类型正确但 start > end | PASS | FAIL |
| start 或 end 类型错误 | 对应字段 FAIL | 不产生额外顺序 violation |
| start 或 end 缺失 | 对应字段 FAIL | 不产生额外顺序 violation |

---

## 10. Distribution 约束

### 10.1 Dataset 必须具有 Distribution

```turtle
ex:DistributionPropertyShape
    sh:path dcat:distribution ;
    sh:minCount 1 ;
    sh:nodeKind sh:BlankNodeOrIRI ;
    sh:class dcat:Distribution ;
    sh:node ex:DistributionShape .
```

含义：

1. Dataset 至少包含一个 `dcat:distribution`；
2. Distribution 节点可以是 blank node 或 IRI；
3. 该节点必须属于 `dcat:Distribution`；
4. 每一个 Distribution 都必须满足 `ex:DistributionShape`。

这里没有 `sh:maxCount 1`，表示一个 Dataset 可以有多个 Distribution。每一个 Distribution 都会被分别验证。

### 10.2 每个 Distribution 的 format

```turtle
sh:minCount 1 ;
sh:maxCount 1 ;
sh:datatype xsd:string ;
sh:in ( "application/json" ) .
```

因此每个 Distribution 都必须明确声明 JSON format。

---

## 11. Distribution 的访问方式

### 11.1 `sh:or` 规则

```turtle
sh:or (
    [ sh:property [
        sh:path dcat:accessURL ;
        sh:minCount 1
    ] ]
    [ sh:property [
        sh:path dcat:accessService ;
        sh:minCount 1
    ] ]
) .
```

含义是每个 Distribution 至少满足下列一个分支：

1. 至少一个 `dcat:accessURL`；
2. 至少一个 `dcat:accessService`。

两者可以同时存在。

### 11.2 直接 accessURL

```turtle
ex:AccessUrlShape
    sh:path dcat:accessURL ;
    sh:maxCount 1 ;
    sh:nodeKind sh:IRI ;
    sh:pattern "^https://" .
```

它本身没有 `sh:minCount 1`，因为 accessURL 在提供 accessService 时可以省略。

如果存在，则必须：

- 最多一个；
- 是 IRI；
- 使用 HTTPS。

### 11.3 accessService

```turtle
ex:AccessServiceShape
    sh:path dcat:accessService ;
    sh:maxCount 1 ;
    sh:nodeKind sh:BlankNodeOrIRI ;
    sh:class dcat:DataService ;
    sh:node ex:DataServiceShape .
```

如果存在 accessService：

- 最多一个；
- 可以是 blank node 或 IRI；
- 必须声明为 `dcat:DataService`；
- 必须满足 DataServiceShape。

### 11.4 DataService endpointURL

```turtle
ex:EndpointUrlShape
    sh:path dcat:endpointURL ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:nodeKind sh:IRI ;
    sh:pattern "^https://" .
```

每个 DataService 必须有且只有一个 HTTPS endpoint URL。

因此下面的访问结构是合规的：

```turtle
ex:dataset1
    a dcat:Dataset ;
    dcat:distribution ex:distribution1 .

ex:distribution1
    a dcat:Distribution ;
    dct:format "application/json" ;
    dcat:accessService ex:service1 .

ex:service1
    a dcat:DataService ;
    dcat:endpointURL <https://api.example.org/energy> .
```

需要注意：TTL 只能检查 endpoint 是否为 HTTPS IRI，不能证明网址实际可访问、证书有效或 API 真正返回 JSON。这些属于 supplemental Test Case。

---

## 12. Closed Shape 与 INAPPLICABLE

TTL 定义了三个 closed shape：

- `DatasetClosedShape`；
- `DistributionClosedShape`；
- `DataServiceClosedShape`。

例如 Dataset：

```turtle
sh:closed true ;
sh:ignoredProperties ( rdf:type ) ;
sh:property [ sh:path dct:identifier ] ;
...
```

### 12.1 `sh:closed true`

表示除明确列出的属性之外，其他属性都被视为 profile 外属性。

例如 Dataset 提交：

```turtle
ex:dataset1 ex:randomField "abc" .
```

会产生 `sh:ClosedConstraintComponent` finding。

### 12.2 `sh:ignoredProperties ( rdf:type )`

closed shape 如果不忽略 `rdf:type`，那么：

```turtle
a dcat:Dataset
```

本身也可能被视为未声明属性。因此这里明确忽略 `rdf:type`。

### 12.3 为什么 severity 是 Violation

```turtle
sh:severity sh:Violation ;
```

当前准入政策规定只有 `PASS` 可以通过，未声明字段也必须拒绝：

- 已声明字段违反约束：`FAIL`，ITB 显示 `FAILURE`；
- 未声明字段：`INAPPLICABLE`，ITB 同样显示 `FAILURE`，但报告消息以 `INAPPLICABLE:` 开头。

因此现有 Validator 无需额外 Wrapper 就能拒绝多余字段。若需要在 ITB 中把 `FAIL` 和 `INAPPLICABLE` 展示为两个独立业务状态，仍需读取每条 validation result，并根据：

```text
source shape
source constraint component
severity
result path
```

把来自三个 ClosedShape 的 `ClosedConstraintComponent` 识别为 `INAPPLICABLE`。这不影响拒绝结果，只影响报告分类和展示。

### 12.4 错误 namespace

如果用户写：

```turtle
wrong:providerName "Provider"
```

而不是：

```turtle
ex:providerName "Provider"
```

会产生两个逻辑结果：

1. 正确的 `ex:providerName` 缺失，因此必填字段为 `FAIL`；
2. `wrong:providerName` 未在 profile 中声明，因此为 `INAPPLICABLE`。

整体结果应按项目聚合规则决定，通常由于已经存在 FAIL，最终 Test Case 为 FAIL，同时报告额外的 INAPPLICABLE finding。

---

## 13. 四类结果如何从 TTL 映射

### 13.1 PASS

满足以下条件：

- TTL 和数据成功解析；
- Validator 成功运行；
- 对应字段属于本 profile；
- 没有相关 violation；
- 若字段可选，未提供也算 PASS。

### 13.2 FAIL

普通 `sh:Violation` 表明已声明字段不符合要求，例如：

- 必填字段缺失；
- 超过 `maxCount`；
- datatype 错误；
- 值不在 `sh:in` 列表；
- endpoint 不是 IRI 或不是 HTTPS；
- start 晚于 end；
- Dataset 数量不是 1。

### 13.3 INAPPLICABLE

仅用于 TTL 未声明的多余字段：

- Dataset 多余字段；
- Distribution 多余字段；
- DataService 多余字段；
- 错误 namespace 形成的额外属性。

Test Case 根据 closed shape 的 validation result 映射。

### 13.4 UNTESTABLE

由运行环境判断，包括：

- JSON-LD 无法解析；
- Turtle shapes 文件无法解析；
- Validator 超时或崩溃；
- 网络、容器、服务不可用；
- Validator 不支持所需的 SHACL-SPARQL。

这些情况不是 SUT metadata 已经被证明不合规，所以不能判 FAIL。

---

## 14. Test Case 建议拆分

TTL 可以由以下 Test Cases 分组执行和汇总：

| Test Case | 主要检查 |
|---|---|
| TC01 Executability | 数据、TTL、Validator 是否可以正常执行；只产生 PASS 或 UNTESTABLE。 |
| TC02 Dataset Structure | Dataset 数量恰好为 1，Dataset 节点为 IRI。 |
| TC03 Required Metadata | identifier、title、providerName、spatial、frequency、unit、起止日期、distribution。 |
| TC04 Controlled Values | hourly、kWh、application/json。 |
| TC05 Temporal Relation | temporalStart 不晚于 temporalEnd。 |
| TC06 Distribution Access | accessURL 或 accessService → DataService → endpointURL。 |
| TC07 Optional Metadata | description 和 license；未写或空白预处理后删除均为 PASS。 |
| TC08 Closed Profile | 多余字段映射为 INAPPLICABLE。 |

---

## 15. 极端情况对应关系

| 极端输入 | TTL/Test Case 处理 | 结果 |
|---|---|---|
| 必填字段缺失 | `sh:minCount 1` | FAIL |
| 必填字段为 `null` 或空数组 | JSON-LD 展开后无值，触发 minCount | FAIL |
| 必填字段只有空格或 Tab | 预处理删除，随后触发 minCount | FAIL |
| 选填字段缺失 | 没有 minCount | PASS |
| 选填字段为空字符串、多个空格、Tab、换行 | 预处理删除 | PASS |
| 字符串字段为数字 | `sh:datatype xsd:string` | FAIL |
| 单值字段存在两个不同值 | `sh:maxCount 1` | FAIL |
| unit 为 `MWh` | `sh:in ( "kWh" )` | FAIL |
| unit 为 `["kWh", "MWh"]` | maxCount 和 in | FAIL |
| frequency 为 `Hourly` | 精确 `sh:in` | FAIL |
| endpoint 写成普通字符串 | `sh:nodeKind sh:IRI` | FAIL |
| endpoint 使用 HTTP | `sh:pattern "^https://"` | FAIL |
| endpoint HTTPS 但网站不可达 | TTL 无法判断，另做访问测试 | 可能 UNTESTABLE 或 FAIL，取决于访问 Test Case 定义 |
| start 晚于 end | SPARQL 时间约束 | FAIL |
| 没有 Dataset | Dataset 数量约束 | FAIL |
| 多个 Dataset | Dataset 数量约束 | FAIL |
| Dataset 有随机字段 | closed shape | INAPPLICABLE |
| Validator 超时 | Test Case 执行状态 | UNTESTABLE |
| JSON-LD 无法解析 | Test Case 执行状态 | UNTESTABLE |

---

## 16. 当前设计的边界和注意事项

### 16.1 空白归一化必须在 JSON-LD 展开前完成

尤其是 license、accessURL、endpointURL 这类 IRI 字段。如果先把无意义空白交给 JSON-LD parser，文件可能直接解析失败，从而变成 UNTESTABLE，而不是“选填字段未提供”。

### 16.2 业务分类不能只读取 `sh:conforms`

closed-shape 现在使用 `sh:Violation`，因此能够直接拒绝提交；但 `sh:conforms=false` 不能区分普通 `FAIL` 与 `INAPPLICABLE`。如需保留四状态业务分类，Test Case 仍必须读取完整 validation report。

### 16.3 SPARQL 支持是必要条件

Dataset 数量和时间关系使用 SHACL-SPARQL。如果 Validator 不支持 SHACL-SPARQL，这两项应为 UNTESTABLE，而不是自动 PASS。

### 16.4 `dct:accrualPeriodicity` 当前被约束为字符串 `hourly`

这是当前项目 profile 的明确选择。它不意味着所有外部 DCAT profile 都使用同样的表达方式。Test Case 应以本 TTL 为唯一判定依据。

### 16.5 完全相同的重复 RDF 值

RDF graph 会合并完全相同的 triple，因此 TTL 不能可靠判断原始 JSON-LD 是否重复写入完全相同的值。若项目需要检查原始 JSON 序列化重复，应增加 JSON-level supplemental test。

---

## 17. 总结

这份 TTL 的设计原则可以概括为：

```text
提交级约束：恰好一个 Dataset
        ↓
Dataset 字段约束：必填、单值、类型、非空、枚举值
        ↓
Distribution/DataService：至少一种访问路径，HTTPS IRI
        ↓
跨字段约束：开始日期不得晚于结束日期
        ↓
Closed Shape：多余字段以 Violation 拒绝，并在报告中标识为 INAPPLICABLE
        ↓
Test Case：将解析或基础设施故障映射为 UNTESTABLE
```

TTL 负责描述合规边界，并让 `FAIL` 与 `INAPPLICABLE` 都拒绝通过；Test Case 负责运行前预处理、解析状态、Validator 状态，以及在需要时细分四类业务结果。两者缺一不可。
