# feishu-nodes-lite 配置指南 v1.2.0

## 📋 概述

**版本**: 1.2.0
**发布日期**: 2025-11-28
**适用范围**: n8n 工作流中 `智能双日报管理系统 v1.2.0` 项目的所有飞书相关节点配置。

## 🔧 安装与配置

### 1. 安装 feishu-nodes-lite

<!-- ```bash
# 进入 n8n 容器
docker exec -it n8n sh

# 创建节点目录
mkdir ~/.n8n/nodes
cd ~/.n8n/nodes

# 安装 n8n-nodes-feishu-lite
npm install n8n-nodes-feishu-lite

# 重启 n8n 容器使节点生效``` -->

### 2. 配置飞书凭据

在 n8n 中创建 Feishu Credentials API 凭据：

```json
{
  "name": "Feishu API",
  "type": "feishuCredentialsApi",
  "data": {
    "baseURL": "https://open.feishu.cn",
    "appID": "cli_xxxxxxxxxxxxxxxx",
    "appSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  }
}
```

## 🗄️ 多维表格操作配置

### 1. 查询记录 (search)

#### 基本查询配置
```json
{
  "resource": "bitable",
  "operation": "bitable:table:record:search",
  "app_token": "QXRxxxxxxxxxxxxxxxxxxxx",
  "table_id": "tblEmployeexxxxx",
  "page_size": 500,
  "user_id_type": "open_id",
  "automatic_fields": false
}
```

#### 带过滤条件的查询
```json
{
  "resource": "bitable",
  "operation": "bitable:table:record:search",
  "app_token": "QXRxxxxxxxxxxxxxxxxxxxx",
  "table_id": "tblEmployeexxxxx",
  "filter": {
    "conjunction": "and",
    "conditions": [
      {
        "field_name": "status",
        "operator": "is",
        "value": ["在职"]
      }
    ]
  },
  "page_size": 100,
  "user_id_type": "open_id"
}
```

#### 日期字段查询
```json
{
  "filter": {
    "conjunction": "and",
    "conditions": [
      {
        "field_name": "date",
        "operator": "is",
        "value": ["Today"]
      },
      {
        "field_name": "date",
        "operator": "isGreater",
        "value": ["ExactDate", "1702449755000"]
      }
    ]
  }
}
```

**支持的日期值格式：**
- `["Today"]`, `["Yesterday"]`, `["Tomorrow"]`
- `["ExactDate", "timestamp"]` (毫秒时间戳)
- `["TheLastWeek"]`, `["TheNextWeek"]`
- `["CurrentWeek"]`, `["LastWeek"]`
- `["CurrentMonth"]`, `["LastMonth"]`

### 2. 创建记录 (add)

#### 单条记录创建
```json
{
  "resource": "bitable",
  "operation": "bitable:table:record:add",
  "app_token": "QXRxxxxxxxxxxxxxxxxxxxx",
  "table_id": "tblDailyReportxx",
  "body": {
    "fields": {
      "employee_link": ["recXXXXX"],
      "date": 1702449755000,
      "work_content": "完成系统架构设计",
      "submission_type": "正常提交"
    }
  },
  "user_id_type": "open_id"
}
```

#### 字段类型格式指南

| 字段类型   | 正确格式                                      |
| :--------- | :-------------------------------------------- |
| **文本**   | `"文本内容"`                                  |
| **数字**   | `100`                                         |
| **日期**   | `1702449755000` (13位毫秒时间戳)              |
| **单选**   | `"选项值"`                                    |
| **多选**   | `["选项1", "选项2"]`                          |
| **复选框** | `true`                                        |
| **人员**   | `[{"id": "ou_xxx"}]`                          |
| **关联**   | `["recXXXXX", "recYYYYY"]`                    |
| **超链接** | `{"text": "链接文本", "link": "https://..."}` |

### 3. 批量创建记录 (batchAdd)

```json
{
  "resource": "bitable",
  "operation": "bitable:table:record:batchAdd",
  "app_token": "QXRxxxxxxxxxxxxxxxxxxxx",
  "table_id": "tblDailyReportxx",
  "body": {
    "records": [
      {
        "fields": { "work_content": "第一条记录内容" }
      },
      {
        "fields": { "work_content": "第二条记录内容" }
      }
    ]
  },
  "user_id_type": "open_id"
}
```

### 4. 更新记录 (update)

```json
{
  "resource": "bitable",
  "operation": "bitable:table:record:update",
  "app_token": "QXRxxxxxxxxxxxxxxxxxxxx",
  "table_id": "tblMissedLogxxxx",
  "record_id": "recXXXXX",
  "body": {
    "fields": {
      "status": "已补交"
    }
  },
  "user_id_type": "open_id"
}
```

### 5. 批量更新记录 (batchUpdate)

```json
{
  "resource": "bitable",
  "operation": "bitable:table:record:batchUpdate",
  "app_token": "QXRxxxxxxxxxxxxxxxxxxxx",
  "table_id": "tblMissedLogxxxx",
  "body": {
    "records": [
      { "record_id": "recXXXXX", "fields": { "status": "已补交" } },
      { "record_id": "recYYYYY", "fields": { "status": "已豁免" } }
    ]
  },
  "user_id_type": "open_id"
}
```

## 💬 消息发送配置

### 1. 发送文本消息

```json
{
  "resource": "message",
  "operation": "message:send",
  "receive_id_type": "open_id",
  "receive_id": "ou_xxx",
  "msg_type": "text",
  "content": "{\"text\":\"请记得提交今日日报\"}"
}
```

### 2. 发送卡片消息

```json
{
  "resource": "message",
  "operation": "message:send",
  "receive_id_type": "open_id",
  "receive_id": "ou_xxx",
  "msg_type": "interactive",
  "content": {
    "type": "template",
    "data": { "template_id": "ctp_xxx" }
  }
}
```

## 📋 最佳实践检查清单 (v1.2.0)

### 节点配置检查
- [ ] **操作名**: 使用 `search`, `get`, `batchAdd`, `batchUpdate`, `update` 等标准操作名。
- [ ] **字段ID**: 字段名称使用英文ID，而非中文显示名。
- [ ] **日期格式**: 日期字段使用纯数字13位毫秒时间戳。
- [ ] **关联格式**: 关联字段使用 `["recXXXXX"]` 格式。
- [ ] **人员格式**: 人员字段使用 `[{"id": "ou_xxx"}]` 格式。

### 性能优化检查
- [ ] **限定字段**: 查询使用 `field_names` 限制返回字段。
- [ ] **批量操作**: 优先使用 `batchAdd`/`batchUpdate`。
- [ ] **分页大小**: 合理设置 `page_size` (建议 50-500)。

### 错误处理检查
- [ ] **开启重试**: 关键节点启用 `Retry On Fail`。
- [ ] **设置次数**: 设置合理的重试次数 (建议3次)。
- [ ] **全局捕获**: 配置工作流级别的 `Error Trigger` 用于告警。

---

## 📝 版本修订历史

*   **v1.2.0 (2025-11-28)**
    *   文档版本号与`智能双日报管理系统 v1.2.0`同步。
    *   在“多维表格操作配置”部分，补充了 `get` 操作的配置示例。
    *   更新了“最佳实践检查清单”中的操作名，增加了 `get` 和 `update`。
*   **v1.0.0 (2025-11-09)**
    *   项目首个稳定版本发布。
    *   本文档作为 `智能双日报管理系统 v1.0.0` 的一部分，统一了版本号，并固化了经过验证的节点配置规范。
