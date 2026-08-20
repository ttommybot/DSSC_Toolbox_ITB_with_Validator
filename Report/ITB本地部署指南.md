# ITB 本地部署与独立 Validator SOAP 接入指南（测试级）

本文档说明如何在 Windows 11 和 Docker Desktop 中部署本地 ITB，并使用同一个独立 SHACL Validator 提供两种用途：

- `energy` 域：供 ITB 通过 GITB SOAP Validation Service 接口调用项目固定规则；
- `any` 域：供人工通过网页同时上传待验证数据和自定义 SHACL `.ttl` 文件。

本项目使用的正式 SHACL 规则是：

```text
building-energy-shapes_D.ttl
```

官方参考文档：

- ITB 测试级安装：https://www.itb.ec.europa.eu/docs/guides/latest/installingTheTestBed/
- RDF Validator 配置：https://www.itb.ec.europa.eu/docs/guides/latest/validatingRDF/
- GITB TDL：https://www.itb.ec.europa.eu/docs/tdl/latest/
- GITB Validation Service：https://www.itb.ec.europa.eu/docs/services/latest/validation/

---

# 0. 当前接入方式和本次修改

## 0.1 接入架构

Validator 不安装到 ITB 容器内部。两者保持为独立服务，通过 Docker 内部网络和 SOAP 接口通信：

```text
用户在 ITB 上传 JSON-LD
        ↓
ITB 执行 GITB TDL verify 步骤
        ↓
读取 Validator SOAP WSDL
        ↓
http://shacl-validator:8080/shacl/soap/energy/validation?wsdl
        ↓
Validator 根据 validationType=v1
加载 building-energy-shapes_D.ttl
        ↓
返回 GITB 验证报告
        ↓
ITB 保存 PASS / FAIL 和详细错误
```

人工验证不经过 ITB，直接访问同一个 Validator 的 `any` 域：

```text
用户打开 http://localhost:8081/shacl/any/upload
        ↓
上传待验证的 RDF / JSON-LD 数据
        ↓
上传自定义 SHACL .ttl 文件
        ↓
Validator 执行验证并显示报告
```

## 0.2 本次完成的修改

1. 在 `testbed/docker-compose.yml` 中保留独立 `shacl-validator` 服务。
2. 为 Validator 增加 `validator.baseSoapEndpointUrl`，使 WSDL 公布 ITB 容器可访问的内部SOAP地址。
3. 让 `gitb-srv` 等待 `shacl-validator` 启动，减少 ITB 先启动而 Validator 尚未可用的问题。
4. 使用 `validator-config/energy/shapes/building-energy-shapes_D.ttl` 作为 `energy/v1` 的固定规则。
5. 新增 `testsuite-soap-smoke` 最小测试套件，使用完整 WSDL 地址作为 `verify` Handler。
6. 测试用例向 SOAP 服务传递正确参数：`contentToValidate`、`contentSyntax`、`validationType`。
7. 不再使用已不推荐的旧式 `module` 导入来表示外部 Validator。
8. 恢复 `validator-config/any` 通用验证域，允许人工上传自定义 `.ttl` 规则。

## 0.3 什么情况下算接入成功

只有同时满足以下条件，才能认为 Validator 已接入 ITB：

1. ITB 和 Validator 都能正常启动；
2. `energy` Validator 页面和 SOAP WSDL 能访问；
3. `any` Validator 页面能访问，并能同时选择待验证数据和自定义 SHACL 规则；
4. ITB 能成功导入 `testsuite-soap-smoke`；
5. 在 ITB 中上传合法 JSON-LD，测试会话显示成功；
6. 在 ITB 中上传非法 JSON-LD，验证步骤显示失败并包含 Validator 返回的 SHACL 报告；
7. Validator 日志中能够看到来自 ITB 的 SOAP 调用。

只看到两个网页都能打开，不能证明两者已经接入。

## 0.4 2026-08-10 实际验证状态

本次修改后已经验证：

- `docker compose config --quiet`通过；
- ITB、数据库、Redis和Validator五个容器均处于运行状态；
- Validator成功加载`energy`验证域和`v1`配置；
- Validator成功加载`any`验证域和`common`配置，并强制要求用户提供外部SHACL规则；
- `energy`和`any`上传页面均返回HTTP 200；
- Validator API同时列出`energy/v1`和`any/common`；
- `any`页面已显示`Include external shapes`和`External shapes`上传控件；
- `itb-srv`容器能够通过Docker内部网络读取Validator WSDL，返回HTTP 200；
- WSDL中的`soap:address`为`http://shacl-validator:8080/shacl/soap/energy/validation`；
- 最小Test Suite的两个XML文件均可正常解析；
- 已生成`testsuite-soap-smoke/energy-validator-soap-smoke.zip`，ZIP层级正确；
- 根目录规则与Validator加载副本的SHA-256一致。

尚需在ITB管理员界面完成一次性操作：把ZIP上传到目标Specification并实际运行Test Case。该操作需要当前ITB管理员登录状态，步骤见第8节。只有实际运行后，才能取得ITB测试会话中的最终SOAP验证报告。

---

# 1. 环境要求

建议环境：

- Windows 11；
- Docker Desktop；
- Docker Compose 2.0 或更高版本；
- PowerShell 7 或 Windows PowerShell。

确认 Docker Desktop 已启动：

```powershell
docker version
docker compose version
```

---

# 2. 项目目录结构

当前与接入有关的目录如下：

```text
ITB/
├── ITB本地部署指南.md
├── testbed/
│   ├── docker-compose.yml
│   ├── .env
│   └── .env.example
├── validator-config/
│   ├── any/
│   │   └── config.properties
│   └── energy/
│       ├── config.properties
│       └── shapes/
│           └── building-energy-shapes_D.ttl
├── testsuite-soap-smoke/
│   ├── testSuite.xml
│   ├── README.md
│   └── testCases/
│       └── tc-shacl-upload.xml
├── testsuite/
│   ├── testSuite.xml
│   ├── Resources/
│   │   └── building-energy-shapes_D.ttl
│   ├── Samples/
│   │   ├── data-product-valid.jsonld
│   │   └── data-product-invalid.jsonld
│   └── testCases/
│       ├── TC01_METADATA_FIELD_VALIDATION.xml
│       ├── TC02_API_RESPONSE_VALIDATION.xml
│       └── TC03_LICENSE_POLICY_VALIDATION.xml
└── validator/
    └── 官方 SHACL Validator 源码，仅供研究
```

`testsuite-soap-smoke` 只负责证明 SOAP 调用链已经打通，不替代后续完整的 Data Space onboarding Test Suite。

---

# 3. ITB 基础服务

## 3.1 环境变量

`testbed/.env` 至少需要：

```properties
MYSQL_ROOT_PASSWORD=gitb
MYSQL_USER=gitb
MYSQL_PASSWORD=gitb
MYSQL_DATABASE=gitb
DB_DEFAULT_PASSWORD=gitb
```

以上值只适合本地学习。生产环境必须使用随机强密码，并妥善保存 `.env`。

## 3.2 启动

进入部署目录：

```powershell
Set-Location "D:\FromC\Working Materials\TIDE_DSSC\DSSC_Tool_Learning\ITB\testbed"
```

检查 Compose 配置：

```powershell
docker compose config --quiet
```

启动服务：

```powershell
docker compose up -d
docker compose ps
```

预期包含五个容器：

```text
itb-mysql         healthy
itb-redis         Up
itb-srv           Up
itb-ui            Up
shacl-validator   Up
```

查看日志：

```powershell
docker logs itb-srv --tail 100
docker logs itb-ui --tail 100
docker logs shacl-validator --tail 100
```

## 3.3 登录

ITB 地址：

http://localhost:9000

默认管理员账号：

```text
admin@itb
```

首次临时密码可从 UI 日志中查找：

```powershell
docker logs itb-ui
```

首次登录后按界面要求修改密码。

---

# 4. 项目专用 Validator 配置

## 4.1 规则文件

完整 Test Suite 中维护的 D 组规则：

```text
ITB/testsuite/Resources/building-energy-shapes_D.ttl
```

Validator 实际加载的副本：

```text
ITB/validator-config/energy/shapes/building-energy-shapes_D.ttl
```

两个文件的规则内容必须一致。由于不同编辑器可能使用不同换行符，直接比较 SHA-256 可能不同；可按行检查内容：

```powershell
$suiteRules = Get-Content ".\testsuite\Resources\building-energy-shapes_D.ttl"
$validatorRules = Get-Content ".\validator-config\energy\shapes\building-energy-shapes_D.ttl"
Compare-Object $suiteRules $validatorRules
```

没有输出表示两份规则逐行一致。Validator运行时实际使用`validator-config/energy/shapes`中的副本。

## 4.2 `energy` 验证域

`validator-config/energy/config.properties`：

```properties
# 定义本 Validator 支持的验证类型
validator.type = v1

# 验证类型在网页中的显示名称
validator.typeLabel.v1 = Building Energy Metadata Profile v1.0

# v1 固定使用 D 组正式规则
validator.shaclFile.v1 = shapes/building-energy-shapes_D.ttl

# 网页标题
validator.uploadTitle = Building Energy Metadata Validator

# 同时启用网页、REST API 和 SOAP API
validator.channels = form, rest_api, soap_api

# 默认报告格式
validator.defaultReportSyntax = application/ld+json
```

因为 `v1` 已固定关联 `building-energy-shapes_D.ttl`，ITB 调用时只需要发送 JSON-LD 和 `validationType=v1`，不需要再次发送 TTL。

当前 D 组准入政策要求 profile 外字段也拒绝通过，因此 Closed Shape 使用`sh:Violation`而不是`sh:Warning`。出现多余字段时，Validator返回`FAILURE`，报告消息以`INAPPLICABLE:`开头；ITB原生状态显示为失败，但可以从报告消息区分普通`FAIL`与`INAPPLICABLE`。

## 4.3 `any` 通用验证域

`validator-config/any/config.properties`：

```properties
# 通用验证类型，不预置规则
validator.type = common
validator.typeLabel.common = Generic SHACL validator

# 允许用户上传自己的 SHACL shapes
validator.externalShapes.common = true

validator.uploadTitle = Generic SHACL Validator
validator.channels = form, rest_api, soap_api
validator.defaultReportSyntax = application/ld+json
```

`any` 域不绑定项目固定规则。人工使用时必须同时提供：

1. 待验证的数据文件，例如 JSON-LD、Turtle 或 RDF/XML；
2. 自定义 SHACL shapes 文件，例如 `.ttl`。

`energy` 与 `any` 两个域互不替代：`energy` 用于稳定、可复现的项目规则验证；`any` 用于临时实验任意 SHACL 规则。

---

# 5. Docker Compose 中的 SOAP 接入配置

## 5.1 Validator 服务

`testbed/docker-compose.yml` 中的关键配置：

```yaml
shacl-validator:
  image: isaitb/shacl-validator:latest
  container_name: shacl-validator
  restart: unless-stopped
  ports:
    - "8081:8080"
  environment:
    validator.resourceRoot: /validator/resources/
    validator.baseSoapEndpointUrl: http://shacl-validator:8080/shacl/soap/
  volumes:
    - ../validator-config:/validator/resources:ro
```

各项作用：

- `8081:8080`：Windows 通过 `localhost:8081` 访问容器内的 `8080`；
- `validator.resourceRoot`：加载 `validator-config` 中的 `energy` 和 `any` 验证域；
- `validator.baseSoapEndpointUrl`：让 WSDL 公布 Docker 内部可达的 SOAP 地址；
- `:ro`：以只读方式挂载规则，避免容器修改项目文件。

## 5.2 ITB 后端启动依赖

`gitb-srv` 的 `depends_on` 中包含：

```yaml
shacl-validator:
  condition: service_started
```

它只保证启动顺序，不替代运行时健康检查。最终仍应通过一次真实 ITB Test Case 调用来确认网络和接口正常。

## 5.3 应用修改

如果服务已经运行，需要重新创建 Validator 和 ITB 后端容器，使新的环境变量生效：

```powershell
Set-Location "D:\FromC\Working Materials\TIDE_DSSC\DSSC_Tool_Learning\ITB\testbed"

docker compose up -d --force-recreate shacl-validator gitb-srv
docker compose ps
```

这不会删除 MySQL 命名卷。不要附加 `-v`。

查看 Validator 启动日志，确认两个域都已加载：

```powershell
docker logs shacl-validator --tail 200
```

然后分别打开：

```text
http://localhost:8081/shacl/energy/upload
http://localhost:8081/shacl/any/upload
```

---

# 6. 地址与端口

## 6.1 Windows 主机访问

Validator 页面：

http://localhost:8081/shacl/energy/upload

通用人工验证页面：

http://localhost:8081/shacl/any/upload

REST API：

http://localhost:8081/shacl/energy/api/validate

Swagger：

http://localhost:8081/shacl/swagger-ui/index.html

SOAP WSDL：

http://localhost:8081/shacl/soap/energy/validation?wsdl

## 6.2 使用 `any` 域人工上传自定义 `.ttl`

1. 打开 http://localhost:8081/shacl/any/upload；
2. 在待验证内容区域选择数据文件，例如 `.jsonld`、`.ttl` 或 `.rdf`；
3. 根据数据文件选择正确的 Content syntax；
4. 在附加 SHACL rules / shapes 区域上传自己的 `.ttl` 规则文件；
5. 点击 Validate；
6. 查看页面中的 SUCCESS / FAILURE、错误数量、focus node、result path 和 message。

需要特别注意：

- 第 2 步上传的是“被检查的数据”；
- 第 4 步上传的是“检查规则”；
- 两个文件即使都使用 `.ttl` 后缀，作用也完全不同；
- `any` 域不会自动使用 `energy` 域的固定规则；
- 如需验证建筑能耗项目正式样例，应使用 `energy` 域，或在 `any` 域中明确上传项目正式 TTL。

## 6.3 ITB 容器访问

ITB Test Case 必须使用：

```text
http://shacl-validator:8080/shacl/soap/energy/validation?wsdl
```

不要在 TDL Handler 中写：

```text
http://localhost:8081/...
```

原因是 `localhost` 在 `itb-srv` 容器内表示 ITB 后端自己；`shacl-validator` 才是 Docker 内部服务名。

## 6.4 WSDL 与实际 SOAP Endpoint

WSDL说明地址：

```text
http://shacl-validator:8080/shacl/soap/energy/validation?wsdl
```

实际SOAP消息发送地址：

```text
http://shacl-validator:8080/shacl/soap/energy/validation
```

TDL只需要填写WSDL地址。ITB读取WSDL后会按照其中的服务合同构造并发送SOAP消息。

---

# 7. 最小 SOAP 冒烟测试套件

## 7.1 为什么单独建立冒烟套件

完整 `testsuite` 还包含 API 格式、许可证和负向验证等设计。为了避免这些尚未完成的测试影响SOAP接入判断，本项目新增了一个只包含一个Test Case的最小套件：

```text
testsuite-soap-smoke/
├── testSuite.xml
└── testCases/
    └── tc-shacl-upload.xml
```

它只验证：

```text
ITB 是否能调用独立 Validator 的 energy/v1 SOAP 接口。
```

## 7.2 Test Case 的核心内容

```xml
<interact id="userData" desc="上传待验证的建筑能耗元数据">
    <request name="content"
             desc="请选择 JSON-LD 元数据文件"
             inputType="UPLOAD"
             required="true"/>
</interact>

<verify handler="http://shacl-validator:8080/shacl/soap/energy/validation?wsdl"
        desc="调用独立 SHACL Validator 验证元数据">
    <input name="contentToValidate">$userData{content}</input>
    <input name="contentSyntax">"application/ld+json"</input>
    <input name="validationType">"v1"</input>
</verify>
```

三个输入的含义：

| 输入 | 含义 |
|---|---|
| `contentToValidate` | 用户上传的 JSON-LD 内容 |
| `contentSyntax` | 输入内容的 RDF 序列化格式 |
| `validationType` | 选择 `energy` 域中的 `v1`，从而加载 D 组正式规则 |

## 7.3 打包

进入冒烟套件目录：

```powershell
Set-Location "D:\FromC\Working Materials\TIDE_DSSC\DSSC_Tool_Learning\ITB\testsuite-soap-smoke"
```

创建ZIP：

```powershell
Compress-Archive `
  -Path ".\testSuite.xml", ".\testCases" `
  -DestinationPath ".\energy-validator-soap-smoke.zip" `
  -Force
```

检查ZIP内容：

```powershell
tar -tf ".\energy-validator-soap-smoke.zip"
```

ZIP根目录应直接包含：

```text
testSuite.xml
testCases/tc-shacl-upload.xml
```

不能在ZIP最外层再多包一层`testsuite-soap-smoke`目录。

---

# 8. 将冒烟套件上传到 ITB

## 8.1 准备测试配置

登录 http://localhost:9000 后：

1. 打开 `Domain management`；
2. 创建或选择一个Domain，例如`DSSC Energy`；
3. 创建或选择Specification，例如`Building Energy Metadata Profile v1.0`；
4. 进入该Specification的Test Suites区域；
5. 点击`Upload test suite`；
6. 选择`energy-validator-soap-smoke.zip`；
7. 点击继续并查看TDL验证结果；
8. 有错误时不要忽略，应根据文件位置修正；
9. 上传成功后应看到Actor `Metadata Provider`和一个Test Case。

测试套件已经完整定义Actor，因此ITB可以在上传时自动创建对应Actor。

## 8.2 建立受测系统

如果当前ITB中还没有测试组织和系统，需要在相应Community中：

1. 关联上面的Domain；
2. 创建测试Organisation；
3. 创建一个System，例如`Energy Data Provider Demo`；
4. 为该System选择`Metadata Provider` Actor，形成Conformance Statement。

完成后，System才能运行这个Test Case并保存测试历史。

## 8.3 执行合法样例

运行：

```text
Upload and validate Building Energy metadata
```

上传：

```text
ITB/testsuite/Samples/data-product-valid.jsonld
```

预期：

- `verify`步骤调用独立Validator；
- Validator加载`energy/v1`；
- 测试会话显示成功；
- 报告中没有SHACL Violation。

## 8.4 执行非法样例

再次运行同一Test Case，上传：

```text
ITB/testsuite/Samples/data-product-invalid.jsonld
```

预期：

- SOAP调用本身成功；
- SHACL验证结果失败；
- ITB的`verify`步骤显示失败；
- 失败步骤中能够看到Validator返回的详细报告。

这里的Test Case是“验证提交内容是否合规”，所以非法样例导致Test Case失败是正确行为。后续如果要设计“Validator成功识别错误，所以负向测试本身通过”，需要另外编写结果反转或断言逻辑。

---

# 9. 如何确认请求确实经过 SOAP

## 9.1 查看 Validator 日志

执行测试时，在另一个PowerShell窗口运行：

```powershell
docker logs -f shacl-validator
```

ITB执行`verify`步骤时，日志中应出现SOAP验证调用或相应验证记录。

## 9.2 查看 ITB 测试会话

测试会话中应存在：

```text
调用独立 SHACL Validator 验证元数据
```

点击该步骤应能查看Validator返回的GITB验证报告。

## 9.3 断开测试

可在没有重要测试运行时暂时停止Validator：

```powershell
docker compose stop shacl-validator
```

此时再次运行Test Case，ITB应报告Handler连接失败。恢复后：

```powershell
docker compose start shacl-validator
```

如果停止Validator不影响ITB测试结果，说明测试用例并未真正调用该服务。

---

# 10. 常见问题

## 10.1 Validator 页面无法打开

```powershell
docker ps -a --filter "name=shacl-validator"
docker logs shacl-validator --tail 200
```

确认主机端口映射为：

```text
8081:8080
```

## 10.2 找不到 `energy` 验证域

检查：

- `validator-config/energy/config.properties`是否存在；
- `validator.resourceRoot`是否为`/validator/resources/`；
- 挂载是否为`../validator-config:/validator/resources:ro`；
- 规则相对路径是否正确。

## 10.3 找不到 `any` 验证域

检查：

- `validator-config/any/config.properties` 是否存在；
- 文件中是否包含 `validator.externalShapes.common = true`；
- Compose 挂载是否为 `../validator-config:/validator/resources:ro`；
- 修改配置后是否重新创建或重启了 `shacl-validator`；
- 启动日志中是否出现 `any` 域加载失败信息。

`any` 域地址必须是：

```text
http://localhost:8081/shacl/any/upload
```

## 10.4 `any` 页面没有自定义规则上传项

确认 `validator-config/any/config.properties` 包含：

```properties
validator.externalShapes.common = true
```

保存后重新创建 Validator 容器：

```powershell
docker compose up -d --force-recreate shacl-validator
```

## 10.5 找不到 `energy` 规则文件

配置必须是：

```properties
validator.shaclFile.v1 = shapes/building-energy-shapes_D.ttl
```

文件必须位于：

```text
validator-config/energy/shapes/building-energy-shapes_D.ttl
```

## 10.6 WSDL能打开，但ITB无法调用

重点检查：

1. TDL Handler是否使用`shacl-validator:8080`；
2. 是否误用了`localhost:8081`；
3. WSDL中的`soap:address`是否为Docker内部可达地址；
4. `contentToValidate`、`contentSyntax`、`validationType`名称是否正确；
5. 是否仍在使用旧式`module`导入；
6. `validationType`是否为配置中存在的`v1`。

## 10.7 测试套件上传失败

确认：

- ZIP根目录直接包含`testSuite.xml`；
- Test Case XML位于ZIP内部；
- `testSuite.xml`引用的Test Case ID与Test Case文件根元素的ID完全一致；
- XML编码为UTF-8；
- XML能够正常解析；
- 没有把README等非TDL XML误放进压缩包。

## 10.8 合法样例意外失败

检查：

1. Validator加载的是否是`building-energy-shapes_D.ttl`；
2. 根目录规则与Validator副本的SHA-256是否一致；
3. JSON-LD上下文、字段名称和日期类型是否满足D组正式规则；
4. Validator日志中是否有RDF解析错误；
5. ITB传入的`contentSyntax`是否为`application/ld+json`。

---

# 11. 停止和恢复

只停止Validator：

```powershell
docker compose stop shacl-validator
```

重新启动Validator：

```powershell
docker compose start shacl-validator
```

停止完整环境并保留数据库和测试历史：

```powershell
docker compose down
```

重新启动：

```powershell
docker compose up -d
```

不要在不理解影响时使用：

```text
docker compose down -v
```

`-v`会删除MySQL等命名卷中的数据，可能丢失账号、配置和测试历史。

---

# 12. 当前范围和后续工作

当前 Validator 同时提供两个用途：

```text
自动/集成：ITB -> energy/v1 -> building-energy-shapes_D.ttl
人工/临时：浏览器 -> any/common -> 用户上传的自定义 SHACL .ttl
```

当前冒烟套件没有处理：

- API响应格式验证；
- 许可证白名单验证；
- 将非法样例识别转换为“负向测试通过”；
- 自动从真实Provider系统获取元数据；
- 完整Data Space onboarding流程。

建议先确认冒烟套件能够在ITB中成功运行，再逐步把SOAP `verify`步骤合并进完整`testsuite`，避免同时排查TDL、SOAP、SHACL、API和业务逻辑问题。
