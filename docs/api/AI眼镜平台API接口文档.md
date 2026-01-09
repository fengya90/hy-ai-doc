# AI眼镜运营平台 API 接口文档

本文档用于描述 **AI眼镜运营平台** 与外部第三方系统之间的 API 交互规范。 

所有接口统一使用 **HTTP POST** 请求方式，业务数据采用 **JSON 格式**，并在传输前通过 **AES 加密** 进行安全保护。 

每个接入方（业务方）将由平台分配以下凭证信息：

- `app_id`：业务方唯一标识 
- `secret_key`：用于生成签名的密钥 
- `aes_key`：AES 加密密钥 
- `aes_iv`：AES 加密偏移量（IV）

---

##  数据加密规则

###  加密算法

- 加密方式：`AES/CBC/PKCS5Padding` 
- 编码格式：`UTF-8` 
- 加密密钥：平台分配的 `aes_key` 
- 偏移量（IV）：平台分配的 `aes_iv`

###  加密流程

1. 业务方根据接口定义生成请求数据（JSON 字符串）。 
2. 使用分配的 `aes_key` 和 `aes_iv` 对该字符串进行 AES 加密。 
3. 将加密后的 Base64 编码结果作为 `biz_data` 上报。

**示例：**

```json
{
  "biz_data": "Vv1a7f9qJ+9v1QKq..."
}
```

---

## 鉴权机制

###  请求头参数

请求头需携带以下字段：

| 字段名              | 说明                                |
| ------------------- | ----------------------------------- |
| `x-hy-ai-app-id`    | 平台分配的业务方唯一标识 (`app_id`) |
| `x-hy-ai-app-token` | 请求签名，用于身份鉴权              |

###  签名计算方式

```text
token = SHA1(biz_data + secret_key)
```

即将加密后的业务数据 `biz_data` 与业务方对应的密钥 `secret_key` 拼接后进行 **SHA1 哈希**，所得结果即为签名 `x-hy-ai-app-token`。

###  平台端校验逻辑

平台在接收到请求后，将执行以下步骤进行鉴权：

1. 根据 `x-hy-ai-app-id` 获取对应的 `secret_key`。  
2. 使用相同规则重新计算 token。  
3. 若 token 不匹配，则拒绝请求并返回鉴权失败错误信息。

---

##  通用响应结构

所有接口返回内容均为 AES 加密后的 JSON 数据，解密后结构如下：

| 字段   | 类型    | 说明                   |
| ------ | ------- | ---------------------- |
| `code` | Integer | 状态码                 |
| `msg`  | String  | 返回消息               |
| `data` | Object  | 业务数据内容（可为空） |

### 成功示例

```json
{
  "code": 2000,
  "msg": "成功",
  "data": null
}
```

### 失败示例

```json
{
  "code": 3000,
  "msg": "参数错误",
  "data": null
}
```

---

##  状态码说明

| 状态码   | 说明                         |
| -------- | ---------------------------- |
| **2000** | 成功                         |
| **3000** | 参数错误                     |
| **5000** | 服务端系统错误               |

---

##  注意事项

1. 所有请求数据均需经过 AES 加密后传输。HTTP body`格式为`{"biz_data": "Vv1a7f9qJ+9v1QKq..."}`
2. 响应数据不加密
3. 平台保留对接口字段及加密算法版本进行升级的权利，若有变更，将另行通知业务方。 

---

## 使用示例

本示例展示业务方如何完成一次完整的加密请求与响应解密过程。

---

###  示例配置

平台为业务方分配以下凭证信息：

| 参数名 | 示例值 |
|--------|--------|
| `app_id` | `demo_app_001` |
| `secret_key` | `s3cr3t@demo` |
| `aes_key` | `1234567890abcdef` |
| `aes_iv` | `abcdef1234567890` |

---

###  示例接口

**接口地址：**  `https://ip:port/glasses/ops/app/api/v1/test`

---

### 示例原始业务数据（未加密前）

```json
{"user_id": "u10086", "face_id": "face_abc_001"}
```

### AES 加密流程

使用 aes_key = 1234567890abcdef 和 aes_iv = abcdef1234567890
加密上方 JSON 字符串（UTF-8 编码），并将结果 Base64 编码。

示例加密结果：
```
A5rt5Q6TMW0laYFZJfpks58W1kTBWMr0ruWUs+rhpx9Ey62zZRU9GxvsjSE6PKGZHV0jpcXPhSQeT5vWBFD7Hg==
```

### 生成签名 token

根据文档中签名规则：
```
token = SHA1(biz_data + secret_key)
```

则：

```
token = SHA1("A5rt5Q6TMW0laYFZJfpks58W1kTBWMr0ruWUs+rhpx9Ey62zZRU9GxvsjSE6PKGZHV0jpcXPhSQeT5vWBFD7Hg==" + "s3cr3t@demo")
```

得到结果（示例）：

```
x-hy-ai-app-token: cb08f772864e933cea1502dfee6051f7e20f111e
```

### 组装最终请求
请求头：

| Header            | 值                                       |
| ----------------- | ---------------------------------------- |
| x-hy-ai-app-id    | demo_app_001                             |
| x-hy-ai-app-token | cb08f772864e933cea1502dfee6051f7e20f111e |
| Content-Type      | application/json                         |


请求体：

```
{
  "biz_data": "A5rt5Q6TMW0laYFZJfpks58W1kTBWMr0ruWUs+rhpx9Ey62zZRU9GxvsjSE6PKGZHV0jpcXPhSQeT5vWBFD7Hg=="
}
```

###  平台端返回响应

```
{"code": 2000,"msg": "成功"}
```



---

### 示例代码

```
import base64
import hashlib
import json
import requests
from Crypto.Cipher import AES

# ======== 平台分配的配置信息 ========
APP_ID = "demo_app_001"
SECRET_KEY = "s3cr3t@demo"
AES_KEY = b"1234567890abcdef"
AES_IV = b"abcdef1234567890"
URL = "https://ip:port/glasses/ops/app/api/v1/test"

# ======== AES 加密解密函数 ========
def aes_encrypt(text: str) -> str:
    """AES/CBC/PKCS5Padding 加密并返回 Base64"""
    data = text.encode("utf-8")
    pad_len = 16 - len(data) % 16
    data += bytes([pad_len]) * pad_len
    cipher = AES.new(AES_KEY, AES.MODE_CBC, AES_IV)
    encrypted = cipher.encrypt(data)
    return base64.b64encode(encrypted).decode()



# ======== 构造原始业务数据 ========
biz_obj = {"user_id": "u10086", "face_id": "face_abc_001"}
biz_json = json.dumps(biz_obj, ensure_ascii=False)
print("🔹 原始数据：", biz_json)

# ======== 1. 加密业务数据 ========
biz_data = aes_encrypt(biz_json)
print("🔹 加密数据：",biz_data)

# ======== 2. 生成签名 token ========
token_str = biz_data + SECRET_KEY
app_token = hashlib.sha1(token_str.encode("utf-8")).hexdigest()
print("🔹 请求token：",app_token)

# ======== 3. 发送请求 ========
headers = {
    "x-hy-ai-app-id": APP_ID,
    "x-hy-ai-app-token": app_token,
    "Content-Type": "application/json"
}
payload = {"biz_data": biz_data}

print("🔹 请求头：", headers)
print("🔹 请求体：", json.dumps(payload, ensure_ascii=False))

# 调用请求
# response = requests.post(URL, headers=headers, json=payload)
# resp_json = response.json()

# 这里mock一个结果
resp_json = {"code": 2000,"msg": "成功"}

print("🔹 响应：", resp_json)
```

---

##  API接口

### 设备信息上报

#### 接口地址

* `/glasses/platform/open/api/v1/device/upload` 

---

#### 


| 参数名            | 类型   | 是否必填 | 说明                                 |
| ----------------- | ------ | -------- | ------------------------------------ |
| `device_id`       | String | 否       | 设备唯一标识                           |
| `vendor`          | String | 是       | 设备厂商                               |
| `brand`           | String | 是       | 设备品牌                               |
| `model`           | String | 是       | 设备型号                               |
| `device_name`  | String | 是       | 设备名称                               |
| `cmei`            | String | 是       | 设备 CMEI                              |
| `mac`             | String | 是       | 设备 MAC 地址                          |
| `sn`              | String | 是       | 设备序列号                             |
| `firmware_ver`    | String | 是       | 固件版本号                             |
| `bind_status`     | String | 是       | 绑定状态：`bound` 或 `unbound`       |
| `action_time` | Long | 是       | 绑定/解绑时间，unix毫秒时间戳 |
| `bind_phone_number`     | String | 是       | 绑定手机号                             |

---

#### 业务数据示例

```json
{
  "device_id": "dev10001",
  "vendor": "UosTech",
  "brand": "UosBrand",
  "model": "XG-1000",
  "device_name": "Uos Smart Glasses",
  "cmei": "123456789012345",
  "mac": "00:1A:2B:3C:4D:5E",
  "sn": "SN1234567890",
  "firmware_ver": "v1.2.3",
  "bind_status": "bound",
  "action_time": 1763020573000,
  "bind_phone_number": "13800138000"
}
```



### 智能问答

#### 接口地址

* `/glasses/platform/open/api/v1/chat` 



---

#### 业务数据示例

```json
{
    "phone": "18884261256",
    "os": 2,
    "messages": [{
        "role": "user",
        "content": "您好"
    }]
}
```

返回：

```
event:[START]

event:message
data:{"type":"llmReply","data":"当然"}

event:message
data:{"type":"llmReply","data":"可以"}

event:message
data:{"type":"llmReply","data":"，"}

event:message
data:{"type":"llmReply","data":"你想"}

event:message
data:{"type":"llmReply","data":"知道"}

event:message
data:{"type":"llmReply","data":"哪个"}

event:message
data:{"type":"llmReply","data":"城市的"}

event:message
data:{"type":"llmReply","data":"天气"}

event:message
data:{"type":"llmReply","data":"情况"}

event:message
data:{"type":"llmReply","data":"？"}

event:message
data:{"type":"llmReply","data":""}

event:[DONE]

```



### 获取token

#### 接口地址

* `/glasses/platform/open/api/v1/getToken` 

---

#### 参数列表

无参数，可以post空或者`{}`,即构建biz_data时用空字符串或者`{}`



#### 返回结果

| 参数名              | 类型   |  说明                           |
| ------------------- | ------ | ------------------------------ |
| `token`             | String        | token                          |
| `expireTime`        | Number        | 过期时间                   |


返回示例

```
{
    "code": 2000,
    "msg": "SUCCESS",
    "data": {
        "token": "17879c343a1a41c5b64c9cf573f35b9d",
        "expireTime": 1764882918
    }
}
```

