# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码仓库中工作时提供指导。

## 项目概述

芋道云（Yudao-Cloud）是基于 Spring Cloud Alibaba 的微服务快速开发平台，提供完整的后台管理系统，支持 SaaS 多租户。

**技术栈：**
- Java 8（master 分支）/ Java 17/21（master-jdk17 分支）
- Spring Boot 2.7.18 / Spring Cloud 2021.0.9 / Spring Cloud Alibaba 2021.0.6.2
- Nacos（注册中心 + 配置中心）、Spring Cloud Gateway、Sentinel、Seata
- MyBatis Plus、Druid、Redis/Redisson
- Flowable（工作流）、XXL-Job（定时任务）

## 构建命令

```bash
# 构建整个项目
mvn clean install -DskipTests

# 构建指定模块
mvn clean install -DskipTests -pl yudao-module-system/yudao-module-system-server -am

# 运行模块测试
mvn test -pl yudao-module-system/yudao-module-system-server

# 运行单个测试类
mvn test -pl yudao-module-system/yudao-module-system-server -Dtest=UserServiceImplTest

# 运行单个测试方法
mvn test -pl yudao-module-system/yudao-module-system-server -Dtest=UserServiceImplTest#testGetUser
```

## 架构说明

### 模块结构

每个业务模块采用双子模块模式：
- `yudao-module-xxx-api`：对外暴露的 API 接口、DTO、枚举，供其他服务通过 Feign 调用
- `yudao-module-xxx-server`：具体实现，包含 controller/service/dal 三层

**核心模块：**
- `yudao-dependencies`：Maven BOM，统一管理依赖版本
- `yudao-framework`：可复用的 Spring Boot Starter（security、mybatis、redis、rpc 等）
- `yudao-gateway`：Spring Cloud Gateway 网关（端口 48080）
- `yudao-server`：单体部署方式（合并所有模块）

**业务模块：**
- `yudao-module-system`：用户、角色、权限、租户、字典、短信、邮件
- `yudao-module-infra`：代码生成、文件存储、API 日志、配置管理
- `yudao-module-bpm`：Flowable 工作流引擎
- `yudao-module-pay`：支付集成（支付宝、微信）
- `yudao-module-member`：会员管理
- `yudao-module-mall`：商城系统（商品、交易、营销、统计）
- `yudao-module-erp`：ERP 系统
- `yudao-module-crm`：CRM 系统
- `yudao-module-mp`：微信公众号
- `yudao-module-report`：积木报表集成
- `yudao-module-ai`：AI 大模型集成（需要 JDK17）
- `yudao-module-iot`：物联网设备管理（MQTT、CoAP、HTTP、TCP、UDP、WebSocket）

### 包结构（server 模块内部）

```
cn.iocoder.yudao.module.{module}
├── api/          # Feign API 实现
├── controller/   # REST 控制器（admin-api、app-api）
├── convert/      # MapStruct 转换器
├── dal/          # 数据访问层
│   ├── dataobject/  # 实体类（DO 后缀）
│   ├── mysql/       # MyBatis Mapper
│   └── redis/       # Redis DAO
├── framework/    # 模块特定配置
├── job/          # XXL-Job 任务处理器
├── mq/           # 消息消费者/生产者
└── service/      # 业务逻辑层
```

### API 路径规范

- 管理后台 API：`/admin-api/{module}/**`
- 用户端 API：`/app-api/{module}/**`

### 服务端口

- Gateway 网关：48080
- system-server：48081
- infra-server：48082
- 其他模块：48xxx

## 单元测试

测试使用 H2 内存数据库（MySQL 兼容模式）。基础测试类：
- `BaseDbUnitTest`：需要数据库的测试
- `BaseDbAndRedisUnitTest`：需要数据库 + Redis 的测试
- `BaseMockitoUnitTest`：纯 Mock 单元测试
- `BaseRedisUnitTest`：仅需要 Redis 的测试

测试配置文件：`src/test/resources/application-unit-test.yaml`

测试类编写规范：
1. 继承合适的基础测试类
2. 使用 `@Import` 加载所需配置
3. SQL 建表脚本放在 `src/test/resources/sql/create_tables.sql`

## 开发规范

### 实体类
- 后缀：`DO`（Data Object）
- 继承 `BaseDO` 获取审计字段（createTime、updateTime、creator、updater、deleted）
- 使用 `@TableName` 注解映射表名

### DTO 类
- 请求：`XxxReqVO`、`XxxPageReqVO`
- 响应：`XxxRespVO`
- 使用 `@Schema` 注解生成 Swagger 文档

### 转换器
- 使用 MapStruct，添加 `@Mapper` 注解
- 类名：`XxxConvert`
- 通过 `INSTANCE` 字段实现单例模式

### 多租户
- 租户隔离通过 `TenantDatabaseInterceptor` 自动实现
- 使用 `@TenantIgnore` 注解跳过租户过滤
- 在 `yudao.tenant.ignore-tables` 配置忽略的表

### 配置文件
- 本地开发：`application-local.yaml`
- 开发环境：`application-dev.yaml`
- Nacos 配置：`{service-name}-{profile}.yaml`

## 数据库

SQL 脚本位置：`sql/mysql/ruoyi-vue-pro.sql`

支持的数据库：MySQL、PostgreSQL、Oracle、SQL Server、达梦 DM、人大金仓 KingBase、OpenGauss、TiDB
