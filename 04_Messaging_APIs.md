# Mercury Messaging APIs

End-to-end encrypted messaging system APIs.

---

## Overview

Mercury is the E2E encrypted messaging system. Messages are encrypted client-side before sending.

---

## API_MessageGetKeys

**Purpose:** Returns the user's messaging keys.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_MessageGetKeys"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | JSON: `{"publicKey":"...", "hasPrivateKey":true, "privateKeyEnc":"..."}` |

### Response Example

```json
{
  "p1": "Success",
  "p3": ["{\"publicKey\":\"base64...\",\"hasPrivateKey\":true,\"privateKeyEnc\":\"base64...\"}"]
}
```

---

## API_MessageGetUserPublicKey

**Purpose:** Returns another user's public key for encryption.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_MessageGetUserPublicKey"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Target user UUID |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | JSON: `{"publicKey":"..."}` |

---

## API_MessageInitKeys

**Purpose:** Initializes or regenerates user's messaging keys.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_MessageInitKeys"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Public key (base64) |
| `p3[1]` | string | Yes | Encrypted private key (base64) |

### Notes

- Client generates keys locally
- Public key is stored unencrypted for sharing
- Private key is encrypted by client before sending
- Server stores encrypted private key for key recovery

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |

---

## API_MessageConversationStart

**Purpose:** Starts a new conversation with another user, or returns existing conversation.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_MessageConversationStart"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Recipient user UUID |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | JSON: `{"conversationUuid":"...", "isExisting":false}` |

### Notes

- Cannot start conversation with yourself
- If conversation already exists, returns existing UUID with `isExisting: true`

---

## API_MessageConversationList

**Purpose:** Returns the user's conversations.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_MessageConversationList"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | No | Limit (default: 20) |
| `p3[1]` | string | No | Offset (default: 0) |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | JSON array of conversation objects |

---

## API_MessageSend

**Purpose:** Sends an encrypted message in a conversation.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_MessageSend"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Conversation UUID |
| `p3[1]` | string | Yes | Message type (see below) |
| `p3[2]` | string | Yes | Encrypted content (base64) |
| `p3[3]` | string | Yes | Content nonce (base64) |
| `p3[4]` | string | No | Reply-to message UUID |
| `p3[5]` | string | No | Encrypted metadata (base64) |

### Message Types

Message types are defined in the messaging library. Check `messaging.MessageTypeText` etc for current values.

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | JSON: `{"messageUuid":"...", "createdAt":"1699876543210"}` |

---

## API_MessageGetMessages

**Purpose:** Returns messages in a conversation.

**Authentication:** Requires valid login (p1=email, p2=token)

### Request RObj

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `req` | string | Yes | `"API_MessageGetMessages"` |
| `p1` | string | Yes | Username (email) |
| `p2` | string | Yes | Token |
| `p3[0]` | string | Yes | Conversation UUID |
| `p3[1]` | string | No | Limit |
| `p3[2]` | string | No | Offset |
| `p3[3]` | string | No | Before timestamp (for pagination) |

### Response RObj

| Field | Description |
|-------|-------------|
| `p1` | `"Success"` or `"Fail"` |
| `p2` | Error message (on failure) |
| `p3[0]` | JSON array of message objects |

---

## API_MessageBlock

**Purpose:** Blocks/unblocks users from messaging.

**Authentication:** Requires valid login (p1=email, p2=token)

### Notes

See actual implementation in `api_messageblock.go` for full details on actions and parameters.

---

## WebSocket Notification Events (SAPI)

Mercury messaging uses WebSocket for real-time delivery. These are **server-push** events - the server sends them to connected clients when messaging events occur.

### SAPI_NewMessage

**Purpose:** Notifies a user that they received a new message.

**Payload Format:**

| Field | Description |
|-------|-------------|
| `req` | `"SAPI_NewMessage"` |
| `p1` | `"Success"` |
| `p2` | JSON object (see below) |

**P2 JSON Object:**

| Field | Type | Description |
|-------|------|-------------|
| `messageUuid` | string | UUID of the new message |
| `conversationUuid` | string | UUID of the conversation |
| `senderUuid` | string | UUID of the sender |
| `senderName` | string | Display name of sender |
| `messageType` | string | Message type code (as string) |
| `createdAt` | string | Unix timestamp in milliseconds (as string) |

**Example:**

```json
{
  "req": "SAPI_NewMessage",
  "p1": "Success",
  "p2": "{\"messageUuid\":\"abc-123\",\"conversationUuid\":\"def-456\",\"senderUuid\":\"user-uuid\",\"senderName\":\"John Smith\",\"messageType\":\"1\",\"createdAt\":\"1702819200000\"}"
}
```

### SAPI_MessageDeleted

**Purpose:** Notifies a user that a message was deleted.

**P2 JSON Object:**

| Field | Type | Description |
|-------|------|-------------|
| `messageUuid` | string | UUID of the deleted message |
| `conversationUuid` | string | UUID of the conversation |

### SAPI_MessageEdited

**Purpose:** Notifies a user that a message was edited.

**P2 JSON Object:**

| Field | Type | Description |
|-------|------|-------------|
| `messageUuid` | string | UUID of the edited message |
| `conversationUuid` | string | UUID of the conversation |
| `editedAt` | string | Unix timestamp in milliseconds (as string) |

### SAPI_ReadReceipt

**Purpose:** Notifies a user that their message was read.

**P2 JSON Object:**

| Field | Type | Description |
|-------|------|-------------|
| `messageUuid` | string | UUID of the read message |
| `readerUuid` | string | UUID of the reader |
| `readAt` | string | Unix timestamp in milliseconds (as string) |

### SAPI_ConversationStarted

**Purpose:** Notifies a user that they were added to a new conversation.

**P2 JSON Object:**

| Field | Type | Description |
|-------|------|-------------|
| `conversationUuid` | string | UUID of the new conversation |
| `initiatorUuid` | string | UUID of the user who started the conversation |
| `initiatorName` | string | Display name of the initiator |

---

## Important Notes

- **All numeric values in WebSocket events are sent as strings** to ensure consistent parsing across platforms
- Message events are only delivered when the user has an active WebSocket connection
- If offline, messages are retrieved via `API_MessageGetMessages` on next connection
