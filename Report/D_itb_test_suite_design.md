# D 组 ITB 与 Test Suite 设计

## 1. 设计目标

D 组使用 ITB（Interoperability Test Bed）组织建筑能耗数据产品的准入测试，并使用独立 SHACL Validator 执行 metadata 规则验证。

本设计要实现以下目标：

- 将建筑能耗数据产品的准入要求转换为可执行 Test Case；
- 检查 Provider 提交的 JSON-LD metadata 是否符合 D 组规则；
- 检查 metadata 中的许可证是否符合 Authority 政策；
- 为未来的 API endpoint 和 response 验证预留 Test Case；
- 统一定义 PASS、FAIL、INAPPLICABLE 和 UNTESTABLE 的业务含义；
- 由 ITB 保存 Test Session、步骤结果和验证报告。

当前正式 Test Suite 版本为 `3.2.0`。TC01 和 TC02 已启用，TC03 已设计但处于禁用状态。

## 2. ITB 在本方案中的设计

### 2.1 ITB 的职责

ITB 在本方案中负责测试管理和测试流程编排，主要承担：

- 管理 Domain 和 Specification；
- 定义参与测试的 Actor；
- 将 Provider System 与 Actor 建立 Conformance Statement；
- 导入和执行 GITB TDL Test Suite；
- 接收用户上传的测试输入；
- 调用外部 Validator 或 ITB 内置 handler；
- 汇总 Test Case 和 Test Session 结果；
- 保存并导出验证报告。

ITB 不直接定义建筑能耗 metadata 的字段规则。字段约束由 SHACL Shapes 定义，许可证政策由白名单定义，API response 结构由 JSON Schema 定义。

### 2.2 ITB 与 Validator 的关系

ITB 和 SHACL Validator 是两个独立组件：

| 组件 | 作用 |
|---|---|
| ITB | 管理测试、执行 Test Case、调用验证服务、保存结果 |
| SHACL Validator | 解析 RDF / JSON-LD，并根据 SHACL Shapes 检查 metadata |

TC01 中的调用关系为：

```text
Provider 上传 JSON-LD
        -> ITB 执行 TC01
        -> ITB 通过 GITB SOAP Validation Service 调用 Validator
        -> Validator 加载 energy/v1 对应的 SHACL Shapes
        -> Validator 返回 GITB validation report
        -> ITB 根据报告判定 Test Case
```

TC02 不调用外部 SHACL Validator，而是使用 ITB 内置 handler 完成 RDF 查询、许可证数量检查和白名单匹配。

本项目同时保留两种 Validator 接入思路：

| 维度 | 独立 SHACL Validator + SOAP API | ITB 内置 Handler |
|---|---|---|
| 是否需要额外部署 | 需要 | 不需要 |
| 适合的检查 | SHACL metadata validation | 许可证策略、HTTP 调用、JSON Schema |
| 规则位置 | `validator-config/energy/shapes/` | `testsuite/Resources/` 或 Test Case XML |
| 当前用途 | TC01 | TC02；TC03 保留但禁用 |
| 主要风险 | 网络、容器和 SOAP 配置 | 目标 ITB 是否提供对应 Handler |

当前正式实现中，TC01 使用独立 SHACL Validator 的 SOAP API；TC02 使用 ITB 内置 Handler；TC03 因缺少稳定的真实 API endpoint，保留设计但暂不启用。

### 2.3 Domain、Specification 和 Conformance Statement

本方案建议在 ITB 中采用以下逻辑关系：

```text
Domain
└── DSSC Energy
    └── Specification
        └── Building Energy Metadata Profile
            └── Test Suite 3.2.0
                ├── TC01 Metadata Field Validation
                ├── TC02 Licence Policy Validation
                └── TC03 API Response Validation（禁用）
```

Conformance Statement 表示某个 Provider System 以 `Energy Data Provider` Actor 的身份，对该 Specification 声明符合性。Test Case 的运行结果为这一声明提供测试证据。

### 2.4 Actor 设计

Test Suite 定义四个 Actor：

| Actor | 类型 | 作用 |
|---|---|---|
| `Provider` | SUT | 提交建筑能耗数据产品 metadata 的被测系统 |
| `Authority` | Simulated | 定义 metadata profile 和许可证政策 |
| `Validator` | Simulated | 执行 SHACL、许可证和 API 规则检查 |
| `Gateway` | Simulated | 在 TC03 中模拟 Data Space gateway 对 Provider API 的调用 |

只有 `Provider` 是 System Under Test。其他 Actor 由测试环境模拟。

## 3. Test Suite 总体结构

正式 Test Suite 的结构为：

```text
testsuite/
├── testSuite.xml
├── testCases/
│   ├── TC01_METADATA_FIELD_VALIDATION.xml
│   ├── TC02_LICENSE_POLICY_VALIDATION.xml
│   └── TC03_API_RESPONSE_VALIDATION.xml
├── Resources/
│   ├── building-energy-shapes_D.ttl
│   ├── license-whitelist.json
│   ├── api-response.schema.json
│   └── openapi.yaml
└── Samples/
    ├── data-product-valid.jsonld
    ├── data-product-invalid.jsonld
    ├── data-product-license-disallowed.jsonld
    ├── data-product-license-missing.jsonld
    ├── data-product-api-local-test.jsonld
    ├── data-product-api-local-invalid-response.jsonld
    └── mock-api/
```

### 3.1 `testSuite.xml`

`testSuite.xml` 是 Test Suite 的入口，负责定义：

- Test Suite 的 ID、名称、版本和描述；
- Test Suite 支持的 Actor；
- Test Case 列表；
- D 组四类结果的总体业务语义。

### 3.2 `testCases/`

`testCases/` 保存 GITB TDL Test Case。每个 XML 文件定义一个可独立执行的测试流程，包括输入、数据处理、验证步骤和最终输出。

本套件使用的主要 TDL 步骤是：

| TDL 步骤 | 作用 |
|---|---|
| `interact` | 接收测试人员上传的 JSON-LD |
| `assign` | 保存变量、转换类型或构造查询 |
| `process` | 调用 RDF、JSONPath 或 Collection handler 处理数据 |
| `send` | 向 Provider API 发送 HTTP 请求 |
| `verify` | 执行规则验证并生成验证报告 |
| `output` | 定义 Test Case 的最终结果说明 |

### 3.3 `Resources/`

| 文件 | 设计用途 |
|---|---|
| `building-energy-shapes_D.ttl` | 定义 metadata 字段、类型、数量、取值和字段范围约束 |
| `license-whitelist.json` | 定义 Authority 允许使用的许可证 IRI |
| `api-response.schema.json` | 定义 API JSON response 必须满足的结构 |
| `openapi.yaml` | 定义建筑能耗 API 合同，并作为 JSON Schema 的设计来源 |

Validator 运行时实际加载的规则位于：

```text
validator-config/energy/shapes/building-energy-shapes_D.ttl
```

`testsuite/Resources/building-energy-shapes_D.ttl` 是随 Test Suite 保存的规则副本。两份规则必须保持一致。

### 3.4 `Samples/`

Samples 用于 Test Suite 的正向和反向测试设计：

| Sample | 设计目的 |
|---|---|
| `data-product-valid.jsonld` | 验证合规 metadata 可以通过 TC01 和 TC02 |
| `data-product-invalid.jsonld` | 验证缺失字段和错误值能被 TC01、TC02 发现 |
| `data-product-invalid-http-endpoint.jsonld` | 验证 HTTP endpoint 会被要求 HTTPS 的 TC01 规则拒绝 |
| `data-product-license-disallowed.jsonld` | 验证白名单外许可证会被 TC02 拒绝 |
| `data-product-license-missing.jsonld` | 验证缺少许可证会被 TC02 拒绝 |
| `data-product-api-local-test.jsonld` | 为 TC03 的合法 API response 测试提供 endpoint |
| `data-product-api-local-invalid-response.jsonld` | 为 TC03 的非法 API response 测试提供 endpoint |

Samples 是测试数据，不是 Test Suite 执行逻辑的一部分，也不是正式业务数据。

## 4. TC01 Metadata Field Validation 设计

### 4.1 测试目的

TC01 检查 Provider 提交的 JSON-LD metadata 是否符合 D 组 Building Energy Metadata Profile。

### 4.2 输入

输入为一个 `application/ld+json` 文件，文件内容应描述一个建筑能耗 `dcat:Dataset`。

### 4.3 验证方式

TC01 使用 `verify` 步骤调用独立 SHACL Validator 的 GITB SOAP 接口，并传递：

| 参数 | 内容 |
|---|---|
| `contentToValidate` | 用户上传的 JSON-LD |
| `contentSyntax` | `application/ld+json` |
| `validationType` | `v1` |

`validationType=v1` 对应 Validator 的 `energy/v1` 配置，并加载 D 组 SHACL Shapes。

### 4.4 SHACL 规则设计

提交的 RDF graph 必须恰好包含一个 `dcat:Dataset`，Dataset 节点必须是 IRI。

必填字段为：

| 字段 | 约束 |
|---|---|
| `ex:datasetId` | 一个非空白 `xsd:string` |
| `dct:title` | 一个非空白 `xsd:string` |
| `ex:providerName` | 一个非空白 `xsd:string` |
| `dct:spatial` | 一个非空白 `xsd:string` |
| `dct:accrualPeriodicity` | 一个 `xsd:string`，精确等于 `hourly` |
| `ex:unit` | 一个 `xsd:string`，精确等于 `kWh` |
| `ex:temporalStart` | 一个 `xsd:date` |
| `ex:temporalEnd` | 一个 `xsd:date` |
| `dcat:endpointURL` | 一个以 `https://` 开头的 IRI |
| `dct:format` | 一个 `xsd:string`，精确等于 `application/json` |

可选字段为：

| 字段 | 约束 |
|---|---|
| `dct:description` | 如果存在，最多一个且为 `xsd:string` |
| `dct:license` | 如果存在，最多一个且为 HTTPS IRI |

补充约束为：

- `temporalStart` 不得晚于 `temporalEnd`；
- 单值字段通过 `sh:maxCount 1` 防止出现多个值；
- 必填字符串通过 `sh:minLength 1` 和 `sh:pattern "\\S"` 防止空白值；
- Closed Shape 检测 profile 未声明的 Dataset 属性；
- profile 外字段使用 `INAPPLICABLE:` 消息标识，并作为 Violation 拒绝。

### 4.5 TC01 输出

- 没有 SHACL Violation：TC01 成功；
- 存在 SHACL Violation：TC01 失败，并返回 focus node、result path、shape、value 和 message；
- metadata 无法解析或 Validator 无法调用：测试无法完成，业务上归类为 UNTESTABLE。

## 5. TC02 Licence Policy Validation 设计

### 5.1 测试目的

TC02 检查 metadata 是否声明了唯一许可证，以及该许可证是否符合 Authority 白名单政策。

TC01 中 `dct:license` 是可选字段；TC02 作为更严格的 onboarding 业务政策，要求许可证必须存在。因此 TC01 通过不代表 TC02 一定通过。

### 5.2 输入

输入为包含 `dct:license` 信息的 JSON-LD metadata。

### 5.3 验证流程

```text
JSON-LD metadata
    -> RdfUtils 解析 RDF
    -> SPARQL 提取 dct:license
    -> JsonPathProcessor 读取结果
    -> CollectionUtils 统计数量
    -> NumberValidator 检查数量等于 1
    -> 读取 license-whitelist.json
    -> CollectionUtils.contains 精确匹配
    -> ExpressionValidator 输出结果
```

TC02 按 RDF 语义提取许可证，不依赖 JSON-LD 文件具体使用 `license`、`dct:license` 或其他 context alias。

### 5.4 白名单设计

当前允许：

- `https://creativecommons.org/licenses/by/4.0/`
- `https://creativecommons.org/licenses/by-sa/4.0/`
- `https://creativecommons.org/publicdomain/zero/1.0/`

许可证 IRI 使用大小写敏感的完全匹配，不进行模糊匹配。

### 5.5 TC02 输出

- 恰好有一个 IRI 类型许可证且位于白名单：PASS；
- 没有许可证、存在多个许可证或值不是 IRI：FAIL；
- 许可证不在白名单：FAIL；
- metadata 无法解析或 ITB handler 无法执行：UNTESTABLE。

## 6. TC03 API Endpoint and Response Validation 设计

### 6.1 当前状态

TC03 在 Test Case XML 中设置了 `disabled="true"`。它属于保留设计，当前不参与 Conformance 判定。

### 6.2 测试目的

TC03 用于检查 metadata 声明的 API endpoint 是否可访问，以及 API response 是否符合建筑能耗数据接口合同。

### 6.3 设计流程

```text
JSON-LD metadata
    -> RdfUtils 和 SPARQL 提取 dcat:endpointURL
    -> 检查恰好存在一个 endpoint IRI
    -> HttpMessagingV2 发送 HTTP GET
    -> NumberValidator 检查 HTTP 200
    -> RegExpValidator 检查 application/json
    -> JsonValidator 按 JSON Schema 检查 response body
```

### 6.4 API response 规则

Response 根对象要求包含：

- `datasetId`；
- `providerName`；
- `records`。

每条 record 要求包含：

- `buildingId`；
- `meterId`；
- `timestamp`；
- `energyKWh`；
- `unit`，且值为 `kWh`。

JSON Schema 使用 `additionalProperties:false`，未在合同中声明的 response 字段会被拒绝。

## 7. 四类结果设计

D 组定义的业务优先级为：

```text
UNTESTABLE > FAIL > INAPPLICABLE > PASS
```

只有 PASS 可以通过 onboarding。

| 结果 | 定义 | 典型情况 |
|---|---|---|
| PASS | 输入可测试，且所有适用规则均满足 | 必填字段完整、值正确、许可证在白名单 |
| FAIL | 输入可测试，但违反已定义规则 | 缺少必填字段、类型错误、日期倒序、许可证不允许 |
| INAPPLICABLE | SUT 提交了规则范围之外的字段或内容 | Dataset 包含 profile 未声明的属性 |
| UNTESTABLE | 因输入解析或测试基础设施问题无法得出合规结论 | JSON-LD 无法解析、Validator 超时、handler 不可用 |

### 7.1 ITB 结果映射

当前 ITB 原生 Test Case 结果主要表现为 SUCCESS 或 FAILURE / ERROR：

| D 组业务结果 | 当前实现方式 |
|---|---|
| PASS | Test Case `SUCCESS` |
| FAIL | Test Case `FAILURE`，报告中说明违反的规则 |
| INAPPLICABLE | Closed Shape 生成带 `INAPPLICABLE:` 前缀的 Violation，Test Case 显示 `FAILURE` |
| UNTESTABLE | 解析、SOAP、handler、网络或超时错误导致 Test Case 无法正常完成 |

因此，四类结果是 D 组的业务分类；当前 ITB 界面尚未将它们显示为四个独立的原生状态。

### 7.2 可选字段空白值设计

D 组要求可选字段遵循：

- 未填写：PASS；
- 填写且合规：PASS；
- 填写空字符串、空格、多个空格或 TAB：按未填写处理；
- 填写非空但类型或格式错误：FAIL。

要完整实现该设计，需要在正式验证前删除仅由空白字符组成的可选字段值，或在对应 SHACL 规则中明确表达等价逻辑。当前 TC01 中的 `metadataOptionalBlankNormalization` 变量只记录这一设计意图，并未执行实际预处理。

## 8. Test Suite 准入判定

当前启用的准入逻辑为：

```text
TC01 Metadata Field Validation = PASS
AND
TC02 Licence Policy Validation = PASS
THEN
D 组 Test Suite = PASS
```

TC03 当前禁用，不参与上述判定。将来启用 TC03 后，准入条件扩展为：

```text
TC01 PASS
AND TC02 PASS
AND TC03 PASS
= D 组 Test Suite PASS
```

任一启用的 Test Case 出现 FAIL、INAPPLICABLE 或 UNTESTABLE，均不能判定 D 组准入通过。

## 9. 关键设计约束

- 正式 Test Suite 以 `3.2.0` 为当前版本；
- TC01 通过 SOAP 调用独立 SHACL Validator；
- TC02 使用 ITB 内置 handler，不依赖额外许可证 Validator；
- TC03 保留但禁用，不能计入当前已实现能力；
- Validator 中的运行时 TTL 与 Test Suite 中的规则副本必须一致；
- Samples 只用于测试，不应被当成 Test Suite 规则或正式业务数据；
- 四类业务结果与 ITB 原生 SUCCESS / FAILURE 状态需要区分；
- `INAPPLICABLE` 和 `UNTESTABLE` 的独立机器可读状态需要后续结果映射组件支持。
