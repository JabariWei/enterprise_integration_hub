# 第十一部分：企业协作 API 中台完整 OpenAPI 设计

这一部分开始定义**对外 API 契约**。

目标：

任何系统：

- CRM
- ERP
- AI Agent
- OA
- RPA
- 自研业务系统

只需要调用这一套 API。

不需要知道：

- 企业微信 API
- 飞书 API
- 钉钉 API

---

# 1. API 设计原则

## 1.1 不暴露 Provider


错误：

```
POST /feishu/send_message
POST /wecom/send_message
POST /dingtalk/send_message
```


正确：

```
POST /messages/send
```


---

## 1.2 API 面向能力


错误：

```
GET /get_feishu_users
```


正确：

```
GET /users
```


---

## 1.3 所有请求携带 Tenant Context


Header：

```http
X-Tenant-ID: tenant001

Authorization: Bearer xxx
```

---

# 2. API 总览


```
/api/v1


├── connectors

├── capabilities

├── users

├── departments

├── messages

├── conversations

├── resources

├── workflows

├── events

├── sync

└── webhooks

```

---

# 3. Connector 管理 API

负责：

企业绑定第三方平台。

---

# 3.1 查询 Connector


## Request


```
GET

/api/v1/connectors
```


---

Response：

```json
[
 {
   "id":"conn001",

   "provider":"feishu",

   "name":"生产飞书",

   "status":"active"
 }
]
```


---

# 3.2 创建 Connector


## Request


```
POST

/api/v1/connectors
```


Body：

```json
{
"provider":"feishu",

"name":"公司飞书",

"credential":{

"app_id":"cli_xxx",

"app_secret":"xxx"

}

}
```

---

Response：

```json
{
"id":"conn001",

"status":"connected"

}
```

---

# 3.3 Connector 健康检查


```
GET

/api/v1/connectors/{id}/health
```


返回：

```json
{
"status":"ok",

"latency":120,

"token_expire":"2026-08-20"

}
```

---

# 4. Capability API

能力查询。


---

# 4.1 查询平台能力


```
GET

/api/v1/connectors/{id}/capabilities
```


返回：

```json
{
"provider":"feishu",

"capabilities":[


{
"name":

"user.read",

"enabled":true

},


{
"name":

"document.create",

"enabled":true

}

]

}
```

---

# 4.2 Capability Schema


```
GET

/api/v1/capabilities/{name}
```


例如：

```
message.send
```


返回：

```json
{

"name":

"message.send",


"input_schema":{


"type":"object",

"properties":{

"receiver":{

"type":"string"

}

}

}

}
```

---

# 5. User 用户 API

---

# 5.1 获取用户列表


```
GET

/api/v1/users
```


参数：

```
department_id

page

size

keyword

```


Response：

```json
{
"items":[

{

"id":"user001",

"name":"张三",

"mobile":"138xxx",

"departments":[

"销售部"

],

"identities":[

{

"provider":"feishu",

"id":"ou_xxx"

}

]

}

]

}
```

---

# 5.2 查询用户详情


```
GET

/api/v1/users/{id}
```


---

# 5.3 同步用户


手动触发同步。


```
POST

/api/v1/users/sync
```


Body：

```json
{

"connector_id":"conn001"

}
```

Response：

```json
{

"job_id":"sync001",

"status":"running"

}
```

---

# 6. Department API


## 获取组织树


```
GET

/api/v1/departments/tree
```


返回：

```json
{
"id":"root",

"name":"公司",

"children":[

{

"name":"研发部"

}

]

}
```

---

# 7. Message API


这是最高频接口。


---

# 7.1 发送消息


```
POST

/api/v1/messages/send
```


Request：


```json
{

"connector_id":

"conn001",


"receiver":{


"type":"user",

"id":"user001"

},


"message":{


"type":"text",

"content":{

"text":"您好"

}

}

}
```

---

Response：

```json
{

"id":"msg001",

"status":"sent",

"provider_message_id":

"om_xxx"

}
```

---

# 7.2 批量发送


```
POST

/api/v1/messages/batch-send
```


Request：

```json
{

"receivers":[

"user001",

"user002"

],


"message":{}

}
```

---

# 7.3 发送卡片消息


统一模型：

```json
{

"type":"card",

"content":{

"title":"审批通知",

"buttons":[]

}

}
```

---

底层转换：

飞书：

Interactive Card


企业微信：

Template Card


钉钉：

ActionCard

---

# 8. Conversation API


## 获取群列表


```
GET

/api/v1/conversations
```


返回：

```json
[
{

"id":"conv001",

"type":"group",

"name":"销售群"

}

]
```

---

# 9. Resource API


解决：

文件、文档、多维表格。


---

# 9.1 查询资源


```
GET

/api/v1/resources
```


参数：

```
type=document

type=file

type=database
```


---

Response：

```json
[
{

"id":"res001",

"type":"document",

"name":"销售政策",

"provider":"feishu"

}

]
```

---

# 9.2 创建资源


```
POST

/api/v1/resources
```


Request：

```json
{

"type":"document",

"title":"会议纪要",

"content":"xxxx"

}
```

---

# 10. Workflow API


统一审批。


---

# 查询审批


```
GET

/api/v1/workflows
```


---

启动审批：

```
POST

/api/v1/workflows/start
```


Request：

```json
{

"type":"approval",

"template_id":"xxx",

"data":{

}

}
```

---

# 11. Event API


查看事件。


---

```
GET

/api/v1/events
```


返回：

```json
[
{

"type":

"message.received",

"source":

"feishu",

"time":""

}

]
```

---

# 12. Webhook API


第三方平台回调入口。


统一：

```
POST

/api/v1/webhooks/{provider}
```


例如：

飞书：

```
POST

/webhooks/feishu
```


企业微信：

```
POST

/webhooks/wecom
```


钉钉：

```
POST

/webhooks/dingtalk
```

---

处理流程：

```
Provider Event


↓

Webhook Controller


↓

Signature Verify


↓

Normalize Event


↓

Event Bus


↓

Subscriber
```

---

# 13. Sync API


同步任务。


---

创建同步：

```
POST

/api/v1/sync/jobs
```


Request：

```json
{
"type":"users",

"connector_id":"conn001"

}
```

---

查询：

```
GET

/api/v1/sync/jobs/{id}
```


Response：

```json
{
"status":"completed",

"total":1000,

"success":998

}
```

---

# 14. 错误码设计


统一：

```json
{
"code":

"CONNECTOR_ERROR",


"message":

"飞书token失效",


"provider":

"feishu"

}
```

---

错误分类：

|Code|说明|
|-|-|
|AUTH_FAILED|认证失败|
|TOKEN_EXPIRED|Token过期|
|PERMISSION_DENIED|权限不足|
|RATE_LIMIT|限流|
|PROVIDER_ERROR|厂商错误|
|INVALID_REQUEST|参数错误|

---

# 15. 权限设计


外部调用：

使用 API Key。


模型：

```
Application

 |

API Key

 |

Permission Scope

 |

Capability
```


例如：

CRM：

只能：

```
user.read

message.send
```


AI Agent：

可以：

```
document.read

workflow.start
```

---

# 16. OpenAPI Tags 设计


FastAPI：

最终：

```yaml
tags:

- Connector

- Capability

- Identity

- Communication

- Resource

- Workflow

- Event

```

---

# 17. MCP 映射设计


未来自动生成 MCP。


API：

```
POST /messages/send
```


自动生成 Tool：

```json
{
"name":

"send_message",

"description":

"发送企业消息"

}
```

---

# 18. 当前 API 第一版范围


MVP：

必须实现：

## Connector

```
创建连接
查询状态
查询能力
```


## Identity

```
users

departments
```


## Communication

```
send message
webhook
```


## Sync

```
同步任务
```

---
