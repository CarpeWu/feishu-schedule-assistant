# **智能日程与效能管理系统 - API配置文档 v1.3.0**

## 🏗️ 系统架构概览

```
┌──────────────────┐                    ┌──────────────────┐     ┌──────────────────┐
│   n8n Server     │◄───(Events)────►│   Lark Feishu    │     │  Database        │
│   (Docker)       │                    │   API Gateway    │◄───►│  Bitable         │
│   DeepSeek V3    │────(AI Analysis)──►│   OpenRouter     │     │  (T1-T7 Tables)  │
└──────────────────┘                    └──────────────────┘     └──────────────────┘
```
## 🔑 认证信息

### n8n 认证

| **项目**       | **值**                                                       | **用途**         |
| :------------- | :----------------------------------------------------------- | :--------------- |
| **服务器地址** | `http://127.0.0.1:5678`                                 | n8n管理界面和API |
| **API密钥**    | `N8N_API_KEY_PLACEHOLDER` | API访问认证      |
| **API调用头**  | `X-N8N-API-KEY: {key}`                                       | API请求头格式    |

### 飞书 Feishu 认证

| **项目**                    | **值**                                                       | **用途**        |
| :-------------------------- | :----------------------------------------------------------- | :-------------- |
| **App ID**                  | `APP_ID_PLACEHOLDER`                                       | 应用标识        |
| **App Secret**              | `APP_SECRET_PLACEHOLDER`                           | 应用密钥        |
| **Tenant Access Token API** | `https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal` | 获取访问令牌    |
| **权限状态**                | 读取✅ 写入✅                                                  | 当前API权限状态 |

### OpenRouter (AI) 认证

| **项目** | **值** | **用途** |
| :--- | :--- | :--- |
| **Base URL** | `https://openrouter.ai/api/v1` | AI 模型接口 |
| **Model** | `deepseek/deepseek-chat-v3.1` | DeepSeek V3 模型 |

## 🗄️ 数据库配置 (Lark Bitable)

### Base App 信息

| **项目**           | **值**                        | **说明**     |
| :----------------- | :---------------------------- | :----------- |
| **Base App Token** | `APP_TOKEN_PLACEHOLDER` | Base应用标识 |
| **Base名称**       | 智能日程与效能管理系统            | 主要数据存储 |

### 数据表结构 (v1.3.0 架构)

| **表ID (TID)**              | **表名**       | **用途**                 | **关键字段 & ID (v1.3.0)**                                   |
| :-------------------------- | :------------- | :----------------------- | :----------------------------------------------------------- |
| **T1** (`TABLE_ID_PLACEHOLDER`) | 漏写记录表     | 存储漏写和补交状态       | **missed\_log\_id (主键)**, employee\_ref, missed\_date, status |
| **T2** (`TABLE_ID_PLACEHOLDER`) | 日报内容表     | 存储所有提交的日报       | **daily\_report\_id (主键)**, employee\_ref, report\_date, submission\_type |
| **T3** (`TABLE_ID_PLACEHOLDER`) | AI分析汇总表   | 存储 AI 分析结果         | **analysis\_id (主键)**, target\_id, period\_start\_date, summary |
| **T4** (`TABLE_ID_PLACEHOLDER`) | 每日排班记录表 | 过程追溯与日志           | **schedule\_log\_id (主键)**, schedule\_date, employee\_ref, status |
| **T5** (`TABLE_ID_PLACEHOLDER`) | 员工信息表     | 存储员工档案、排班、状态 | **employee\_id (主键)**, employee\_person, open\_id, status, shift\_template |
| **T6** (`TABLE_ID_PLACEHOLDER`) | 特殊日程表     | 存储请假、节假日等例外   | **special\_schedule\_id (主键)**, approval\_status (**fldWrj9ezk** - 用于变更检测) |
| **T7** (`TABLE_ID_PLACEHOLDER`) | 系统配置参数表 | 存储工作流动态参数       | **config\_id (主键)**, parameter\_name, parameter\_value, category |

## 🔧 feishu-nodes-lite 配置优化

### 推荐节点配置模式

#### 1. 认证节点配置

```json
{
  "nodeType": "feishu-nodes-lite.tenantAccessToken",
  "parameters": {
    "app_id": "APP_ID_PLACEHOLDER",
    "app_secret": "APP_SECRET_PLACEHOLDER"
  }
}```

#### 2. 数据查询节点配置 (推荐使用)

```json
{
  "nodeType": "feishu-nodes-lite.RECORD_ID_PLACEHOLDER",
  "parameters": {
    "app_token": "APP_TOKEN_PLACEHOLDER",
    "table_id": "tbl{{TABLE_ID}}",
    "search_criteria": {
      "filter": {
        "conjunction": "AND",
        "conditions": [
          {
            "field_name": "{{field_name}}",
            "operator": "is",
            "value": ["{{value}}"]
          }
        ]
      }
    }
  }
}
```

#### 3. 数据写入节点配置

```json
{
  "nodeType": "feishu-nodes-lite.recordsAdd",
  "parameters": {
    "app_token": "APP_TOKEN_PLACEHOLDER",
    "table_id": "tbl{{TABLE_ID}}",
    "fields": {
      "field_name": "field_value"
    }
  }
}
```

### 字段数据格式规范 (v1.3.0)

| **字段类型** | **数据格式**                              | **示例**                    | **注意事项**        |
| :----------- | :---------------------------------------- | :-------------------------- | :------------------ |
| **人员字段** | `[{ "id": "ou_xxx" }]`                    | `[{ "id": "ou_123456" }]`   | 必须使用Open ID格式 |
| **关联字段** | `["recXXXXX"]`                            | `["rec12345"]`              | 使用记录ID数组      |
| **日期字段** | 毫秒时间戳                                | `1703001600000`             | UTC时间戳格式       |
| **日期筛选** | `["Today"]` 或 `["ExactDate", timestamp]` | `["Today"]`                 | 支持官方筛选格式    |
| **链接字段** | `{ "link": "url" }`                       | `{ "link": "https://xxx" }` | 链接对象格式        |
| **多选字段** | `["option1", "option2"]`                  | `["已完成", "进行中"]`      | 选项名称数组        |

## 🔧 工作流配置 (v1.3.0)

### 核心工作流列表

| **工作流ID**       | **工作流名称**                 | **触发方式**        | **执行频率/条件**                |
| :----------------- | :----------------------------- | :------------------ | :------------------------------- |
| `LpwfY8rzaH8piuJp` | [WF-01] 排班计算引擎模块       | 定时触发            | 每日 **20:00**                   |
| `zelirAnE4ULr7pgD` | [WF-02] 提醒与通知服务模块     | 定时触发            | 每日 21:00 / 22:00 / 23:00       |
| `pUiN0mvdIN3RYKp0` | [WF-03] 漏写判定与记录服务模块 | 定时触发            | **次日 08:00**                   |
| `V0gr95ff9FRFy6qe` | [WF-04] 统一提交处理模块-event | Webhook (Event)     | **实时** (监听 T2, T5, T6 变更)  |
| `pO5VWnXk5enPhzvH` | [WF-05] AI分析服务模块         | 定时触发            | 每日 09:00 (代码控制**周一/1号**执行) |
| `52o1x15zLjQbazlB` | [WF-06] 周五赐休提醒服务       | 定时触发            | 每周五 19:00                     |

### 时区配置状态

*   **目标时区**: `Asia/Shanghai` (东八区)
*   **配置方式**: 在 n8n 的 `Schedule Trigger` 节点中明确设置 "timezone": "Asia/Shanghai"

## 🌐 API 端点

### n8n Webhook (WF-04)

| **用途** | **方法** | **URL Path (UUID)** | **说明** |
| :--- | :--- | :--- | :--- |
| **飞书事件订阅地址** | POST | `/webhook/WEBHOOK_UUID_PLACEHOLDER_2` | 需在飞书开放平台配置此地址，并订阅 `bitable.record.changed` 事件 |

### n8n API (Legacy)

| **端点**                 | **方法** | **用途**                 |
| :----------------------- | :------- | :----------------------- |
| `/api/v1/workflows`      | GET      | 获取工作流列表           |
| `/api/v1/workflows/{id}` | GET/PUT  | 获取/更新特定工作流      |

### 飞书 API (feishu-nodes-lite 优化)

| **推荐节点**                           | **用途**     | **替代传统API** |
| :------------------------------------- | :----------- | :-------------- |
| `feishu-nodes-lite.RECORD_ID_PLACEHOLDER`      | 查询记录     | GET /records    |
| `feishu-nodes-lite.recordsAdd`         | 添加记录     | POST /records   |
| `feishu-nodes-lite.RECORD_ID_PLACEHOLDER`      | 更新记录     | PUT /records    |
| `feishu-nodes-lite.RECORD_ID_PLACEHOLDER` | 批量更新记录 | PATCH /records  |
| `feishu-nodes-lite.messagesSend`       | 发送消息     | POST /messages  |

## 📊 系统配置参数 (T7 表核心)

| **参数名 (parameter\_name)** | **建议值 (parameter\_value)** | **描述 (description)**                 |
| :--------------------------- | :---------------------------- | :------------------------------------- |
| `daily_reminder_time`        | `21:00`                       | [WF-02] 每日提醒开始时间 (HH:mm)       |
| `reminder_interval_minutes`  | `60`                          | [WF-02] 提醒循环间隔 (分钟)            |
| `missed_deadline_time`       | `08:00`                       | [WF-03] 漏写判定截止时间 (HH:mm)       |
| `ai_analysis_cycle_days`     | `7`                           | [WF-05] AI 分析周期 (天)               |
| `ai_analysis_trigger_day`    | `1`                           | [WF-05] AI 分析触发日 (1=周一, 7=周日) |
| `SPECIAL_SCHEDULE_CC_IDS`    | `ou_xxx,ou_yyy`               | [WF-04] 特殊日程审批通过后的抄送对象   |

## 🚀 性能优化建议

### API 调用优化

1.  **批量操作**: 使用 `RECORD_ID_PLACEHOLDER` 节点进行批量更新
2.  **分页查询**: 设置 `page_size` 参数控制返回数据量
3.  **字段筛选**: 仅查询需要的字段，减少数据传输
4.  **缓存机制**: 在 n8n 中使用缓存节点存储频繁访问的数据

### 工作流执行优化

1.  **并行处理**: 使用 `Split in batches` 节点并行处理多条记录
2.  **错误重试**: 配置适当的重试机制和错误处理
3.  **资源监控**: 监控工作流执行时间和资源消耗

## 🛠️ 错误处理指南

### 常见错误类型

| **错误代码**       | **可能原因** | **解决方案**                     |
| :----------------- | :----------- | :------------------------------- |
| `400 Bad Request`  | 字段格式错误 | 检查字段数据格式是否符合规范     |
| `401 Unauthorized` | 认证失败     | 检查App ID/Secret或Token是否有效 |
| `403 Forbidden`    | 权限不足     | 确认应用具有相应的读写权限       |
| `404 Not Found`    | 资源不存在   | 检查Base Token或Table ID是否正确 |
| `429 Rate Limited` | 调用频率超限 | 降低API调用频率，考虑批量操作    |

### 调试建议

1.  **启用详细日志**: 在开发阶段启用完整的执行日志
2.  **测试节点**: 使用测试功能验证单个节点配置
3.  **监控面板**: 利用 n8n 的执行监控面板分析问题

---

**文档版本**: 1.3.0

**最后更新**: 2025-12-11

**维护人员**: CarpeWu

**主要更新内容**:

*   **v1.3.0 (2025-12-11)**
    *   新增 T7 配置项 `SPECIAL_SCHEDULE_CC_IDS`
    *   记录 T6 表 `approval_status` 字段 ID (fldWrj9ezk)

*   **v1.2.0 (2025-11-28)**
    *   新增 DeepSeek V3 模型支持 (via OpenRouter)
    *   更新 WF-04 为事件驱动架构 (Webhook)
    *   新增 WF-06 周五赐休提醒服务
    *   更新时区及调度配置

*   **v1.1.0 (2025-11-10)**
    *   版本号同步至 “守望者版”，内容经审查与此版本兼容，无直接修改。

*   **v1.0.0 (2025-11-09)**
    *   项目首个稳定版本发布。
