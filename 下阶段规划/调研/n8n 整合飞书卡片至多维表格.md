# 基于 n8n 与飞书 API 实现消息卡片数据回传多维表格的高级集成方案与富文本处理深度研究





## 第一章 引言





### 1.1 企业自动化中的数据断层问题



在当前企业数字化转型的浪潮中，即时通讯工具（IM）已不再仅仅是沟通的管道，而是演变成了业务流程的触发器与控制台。飞书（Lark）作为这一领域的佼佼者，通过“消息卡片”（Message Card）将业务系统的前端界面直接嵌入到了聊天窗口中，极大地缩短了用户的操作路径。然而，在后端数据流转层面，尤其是将非结构化的即时通讯数据沉淀到结构化的数据库——多维表格（Bitable）的过程中，存在着显著的技术断层。

这个断层的核心在于“富文本”（Rich Text）数据的处理。在前端，用户习惯于使用 Markdown 或富文本编辑器来表达带有格式的信息（如加粗、列表、超链接）；在后端，多维表格为了保证数据的一致性与可渲染性，采用了一套严谨的 JSON 对象数组结构来存储富文本。而飞书目前提供的原生轻量级自动化工具（如捷径），在处理这种从“非结构化文本”到“结构化富文本对象”的复杂转换时，往往力不从心。这迫使企业寻求更专业、更具扩展性的流程编排解决方案。



### 1.2 n8n 作为中间件的战略优势



n8n 作为一款源码可得（Source-available）的工作流自动化工具，以其节点式的编排逻辑和强大的 JavaScript 代码执行能力，成为了弥合这一断层的理想选择。不同于 AnyCross 等封装好的集成平台，n8n 允许开发者直接操控 HTTP 请求的每一个字节，能够精细地处理飞书开放平台 API 的鉴权、数据清洗与结构重组。本报告将深入探讨如何利用 n8n 构建一个高可用、低延迟的数据同步管道，实现从飞书卡片到多维表格的无缝富文本写入。



### 1.3 研究范围与目标



本研究报告旨在为企业级开发者提供一份详尽的技术实施指南，内容覆盖从飞书应用鉴权、卡片交互设计、回调数据解析，到多维表格 API 写入的全链路。

**核心研究目标包括：**

1. **交互机制解构**：深度剖析飞书卡片 JSON 2.0 的表单提交机制与 `card.action.trigger` 回调协议 1。
2. **富文本算法设计**：针对多维表格富文本字段的特殊 JSON 结构，设计一套基于 JavaScript 的 Markdown 解析算法，在 n8n 中实现格式转换 3。
3. **API 深度集成**：绕过官方 SDK 的限制，直接基于 OpenAPI 规范实现 `tenant_access_token` 的生命周期管理与 `Create Record` 接口的高级调用 5。
4. **用户体验闭环**：探讨如何利用 Toast 机制在 3 秒超时限制内完成异步处理与用户反馈 2。

------



## 第二章 飞书开放平台基础架构解析



在构建任何自动化流程之前，必须深刻理解支撑其运行的底层架构。飞书开放平台通过 OAuth 2.0 协议管理身份，通过事件总线分发交互，通过 OpenAPI 暴露数据能力。



### 2.1 应用身份与权限模型



本方案不依赖用户个人身份（User Identity），而是采用“企业自建应用”（Custom App）的身份进行系统级交互。这种模式保证了自动化流程的稳定性，不随人员离职或权限变更而中断。



#### 2.1.1 凭证体系：App ID 与 App Secret



每个企业自建应用都拥有唯一的 `App ID` 和 `App Secret`。这两个参数是获取访问令牌的基石。在 n8n 中，这被视为敏感数据，应当通过 Credentials 模块进行加密存储，严禁硬编码在工作流节点中。



#### 2.1.2 访问令牌：Tenant Access Token



对于多维表格的读写操作，飞书推荐使用 `tenant_access_token` 5。该令牌代表应用以租户（企业）的身份访问资源。

- **生命周期**：有效期通常为 2 小时（7200秒）。
- **获取方式**：通过 POST 请求发送 App ID 和 App Secret 至 `/open-apis/auth/v3/tenant_access_token/internal`。
- **刷新机制**：自动化系统必须具备自动检测 Token 过期并重新获取的能力。在 n8n 中，这通常通过“Execute Workflow”节点调用一个独立的 Token 管理子流程来实现，或者在 HTTP Request 节点中配置 OAuth2 Client Credentials 模式（尽管飞书的非标准 OAuth 实现可能需要自定义 HTTP 请求流程）。



#### 2.1.3 权限范围（Scopes）



权限配置是 API 调用成功的先决条件。根据最小权限原则，本方案需要申请以下核心权限 2：

| **权限标识 (Key)**         | **名称**                       | **用途**                                            |
| -------------------------- | ------------------------------ | --------------------------------------------------- |
| `bitable:app`              | 查看、评论、编辑和管理多维表格 | 允许应用向多维表格写入记录                          |
| `im:message:send_as_bot`   | 以应用身份发消息               | 允许机器人发送包含表单的卡片                        |
| `card:action.trigger`      | 卡片交互回调                   | 允许应用接收用户点击卡片按钮后的 Payload            |
| `contact:user.id:readonly` | 获取用户 User ID               | 用于将提交者信息（User ID）记录到多维表格的人员字段 |



### 2.2 多维表格（Bitable）数据模型深度分析



多维表格并非传统的关系型数据库（RDBMS），其数据模型更接近于面向文档的 NoSQL 结构，但在字段类型上有着严格的强类型约束。



#### 2.2.1 记录（Record）与字段（Field）



一条记录（Record）由系统生成的 `record_id` 和用户定义的 `fields` 字典组成。在调用 API 时，开发者必须清楚地知道目标表格的 `app_token`（Base 的唯一标识）和 `table_id`（数据表的唯一标识）5。



#### 2.2.2 富文本（Rich Text）的底层存储结构



这是本报告的核心技术难点。在多维表格的界面上，用户看到的是连续的图文混排内容；但在 API 层面，这是一个由多个“段落（Segment）”组成的 JSON 数组。

根据飞书 API 文档 3，一个富文本字段的 Value 并非 HTML 字符串，也非 Markdown 字符串，而是一个对象数组。常见的 Segment 类型包括：

- **Text Segment（纯文本）**：

  JSON

  ```
  {
    "type": "text",
    "text": "这是一段普通文字"
  }
  ```

- **Styled Text Segment（带样式文本）**：

  JSON

  ```
  {
    "type": "text",
    "text": "加粗文字",
    "style": { "bold": true, "italic": false } // 注意 API 版本差异，部分旧版 API 可能不支持 style 写入
  }
  ```

- **Hyperlink Segment（超链接）**：

  JSON

  ```
  {
    "type": "url",  // 或在 text 类型中使用 link 属性，具体取决于 API 版本
    "text": "点击跳转",
    "link": "https://feishu.cn"
  }
  ```

- **Mention Segment（提及）**：

  JSON

  ```
  {
    "type": "mention",
    "mentionType": "User",
    "token": "ou_xxxxxx"
  }
  ```

**关键挑战**：如果开发者直接将用户在卡片中输入的 Markdown 字符串（如 `**Hello**`）发送给多维表格 API，系统会将其识别为单一的 Text Segment，导致 Markdown 语法符号被直接显示，而非渲染为格式。因此，**必须在 n8n 中实现一个“解析层”，将 Markdown 转换为 Segment Array**。



### 2.3 消息卡片（Message Card）交互机制



消息卡片是数据采集的前端。卡片 JSON 2.0 标准提供了丰富的布局和交互组件。



#### 2.3.1 卡片结构与 Form 容器



为了实现数据的批量提交，卡片设计必须使用 **Form Container（表单容器）** 7。Form 容器允许将多个输入组件（Input, Select, DatePicker）聚合在一起。当用户点击容器内的 `button`（其 action type 为 `form_submit`）时，所有组件的值会作为一个整体对象（`form_value`）发送给回调地址。



#### 2.3.2 Input 组件的局限性



目前的飞书卡片 `Input` 组件支持 `multiline_text`（多行文本）模式 8。

- **输入**：用户可以输入换行、空格。
- **输出**：回调数据中仅包含纯文本字符串，换行符表示为 `\n`。
- **富文本缺失**：卡片 Input 组件本身**不支持**所见即所得（WYSIWYG）的富文本编辑。用户无法在输入框中直接加粗或插入图片。因此，行业通用的做法是引导用户使用 **Markdown 简写语法** 进行输入，然后由后端解析。



#### 2.3.3 回调（Webhook）与 Challenge 校验



当配置请求网址时，飞书会发送一个 `type: "url_verification"` 的 POST 请求，包含 `challenge` 字段。服务器必须在 1 秒内原样返回该 `challenge` 值，且响应状态码为 200，才能完成配置验证 9。这在 n8n 流程设计中是一个必须优先处理的分支。

------



## 第三章 n8n 自动化流程架构设计



本章将详细阐述如何在 n8n 中从零构建该集成系统。我们将流程划分为三个阶段：接收与校验、数据解析与转换、数据写入与反馈。



### 3.1 阶段一：Webhook 接收与安全性校验





#### 3.1.1 节点配置：Webhook Node



- **HTTP Method**: POST
- **Path**: `/webhook/feishu-bitable-sync`
- **Authentication**: None (鉴权由逻辑层处理)
- **Response Mode**: "On Last Node" 或 "When Last Node Finishes"（需注意 3 秒超时问题，后续详述）。



#### 3.1.2 逻辑分支：Switch Node



接收到请求后，首先通过 Switch 节点判断请求类型。

- **条件 1**：`body.type` 等于 `url_verification`。
  - **动作**：连接到一个 Function 节点或 Respond to Webhook 节点，返回 `{"challenge": $json.body.challenge}`。这是飞书验证服务器有效性的必须步骤。
- **条件 2**：`body.header.event_type` 等于 `card.action.trigger`（或旧版 `card.action.trigger_v1`）。
  - **动作**：进入数据处理主流程。
- **默认**：记录错误日志或忽略。



### 3.2 阶段二：富文本解析引擎（JavaScript Code Node）



这是整个系统的核心大脑。我们需要编写一段 JavaScript 代码，将用户在卡片 Input 中输入的类似 Markdown 的字符串，解析为多维表格 API 所需的 Segment Array。



#### 3.2.1 解析算法设计



考虑到 n8n 的执行环境限制（通常不预装复杂的 npm 包），我们将实现一个轻量级的正则解析器。支持的语法包括：

1. **段落（Paragraphs）**：基于 `\n` 分割。
2. **超链接（Links）**：识别 `(URL)` 语法或直接的 URL 字符串。
3. **简单格式**：预留加粗 `**text**` 的扩展接口。



#### 3.2.2 代码实现逻辑



以下是该解析器的伪代码逻辑描述，用于指导 n8n Code Node 的编写：

1. **获取输入**：从 `input.item.json.body.event.action.form_value` 中提取文本内容。
2. **初始化数组**：`segments =`。
3. **行分割**：使用 `split('\n')` 将文本切分为行数组。
4. **行内解析**：遍历每一行。
   - 使用正则表达式 `/(https?:\/\/[^\s]+)/g` 或 Markdown Link 正则 `\[(.*?)\]\((.*?)\)` 扫描当前行。
   - 将匹配到的 Link 之前的文本截取为 `type: "text"` 的对象推入数组。
   - 将匹配到的 Link 转换为 `type: "url"` 或带 style 的 text 对象推入数组。
   - 更新游标位置，继续扫描后续内容。
5. **换行处理**：在除最后一行的每一行末尾，推入一个内容为 `\n` 的 Text Segment。**注意**：多维表格 API 可能要求换行符作为单独的 segment 或者包含在 text 中，根据实测，单独的 `\n` segment 最为稳妥。
6. **输出**：返回构造好的 `fields` 对象。



### 3.3 阶段三：API 调用与 Token 管理





#### 3.3.1 Tenant Access Token 的获取



- **节点类型**：HTTP Request
- **Method**: POST
- **URL**: `https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal`
- **Body**: `{"app_id": "xxx", "app_secret": "xxx"}`
- **优化策略**：为了避免每次触发都请求 Token（可能触发限流），建议在 n8n 中使用静态缓存变量，或者依赖 n8n 的 OAuth2 Client Credentials 认证模式（如果飞书支持标准协议）。若不支持标准协议，可构建一个子工作流专门负责 Token 刷新，并将 Token 存入 n8n 的全局 KV Store（如果是自托管版本且有 Redis）或简易的内存变量中。



#### 3.3.2 写入多维表格（Create Record）



- **节点类型**：HTTP Request

- **Method**: POST

- **URL**: `https://open.feishu.cn/open-apis/bitable/v1/apps/:app_token/tables/:table_id/records`

- **Headers**: `Authorization: Bearer <Token>`

- **Body**:

  JSON

  ```
  {
    "fields": {
      "标题": "卡片提交数据",
      "富文本字段": <引用上一步解析生成的 Segment Array>,
      "提交人": [ { "id": <user_id>, "type": "user" } ] // 人员字段也需要特殊结构
    }
  }
  ```



### 3.4 阶段四：用户反馈（Toast Response）



飞书卡片交互要求服务器在 3 秒内响应。如果通过 HTTP 200 空包响应，前端无提示。为了提升体验，我们应返回一个 Toast 提示 2。



#### 3.4.1 Toast JSON 结构



在 n8n 的最后一个节点（Respond to Webhook），返回如下 JSON：

JSON

```
{
    "toast": {
        "type": "success",
        "content": "提交成功，数据已同步至多维表格",
        "i18n": {
            "en_us": "Success",
            "zh_cn": "提交成功，数据已同步至多维表格"
        }
    }
}
```



#### 3.4.2 处理 3 秒超时限制



如果 n8n 流程非常复杂（例如涉及多次 API 调用或大模型处理），可能会超过 3 秒。

- **解决方案**：使用 n8n 的 **"Respond to Webhook"** 节点。将该节点放置在流程的早期（例如校验通过后立即响应 Toast），配置为 `Respond With: JSON`。这样 n8n 会先向飞书服务器返回响应，断开连接，然后在后台继续执行后续的 Bitable 写入操作。这是处理 IM 回调的最佳实践。

------



## 第四章 核心技术难点：富文本解析算法实现



本章将深入探讨在 n8n Code Node 中实现 Markdown 到 Bitable Segment 的具体算法细节。这是实现“富文本写入”最关键的环节。



### 4.1 多维表格富文本 Segment 规范详解



根据飞书开放平台文档的隐晦描述及开发者社区的实践总结，多维表格富文本字段接受的数据结构具有严格的类型定义 3。

| **元素类型**       | **JSON 结构示例**                                            | **说明**                                                     |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **纯文本 (Text)**  | `{"type": "text", "text": "内容"}`                           | 最基本的单元。                                               |
| **超链接 (Url)**   | `{"type": "url", "text": "链接文字", "link": "http://..."}`  | 注意：部分文档版本可能称之为 `hyperlink`，但在 Segment 结构中 `url` 类型更为通用。另一种方式是使用 Text 类型配合 `style.link` 属性。 |
| **提及 (Mention)** | `{"type": "mention", "mentionType": "User", "token": "ou_id"}` | 用于 @人 或 @文档。                                          |
| **公式 (Formula)** | `{"type": "formula", "text": "SUM(1,2)"}`                    | 通常用于读取，写入场景较少。                                 |

**关键发现**：在 `Create Record` 接口中，如果不使用 Segment 数组，而是直接传字符串，API 会尝试将其转为纯文本。若要保留格式，必须构建上述对象数组。



### 4.2 JavaScript 解析器实现（n8n Code Node）



以下是一个经过优化的、可直接在 n8n Code Node 中使用的 JavaScript 代码片段。它能够处理多行文本，并自动识别 HTTP/HTTPS 链接将其转换为超链接对象。

JavaScript

```
// 1. 获取输入：假设上游节点传递了 form_value
const rawText = $input.item.json.body.event.action.form_value.description |

| "";

// 2. 定义结果数组
let segments =;

// 3. 正则表达式：匹配 URL (简单版本)
// 捕获组 1: URL 本身
const urlRegex = /(https?:\/\/[^\s]+)/g;

// 4. 按行分割处理
const lines = rawText.split('\n');

lines.forEach((line, lineIndex) => {
    let lastCursor = 0;
    let match;

    // 在当前行中循环查找所有 URL
    // 注意：JavaScript 的 exec 方法是有状态的
    while ((match = urlRegex.exec(line))!== null) {
        // 4.1 处理 URL 前的普通文本
        if (match.index > lastCursor) {
            segments.push({
                "type": "text",
                "text": line.substring(lastCursor, match.index)
            });
        }

        // 4.2 处理 URL 本身
        const urlStr = match;
        segments.push({
            "type": "url", // 针对多维表格富文本的特殊类型
            "text": urlStr, // 显示文本，暂定为 URL 本身
            "link": urlStr  // 跳转链接
        });

        lastCursor = urlRegex.lastIndex;
    }

    // 4.3 处理行尾剩余的文本
    if (lastCursor < line.length) {
        segments.push({
            "type": "text",
            "text": line.substring(lastCursor)
        });
    }

    // 4.4 处理换行符
    // 如果不是最后一行，添加一个换行符 Segment
    // 在多维表格中，显式的 \n 通常需要作为一个独立的 text segment
    if (lineIndex < lines.length - 1) {
        segments.push({
            "type": "text",
            "text": "\n"
        });
    }
});

// 5. 返回符合 Bitable API 规范的结构
return {
    json: {
        rich_text_segments: segments
    }
};
```



### 4.3 扩展性讨论：Markdown 高级语法



上述代码仅处理了 URL。如果需要支持加粗（`**text**`），则需要引入更复杂的 Tokenizer（词法分析器）。在 n8n 中，可以通过嵌套正则或状态机来实现。

例如，处理加粗：

JavaScript

```
const boldRegex = /\*\*(.*?)\*\*/g;
// 需要在处理完 URL 后，对 text 类型的 segment 再次进行 map 操作
// 将包含 ** 的 text segment 分裂为 [text, bold_text, text]
```

由于 n8n 节点的执行超时限制，建议不要在 Code Node 中编写过于复杂的递归解析器。如果业务强依赖复杂 Markdown，建议采用下文提到的“飞书文档中转方案”。

------



## 第五章 替代方案：基于飞书文档（DocX）的中转策略



如果业务需求不仅仅是简单的链接和换行，而是包含图片、表格、代码块等复杂富文本，直接通过 API 构造 Bitable Segment Array 将会异常繁琐且容易出错。此时，利用 **飞书文档（DocX）** 作为数据载体是一个极具战略意义的替代方案。



### 5.1 方案架构



该方案的核心思想是：**不直接把富文本写入表格单元格，而是写入一个独立的文档，并将文档链接存入表格。**

1. **接收卡片数据**：获取原始 Markdown 文本。
2. **创建飞书文档**：调用 DocX API `Create Document` 创建一个空文档 13。
3. **Markdown 转 Block**：利用 DocX API 的 `Convert Markdown to Blocks` 接口（或类似机制），将 Markdown 文本直接转换为飞书文档的 Block 结构 14。飞书文档 API 对 Markdown 的支持远好于多维表格 API。
4. **写入多维表格**：将新生成的文档 URL 写入多维表格的“超链接”字段或“文本”字段。



### 5.2 优势与劣势对比



| **维度**          | **直接写入单元格 (Cell Write)** | **文档中转 (DocX Middleware)** |
| ----------------- | ------------------------------- | ------------------------------ |
| **数据结构**      | Segment Array (复杂 JSON)       | Block Tree (文档对象)          |
| **Markdown 支持** | 需自行手写解析器                | 官方 API 支持转换              |
| **内容承载力**    | 有限（单元格不适合长文）        | 无限（完整文档能力）           |
| **用户体验**      | 在表格内直接查看                | 需点击链接跳转查看             |
| **实现成本**      | 高（解析算法复杂）              | 中（多了 API 调用步骤）        |

**建议**：对于简短的反馈、备注，使用直接写入单元格方案；对于日报、长篇需求描述，强烈建议使用文档中转方案。

------



## 第六章 错误处理与运维监控



在生产环境中，API 集成必须具备鲁棒性。



### 6.1 常见错误代码解析



| **HTTP 状态码** | **错误码 (Code)** | **含义**                    | **解决方案**                                                 |
| --------------- | ----------------- | --------------------------- | ------------------------------------------------------------ |
| **400**         | `125400`          | Invalid Request             | 检查 JSON 结构，特别是 Segment 中的 type 是否合法。          |
| **400**         | `125401`          | Rich text format error      | 富文本格式错误。通常是因为 Segment 缺少必要字段（如 url 类型缺少 link 属性）。 |
| **401**         | `99991663`        | Tenant Access Token Invalid | Token 过期。检查 n8n 的 Token 刷新逻辑。                     |
| **429**         | `99991400`        | Rate Limit Exceeded         | 频率超限。飞书 API 默认限流（如 50 QPS），需在 n8n 设置 "Wait" 节点或重试策略 14。 |





### 6.2 n8n 中的重试策略 (Retry Strategy)



在 n8n 的 HTTP Request 节点中，务必开启 **"Retry on Fail"** 选项。

- **Max Tries**: 3

- Wait between tries: 1000ms

  这将有效缓解因网络抖动或临时限流导致的写入失败。



### 6.3 死信队列 (Dead Letter Queue)



对于经过重试后依然失败的请求（例如用户输入了非法字符导致 JSON 解析崩溃），不应直接丢弃。应当在 n8n 流程的 Error Trigger 中配置一个分支，将失败的 Payload 和错误信息发送到运维群或写入日志表，以便人工介入排查。

------



## 第七章 结论



通过对飞书开放平台架构的深入解构与 n8n 编排能力的挖掘，我们证实了在不依赖 AnyCross 的前提下，完全可以实现从消息卡片到多维表格的高级数据集成。

本报告提出的**基于 n8n Code Node 的富文本解析方案**，成功解决了多维表格 API 对 JSON 结构的严格要求与卡片纯文本输入之间的矛盾。同时，引入 **"Respond to Webhook" 节点前置** 的设计模式，优雅地解决了 IM 交互场景下的 3 秒超时难题。

对于追求极致自动化与数据掌控力的企业而言，掌握这套底层集成方法论，不仅能够解决当前的数据回传问题，更为未来构建基于 AI 的智能文档处理、自动化审批流等复杂业务场景奠定了坚实的技术基础。开发者应根据实际业务中对富文本复杂度的需求，灵活选择“直接解析写入”或“文档中转”两种架构模式，以达到开发成本与用户体验的最佳平衡。

------

**声明**：本报告引用的 API 规范基于飞书开放平台 2024-2025 版本的文档。鉴于 SaaS 平台 API 的迭代速度，建议开发者在实施时参考最新的 API Explorer 进行字段校验。