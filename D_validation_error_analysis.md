# D 组 Validation Error Analysis

## 1. 文档目的

本文档分析 D 组建筑能耗数据产品元数据的四份实际 ITB 验证报告，回答以下问题：

- 正确 JSON-LD 为什么通过 TC01 和 TC02；
- 错误 JSON-LD 为什么被 TC01 和 TC02 拒绝；
- 每一项失败对应哪个输入字段、规则和测试步骤；
- 四份报告能够证明什么，以及不能证明什么。

## 2. 分析范围

本分析严格限定为以下两个原始 JSON-LD 文件：

- `Report/交付B组/metadata/data-product-valid.jsonld`
- `Report/交付B组/metadata/data-product-invalid.jsonld`

测试矩阵为：

```text
(TC01 + TC02) × (valid + invalid) = 4 份报告
```

对应报告为：

- `Report/交付B组/validation-reports/valid_TC01.pdf`
- `Report/交付B组/validation-reports/valid_TC02.pdf`
- `Report/交付B组/validation-reports/Invalid_TC01.pdf`
- `Report/交付B组/validation-reports/Invalid_TC02.pdf`

`testsuite/Samples/` 下的其他 JSON-LD 文件不在本文档分析范围内，也不作为本文档结论的证据。

## 3. 测试对象与验证逻辑

### 3.1 TC01：Metadata Field Validation

TC01 将用户上传的 JSON-LD 发送给独立 SHACL Validator。Validator 把 JSON-LD 解析为 RDF 图，并根据 `building-energy-shapes_D.ttl` 检查：

- Dataset 数量和节点类型；
- 必填字段是否存在；
- 单值字段是否只出现一次；
- 字段 datatype 是否正确；
- 字符串是否为空白；
- `frequency`、`unit`、`format` 是否为允许值；
- endpoint 是否为 HTTPS IRI；
- 起止日期类型和先后顺序；
- 是否存在 Profile 未声明的额外属性。

TC01 的成功表示“该文件在本次测试所加载的 SHACL Profile 下没有产生 Violation”。

### 3.2 TC02：Licence Policy Validation

TC02 不复用 TC01 的最终判断，而是独立执行许可证政策检查：

1. 按 RDF 语义提取 `dct:license`；
2. 检查许可证数量是否恰好为 1；
3. 检查该值是否为 IRI；
4. 检查许可证 IRI 是否存在于 Authority 白名单。

当前白名单包含：

- `https://creativecommons.org/licenses/by/4.0/`
- `https://creativecommons.org/licenses/by-sa/4.0/`
- `https://creativecommons.org/publicdomain/zero/1.0/`

TC01 中的 SHACL `LicenseShape` 将许可证定义为可选字段；TC02 的 onboarding 政策更严格，要求许可证必须存在。因此 TC01 通过不能替代 TC02 通过。

## 4. 两个原始输入的关键差异

| 检查项 | valid JSON-LD | invalid JSON-LD | 对测试的影响 |
|---|---|---|---|
| Dataset IRI | `.../building-energy-hourly-v1` | `.../building-energy-hourly-v1-invalid` | 用作报告中的 Focus node |
| `providerName` | `Energy Data Provider Ltd.` | 缺失 | invalid 在 TC01 失败 |
| `unit` | `kWh` | `MWh` | invalid 在 TC01 失败 |
| `temporalEnd` | `2026-05-02` | 缺失 | invalid 在 TC01 失败 |
| `license` | CC BY 4.0 IRI | 缺失 | invalid 在 TC02 失败 |
| 其他本次被报告字段 | 满足约束 | 未产生额外报告错误 | 不改变四报告结论 |

这里的“未产生额外报告错误”仅表示报告没有列出其他 Violation，不表示 invalid 文件的业务内容在所有方面都正确。

## 5. 四份报告结果总览

| 输入 | Test Case | PDF 结果 | Findings | 结论 |
|---|---|---:|---:|---|
| valid | TC01 | `SUCCESS` | 0 error | 元数据满足本次 SHACL 字段规则 |
| valid | TC02 | `SUCCESS` | 0 error，2 条成功信息 | 许可证数量及白名单均通过 |
| invalid | TC01 | `FAILURE` | 3 errors | 单位错误并缺少两个必填字段 |
| invalid | TC02 | `FAILURE` | 1 error | 未声明许可证 |

## 6. 报告一：valid + TC01

### 6.1 报告证据

| 项目 | 报告内容 |
|---|---|
| 报告 | `valid_TC01.pdf` |
| Test name | `Metadata Field Validation - D Priority Logic` |
| 结果 | `SUCCESS` |
| 开始时间 | `11/08/2026 17:32:34` |
| 结束时间 | `11/08/2026 17:32:39` |
| Session ID | `d7a9c484-2c4b-4ae7-bebc-fe6bd87fc4a3` |
| 页数 | 4 |

报告显示：

1. 上传 JSON-LD 的交互步骤为 `SUCCESS`；
2. 调用官方 SHACL Validator SOAP API 的验证步骤为 `SUCCESS`；
3. 测试会话以 `COMPLETED` 结束；
4. 报告没有列出 SHACL error、warning 或 violation。

### 6.2 为什么通过

根据原始 valid JSON-LD 与当前规则，可核对出：

- `unit` 为精确值 `kWh`；
- `providerName` 存在且为非空字符串；
- `temporalStart = 2026-05-01`；
- `temporalEnd = 2026-05-02`；
- 起始日期不晚于结束日期；
- `frequency = hourly`；
- `format = application/json`；
- `endpointUrl` 为 HTTPS IRI；
- 必填字段均存在。

其中字段级原因是根据“输入文件 + SHACL 规则”进行的解释；PDF 本身直接证明的是 Validator 步骤和整个 Test Case 均为 `SUCCESS`。

### 6.3 结论

`data-product-valid.jsonld` 满足 TC01 所加载的 D 组元数据 Profile，可以作为元数据字段符合性证据。

## 7. 报告二：valid + TC02

### 7.1 报告证据

| 项目 | 报告内容 |
|---|---|
| 报告 | `valid_TC02.pdf` |
| Test name | `TC02 - Licence Policy Validation` |
| 结果 | `SUCCESS` |
| 开始时间 | `11/08/2026 17:32:41` |
| 结束时间 | `11/08/2026 17:32:45` |
| Session ID | `92f22731-237c-4e0f-bceb-d11be39cd0c7` |
| 页数 | 5 |

报告中的许可证数量检查显示：

```text
actual   = 1.0
expected = 1.0
```

报告随后给出第二条成功信息：

```text
The declared licence is present in the Authority whitelist.
```

### 7.2 为什么通过

原始 valid JSON-LD 声明：

```text
https://creativecommons.org/licenses/by/4.0/
```

JSON-LD Context 将 `license` 映射为 `dct:license`，并将值声明为 `@id`。因此 RDF 语义处理得到一个 IRI 类型许可证。

该 IRI 恰好出现一次，并且存在于 Authority 白名单，所以数量检查和白名单检查均成功。

### 7.3 结论

`data-product-valid.jsonld` 满足 TC02 的“恰好一个 IRI 许可证 + Authority 白名单”政策。

## 8. 报告三：invalid + TC01

### 8.1 报告证据

| 项目 | 报告内容 |
|---|---|
| 报告 | `Invalid_TC01.pdf` |
| Test name | `Metadata Field Validation - D Priority Logic` |
| 结果 | `FAILURE` |
| 开始时间 | `11/08/2026 17:31:56` |
| 结束时间 | `11/08/2026 17:32:13` |
| Session ID | `a2c4433d-5209-4b64-9b23-b56d4202c38c` |
| Findings | 3 errors、0 warnings、0 messages |
| 页数 | 4 |

上传步骤成功，说明文件已被 ITB 接收。失败发生在 SHACL Validator 验证步骤，因此这是数据不合规，而不是“没有选中文件”或“上传失败”。

三个错误均指向同一 Focus node：

```text
https://example.org/dssc-energy/datasets/building-energy-hourly-v1-invalid
```

### 8.2 Error 1：`unit` 值错误

| 项目 | 内容 |
|---|---|
| Result path | `https://example.org/dssc-energy#unit` |
| Shape | `https://example.org/dssc-energy#UnitShape` |
| 实际值 | `MWh` |
| 规则要求 | 精确等于 `kWh`，区分大小写 |

报告消息：

```text
ex:unit is required and must equal kWh (exact match, case-sensitive).
```

这不是单位换算检查。规则要求元数据使用统一的规范值 `kWh`，所以即使 `MWh` 在物理意义上可以换算，也不满足该 Profile 的精确枚举约束。

### 8.3 Error 2：缺少 `temporalEnd`

| 项目 | 内容 |
|---|---|
| Result path | `https://example.org/dssc-energy#temporalEnd` |
| Shape | `https://example.org/dssc-energy#TemporalEndShape` |
| 实际数量 | 0 |
| 规则要求 | 恰好一个 `xsd:date` |

报告消息：

```text
ex:temporalEnd is required and must be one xsd:date.
```

invalid JSON-LD 只提供了 `temporalStart`，没有提供 `temporalEnd`，所以违反 `sh:minCount 1`。

### 8.4 Error 3：缺少 `providerName`

| 项目 | 内容 |
|---|---|
| Result path | `https://example.org/dssc-energy#providerName` |
| Shape | `https://example.org/dssc-energy#ProviderNameShape` |
| 实际数量 | 0 |
| 规则要求 | 恰好一个非空 `xsd:string` |

报告消息：

```text
ex:providerName is required and must be one non-blank xsd:string.
```

invalid JSON-LD 完全缺少该字段，因此在数量检查阶段失败。若字段存在但只含空格，也会被非空白约束拒绝。

### 8.5 结论

invalid 文件在 TC01 中被正确拒绝。三个 Finding 都能由原始输入直接复现，属于确定的数据质量/规范符合性错误，不是 Validator 基础设施故障。

## 9. 报告四：invalid + TC02

### 9.1 报告证据

| 项目 | 报告内容 |
|---|---|
| 报告 | `Invalid_TC02.pdf` |
| Test name | `TC02 - Licence Policy Validation` |
| 结果 | `FAILURE` |
| 开始时间 | `11/08/2026 17:32:14` |
| 结束时间 | `11/08/2026 17:32:24` |
| Session ID | `9ced8197-fee8-4388-8a90-31b91ee17578` |
| Findings | 1 error、0 warnings、0 messages |
| 页数 | 4 |

报告中的数量检查显示：

```text
actual   = 0.0
expected = 1.0
```

报告消息：

```text
Metadata must declare exactly one IRI-valued dct:license.
```

### 9.2 为什么失败

invalid JSON-LD 中没有 `license` 或 `dct:license`。RDF 语义查询提取到的许可证数量为 0，而 TC02 要求数量恰好为 1。

数量检查失败后，Test Case 不再继续执行白名单匹配。因此本报告只能得出“许可证缺失”，不能把它解释为“许可证存在但不在白名单”。

### 9.3 结论

invalid 文件违反 TC02 的强制许可证政策，被正确拒绝。

## 10. TC01 与 TC02 的边界

| 问题 | TC01 | TC02 |
|---|---|---|
| 元数据必填字段、类型和值 | 检查 | 不负责 |
| `license` 如果存在是否为 HTTPS IRI | 检查 | 作为 IRI 数量检查的一部分 |
| `license` 是否必须存在 | SHACL 中不是必填 | 必须恰好一个 |
| 许可证是否在 Authority 白名单 | 不检查 | 检查 |
| 执行组件 | 独立 SHACL Validator | ITB 内置 RDF/集合/验证 Handler |

最终准入逻辑应为：

```text
TC01 SUCCESS
AND TC02 SUCCESS
= D 组元数据准入通过
```

任何一个启用的 Test Case 失败，都不能判定该元数据通过 D 组准入。

## 11. 失败分类

本次四报告只实际出现两类界面结果：`SUCCESS` 和 `FAILURE`。

| D 组业务分类 | 本次是否出现 | 说明 |
|---|---:|---|
| PASS | 是 | 两份 valid 报告 |
| FAIL | 是 | 两份 invalid 报告 |
| INAPPLICABLE | 否 | 本次两个原始 JSON-LD 没有触发 profile 外字段报告 |
| UNTESTABLE | 否 | 四次运行均成功接收文件并执行到验证逻辑 |

invalid 报告中的 `ERROR` 会话日志表示测试以错误结果结束；结合报告中的明确规则 Finding，可判定其业务原因是输入违反规则，而不是无法测试。

## 12. 证据完整性

### 12.1 输入文件 SHA-256

| 文件 | SHA-256 |
|---|---|
| `data-product-valid.jsonld` | `045A44A3180F3781579F95B762E25C25F32A34EAD418D1863800BFBDF4728B10` |
| `data-product-invalid.jsonld` | `0ADE622953414C5F53FDC40300972C6D47C6EA6FDF28E12071F7EA6EB86F7BD2` |

### 12.2 报告文件 SHA-256

| 报告 | SHA-256 |
|---|---|
| `valid_TC01.pdf` | `1B5D98E4380FEF330ACB8DE5FBE15144BF94269CEB5C947383AF87A78C33E767` |
| `valid_TC02.pdf` | `DEBAE9C0D834D4284AE277F502021F93D57F4986825231623FCFA51145FD35A9` |
| `Invalid_TC01.pdf` | `1EFC9CC791DD49E6D27574300809F1E45AA861304F5045A8E145F0ACC88FF7B4` |
| `Invalid_TC02.pdf` | `F00A328DB7F10F40564CB76D2BEC24C56FE50C3750D8631B46E26DFF3AA22A8F` |

### 12.3 规则副本一致性

当前以下三份 TTL 的 SHA-256 均为：

```text
A6A5D3E408B735906512A7A2EC04CA1CB6EF6C6E605B3DFD0E5F09137F8B1307
```

- `Report/交付B组/shacl-rules/building-energy-shapes_D.ttl`
- `testsuite/Resources/building-energy-shapes_D.ttl`
- `validator-config/energy/shapes/building-energy-shapes_D.ttl`

这证明当前交付规则、Test Suite 规则副本和 Validator 运行时规则副本一致。该一致性检查本身不能反向证明 2026-08-11 生成 PDF 时加载的文件字节完全相同；若要达到严格审计级复现，还应在每次会话中同时保存运行时 TTL 哈希和 Validator 请求/响应记录。若任何规则文件发生修改，应重新运行四次测试并重新生成报告。


## 13. 最终结论

1. 原始 `data-product-valid.jsonld` 在 TC01 和 TC02 中均为 `SUCCESS`，满足当前 D 组元数据字段规则和许可证准入政策。
2. 原始 `data-product-invalid.jsonld` 在 TC01 中产生 3 项确定错误：`unit` 值错误、缺少 `temporalEnd`、缺少 `providerName`。
3. 同一 invalid 文件在 TC02 中因缺少 `dct:license` 失败，实际数量为 0，预期数量为 1。
4. 四份报告的结果与两个原始 JSON-LD 的实际内容一致，没有发现结果与输入矛盾。
5. 正式 onboarding 应同时要求 TC01 和 TC02 成功；任一失败都应拒绝准入。
