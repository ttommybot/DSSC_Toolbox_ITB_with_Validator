## 1. 这个工具在 data space 架构中属于哪一层？

它属于 **测试与合规验证层**，可以把它理解成 data space onboarding 流程里的一道“质量门”。

它不负责真正传输建筑能耗数据，也不代替 connector、catalog 或 identity service。它主要负责在数据产品进入 data space 之前，按照已经制定好的规则检查 Provider 提交的内容是否合规，并保存测试结果。

在我们 D 组当前的 `3.2.0` Test Suite 设计中，ITB 负责组织三类测试：

- TC01：检查 JSON-LD metadata 是否符合 SHACL 规则；
- TC02：检查 metadata 中的许可证是否存在并且位于白名单中；
- TC03：检查 API endpoint 和 response，目前已经设计但暂时禁用。

所以，ITB 更像一个测试管理和执行平台，而真正的字段约束由 SHACL Shapes、许可证白名单和 JSON Schema 来定义。

## 2. 它解决的问题是什么，不解决的问题是什么？

它解决的主要问题是：把原本需要人工查看的准入要求，变成可以重复执行的 Test Case。

例如，Provider 上传一份建筑能耗 JSON-LD metadata 后，系统可以自动检查：必填字段有没有缺失、字段类型对不对、单位是不是 `kWh`、时间顺序是否正确、endpoint 是否使用 HTTPS，以及许可证是否位于 Authority 白名单中。测试完成后，ITB 还会保存具体的错误位置和测试报告。

但是它不解决下面这些问题：

- 不判断建筑能耗数据本身是否真实、准确；
- 不负责实际的数据传输；
- 不代替 connector、catalog 或 registry；
- 不验证 Provider 的真实身份和法律资质；
- 不完成 Gaia-X trust、VP-JWT 或完整 compliance 检查；
- 不做渗透测试、隐私审计或 API 性能测试。

简单来说，它回答的是“提交内容是否符合我们写下来的规则”，不能回答“数据是否真实”“整个参与方是否完全可信”。

## 3. 它的输入是什么，输出是什么？

从整个 ITB 方案来看，输入可以分成三类。

第一类是测试定义，包括：

- `testSuite.xml`；
- 三个 Test Case XML；
- SHACL TTL；
- 许可证白名单；
- JSON Schema。

第二类是 SUT 提交的测试数据。我们当前主要让测试人员在 ITB 页面上传 JSON-LD metadata。

第三类是 ITB 里的测试配置，例如 Actor、System 和 Conformance Statement。它们用来说明“哪个系统以什么身份，对哪个 Specification 进行测试”。

输出主要包括：

- 每个验证步骤是成功还是失败；
- SHACL 发现的 focus node、result path、错误值和错误消息；
- 许可证数量和白名单匹配结果；
- Test Case 的最终状态；
- 完整 Test Session 和可导出的测试报告。

需要注意，PASS、FAIL、INAPPLICABLE、UNTESTABLE 是我们 D 组定义的四种业务结果，目前 ITB 并没有把它们原生显示成四种独立状态：

- PASS 通常显示为 `SUCCESS`；
- FAIL 显示为 `FAILURE`；
- INAPPLICABLE 目前也显示为 `FAILURE`，通过 `INAPPLICABLE:` 错误消息区分；
- UNTESTABLE 通常表现为 JSON-LD 解析失败、SOAP 调用失败、handler 错误或超时。

另外，TC01 运行时真正加载的是 `validator-config/energy/shapes/building-energy-shapes_D.ttl`。Test Suite 的 `Resources` 目录中也保存了一份规则副本，两份内容必须保持一致。

## 4. 它依赖哪些标准或协议？

这个项目依赖的标准比较多，但每一种都有明确作用：

- **RDF**：metadata 的语义数据模型；
- **JSON-LD**：我们提交 metadata 时使用的 RDF 序列化格式；
- **Turtle**：编写 SHACL Shapes 时使用的文本格式；
- **SHACL**：定义字段是否必填、类型、数量、允许值和字段范围；
- **DCAT**：定义 `Dataset`、`endpointURL` 等数据目录概念；
- **DCTERMS**：定义 `title`、`license`、`format`、`spatial` 等 metadata 属性；
- **SPARQL**：TC02 用它从 RDF graph 中提取 `dct:license`；
- **GITB TDL**：定义 ITB 的 Test Suite 和 Test Case；
- **GITB SOAP Validation Service**：TC01 通过它调用独立 SHACL Validator；
- **HTTP**：TC03 将来用于访问 Provider API；
- **JSON Schema**：检查 API response 的 JSON 结构；
- **OpenAPI**：描述建筑能耗 API 合同，并作为 JSON Schema 的设计来源。

我们当前最核心的一条技术链路是：

```text
JSON-LD -> RDF -> SHACL Shapes -> GITB validation report -> ITB Test Session
```

## 5. 它和其他工具如何配合？

在真实 data space 中，ITB 通常不会单独完成全部工作，而是作为 onboarding 流程中的一个验证环节。

比较合理的配合方式是：Provider 准备数据产品和 metadata，catalog 或 registry 负责登记和发现，connector 负责实际数据交换，identity 和 compliance 工具检查参与方身份与信任要求，ITB 则负责执行已经定义好的符合性测试。

我们当前项目还没有把 connector、registry 和 compliance service 自动接入 ITB。现在的实际操作是测试人员在 ITB 页面手工上传 JSON-LD，然后运行 TC01 和 TC02。

因此，当前已经实现的是“人工上传后的自动验证”，而不是完整的“connector 自动提交、ITB 自动测试、catalog 自动准入”流水线。后续可以通过 ITB API、SOAP 或其他系统接口把这些步骤连接起来。

## 6. 它是否适合本地部署？部署成本在哪里？

适合本地部署。我们当前使用 Docker Compose 运行五个服务：

- `itb-ui`：ITB 网页界面；
- `itb-srv`：ITB Test Engine；
- `itb-mysql`：保存账号、配置和测试历史；
- `itb-redis`：缓存和运行协调；
- `shacl-validator`：独立 SHACL Validator。

本地部署本身不算特别困难，主要成本在配置和排错：

- 需要安装 Docker Desktop，并给容器提供足够的内存；
- MySQL 密码和 `.env` 必须正确配置；
- ITB 容器调用 Validator 时必须使用 Docker 内部地址，不能误用浏览器访问的 `localhost` 地址；
- SOAP WSDL、`validationType=v1` 和 Validator domain 必须一致；
- Test Suite ZIP 的目录层级必须正确；
- Validator 使用的 TTL 和 Test Suite 中保存的 TTL 副本必须同步；
- Docker 镜像升级后需要重新做回归测试。

所以它的主要成本不是购买许可证，而是学习 ITB 配置、维护 Docker 环境、编写测试用例和保证规则版本一致。

## 7. 它是否适合二次开发？源码结构是否清晰？

可以二次开发，但需要区分“改测试规则”和“改工具源码”。

对于我们 D 组来说，更适合修改的是：

- SHACL TTL；
- Test Case XML；
- 许可证白名单；
- JSON Schema 和 OpenAPI；
- Validator 的 `config.properties`。

这些修改不需要重新开发整个 ITB，也比较容易维护。

如果要修改 ITB 本身的页面、Test Engine 或报告机制，成本就会高很多，因为 ITB 是一套完整的平台。项目里还包含独立 `ISAITB/shacl-validator` 的 Java / Maven 源码，它本身也由 common、service、web、jar、war 等模块组成。

当前方案没有修改 ITB 或 Validator 的 Java 源码，而是通过配置、GITB TDL 和 SOAP 接口完成集成。这种方式更适合小组项目，因为可以继续使用官方镜像，也更方便将来升级。

如果以后必须让 ITB 原生显示 PASS、FAIL、INAPPLICABLE 和 UNTESTABLE 四种机器可读状态，可能需要开发 validator wrapper、custom validation service 或额外的结果映射组件。

## 8. 它的成熟度、维护活跃度、license 风险如何？

ITB 和 SHACL Validator 都由欧盟委员会相关团队维护，具有正式文档、Docker 镜像、API 和开源代码，整体成熟度比普通课程项目或个人工具高。

仓库中的 SHACL Validator `publiccode.yml` 标记软件版本为 `1.12.1`，并标记为 stable。不过我们当前 Docker Compose 使用的是：

```text
isaitb/shacl-validator:latest
```

这意味着实际运行版本可能随着镜像更新发生变化。正式环境最好固定经过测试的镜像版本或 image digest，避免某次更新后验证行为突然变化。

Validator 使用 **EUPL 1.2**。它允许使用、修改和再分发，但不能简单理解成“完全没有要求的宽松许可证”。如果修改后对外发布，需要保留版权和许可证说明、标明修改，并按照 EUPL 的要求提供源码或源码获取方式。

如果只是本地学习、内部测试和使用官方镜像，许可证风险通常较低；如果准备修改源码并对外发布，则应专门检查 EUPL 1.2 的署名、源码提供和 copyleft 要求。

## 9. 如果真实 data space 使用它，最大风险是什么？

最大的风险不是 ITB 能不能运行，而是“验证规则是否真的代表业务要求”。

如果 SHACL Profile 写得太严格，真实但合理的数据可能被错误拒绝；如果规则覆盖不完整，不合规数据又可能错误通过。例如，把 identifier 固定成某个值会误伤其他 Dataset，而没有检查时间顺序又会放过明显错误。

当前项目还有几个特别需要注意的风险：

- `testsuite/Resources/` 和 `validator-config/energy/shapes/` 各保存一份 TTL，如果两份没有同步，大家看到的规则可能和 Validator 实际执行的规则不同；
- ITB 当前没有原生的四状态输出，INAPPLICABLE 和 UNTESTABLE 可能与普通 FAIL 混在一起；
- TC03 还没有启用，因此目前不能证明 Provider API 真的可访问或 response 真的符合合同；
- Docker Compose 使用 `latest` 镜像，升级后可能产生行为变化；
- metadata 验证通过不等于参与方身份、安全、法律和数据质量全部合规。

所以真实使用时，最重要的是做好规则评审、版本固定、正反向回归测试和报告留档，并明确说明每个 Test Case 能证明什么、不能证明什么。
