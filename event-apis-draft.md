# 事件服務 API 草稿

## 目錄

- [API 概覽](#api-概覽)
- [一般](#一般)
  - [列出事件](#1-列出事件)
  - [取得事件詳情](#2-取得事件詳情)
  - [列出事件回覆](#3-列出事件回覆)
  - [統計有效回覆數](#4-統計有效回覆數)
  - [取得抽獎結果](#5-取得抽獎結果)
  - [生成簽名明文挑戰](#6-生成簽名明文挑戰)
  - [提交回覆](#7-提交回覆)
  - [建立事件](#8-建立事件)
  - [追加獎金](#9-追加獎金)
- [Admin](#admin)
  - [更新事件狀態](#10-更新事件狀態)
  - [取消事件](#11-取消事件)
  - [隱藏事件](#12-隱藏事件)
  - [鎖定事件](#13-鎖定事件)
  - [執行抽獎](#14-執行抽獎)
  - [建立快照](#15-建立快照)
  - [隱藏回覆](#16-隱藏回覆)
- [統一回應格式](#統一回應格式)
- [錯誤碼說明](#錯誤碼說明)
- [使用範例](#使用範例)

---

## API 概覽

| 端點 | HTTP 方法 | 認證要求 | 角色要求 | 說明 |
|------|-----------|----------|----------|------|
| `/api/v1/events` | GET | - | - | 事件列表 |
| `/api/v1/events/:event_id` | GET | - | - | 取得單一事件詳情 |
| `/api/v1/events/:event_id/replies` | GET | - | - | 列出事件的所有回覆 |
| `/api/v1/events/:event_id/replies/count` | GET | - | - | 統計事件的有效回覆數 |
| `/api/v1/events/:event_id/lottery-results` | GET | - | - | 取得事件的抽獎結果 |
| `/api/v1/events/:event_id/challenges` | POST | - | - | 生成簽名明文挑戰 |
| `/api/v1/events/:event_id/replies` | POST | - | - | 提交事件回覆 |
| `/api/v1/events` | POST | - | - | 建立新事件 |
| `/api/v1/events/:event_id/rewards` | POST | - | - | 為事件追加獎金 |
| `/api/v1/events/:event_id/state` | PUT | JWT | Admin | 更新事件狀態（觸發 FSM 轉換） |
| `/api/v1/events/:event_id/cancel` | PUT | JWT | Admin | 取消事件 |
| `/api/v1/events/:event_id/hide` | PUT | JWT | Admin | 隱藏事件 |
| `/api/v1/events/:event_id/lock` | PUT | JWT | Admin | 鎖定事件 |
| `/api/v1/events/:event_id/lottery` | POST | JWT | Admin | 執行抽獎流程 |
| `/api/v1/events/:event_id/snapshot` | POST | JWT | Admin | 建立快照 |
| `/api/v1/events/:event_id/replies/:reply_id/hide` | PUT | JWT | Admin | 隱藏回覆 |

---

## 一般

以下 API 涵蓋了事件服務的基本功能。

### 1. 列出事件

取得事件列表，支援多種過濾條件和分頁。

**端點：** `GET /api/v1/events`

**請求參數（查詢）：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `state` | 字串 | 否 | 事件狀態過濾 | `ACTIVE` |
| `creator_address` | 字串 | 否 | 建立者 BTC 地址 | `bc1q...` |
| `type` | 字串 | 否 | 事件類型 | `single_choice` |
| `is_hidden` | 布林值 | 否 | 隱藏狀態 | `false` |
| `is_locked` | 布林值 | 否 | 鎖定狀態 | `false` |
| `keyword` | 字串 | 否 | 搜尋關鍵詞（標題、描述） | `比特幣` |
| `limit` | 整數 | 否 | 每頁數量（預設 20，最大 100） | `20` |
| `offset` | 整數 | 否 | 偏移量（預設 0） | `0` |

**狀態可選值：**
- `PENDING_PAYMENT` - 等待支付
- `ACTIVE` - 進行中
- `ENDED` - 已截止
- `COMPLETED` - 已完成
- `CANCELLED` - 已取消
- `REFUNDING` - 退款中
- `REFUNDED` - 已退款

**類型可選值：**
- `single_choice` - 單選題
- `open_ended` - 開放式問答

**請求範例：**
```bash
GET /api/v1/events?state=ACTIVE&limit=20&offset=0
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "events": [
      {
        "event_id": "550e8400-e29b-41d4-a716-446655440000",
        "title": "2025 年比特幣價格預測",
        "state": "ACTIVE",
        "deadline_at": "2025-01-03T10:15:00Z",
        "total_reward_btc": "0.00080000"
      }
    ]
  },
  "meta": {
    "total": 125,
    "limit": 20,
    "offset": 0
  }
}
```

### 2. 取得事件詳情

取得單個事件的完整詳細資訊。

**端點：** `GET /api/v1/events/:event_id`

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求範例：**
```bash
GET /api/v1/events/550e8400-e29b-41d4-a716-446655440000
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "2025 年比特幣價格預測",
    "description": "你認為 2025 年底比特幣價格會達到多少？",
    "state": "ACTIVE",
    "type": "single_choice",
    "options": ["$50,000-$75,000", "$75,000-$100,000", "$100,000-$150,000", "$150,000+"],
    "deadline_at": "2025-01-03T10:15:00Z",
    "total_reward_btc": "0.00080000"
  }
}
```

### 3. 列出事件回覆

取得指定事件的所有回覆列表，支援分頁。

**端點：** `GET /api/v1/events/:event_id/replies`

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求參數（查詢）：**

| 參數名 | 類型 | 必填 | 說明 | 預設值 |
|--------|------|------|------|--------|
| `limit` | 整數 | 否 | 每頁數量 | 20 |
| `offset` | 整數 | 否 | 偏移量（用於分頁） | 0 |

**請求範例：**
```bash
GET /api/v1/events/550e8400-e29b-41d4-a716-446655440000/replies?limit=20&offset=0
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "replies": [
      {
        "id": 1,
        "btc_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
        "content": "$100,000-$150,000",
        "is_hidden": false,
        "created_at": "2025-01-01T11:00:00Z"
      }
    ]
  },
  "meta": {
    "total": 42,
    "limit": 20,
    "offset": 0
  }
}
```

---

### 4. 統計有效回覆數

統計指定事件的有效回覆總數。

**端點：** `GET /api/v1/events/:event_id/replies/count`

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**有效回覆定義：**

滿足以下所有條件的回覆被視為有效：
- `is_signature_valid = true` - 簽名驗證通過
- `is_invalidated = false` - 未被全站重複簽名檢測標記無效
- `is_hidden = false` - 未被管理員隱藏
- `deleted_at IS NULL` - 未軟刪除

**請求範例：**
```bash
GET /api/v1/events/550e8400-e29b-41d4-a716-446655440000/replies/count
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "valid_replies_count": 42
  }
}
```

### 5. 取得抽獎結果

取得指定事件的抽獎結果和中獎者列表。

**端點：** `GET /api/v1/events/:event_id/lottery-results`

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求範例：**
```bash
GET /api/v1/events/550e8400-e29b-41d4-a716-446655440000/lottery-results
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "results": [
      {
        "reply_id": 42,
        "btc_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
        "final_amount_sats": 365000,
        "distribution_tx_id": "abc123def456..."
      }
    ]
  }
}
```

### 6. 生成簽名明文挑戰

用戶提交回覆前，必須先獲取待簽名的明文。此 API 生成符合 Event Service 規範的簽名明文。

**端點：** `POST /api/v1/events/:event_id/challenges`

**請求頭：**
```
Content-Type: application/json
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：**
```json
{
  "btc_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
  "content": "$100,000-$150,000"
}
```

**備註：** 明文格式為 `koinvote.com | type:{single|open} | ... | {event_id} | {timestamp} | {nonce}`，有效期 15 分鐘。

**請求範例：**
```bash
POST /api/v1/events/550e8400-e29b-41d4-a716-446655440000/challenges
Content-Type: application/json

{
  "btc_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
  "content": "$100,000-$150,000"
}
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "challenge_id": 12345,
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "btc_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
    "plaintext": "koinvote.com | type:single | $100,000-$150,000 | 550e8400-e29b-41d4-a716-446655440000 | 1704096000 | a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "expires_at": "2025-01-01T11:15:00.000000Z",
    "created_at": "2025-01-01T11:00:00.000000Z"
  }
}
```

### 7. 提交回覆

用戶使用 BTC 錢包對明文簽名後,提交簽名和回覆內容。系統將驗證簽名、檢測重複、記錄餘額並創建回覆記錄。

**端點：** `POST /api/v1/events/:event_id/replies`

**請求頭：**
```
Content-Type: application/json
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|---------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：**
```json
{
  "btc_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
  "content": "$100,000-$150,000",
  "signature": "H1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w4x5y6z7...",
  "challenge_id": 12345
}
```

**請求範例：**
```bash
POST /api/v1/events/550e8400-e29b-41d4-a716-446655440000/replies
Content-Type: application/json

{
  "btc_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
  "content": "$100,000-$150,000",
  "signature": "H1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w4x5y6z7...",
  "challenge_id": 12345
}
```

**回應範例（201 已創建）：**
```json
{
  "success": true,
  "data": {
    "id": 42,
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "is_signature_valid": true,
    "is_hidden": false,
    "created_at": "2025-01-01T11:05:00Z"
  },
  "meta": {
    "invalidated_other_replies": 2
  }
}
```

**備註：** 後端會進行重複簽名檢測；若同一簽名在其它 `ACTIVE` 事件出現，舊回覆會被標記為失效。

**失效操作：**
對查詢結果的所有回覆執行：
- `is_invalidated = true`
- `invalidated_at = NOW()`
- `invalidated_by_reply_id = 新回覆ID`
- 更新相關事件的 `valid_reply_count`（減少失效回覆數）

**重要說明：**
- 只檢測 ACTIVE 狀態的事件
- ENDED、COMPLETED 狀態的事件不受影響（快照已完成）
- 同一地址可在不同時間參與多個事件
- 檢測基於 `signature` 和 `btc_address` 組合

---

### 8. 建立事件

建立新的事件，系統將自動生成事件 ID 並建立支付地址。

**端點：** `POST /api/v1/events`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

**請求體：**
```json
{
  "title": "2025 年比特幣價格預測",
  "description": "你認為 2025 年底比特幣價格會達到多少？",
  "content": "請根據市場趨勢和技術分析，選擇你認為最可能的價格區間。",
  "creator_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
  "type": "single_choice",
  "options": ["$50,000-$75,000", "$75,000-$100,000", "$100,000-$150,000", "$150,000+"],
  "initial_reward_btc": "0.0005",
  "duration_hours": 48
}
```

**回應範例（201 已建立）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "state": "PENDING_PAYMENT",
    "payment_address": "bc1q...",
    "payment_amount_btc": "0.00050000",
    "payment_expires_at": "2025-01-01T11:00:00Z"
  }
}
```

---

### 9. 追加獎金

為已建立的事件追加額外獎金。只能為 `ACTIVE` 狀態的事件追加獎金。

**端點：** `POST /api/v1/events/:event_id/rewards`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：**
```json
{
  "reward_btc": "0.0003"
}
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "additional_reward_btc": "0.00030000",
    "new_total_reward_btc": "0.00080000",
    "payment_address": "bc1q...",
    "payment_expires_at": "2025-01-02T11:00:00Z"
  }
}
```

---

## Admin

以下 API 需要在請求頭中包含有效的 JWT 權杖，並需具備管理員角色權限。

### 10. 更新事件狀態

手動觸發 FSM 狀態轉換事件。通常由系統自動觸發，管理員可手動干預。

**端點：** `PUT /api/v1/events/:event_id/state`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：**
```json
{
  "event": "PAYMENT_CONFIRMED"
}
```

**可用事件：** `PAYMENT_CONFIRMED`, `PAYMENT_TIMEOUT`, `PAYMENT_NEED_REFUND`, `EXPIRED`, `ADMIN_CANCEL`, `LOTTERY_COMPLETED`, `REFUND_INITIATED`, `REFUND_COMPLETED`

**請求範例：**
```bash
PUT /api/v1/events/550e8400-e29b-41d4-a716-446655440000/state
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "message": "事件狀態更新成功",
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "previous_state": "PENDING_PAYMENT",
    "current_state": "ACTIVE",
    "triggered_event": "PAYMENT_CONFIRMED",
    "transitioned_at": "2025-01-01T10:15:00.000000Z"
  }
}
```

### 11. 取消事件

取消事件，可在 `ACTIVE` 或 `ENDED` 狀態下執行。

**端點：** `PUT /api/v1/events/:event_id/cancel`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：**
```json
{
  "reason": "違反社區規則：包含誤導性資訊"
}
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "current_state": "CANCELLED",
    "refund_initiated": true
  }
}
```

---

### 12. 隱藏事件

隱藏事件，使其不在公開列表中顯示。不影響事件狀態。

**端點：** `PUT /api/v1/events/:event_id/hide`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：** 無

**備註：** 操作後事件仍可透過直接連結存取，但不再顯示於公開列表。

**請求範例：**
```bash
PUT /api/v1/events/550e8400-e29b-41d4-a716-446655440000/hide
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "message": "事件已成功隱藏",
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "is_hidden": true,
    "hidden_at": "2025-01-01T15:45:00.000000Z"
  }
}
```

---

### 13. 鎖定事件

鎖定事件，禁止任何修改操作（如追加獎金、提交回覆）。

**端點：** `PUT /api/v1/events/:event_id/lock`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：** 無

**備註：** 鎖定後阻擋使用者操作（提交回覆、追加獎金等），但系統自動流程仍會照常運行。

**請求範例：**
```bash
PUT /api/v1/events/550e8400-e29b-41d4-a716-446655440000/lock
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "message": "事件已成功鎖定",
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "is_locked": true,
    "locked_at": "2025-01-01T16:00:00.000000Z"
  }
}
```

---

### 14. 執行抽獎

手動觸發抽獎流程。通常由系統在事件截止後自動執行。

**端點：** `POST /api/v1/events/:event_id/lottery`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：** 無

**備註：** 僅在事件 `ENDED` 後呼叫；後端會自動處理快照、抽籤與派獎流程並標記 `LOTTERY_COMPLETED`。

**請求範例：**
```bash
POST /api/v1/events/550e8400-e29b-41d4-a716-446655440000/lottery
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "total_winners": 3,
    "lottery_results": [
      {
        "reply_id": 42,
        "btc_address": "bc1q...",
        "final_amount_sats": 365000
      }
    ],
    "distribution_report_id": 1
  }
}
```

### 15. 建立快照

手動建立參與者餘額快照。通常在事件截止時自動執行。

**端點：** `POST /api/v1/events/:event_id/snapshot`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |

**請求體：** 無

**備註：** 後端會呼叫支付服務產生餘額快照，並更新事件的快照高度與回覆餘額。

**請求範例：**
```bash
POST /api/v1/events/550e8400-e29b-41d4-a716-446655440000/snapshot
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "message": "建立快照成功",
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "snapshot_block_height": 820000,
    "snapshot_block_hash": "00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054",
    "snapshot_taken_at": "2025-01-03T10:15:30.000000Z",
    "total_addresses_snapshotted": 42,
    "total_balance_btc": "2.50000000"
  }
}
```

---

### 16. 隱藏回覆

管理員隱藏不當回覆，使其不在公開列表中顯示。隱藏的回覆不計入有效回覆數。

**端點：** `PUT /api/v1/events/:event_id/replies/:reply_id/hide`

**請求頭：**
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

**路徑參數：**

| 參數名 | 類型 | 必填 | 說明 | 範例 |
|--------|------|------|------|------|
| `event_id` | 字串 | 是 | 事件唯一標識符 | `550e8400-e29b-41d4-a716-446655440000` |
| `reply_id` | 整數 | 是 | 回覆唯一標識 | `42` |

**請求體：**
```json
{
  "reason": "垃圾內容"
}
```

**回應範例（200 成功）：**
```json
{
  "success": true,
  "data": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "reply_id": 42,
    "is_hidden": true
  }
}
```

## 統一回應格式

### 成功回應

所有成功的 API 回應遵循以下格式：

```json
{
  "success": true,
  "data": {
    // 回應資料
  },
  "meta": {
    // 元資料（分頁、統計等）
  }
}
```

### 錯誤回應

所有錯誤回應遵循以下格式：

```json
{
  "success": false,
  "error": {
    "code": 400,
    "message": "輸入資料無效",
    "details": "必須提供事件 ID"
  }
}
```

## 錯誤碼說明

### HTTP 狀態碼

| 狀態碼 | 說明 | 常見場景 |
|--------|------|----------|
| 200 | 成功 | 請求成功 |
| 201 | 已建立 | 資源建立成功 |
| 400 | 錯誤請求 | 請求參數錯誤、驗證失敗 |
| 401 | 未授權 | 未提供 JWT 權杖或權杖無效 |
| 403 | 禁止訪問 | 權限不足（如非管理員存取管理員 API） |
| 404 | 未找到 | 資源不存在（如事件 ID 不存在） |
| 409 | 衝突 | 業務衝突（如重複簽名、無效狀態轉換） |
| 500 | 伺服器內部錯誤 | 伺服器內部錯誤 |
| 503 | 服務暫時不可用 | 支付服務不可用 |

## 使用範例

- 建立事件：POST `/api/v1/events` → 取得 `event_id`、付款地址與金額 → 完成支付後查詢事件轉為 `ACTIVE`。
- 追加獎金：對同一 `event_id` 呼叫 `/rewards` 取得新的付款資訊並完成支付。
- 用戶提交回覆：`POST /challenges` 取得明文 → 客端簽名 → `POST /replies` 上傳簽名結果。
- 管理員流程：事件 `ENDED` 後先 `POST /snapshot`（如需）→ `POST /lottery` 發獎；必要時可 `cancel`、`hide`、`lock` 事件或 `hide` 指定回覆。
