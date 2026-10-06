# Xanh Lộc Phát Partner Integration Guide

This guide describes the APIs for creating deposit orders and withdrawal requests with Xanh Lộc Phát. All amounts are in Vietnamese dong (VND).

## 1. Connection details

Before integration, obtain the following details from your Xanh Lộc Phát integration contact:

| Item | Purpose |
| --- | --- |
| `BASE_URL` | API base URL for the assigned environment. |
| API key | Partner credential sent in `x-secret-key`. |
| `signatureKey` | Separate key used to sign withdrawal requests. |
| IP allowlist | Register the source IP addresses that will call the APIs, if an allowlist is enabled. |
| Amount limits | Confirm the deposit and withdrawal minimum and maximum for the partner account. |
| Webhook URL | Provide the partner's callback receiver URL for registration. |

The partner account must be `ACTIVE`. All requests use `Content-Type: application/json`. The API key and `signatureKey` are different keys.

Example URLs, account details and IDs below are illustrative. Replace them with the details assigned for your integration. The QR image in the response example is abbreviated.

## 2. Create a deposit order

| Property | Value |
| --- | --- |
| Method | `POST` |
| URL | `{BASE_URL}/api/orders` |
| Required authentication header | `x-secret-key: <API_KEY>` |
| Successful response | `201 Created` |

This endpoint creates an order for the customer to transfer funds to a collection account and returns a payment QR image. A receiving bank is selected randomly from the enabled collection accounts; the request cannot specify a bank.

### Request fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `orderId` | string | Yes | Partner's unique order ID for reconciliation. Recommended format: `^[A-Za-z0-9_-]+$`, 3–100 characters. |
| `amount` | number | Yes | Amount in VND. Reference range: 2,000,000–299,000,000; confirm the account limits. |
| `orderInfo` | object | No | Additional JSON information for reference or reconciliation. |
| `description` | string | Yes | Transaction description, 5–255 characters: letters, digits, spaces, `-` and `#`. |

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

### Successful response

Deposit responses use a `success` / `message` / `data` wrapper.

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

| Field in `data` | Type | Description |
| --- | --- | --- |
| `orderCode` | string | Platform order code: `LP` followed by 10 uppercase letters/digits, 12 characters in total. |
| `orderId` | string | The partner order ID from the request. |
| `amount` | number | Order amount in VND. |
| `callbackUrl` | string or null | The registered partner webhook URL, or `null` if not configured. |
| `qrCode` | string | QR image Data URL, typically `data:image/png;base64,...`. |
| `bankAccount` | object | Contains `bankCode`, `accountNumber` and `accountName`. |

Default order validity is **30 minutes**. Display the QR image and bank details returned for that order.

### Duplicate orders

`orderId` must be globally unique. An existing ID is rejected with HTTP `400` and a message such as `OrderId {orderId} exists`. This endpoint does **not** return the existing order on a duplicate request.

## 3. Register the callback URL

Provide the partner's webhook URL to the Xanh Lộc Phát integration contact. Once registered, it is returned as `data.callbackUrl` when creating a deposit order. `callbackUrl` is a response field, not a field in the deposit creation request.

The creation APIs do not specify the callback payload, signature, result statuses, acknowledgment or retry rules. Confirm the callback specification separately before implementing the receiver.

## 4. Create a withdrawal request

| Property | Value |
| --- | --- |
| Method | `POST` |
| URL | `{BASE_URL}/api/withdrawals/create` |
| Required headers | `x-secret-key: <API_KEY>` and `x-signature: <SIGNATURE>` |
| Successful response | `201 Created` |

This endpoint requests a payout to a beneficiary bank account.

### Request fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `withdrawalId` | string | Yes | Partner's unique withdrawal ID. Recommended format: `^[A-Za-z0-9_-]+$`. Used for idempotency within the same partner account. |
| `bankCode` | enum | Yes | Bank code from the list below. |
| `beneficiaryAccountNo` | string | Yes | Beneficiary account number; letters and digits. |
| `beneficiaryName` | string | Yes | Beneficiary name in Latin letters. Preserve its original value when signing. |
| `amount` | integer | Yes | Amount in VND. Reference range: 2,000,000–299,000,000; actual limits depend on the partner account. |
| `description` | string | Yes | Transaction description for display and reconciliation. |

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

### Calculate `x-signature`

Concatenate the following fields in this exact order, with **no separators**:

```text
withdrawalId + bankCode + beneficiaryAccountNo + beneficiaryName + amount
```

Use the decimal integer representation of `amount`. Keep `beneficiaryName` exactly as supplied in the JSON, including spaces. The `description` field is not part of this string.

For the request above, the string to sign is:

```text
WDR_001VIETCOMBANK1234567890Nguyen Van A5000000
```

Calculate **HMAC-SHA256** using `signatureKey`, encode the result as 64 hexadecimal characters and send it in `x-signature`.

Node.js example, where `body` is the request object and `signatureKey` is the separately supplied signing key:

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

### Successful response

Withdrawal responses are returned directly, **without** a `success` / `data` wrapper.

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

| Field | Type | Description |
| --- | --- | --- |
| `withdrawalId` | string | Partner withdrawal ID from the request. |
| `withdrawalCode` | string | Platform withdrawal code. |
| `amount` | number | Withdrawal amount in VND. |
| `status` | string | A new request normally starts as `PENDING` and is updated as processing continues. |
| `bankCode` | string | Beneficiary bank code. |
| `beneficiaryAccountNo` | string | Beneficiary account number. |
| `beneficiaryName` | string | Beneficiary name; may be normalized to uppercase. |
| `description` | string | Transaction description. |
| `createdAt` | string | Creation time in ISO 8601 format. |

### Duplicate withdrawals

Reusing the same `withdrawalId` within the same partner account returns the existing record. Retain the original ID when retrying the same withdrawal.

### Bank codes

Use the exact spelling and case:

```text
VIETCOMBANK, AGRIBANK, BIDV, SACOMBANK, TECHCOMBANK, TPBANK,
HDBank, VPBANK, MBBANK, MSB, LPBANK, ACB, ABBANK, NCB,
EXIMBANK, SEABANK, SCB, PVCOMBANK, KIENLONGBANK, BVBANK,
VIETINBANK, OCB, VIETBANK, SHB, NAMABANK, SHINHANBANKVIETNAM,
CIMB, SAIGONBANK, BACABANK, VIETABANK, GPBANK, HSBC, VRB,
VIB, PGBANK, SEPAY
```

## 5. Error responses

An error response generally has the following structure; additional fields may be present:

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

Validation messages may be a string or a list of messages.

| HTTP | Message or condition | Meaning |
| --- | --- | --- |
| `400` | Request validation error | Invalid request fields. |
| `400` | `Amount must be in range of {minDepositAmount} - {maxDepositAmount}` | Deposit amount outside the configured limits. |
| `400` | `OrderId {orderId} exists` | Duplicate deposit order ID. |
| `400` | `Failed to create order` | Deposit order could not be created. |
| `400` | `Amount must be in range of {min} - {max}` | Withdrawal amount outside the configured limits. |
| `400` | `x-signature header is required` | Missing withdrawal signature. |
| `400` | `Invalid signature` | Incorrect withdrawal signature. |
| `401` | `Secret key is required` | Missing API key. |
| `401` | `Invalid secret key` | Invalid API key. |
| `401` | `Exchange account is not active` | Partner account is not active. |
| `401` | `Forbidden (cors)` | Source IP is not allowed by the configured IP allowlist. |
| `403` | Withdrawals disabled or under maintenance | Withdrawal requests are temporarily unavailable. |
| `429` | Too Many Requests | Request rate limit exceeded. |
| `500` | Internal Server Error | The service could not complete the request. |
