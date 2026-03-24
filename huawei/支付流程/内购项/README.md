# 华为 IAP 内购项支付流程

## 目录

1. [游戏支付流程类型](#游戏支付流程类型)
2. [商品配置](#商品配置)
3. [支付流程图](#支付流程图)
4. [服务端 API 文档](#服务端-api-文档)
5. [资损相关字段](#资损相关字段)

---

## 游戏支付流程类型

### 主动查询

华为 IAP **提供订单主动查询接口**，服务端可主动调用以下接口进行订单核查：

- **Order 服务验证购买 Token**（`/applications/purchases/tokens/verify`）：验证客户端支付结果中的购买令牌，确认支付结果的准确性。该接口只针对非订阅型商品（消耗型和非消耗型）。

### 被动回调

对于**消耗型/非消耗型商品**，华为采用"客户端获取购买结果 → 服务端验签"的方式：

- 用户完成支付后，华为 IAP SDK 通过客户端回调将 `InAppPurchaseData` 和 `dataSignature` 返回给游戏客户端。
- 游戏客户端将购买信息上报给游戏服务端，由服务端调用 `verifyToken` 接口向华为服务器验签。
- 推荐流程：客户端调用 IAP SDK 完成支付 → 客户端获取 `InAppPurchaseData` 和 `dataSignature` → 服务端调用 `verifyToken` 接口验签和核查 → 发货 → 调用确认购买接口。

---

## 商品配置

### 是否需要在华为后台配置商品

**是**，需要在 AppGallery Connect（华为开发者平台）中配置商品信息：

- 登录 [AppGallery Connect](https://developer.huawei.com/consumer/cn/service/josp/agc/index.html)，进入「我的应用」→「运营」→「产品运营」→「商品管理」。
- 需要配置并激活商品后，客户端才能拉取到商品信息并发起购买。

### 商品 ID

- 每种商品必须有**唯一的商品 ID**（`productId`），由开发者在 AppGallery Connect 中配置。
- 商品 ID 来源于 AppGallery Connect 中配置商品信息时设置的值。

### 币种与地区定价

- 定价货币的币种遵循 **ISO 4217 标准**（如 `CNY`、`USD`）。
- 华为 IAP 支持根据用户帐号服务地所在国家/地区，自动展示本地化语言和货币价格。
- 开发者只需在后台配置一套商品定价，华为 IAP 可自动按地区展示对应货币价格。

### 优惠配置

- 华为商品管理支持**营销活动配置**（如限时折扣、优惠券等），通过「管理商品营销」功能配置。

---

## 支付流程图

### 主动查询流程

客户端发起支付，服务端主动调用华为接口验证购买结果：

```mermaid
sequenceDiagram
    participant C as 客户端（游戏 App）
    participant S as 游戏服务端
    participant H as 华为 IAP 服务器

    C->>H: 1. 调用 IAP SDK 查询商品信息（obtainProductInfo）
    H-->>C: 返回商品列表（含价格、名称等）

    C->>H: 2. 调用 IAP SDK 创建订单（createPurchaseIntent）
    H-->>C: 拉起 IAP 收银台，用户完成支付

    H-->>C: 3. SDK 回调返回支付结果（InAppPurchaseData + dataSignature）

    C->>S: 4. 上报购买信息（InAppPurchaseData + dataSignature + purchaseToken）

    S->>H: 5. 调用 Order 服务验证购买 Token（verifyToken）
    H-->>S: 返回购买数据及签名（purchaseTokenData + dataSignature）

    S->>S: 6. 验签（使用 RSA IAP 公钥验证签名）

    S->>S: 7. 校验订单信息（金额、商品 ID、状态等）

    S->>C: 8. 发货（下发游戏道具/货币等）

    S->>H: 9. 调用 Order 服务确认购买（confirm）
    H-->>S: 返回确认结果

    Note over S,H: 消耗型商品必须调用确认购买，否则无法再次购买
```

### 回调流程

用户支付后，华为 IAP SDK 回调客户端，客户端将结果上报服务端处理：

```mermaid
sequenceDiagram
    participant U as 用户
    participant C as 客户端（游戏 App）
    participant S as 游戏服务端
    participant H as 华为 IAP 服务器

    U->>C: 触发购买（点击购买按钮）
    C->>H: 调用 createPurchaseIntent 拉起收银台
    U->>H: 完成支付操作

    H-->>C: SDK 回调 onActivityResult<br/>（InAppPurchaseData + dataSignature）

    C->>S: 上报 purchaseToken + InAppPurchaseData
    S->>H: verifyToken 验证购买 Token
    H-->>S: 返回验签结果

    S->>S: 校验通过，执行发货
    S->>C: 返回发货结果
    S->>H: 调用 confirm 确认消耗
```

### 订单状态流转图

```mermaid
stateDiagram-v2
    [*] --> 初始化: 创建订单（purchaseState = -1）
    初始化 --> 已购买: 用户完成支付（purchaseState = 0）
    初始化 --> 已取消: 用户取消支付（purchaseState = 1）
    已购买 --> 未消耗: consumptionState = 0
    未消耗 --> 已消耗: 调用确认购买接口（consumptionState = 1）
    已购买 --> 已退款: 退款成功（purchaseState = 2）
    初始化 --> 待处理: 待处理状态（purchaseState = 3）
```

---

## 服务端 API 文档

### 接口列表

| 接口名称 | URL | 方法 | 功能说明 |
|---------|-----|------|---------|
| 获取应用级 Access Token | `https://oauth-login.cloud.huawei.com/oauth2/v3/token` | HTTPS POST | 获取调用华为服务端 API 所需的鉴权 Token |
| Order 服务验证购买 Token | `{rootUrl}/applications/purchases/tokens/verify` | HTTPS POST | 向华为 IAP 服务器校验支付结果中的购买令牌，确认支付结果准确性。仅针对非订阅型商品。 |
| Order 服务确认购买 | `{rootUrl}/applications/v2/purchases/confirm` | HTTPS POST | 消耗型商品发货成功后，通知华为 IAP 服务器更新商品发货状态（消耗确认）。若不调用，将无法再次购买该商品。 |

> `rootUrl` 在不同站点有不同的值，请参考华为文档中的「站点信息和站点选择」。

---

### 0. 获取应用级 Access Token

**接口 URL**：`https://oauth-login.cloud.huawei.com/oauth2/v3/token`

**说明**：调用华为服务端 API 前，需先获取应用级 Access Token（App AT）用于鉴权。Token 有有效期，建议在 HTTP 401 时才重新申请，避免频繁申请触发服务限流。

**Client ID 和 Client Secret 获取方式**：登录 AppGallery Connect → 全部服务 → 盈利 → 应用内支付服务 → 查看 Client ID 和 Client Secret。

#### 请求参数

**Request Header**

| 参数 | 是否必选 | 类型 | 描述 |
|------|----------|------|------|
| Content-Type | 是 | String | 取值为：`application/x-www-form-urlencoded; charset=UTF-8` |

**Request Body（表单格式）**

| 参数 | 是否必选 | 类型 | 描述 |
|------|----------|------|------|
| grant_type | 是 | String | 固定值：`client_credentials` |
| client_id | 是 | String | 应用 ID（App ID），在 AppGallery Connect 创建应用后自动分配 |
| client_secret | 是 | String | 应用密钥（App Secret），在 AppGallery Connect 中查看 |

#### 响应参数

| 参数 | 类型 | 描述 |
|------|------|------|
| access_token | String | 应用级 Access Token |
| token_type | String | Token 类型，固定为 `Bearer` |
| expires_in | Integer | Token 有效期，单位：秒 |

#### 请求示例

```http
POST https://oauth-login.cloud.huawei.com/oauth2/v3/token
Content-Type: application/x-www-form-urlencoded; charset=UTF-8

grant_type=client_credentials&client_id=1234567&client_secret=appsecret
```

#### 响应示例

```json
{
  "access_token": "CFyJ...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

#### Authorization 构造方式

获取到 Access Token 后，在调用其他接口时，Header 中的 `Authorization` 构造方式如下：

```
Authorization: Basic {Base64("APPAT:{access_token}")}
```

---

### 1. Order 服务验证购买 Token

**接口 URL**：`{rootUrl}/applications/purchases/tokens/verify`

**说明**：向华为应用内支付服务器校验支付结果中的购买令牌，确认支付结果的准确性。只针对非订阅型商品（消耗型和非消耗型）。建议在发货前调用该接口验证，验证一致后再发货，发货成功后再调用确认购买接口。

#### 请求参数

**Request Header**

| 参数 | 是否必选 | 类型 | 描述 |
|------|----------|------|------|
| Content-Type | 是 | String | 取值为：`application/json; charset=UTF-8` |
| Authorization | 是 | String | 认证信息，格式：`Basic {Base64("APPAT:{AccessToken}")}` |
| HW-IAP-APPINFO | 否 | String | 扩展信息，支持传递签名算法 |

**Request Body**

| 参数 | 是否必选 | 类型 | 描述 |
|------|----------|------|------|
| purchaseToken | 是 | String | 待验证的购买 Token，唯一标识商品和用户的对应关系 |
| productId | 是 | String | 待下发商品 ID，来源于 AppGallery Connect 中配置的商品 ID |

#### 响应参数

**Response Body**

| 参数 | 是否必选 | 类型 | 描述 |
|------|----------|------|------|
| responseCode | 是 | String | 返回码。`0`：成功；其他：失败 |
| responseMessage | 否 | String | 响应描述 |
| purchaseTokenData | 否 | String | 包含购买数据的 JSON 字符串（InAppPurchaseData），该字段原样参与签名 |
| dataSignature | 否 | String | purchaseTokenData 基于应用 RSA IAP 私钥的签名信息，应用需使用 IAP 公钥验签 |
| signatureAlgorithm | 是 | String | 签名算法（如 `SHA256WithRSA`） |

#### 请求示例

```http
POST /applications/purchases/tokens/verify
Content-Type: application/json; charset=UTF-8
Authorization: Basic QVQ6Q1Yz...

{
  "purchaseToken": "00000173741056a37eef310dff9c6a86...",
  "productId": "prd1"
}
```

#### 响应示例

```json
{
  "responseCode": "0",
  "purchaseTokenData": "{\"autoRenewing\":false,\"orderId\":\"202008172303339595b1212421.123456\",\"packageName\":\"com.huawei.packagename\",\"applicationId\":123456,\"kind\":0,\"productId\":\"3\",\"productName\":\"商品名称\",\"purchaseTime\":1597676623000,\"purchaseTimeMillis\":1597676623000,\"purchaseState\":0,\"developerPayload\":\"payload data\",\"purchaseToken\":\"00000173741056a37...\",\"consumptionState\":0,\"confirmed\":0,\"currency\":\"CNY\",\"price\":\"100\",\"country\":\"CN\",\"payOrderId\":\"WX123456789ce8e23ee927\",\"payType\":\"17\"}",
  "dataSignature": "FiJJZYRdVIFgEzDA...",
  "signatureAlgorithm": "SHA256WithRSA"
}
```

---

### 2. Order 服务确认购买

**接口 URL**：`{rootUrl}/applications/v2/purchases/confirm`

**说明**：对于消耗型商品，发货成功后需调用此接口通知华为 IAP 服务器更新商品发货状态。若不调用，将导致用户无法再次购买该商品。等同于客户端的 `IapClient.consumeOwnedPurchase` 接口。

#### 请求参数

**Request Header**

| 参数 | 是否必选 | 类型 | 描述 |
|------|----------|------|------|
| Content-Type | 是 | String | 取值为：`application/json; charset=UTF-8` |
| Authorization | 是 | String | 认证信息，格式：`Basic {Base64("APPAT:{AccessToken}")}` |

**Request Body**

| 参数 | 是否必选 | 类型 | 描述 |
|------|----------|------|------|
| purchaseToken | 是 | String | 商品的购买 Token |
| productId | 是 | String | 商品 ID |

#### 响应参数

**Response Body**

| 参数 | 是否必选 | 类型 | 描述 |
|------|----------|------|------|
| responseCode | 是 | String | 返回码。`0`：成功；其他：失败 |
| responseMessage | 否 | String | 响应描述 |

#### 请求示例

```http
POST /applications/v2/purchases/confirm
Content-Type: application/json; charset=UTF-8
Authorization: Basic QVQ6Q1Yz...

{
  "purchaseToken": "00000173741056a37eef310dff9c6a86...",
  "productId": "prd1"
}
```

#### 响应示例

```json
{
  "responseCode": "0",
  "responseMessage": "consume success"
}
```

---

## 资损相关字段

资损防控需要重点关注以下字段，均来源于 `InAppPurchaseData`（由 `verifyToken` 接口的 `purchaseTokenData` 解析得到）。

### 金额信息字段

| 字段 | 类型 | 描述 |
|------|------|------|
| price | Long | 商品实际价格 × 100（精确到小数点后 2 位）。如值为 `501` 表示价格为 5.01。**验签后必须校验，避免资损** |
| currency | String | 定价货币币种（ISO 4217，如 `CNY`）。**验签后必须校验** |

### 购买者账号字段

| 字段 | 类型 | 描述 |
|------|------|------|
| applicationId | Long | 应用 ID，用于确认购买所属应用。**验签后必须校验** |
| packageName | String | 应用包名 |
| accountFlag | Integer | 账户类型：`1`=华为帐号，`0`=AppTouch 账户 |

### 订单状态字段

| 字段 | 类型 | 描述 |
|------|------|------|
| purchaseState | Integer | 购买状态：`0`=成功，`1`=已取消，`2`=已退款，`3`=待处理（延迟付款） |
| consumptionState | Integer | 消耗状态：`0`=未消耗，`1`=已消耗 |

### 发货商品字段

| 字段 | 类型 | 描述 |
|------|------|------|
| productId | String | 商品 ID。**验签后必须校验，与服务器本地商品 ID 对比** |
| productName | String | 商品名称 |
| kind | Integer | 商品类型：`0`=消耗型，`1`=非消耗型 |
| purchaseToken | String | 购买令牌，唯一标识一笔购买，用于发货和确认购买。**务必防止重复发货（幂等校验）** |
| orderId | String | 华为订单 ID |
| developerPayload | String | 开发者在发起购买时透传的自定义信息，用于关联自有订单 |
| purchaseTime | Long | 购买时间，UTC 时间戳（毫秒） |
