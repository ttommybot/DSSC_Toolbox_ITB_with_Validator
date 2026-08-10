# DSSC 建筑能耗数据产品 Onboarding Test Suite


| Test Case | 当前状态 | Validator / 规则来源 |
|---|---|---|
| `TC01_METADATA_FIELD_VALIDATION` | 已接本地独立 SHACL Validator | `http://shacl-validator:8080/shacl/soap/energy/validation?wsdl`，`validationType=v1` |
| `TC02_API_RESPONSE_VALIDATION` | 已接 ITB `JsonValidator` | `Resources/api-response.schema.json`，由 `Resources/openapi.yaml` 的 200 response schema 抽取 |
| `TC03_LICENSE_POLICY_VALIDATION` | 保留测试逻辑，待接 license policy validator / wrapper | `Resources/license-whitelist.json` |

TC01 的 handler 地址是 Docker Compose 内部地址，供 `itb-srv` 容器访问 `shacl-validator` 容器。浏览器访问 validator 页面时使用 `http://localhost:8081/...`，test case handler 不应改成 `localhost:8081`。

## 目的

本测试套件用于验证 Energy Data Provider 提交的建筑能耗数据产品是否满足 Data Space 的准入要求。正式执行时由测试者在 ITB 页面上传实际文件，`Samples/` 只作为本地 demo 和人工 review 示例。

准入逻辑按设计保留：

```text
untestable > fail > inapplicable > pass
```

只有 `pass` 可以通过 onboarding；`fail`、`inapplicable`、`untestable` 都应被拒绝。当前 D 组 TTL 已将 profile 外字段设置为 `sh:Violation`，因此 `inapplicable` 会被现有 Validator 和 ITB 拒绝，但 ITB 原生状态显示为 `FAILURE`，具体业务原因通过报告中的 `INAPPLICABLE` 消息区分。若要在 ITB 中形成独立的四状态展示，或可靠区分 `untestable`，仍需要后续 validator wrapper 或 custom validation service。

## TC01 - Metadata Field Validation

Data Provider 上传完整 metadata JSON-LD。ITB 调用本地独立 SHACL Validator：

```text
http://shacl-validator:8080/shacl/soap/energy/validation?wsdl
```

传入参数：

| 参数 | 值 |
|---|---|
| `contentToValidate` | 用户上传的 metadata JSON-LD |
| `contentSyntax` | `application/ld+json` |
| `validationType` | `v1` |

`v1` 在本地 validator 配置中绑定 D 组规则文件：

```text
building-energy-shapes_D.ttl
```

## TC02 - API Response Validation

Data Provider 上传 metadata JSON-LD，以及从 metadata 中 `endpointURL` 对应接口获取到的 JSON API response。

当前 test case 使用 ITB 自带 `JsonValidator` 验证 API response：

```xml
<verify handler="JsonValidator">
```

校验 schema 为：

```text
Resources/api-response.schema.json
```

该 schema 从 `Resources/openapi.yaml` 的 200 response schema 抽取，并保留 `additionalProperties:false`，因此 OpenAPI 未声明的额外字段会被 JSON Schema validator 拒绝。

注意：最终自动化版本更理想的流程是只上传 metadata，由 wrapper/custom validator 从 metadata 提取 endpoint、请求 API、再验证 response。当前版本为了使用 ITB 自带 `JsonValidator`，需要测试者上传 API response JSON。

## TC03 - License Policy Validation

License 不应作为独立材料上传，而应从 metadata 的 `dct:license` / `license` 字段中提取。

当前 TC03 保留 license policy 流程设计和白名单资源：

```text
Resources/license-whitelist.json
```

ITB 本身没有专门的 license policy validator。如果只做字段格式检查，可以把规则写入 SHACL；如果要做白名单或业务 policy 判断，需要后续接 license validator / wrapper。

## Samples

当前只保留两个 metadata 示例：

| Sample | 作用 |
|---|---|
| `Samples/metadata-valid.jsonld` | 合规 metadata 示例 |
| `Samples/metadata-invalid.jsonld` | 不合规 metadata 示例 |


## 文件结构

```text
testsuite/
├── testSuite.xml
├── testCases/
│   ├── TC01_METADATA_FIELD_VALIDATION.xml
│   ├── TC02_API_RESPONSE_VALIDATION.xml
│   └── TC03_LICENSE_POLICY_VALIDATION.xml
├── Resources/
│   ├── building-energy-shapes_D.ttl
│   ├── api-response.schema.json
│   ├── openapi.yaml
│   └── license-whitelist.json
└── Samples/
    ├── metadata-valid.jsonld
    └── metadata-invalid.jsonld
```
