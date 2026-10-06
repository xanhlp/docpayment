# Xanh Lộc Phát 系统合作伙伴接入指南

本指南介绍通过 Xanh Lộc Phát 创建充值订单和提现请求的 API。所有金额均以越南盾（VND）计价。

## 1. 接入信息

接入前，请向 Xanh Lộc Phát 接入联系人获取或确认以下信息：

| 信息 | 用途 |
| --- | --- |
| `BASE_URL` | 对应环境的 API 基础地址。 |
| API key | 合作伙伴凭据，通过 `x-secret-key` 发送。 |
| `signatureKey` | 用于提现请求签名的独立密钥。 |
| IP 白名单 | 如启用白名单，须登记调用 API 的来源 IP。 |
| 金额限制 | 确认合作伙伴账户的充值及提现金额上下限。 |
| Webhook URL | 提供合作伙伴回调接收地址用于登记。 |

合作伙伴账户必须为 `ACTIVE`。所有请求均使用 `Content-Type: application/json`。API key 与 `signatureKey` 是两个不同的密钥。

以下 URL、账户信息及交易编号均为示例，须替换为实际交付的接入信息。响应示例中的二维码图片数据已省略部分内容。

## 2. 创建充值订单

| 属性 | 值 |
| --- | --- |
| 方法 | `POST` |
| URL | `{BASE_URL}/api/orders` |
| 必填认证请求头 | `x-secret-key: <API_KEY>` |
| 成功响应 | `201 Created` |

此接口创建供客户向代收账户转账的订单，并返回支付二维码图片。系统从已启用的收款账户中随机选择银行，请求参数不能指定银行。

### 请求字段

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `orderId` | string | 是 | 合作伙伴侧唯一订单号，用于对账。建议格式为 `^[A-Za-z0-9_-]+$`，长度 3–100 个字符。 |
| `amount` | number | 是 | VND 金额。参考范围为 2,000,000–299,000,000，须确认账户限额。 |
| `orderInfo` | object | 否 | 附加 JSON 信息，用于参考或对账。 |
| `description` | string | 是 | 交易描述，长度 5–255 个字符，允许字母、数字、空格、`-` 和 `#`。 |

```json
{
  "orderId": "ORD_123456789",
  "amount": 10000000,
  "description": "USDT Purchase - Order #123456789",
  "orderInfo": {
    "customerId": "12345",
    "orderType": "buy",
    "usdtAmount": 40.5,
    "exchangeRate": 25000
  }
}
```

### 成功响应

充值响应使用 `success` / `message` / `data` 外层结构。

```json
{
  "success": true,
  "message": "Order created successfully",
  "data": {
    "orderCode": "LPA1B2C3D4E5",
    "orderId": "ORD_123456789",
    "amount": 10000000,
    "callbackUrl": "https://exchange.example.com/webhook/payment",
    "qrCode": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
    "bankAccount": {
      "bankCode": "TPBANK",
      "accountNumber": "1234567890",
      "accountName": "XANH LOC PHAT"
    }
  }
}
```

| `data` 字段 | 类型 | 说明 |
| --- | --- | --- |
| `orderCode` | string | 平台订单号：`LP` 加 10 个大写字母或数字，共 12 个字符。 |
| `orderId` | string | 请求中的合作伙伴订单号。 |
| `amount` | number | 订单金额，单位为 VND。 |
| `callbackUrl` | string 或 null | 已登记的合作伙伴 webhook URL；未配置时为 `null`。 |
| `qrCode` | string | 二维码图片 Data URL，通常为 `data:image/png;base64,...`。 |
| `bankAccount` | object | 包含 `bankCode`、`accountNumber` 和 `accountName`。 |

订单默认有效期为 **30 分钟**。向客户展示该订单返回的二维码和收款账户信息。

### 重复订单

`orderId` 必须全局唯一。已有 ID 将被拒绝，返回 HTTP `400`，消息形如 `OrderId {orderId} exists`。此接口**不会**在重复请求时返回旧订单。

## 3. 登记回调 URL

向 Xanh Lộc Phát 接入联系人提供合作伙伴的 webhook URL。登记后，创建充值订单时会在 `data.callbackUrl` 中返回该地址。`callbackUrl` 是响应字段，不是创建充值订单的请求字段。

创建接口未定义回调 payload、签名、结果状态、接收确认或重试规则。实现接收端前，须另行确认回调规范。

## 4. 创建提现请求

| 属性 | 值 |
| --- | --- |
| 方法 | `POST` |
| URL | `{BASE_URL}/api/withdrawals/create` |
| 必填请求头 | `x-secret-key: <API_KEY>` 和 `x-signature: <SIGNATURE>` |
| 成功响应 | `201 Created` |

此接口向收款人的银行账户发起付款请求。

### 请求字段

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `withdrawalId` | string | 是 | 合作伙伴侧唯一提现 ID。建议格式为 `^[A-Za-z0-9_-]+$`，用于同一合作伙伴账户内的幂等处理。 |
| `bankCode` | enum | 是 | 下方列表中的银行代码。 |
| `beneficiaryAccountNo` | string | 是 | 收款人银行账号，允许字母和数字。 |
| `beneficiaryName` | string | 是 | 拉丁字母形式的收款人姓名。签名时保留原值。 |
| `amount` | integer | 是 | VND 金额。参考范围为 2,000,000–299,000,000，实际以合作伙伴账户限额为准。 |
| `description` | string | 是 | 用于展示和对账的交易描述。 |

```json
{
  "withdrawalId": "WDR_001",
  "bankCode": "VIETCOMBANK",
  "beneficiaryAccountNo": "1234567890",
  "beneficiaryName": "Nguyen Van A",
  "amount": 5000000,
  "description": "USDT Withdrawal - Order #1"
}
```

### 计算 `x-signature`

按以下顺序直接拼接字段，**中间不加任何分隔符**：

```text
withdrawalId + bankCode + beneficiaryAccountNo + beneficiaryName + amount
```

`amount` 使用十进制整数字符串。`beneficiaryName` 必须与 JSON 中的原值一致，包括空格。`description` 不参与拼接。

上述请求对应的待签名字符串为：

```text
WDR_001VIETCOMBANK1234567890Nguyen Van A5000000
```

使用 `signatureKey` 计算 **HMAC-SHA256**，将结果编码为 64 位十六进制字符串，通过 `x-signature` 发送。

Node.js 示例：`body` 为请求对象，`signatureKey` 为单独交付的签名密钥。

```js
import { createHmac } from 'node:crypto';

const payload = body.withdrawalId
  + body.bankCode
  + body.beneficiaryAccountNo
  + body.beneficiaryName
  + String(body.amount);

const signature = createHmac('sha256', signatureKey)
  .update(payload, 'utf8')
  .digest('hex');
```

### 成功响应

提现响应直接返回对象，**没有** `success` / `data` 外层。

```json
{
  "withdrawalId": "WDR_001",
  "withdrawalCode": "WDR_A1B2C3D4E5F6",
  "amount": 5000000,
  "status": "PENDING",
  "bankCode": "VIETCOMBANK",
  "beneficiaryAccountNo": "1234567890",
  "beneficiaryName": "NGUYEN VAN A",
  "description": "USDT Withdrawal - Order #1",
  "createdAt": "2026-10-05T00:00:00.000Z"
}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `withdrawalId` | string | 请求中的合作伙伴提现 ID。 |
| `withdrawalCode` | string | 平台提现编号。 |
| `amount` | number | 提现金额，单位为 VND。 |
| `status` | string | 新请求通常为 `PENDING`，随后随处理过程更新。 |
| `bankCode` | string | 收款银行代码。 |
| `beneficiaryAccountNo` | string | 收款人银行账号。 |
| `beneficiaryName` | string | 收款人姓名，可能规范化为大写。 |
| `description` | string | 交易描述。 |
| `createdAt` | string | ISO 8601 格式的创建时间。 |

### 重复提现

同一合作伙伴账户重复使用相同的 `withdrawalId` 时，返回已有记录。重试同一笔提现时应保留原 ID。

### 银行代码

请严格使用以下拼写及大小写：

```text
VIETCOMBANK, AGRIBANK, BIDV, SACOMBANK, TECHCOMBANK, TPBANK,
HDBank, VPBANK, MBBANK, MSB, LPBANK, ACB, ABBANK, NCB,
EXIMBANK, SEABANK, SCB, PVCOMBANK, KIENLONGBANK, BVBANK,
VIETINBANK, OCB, VIETBANK, SHB, NAMABANK, SHINHANBANKVIETNAM,
CIMB, SAIGONBANK, BACABANK, VIETABANK, GPBANK, HSBC, VRB,
VIB, PGBANK, SEPAY
```

## 5. 错误响应

错误响应通常采用以下结构，可能包含额外字段：

```json
{
  "success": false,
  "data": null,
  "error": {
    "statusCode": 400,
    "message": "...",
    "timestamp": "..."
  }
}
```

参数校验消息可能为字符串或消息数组。

| HTTP | 消息或情况 | 含义 |
| --- | --- | --- |
| `400` | 请求参数校验错误 | 请求字段不合法。 |
| `400` | `Amount must be in range of {minDepositAmount} - {maxDepositAmount}` | 充值金额超出配置限额。 |
| `400` | `OrderId {orderId} exists` | 充值订单号重复。 |
| `400` | `Failed to create order` | 无法创建充值订单。 |
| `400` | `Amount must be in range of {min} - {max}` | 提现金额超出配置限额。 |
| `400` | `x-signature header is required` | 缺少提现签名。 |
| `400` | `Invalid signature` | 提现签名错误。 |
| `401` | `Secret key is required` | 缺少 API key。 |
| `401` | `Invalid secret key` | API key 无效。 |
| `401` | `Exchange account is not active` | 合作伙伴账户未启用。 |
| `401` | `Forbidden (cors)` | 来源 IP 不在已配置的白名单内。 |
| `403` | 提现关闭或维护中 | 暂时无法发起提现。 |
| `429` | Too Many Requests | 超过请求频率限制。 |
| `500` | Internal Server Error | 服务未能完成请求。 |
