# ITB 本地部署指南（测试级）

本文档为基于官方文档与实操经验，在windows系统上本地部署测试级ITB & Validator的指南。

官方安装文档：

测试级：
https://www.itb.ec.europa.eu/docs/guides/latest/installingTheTestBed/index.html

生产级：
https://www.itb.ec.europa.eu/docs/guides/latest/installingTheTestBedProduction/index.html


---

# 1. 安装环境

官方要求：
Docker >= 17.06
Docker Compose >= 2.0

推荐：
Windows 11
Docker Desktop

---

# 2. ITB安装步骤

## 2.1 安装目录

新建文件夹 ../ITB（可改名）/testbed/

在testbed文件夹中，
新建
docker-compose.yml
.env
.gitignore

docker-compose.yml: 
四个核心容器gitb-ui gitb-srv MySQL Redis的配置文档，配合.env完成初始密码设置；官方文档给出的docker-compose.yml样例没有完成初始设置，直接使用会在mysql层报错，交付的docker-compose.yml可直接使用。

.env: mysql层的初始密码设置，可通过.gitignore等限制上传。
生产环境请修改为随机密码。

---

## 2.2 启动

第一次：
```
bash
cd ..\ITB\testbed
docker compose up -d
```

查看：
```
docker compose ps
```

正确结果：
itb-mysql   healthy
itb-ui      up
itb-srv     up
itb-redis   up

查看:
```
docker logs -f itb-srv
docker logs -f itb-ui
```

均出现大的
ITB READY为成功，前后端已配好。

关闭服务（保留数据库卷和历史数据）：
```
docker compose down
```

不要在不清楚影响时使用 `docker compose down -v`，因为 `-v` 会同时删除 MySQL 等命名卷中的数据。

若出现容器名称冲突，应先用 `docker ps -a` 找到冲突容器，再针对明确的容器名称执行停止和删除；不要使用没有指定目标的清理命令。

常见错误：

（1）MySQL Restarting

原因：
MYSQL_PASSWORD没有配置。必须配置默认mysql账密才可继续。

报错：
No MySQL application password could be determined

解决：
配置.env中的
MYSQL_ROOT_PASSWORD
MYSQL_USER
MYSQL_PASSWORD
MYSQL_DATABASE

（2）UI UnknownHostException

报哪个gitb-ui gitb-srv gitb-mysql gitb-redis就是哪个服务没起来，本地组件不全，尚未到运行itb这一层。

（3）Container name conflict

例：
Conflict
itb-redis

原因：
之前已有itb-redis同名服务在跑。

---

## 2.3 登录

地址：
http://localhost:9000

默认账号：
admin@itb

首次密码：
查看：
```
docker logs itb-ui
```
日志中会打印：
The one-time password...

首次登录后按照指引修改密码。可重复登陆进去即成功。

---

# 3. Validator 本地部署

官方 RDF Validator 配置指南：
https://www.itb.ec.europa.eu/docs/guides/latest/validatingRDF/

本项目未修改Validator源码，因而使用官方 `isaitb/shacl-validator` Docker 镜像。

本地源码用于源码结构研究，源码需要 Maven 和 `itb-commons` 等依赖编译后才能生成可运行的 `validator.jar`。

当前 ITB Server 已占用主机的 `8080` 端口，因此 Validator 使用主机端口 `8081`，映射到容器内部的 `8080`。

## 3.1 快速启动通用 Validator

启动Docker Desktop后，通用模式适合先确认 Validator 能正常启动。它使用 `any` domain，每次验证时需要同时上传 Data Graph 和 Shapes Graph。

```powershell
docker run -d `
  --name shacl-validator `
  -p 8081:8080 `
  isaitb/shacl-validator:latest
```

查看容器状态：

```powershell
docker ps --filter "name=shacl-validator"
```

查看启动日志：

```powershell
docker logs shacl-validator --tail 100
```

浏览器打开：

http://localhost:8081/shacl/any/upload

在通用页面中上传：

- Data Graph：`data-product-valid.jsonld` 或 `data-product-invalid.jsonld`；
- Shapes Graph：项目唯一正式规则 `building-energy-shapes_D_2.ttl`。

合法样例应返回 `SUCCESS`，非法样例应返回 `FAILURE` 并列出具体违规项。

如果再次执行 `docker run` 时出现容器名称冲突，说明同名容器已经存在。先查看状态：

```powershell
docker ps -a --filter "name=shacl-validator"
```

若容器只是停止了，直接重新启动：

```powershell
docker start shacl-validator
```

## 3.3 配置项目专用 Energy Validator

通用 `any` 模式需要每次手动上传 TTL。最终 Demo 建议建立 `energy` domain，让 Validator 自动加载 `D_2.ttl`。

建议目录结构：

```text
ITB/
├── testbed/
│   └── docker-compose.yml
├── validator-config/
│   └── energy/
│       ├── config.properties
│       └── shapes/
│           └── building-energy-shapes.ttl
├── testsuite/
└── validator/
    └── 官方源码，仅供学习和源码分析
```

从项目根目录执行：

```powershell
New-Item -ItemType Directory -Force ".\ITB\validator-config\energy\shapes"

Copy-Item `
  -LiteralPath ".\building-energy-shapes_D_2.ttl" `
  -Destination ".\ITB\validator-config\energy\shapes\building-energy-shapes.ttl" `
  -Force
```

复制后的 `building-energy-shapes.ttl` 必须与 `D_2.ttl` 内容一致。可以比较 SHA-256：

```powershell
Get-FileHash ".\building-energy-shapes_D_2.ttl"
Get-FileHash ".\ITB\validator-config\energy\shapes\building-energy-shapes.ttl"
```

两个 Hash 应完全相同。

新建 `ITB/validator-config/energy/config.properties`，内容如下：

```properties
# 定义本 Validator 支持的验证类型
validator.type = v1

# 验证类型在网页中的显示名称
validator.typeLabel.v1 = Building Energy Metadata Profile v1.0

# 固定加载 D 组最终 SHACL 规则
validator.shaclFile.v1 = shapes/building-energy-shapes.ttl

# 网页标题
validator.uploadTitle = Building Energy Metadata Validator

# 同时启用网页、REST API 和 SOAP API
validator.channels = form, rest_api, soap_api

# 默认报告格式
validator.defaultReportSyntax = application/ld+json
```

## 3.4 将 Validator 加入 ITB Docker Compose

在 `ITB/testbed/docker-compose.yml` 的 `services:` 下增加第五个服务：

```yaml
  shacl-validator:
    image: isaitb/shacl-validator:latest
    container_name: shacl-validator
    restart: unless-stopped
    ports:
      - "8081:8080"
    environment:
      validator.resourceRoot: /validator/resources/
    volumes:
      - ../validator-config:/validator/resources:ro
```

如果之前通过 `docker run` 创建了独立的 `shacl-validator` 容器，应先停止并删除这个明确的旧容器，避免与 Compose 服务同名：

```powershell
docker stop shacl-validator
docker rm shacl-validator
```

该通用容器没有配置持久化卷，删除容器不会删除项目中的 TTL 和 metadata 文件。

进入 Compose 目录并检查配置：

```powershell
Set-Location "D:\FromC\Working Materials\TIDE_DSSC\DSSC_Tool_Learning\ITB\testbed"
docker compose config --quiet
```

启动 ITB 和 Validator：

```powershell
docker compose up -d
docker compose ps
```

预期看到五个容器：

```text
itb-mysql         healthy
itb-redis         Up
itb-srv           Up
itb-ui            Up
shacl-validator   Up
```

## 3.5 项目专用 Validator 访问地址

浏览器页面：

http://localhost:8081/shacl/energy/upload

REST API：

http://localhost:8081/shacl/energy/api/validate

Swagger API 文档：

http://localhost:8081/shacl/swagger-ui/index.html

SOAP WSDL：

http://localhost:8081/shacl/soap/energy/validation?wsdl

在 `energy` 页面中，`D_2.ttl` 已由配置自动加载，只需要上传待验证的 JSON-LD metadata。

## 3.6 从 ITB Test Case 调用 Validator

ITB 和 Validator 位于同一个 Docker Compose 网络中。ITB Test Case 的 `verify` 步骤应使用 Validator 容器服务名，而不是 `localhost`：

```xml
<interact id="userData" desc="上传待验证的建筑能耗元数据">
    <request
        name="content"
        desc="请选择 JSON-LD 元数据文件"
        inputType="UPLOAD"/>
</interact>

<verify
    handler="http://shacl-validator:8080/shacl/soap/energy/validation?wsdl"
    desc="使用 Building Energy Metadata Profile v1.0 验证元数据">
    <input name="contentToValidate">$userData{content}</input>
    <input name="contentSyntax">"application/ld+json"</input>
    <input name="validationType">"v1"</input>
</verify>
```

原因：在 `itb-srv` 容器内部，`localhost` 指向 `itb-srv` 自己；`shacl-validator` 才是 Compose 网络中的 Validator 服务名。

## 3.7 Validator 部署完成标准

至少满足以下条件，才能认为 Validator 已完成独立本地部署：

1. `shacl-validator` 容器状态为 `Up`；
2. `http://localhost:8081/shacl/energy/upload` 可以打开；
3. SOAP WSDL 地址可以打开；
4. 合法 metadata 返回 `SUCCESS`；
5. 非法 metadata 返回 `FAILURE`；
6. 实际加载的 Shape 与 `D_2.ttl` Hash 相同。

如果 ITB Test Case 还能通过 SOAP 地址得到并保存验证报告，则可以认为 Validator 已完成与 ITB 的本地集成。

## 3.8 常见问题

### Validator 页面无法打开

检查容器和日志：

```powershell
docker ps -a --filter "name=shacl-validator"
docker logs shacl-validator --tail 200
```

### 8080 端口冲突

ITB Server 已使用主机 `8080`，因此 Validator 必须使用：

```text
8081:8080
```

### 找不到 energy domain

检查：

- `validator-config/energy/config.properties` 是否存在；
- `validator.resourceRoot` 是否为 `/validator/resources/`；
- volume 是否为 `../validator-config:/validator/resources:ro`；
- `config.properties` 文件名是否以 `.properties` 结尾。

### 找不到 SHACL 文件

检查配置中的相对路径：

```properties
validator.shaclFile.v1 = shapes/building-energy-shapes.ttl
```

并确认文件位于：

```text
validator-config/energy/shapes/building-energy-shapes.ttl
```

### ITB 能打开，但无法调用 Validator

在 TDL 中使用：

```text
http://shacl-validator:8080/shacl/soap/energy/validation?wsdl
```

不要使用 `http://localhost:8081` 作为容器间调用地址。

## 3.9 停止与重新启动

如果使用 Docker Compose：

```powershell
docker compose stop shacl-validator
docker compose start shacl-validator
```

停止完整环境并保留数据：

```powershell
docker compose down
```

重新启动完整环境：

```powershell
docker compose up -d
```

## 3.10 Validator 源码说明

本地源码可用于学习和二次开发：

```powershell
Set-Location "你的文件位置\ITB"
git clone https://github.com/ISAITB/shacl-validator.git validator
```

源码构建需要 JDK 17+、Maven 3+，并需要先构建匹配版本的 `itb-commons`。成功执行 `mvn clean install` 后，可运行程序位于：

```text
validator/shaclvalidator-war/target/validator.jar
```

如果不修改 Validator 核心代码，无需为了本地 Demo 重新编译源码；官方 Docker 镜像与 D 组自己的 `validator-config`、`D_2.ttl` 配合即可完成部署。
