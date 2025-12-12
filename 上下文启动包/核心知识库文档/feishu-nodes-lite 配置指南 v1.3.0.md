# feishu-nodes-lite 配置指南 v1.3.0

## 📋 概述

**版本**: 1.3.0
**发布日期**: 2025-12-11
**适用范围**: n8n 工作流中 `智能日程与效能管理系统 v1.3.0` 项目的所有飞书相关节点配置。

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
    "appID": "APP_ID_PLACEHOLDER",
    "appSecret": "APP_SECRET_PLACEHOLDER"
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
  "app_token": "APP_TOKEN_PLACEHOLDER",
  "table_id": "TABLE_ID_PLACEHOLDER",
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
  "app_token": "APP_TOKEN_PLACEHOLDER",
  "table_id": "TABLE_ID_PLACEHOLDER",
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
  "app_token": "APP_TOKEN_PLACEHOLDER",
  "table_id": "TABLE_ID_PLACEHOLDER",
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
  "app_token": "APP_TOKEN_PLACEHOLDER",
  "table_id": "TABLE_ID_PLACEHOLDER",
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
  "app_token": "APP_TOKEN_PLACEHOLDER",
  "table_id": "TABLE_ID_PLACEHOLDER",
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
  "app_token": "APP_TOKEN_PLACEHOLDER",
  "table_id": "TABLE_ID_PLACEHOLDER",
  "body": {
    "records": [
      { "record_id": "recXXXXX", "fields": { "status": "已补交" } },
      { "record_id": "recYYYYY", "fields": { "status": "已豁免" } }
    ]
  },
  "user_id_type": "open_id"
}
```

## 🚀 高级模式最佳实践 (v1.3.0 新增)

### 1. Switch 路由配置

用于 WF-04 等需要根据 `table_id` 分发不同业务逻辑的场景。

*   **Node Type**: Switch
*   **Mode**: Rules
*   **Data Type**: String
*   **Rule 1 (T2:日报表)**:
    *   Value 1: `{{ $json.body.event.table_id }}`
    *   Operator: `Equal`
    *   Value 2: `TABLE_ID_PLACEHOLDER` (T2 Table ID)
*   **Rule 2 (T5:员工表)**:
    *   Value 1: `{{ $json.body.event.table_id }}`
    *   Operator: `Equal`
    *   Value 2: `TABLE_ID_PLACEHOLDER` (T5 Table ID)
*   **Rule 3 (T6:特殊日程表)**:
    *   Value 1: `{{ $json.body.event.table_id }}`
    *   Operator: `Equal`
    *   Value 2: `TABLE_ID_PLACEHOLDER` (T6 Table ID)

### 2. Code 节点：API 降噪/变更检测 (标准范式)

用于节省 API 调用次数，仅在关键字段真正变化时才继续执行。必须开启 `Run Once for All Items`。

```javascript
// Mode: Run Once for All Items
const targetFieldId = 'fldWrj9ezk'; // 需监控的字段ID (例如 approval_status)
const validValue = '已批准'; // 目标状态

const items = $input.all();
const changedItems = [];

for (const item of items) {
  const event = item.json.body.event;
  if (!event || !event.action_list) continue;

  const actionList = event.action_list;
  // 遍历所有变更动作，寻找目标字段的变更
  const statusChange = actionList.find(action => action.field_identity === targetFieldId);

  if (statusChange) {
    // 检查变更后的值是否符合预期
    // 注意：action.after_value 通常是 raw value
    if (statusChange.after_value && statusChange.after_value.includes(validValue)) {
      changedItems.push(item);
    }
  }
}

return changedItems; // 返回符合条件的项，若为空数组则流程自动停止
```

### 3. Code 节点：T7 配置读取与 Fan Out (分裂)

用于将逗号分隔的配置字符串（如 `ou_a,ou_b`）分裂为多条消息发送任务。

```javascript
// Mode: Run Once for All Items
// 假设上游节点 'Get Config' 返回了 parameter_value = "ou_xxx,ou_yyy"
const configItem = $("Get Config").first();
const ccIdsStr = configItem.json.parameter_value; 

if (!ccIdsStr) return [];

const ccIds = ccIdsStr.split(',').map(id => id.trim()).filter(id => id);
const originalEvent = $("Webhook").first().json; // 保留原始事件上下文

// 将每个 ID 映射为一个独立的 Item
return ccIds.map(id => ({
  json: {
    cc_open_id: id,
    ...originalEvent // 携带原始信息
  }
}));
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

## 📋 最佳实践检查清单 (v1.3.0)

### 节点配置检查
- [ ] **操作名**: 使用 `search`, `get`, `batchAdd`, `batchUpdate`, `update` 等标准操作名。
- [ ] **字段ID**: 字段名称使用英文ID，而非中文显示名。
- [ ] **日期格式**: 日期字段使用纯数字13位毫秒时间戳。
- [ ] **关联格式**: 关联字段使用 `["recXXXXX"]` 格式。
- [ ] **人员格式**: 人员字段使用 `[{"id": "ou_xxx"}]` 格式。
- [ ] **Switch路由**: 检查 Table ID 是否配置正确。
- [ ] **Code模式**: 过滤/分裂逻辑务必使用 `Run Once for All Items`。

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

*   **v1.3.0 (2025-12-11)**
    *   **[新增]** Switch 节点配置最佳实践。
    *   **[新增]** Code 节点 API 降噪/变更检测代码模板。
    *   **[新增]** Code 节点 Fan Out 分裂代码模板。
*   **v1.2.0 (2025-11-28)**
    *   文档版本号与`智能日程与效能管理系统 v1.2.0`同步。
    *   在“多维表格操作配置”部分，补充了 `get` 操作的配置示例。
*   **v1.0.0 (2025-11-09)**
    *   项目首个稳定版本发布。
