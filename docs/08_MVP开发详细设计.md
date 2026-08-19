# 第八部分：企业协作 API 中台 MVP 开发详细设计

这一部分开始进入**工程落地阶段**。

目标：

设计一个可以直接交给开发团队 / Codex 开始开发的版本。

项目暂定名：

# Enterprise Integration Hub（EIH）

---

# 1. MVP 开发目标

第一阶段不追求覆盖所有企业能力。

目标：

建立稳定底座：

```
企业系统
    |
统一 API
    |
Connector Framework
    |
企业微信 / 飞书 / 钉钉
```

---

# 2. MVP 功能范围

## P0 基础框架

必须完成：

### 租户管理

支持：

- 企业
- 应用
- Connector


### Connector Framework

支持：

- Connector 注册
- Connector 加载
- Capability 声明


### 认证中心

支持：

- App Key
- Secret
- Token 管理


### 统一模型

支持：

- User
- Department
- Message
- Resource


---

## P1 企业基础能力


支持：

### 企业微信

### 飞书


能力：

|能力|说明|
|-|-|
|用户同步|员工同步|
|部门同步|组织架构|
|消息发送|单聊/群聊|
|Webhook|事件接收|


---

## P2 企业办公能力


增加：

钉钉。


能力：

|能力|
|-|
|审批|
|考勤|
|日历|
|待办|


---

## P3 开放生态


增加：

- Connector Marketplace
- MCP Server
- Agent Tool Registry


---

# 3. 技术选型


## Backend

推荐：

```
Python 3.12

FastAPI

SQLAlchemy 2

Pydantic v2

Alembic

PostgreSQL

Redis
```

---

## Worker

第一阶段：

```
Celery
```

或者：

```
ARQ
```

推荐：

ARQ。

原因：

- Python 原生
- Redis
- 轻量

---

## HTTP Client

统一：

```
httpx
```

---

## 配置

```
pydantic-settings
```

---

# 4. 项目目录设计


建议：

```
enterprise-hub/


backend/


├── app/

│
├── api/
│
│   ├── v1/
│   │
│   ├── users.py
│   ├── messages.py
│   ├── connectors.py
│
│
├── core/
│
│   ├── config.py
│   ├── security.py
│
│
├── models/
│
│   ├── tenant.py
│   ├── connector.py
│
│
├── schemas/
│
│   ├── user.py
│   ├── message.py
│
│
├── services/
│
│   ├── user_service.py
│   ├── message_service.py
│
│
├── connectors/
│
│   ├── base.py
│
│   ├── registry.py
│
│   ├── feishu/
│   │
│   ├── wecom/
│   │
│   └── dingtalk/
│
│
├── sync/
│
│
├── events/
│
│
├── workers/
│
│
└── main.py

```

---

# 5. 核心数据库设计


## 5.1 tenant


企业租户


```sql
CREATE TABLE tenant
(
id UUID PRIMARY KEY,

name VARCHAR(128),

status VARCHAR(32),

created_at TIMESTAMP

);
```

---

# 5.2 connector_instance


连接实例。


例如：

某公司绑定：

飞书应用。


```sql
CREATE TABLE connector_instance
(

id UUID PRIMARY KEY,


tenant_id UUID,


provider VARCHAR(32),


name VARCHAR(128),


status VARCHAR(32)

);
```

---

# 5.3 credential


认证信息。


```sql
CREATE TABLE credential
(

id UUID PRIMARY KEY,


connector_id UUID,


encrypted_config JSONB,


expire_time TIMESTAMP

);
```

---

# 5.4 capability


能力。


```sql
CREATE TABLE capability
(

id UUID PRIMARY KEY,


name VARCHAR(128),


description TEXT

);
```

---

例如：

数据：

```
user.read

department.read

message.send

document.create

```

---

# 5.5 connector_capability


平台能力映射。


```sql
CREATE TABLE connector_capability
(

connector_id UUID,


capability_id UUID

);
```

---

# 5.6 unified_user


统一用户。


```sql
CREATE TABLE unified_user
(

id UUID,


tenant_id UUID,


name VARCHAR(128),


mobile VARCHAR(32),


email VARCHAR(128),


status VARCHAR(32)

);
```

---

# 5.7 external_identity


外部账号。


```sql
CREATE TABLE external_identity
(

id UUID,


user_id UUID,


provider VARCHAR(32),


external_id VARCHAR(128)

);
```

---

例如：


内部：

```
user_id=001
```


绑定：

```
wecom:
zhangsan


feishu:
ou_xxx


dingtalk:
123456

```

---

# 6. Connector SDK 设计


## 6.1 BaseConnector


```python
from abc import ABC


class BaseConnector(ABC):


    name:str


    def authenticate(
        self,
        credential
    ):
        pass



    def get_capabilities(
        self
    ):
        pass



    def execute(
        self,
        action:str,
        params:dict
    ):
        pass



    def handle_event(
        self,
        event
    ):
        pass

```

---

# 7. Connector Manifest


每个 Connector 必须提供：

manifest.yaml


例如：

## feishu


```yaml
name: feishu

version: 1.0


auth:

 type: app_secret


capabilities:


 - user.read

 - department.read

 - message.send


events:


 - user.created

 - message.received

```

---

# 8. Connector Registry


动态加载。


例如：

```python
class ConnectorRegistry:


    connectors={}



    def register(
        self,
        connector
    ):

        self.connectors[
            connector.name
        ]=connector



    def get(
        self,
        name
    ):

        return self.connectors[name]

```

---

# 9. API 设计


## 9.1 查询连接器


GET:

```
/api/v1/connectors
```


返回：

```json
[
{
"name":"feishu",

"status":"active"

}
]
```

---

# 9.2 查询能力


GET:

```
/api/v1/connectors/{id}/capabilities
```


返回：

```json
[
"user.read",

"message.send"

]
```

---

# 9.3 用户查询


统一：

```
GET /api/v1/users
```


内部：

调用：

```
UserService

↓

Connector

↓

Provider API

```

---

# 9.4 消息发送


接口：

```
POST /api/v1/messages/send
```


请求：

```json
{
"provider":"feishu",


"receiver":
{
"type":"user",

"id":"001"

},


"message":
{

"type":"text",

"content":"hello"

}

}
```

---

# 10. Sync Engine


## 用户同步流程


```
Scheduler

   |

sync_users_job


   |

Connector.get_users()


   |

Mapping


   |

Unified User

```

---

代码：

```python
async def sync_users(
    connector
):

    users = await connector.get_users()


    for user in users:

        save_user(user)

```

---

# 11. Webhook 流程


统一入口：

```
POST

/api/v1/webhooks/{provider}

```


---

例如：

飞书：

```
message.receive
```


进入：

```
FeishuWebhookHandler


        |

Normalize


        |

EventBus

```

---

# 12. Event Model


统一：


```json
{
"id":"",

"type":"message.received",


"provider":"feishu",


"payload":{}

}
```

---

# 13. 第一个 Connector 实现顺序


建议：

不要先企业微信。

原因：

飞书能力最完整。


开发顺序：

## Step 1

飞书：

完成：

```
Authentication

User

Department

Message

Webhook

```

---

## Step 2

企业微信：

映射：

```
User

Department

Message

Webhook

```

---

## Step 3

钉钉：

增加：

```
Workflow

Attendance

Task

```

---

# 14. 测试策略


## Connector 测试


每个 Connector：

必须：

```
Capability Test
```

例如：

```python
test_user_query()

test_send_message()

test_webhook()
```

---

# 15. 安全设计


必须：

## Secret 加密


禁止：

数据库：

```json
{
"secret":"123456"
}
```


应该：

```
AES256

+

KMS

```

---

## API Key


外部系统：

使用：

```
integration_api_key
```


---

# 16. MVP 时间规划


## Sprint 1

基础框架：

- FastAPI
- DB
- Connector SDK
- Registry


---

## Sprint 2

飞书 Connector：

- Auth
- User
- Department
- Message


---

## Sprint 3

企业微信：

- User
- Department
- Message


---

## Sprint 4

Webhook + Sync


---

# 17. 当前阶段输出总结


已经确定：

## 架构

```
Capability Layer

↓

Connector Framework

↓

Provider Adapter

```

---

## 核心模型

```
User

Department

Message

Resource

Event

Capability

```

---

## 第一阶段目标

完成：

> 一个企业绑定飞书或者企业微信后，第三方系统可以通过统一 API 获取员工、组织架构，并发送消息。


---

