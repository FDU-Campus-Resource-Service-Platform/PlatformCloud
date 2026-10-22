# PlatformCloud · 校园资源服务平台后端

PlatformCloud 使用 Maven 聚合工程及 `api` / `biz` 组织方式。JDK 21，Maven 3.9+，Java 基础包为 `com.oriole.platform`。

## 模块

| 目录 | 职责 |
| --- | --- |
| `platform-common` | 公共基础模块 |
| `platform-seat-service` | M1 自习座位预约，包含 `platform-seat-api` 和 `platform-seat-biz` |
| `platform-classroom-service` | M2 整教室预约与审批，包含 `platform-classroom-api` 和 `platform-classroom-biz` |
| `platform-maintenance-service` | M3 物业报修工单，包含 `platform-maintenance-api` 和 `platform-maintenance-biz` |

当前为一个 common 加三个业务服务；D 的公共职责由公共模块与各服务后续分工承接，没有独立 system 服务。

各 API 包只建到 `com.oriole.platform.<domain>.api`，不预建 `constant`、`domain`、`enums`、`validation` 等具体子包。Biz 只保留启动类、基础配置与健康检查测试。

## 项目约定

`AGENTS.md` 与 `docs/agents/` 使用 PlatformCloud 项目名称，模块边界和验证命令对应当前一个 common 加三个业务服务；其余编码、日志、错误码和 OpenAPI 约定沿用参考规范。文档中的框架类型与注解须在代码中实际存在后使用，不能视为已经实现。

`docs/questionnaire-structure.md` 是初始化时保留的外部问卷结构参考，不属于本项目的业务实现。

根 POM 沿用参考项目的 Spring Boot 3.2.4、Spring Cloud 2023.0.1、Spring Cloud Alibaba 2023.0.1.0 和相关依赖版本；构建插件显式指定版本，确保 Spring Boot 打包和 JUnit 5 测试可执行。

## 导入与构建

在 IDEA 中打开本目录的 `pom.xml`，作为 Maven 项目导入；Project SDK 与 Maven Runner 选择 JDK 21。

Windows：

```powershell
.\mvnw.cmd --batch-mode --no-transfer-progress clean verify
```

Linux / macOS：

```bash
bash ./mvnw --batch-mode --no-transfer-progress clean verify
```

构建后分别启动：

```text
java -jar platform-seat-service/platform-seat-biz/target/platform-seat-biz-1.0.0.jar
java -jar platform-classroom-service/platform-classroom-biz/target/platform-classroom-biz-1.0.0.jar
java -jar platform-maintenance-service/platform-maintenance-biz/target/platform-maintenance-biz-1.0.0.jar
```

默认端口为 18081、18082、18083，健康端点均为 `/actuator/health`。这些端点只用于验证应用启动，业务、数据库、鉴权与实际跨服务集成尚未实现。

## IDEA 包显示

Project 面板启用 `Compact Middle Packages` / `Hide Empty Middle Packages`，让连续空包折叠为 `com.oriole.platform.seat.api` 这样的单行名称。该功能只改变显示方式，磁盘仍使用标准 Java 包目录。

如果外层目录已作为普通项目打开，请先重新加载 Maven 工程；也可以直接用本目录的 `pom.xml` 打开后端项目。

## Git 与 CI

构建产物、本地配置与缓存被忽略；`AGENTS.md`、`docs/`、Maven Wrapper 和 `.github/` 保持可提交。`.github/workflows/ci.yml` 对提交与 PR 执行全量构建测试，GitHub 分支保护仍需在远端配置。提交代码前由本人验收 AI 改动。
