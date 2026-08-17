# DSSC 建筑能耗数据产品 Onboarding Test Suite


| Test Case | 当前状态 | Validator / 规则来源 |
|---|---|---|
| `TC01_METADATA_FIELD_VALIDATION` | 已接本地独立 SHACL Validator | `http://shacl-validator:8080/shacl/soap/energy/validation?wsdl`，`validationType=v1` |
| `TC02_LICENSE_POLICY_VALIDATION` | 已启用 RDF 语义提取与白名单校验 | `RdfUtils`、`JsonPathProcessor`、`CollectionUtils`、`ExpressionValidator` |
| `TC03_API_RESPONSE_VALIDATION` | 已保留但禁用；有真实 API 后可重新启用 | `RdfUtils`、`HttpMessagingV2`、`NumberValidator`、`RegExpValidator`、`JsonValidator` |

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

## TC02 - License Policy Validation

License 不应作为独立材料上传，而应从 metadata 的 `dct:license` / `license` 字段中提取。

TC02 按以下顺序自动执行：

1. 使用 `RdfUtils` 和 SPARQL 按 RDF 语义提取 `dct:license`；
2. 检查恰好存在一个 IRI 类型的 Licence；
3. 使用 `JsonPathProcessor` 读取白名单资源中的 `allowedLicenses`；
4. 使用 `CollectionUtils` 进行大小写敏感的精确匹配；
5. 使用 `ExpressionValidator` 将匹配结果转换为正式 PASS / FAIL。

白名单资源：

```text
Resources/license-whitelist.json
```

TC02 使用 ITB 内置 handler，不需要 Licence Validator、wrapper，也不需要修改 SHACL TTL。

## TC03 - API Response Validation（当前禁用）

TC03 在 XML 中设置了 `disabled="true"`。普通测试者默认看不到，也不能执行，结果不计入 Conformance。以下逻辑完整保留，等有真实 API 时可以重新启用。

Data Provider 只上传 metadata JSON-LD，不再人工上传 API response。启用后的 TC03 按以下顺序自动执行：

1. 使用 `RdfUtils` 和 SPARQL 按 RDF 语义提取 `dcat:endpointURL`；
2. 检查恰好存在一个 IRI 类型的 endpoint；
3. 使用 `HttpMessagingV2` 对 endpoint 发送 HTTP GET；
4. 检查响应状态为 HTTP 200；
5. 检查响应 `Content-Type` 为 `application/json`，允许附带 `charset` 等参数；
6. 使用 `JsonValidator` 按 JSON Schema 检查响应体。

校验 schema 为：

```text
Resources/api-response.schema.json
```

该 schema 从 `Resources/openapi.yaml` 的 200 response schema 抽取，并保留 `additionalProperties:false`，因此 OpenAPI 未声明的额外字段会被 JSON Schema validator 拒绝。

TC03 使用 ITB 内置 handler，不需要 API wrapper 或新的 Validator。重新启用后，endpoint 必须能够从 `itb-srv` 容器访问。

## 运行前提

- TC01 继续使用现有 SHACL Validator；本次没有修改 TTL，也没有修改 Validator 源码。
- TC02、TC03 不需要加载新的外部 Validator，也不需要部署 wrapper。
- TC02、TC03 使用较新的 ITB 内置 handler。建议使用 ITB 1.28.0 或更高版本；
  当前本地 Docker Compose 使用的 `isaitb/gitb-*` 镜像满足这一设计方向。
- 当前 TC03 已禁用；以后重新启用时，其 endpoint 必须能从 `itb-srv` 容器访问，而不仅仅是能从浏览器访问。

## Samples

当前提供以下正向和反向测试示例：

| Sample | 作用 |
|---|---|
| `Samples/data-product-valid.jsonld` | 合规 metadata 示例 |
| `Samples/data-product-invalid.jsonld` | TC01 不合规 metadata 示例 |
| `Samples/data-product-api-local-test.jsonld` | 已禁用 TC03 的本地 HTTP mock API 示例；不满足 TC01 的 HTTPS 规则 |
| `Samples/data-product-api-local-invalid-response.jsonld` | 已禁用 TC03 的反向示例；endpoint 返回不符合 Schema 的 JSON |
| `Samples/data-product-license-disallowed.jsonld` | 结构有效但 Licence 不在白名单，TC02 应失败 |
| `Samples/data-product-license-missing.jsonld` | 结构有效但缺少 Licence，TC02 应失败 |
| `Samples/mock-api/api-response-valid.json` | TC03 本地 mock API 合法响应 |
| `Samples/mock-api/api-response-invalid.json` | TC03 本地 mock API 非法响应 |


## 文件结构

```text
testsuite/
├── testSuite.xml
├── testCases/
│   ├── TC01_METADATA_FIELD_VALIDATION.xml
│   ├── TC02_LICENSE_POLICY_VALIDATION.xml
│   └── TC03_API_RESPONSE_VALIDATION.xml
├── Resources/
│   ├── building-energy-shapes_D.ttl
│   ├── api-response.schema.json
│   ├── openapi.yaml
│   └── license-whitelist.json
└── Samples/
    ├── data-product-valid.jsonld
    ├── data-product-invalid.jsonld
    ├── data-product-invalid-http-endpoint.jsonld
    ├── data-product-api-local-test.jsonld
    ├── data-product-api-local-invalid-response.jsonld
    ├── data-product-license-disallowed.jsonld
    ├── data-product-license-missing.jsonld
    └── mock-api/
        ├── api-response-valid.json
        └── api-response-invalid.json
```

## 本地组装 Test Suite

ZIP 根目录必须直接包含 `testSuite.xml`，不能额外包一层 `testsuite` 目录。在 PowerShell 中执行：

```powershell
Set-Location "D:\FromC\Working Materials\TIDE_DSSC\DSSC_Tool_Learning\ITB\testsuite"

tar -a -c `
  -f ".\dssc-energy-onboarding-testsuite-3.2.0.zip" `
  ".\testSuite.xml" `
  ".\testCases" `
  ".\Resources\building-energy-shapes_D.ttl" `
  ".\Resources\api-response.schema.json" `
  ".\Resources\license-whitelist.json"

tar -tf ".\dssc-energy-onboarding-testsuite-3.2.0.zip"
```

预期 ZIP 根目录包含：

```text
testSuite.xml
testCases/...
Resources/building-energy-shapes_D.ttl
Resources/api-response.schema.json
Resources/license-whitelist.json
```

`Samples/` 和 `Resources/openapi.yaml` 保留在本地开发目录中，但不放入最终上传包。
它们不参与 test case 运行；从生产 ZIP 中排除可以减少 ITB 对未使用资源的警告。

## TC03 本地 mock API（重新启用后使用）

TC03 当前禁用。重新启用后，它必须访问真实 endpoint。为了在本机测试，可以在 `Samples` 目录启动只读静态 HTTP 服务：

```powershell
Set-Location "D:\FromC\Working Materials\TIDE_DSSC\DSSC_Tool_Learning\ITB\testsuite\Samples"
python -m http.server 8765
```

不要关闭这个窗口。`data-product-api-local-test.jsonld` 中的 endpoint 已配置为：

```text
http://host.docker.internal:8765/mock-api/api-response-valid.json
```

该地址供 Docker 中的 `itb-srv` 访问 Windows 宿主机。这个 metadata 只用于 TC03 本地功能测试，因为它使用 HTTP，不满足 TC01 的 HTTPS 生产规则。

若要测试 TC03 的 Schema 失败结果，直接上传
`data-product-api-local-invalid-response.jsonld`。它已指向
`http://host.docker.internal:8765/mock-api/api-response-invalid.json`。

## 上传与运行

1. 启动 ITB：进入 `ITB/testbed`，执行 `docker compose up -d`；
2. 打开 `http://localhost:9000` 并使用管理员账号登录；
3. 在 `Domain management` 中创建或选择 Domain；
4. 创建或选择 Specification；
5. 进入 Test Suites，上传 `dssc-energy-onboarding-testsuite-3.2.0.zip`；
6. 创建测试 Organisation 和 System，并为 System 选择 `Energy Data Provider` Actor；
7. 运行 TC01 和 TC02；TC03 默认隐藏且不可执行。

推荐测试矩阵：

| Test Case | 上传文件 | 预期 |
|---|---|---|
| TC01 | `data-product-valid.jsonld` | PASS |
| TC01 | `data-product-invalid.jsonld` | FAIL |
| TC02 | `data-product-valid.jsonld` | PASS |
| TC02 | `data-product-license-disallowed.jsonld` | FAIL |
| TC02 | `data-product-license-missing.jsonld` | FAIL |
| TC03（禁用） | 不执行 | 默认隐藏，不影响 Conformance |
