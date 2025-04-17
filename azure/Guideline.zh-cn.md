# 微软 Azure REST API 手册

<!-- cspell:ignore autorest, BYOS, etag, idempotency, maxpagesize, innererror, trippable, nextlink, condreq, etags -->
<!-- markdownlint-disable MD033 MD049 MD055 -->

<!--
Note to contributors: All guidelines now have an anchor tag to allow cross-referencing from associated tooling.
The anchor tags within a section using a common prefix to ensure uniqueness with anchor tags in other sections.
Please ensure that you add an anchor tag to any new guidelines that you add and maintain the naming convention.
-->

## 历史

<details>
  <summary>展开变更历史</summary>

| 时间 | 记录 |
| --- | --- |
| 2025-03-28 | 添加了有关 JSON ID 和空值的指南 |
| 2024-03-17 | 更新的 LRO 指南 |
| 2024-01-17 | 添加了有关返回字符串偏移量和长度的指南 |
| 2023-03-12 | 解释缺失/不支持的服务响应 `api-version` |
| 2023-04-21 | 更新/澄清 POST 方法可重复性的指导原则 |
| 2023-04-07 | 更新/澄清多态性指南 |
| 2022-09-07 | 更新了 DNS Done Right 的 URL 指南 |
| 2022-07-15 | 更新有关长期运行操作的指导 |
| 2022-05-11 | 删除有关版本发现的指导  |
| 2022-03-29 | 添加有关使用持续时间的指南 |
| 2022-03-25 | 更新头中日期值的指南以遵循 RFC 7231 |
| 2022-02-01 | 更新了错误指南 |
| 2021-09-11 | 添加长期运行操作指导 |
| 2021-08-06 | 根据 Azure API 管理委员会更新了 Azure REST 指南 |
| 2020-07-31 | 增加了初始版本的服务建议 |
| 2020-03-31 | Azure REST API指南的首次公开发布 |

</details>

## 介绍

这些准则适用于 Azure 服务团队。它们提供了 Azure 服务团队必须遵循的规范性指导，以确保通过设计满足以下目标的 API 来为客户提供良好的体验:
✅  通过一致的模式和 Web 标准（HTTP、REST、JSON）方便开发人员

✅  高效且经济

✅  与多种编程语言的 SDK 完美兼容

✅  客户可以通过支持重试/幂等性/乐观并发来创建容错应用程序

✅  通过明确的 API 契约实现可持续性和版本可控性，但需满足以下两个要求:
  1. 客户工作负载绝不能因服务变更而中断
  2. 客户无需更改代码即可采用版本


技术和软件瞬息万变，因此，本文档旨在保持动态更新. 您可以[提交问题](https://github.com/microsoft/api-guidelines/issues/new/choose) 来提出修改建议或新想法. 请阅读 [服务设计注意事项](./ConsiderationsForServiceDesign.md) 了解有关 Azure 服务 API 设计主题的介绍。对于已正式发布的服务，请勿更改/破坏其现有 API；相反，应将这些概念应用于未来的 API，同时优先考虑现有服务的一致性.

*注意：如果您正在创建管理平面（ARM）API，请参阅 [Azure 资源管理器资源提供者合约](https://github.com/cloud-and-ai-microsoft/resource-provider-contract).*

## 构建模块：HTTP、REST 和 JSON
Microsoft Azure 云平台通过 Internet 的核心构建块公开其 API；即 HTTP、REST 和 JSON。本节将帮助您大致了解在创建服务时应如何应用这些技术.

<a href="#http" name="http"></a>
### HTTP
服务必须遵守 HTTP 规范, [RFC 7231](https://tools.ietf.org/html/rfc7231). 本节进一步完善和限制了服务实施者应如何应用 HTTP 规范中定义的构造。因此，您必须牢牢理解以下概念:

✅  [统一资源定位符 (URL)](#uniform-resource-locators-urls)

✅  [HTTP 请求/响应模式](#http-request--response-pattern)

✅  [HTTP 查询参数和标头值](#http-query-parameters-and-header-values)

#### 统一资源定位符 (URL)

统一资源定位符 (URL) 是开发者访问服务资源的方式。归根结底，URL 是开发者对服务资源形成认知模型的方式. 请使用此 URL 模式：
```text
https://<host>/<service>/<resource-collection>/<resource-id>
```

说明:
 | 字段 | 描述
 | - | - |
 | host | 服务域名
 | service | 服务名称（例如：blobstore、servicebus、目录或管理）
 | resource&#x2011;collection | 集合名称（未缩写，复数形式）
 | resource&#x2011;id | 资源集合中的资源 ID。该 ID 必须是原始字符串/数字/GUID 值，不带引号，但经过正确转义以适合 URL 片段。


✅  务必对 URL 路径段使用短横线命名法（推荐）或驼峰命名法。如果路径段引用 JSON 字段，请使用驼峰命名法

✅  如果 URL 超过 2083 个字符，则返回414-URI Too Long

✅  务必将服务定义的 URL 路径段视为区分大小写。如果传入的大小写与服务预期不符，则请求必须失败并404-Not found返回 HTTP 返回代码。

如果某些客户提供的路径段值所代表的抽象通常不区分大小写，则这些路径段值可能会进行不区分大小写的比较。例如，UUID 路径段“c55f6b35-05f6-42da-8321-2af5099bd2a2”应被视为与“C55F6B35-05F6-42DA-8321-2AF5099BD2A2”相同

✅  在 HTTP 响应标头值或 JSON 响应正文中返回 URL 时，请确保大小写正确

✅  将服务定义的路径段中的字符限制为0-9  A-Z  a-z  -  .  _  ~，并且:仅允许如下所述指定动作操作。

✅  您应该将用户指定的路径段（即路径参数值）中允许的字符限制为0-9  A-Z  a-z  - .  _  ~（不允许:）。

✅  您应该保持 URL 可读；如果可能，请避免使用 UUID 和 % 编码（例如：Cádiz 的 % 编码为 C%C3%A1diz）

✅  您可以在 URL 路径中使用这些其他字符，但它们可能需要 % 编码 [ RFC 3986 ]：/  ?  #  [  ]  @  !  $  &  '  (  )  *  +  ,  ;  =

✅  您可以使用 URL 作为值
```text
https://api.contoso.com/items?url=https://resources.contoso.com/shoes/fancy
```

#### HTTP 请求/响应模式
HTTP 请求/响应模式决定了 API 的行为方式。例如：创建资源的 POST 方法必须是幂等的，GET 方法的结果可以缓存，If-Modified 和 ETag 标头提供乐观并发。服务的 URL 及其请求/响应主体确立了开发者与服务之间的总体契约。作为服务提供商，如何管理整体请求/响应模式应该是您首先要做出的实施决策之一。

云应用拥抱故障。因此，为了让客户能够编写容错应用程序，所有服务操作（包括 POST）都必须是幂等的。以幂等的方式实现服务，并遵循“恰好一次”的语义，使开发人员能够重试请求，而不会产生意外后果的风险。

##### 恰好一次行为 = 客户端重试 & 服务幂等性

✅  一定要确保所有HTTP 方法都是幂等的。

✅  您应该使用 PUT 或 PATCH 来创建资源，因为这些 HTTP 方法易于实现，允许客户命名自己的资源，并且是幂等的。

✅  您可以使用 POST 请求创建资源，但必须确保其幂等性，并且响应必须返回已创建资源的 URL 以及 201-Created 状态码。确保 POST 请求幂等性的一种方法是使用 Repeatability-Request-ID 和 Repeatability-First-Sent 标头（请参阅请求的可重复性）。

##### HTTP 返回代码

当方法同步完成并成功时，请遵守下表中的返回代码

方法 | 描述 | 状态码
-------|-------------|---------------------
PATCH  | 使用 JSON Merge Patch 创建/修改资源 | `200-OK`, `201-Created`
PUT    | 创建/替换整个资源 | `200-OK`, `201-Created`
POST   | 创建新资源（由服务设置的ID） | `201-Created` 使用所创建资源的 URL
POST   | 行为 | `200-OK`
GET    | 读取（即列出）资源集合 | `200-OK`
GET    | 读取资源 | `200-OK`
DELETE | 删除资源 | `204-No Content`\; 避免使用 `404-Not Found`

✅ 当 PUT、POST 或 DELETE 方法异步完成时，请返回状态代码并遵循 [长时间运行的操作和作业](#long-running-operations--jobs) 202-Accepted中的指导。

✅ 务必将方法名称视为区分大小写，并且应始终使用大写

✅ 在 PUT、PATCH、POST 或 GET 操作之后， 请200-OK使用或返回资源的状态201-Created。

✅ 对于 DELETE 操作， 请返回204-No Content不带资源/主体的 （即使 URL 标识的资源不存在；也不要返回404-Not Found）

✅ 务必从 POST Action返回200-OK。即使响应中没有属性，也应包含正文，以便将来需要时添加属性。

✅ 当用户无权访问资源时，请返回 403-Forbidden，除非处于安全或隐私考虑不希望泄露有关资源存在的信息 在这种情况下，应该响应 404-Not Found。[理由：通常情况下响应403-Forbidden更容易调试， 但如果有特殊安全或隐私要求，响应403-Forbidden即承认资源存在，承认资源存在也存在泄漏客户机密的的可能，有特殊要求时不应使用。]

✅ 通过遵守、if-modified-since 和 if-unmodified-since 请求标头并返回 ETag 和 last-modified 响应标头来支持缓存和乐观并发If-MatchIf-None-Match

#### HTTP 查询参数和标头值

✅ 请使用驼峰式命名法来命名查询参数名称。

注意：某些旧式查询参数名称使用 kebab-casing，仅允许向后兼容。

由于服务 URL 以及请求/响应中的信息都是字符串，因此必须有一个可预测的、定义明确的方案将字符串转换为其对应的值。

✅ 务必验证所有查询参数和请求标头的值，如果任何值未通过验证，则操作失败 400-Bad Request。返回错误响应，如“处理错误”部分所述，指出错误所在，以便客户诊断问题并自行修复。

✅ 翻译字符串时 请使用下表：

数据类型 | 记录该字符串必须是
--------- | -------
Boolean   | true / false (全部写)
Integer   | -2<sup>53</sup>+1 to +2<sup>53</sup>-1 (为了与 JSON 对整数的限制[RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259)) 保持一致
Float     | [IEEE-754 二进制64](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)
String    | (Un)quoted?, 最大长度，合法字符，区分大小写，多个分隔符
UUID      | 123e4567-e89b-12d3-a456-426614174000 (无 {}、连字符、不区分大小写) [RFC 4122](https://datatracker.ietf.org/doc/html/rfc4122)
Date/Time (Header) | Sun, 06 Nov 1994 08:49:37 GMT [RFC 7231, Section 7.1.1.1](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.1.1)
Date/Time (Query parameter) | YYYY-MM-DDTHH:mm:ss.sssZ (最多 3 位秒的小数) [RFC 3339](https://datatracker.ietf.org/doc/html/rfc3339)
Byte array | Base64 编码，最大长度
Array      | 以逗号分隔的值列表（首选），或 name=value数组中每个值的单独参数实例


下表列出了服务最常用的标头

头          | 适用于 | 举例
------------------- | ---------- | -------------
_authorization_     | Request    | Bearer eyJ0...Xd6j（支持 Azure Active Directory）
_x-ms-useragent_    | Request    | (参见 [分布式跟踪和遥测](#distributed-tracing--telemetry))
traceparent         | Request    | (参见 [分布式跟踪和遥测](#distributed-tracing--telemetry))
tracecontext        | Request    | (参见 [分布式跟踪和遥测](#distributed-tracing--telemetry))
accept              | Request    | application/json
If-Match            | Request    | "67ab43" 或 * (无引号) (参见 [条件请求](#conditional-requests))
If-None-Match       | Request    | "67ab43" 或 * (无引号) (参见 [条件请求](#conditional-requests))
If-Modified-Since   | Request    | Sun, 06 Nov 1994 08:49:37 GMT (参见 [条件请求](#conditional-requests))
If-Unmodified-Since | Request    | Sun, 06 Nov 1994 08:49:37 GMT (参见 [条件请求](#conditional-requests))
date                | Both       | Sun, 06 Nov 1994 08:49:37 GMT (参见 [RFC 7231, 第 7.1.1.2 节](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.1.2))
_content-type_      | Both       | application/merge-patch+json
_content-length_    | Both       | 1024
_x-ms-request-id_   | Response   | 4227cdc5-9f48-4e84-921a-10967cb785a0
ETag                | Response   | "67ab43" (参加 [条件请求](#conditional-requests))
last-modified       | Response   | Sun, 06 Nov 1994 08:49:37 GMT
_x-ms-error-code_   | Response   | (参见 [错误处理](#handling-errors))
_azure-deprecating_ | Response   | (参见 [弃用行为通知](#deprecating-behavior-notification))
retry-after         | Response   | 180 (参见 [RFC 7231, 第 7.1.3 节](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.3))

✅ 支持所有斜体标题

✅ 使用 kebab-casing 指定headers

✅ 比较请求头名称时不区分大小写

✅如果请求头名称要求， 请区分大小写地比较请求头的值

✅ 接受HTTP-Date 格式的标头中的日期值，并返回 IMF-fixdate 格式的标头中的日期值，如RFC 7231 第 7.1.1.1 节所定义，例如“Sun, 06 Nov 1994 08:49:37 GMT”。

注意：RFC 7231 IMF-fixdate 格式是 RFC 1123 / RFC 5822 格式的“固定长度和单区域子集”，这意味着：a) 年份必须是四位数字，b) 时间的秒数部分是必需的，c) 时区必须是 GMT。

✅ 请创建一个唯一标识请求的不透明值，并在x-ms-request-id响应标头中返回该值。

您的服务应该x-ms-request-id在错误日志中包含该值，以便用户可以使用该值提交针对特定故障的支持请求。

⛔ 请勿拒绝包含无法识别标头的请求。标头可能由 API 网关或中间件添加，因此必须容忍这种情况。

⛔ 不要对自定义标头使用“x-”前缀，除非该标头已存在于生产中 [ RFC 6648 ]。

**其他参考**
- [StackOverflow - http 参数和 http 标头之间的区别](https://stackoverflow.com/questions/40492782)
- [标准 HTTP 标头](https://httpwg.org/specs/rfc7231.html#header.field.registration)
- [为什么不允许 HTTP PUT 在 REST API 中进行部分更新？](https://stackoverflow.com/questions/19732423/why-isnt-http-put-allowed-to-do-partial-updates-in-a-rest-api)

<a href="#rest" name="rest"></a>
### 表述性状态转移 (REST)
REST 是一种具有广泛影响力的架构风格，强调可扩展性、通用性、独立部署、通过缓存减少延迟和安全性。将 REST 应用于 API 时，可以将服务的资源定义为项目集合。这些通常是服务词汇表中的名词。服务的 [URLs](#uniform-resource-locators-urls)决定了开发人员用于对资源执行 CRUD（创建、读取、更新和删除）操作的层次路径。请注意，重要的是对资源状态进行建模，而不是对行为进行建模。本指南后面的模式描述了如何在服务上调用行为。有关 REST API 设计模式的更详细讨论，请参阅[Azure 架构中心](https://docs.microsoft.com/azure/architecture/best-practices/api-design) 的这篇文章。

在设计服务时，针对使用 API 的开发人员进行优化非常重要。

✅ 一定要高度重视清晰一致的命名

✅ 确保你的资源路径合理

✅ 务必使用少量必需的查询参数和 JSON 字段来简化操作

✅ 为字符串值建立清晰的契约

✅ 请使用正确的响应代码/主体，以便客户可以诊断自己的问题并修复它们，而无需联系 Azure 支持或服务团队

资源模式和字段可变性
✅ 务必对给定 URL 路径上的 PUT 请求/响应、PATCH 响应、GET 响应和 POST 请求/响应使用相同的 JSON 架构。PATCH 请求架构应包含所有相同的字段，但不含必填字段。这样可以允许使用一种 SDK 类型进行输入/输出操作，并允许将响应在请求中传回。

✅ 请考虑一下你的资源的字段以及它们的使用方式：

字段可变性 | 服务请求针对此字段的行为
-----------------| -----------------------------------------
**Create** | 服务仅在创建资源时才会启用该字段。请尽量减少仅创建字段，以便客户无需删除并重新创建资源
**Update** | 创建或更新资源时服务荣誉字段
**Read**   | 服务在响应中返回此字段。如果客户端传递了只读字段，则服务必须拒绝该请求，除非传入的值与资源的当前值匹配

除上述情况外，字段可能是“必需”或“可选”的。必需字段保证始终存在，并且通常不会成为SDK 数据结构中的可空字段。这允许客户编写代码而无需执行空值检查。因此，必需字段只能在服务的第一个版本中引入；在更高版本中引入必需字段是一项重大更改。此外，删除必需字段或将可选字段设为必需或反之亦然是一项重大更改。

✅ 务必使字段简单并保持浅层次结构。

✅ 使用GET 进行资源检索并在响应主体中返回 JSON

✅ 使用 PATCH [RFC 5789] 和 JSON Merge Patch [(RFC 7396)](https://datatracker.ietf.org/doc/html/rfc7396) 请求正文创建和更新资源。

✅ 务必使用 JSON 格式的 PUT 语句进行批量创建/替换操作。注意：如果 v1 客户端对资源执行了 PUT 操作，则 V2+ 版本中引入的任何字段都应重置为其默认值（相当于先执行 DELETE 操作，再执行 PUT 操作）。

✅ 使用DELETE 来删除资源。

✅ 如果请求格式不正确，或者特定版本的服务无法完全理解任何 JSON 字段名称或值，则务必让操作失败。返回错误响应，并按照[处理错误](#handling-errors) 中的说明指出错误所在，以便客户能够诊断问题并自行修复。400-Bad Request

✔️ 如果绝对必要，您可以通过 POST 返回秘密字段。

⛔ 请勿通过 GET 方式返回机密字段。例如，请勿administratorPassword以 JSON 格式返回。

⛔ 如果值可以轻松从其他字段计算出来，请不要向 JSON 添加字段，以免导致主体膨胀。

##### Create / Update / Replace Processing Rules

<a href="#rest-put-patch-status-codes" name="rest-put-patch-status-codes">:white_check_mark:</a> **DO** follow the processing below to create/update/replace a resource:

使用此方法时 | 如果发生这种情况 | 使用此响应代码
---------------------- | ------------------------- | ----------------------
PATCH/PUT | 任何 JSON 字段名称/值对于 api-version 未知/无效 | `400-Bad Request`
PATCH/PUT | 传递任何读取字段（客户端无法设置读取字段） | `400-Bad Request`
| **如果资源不存在** |
PATCH/PUT | 缺少任何必填的创建/更新字段 | `400-Bad Request`
PATCH/PUT | 使用创建/更新字段创建资源 | `201-Created`
| **如果资源已经存在** |
PATCH | 任何创建字段与当前值不匹配（允许重试） | `409-Conflict`
PATCH | 使用更新字段更新资源 | `200-OK`
PUT | 缺少任何必填的创建/更新字段 | `400-Bad Request`
PUT | 使用创建/更新字段完全覆盖资源 | `200-OK`

#### 处理错误
有两种错误:
- 您期望客户代码在运行时正常恢复的错误
- 错误表明客户代码中存在错误，该错误不太可能在运行时恢复；客户必须修复其代码

✅ 确实返回一个x-ms-error-code带有字符串错误代码的响应头，指示出了什么问题。

*注意: `x-ms-error-code` 值是 API 合同的一部分（因为客户代码可能会对它们进行比较）并且将来不能改变。

✔️ 您可以将值实现x-ms-error-code为枚举，"modelAsString": true因为可以随着时间的推移添加新值。具体而言，只有当相同条件导致不同的顶级错误代码时，这才是重大变更。

⚠️ 您不应该在未升级服务版本的情况下向现有 API 添加新的顶级错误代码。

✅ 务必为运行时可恢复的错误精心设计唯一的x-ms-error-code字符串值。对于不可恢复的使用错误，请重复使用常见的错误代码。

✔️ 您可以将常见的客户代码错误分组为几个x-ms-error-code字符串值。

✅ 一定要确保顶级错误的code值与标题的值相同x-ms-error-code。

✅ 请提供具有以下结构的响应主体：

**错误响应** : Object

属性 | 类型 | 必须 | 描述
-------- | ---- | :------: | -----------
`error` | ErrorDetail | ✔ | 与响应头 `code` 匹配的顶级错误对象 `x-ms-error-code`

**错误详细信息** : Object

属性 | 类型 | 必须 | 描述
-------- | ---- | :------: | -----------
`code` | String | ✔ | 服务器定义的错误代码集之一
`message` | String | ✔ | 人类可读的错误信息
`target` | String |  | 错误的目标.
`details` | ErrorDetail[] |  | 有关导致此报告错误的具体错误的详细信息数组.
`innererror` | InnerError |  | 包含比当前对象更具体的错误信息的对象
_additional properties_ |   | | 调试时可能有用的附加属性。

**内部错误** : Object

属性 | 类型 | 必须 | 描述
-------- | ---- | :------: | -----------
`code` | String |  | 比包含的错误提供的错误代码更具体的错误代码。
`innererror` | InnerError |  | 包含比当前对象更具体的错误信息的对象。

例子:
```json
{
  "error": {
    "code": "InvalidPasswordFormat",
    "message": "Human-readable description",
    "target": "target of error",
    "innererror": {
      "code": "PasswordTooShort",
      "minLength": 6,
    }
  }
}
```

✅ 一定要记录服务的顶级错误代码字符串；它们是 API 合同的一部分。

✔️ 您可以随意处理其他字段，因为它们不属于您服务 API 契约的一部分，客户不应依赖它们或它们的值。它们旨在帮助客户自我诊断问题。

✔️ 您可以为错误消息中的任何数据值添加其他属性，以便客户无需解析错误消息。例如，带有错误信息的错误"message": "A maximum of 16 keys are allowed per account."也可能添加"maximumKeys": 16属性。这不属于 API 契约的一部分，仅应用于诊断问题。

*注意: 不要使用此机制在代码中提供开发人员需要依赖的信息（例如：错误消息可以提供有关您被限制的原因的详细信息，但Retry-After应该是开发人员依赖的内容来退出）。

⚠️ 您不应该在 OpenAPI/Swagger 规范中记录特定的错误状态代码，除非“默认”响应无法正确描述特定的错误响应（例如，主体模式不同）。

### JSON

✅ 请对所有 JSON 字段名称使用驼峰命名法。请勿使用大写缩写；请使用驼峰命名法。

✅ 请区分大小写地处理 JSON 字段名称。

✅ 务必对 JSON 字段值区分大小写。可能会有一些例外，但请尽可能避免。

✅ 务必将表示唯一 ID 的 JSON 字段值视为不透明字符串值，并区分大小写进行比较。例如，ID 通常为[UUIDs](https://en.wikipedia.org/wiki/Universally_unique_identifier), [CUIDs](https://github.com/paralleldrive/cuid2), [Nano ID](https://blog.openapihub.com/en-us/what-is-nano-id-its-difference-from-uuid-as-unique-identifiers/)或其他格式。ID 格式的选择是服务实现的细节。客户代码应该只需要获取、存储和发送这些值，而不应执行任何其他类型的 ID 值解析或解释。

服务以及访问它们的客户端可能使用多种语言编写。为了确保互操作性，JSON 建立了“最低公分母”类型系统，该系统始终以 UTF-8 字节的形式通过网络发送。该系统非常简单，包含三种类型：

 类型 | 描述
 ---- | -----------
 Boolean | true/false (始终小写)
 Number  | 有符号浮点数 (IEEE-754 binary64; int 范围: -2<sup>53</sup>+1 to +2<sup>53</sup>-1)
 String  | 用于其他一切

⛔ 请勿将值为 null 的 JSON 字段从服务端发送到客户端。相反，服务端应该完全不发送此字段（这样可以减少负载大小）。从语义上讲，Azure 服务会将缺失的字段和值为 null 的字段视为相同。

✅ 务必仅在使用 JSON Merge Patch 负载的 PATCH 操作中接受值为 null 的 JSON 字段。值为 null 的字段指示服务删除该字段。如果该字段无法删除，则返回 400-BadRequest，否则返回响应负载中缺失已删除字段的资源（参见上文）。

✅ 请使用 JSON 数字可接受范围内的整数。

✅ 务必为字符串格式建立明确定义的约定。例如，确定最小长度、最大长度、合法字符、大小写敏感比较等等。尽可能使用标准格式，例如，日期/时间格式应遵循 RFC 3339。

✅ 请使用众所周知且易于被许多编程语言解析/格式化的字符串格式，例如用于日期/时间的 RFC 3339。

✅ 一定要确保您的服务和任何客户端之间交换的信息可以跨多种编程语言“往返”。

✅ 请使用[RFC 3339](https://datatracker.ietf.org/doc/html/rfc3339)来表示日期/时间

✅ 使用固定时间间隔来表示持续时间，例如毫秒、秒、分钟、天等，并在属性名称中包含时间单位，例如 backupTimeInMinutes 或ttlSeconds。

✔️ 仅当用户必须能够指定可能逐月或逐年变化的时间间隔时， 才可以使用 [RFC 3339 time intervals](https://wikipedia.org/wiki/ISO_8601#Durations) 时间间隔。例如，“P3M”表示 3 个月，无论开始日期和结束日期之间有多少天；“P1Y”表示闰年 366 天。该值必须可往返。

✅ 请对 UUID使用 [RFC 4122](https://datatracker.ietf.org/doc/html/rfc4122) for UUIDs.

✔️ 您可以使用 JSON 对象将子字段组合在一起。

✔️如果需要维护值的顺序， 您可以使用 JSON 数组。在其他情况下，请避免使用数组，因为数组使用起来可能比较困难且效率低下，尤其是在使用 JSON Merge Patch 时，需要在对数组执行任何操作之前读取整个数组。

☑️ 您应该尽可能使用 JSON 对象而不是数组。

#### 枚举和 SDK（客户端库）

字符串通常具有一组显式的值。这些值通常在 OpenAPI 定义中以枚举的形式体现。这些枚举对于开发人员工具（例如代码补全和客户端库生成）非常有用。

然而，值集在服务的整个生命周期内不断增长的情况并不少见。因此，微软的工具使用了“可扩展枚举”的概念，这意味着值集应该被视为部分列表。这向客户端库和客户表明，枚举字段的值应该有效地视为字符串，并且将来可能会返回未记录的值。这使得值集能够随着时间的推移而增长，同时确保客户端库和客户代码的稳定性。

☑️ 您应该使用可扩展枚举，除非您确信符号集永远不会随着时间而改变。

✅ 一定要向客户说明将来可能会出现的新值，以便客户今天编写代码时能够期待明天出现这些新值。

✔️ 您可能会返回一个可扩展枚举的值，该值不是请求中指定的 api 版本定义的值之一。

⚠️ 您不应该接受可扩展枚举的值，该值不是请求中指定的 api-version 定义的值之一。

⛔ 请勿从枚举列表中删除值，因为这会破坏客户代码。

#### 多态类型

REST API 中的多态类型是指可以使用请求或响应的相同属性来实现相似但不同的形状。这通常 `oneOf` 在 JsonSchema 或 OpenAPI 中表示为 。为了简化如何确定给定请求或响应有效负载对应的特定类型，Azure 要求使用显式鉴别器字段。

注意：多态类型可能会使你的服务更难以被名义类型语言使用。有关更多信息，请参阅[服务设计注意事项中的相应部分](./ConsiderationsForServiceDesign.md#avoid-surprises)

✅ 一定要定义一个鉴别器字段来指示资源的种类，并在正文中包含任何特定于种类的字段。

下面是带有名为 的鉴别器字段的矩形和圆形的 JSON 示例`kind`：

**长方形**
```json
{
   "kind": "rectangle",
   "x": 100,
   "y": 50,
   "width": 10,
   "length": 24,
   "fillColor": "Red",
   "lineColor": "White",
   "subscription": {
      "kind": "free"
   }
}
```

**圆**
```json
{
   "kind": "circle",
   "x": 100,
   "y": 50,
   "radius": 10,
   "fillColor": "Green",
   "lineColor": "Black",
   "subscription": {
      "kind": "paid",
      "expiration": "2024",
      "invoice": "123456"
   }
}
```
长方形和圆都有公共字段：`kind`、`fillColor`、`lineColor和subscription`。长方形还具有`x`、`y`、`width`和 `length`而圆具有`x`、`y`和`radius`。`subscription`是嵌套多态类型。`subscriptionfree`没有其他字段，而`paidsubscription` 有`expiration`和`invoice`字段。

[命名指南](./ConsiderationsForServiceDesign.md#common-names) 建议将鉴别器字段命名为`kind`。

☑️ 您应该将多态类型的鉴别器字段定义为可扩展枚举。

⚠️ 您不应该允许更新（补丁）更改多态类型的鉴别器字段。

⚠️ 您不应返回未在请求中指定的 api-version 定义的多态类型的属性。

⚠️ 您不应该拥有可更新资源的属性，其值是多态对象数组。

如果数组包含多态类型，则使用 JSON 合并补丁更新数组属性不具有版本弹性。

## 常见的 API 模式

### 执行操作
REST 规范用于对资源状态进行建模，主要用于处理 CRUD（创建、读取、更新、删除）操作。然而，许多服务需要能够对资源执行操作，例如获取图像缩略图或重启虚拟机。有时，对集合执行操作也很有用。

☑️ 你应该像这样对你的 URL 进行模式化，以便对资源URL 模式执行操作

```text
https://.../<resource-collection>/<resource-id>:<action>?<input parameters>
```

**例子**
```text
https://.../users/Bob:grant?access=read
```

☑️ 你应该像这样对你的 URL 进行模式化，以便对集合URL 模式执行操作
```text
https://.../<resource-collection>:<action>?<input parameters>
```

**例子**
```text
https://.../users:grant?access=read
```

注意：为了避免操作和资源 ID 之间发生潜在的冲突，您应该禁止在资源 ID 中使用“：”字符。

✅ 对资源或集合执行任何操作时，请使用 POST 操作。

✅ 如果重试时操作需要幂等，请支持 Repeatability-Request-ID 和 Repeatability-First-Sent 请求标头。

✅ 当操作同步且成功完成时，请返回。200-OK

☑️ 你应该使用动词作为<action>路径的组成部分。

⛔ 当操作行为可以合理地定义为标准 REST 创建、读取、更新、删除或列出操作之一时，请勿使用动作操作。

### 收藏
✅ 将对列表操作的响应构造为一个对象，其顶级数组字段包含资源集合（或子集）。

☑️ 如果将来项目数量有可能变得非常大，那么您今天就应该支持分页。

注意：将来添加分页功能是一项重大改变

✔️ 您可以通过支持带有资源集合 URL（而不是资源 ID）的 GET 方法来公开列出您的资源的操作。

**响应主体示例**
```json
{
    "value": [
       { "id": "Item 01", "etag": "\"abc\"", "price": 99.95, "size": "Medium" },
       { … },
       { … },
       { "id": "Item 99", "etag": "\"def\"", "price": 59.99, "size": "Large" }
    ],
    "nextLink": "{opaqueUrl}"
 }
```

✅ 请务必为每件商品添加id字段和etag字段（如果支持），这样方便顾客在后续操作中修改商品。请注意，etag 字段必须包含转义引号；例如，""abc"" 或 W/""abc""。

✅ 一定要清楚地记录资源可能会在分页集合的页面之间被跳过或重复，除非操作已做出特殊规定来防止这种情况（例如对集合进行时间过期快照）。

✅ 确实返回一个nextLink带有绝对 URL 的字段，客户端可以通过 GET 来检索集合的下一页。

注意：该服务负责执行 URL 所需的任何 URL 编码nextLink。

✅ 请在中包含服务所需的任何查询参数nextLink，包括api-version。

☑️ 您应该使用它value作为顶级数组字段的名称，除非有更合适的名称。

⛔ 返回集合的最后一页时，根本不要返回该字段。nextLink

⛔ 不要返回nextLink值为空的字段。

⚠️ 您不应该返回count集合中的所有对象，因为这可能计算成本很高。

#### 查询选项

✔️ 您可以支持以下查询参数，允许客户控制列表操作：

参数名称 | 类型 | 描述
------------------- | ---- | -----------
`filter`       | string            | 选择要返回的资源的资源类型表达式
`orderby`      | string&nbsp;array | 指定返回资源顺序的表达式列表
`skip`         | integer           | 返回第一个资源集合的偏移量
`top`          | integer           | 从集合中返回的最大资源数
`maxpagesize`  | integer           | 单个响应中可包含的最大资源数量
`select`       | string&nbsp;array | 为每个资源返回的字段名称列表
`expand`       | string&nbsp;array | 与每项资源相符的相关资源列表

✅ 如果客户端指定了服务不支持的任何参数，则返回错误。

✅ 请将这些查询参数名称视为区分大小写。

✅ 在应用上表中的所有查询选项后，请应用select或选项。expand

✅ 请按照上表所示的顺序将查询选项应用于集合。

⛔ 请勿在任何查询参数名称前添加“$”作为前缀（[OData标准](http://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html#sec_QueryingCollections)）中的约定.

#### 筛选

✔️ 您可能支持使用查询参数过滤列表操作的结果filter。

查询参数的值filter是一个涉及资源字段的表达式，该表达式会生成一个布尔值。系统会针对集合中的每个资源计算此表达式，只有表达式结果为 true 的项才会包含在响应中。

✅ 应该省略集合中所有过滤表达式计算结果为 false 或 null 的资源，或者引用由于权限原因不可用的属性。

示例：返回所有价格低于 10.00 美元的产品

```text
GET https://api.contoso.com/products?filter=price lt 10.00
```

##### filter operators

:heavy_check_mark: **YOU MAY** support the following operators in filter expressions:

操作                 | 描述           | 例子
--------------------     | --------------------- | -----------------------------------------------------
**比较运算符** |                       |
eq                       | 等于       | city eq 'Redmond'
ne                       | 不等于     | city ne 'London'
gt                       | 大于       | price gt 20
ge                       | 大于或等于  | price ge 10
lt                       | 小于       | price lt 20
le                       | 小于或等于  | price le 100
**逻辑运算符**    |                       |
and                      | 与           | price le 200 and price gt 3.5
or                       | 或           | price le 3.5 or price gt 200
not                      | 非           | not price le 3.5
**分组运算符**   |                       |
( )                      | 优先分组      | (priority eq 1 or city eq 'Redmond') and price gt 100

✅ 如果客户端在过滤表达式中包含操作不支持的运算符，请按照[错误处理](#handling-errors)部分中定义的方式响应错误消息。

✅在评估筛选表达式时， 请务必对支持的运算符使用以下运算符优先级。运算符按类别从高到低的顺序列出。同一类别中的运算符具有相同的优先级，应从左到右进行评估：

| 分组           | 操作 | 描述
| ----------------|----------|------------
| Grouping        | ( )      | 优先分组   |
| Unary           | not      | 逻辑否定     |
| Relational      | gt       | 大于          |
|                 | ge       | 大于或等于 |
|                 | lt       | 少于           |
|                 | le       | 小于或等于   |
| Equality        | eq       | 等于                 |
|                 | ne       | 不等于               |
| Conditional AND | and      | 与        |
| Conditional OR  | or       | 或   |

✔️ 您可以支持 orderby 和 filter 函数，例如 concat 和 contains。有关更多信息，请参阅[odata 规范汗水](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part2-url-conventions.html#_Toc31360979).

##### 运算符示例

以下示例说明了每个逻辑运算符的用法和语义。

例如：所有名称为“牛奶”的产品

```text
GET https://api.contoso.com/products?filter=name eq 'Milk'
```

例如：所有名称不等于“牛奶”的产品

```text
GET https://api.contoso.com/products?filter=name ne 'Milk'
```

例如：所有名称为“牛奶”且价格低于 2.55 的产品：

```text
GET https://api.contoso.com/products?filter=name eq 'Milk' and price lt 2.55
```

例如：所有名称为“牛奶”或价格低于 2.55 的产品：

```text
GET https://api.contoso.com/products?filter=name eq 'Milk' or price lt 2.55
```

例如：所有名称为“牛奶”或“鸡蛋”且价格低于 2.55 的产品：

```text
GET https://api.contoso.com/products?filter=(name eq 'Milk' or name eq 'Eggs') and price lt 2.55
```

#### orderby

✔️ 您可以使用查询参数支持对列表操作的结果进行排序orderby。 注意：服务很少支持此功能，orderby因为它需要对整个大型集合进行排序才能返回任何结果，因此实现成本非常高。

该参数的值orderby是一个逗号分隔的表达式列表，用于对项目进行排序。这种表达式的一个特殊情况是以原始属性结尾的属性路径。

参数值中的每个表达式orderby可能包含后缀“asc”（表示升序）或“desc”（表示降序），并以一个或多个空格与表达式分隔。

✅ 如果未指定“asc”或“desc”，则按表达式的升序对集合进行排序。

✅ 将NULL 值排序为“小于”非 NULL 值。

✅按照第一个表达式的结果值对项目 进行排序，然后按照第二个表达式的结果值对第一个表达式具有相同值的项目进行排序，依此类推。

✅ 请使用字段类型固有的排序顺序。例如，日期时间值应按时间顺序排序，而不是按字母顺序排序。

✅ 如果客户端请求按操作不支持的字段进行排序，请按照“处理错误”部分中定义的方式响应错误消息。

例如，返回按姓名升序排列的所有人：
```text
GET https://api.contoso.com/people?orderby=name
```

例如，返回按姓名降序排列的所有人员，并按雇佣日期升序排列的辅助排序顺序。
```text
GET https://api.contoso.com/people?orderby=name desc,hireDate
```

排序必须与过滤相结合，以便：
```text
GET https://api.contoso.com/people?filter=name eq 'david'&orderby=hireDate
```
将返回所有名字为 David 的人员，并按雇佣日期升序排列。

##### 分页排序的注意事项

✅ 对分页列表操作响应的所有页面使用相同的过滤选项和排序顺序。

##### 跳过

✅ 请将参数定义skip为整数，其默认值和最小值为 0。

✔️ 您可以允许客户端传递skip查询参数来指定要返回的第一个资源集合的偏移量。

##### 顶部

✔️ 您可以允许客户端传递top查询参数来指定从集合中返回的最大资源数。

如果支持top，请将参数定义top为最小值为 1 的整数。如果未指定，top则默认值为无穷大。

✅ 返回集合的top资源数量（如果可用），从 开始skip。

##### 最大页面大小

✔️ 您可以允许客户端传递maxpagesize查询参数来指定单个页面响应中包含的最大资源数。

✅ 请将参数定义maxpagesize为可选整数，并具有适合集合的默认值。

✅ 在参数文档中明确说明操作maxpagesize可能会选择返回比指定值更少的资源。

### API 版本控制

服务需要随着时间推移而变化。但是，更改服务时需要满足两个要求：
 1. 已运行的客户工作负载不得因服务变更而中断
 2. 客户可以采用新的服务版本，而无需进行任何代码更改（当然，客户必须修改代码才能利用任何新的服务功能。）

注意：Azure 重大变更政策（第 5 节）中包含表格，其中描述了哪些类型的变更被视为重大变更。如果Azure 重大变更审核人员批准，则允许进行重大变更（出于安全/合规性等原因），但必须与客户充分沟通并经过较长的弃用期。

✅ 请与 Azure API 管理委员会一起审查任何 API 变更

客户端指定在对服务的每次请求中使用的 API 版本，甚至对服务返回的Operation-Location或URL 的请求。nextLink

✅在每个操作中 都使用一个名为必需的查询参数，api-version以便客户端指定 API 版本。

✅ 请使用YYYY-MM-DD日期值（带有-preview预览版本的后缀）作为 的有效值api-version。

✅如果客户端省略查询参数， 则返回 HTTP 400，错误代码为“MissingApiVersionParameter”，并显示消息“所有请求都需要 api-version 查询参数 (?api-version=)” api-version。

✅如果客户端传递了服务无法识别的值， 请返回 HTTP 400 错误，错误代码为“UnsupportedApiVersionValue”，并显示“不支持的 API 版本‘{0}’。支持的 API 版本为‘{1}’。”api-version支持的 API 版本，请列出服务仍支持的所有稳定版本以及最新的公开预览版（如有）。

```text
PUT https://service.azure.com/users/Jeff?api-version=2021-06-04
```

✅ 务必为每个新预览版本设置一个更晚的日期

发布新预览版时，服务团队可能会在给予客户至少 90 天的时间来升级代码后，完全淘汰任何以前的预览版本

⛔ 请勿对服务引入任何重大变更。

⛔ 请勿在任何操作路径中包含版本号段。

⛔从预览版 API 过渡到正式版 API 时， 请勿使用相同的日期。如果预览版的日期api-version是“2021-06-04-preview”，则 API 的正式版日期必须晚于 2021-06-04

⛔ 请勿将预览功能保留在预览状态超过 1 年；它必须在推出后 1 年内进入 GA（或被删除）。

#### 使用可扩展枚举

虽然从枚举中删除值是一项重大更改，但可以使用可扩展枚举来处理向枚举中添加值。可扩展枚举是一个带有特殊标记的字符串值 -在代码块modelAsString中设置为 true x-ms-enum。例如：

```json
"createdByType": {
   "type": "string",
   "description": "The type of identity that created the resource.",
   "enum": [
      "User",
      "Application",
      "ManagedIdentity",
      "Key"
   ],
   "x-ms-enum": {
      "name": "createdByType",
      "modelAsString": true
   }
}
```

☑️ 您应该使用可扩展枚举，除非您确信符号集永远不会随着时间而改变。

### 弃用行为通知

如果无法遵循上述API 版本控制指南，且Azure 重大变更审阅者批准了对特定 API 版本的重大变更，则必须将该变更告知其调用者。即将弃用的 API 版本必须添加一个azure-deprecating响应标头，其中包含一个以分号分隔的字符串，用于通知调用者哪些 API 即将被弃用、何时将不再起作用，以及一个指向更多信息（例如应改用哪些新操作）的 URL 链接。

其目的是告知客户（在调试/记录响应时），他们必须采取措施修改对服务操作的调用并使用较新的 API 版本，否则他们的调用将很快完全停止工作。客户端代码不应以任何方式检查/解析此标头的值；它仅供人类参考。该字符串不属于API契约的一部分（分号分隔符除外），并且可以随时更改/改进，而不会导致重大变更。

✅ 仅当操作将来会停止工作并且客户端必须采取行动才能使其继续工作时，才azure-deprecating在操作的响应中包含标头。

注意：我们不想用这个标题吓到客户。

✅ 务必使标头的值成为一个以分号分隔的字符串，该字符串指示一组弃用内容，其中每个弃用内容均指示弃用的内容、弃用时间以及指向更多信息的 URL。

弃用应使用以下模式：

```text
<description> will retire on <date> (<url>)
```

允许多次弃用，以分号分隔。

应提供以下占位符：
- `description`: 关于弃用内容的可读描述
- `date`: 此方法将被弃用的预期日期。应遵循[ISO 8601](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.1.1), e.g. "2022-10-31".
- `url`: 一个完全限定的 URL，用户可以关注该 URL 来了解有关弃用内容的更多信息，最好是 Azure 更新

例如:
- `azure-deprecating: API version 2009-27-07 will retire on 2022-12-01 (https://azure.microsoft.com/updates/video-analyzer-retirement);TLS 1.0 & 1.1 will retire on 2020-10-30 (https://azure.microsoft.com/updates/azure-active-directory-registration-service-is-ending-support-for-tls-10-and-11/)`
- `azure-deprecating: Model version 2021-01-15 used in Sentiment analysis will retire on 2022-12-01 (https://aka.ms/ta-modelversions?sentimentAnalysis)`
- `azure-deprecating: TLS 1.0 & 1.1 support will retire on 2022-10-01 (https://devblogs.microsoft.com/devops/deprecating-weak-cryptographic-standards-tls-1-0-and-1-1-in-azure-devops-services/)`

⛔未经 Azure 重大变更审阅者批准和Azure 更新上的官方弃用通知，请勿引入此标头。

### 请求的可重复性

容错应用程序要求客户端重试从未收到响应的请求，并且服务必须幂等地处理这些重试的请求。在 Azure 中，所有 HTTP 操作都自然是幂等的，但用于创建资源的 POST 和 [用于调用操作的 POST](
https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md#performing-an-action)除外.

☑️ 您应该支持[OASIS](https://docs.oasis-open.org/odata/repeatable-requests/v1.0/repeatable-requests-v1.0.html)可重复请求版本 1.0 中定义的POST 操作的可重复请求，以使其可重试。

- 跟踪的时间窗口（该Repeatability-First-Sent值与当前时间之间的差异）必须至少为 5 分钟。
- 在 API 契约和文档中记录 POST 操作对Repeatability-First-Sent、Repeatability-Request-ID和标头的支持。Repeatability-Result
- 对于包含有效可重复性请求标头的任何请求，任何不支持可重复性标头的操作都应返回 501（未实现）响应。

### 长期运行的操作和作业

长时间运行操作 (LRO)通常应该同步执行，但由于服务不希望维持长连接（超过 1 秒）且负载均衡器超时，因此该操作必须异步执行。对于这种模式，客户端在服务上启动操作，然后客户端反复轮询服务（通过另一个 API 调用）以跟踪操作的进度/完成情况

LRO 始终由一个逻辑客户端启动，并可能由同一客户端、另一个客户端甚至多个客户端/浏览器轮询（检查其状态）。例如，显示所有操作及其状态的仪表板或门户。有关长时间运行操作设计的介绍，请参阅“服务设计注意事项”中的“长时间运行操作”部分。

✅ 如果第 99 个百分位响应时间大于 1 秒，并且客户端应在取得更多进展之前轮询该操作，则应将操作实现为 LRO。

⛔ 请勿将 PATCH 实现为 LRO。如果需要 LRO 更新语义，请使用LRO POST 操作模式实现。

#### 启动长时间运行操作的模式

✅ 在启动 LRO 操作时，请尽可能多地执行验证，以便尽早向客户端发出错误警报。

✅ 请在响应​​标头中operation-location包含操作状态监视器的绝对 URL。

☑️ 您应该api-version在响应标头中包含查询参数operation-location，其版本与初始请求中传递的版本相同，但希望客户端将该api-version值更改为新/不同客户端希望的值。

✅ 请在响应​​标头中包含向状态监视器发出GET 轮询请求所需的任何附加值（例如位置）。

#### 创建或替换具有额外长时间运行处理的操作

✅ 在实现创建或替换涉及额外长时间运行处理的资源的操作时，请使用以下模式：

```text
PUT /UrlToResourceBeingCreated?api-version=<api-version>
operation-id: <optionalStatusMonitorResourceId>

<JSON Resource in body>
```

响应必须如下所示：

```text
201 Created
operation-id: <statusMonitorResourceId>
operation-location: https://operations/<operation-id>?api-version=<api-version>

<JSON Resource in body>
```

请求和响应主体模式必须相同并代表资源。

PUT 立即创建或替换资源并返回，但额外的长时间运行的处理可能需要一些时间才能完成。

对于幂等 PUT（operation-id在某个短时间窗口内相同或相同的请求主体），服务应该返回与上面相同的响应。

对于非幂等 PUT，服务可以选择覆盖现有资源（就好像资源被删除一样）或者服务可以返回409-Conflict错误的代码属性，指示此 PUT 操作失败的原因。

✅ 确实允许客户端传递一个Operation-Id带有操作状态监视器 ID 的标头。

如果Operation-Id未指定标头，服务可能会创建一个操作 ID（通常是 GUID）并通过operation-id和operation-location响应标头返回它；在这种情况下，服务必须弄清楚如何处理重试/幂等性。

✅ 如果客户端未传递标头，则为状态监视器生成一个 ID（通常是 GUID） 。Operation-Id

✅ 如果标头与现有操作匹配，则请使请求失败，除非该请求与之前的请求相同（重试场景）。409-ConflictOperation-Id

✅ 在启动操作时，请尽可能多地执行验证，以便尽早提醒客户端错误。

✅ 如果资源已成功创建或替换，则返回初始请求的201-Created创建或替换状态代码，并附带资源的表示形式。200-OK

✅在响应中 包含一个Operation-Id带有操作状态监视器 ID 的标题。

☑️ 您应该在响应中包含一个Operation-Location标题，其中包含操作状态监视器的绝对 URL。

☑️ 您应该api-version在标头中包含Operation-Location与初始请求中传递的版本相同的查询参数。

#### DELETE LRO pattern

<a href="#lro-delete" name="lro-delete">:white_check_mark:</a> **DO** use the following pattern when implementing an LRO operation to delete a resource:

```text
DELETE /UrlToResourceBeingDeleted?api-version=<api-version>
operation-id: <optionalStatusMonitorResourceId>
```

The response must look like this:

```text
202 Accepted
operation-id: <statusMonitorResourceId>
operation-location: https://operations/<operation-id>
```

Consistent with non-LRO DELETE operations, if a request body is specified, return `400-Bad Request`.

<a href="#lro-delete-operation-id-request-header" name="lro-delete-operation-id-request-header">:white_check_mark:</a> **DO** allow the client to pass an `Operation-Id` header with an ID for the operation's status monitor.

<a href="#lro-delete-operation-id-default-is-guid" name="lro-delete-operation-id-default-is-guid">:white_check_mark:</a> **DO** generate an ID (typically a GUID) for the status monitor if the `Operation-Id` header was not passed by the client.

<a href="#lro-delete-returns-202" name="lro-delete-returns-202">:white_check_mark:</a> **DO** return a `202-Accepted` status code from the request that initiates an LRO if the processing of the operation was successfully initiated.

<a href="#lro-delete-returns-only-202" name="lro-delete-returns-only-202">:warning:</a> **YOU SHOULD NOT** return any other `2xx` status code from the initial request of an LRO -- return `202-Accepted` and a status monitor even if processing was completed before the initiating request returns.

#### LRO action on a resource pattern
<a href="#post-or-delete-lro-pattern" name="post-or-delete-lro-pattern"></a><!-- Preserve old header link -->

<a href="#lro-existing-resource" name="lro-existing-resource">:white_check_mark:</a> **DO** use the following pattern when implementing an LRO action operating on an existing resource:

```text
POST /UrlToExistingResource:<action>?api-version=<api-version>&<actionParamsGoHere>
operation-id: <optionalStatusMonitorResourceId>`

<JSON Action parameters can go in body if query params don't work>
```

The response must look like this:

```text
202 Accepted
operation-id: <statusMonitorResourceId>
operation-location: https://operations/<operation-id>

<JSON Status Monitor Resource in body>
```

The request body contains information to be used to execute the action.

For an idempotent POST (same `operation-id` and request body within some short time window), the service should return the same response as the initial request.

For a non-idempotent POST, the service can treat the POST operation as idempotent (if performed within a short time window) or can treat the POST operation as initiating a brand new LRO action operation.

<a href="#lro-no-post-create" name="lro-no-post-create">:no_entry:</a> **DO NOT** use a long-running POST to create a resource -- use PUT as described above.

<a href="#lro-operation-id-request-header" name="lro-operation-id-request-header">:white_check_mark:</a> **DO** allow the client to pass an `Operation-Id` header with an ID for the operation's status monitor.

<a href="#lro-operation-id-default-is-guid" name="lro-operation-id-default-is-guid">:white_check_mark:</a> **DO** generate an ID (typically a GUID) for the status monitor if the `Operation-Id` header was not passed by the client.

<a href="#lro-operation-id-unique-except-retries" name="lro-operation-id-unique-except-retries">:white_check_mark:</a> **DO** fail a request with a `409-Conflict` if the `Operation-Id` header matches an existing operation unless the request is identical to the prior request (a retry scenario).

<a href="#lro-returns-202" name="lro-returns-202">:white_check_mark:</a> **DO** return a `202-Accepted` status code from the request that initiates an LRO action on a resource if the processing of the operation was successfully initiated.

<a href="#lro-returns-only-202" name="lro-returns-only-202">:warning:</a> **YOU SHOULD NOT** return any other `2xx` status code from the initial request of an LRO -- return `202-Accepted` and a status monitor even if processing was completed before the initiating request returns.

<a href="#lro-returns-status-monitor" name="lro-returns-status-monitor">:white_check_mark:</a> **DO** return a status monitor in the response body as described in [Obtaining status and results of long-running operations](#obtaining-status-and-results-of-long-running-operations).

#### LRO action with no related resource pattern

<a href="#lro-action-no-resource" name="lro-action-no-resource">:white_check_mark:</a> **DO** use the following pattern when implementing an LRO action not related to a specific resource (such as a batch operation):

```text
PUT <operation-endpoint>/<operation-id>?api-version=<api-version>

<JSON body with parameters for the operation>>
```

The response must look like this:

```text
201 Created
operation-location: <absolute URL of status monitor>

<JSON Status Monitor Resource in body>
```

<a href="#lro-put-action-operation-endpoint" name="lro-put-action-operation-endpoint">:ballot_box_with_check:</a> **YOU SHOULD**
define a unique operation endpoint for each LRO action with no related resource.

<a href="#lro-put-action-operation-id" name="lro-put-action-operation-id-in-path">:white_check_mark:</a> **DO** require the
`Operation-Id` as the final path segment in the URL.

Note: The `operation-id` URL segment (not header) is *required*, forcing the client to specify the status monitor's resource ID
and is also used for retries/idempotency.

<a href="#lro-put-action-returns-201" name="lro-put-action-returns-201">:white_check_mark:</a> **DO** return a `201 Created` status code
with an `operation-location` response header if the LRO Action operation was accepted for processing.

<a href="#lro-put-action-returns-status-monitor" name="lro-put-action-returns-status-monitor">:white_check_mark:</a> **DO** return a
status monitor in the response body that contains the operation status, request parameters, and when the operation completes either
the operation result or error.

Note: Since all request parameters must be present in the status monitor,
the request and response body of the PUT can be defined with a single schema.

<a href="#lro-put-action-status-monitor-url" name="lro-put-action-status-monitor-url">:ballot_box_with_check:</a> **YOU SHOULD**
return the status monitor for an operation for a subsequent GET on the URL that initiates the LRO, and use this endpoint as
the status monitor URL returned in the `operation-location` response header.

#### The Status Monitor Resource

All patterns that initiate a LRO either implicitly or explicitly create a [Status Monitor resource](https://datatracker.ietf.org/doc/html/rfc7231#section-6.3.3) in the service's `operations` collection.

<a href="#lro-status-monitor-structure" name="lro-status-monitor-structure">:white_check_mark:</a> **DO** return a status monitor in the response body that conforms with the following structure:

Property | Type        | Required | Description
-------- | ----------- | :------: | -----------
`id`     | string      | true     | The unique id of the operation
`kind`   | string enum | true(*)  | The kind of operation
`status` | string enum | true     | The operation's current status: "NotStarted", "Running", "Succeeded", "Failed", and "Canceled"
`error`  | ErrorDetail |          | If `status`=="Failed", contains reason for failure
`result` | object      |          | If `status`=="Succeeded" && Action LRO (POST or PUT), contains success result if needed
additional<br/>properties | | | Additional named or dynamic properties of the operation

(*): When a status monitor endpoint supports multiple operations with different result structures or additional properties,
the status monitor **must** be polymorphic -- it **must** contain a required `kind` property that indicates the kind of long-running operation.

#### Obtaining status and results of long-running operations

<a href="#lro-poll" name="lro-poll">:white_check_mark:</a> **DO** use the following pattern to allow clients to poll the current state of a Status Monitor resource:

```text
GET <operation-endpoint>/<operation-id>?api-version=<api-version>
```

The response must look like this:

```text
200 OK
retry-after: <delay-seconds>    (if status not terminal)

<JSON Status Monitor Resource in body>
```

<a href="#lro-status-monitor-get-returns-200" name="lro-status-monitor-get-returns-200">:white_check_mark:</a> **DO** support the GET method on the status monitor endpoint that returns a `200-OK` response with the current state of the status monitor.

<a href="#lro-status-monitor-accepts-any-api-version" name="lro-status-monitor-accepts-any-api-version">:ballot_box_with_check:</a> **YOU SHOULD** allow any valid value of the `api-version` query parameter to be used in the GET operation on the status monitor.

- Note: Clients may replace the value of `api-version` in the `operation-location` URL with a value appropriate for their application. Remember that the client initiating the LRO may not be the same client polling the LRO's status.

<a href="#lro-status-monitor-includes-all-fields" name="lro-status-monitor-includes-all-fields">:white_check_mark:</a> **DO** include the `id` of the operation and any other values needed for the client to form a GET request to the status monitor (e.g. a `location` path parameter).

<a href="#lro-status-monitor-post-action-result" name="lro-status-monitor-post-action-result">:white_check_mark:</a> **DO** include the `result` property (if any) in the status monitor for a POST action-type long-running operation when the operation completes successfully.

<a href="#lro-status-monitor-no-resource-result" name="lro-status-monitor-no-resource-result">:no_entry:</a> **DO NOT** include a `result` property in the status monitor for a long-running operation that is not an action-type long-running operation.

<a href="#lro-status-monitor-retry-after" name="lro-status-monitor-retry-after">:white_check_mark:</a> **DO** include a `retry-after` header in the response if the operation is not complete. The value of this header should be an integer number of seconds that the client should wait before polling the status monitor again.

<a href="#lro-status-monitor-retention" name="lro-status-monitor-retention">:white_check_mark:</a> **DO** retain the status monitor resource for some publicly documented period of time (at least 24 hours) after the operation completes.

#### Pattern to List Status Monitors

Use the following patterns to allow clients to list Status Monitor resources.

<a href="#lro-list-status-monitors" name="lro-list-status-monitors">:ballot_box_with_check:</a>
**YOU MAY** support a GET method on any status monitor collection URL that returns a list of the status monitors in that collection.

<a href="#lro-put-action-list-status-monitors" name="lro-put-action-list-status-monitors">:ballot_box_with_check:</a>
**YOU SHOULD** support a list operation for any status monitor collection that includes status monitors for LRO Actions with no related resource.

<a href="#lro-list-status-monitors-filter" name="lro-list-status-monitors-filter">:ballot_box_with_check:</a>
**YOU SHOULD** support the `filter` query parameter on the list operation for any polymorphic status monitor collection and support filtering on the `kind` value of the status monitor.

For example, the following request should return all status monitor resources whose `kind` is either "VMInitializing" *or* "VMRebooting"
and whose status is "NotStarted" *or* "Succeeded".

```text
GET /operations?filter=(kind eq 'VMInitializing' or kind eq 'VMRebooting') and (status eq 'NotStarted' or status eq 'Succeeded')
```

<a href="#byos" name="byos"></a>
### Bring your own Storage (BYOS)

Many services need to store and retrieve data files. For this scenario, the service should not implement its own
storage APIs and should instead leverage the existing Azure Storage service. When doing this, the customer
"owns" the storage account and just tells your service to use it. Colloquially, we call this <i>Bring Your Own Storage</i> as the customer is bringing their storage account to another service. BYOS provides significant benefits to service implementors: security, performance, uptime, etc. And, of course, most Azure customers are already familiar with the Azure Storage service.

While Azure Managed Storage may be easier to get started with, as your service evolves and matures, BYOS provides the most flexibility and implementation choices. Further, when designing your APIs, be cognizant of expressing storage concepts and how clients will access your data. For example, if you are working with blobs, then you should not expose the concept of folders.

<a href="#byos-pattern" name="byos-pattern">:white_check_mark:</a> **DO** use the Bring Your Own Storage pattern.

<a href="#byos-prefix-for-folder" name="byos-prefix-for-folder">:white_check_mark:</a> **DO** use a blob prefix for a logical folder (avoid terms such as `directory`, `folder`, or `path`).

<a href="#byos-allow-container-reuse" name="byos-allow-container-reuse">:no_entry:</a> **DO NOT** require a fresh container per operation.

<a href="#byos-authorization" name="byos-authorization">:white_check_mark:</a> **DO** use managed identity and Role Based Access Control ([RBAC](https://docs.microsoft.com/azure/role-based-access-control/overview)) as the mechanism allowing customers to grant permission to their Storage account to your service.

<a href="#byos-define-rbac-roles" name="byos-define-rbac-roles">:white_check_mark:</a> **DO** Add RBAC roles for every service operation that requires accessing Storage scoped to the exact permissions.

<a href="#byos-rbac-compatibility" name="byos-rbac-compatibility">:white_check_mark:</a> **DO** Ensure that RBAC roles are backward compatible, and specifically, do not take away permissions from a role that would break the operation of the service. Any change of RBAC roles that results in a change of the service behavior is considered a breaking change.


#### Handling 'downstream' errors
It is not uncommon to rely on other services, e.g. storage, when implementing your service. Inevitably, the services you depend on will fail. In these situations, you can include the downstream error code and text in the inner-error of the response body. This provides a consistent pattern for handling errors in the services you depend upon.

<a href="#byos-include-downstream-errors" name="byos-include-downstream-errors">:white_check_mark:</a> **DO** include error from downstream services as the 'inner-error' section of the response body.

#### Working with files
Generally speaking, there are two patterns that you will encounter when working with files; single file access, and file collections.

##### Single file access
Designing an API for accessing a single file, depending on your scenario, is relatively straight forward.

<a href="#byos-sas-token" name="byos-sas-token">:heavy_check_mark:</a> **YOU MAY** use a Shared Access Signature [SAS](https://docs.microsoft.com/azure/storage/common/storage-sas-overview) to provide access to a single file. SAS is considered the minimum security for files and can be used in lieu of, or in addition to, RBAC.

<a href="#byos-http-insecure" name="byos-http-insecure">:ballot_box_with_check:</a> **YOU SHOULD** if using HTTP (not HTTPS) document to users that all information is sent over the wire in clear text.

<a href="#byos-http-status-code" name="byos-http-status-code">:white_check_mark:</a> **DO** return an HTTP status code representing the result of your service operation's behavior.

<a href="#byos-include-storage-error" name="byos-include-storage-error">:white_check_mark:</a> **DO** include the Storage error information in the 'inner-error' section of an error response if the error was the result of an internal Storage operation failure. This helps the client determine the underlying cause of the error, e.g.: a missing storage object or insufficient permissions.

<a href="#byos-support-single-object" name="byos-support-single-object">:white_check_mark:</a> **DO** allow the customer to specify a URL path to a single Storage object if your service requires access to a single file.

<a href="#byos-last-modified" name="byos-last-modified">:heavy_check_mark:</a> **YOU MAY** allow the customer to provide a [last-modified](https://datatracker.ietf.org/doc/html/rfc7232#section-2.2) timestamp (in RFC 7231 format) for read-only files. This allows the client to specify exactly which version of the files your service should use.
When reading a file, your service passes this timestamp to Azure Storage using the [if-unmodified-since](https://datatracker.ietf.org/doc/html/rfc7232#section-3.4) request header. If the Storage operation fails with 412, the Storage object was modified and your service operation should return an appropriate 4xx status code and return the Storage error in your operation's 'inner-error' (see guideline above).

<a href="#byos-folder-support" name="byos-folder-support">:white_check_mark:</a> **DO** allow the customer to specify a URL path to a logical folder (via prefix and delimiter) if your service requires access to multiple files (within this folder). For more information, see [List Blobs API](https://docs.microsoft.com/rest/api/storageservices/list-blobs)

<a href="#byos-extensions" name="byos-extensions">:heavy_check_mark:</a> **YOU MAY** offer an `extensions` field representing an array of strings indicating file extensions of desired blobs within the logical folder.

A common pattern when working with multiple files is for your service to receive requests that contain the location(s) of files to process ("input") and a location(s) to place any files that result from processing ("output"). Note: the terms "input" and "output" are just examples; use terms more appropriate to your service's domain.

For example, a service's request body to configure BYOS may look like this:

```json
{
  "input":{
    "location": "https://mycompany.blob.core.windows.net/documents/english/?<sas token>",
    "delimiter": "/",
    "extensions" : [ ".bmp", ".jpg", ".tif", ".png" ],
    "lastModified": "Wed, 21 Oct 2015 07:28:00 GMT"
  },
  "output":{
    "location": "https://mycompany.blob.core.windows.net/documents/spanish/?<sas token>",
    "delimiter":"/"
  }
}
```

Depending on the requirements of the service, there can be any number of "input" and "output" sections, including none.

<a href="#byos-location-and-delimiter" name="byos-location-and-delimiter">:white_check_mark:</a> **DO** include a JSON object that has string values for "location" and "delimiter". For "location", the customer must pass a URL to a blob prefix which represents a directory. For "delimiter", the customer must specify the delimiter character they desire to use in the location URL; typically "/" or "\".

<a href="#byos-directory-last-modified" name="byos-directory-last-modified">:heavy_check_mark:</a> **YOU MAY** support the "lastModified" field for input directories (see guideline above).

<a href="#byos-sas-for-input-location" name="byos-sas-for-input-location">:white_check_mark:</a> **DO** support a "location" URL with a container-scoped SAS that has a minimum of `listing` and `read` permissions for input directories.

<a href="#byos-sas-for-output-location" name="byos-sas-for-output-location">:white_check_mark:</a> **DO** support a "location" URL with a container-scoped SAS that has a minimum of `write` permissions for output directories.

<a href="#condreq" name="condreq"></a>
### Conditional Requests

The [HTTP Standard][] defines request headers that clients may use to specify a _precondition_
for execution of an operation. These headers allow clients to implement efficient caching mechanisms
and avoid data loss in the event of concurrent updates to a resource. The headers that specify conditional execution are `If-Match`, `If-None-Match`, `If-Modified-Since`, `If-Unmodified-Since`, and `If-Range`.

[HTTP Standard]: https://datatracker.ietf.org/doc/html/rfc9110

<!-- condreq-support-etags-consistently has been subsumed by condreq-support but we retain the anchor to avoid broken links -->
<a href="#condreq-support-etags-consistently" name="condreq-support-etags-consistently"></a>
<!-- condreq-for-read has been subsumed by condreq-support but we retain the anchor to avoid broken links -->
<a href="#condreq-for-read" name="condreq-for-read"></a>
<!-- condreq-no-pessimistic-update has been subsumed by condreq-support but we retain the anchor to avoid broken links -->
<a href="#condreq-no-pessimistic-update" name="condreq-no-pessimistic-update"></a>
<a href="#condreq-support" name="condreq-support">:white_check_mark:</a> **DO** honor any precondition headers received as part of a client request.

The HTTP Standard does not allow precondition headers to be ignored, as it can be unsafe to do so.

<a href="#condreq-unsupported-error" name="condreq-unsupported-error">:white_check_mark:</a> **DO** return the appropriate precondition failed error response if the service cannot verify the truth of the precondition.

Note: The Azure Breaking Changes review board will allow a GA service that currently ignores precondition headers to begin honoring them in a new API version without a formal breaking change notification. The potential for disruption to customer applications is low and outweighed by the value of conforming to HTTP standards.

While conditional requests can be implemented using last modified dates, entity tags ("ETags") are strongly
preferred since last modified dates cannot distinguish updates made less than a second apart.

<a href="#condreq-return-etags" name="condreq-return-etags">:ballot_box_with_check:</a> **YOU SHOULD** return an `ETag` with any operation returning the resource or part of a resource or any update of the resource (whether the resource is returned or not).

#### Conditional Request behavior

This section gives a summary of the processing to perform for precondition headers.
See the [Conditional Requests section of the HTTP Standard][] for details on how and when to evaluate these headers.

[Conditional Requests section of the HTTP Standard]: https://datatracker.ietf.org/doc/html/rfc9110#name-conditional-requests

<a href="#condreq-for-read-behavior" name="condreq-for-read-behavior">:white_check_mark:</a> **DO** adhere to the following table for processing a GET request with precondition headers:

| GET Request | Return code | Response                                    |
|:------------|:------------|:--------------------------------------------|
| ETag value = `If-None-Match` value   | `304-Not Modified` | no additional information   |
| ETag value != `If-None-Match` value  | `200-OK`           | Response body include the serialized value of the resource (typically JSON)    |

For more control over caching, please refer to the `cache-control` [HTTP header](https://developer.mozilla.org/docs/Web/HTTP/Headers/Cache-Control).

<a href="#condreq-behavior" name="condreq-behavior">:white_check_mark:</a> **DO** adhere to the following table for processing a PUT, PATCH, or DELETE request with precondition headers:

| Operation   | Header        | Value | ETag check | Return code | Response       |
|:------------|:--------------|:------|:-----------|:------------|----------------|
| PATCH / PUT | `If-None-Match` | *     | check for _any_ version of the resource ('*' is a wildcard used to match anything), if none are found, create the resource. | `200-OK` or </br> `201-Created` </br> | Response header MUST include the new `ETag` value. Response body SHOULD include the serialized value of the resource (typically JSON).  |
| PATCH / PUT | `If-None-Match` | *     | check for _any_ version of the resource, if one is found, fail the operation |  `412-Precondition Failed` | Response body SHOULD return the serialized value of the resource (typically JSON) that was passed along with the request.|
| PATCH / PUT | `If-Match` | value of ETag     | value of `If-Match` equals the latest ETag value on the server, confirming that the version of the resource is the most current | `200-OK` or </br> `201-Created` </br> | Response header MUST include the new `ETag` value. Response body SHOULD include the serialized value of the resource (typically JSON).  |
| PATCH / PUT | `If-Match` | value of ETag     | value of `If-Match` header DOES NOT equal the latest ETag value on the server, indicating a change has ocurred since after the client fetched the resource|  `412-Precondition Failed` | Response body SHOULD return the serialized value of the resource (typically JSON) that was passed along with the request.|
| DELETE      | `If-Match` | value of ETag     | value matches the latest value on the server | `204-No Content` | Response body SHOULD be empty.  |
| DELETE      | `If-Match` | value of ETag     | value does NOT match the latest value on the server | `412-Preconditioned Failed` | Response body SHOULD be empty.|

#### Computing ETags

The strategy that you use to compute the `ETag` depends on its semantic. For example, it is natural, for resources that are inherently versioned, to use the version as the value of the `ETag`. Another common strategy for determining the value of an `ETag` is to use a hash of the resource. If a resource is not versioned, and unless computing a hash is prohibitively expensive, this is the preferred mechanism.

<a href="#condreq-etag-is-hash" name="condreq-etag-is-hash">:ballot_box_with_check:</a> **YOU SHOULD** use a hash of the representation of a resource rather than a last modified/version number

While it may be tempting to use a revision/version number for the resource as the ETag, it interferes with client's ability to retry update requests. If a client sends a conditional update request, the service acts on the request, but the client never receives a response, a subsequent identical update will be seen as a conflict even though the retried request is attempting to make the same update.

<a href="#condreq-etag-hash-entire-resource" name="condreq-etag-hash-entire-resource">:ballot_box_with_check:</a> **YOU SHOULD**, if using a hash strategy, hash the entire resource.

<a href="#condreq-strong-etag-for-range-requests" name="condreq-strong-etag-for-range-requests">:ballot_box_with_check:</a> **YOU SHOULD**, if supporting range requests, use a strong ETag in order to support caching.

<a href="#condreq-timestamp-precision" name="condreq-timestamp-precision">:heavy_check_mark:</a> **YOU MAY** use or, include, a timestamp in your resource schema. If you do this, the timestamp shouldn't be returned with more than subsecond precision, and it SHOULD be consistent with the data and format returned, e.g. consistent on milliseconds.

<a href="#condreq-weak-etags-allowed" name="condreq-weak-etags-allowed">:heavy_check_mark:</a> **YOU MAY** consider Weak ETags if you have a valid scenario for distinguishing between meaningful and cosmetic changes or if it is too expensive to compute a hash.

<a href="#condreq-etag-depends-on-encoding" name="condreq-etag-depends-on-encoding">:white_check_mark:</a> **DO**, when supporting multiple representations (e.g. Content-Encodings) for the same resource, generate different ETag values for the different representations.

<a href="#substrings" name="substrings"></a>
### Returning String Offsets & Lengths (Substrings)

All string values in JSON are inherently Unicode and UTF-8 encoded, but clients written in a high-level programming language must work with strings in that language's string encoding, which may be UTF-8, UTF-16, or CodePoints (UTF-32).
When a service response includes a string offset or length value, it should specify these values in all 3 encodings to simplify client development and ensure customer success when isolating a substring.
See the [Returning String Offsets & Lengths] section in Considerations for Service Design for more detail, including an example JSON response containing string offset and length fields.

[Returning String Offsets & Lengths]: https://github.com/microsoft/api-guidelines/blob/vNext/azure/ConsiderationsForServiceDesign.md#returning-string-offsets--lengths-substrings

<a href="#substrings-return-value-for-each-encoding" name="substrings-return-value-for-each-encoding">:white_check_mark:</a> **DO** include all 3 encodings (UTF-8, UTF-16, and CodePoint) for every string offset or length value in a service response.

<a href="#substrings-return-value-structure" name="substrings-return-value-structure">:white_check_mark:</a> **DO** define every string offset or length value in a service response as an object with the following structure:

| Property    | Type    | Required | Description |
| ----------- | ------- | :------: | ----------- |
| `utf8`      | integer | true     | The offset or length of the substring in UTF-8 encoding |
| `utf16`     | integer | true     | The offset or length of the substring in UTF-16 encoding |
| `codePoint` | integer | true     | The offset or length of the substring in CodePoint encoding |

<a href="#telemetry" name="telemetry"></a>
### Distributed Tracing & Telemetry

Azure SDK client guidelines specify that client libraries must send telemetry data through the `User-Agent` header, `X-MS-UserAgent` header, and Open Telemetry.
Client libraries are required to send telemetry and distributed tracing information on every  request. Telemetry information is vital to the effective operation of your service and should be a consideration from the outset of design and implementation efforts.

<a href="#telemetry-headers" name="telemetry-headers">:white_check_mark:</a> **DO** follow the Azure SDK client guidelines for supporting telemetry headers and Open Telemetry.

<a href="#telemetry-allow-unrecognized-headers" name="telemetry-allow-unrecognized-headers">:no_entry:</a> **DO NOT** reject a call if you have custom headers you don't understand, and specifically, distributed tracing headers.

**Additional References**
- [Azure SDK client guidelines](https://azure.github.io/azure-sdk/general_azurecore.html)
- [Azure SDK User-Agent header policy](https://azure.github.io/azure-sdk/general_azurecore.html#azurecore-http-telemetry-x-ms-useragent)
- [Azure SDK Distributed tracing policy](https://azure.github.io/azure-sdk/general_azurecore.html#distributed-tracing-policy)
- [Open Telemetry](https://opentelemetry.io/)

In addition to distributed tracing, Azure also uses a set of common correlation headers:

|Name                         |Applies to|Description|
|-----------------------------|----------|-----------|
|x-ms-client-request-id       |Both      |Optional. Caller-specified value identifying the request, in the form of a GUID with no decoration such as curly braces (e.g. `x-ms-client-request-id: 9C4D50EE-2D56-4CD3-8152-34347DC9F2B0`). If the caller provides this header the service **must** include this in their log entries to facilitate correlation of log entries for a single request. Because this header can be client-generated, it should not be assumed to be unique by the service implementation.
|x-ms-request-id              |Response  |Required. Service generated correlation id identifying the request, in the form of a GUID with no decoration such as curly braces. In contrast to the the `x-ms-client-request-id`, the service **must** ensure that this value is globally unique. Services should log this value with their traces to facilitate correlation of log entries for a single request.

## Final thoughts
These guidelines describe the upfront design considerations, technology building blocks, and common patterns that Azure teams encounter when building an API for their service. There is a great deal of information in them that can be difficult to follow. Fortunately, at Microsoft, there is a team committed to ensuring your success.

The Azure REST API Stewardship board is a collection of dedicated architects that are passionate about helping Azure service teams build interfaces that are intuitive, maintainable, consistent, and most importantly, delight our customers.
Because APIs affect nearly all downstream decisions, you are encouraged to reach out to the Stewardship board early in the development process.
These architects will work with you to apply these guidelines and identify any hidden pitfalls in your design. For more information on how to part with the Stewardship board, please refer to [Considerations for Service Design](./ConsiderationsForServiceDesign.md).
