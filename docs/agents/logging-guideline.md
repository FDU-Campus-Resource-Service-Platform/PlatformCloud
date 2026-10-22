# PlatformCloud 微服务日志治理指南

本文档适用于 `PlatformCloud` 仓库下所有 Java 微服务的 `*-biz` 模块。Python、NodeJS 仓库也应参考“单条日志格式与语法”部分。

仓库统一使用 SLF4J、Logback、OT Java Agent、Loki / Grafana。业务代码默认使用 Lombok `@Slf4j` 注入 `org.slf4j.Logger`。业务审计日志由 `@Log` 注解承担并落库。

## 1. 日志级别语义

| 级别 | 唯一释义 | 判断标准 | 受众 |
| --- | --- | --- | --- |
| `ERROR` | 本次请求或消息无法完成，且原因非业务预期 | 凌晨来处理是否值得，值得就是 `ERROR` | 值班，可触发告警 |
| `WARN` | 业务可继续但状态需关注，例如兜底、降级、重试、数据不自洽、安全鉴权拒绝 | 次日上工是否需要看一眼，需要就是 `WARN` | 工程师次日审视 |
| `INFO` | 有业务意义的状态变更，例如登录、资源创建、状态机推进、定时任务汇总、不可逆副作用 | 只看 `INFO` 是否能讲完业务故事，能就是 `INFO` | 运维和工程师 |
| `DEBUG` | 协助定位问题的中间过程，例如 MQ 明细、缓存命中、循环内单条 | 默认关闭，排障时由 Nacos 临时打开 | 排障 |
| `TRACE` | 比 `DEBUG` 更细的临时细节，例如完整 payload、字段级 diff | 几乎不用 | 临时排障 |

高频路径默认使用 `DEBUG`。如果高频路径必须打 `INFO`，PR 或说明中必须量化估算 QPS。

## 2. 必打位

- HTTP Controller 异常出口由公共异常处理器统一输出，业务代码无需补充。
- `@KafkaListener` 入口打 `INFO`，包含 `topic` 和主业务 ID。
- `@KafkaListener` 异常出口打 `ERROR`，包含 `topic`、主业务 ID 和 `Throwable`。
- `@Scheduled` 入口和出口打 `INFO`，包含任务名、处理条数和耗时。
- 状态机迁移点打 `INFO`，包含实体 ID、`from`、`to`。
- 持久化成功后的业务状态变更或不可逆副作用完成后打 `INFO`。
- 所有非空 catch 块如果降级返回或吞掉异常，必须打 `WARN` 或 `ERROR`，包含 `Throwable` 和业务 ID。

## 3. 禁打位

- Getter、Setter、字段拷贝。
- 循环内逐条 `INFO` 日志；应在循环结束后汇总总数。
- 纯 POJO，例如 Entity、DTO、Request、Response、Message。
- 重复的“我到这里了”型断点替代日志。
- 把 `DEBUG` 当作永久可见日志；要长期可见应使用 `INFO`，否则默认关闭。

## 4. 单条日志格式

日志文本使用 logfmt：

```text
<message phrase>. key1=value1 key2="value with space" key3=value3
```

- message phrase 是稳定、可读的英文短语，句尾统一使用英文句点 `.`。
- key 使用 lowerCamelCase，并与领域字段同名，例如 `documentId`，不要写 `docId` 或 `doc_id`。
- lowerCamelCase 约束结构化字段 key，不要求 message phrase 写成驼峰事件码。
- value 默认不加引号；含空格时用双引号包裹；含双引号时使用 `\"` 转义。
- 分隔符使用单个空格。
- 单行日志不出现 `\n`、`\t`。
- value 为 null 写作 `null`，不写 `NULL`、`<null>` 或空字符串。

## 5. 日志语法

message phrase 使用自然英文短语，形如 `<noun phrase> <past-tense verb>`：

```text
parse task publish requested. documentId=abc123
login succeeded. userId=10086
gc started. task=storageZombie
document status changed. documentId=abc123 from=UPLOADING to=UPLOADED
permission denied. op=deleteResource userId=10086 resourceId=res-1
mail send failed. to=alice@x.edu subject="reset pwd"
```

- noun phrase 使用稳定、可读的领域短语，不把 message phrase 写成自造事件码。
- verb 优先使用下方常用动词；若自然英文短语更清楚，可以使用稳定、行业通用的过去式动词。
- 允许 `<noun phrase> <opVerb> <outcomeVerb>`，例如 `event consumption failed.`，但不要把 message phrase 写成自造事件码。
- message phrase 建议简短，优先不超过 60 个字符。
- `reason` 只用于同一日志族中可聚合的业务决策原因，值使用自然语言并加引号，例如 `reason="status mismatch"`；没有同类日志族或字段本身已经说明原因时，移除 `reason`，把原因写进稳定的 message phrase。

常用动词参考：

| 类别 | 允许动词 |
| --- | --- |
| 实体生命周期 | `created` / `updated` / `deleted` / `restored` |
| 操作结果 | `succeeded` / `failed` / `skipped` / `retried` |
| 任务生命周期 | `started` / `finished` / `aborted` |
| 消息生命周期 | `received` / `publish requested` / `published` / `consumed` / `dropped` / `dispatched` |
| 状态变更 | `changed` / `promotedTo` |
| 增补状态变更 | `renamed` / `moved` |
| 鉴权 | `granted` / `denied` / `stripped` |
| 缓存 | `cached` / `evicted` / `hit` / `missed` |
| 补偿 | `recovered` / `compensated` / `degraded` |
| 校验 | `validated` / `rejected` |
| 清理 | `purged` |

## 6. 必含业务字段

每条业务日志至少包含 1 个领域 ID：

| 领域 | 必含字段 |
| --- | --- |
| user | `userId`，登录前阶段用 `account` |
| group | `groupId` |
| resource | `resourceId` |
| document | `documentId` |
| storage | `objectKey` |
| note | `resourceId`，涉及版本时附 `version` |
| task / consumer | `topic` 和主业务 ID |

批量场景：

- 单实体使用 `xxxId`。
- 批量资源影响使用 `affectedXxxs` 和 `affectedXxxIds`。
- 批量删除事实使用 `count` 和 `xxxIds`。
- ID 列表必须摘要输出，使用 `LogIdUtils.summarizeIds`。

## 7. 占位符与成本

- 一律使用 SLF4J 占位符 `{}`。
- 不使用 `log.<level>(String.format(...))`。
- `Throwable` 作为 logger 调用的最后一个额外参数传入，不写进占位符。
- 参数构造昂贵时，例如序列化大对象，使用 `if (log.isDebugEnabled())` 守卫。

## 8. 异常处理

- catch 后继续抛出：默认不打日志，由最外层处理。
- 降级返回：打 `WARN`，包含 `Throwable`、业务 ID 和降级理由。
- 吞掉异常：打 `WARN` 或 `ERROR`，包含 `Throwable`、业务 ID 和吞掉理由。
- catch 块内调用 logger，必须把 `Throwable` 作为最后一个参数传入。

## 9. 敏感信息

- 严禁入日志：密码、明文 token、sessionId、Cookie、Authorization header、OAuth/STS 凭证、任何可冒充用户身份的字符串。
- email、手机号、学号、真实姓名、`userId`、`groupId` 等业务 ID 视为日志体系内机密，按当前项目约定可直接打印。
- 大对象、实体全文、长文本、二进制禁止打全文。必要时只打 `id + size + 摘要`。

## 10. 跨进程边界模板

### HTTP Controller

HTTP 异常日志由公共异常处理器统一输出。涉及副作用的 Controller 方法使用 `@Log` 做业务审计。

### Feign 出口

不在每次 Feign 调用前后打日志，OT span 已覆盖。Feign 调用失败需要降级时，在 catch 处打一次 `WARN`。

### `@KafkaListener`

```java
@KafkaListener(topics = TOPIC_X, groupId = "...")
public void onX(XMessage message) {
    log.info("x received. topic={} bizId={}", TOPIC_X, message.getBizId());
    try {
        service.process(message);
        log.debug("x consumed. topic={} bizId={}", TOPIC_X, message.getBizId());
    } catch (Exception e) {
        log.error("x consumption failed. topic={} bizId={}", TOPIC_X, message.getBizId(), e);
        throw e;
    }
}
```

### Kafka Producer

业务侧 Producer 默认只记录提交给发布器的意图。真正异步发送成功或失败由 MQ 基础设施层记录。

```java
public void publishX(XMessage message) {
    try {
        kafkaTemplate.send(TOPIC, message.getKey(), message);
        log.debug("x publish requested. topic={} key={}", TOPIC, message.getKey());
    } catch (Exception e) {
        log.error("x publish request failed. topic={} key={}", TOPIC, message.getKey(), e);
    }
}
```

业务侧不要用 `published` 表示尝试发布。只有基础设施层确认异步发送成功时，才使用 `published`。

### `@Scheduled`

```java
@Scheduled(cron = "${...}")
public void doGc() {
    long start = System.currentTimeMillis();
    log.info("gc started. task=storageZombie");
    int processed = 0;
    int failed = 0;
    try {
        for (XEntity entity : repository.findExpired()) {
            try {
                gcOne(entity);
                processed++;
            } catch (Exception e) {
                failed++;
                log.warn("gc one failed. xId={}", entity.getId(), e);
            }
        }
    } catch (Exception e) {
        log.error("gc aborted. task=storageZombie", e);
        return;
    }
    log.info("gc finished. task=storageZombie processed={} failed={} costMs={}",
            processed, failed, System.currentTimeMillis() - start);
}
```

### `@TransactionalEventListener`

事务事件监听器必须自己 catch 顶层异常，避免污染外层事务的 commit-after 链。

```java
@TransactionalEventListener
public void handleXxxEvent(XxxEvent event) {
    log.info("xxx event received. bizId={} trigger={}", event.getBizId(), event.getTrigger());
    try {
        service.doX(event);
    } catch (Exception e) {
        log.error("xxx event handling failed. bizId={}", event.getBizId(), e);
    }
}
```

## 11. 启动、关闭和配置加载

涉及外部资源就绪的代码，例如 `@PostConstruct`、`ApplicationRunner`、`@PreDestroy`，必须用 `INFO` 输出加载结果，并包含可判断字段。

代码内禁止写死 DEBUG 日志级别。临时把业务包提升到 DEBUG 必须通过 Nacos 临时配置和排障 ticket 控制，事后立即复位。

## 12. Review 检查清单

- logger 是否通过 `@Slf4j` 注入。
- catch 块 logger 是否把 `Throwable` 作为最后参数。
- 是否没有 `String.format` 构造日志。
- 每条业务日志是否至少包含 1 个领域 ID。
- message phrase 是否符合 `<noun phrase> <past-tense verb>` 并以 `.` 结束。
- key 是否为 lowerCamelCase。
- 是否没有打印密码、sessionId、token、Authorization。
- `@KafkaListener` 是否入口 `INFO`、完成 `DEBUG`、异常 `ERROR` 并继续抛出。
- `@Scheduled` 是否入口 `INFO`、汇总 `INFO`。
- 是否没有在循环体内打 `INFO`。
- 涉及副作用的 Controller 是否有 `@Log` 或明确白名单理由。
