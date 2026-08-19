# 第十三部分：企业协作 API 中台后端开发规范（FastAPI）

目标：

形成可以直接交给开发人员 / Codex 执行的后端开发规范。

本阶段假设：

- Python 3.12+
- FastAPI
- PostgreSQL
- Redis
- Docker
- Async IO
- 插件式 Connector 架构

---

# 1. 后端整体架构规范

采用：

> Modular Monolith（模块化单体）

而不是一开始微服务。


原因：

这个项目核心复杂度在：

- Connector
- 数据模型
- 能力抽象

不是服务数量。


---

整体：

```text
backend/


app/

├── main.py

├── api/

├── core/

├── models/

├── schemas/

├── repositories/

├── services/

├── connectors/

├── workers/

├── events/

├── sync/

└── utils/

```

---

# 2. 分层架构


严格分：

```text
API Layer

    ↓

Service Layer

    ↓

Repository Layer

    ↓

Database


Connector Layer

    ↓

External Provider API

```

---

不要：

Router 直接调用：

```python
db.query()
```

错误：

```python
@router.get("/users")
def users():

    db.execute(
       "select..."
    )
```

---

正确：

```python
@router.get("/users")
async def users():

    return await user_service.list()
```

---

# 3. 项目目录详细设计


```text
app/


├── api
│
│   └── v1
│       ├── connectors.py
│       ├── users.py
│       ├── messages.py
│       ├── resources.py
│       └── webhooks.py


├── core

│   ├── config.py
│   ├── database.py
│   ├── security.py
│   ├── exceptions.py


├── models

│   ├── tenant.py
│   ├── user.py
│   ├── connector.py


├── schemas

│   ├── user.py
│   ├── message.py


├── repositories

│   ├── user_repository.py
│   └── connector_repository.py


├── services

│   ├── user_service.py
│   ├── message_service.py
│   └── sync_service.py


├── connectors


│   ├── base.py
│   ├── registry.py
│
│   ├── feishu
│   ├── wecom
│   └── dingtalk


├── workers

│   ├── sync_worker.py


├── events

│   ├── bus.py
│   └── handlers.py


└── utils

```

---

# 4. 配置管理


使用：

```text
pydantic-settings
```


---

配置：

```python
class Settings:

    APP_NAME="Enterprise Hub"


    DATABASE_URL=""


    REDIS_URL=""


    SECRET_KEY=""

```

---

环境：

```text
.env

.env.production

.env.test
```

---

# 5. 数据库规范


使用：

SQLAlchemy 2.x


---

Model:

```python
class User(Base):

    __tablename__="users"


    id=Mapped[UUID]


    name=Mapped[str]


    tenant_id=Mapped[UUID]

```

---

禁止：

业务代码直接操作 Model。

---

必须：

Repository。


---

# 6. Repository 规范


例如：

User Repository:


```python
class UserRepository:


    async def get_by_id(
        self,
        id
    ):

        ...


    async def save(
        self,
        user
    ):

        ...

```

---

职责：

只负责：

- CRUD
- 查询


不负责：

业务逻辑。

---

# 7. Service 规范


例如：

UserService。


```python
class UserService:


    def __init__(
        self,
        repo,
        connector_registry
    ):

        self.repo=repo

        self.registry=registry



    async def sync_users(
        self,
        connector_id
    ):


        connector = (
            self.registry
            .get(connector_id)
        )


        users = await connector.execute(

            "user.read",

            {}

        )


        return await self.save(users)

```

---

Service 负责：

- 业务流程
- Connector 调用
- 数据转换

---

# 8. Pydantic Schema 规范


例如：

User Response:


```python
class UserResponse(
    BaseModel
):

    id:str

    name:str

    mobile:str | None

```

---

禁止：

直接返回 ORM。


---

# 9. API Router 规范


示例：

users.py


```python
router = APIRouter(
prefix="/users"
)



@router.get("")
async def list_users(

    service:
    UserService=Depends()

):

    return await service.list()

```

---

Router 只做：

- 参数校验
- 权限检查
- 调 service

---

# 10. Connector SDK 编码规范


这是项目核心。


---

Base Connector:


```python
class BaseConnector:


    provider:str


    async def execute(

        self,

        capability:str,

        params:dict

    ):

        raise NotImplementedError

```

---

所有平台：

必须实现。


---

# 11. Connector 结构规范


例如：

feishu:


```text
connectors/feishu/


connector.py


client.py


services/


    user.py

    message.py


mapper/


    user.py

```

---

# 12. Provider API Client


统一：

```python
class ProviderClient:


    async def request(

        method,

        url,

        params=None

    ):

        ...

```

---

封装：

- token
- retry
- timeout
- error


---

# 13. Token 管理规范


流程：


```text
Service


 ↓


Connector


 ↓


Token Manager


 ↓


Redis


 ↓


Provider API

```

---

TokenManager:


```python
class TokenManager:


    async def get_token(
        connector_id
    ):

        ...


```

---

# 14. 外部 API 调用规范


所有：

httpx async。


禁止：

requests。


---

必须：

```python
timeout=10

retry=3

```

---

# 15. 异常体系


统一异常：

```python
class IntegrationException(Exception):


    provider:str


    code:str


    message:str

```

---

例如：

飞书：

```python
raise IntegrationException(

provider="feishu",

code="TOKEN_EXPIRED"

)
```

---

转换：

API：

```json
{
"code":

"TOKEN_EXPIRED",

"message":

"Token expired"

}
```

---

# 16. 日志规范


使用：

structlog


日志必须包含：

```json
{
"request_id":"",

"tenant_id":"",

"connector":"feishu",

"capability":"message.send"

}
```

---

禁止：

打印：

```text
secret

token

password
```

---

# 17. 请求链路追踪


每次 API：

生成：

```text
request_id
```


贯穿：

```text
API

↓

Service

↓

Connector

↓

Provider API

```

---

# 18. Sync Worker 规范


同步任务：

不要阻塞 API。


流程：

```text
API

 |

Create Sync Job

 |

Redis Queue

 |

Worker

 |

Connector

 |

Database

```

---

例如：

```python
@task

async def sync_user_job():

    await sync_service.run()

```

---

# 19. Webhook 开发规范


流程：


```text
Provider

 |

Webhook Router

 |

Signature Verify

 |

Normalize

 |

Event Bus

 |

Handler

```

---

Handler:

```python
class UserCreatedHandler:


    async def handle(
        event
    ):

        ...

```

---

# 20. Event Bus 规范


第一阶段：

Redis Stream。


Event:

```json
{
"type":

"user.created",

"tenant_id":"",

"payload":{}

}
```

---

# 21. 测试规范


测试分层：

---

## Unit Test


测试：

- Mapper
- Service


---

## Connector Test


必须：

每个平台：

```text
test_auth

test_user_sync

test_message_send

```

---

## Integration Test


Docker:

启动：

- PostgreSQL
- Redis

---

# 22. Docker 部署


MVP：


```yaml
services:


 api:

   image: enterprise-api


 postgres:

   image: postgres


 redis:

   image: redis


 worker:

   image: enterprise-worker

```

---

# 23. CI/CD


建议：

GitHub Actions。


流程：

```text
push


↓

lint

↓

test

↓

build docker

↓

deploy
```

---

# 24. 代码质量规范


工具：

## Ruff

代码检查


## Black

格式


## MyPy

类型检查


## Pytest

测试


---

# 25. 安全规范


必须：

## Secret

AES 加密。


## API Key

Hash 保存。


## RBAC

权限：

```text
Tenant Admin

Developer

Application

```

---

# 26. MVP 开发顺序（最终）


## Sprint 1

基础：

- FastAPI
- PostgreSQL
- Tenant
- Connector


---

## Sprint 2

Connector SDK：

- BaseConnector
- Registry
- Manifest


---

## Sprint 3

Feishu Connector：

- Auth
- User
- Department
- Message


---

## Sprint 4

WeCom Connector


---

## Sprint 5

Sync + Webhook


---

# 当前阶段成果


现在已经形成：

|层|状态|
|-|-|
|产品定位|✅|
|厂商能力分析|✅|
|统一模型|✅|
|API设计|✅|
|数据库设计|✅|
|后端规范|✅|
|Connector规范|✅|


---

