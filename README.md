# ITB 本地部署指南（测试级）

本文档为基于官方文档与实操经验，在windows系统上本地部署测试级ITB的指南。

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

# 2. 安装步骤

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

关闭服务：
```
docker compose down -v
```

若服务报错，需清理残余：
```
docker rm
```

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


