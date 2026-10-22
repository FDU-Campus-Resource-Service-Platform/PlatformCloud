# PlatformCloud 错误码治理指南

本文档约束业务错误码、异常抛出和 OpenAPI 失败项。目标是让错误码可搜索、可审计、可复用，不出现临时字符串错误或编造枚举名。

## 1. 现有体系

- 所有错误码枚举实现 `IResult`。
- `IResult` 包含 `code`、`ResultKey` 和 `msg`。
- `ResultKey` 由 `BusinessDomain`、业务 Subject 和 `ErrorReason` 组成。
- 公共框架错误放 `CommonError`。
- 权限认证错误放 `PermissionError`，并通过 `PermissionException` 抛出。
- 业务服务错误放对应服务的 `*Error` 枚举，例如 `UserError`、`ResourceError`、`DocumentError`、`NoteError`、`SkillError`、`FileStorageError`。

## 2. 新增错误码

- 新增业务失败前，先搜索现有 `CommonError`、`PermissionError` 和当前服务 `*Error`，优先复用已有语义。
- 确实需要新增时，必须先落到真实枚举，再在业务代码和 OpenAPI 文案中引用。
- 不允许在业务代码中硬编码裸数字错误码。
- 不允许返回临时字符串错误。
- 不允许在 OpenAPI `失败` 字段中编造不存在的 `ErrorEnum.CONSTANT`。
- 错误枚举名使用大写下划线，表达业务失败条件，例如 `RESOURCE_NOT_FOUND`、`DOCUMENT_PREVIEW_NOT_READY`。

## 3. 错误归属

- 跨所有服务通用的框架、请求、参数、安全上下文错误放 `CommonError`。
- 登录态、身份、角色、权限切面失败放 `PermissionError`。
- 具体业务对象不存在、状态不允许、业务冲突、外部服务失败，放当前服务的 `*Error`。
- 如果失败由被调用服务产生，优先透传或转换为当前业务语义清晰的错误，不复制对方内部错误枚举。
- 新增 `BusinessDomain` 或 Subject 前必须确认不是已有业务域可以表达。

## 4. 异常抛出

- 业务失败使用 `new ServiceException(SomeError.SOME_CONSTANT)`。
- 需要补充上下文时使用 `ServiceException(IResult, String)`，补充信息应简短且不能包含敏感信息。
- 权限认证失败使用 `PermissionException` / `PermissionError`。
- 不用 `RuntimeException`、`IllegalStateException`、`IllegalArgumentException` 直接表达业务失败，除非该异常只在内部被立即捕获并转换。
- 不吞掉业务异常；如果降级返回，必须按日志治理说明记录降级原因。

## 5. OpenAPI 失败项

- `@Operation.description` 的 `- 失败：` 只写当前接口会触发且联调方容易踩到的失败点。
- 每个失败点使用 `触发条件 -> ErrorEnum.CONSTANT`。
- `ErrorEnum.CONSTANT` 必须真实存在。
- 参数校验错误默认不写，除非接口显式把参数转换为业务错误。
- 未捕获异常只在确有明确兜底风险时写 `触发条件 -> CommonError.INTERNAL_ERROR`。

## 6. Review 检查清单

- 新错误是否先搜索并确认无法复用现有枚举。
- 业务失败是否通过 `ServiceException` 携带真实 `IResult`。
- 权限失败是否使用权限异常体系。
- 是否没有裸数字错误码、临时字符串错误或编造枚举。
- OpenAPI `失败` 字段引用的枚举是否真实存在。
