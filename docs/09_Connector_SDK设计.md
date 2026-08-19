# 第九部分：Connector SDK 详细规范 + 第一个 Feishu Connector 代码设计

这一部分开始定义整个生态的**插件标准**。

目标：

未来任何开发者都可以按照规范开发：

- connector-feishu
- connector-wecom
- connector-dingtalk
- connector-salesforce
- connector-hubspot
- connector-slack

而不修改核心系统。

---

# 1. Connector Framework 设计目标

Connector 本质：

> 将第三方系统能力转换成 Enterprise Capability。


例如：

飞书：

```
Feishu API

/user/v3/users

/message/v1/send

/drive/v1/files

        ↓

Feishu Connector

        ↓

Unified Capability

        ↓

user.read

message.send

resource.read
```

---

# 2. Connector 分层设计

一个 Connector 不应该直接写业务逻辑。

推荐五层：

```
connector-feishu


├── manifest.yaml

├── client/

│   ├── auth_client.py
│   ├── http_client.py
│
├── resources/

│   ├── user.py
│   ├── department.py
│   ├── message.py
│   └── document.py
│
├── mapper/

│   ├── user_mapper.py
│   └── message_mapper.py
│
├── webhook/

│   └── handler.py
│
└── connector.py
```

---

# 3. Connector 生命周期

统一生命周期：

```
INSTALL

↓

REGISTER

↓

CONFIGURE

↓

AUTHENTICATE

↓

ENABLE

↓

RUN

↓

SYNC

↓

DISABLE
```

---

# 4. Manifest 规范

每个 Connector 必须包含：

```
manifest.yaml
```

例如：

## Feishu

```yaml
name: feishu

display_name: 飞书

version: 1.0.0


author:
  name: xxx


authentication:

  type: app_secret


capabilities:


  - identity.user.read

  - organization.department.read

  - communication.message.send

  - resource.document.read


events:

  - user.created

  - user.updated

  - message.received


resources:


  - user

  - department

  - message

  - document

```

---

# 5. Capability 定义规范

核心思想：

不要暴露 API。

暴露能力。

错误：

```
feishu.get_user_list()
```

正确：

```
identity.user.read
```


---

Capability Schema：

```json
{
"name":

"identity.user.read",


"description":

"读取企业用户",


"input_schema":{},

"output_schema":{

"type":"array",

"items":

"User"

}

}
```

---

# 6. Action 模型

所有操作统一：

```python
Action
```

例如：

发送消息：

```json
{
"name":

"communication.message.send",


"input":

{

"receiver":"",

"content":""

}

}
```

---

执行：

```python
connector.execute(
    
    "communication.message.send",

    params

)
```

---

# 7. BaseConnector 详细设计


```python
from abc import ABC, abstractmethod


class BaseConnector(ABC):


    name = None



    @abstractmethod
    async def authenticate(
        self,
        credential
    ):
        pass



    @abstractmethod
    async def capabilities(
        self
    ):
        pass



    @abstractmethod
    async def execute(
        self,
        capability,
        params
    ):
        pass



    @abstractmethod
    async def health_check(
        self
    ):
        pass

```

---

# 8. Feishu Connector 设计


目录：

```
connector-feishu

├── connector.py

├── client

│   ├── base.py
│   └── feishu_client.py


├── services

│   ├── user_service.py
│   ├── department_service.py
│   ├── message_service.py


├── mapper

│   ├── user_mapper.py

└── webhook

```

---

# 9. Feishu API Client


统一封装：

不要业务代码直接请求。


例如：

```python
class FeishuClient:


    def __init__(
        self,
        app_id,
        secret
    ):

        self.app_id=app_id

        self.secret=secret



    async def request(
        self,
        method,
        url,
        **kwargs
    ):

        token = await self.get_token()


        headers={

        "Authorization":

        f"Bearer {token}"

        }


        return await http.request(
            method,
            url,
            headers=headers
        )

```

---

# 10. Token 管理


不能：

每次请求获取 token。


设计：

```
Token Cache

Redis

     |

tenant_id

provider

token

expire_time

```

---

流程：

```
request

 |

check redis


 |

token exists?

 |

yes

 |

call api

```

---

# 11. User Service


负责：

飞书用户 API。


```python
class FeishuUserService:


    async def list_users():

        result = await client.request(

            "GET",

            "/open-apis/contact/v3/users"

        )


        return UserMapper.to_domain(
            result
        )

```

---

# 12. Mapper 层


非常重要。


不能：

API 返回什么，数据库存什么。


例如：

飞书：

```json
{
open_id:"ou_xxx",

name:"张三"

}
```


转换：

```python
class UserMapper:


    def to_domain(
        data
    ):

        return User(

            external_id=data["open_id"],

            name=data["name"]

        )

```

---

# 13. Domain Model


核心系统只认识：

```python
User
```

而不是：

```python
FeishuUser
```

---

例如：

```python
class User:

    id:str

    name:str

    mobile:str

    identities:list

```

---

# 14. Message Service


统一：

```python
class FeishuMessageService:


    async def send(
        self,
        message
    ):


        payload = {


        "receive_id":
        message.receiver,


        "msg_type":
        message.type,


        "content":
        message.content

        }


        return await client.request(

        "POST",

        "/open-apis/im/v1/messages",

        json=payload

        )

```

---

# 15. Connector 主入口


```python
class FeishuConnector(
    BaseConnector
):


    name="feishu"



    async def capabilities(self):

        return [

        "identity.user.read",

        "organization.department.read",

        "communication.message.send"

        ]



    async def execute(
        self,
        capability,
        params
    ):


        if capability=="identity.user.read":

            return await user_service.list()


        if capability=="communication.message.send":

            return await message_service.send(
                params
            )

```

---

# 16. Connector Registry


动态加载。


例如：

```python
registry.register(
    FeishuConnector()
)

registry.register(
    WeComConnector()
)

```


---

调用：

```python
connector = registry.get(
    "feishu"
)


await connector.execute(

"user.read",

{}

)
```

---

# 17. 错误统一处理


第三方错误：

飞书：

```json
{
code:99991663
}
```

企业微信：

```json
{
errcode:40014
}
```


统一：

```python
IntegrationError

{

provider:"feishu",

code:"TOKEN_EXPIRED",

message:""

}

```

---

# 18. Retry 机制


第三方 API 不稳定。


统一：

```
Request

 |

Retry Middleware

 |

Connector

 |

Provider API
```


策略：

|错误|处理|
|-|-|
|Token过期|刷新|
|429|等待重试|
|500|重试|
|权限错误|直接失败|

---

# 19. Feishu Connector MVP 范围


第一版：

实现：

## Authentication

✅ tenant_access_token


## Organization

✅ User

✅ Department


## Message

✅ Send Text


## Event

✅ Webhook Receive


暂不：

- Doc
- Bitable
- Calendar

---

# 20. 开发完成后的效果


业务系统：

调用：

```http
POST /messages/send
```

不用知道：

- 飞书
- 企业微信
- 钉钉


中台：

自动：

```
Message API

 ↓

Capability Engine

 ↓

Connector Runtime

 ↓

Feishu Connector

 ↓

Feishu API
```

---

