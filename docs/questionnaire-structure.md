# 问卷版本与答卷结构

> 本文档为初始化时保留的外部问卷结构参考，不属于 PlatformCloud 当前三个业务服务的实现范围。

## 存储边界

- `questionnaire_tables`：保留原有主档结构，`version` 指向当前发布版本，`0` 表示尚未发布。
- `questionnaire_table_versions`：每条记录在根层保存 `resourceId`、`version`、状态和审计时间，并通过 `questionnaireInfo` 子文档保存 `description`、`completionMessage`、`submissionPolicy` 和 `structure`；保留 `resourceId + version` 唯一索引。
- `questionnaire_table_rows`：保存真实填写者、答案和时间，不再保存 `tableVersion` 或独立匿名标记。读取和修改答卷时使用当前最新发布的问卷结构，匿名展示统一由当前发布策略控制。

`structure.pages[].items[].column` 保存完整题目定义，`items[].hidden` 保存展示属性。页面及题目均按数组顺序展示，不再保存 `pageNumber`。`columnId` 在整个问卷中唯一，调整位置时保持不变；答卷的 `values` 仍使用 `columnId` 作为键。

隐藏题目仍属于问卷结构；本次重构不改变其答案校验和保留规则。只有当前结构中已不存在的题目，才按原有逻辑在答卷更新时清理旧答案。

Java 模型参考 note 的组织方式：`QuestionnairePageDefinition` 与 `QuestionnaireColumnItem` 作为 `QuestionnaireStructure` 的静态内部类；独立使用的 `QuestionnaireSubmissionPolicy` 和多态题型继续保留独立文件。页面和题目内部结构保持不变，本轮仅调整版本文档及组合响应中问卷内容的归属层级。

共用问卷内容放在 API 模块的 `domain/base` 中，持久化实体和草稿更新请求均通过组合使用：

- `QuestionnaireDefinitionBase` 只声明说明、完成提示、提交策略和页面题目结构，并保留相应参数校验。它不包含资源身份、版本状态或数据库元数据。
- `TableVersionEntity` 在根层声明数据库 ID、`resourceId`、`version`、`status` 和审计时间，并以 `questionnaireInfo` 字段组合 `QuestionnaireDefinitionBase`。草稿保存、发布及读取的状态校验仍使用实体字段，`resourceId + version` 查询和唯一索引保持根层结构。
- 草稿更新请求在根层声明必填的 `resourceId` 和 `version`，并以必填的 `questionnaireInfo` 组合 Base，保持身份、版本和问卷内容边界一致。
- 答卷实体、提交请求、更新请求、详情响应分别声明 `values`，不为单个答案字段额外抽取 Base。用户归属、数据库 ID 及审计时间仍由实体声明。
- Base 继续使用 `@SuperBuilder` 供问卷内容自身构建；通过组合保存或传递内容的实体、请求及独立数据类使用 `@Builder`。摘要响应按实际返回字段独立定义；`TableEntity` 保持原结构。

Base 整理将 Mongo 版本文档中的四个问卷内容字段收拢到 `questionnaireInfo` 子文档，`resourceId` 和 `version` 仍位于根层；对外响应同样将内容放在 `questionnaireInfo`，版本号单独放在响应根层。

参照 note 的详情查询，`getQuestionnaireDraft`、`getQuestionnaire` 和 `getQuestionnaireForFilling` 的 Service 方法直接返回 `TableVersionEntity`，Controller 直接取实体中的 `questionnaireInfo` 组装命名的资源组合响应变量，同时单独返回 `version`，不暴露实体的数据库 ID、状态和审计时间。填写视图仍直接通过 Repository 读取主档和当前发布版本，并保留版本状态、填写起止时间和个人提交次数校验。

提交和更新答卷在 Service 内使用版本实体中的 `questionnaireInfo`。答卷详情属于加工结果，仍在 Service 中完成答案映射和响应组装，并直接返回嵌入实体的 Base；版本号单独返回，实体元数据不会进入响应。

## 草稿更新契约

`POST /questionnaire/updateQuestionnaireDraft` 全量替换当前草稿内容。`questionnaireInfo` 必传，其中 `structure`、`submissionPolicy` 必传；`description`、`completionMessage` 未传时清空。草稿允许空页面列表、空题目列表；嵌套对象、列表元素和完整题目定义接受参数校验。

```json
{
  "resourceId": "questionnaire-resource-id",
  "version": 2,
  "questionnaireInfo": {
    "description": "员工调查",
    "completionMessage": "感谢参与",
    "submissionPolicy": {
      "anonymous": true,
      "maxSubmissionsPerUser": 1,
      "draftAllowed": true,
      "editableAfterSubmit": true,
      "startTime": null,
      "endTime": null
    },
    "structure": {
      "pages": [
        {
          "title": "基本信息",
          "description": "请填写以下内容",
          "items": [
            {
              "hidden": false,
              "column": {
                "type": "NUMBER",
                "columnId": "age",
                "name": "年龄",
                "required": true,
                "min": 18,
                "max": 100,
                "scale": 0
              }
            }
          ]
        }
      ]
    }
  }
}
```

`getQuestionnaireDraft`、`getQuestionnaire` 和 `getQuestionnaireView` 统一返回 `QuestionnaireResourceInfoResponse`，包含 `resourceInfo`、`version` 和 `questionnaireInfo`。`resourceInfo` 使用资源服务的 `ResourceItemResponse`；`version` 表示实际返回的问卷版本；`questionnaireInfo` 使用 `QuestionnaireDefinitionBase`，包含 `description`、`completionMessage`、`submissionPolicy` 和 `structure`。资源 ID 和名称统一读取 `resourceInfo`，问卷定义不再保存 `title` 或返回 `status`。

`getSubmission` 返回 `QuestionnaireSubmissionDetailResponse`，同样以 `resourceInfo`、`version` 和 `questionnaireInfo` 表示资源信息、映射答案所用版本和最新发布的问卷定义，另有 `submissionId`、`values`、`submitTime`、`updateTime`。匿名状态统一读取 `questionnaireInfo.submissionPolicy.anonymous`。

资源信息由 Controller 复用本次鉴权时的资源服务查询结果组装，不写入问卷数据库。Service 内部继续使用独立的问卷定义，不依赖请求用户的资源展示信息。此组织方式参考 AI-asset 的资源组合响应和 note 的 Controller 组装方式。

前端读取路径同步调整：问卷内容读取 `data.questionnaireInfo.structure`、`data.questionnaireInfo.submissionPolicy` 等；草稿更新提交根层 `resourceId`、`version` 以及完整的 `questionnaireInfo`；答卷详情的问卷内容读取 `data.questionnaireInfo`；资源 ID 和名称统一读取 `data.resourceInfo`。最新资源信息、问卷版本和问卷内容分别表达各自含义。

提交及局部更新答卷的请求使用 `values`，不再接收填写者的 `anonymous` 选择；不需要把页面结构放进答卷请求。`listMySubmissions` 和删除答卷接口的契约保持原样。`listAllSubmissions` 只调整匿名身份展示，表格查询的筛选与排序仍不属于此次重构。

## 匿名策略

`submissionPolicy.anonymous` 为必填布尔值，由发起者设置，创建问卷时默认 `false`。策略发布后，`true` 表示全部答卷匿名展示，`false` 表示全部答卷实名展示；同一问卷按当前发布策略统一处理，包括已有答卷，不再逐份读取匿名标记。

匿名是身份展示规则。后端仍保存真实 `userId`，用于“我的答卷”、修改归属校验和提交次数限制。发起者分页查询时，匿名问卷的每条答卷响应均省略 `userId`，非匿名问卷正常返回；尚未发布的问卷仍支持查询空答卷列表。

填写者无法在提交或更新请求中决定匿名与否；原匿名选择校验和 `SUBMISSION_ANONYMOUS_NOT_ALLOWED` 错误码已移除。前端需将策略字段 `anonymousAllowed` 改为 `anonymous`，移除填写表单中的匿名选择控件。

## 版本生命周期

创建问卷时初始化完整的版本 1 草稿。编辑时检查跨页面题目 ID 唯一、题型配置和默认答案，并校验提交策略组合；策略非法返回 `QuestionnaireError.SUBMISSION_POLICY_INVALID`（数字码保留为 `10321`）。

发布时将当前草稿设为已发布，并将四个内容字段一起复制至下一版草稿。复制问卷时将指定发布版本的完整内容复制为新问卷的首个发布版本及下一版草稿。删除问卷时删除主档、全部版本及答卷。

## 旧数据迁移边界

新代码不再读写 `questionnaire_views`，也不兼容旧的独立 `columns` 与视图结构。现有数据库若要保留使用，部署前需要迁移；仅替换 Java 类不会自动转换旧文档。

迁移时按版本记录的 `resourceId + version` 查找旧视图的 `resourceId + tableVersion`，将说明、完成提示、提交策略和页面题目结构写入版本记录的 `questionnaireInfo` 子文档；按旧页面数组顺序遍历题目引用，从该版本的 `columns` 找到完整定义，写入 `questionnaireInfo.structure.pages[].items[].column`，保留 `hidden`，去掉页码。

旧结构允许存在未被页面引用的题目。迁移前必须检查缺失视图、悬空引用、重复 ID 以及未被引用的题目，不能静默丢弃或自行安排这些题目。旧 Mongo 文档的多态类型标记也需要适配新模型，不能原样复制已删除 Java 类型的 `_class`。

完成备份、迁移和读取验证后，再另行处理旧 `columns`、答卷 `tableVersion` 字段及旧视图集合。本次代码修改不执行数据库迁移或删除实际集合。

旧 `anonymousAllowed` 表达“允许填写者选择匿名”，新 `anonymous` 表达“全部答卷统一匿名”，迁移时需要明确每份问卷的新策略，不能直接把旧值当作等价设置。旧答卷上的 `anonymous` 字段不再参与读取或更新，确认迁移后可另行清理。
