# Mercury Messaging System - Flutter Implementation Guide

## Overview

Mercury is LexCV's end-to-end encrypted messaging system with forward secrecy. Messages are encrypted on the client before sending and can only be decrypted by conversation participants.


---

## CRYPTOGRAPHIC SPECIFICATION (READ THIS FIRST)

### Algorithms Used

| Purpose | Algorithm | Library |
|----|----|----|
| Key Exchange | X25519 (Curve25519 ECDH) | `golang.org/x/crypto/curve25519` (server) |
| Symmetric Encryption | **XChaCha20-Poly1305** | `golang.org/x/crypto/chacha20poly1305` (server) |
| Encoding | Base64 (standard, with padding) | Standard library |

### Key Sizes

| Item | Size (bytes) |
|----|----|
| X25519 Private Key | 32 |
| X25519 Public Key | 32 |
| Session Key (symmetric) | 32 |
| XChaCha20 Nonce | 24 |
| Poly1305 Auth Tag | 16 |

### Session Key Encryption Format

When the server returns `sessionKeyEnc`, it is a base64-encoded blob with this **exact** binary format:

```
┌─────────────────────────┬─────────────────┬─────────────────────────────────┐
│ Ephemeral Public Key    │ Nonce           │ Ciphertext + Auth Tag           │
│ (32 bytes)              │ (24 bytes)      │ (32 + 16 = 48 bytes)            │
└─────────────────────────┴─────────────────┴─────────────────────────────────┘
                         Total: 104 bytes → ~140 chars base64
```

**Fields:**


1. **Ephemeral Public Key (32 bytes)**: Server generates a one-time X25519 key pair per encryption
2. **Nonce (24 bytes)**: Random nonce for XChaCha20-Poly1305
3. **Ciphertext (48 bytes)**: The 32-byte session key encrypted with XChaCha20-Poly1305 (includes 16-byte auth tag)

### How to Decrypt Session Key (Client Algorithm)

```
1. Decode sessionKeyEnc from base64 → raw bytes (should be 104 bytes)
2. Extract:
   - ephemeralPubKey = bytes[0:32]
   - nonce = bytes[32:56]
   - ciphertext = bytes[56:104]
3. Compute shared secret:
   - sharedSecret = X25519(myPrivateKey, ephemeralPubKey)
4. Decrypt session key:
   - sessionKey = XChaCha20-Poly1305.Open(sharedSecret, nonce, ciphertext)
5. sessionKey is 32 bytes - use for message encryption/decryption
```

### Message Encryption Format

When sending messages, the client encrypts content with the session key:

```
contentEnc = base64(XChaCha20-Poly1305.Seal(sessionKey, nonce, plaintext))
contentNonce = base64(nonce)
```

The ciphertext includes the 16-byte Poly1305 authentication tag appended.

### CRITICAL: Do NOT Use NaCl SecretBox

The server uses **XChaCha20-Poly1305** (RFC 8439 extended), NOT NaCl SecretBox (which uses XSalsa20-Poly1305). These are different algorithms and are NOT interoperable.

**Dart libraries:**

* ✅ Use: `cryptography` package with `Xchacha20.poly1305Aead()`
* ❌ Do NOT use: `pinenacl` SecretBox (uses XSalsa20)


---

## API Response Convention

All Mercury APIs follow the LexCV standard response format:

* **P1**: Status (`"Success"` or `"Fail"`)
* **P2**: Error message (only used when P1 = `"Fail"`)
* **P3\[0\]**: JSON response payload (success data, parsed with `jsonDecode(response.p3[0])`)
* **P4-P6**: Additional payload data (rarely used)

```dart
// Standard API response handling pattern
final response = await api.call('API_MessageXxx', p3: [args...]);
if (response.p1 == 'Success') {
  final data = jsonDecode(response.p3[0]);  // Success: parse P3
  // use data...
} else {
  throw Exception(response.p2);  // Fail: P2 contains error message
}
```


---

## Dependencies

Add these to `pubspec.yaml`:

```yaml
dependencies:
  cryptography: ^2.5.0  # For X25519 and XChaCha20-Poly1305
```

**⚠️ Do NOT use** `pinenacl` for encryption - it uses XSalsa20 which is incompatible with the server's XChaCha20.


---

## Key Management

### 1. Generate Identity Key Pair (First Time Setup)

> **⚠️ CRITICAL: Client-Side Key Generation Required**
>
> The user's key pair MUST be generated on the client device. The server **never** generates keys.
> This is essential for true end-to-end encryption - the server never sees the private key plaintext.
>
> **Mandatory flow on first login / app install:**
>
> 
> 1. Generate key pair locally (X25519)
> 2. Encrypt private key with password-derived key (AES-GCM)
> 3. Store encrypted private key locally (SecureStorage)
> 4. Send public key + encrypted private key to server via `API_MessageInitKeys`
> 5. Server stores keys and marks user as messaging-enabled
>
> If keys are not initialized, the user **cannot** start or receive conversations.
> The server will return `initiator_keys_not_initialized` or `recipient_keys_not_initialized`.

When user first accesses messaging, generate and store their identity key pair:

```dart
import 'package:cryptography/cryptography.dart';
import 'dart:convert';

class MercuryKeys {
  static final _x25519 = X25519();
  
  /// Generate a new X25519 key pair
  static Future<SimpleKeyPair> generateKeyPair() async {
    return await _x25519.newKeyPair();
  }
  
  /// Encode key to base64
  static String encodeKey(List<int> key) {
    return base64.encode(key);
  }
  
  /// Decode base64 key
  static List<int> decodeKey(String encoded) {
    return base64.decode(encoded);
  }
  
  /// Initialize user keys and send to server
  static Future<void> initializeKeys(String userPassword) async {
    final keyPair = await generateKeyPair();
    final publicKey = await keyPair.extractPublicKey();
    final privateKey = await keyPair.extractPrivateKeyBytes();
    
    // Encrypt private key with password-derived key before storing
    final privateKeyEnc = await _encryptPrivateKey(privateKey, userPassword);
    
    final publicKeyB64 = encodeKey(publicKey.bytes);
    final privateKeyEncB64 = encodeKey(privateKeyEnc);
    
    // Send to server
    await api.call('API_MessageInitKeys', p3: [publicKeyB64, privateKeyEncB64]);
    
    // Store locally for quick access
    await SecureStorage.write('mercury_public_key', publicKeyB64);
    await SecureStorage.write('mercury_private_key_enc', privateKeyEncB64);
  }
  
  /// Encrypt private key with user's password
  static Future<List<int>> _encryptPrivateKey(List<int> privateKey, String password) async {
    // Derive key from password using Argon2 or PBKDF2
    final algorithm = AesGcm.with256bits();
    final secretKey = await _deriveKeyFromPassword(password);
    final nonce = algorithm.newNonce();
    
    final secretBox = await algorithm.encrypt(
      privateKey,
      secretKey: secretKey,
      nonce: nonce,
    );
    
    // Return nonce + ciphertext + mac
    return [...nonce, ...secretBox.cipherText, ...secretBox.mac.bytes];
  }
}
```

### 2. Retrieve Keys from Server

```dart
/// Get user's own keys
Future<MercuryUserKeys> getMyKeys() async {
  final response = await api.call('API_MessageGetKeys');
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    return MercuryUserKeys(
      publicKey: data['publicKey'],
      privateKeyEnc: data['privateKeyEnc'],
      hasPrivateKey: data['hasPrivateKey'],
    );
  }
  throw Exception(response.p2);
}

/// Get another user's public key (for encryption)
Future<String> getUserPublicKey(String userUUID) async {
  final response = await api.call('API_MessageGetUserPublicKey', p3: [userUUID]);
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    return data['publicKey'];
  }
  throw Exception(response.p2);
}
```


---

## Conversations

### 1. Start a Conversation

```dart
/// Start a new 1:1 conversation with another user
Future<StartConversationResult> startConversation(String recipientUUID) async {
  final response = await api.call('API_MessageConversationStart', p3: [recipientUUID]);
  
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    return StartConversationResult(
      conversationUUID: data['conversationUuid'],
      isExisting: data['isExisting'] ?? false,
    );
  }
  throw Exception(response.p2);
}

class StartConversationResult {
  final String conversationUUID;
  final bool isExisting;  // True if conversation already existed
  
  StartConversationResult({required this.conversationUUID, required this.isExisting});
}
```

### 2. List Conversations

```dart
/// Get paginated list of conversations
Future<List<Conversation>> getConversations({int limit = 20, int offset = 0}) async {
  final response = await api.call('API_MessageConversationList', 
    p3: [limit.toString(), offset.toString()]);
  
  if (response.p1 == 'Success') {
    final List<dynamic> data = jsonDecode(response.p3[0]);
    return data.map((c) => Conversation.fromJson(c)).toList();
  }
  throw Exception(response.p2);
}

class Conversation {
  final String uuid;
  final int conversationType;  // 1 = direct, 2 = group
  final String? title;
  final String createdBy;
  final int createdAt;
  final int updatedAt;
  final int? lastMessageAt;
  final String? lastMessagePreview;
  final int messageCount;
  final int unreadCount;
  final String? otherUserUUID;    // For 1:1 conversations
  final String? otherUserName;
  final String? otherUserPicUrl;
  final List<ParticipantInfo> participants;
  
  // Encryption fields - included in conversation list/get responses
  final String sessionKeyEnc;        // Encrypted session key for this user
  final String ephemeralPublicKey;   // Server's ephemeral key used for encryption
  final String? recipientPubkeyHash; // Hash of pubkey used to encrypt session key (null for legacy)
  final bool isDecryptable;          // True if user's current key can decrypt this conversation
  
  Conversation.fromJson(Map<String, dynamic> json) : 
    uuid = json['uuid'],
    conversationType = json['conversationType'],
    sessionKeyEnc = json['sessionKeyEnc'] ?? '',
    ephemeralPublicKey = json['ephemeralPublicKey'] ?? '',
    recipientPubkeyHash = json['recipientPubkeyHash'],
    isDecryptable = json['isDecryptable'] ?? true,
    // ... etc
}
```

> **Key Recovery Support:** Conversations now include `isDecryptable` and `recipientPubkeyHash` fields.
>
> * If `isDecryptable == false`, the conversation was encrypted with a different key
> * Client should show a "locked" state and prompt user to import old keys
> * When user imports their old key via `API_MessageInitKeys`, matching conversations become decryptable again
> * Multiple conversations with the same person are allowed if keys have changed

```

### 3. Get Session Key (Required for Encryption/Decryption)

```dart
/// Get the encrypted session key for a conversation
Future<String> getSessionKey(String conversationUUID) async {
  final response = await api.call('API_MessageSessionKey', p3: [conversationUUID]);
  
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    // The sessionKeyEnc contains: ephemeral_public_key (32) + nonce (24) + ciphertext
    return data['sessionKeyEnc'] as String;
  }
  throw Exception(response.p2);
}
```

### 4. Decrypt Session Key

```dart
/// Decrypt the session key using your private key
/// The sessionKeyEnc format is: ephemeral_public_key (32) + nonce (24) + ciphertext (48)
/// Total: 104 bytes
Future<List<int>> decryptSessionKey(String sessionKeyEnc, List<int> myPrivateKey) async {
  final x25519 = X25519();
  
  // Decode the encrypted session key blob
  final encryptedData = MercuryKeys.decodeKey(sessionKeyEnc);
  
  if (encryptedData.length != 104) {
    throw Exception('Invalid sessionKeyEnc length: ${encryptedData.length}, expected 104');
  }
  
  // Extract components: ephemeral_public_key (32) + nonce (24) + ciphertext (48)
  final ephemeralPubKey = encryptedData.sublist(0, 32);
  final nonce = encryptedData.sublist(32, 56);
  final ciphertext = encryptedData.sublist(56);  // 48 bytes = 32 plaintext + 16 MAC
  
  // IMPORTANT: Reconstruct proper key pair from private key bytes
  // Using newKeyPairFromSeed ensures proper X25519 handling including clamping
  final myKeyPair = await x25519.newKeyPairFromSeed(myPrivateKey);
  
  // Derive shared secret using ECDH: sharedSecret = X25519(myPrivateKey, ephemeralPubKey)
  final sharedSecret = await x25519.sharedSecretKey(
    keyPair: myKeyPair,
    remotePublicKey: SimplePublicKey(ephemeralPubKey, type: KeyPairType.x25519),
  );
  
  // Decrypt session key using XChaCha20-Poly1305
  // The ciphertext includes the 16-byte Poly1305 MAC appended at the end
  final algorithm = Xchacha20.poly1305Aead();
  
  // Split ciphertext: first 32 bytes = encrypted data, last 16 bytes = MAC
  final actualCiphertext = ciphertext.sublist(0, ciphertext.length - 16);  // 32 bytes
  final mac = Mac(ciphertext.sublist(ciphertext.length - 16));  // 16 bytes
  
  final secretBox = SecretBox(actualCiphertext, nonce: nonce, mac: mac);
  
  final sessionKey = await algorithm.decrypt(
    secretBox,
    secretKey: sharedSecret,
  );
  
  if (sessionKey.length != 32) {
    throw Exception('Decrypted session key wrong length: ${sessionKey.length}, expected 32');
  }
  
  return sessionKey;  // 32 bytes
}
```

> **Note on XChaCha20-Poly1305 ciphertext format:**
> The server's `chacha20poly1305.Seal()` appends the 16-byte MAC to the ciphertext.
> Total ciphertext length = plaintext length + 16 bytes.
> For a 32-byte session key: ciphertext = 48 bytes (32 + 16 MAC).

### Debug Helper: Verify Shared Secret

If decryption fails with `SecretBoxAuthenticationError`, use this to debug the shared secret:

```dart
import 'package:crypto/crypto.dart';  // For sha256

/// Debug helper to verify key agreement matches server
Future<void> debugSessionKeyDecryption(
  String sessionKeyEnc, 
  List<int> myPrivateKey,
  String myPublicKeyB64,  // Your stored public key for comparison
) async {
  final x25519 = X25519();
  final encryptedData = MercuryKeys.decodeKey(sessionKeyEnc);
  
  print('[Mercury Debug] sessionKeyEnc base64 length: ${sessionKeyEnc.length}');
  print('[Mercury Debug] decoded blob length: ${encryptedData.length} (expected 104)');
  
  final ephemeralPubKey = encryptedData.sublist(0, 32);
  final nonce = encryptedData.sublist(32, 56);
  final ciphertext = encryptedData.sublist(56);
  
  print('[Mercury Debug] ephemeralPubKey length: ${ephemeralPubKey.length}');
  print('[Mercury Debug] nonce length: ${nonce.length}');
  print('[Mercury Debug] ciphertext length: ${ciphertext.length} (expected 48)');
  print('[Mercury Debug] ephemeralPubKey sha256: ${sha256.convert(ephemeralPubKey)}');
  print('[Mercury Debug] nonce sha256: ${sha256.convert(nonce)}');
  print('[Mercury Debug] ciphertext sha256: ${sha256.convert(ciphertext)}');
  
  // Reconstruct key pair from seed
  final myKeyPair = await x25519.newKeyPairFromSeed(myPrivateKey);
  final myPubKey = await myKeyPair.extractPublicKey();
  
  print('[Mercury Debug] myPrivateKey length: ${myPrivateKey.length}');
  print('[Mercury Debug] myPublicKey from keyPair: ${base64.encode(myPubKey.bytes)}');
  print('[Mercury Debug] myPublicKey stored: $myPublicKeyB64');
  print('[Mercury Debug] Public keys match: ${base64.encode(myPubKey.bytes) == myPublicKeyB64}');
  
  // Compute shared secret
  final sharedSecret = await x25519.sharedSecretKey(
    keyPair: myKeyPair,
    remotePublicKey: SimplePublicKey(ephemeralPubKey, type: KeyPairType.x25519),
  );
  final sharedSecretBytes = await sharedSecret.extractBytes();
  print('[Mercury Debug] sharedSecret sha256: ${sha256.convert(sharedSecretBytes)}');
  
  // Compare this with server log (MERCURY_CRYPTO_DEBUG=1):
  // Look for: sharedSecretSha256=<hex>
  // If they don't match, the X25519 key agreement failed (wrong private key or encoding issue)
}
```

**To enable server-side debug logging:** Set environment variable `MERCURY_CRYPTO_DEBUG=1` and restart the server. The server will log SHA-256 hashes of cryptographic values during encryption.


---

## Sending Messages

### 1. Encrypt and Send a Message

```dart
/// Send an encrypted message
Future<SendMessageResult> sendMessage({
  required String conversationUUID,
  required String plaintext,
  required List<int> sessionKey,
  int messageType = 1,  // 1=text, 2=image, 3=file, 4=system
  String? replyToUUID,
  Map<String, dynamic>? metadata,
}) async {
  // Generate random 24-byte nonce
  final nonce = _generateNonce();  // Must be 24 bytes for XChaCha20
  
  // Encrypt message content with XChaCha20-Poly1305
  final algorithm = Xchacha20.poly1305Aead();
  final secretKey = SecretKey(sessionKey);
  
  final secretBox = await algorithm.encrypt(
    utf8.encode(plaintext),
    secretKey: secretKey,
    nonce: nonce,
  );
  
  // Combine ciphertext + MAC (server expects them concatenated)
  final contentEnc = base64.encode([...secretBox.cipherText, ...secretBox.mac.bytes]);
  final contentNonce = base64.encode(nonce);
  
  // Encrypt metadata if provided
  String? metadataEnc;
  if (metadata != null) {
    final metaBox = await algorithm.encrypt(
      utf8.encode(jsonEncode(metadata)),
      secretKey: secretKey,
      nonce: _generateNonce(),
    );
    metadataEnc = base64.encode([...metaBox.nonce, ...metaBox.cipherText, ...metaBox.mac.bytes]);
  }
  
  // Send to server
  final response = await api.call('API_MessageSend', p3: [
    conversationUUID,
    messageType.toString(),
    contentEnc,
    contentNonce,
    replyToUUID ?? '',
    metadataEnc ?? '',
  ]);
  
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    return SendMessageResult(
      success: true,
      messageUUID: data['messageUuid'],
      createdAt: data['createdAt'],
    );
  }
  throw Exception(response.p2);
}

List<int> _generateNonce() {
  final random = Random.secure();
  return List.generate(24, (_) => random.nextInt(256));
}
```

### 2. Get and Decrypt Messages

```dart
/// Get messages from a conversation
Future<List<Message>> getMessages({
  required String conversationUUID,
  required List<int> sessionKey,
  int limit = 50,
  int offset = 0,
  int? beforeTimestamp,
}) async {
  final response = await api.call('API_MessageGetMessages', p3: [
    conversationUUID,
    limit.toString(),
    offset.toString(),
    beforeTimestamp?.toString() ?? '',
  ]);
  
  if (response.p1 == 'Success') {
    final List<dynamic> data = jsonDecode(response.p3[0]);
    
    // Decrypt each message
    final messages = <Message>[];
    for (final m in data) {
      final decrypted = await _decryptMessage(m, sessionKey);
      messages.add(decrypted);
    }
    return messages;
  }
  throw Exception(response.p2);
}

/// Decrypt a single message
Future<Message> _decryptMessage(Map<String, dynamic> encrypted, List<int> sessionKey) async {
  final contentEnc = base64.decode(encrypted['contentEnc']);
  final nonce = base64.decode(encrypted['contentNonce']);
  
  // Split ciphertext and MAC (last 16 bytes is MAC)
  final ciphertext = contentEnc.sublist(0, contentEnc.length - 16);
  final mac = Mac(contentEnc.sublist(contentEnc.length - 16));
  
  final algorithm = Xchacha20.poly1305Aead();
  final secretBox = SecretBox(ciphertext, nonce: nonce, mac: mac);
  
  final plaintext = await algorithm.decrypt(
    secretBox,
    secretKey: SecretKey(sessionKey),
  );
  
  return Message(
    uuid: encrypted['uuid'],
    conversationUUID: encrypted['conversationUuid'],
    senderUUID: encrypted['senderUuid'],
    messageType: encrypted['messageType'],
    content: utf8.decode(plaintext),  // Decrypted content
    replyToUUID: encrypted['replyToUuid'],
    createdAt: encrypted['createdAt'],
    editedAt: encrypted['editedAt'],
    deleted: encrypted['deleted'] ?? false,
  );
}
```


---

## Read Receipts

### 1. Record a Read Receipt

```dart
/// Mark a message as read
Future<void> markMessageRead(String messageUUID) async {
  await api.call('API_MessageReceipt', p3: [
    messageUUID,
    'true',  // isRead = true
  ]);
}

/// Mark a message as received (delivered)
Future<void> markMessageReceived(String messageUUID) async {
  await api.call('API_MessageReceipt', p3: [
    messageUUID,
    'false',  // isRead = false (just received)
  ]);
}
```

### 2. Mark Entire Conversation as Read

```dart
Future<void> markConversationRead(String conversationUUID) async {
  await api.call('API_MessageConversationRead', p3: [conversationUUID]);
}
```


---

## User Preferences

### 1. Get Preferences

```dart
Future<MessagePrefs> getMessagePrefs() async {
  final response = await api.call('API_MessageGetPrefs');
  
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    return MessagePrefs(
      allowMessages: data['allowMessages'] ?? 'all',  // "all", "followers", "none"
      messageNotifications: data['messageNotifications'] ?? true,
      readReceipts: data['readReceipts'] ?? true,
    );
  }
  throw Exception(response.p2);
}

class MessagePrefs {
  final String allowMessages;        // "all" | "followers" | "none"
  final bool messageNotifications;   // Show push notifications
  final bool readReceipts;           // Show read receipts to others
  
  MessagePrefs({
    required this.allowMessages,
    required this.messageNotifications,
    required this.readReceipts,
  });
  
  Map<String, dynamic> toJson() => {
    'allowMessages': allowMessages,
    'messageNotifications': messageNotifications,
    'readReceipts': readReceipts,
  };
}
```

### 2. Update Preferences

```dart
Future<void> setMessagePrefs(MessagePrefs prefs) async {
  final response = await api.call('API_MessageSetPrefs', 
    p3: [jsonEncode(prefs.toJson())]);
  
  if (response.p1 != 'Success') {
    throw Exception(response.p2);
  }
}

// Example: Only allow messages from people I follow
await setMessagePrefs(MessagePrefs(
  allowMessages: 'followers',
  messageNotifications: true,
  readReceipts: true,
));
```

### 3. Check if Can Message User

```dart
Future<CanMessageResult> canMessageUser(String targetUUID) async {
  final response = await api.call('API_MessageCanMessage', p3: [targetUUID]);
  
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    return CanMessageResult(
      canMessage: data['canMessage'],
      reason: data['reason'],  // "blocked", "followers_only", "messages_disabled", etc.
    );
  }
  throw Exception(response.p2);
}
```


---

## Blocking Users

```dart
/// Block a user
Future<void> blockUser(String userUUID, {String? reason}) async {
  await api.call('API_MessageBlockUser', p3: [userUUID, reason ?? '']);
}

/// Unblock a user
Future<void> unblockUser(String userUUID) async {
  await api.call('API_MessageUnblockUser', p3: [userUUID]);
}

/// Get list of blocked users
Future<List<String>> getBlockedUsers() async {
  final response = await api.call('API_MessageGetBlocked');
  if (response.p1 == 'Success') {
    return List<String>.from(jsonDecode(response.p3[0]));
  }
  throw Exception(response.p2);
}

/// Check if a user is blocked
Future<bool> isUserBlocked(String userUUID) async {
  final response = await api.call('API_MessageIsBlocked', p3: [userUUID]);
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    return data['isBlocked'] ?? false;
  }
  throw Exception(response.p2);
}
```


---

## Forward Secrecy (Key Rotation)

### 1. Rotate Session Key

Call periodically (e.g., every 100 messages or every 24 hours) for forward secrecy:

```dart
/// Rotate the session key for a conversation
Future<String> rotateSessionKey(String conversationUUID) async {
  final response = await api.call('API_MessageRotateKey', p3: [conversationUUID]);
  
  if (response.p1 == 'Success') {
    final data = jsonDecode(response.p3[0]);
    // Update cached session key
    return data['newSessionKeyEnc'];
  }
  throw Exception(response.p2);
}
```

### 2. Get Key History (For Decrypting Old Messages)

```dart
/// Get all session keys for a conversation (for decrypting old messages)
Future<List<SessionKeyInfo>> getKeyHistory(String conversationUUID) async {
  final response = await api.call('API_MessageKeyHistory', p3: [conversationUUID]);
  
  if (response.p1 == 'Success') {
    final List<dynamic> data = jsonDecode(response.p3[0]);
    return data.map((k) => SessionKeyInfo(
      sessionKeyEnc: k['sessionKeyEnc'],
      ephemeralPublicKey: k['ephemeralPublicKey'],
    )).toList();
  }
  throw Exception(response.p2);
}
```


---

## WebSocket Notifications

Listen for real-time messaging events via WebSocket:

### SAPI Event Types

| Event | Description | P2 Data |
|----|----|----|
| `SAPI_NewMessage` | New message received | `{messageUuid, conversationUuid, senderUuid, senderName, messageType, createdAt}` |
| `SAPI_MessageDeleted` | Message was deleted | `{messageUuid, conversationUuid}` |
| `SAPI_MessageEdited` | Message was edited | `{messageUuid, conversationUuid, editedAt}` |
| `SAPI_ReadReceipt` | Your message was read | `{messageUuid, readerUuid, readAt}` |
| `SAPI_ConversationStarted` | Added to new conversation | `{conversationUuid, initiatorUuid, initiatorName}` |

### Handle WebSocket Events

```dart
void handleWebSocketMessage(Map<String, dynamic> message) {
  final req = message['req'];
  final p2 = message['p2'];
  
  switch (req) {
    case 'SAPI_NewMessage':
      final data = jsonDecode(p2);
      _onNewMessage(
        messageUUID: data['messageUuid'],
        conversationUUID: data['conversationUuid'],
        senderUUID: data['senderUuid'],
        senderName: data['senderName'],
        messageType: data['messageType'],
        createdAt: data['createdAt'],
      );
      break;
      
    case 'SAPI_MessageDeleted':
      final data = jsonDecode(p2);
      _onMessageDeleted(data['messageUuid'], data['conversationUuid']);
      break;
      
    case 'SAPI_MessageEdited':
      final data = jsonDecode(p2);
      _onMessageEdited(data['messageUuid'], data['conversationUuid'], data['editedAt']);
      break;
      
    case 'SAPI_ReadReceipt':
      final data = jsonDecode(p2);
      _onReadReceipt(data['messageUuid'], data['readerUuid'], data['readAt']);
      break;
      
    case 'SAPI_ConversationStarted':
      final data = jsonDecode(p2);
      _onConversationStarted(data['conversationUuid'], data['initiatorUuid'], data['initiatorName']);
      break;
  }
}
```


---

## Complete API Reference (Full RObj Structures)

All Mercury APIs use the standard LexCV RObj protocol:

**Request:** `{ req: "API_Name", p1: "", p2: "", p3: [...], p4: [], p5: [], p6: [] }`

**Response:** `{ req: "API_Name", p1: "Success|Fail", p2: "error message", p3: [...], ... }`


---

### Key Management APIs

#### `API_MessageInitKeys`

Initialize or update user's encryption keys. **MUST be called before messaging can be used.**

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageInitKeys"` |
|    | `p3[0]` | Base64-encoded X25519 public key (44 chars) |
|    | `p3[1]` | Base64-encoded encrypted private key (client-encrypted with password) |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error message if failed |


---

#### `API_MessageGetKeys`

Get own encryption keys (for backup/recovery UI).

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageGetKeys"` |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON: `{"publicKey": "...", "hasPrivateKey": true, "privateKeyEnc": "..."}` |

**Response JSON fields:**

| Field | Type | Description |
|----|----|----|
| `publicKey` | string | Base64 X25519 public key |
| `hasPrivateKey` | bool | True if encrypted private key is stored |
| `privateKeyEnc` | string | Base64 encrypted private key (for backup) |


---

#### `API_MessageGetUserPublicKey`

Get another user's public key for encryption.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageGetUserPublicKey"` |
|    | `p3[0]` | Target user UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON: `{"publicKey": "..."}` |


---

### Conversation APIs

#### `API_MessageConversationStart`

Start a new 1:1 conversation or get existing one.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageConversationStart"` |
|    | `p3[0]` | Recipient user UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error: `"initiator_keys_not_initialized"`, `"recipient_keys_not_initialized"`, `"recipient_not_found"`, `"blocked"` |
|    | `p3[0]` | JSON: `{"conversationUuid": "...", "isExisting": false}` |


---

#### `API_MessageConversationList`

Get paginated list of conversations.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageConversationList"` |
|    | `p3[0]` | Limit (optional, default 20, max from config) |
|    | `p3[1]` | Offset (optional, default 0) |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON array of Conversation objects (see below) |


---

#### `API_MessageConversationGet`

Get a single conversation with full details.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageConversationGet"` |
|    | `p3[0]` | Conversation UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error: `"not a participant"` |
|    | `p3[0]` | JSON Conversation object (see below) |


---

#### `API_MessageConversationDelete`

Soft-delete a conversation (user won't see it anymore).

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageConversationDelete"` |
|    | `p3[0]` | Conversation UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error: `"not a participant"` |


---

#### `API_MessageConversationMute`

Mute/unmute a conversation.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageConversationMute"` |
|    | `p3[0]` | Conversation UUID |
|    | `p3[1]` | Muted until: `"0"` = forever, `"-1"` = unmute, timestamp = until |
| **Response** | `p1` | `"Success"` or `"Fail"` |


---

#### `API_MessageConversationRead`

Mark entire conversation as read.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageConversationRead"` |
|    | `p3[0]` | Conversation UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |


---

#### `API_MessageSessionKey`

Get the encrypted session key for a conversation.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageSessionKey"` |
|    | `p3[0]` | Conversation UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON: `{"sessionKeyEnc": "..."}` |

> **Note:** The 104-byte session key blob contains: ephemeral public key (32) + nonce (24) + ciphertext (48)


---

#### `API_MessageKeyHistory`

Get all session keys for a conversation (for decrypting messages after key rotation).

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageKeyHistory"` |
|    | `p3[0]` | Conversation UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON array: `[{"sessionKeyEnc": "...", "ephemeralPublicKey": "...", "current": true}]` |


---

#### `API_MessageRotateKey`

Rotate the session key for forward secrecy.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageRotateKey"` |
|    | `p3[0]` | Conversation UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON: `{"success": true, "newSessionKeyEnc": "..."}` |


---

### Message APIs

#### `API_MessageSend`

Send an encrypted message.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageSend"` |
|    | `p3[0]` | Conversation UUID |
|    | `p3[1]` | Message type: `"1"` = text, `"2"` = image, `"3"` = file, `"4"` = system |
|    | `p3[2]` | Base64 encrypted content (ciphertext + 16-byte MAC) |
|    | `p3[3]` | Base64 nonce (24 bytes) |
|    | `p3[4]` | Reply-to message UUID (optional, `""` if none) |
|    | `p3[5]` | Base64 encrypted metadata JSON (optional, `""` if none) |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error: `"not a participant"`, `"blocked by recipient"` |
|    | `p3[0]` | JSON: `{"messageUuid": "...", "createdAt": "1702732800000"}` |


---

#### `API_MessageGetMessages`

Get paginated messages from a conversation.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageGetMessages"` |
|    | `p3[0]` | Conversation UUID |
|    | `p3[1]` | Limit (optional, default 50, max from config) |
|    | `p3[2]` | Offset (optional, default 0) |
|    | `p3[3]` | Before timestamp (optional, for pagination) |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON array of Message objects (see below) |


---

#### `API_MessageGetMessage`

Get a single message.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageGetMessage"` |
|    | `p3[0]` | Message UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON Message object (see below) |


---

#### `API_MessageDeleteMessage`

Delete a message (soft-delete, shows "deleted message" to others).

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageDeleteMessage"` |
|    | `p3[0]` | Message UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error: `"only sender can delete message"`, `"message not found"` |


---

#### `API_MessageEditMessage`

Edit an encrypted message (within 24 hours).

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageEditMessage"` |
|    | `p3[0]` | Message UUID |
|    | `p3[1]` | New Base64 encrypted content |
|    | `p3[2]` | New Base64 nonce |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error: `"only sender can edit message"`, `"message too old to edit"` |


---

#### `API_MessageReceipt`

Record a message receipt (delivered or read).

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageReceipt"` |
|    | `p3[0]` | Message UUID |
|    | `p3[1]` | `"true"` = read, `"false"` = delivered only |
| **Response** | `p1` | `"Success"` or `"Fail"` |


---

### User Preferences APIs

#### `API_MessageGetPrefs`

Get user's messaging preferences.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageGetPrefs"` |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON: `{"allowMessages": "all", "messageNotifications": true, "readReceipts": true}` |

**Response JSON fields:**

| Field | Type | Values | Description |
|----|----|----|----|
| `allowMessages` | string | `"all"`, `"followers"`, `"none"` | Who can message this user |
| `messageNotifications` | bool |    | Push notifications enabled |
| `readReceipts` | bool |    | Show read receipts to others |


---

#### `API_MessageSetPrefs`

Update user's messaging preferences.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageSetPrefs"` |
|    | `p3[0]` | JSON object with preferences to update |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error: `"invalid JSON: ..."` |


---

#### `API_MessageCanMessage`

Check if current user can message another user.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageCanMessage"` |
|    | `p3[0]` | Target user UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON: `{"canMessage": true, "reason": ""}` |

**Possible** `reason` values when `canMessage` is false:

| Reason | Meaning |
|----|----|
| `"blocked"` | User is blocked or has blocked you |
| `"messages_disabled"` | User has set `allowMessages: "none"` |
| `"followers_only"` | User only accepts messages from followers |


---

### Blocking APIs

#### `API_MessageBlockUser`

Block a user from messaging.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageBlockUser"` |
|    | `p3[0]` | User UUID to block |
|    | `p3[1]` | Reason (optional) |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p2` | Error: `"cannot block yourself"` |


---

#### `API_MessageUnblockUser`

Unblock a previously blocked user.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageUnblockUser"` |
|    | `p3[0]` | User UUID to unblock |
| **Response** | `p1` | `"Success"` or `"Fail"` |


---

#### `API_MessageGetBlocked`

Get list of blocked users with profile details.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageGetBlocked"` |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON array of BlockedUserInfo objects (see below) |


---

#### `API_MessageIsBlocked`

Check block status between current user and another user.

| Direction | Field | Value |
|----|----|----|
| **Request** | `req` | `"API_MessageIsBlocked"` |
|    | `p3[0]` | Other user UUID |
| **Response** | `p1` | `"Success"` or `"Fail"` |
|    | `p3[0]` | JSON: `{"isBlocked": false, "blockedByMe": false, "blockedByThem": false}` |


---

## Response Object Schemas

### Conversation Object

Returned by `API_MessageConversationList` (array) and `API_MessageConversationGet` (single):

```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "conversationType": 1,
  "title": null,
  "createdBy": "user-uuid",
  "createdAt": "1702732800000",
  "updatedAt": "1702732900000",
  "lastMessageAt": "1702732900000",
  "lastMessagePreview": null,
  "messageCount": 15,
  "unreadCount": 3,
  "otherUserUUID": "other-user-uuid",
  "otherUserName": "John Doe",
  "otherUserPicUrl": "https://cdn.lexcv.co/...",
  "participants": [
    {
      "uuid": "other-user-uuid",
      "name": "John Doe",
      "picUrl": "https://cdn.lexcv.co/..."
    }
  ],
  "sessionKeyEnc": "base64...",
  "ephemeralPublicKey": "base64...",
  "recipientPubkeyHash": "abcd1234...",
  "isDecryptable": true
}
```

| Field | Type | Description |
|----|----|----|
| `uuid` | string | Conversation UUID |
| `conversationType` | int | `1` = direct (1:1), `2` = group (future) |
| `title` | string? | Optional title (for groups) |
| `createdBy` | string | UUID of user who started conversation |
| `createdAt` | string | Unix timestamp (ms) |
| `updatedAt` | string | Last activity timestamp (ms) |
| `lastMessageAt` | string? | Timestamp of last message (ms) |
| `lastMessagePreview` | string? | Preview text (encrypted, may be null) |
| `messageCount` | int | Total messages in conversation |
| `unreadCount` | int | Unread messages for this user |
| `otherUserUUID` | string? | For 1:1: the other user's UUID |
| `otherUserName` | string? | For 1:1: the other user's display name |
| `otherUserPicUrl` | string? | For 1:1: signed CDN URL for avatar |
| `participants` | array | List of other participants (see Participant) |
| `sessionKeyEnc` | string | Encrypted session key (104 bytes when decoded) |
| `ephemeralPublicKey` | string | Server's ephemeral X25519 public key |
| `recipientPubkeyHash` | string? | SHA-256 hash prefix (32 hex chars) of pubkey used for encryption |
| `isDecryptable` | bool | `true` if current user's keys can decrypt |

### Participant Object

Found in `participants` array:

| Field | Type | Description |
|----|----|----|
| `uuid` | string | User UUID |
| `name` | string | Full name (first + last) |
| `picUrl` | string? | Signed CDN URL for avatar (time-limited) |

### Message Object

Returned by `API_MessageGetMessages` (array) and `API_MessageGetMessage` (single):

```json
{
  "uuid": "msg-uuid-here",
  "conversationUuid": "conv-uuid-here",
  "senderUuid": "sender-uuid",
  "messageType": 1,
  "contentEnc": "base64-encrypted-content...",
  "contentNonce": "base64-nonce...",
  "ratchetKey": null,
  "replyToUuid": null,
  "metadataEnc": null,
  "createdAt": "1702732800000",
  "editedAt": "0",
  "deleted": false
}
```

| Field | Type | Description |
|----|----|----|
| `uuid` | string | Message UUID |
| `conversationUuid` | string | Parent conversation UUID |
| `senderUuid` | string | Sender's user UUID |
| `messageType` | int | `1` = text, `2` = image, `3` = file, `4` = system |
| `contentEnc` | string | Base64 encrypted content (ciphertext + MAC) |
| `contentNonce` | string | Base64 XChaCha20 nonce (24 bytes) |
| `ratchetKey` | string? | New session key if rotated with this message |
| `replyToUuid` | string? | UUID of message being replied to |
| `metadataEnc` | string? | Base64 encrypted metadata (file info, etc.) |
| `createdAt` | string | Unix timestamp (ms) |
| `editedAt` | string | Edit timestamp (ms), `"0"` if never edited |
| `deleted` | bool | `true` if sender deleted the message |

### BlockedUserInfo Object

Returned by `API_MessageGetBlocked` (array):

```json
{
  "uuid": "blocked-user-uuid",
  "fullName": "Jane Smith",
  "headline": "Partner at Law Firm",
  "smallPicUrl": "https://cdn.lexcv.co/...",
  "smallPicToken": "token-for-refresh",
  "blockedAt": "1702732800000",
  "reason": ""
}
```

| Field | Type | Description |
|----|----|----|
| `uuid` | string | Blocked user's UUID |
| `fullName` | string | User's display name |
| `headline` | string | User's headline/title |
| `smallPicUrl` | string | Signed CDN URL for small avatar |
| `smallPicToken` | string | Token for URL refresh |
| `blockedAt` | string | Unix timestamp (ms) when blocked |
| `reason` | string | Optional reason provided when blocking |


---

## Messaging Status in Search/List Results

User search and list APIs include messaging status fields to enable pre-flight UI checks:

### APIs That Include Messaging Status

| API | Where Messaging Status Appears |
|----|----|
| `API_SearchMain` | Each result item |
| `API_SearchAuto` | Each result item |
| `API_ProfileFollowersList` | Each user item |
| `API_ProfileFollowingList` | Each user item |
| `API_ProfileShortlistList` | Each user item |

### Item Format (P6 Array Items)

Each user in search/list results is an array with this structure:

```
Index:  [0]    [1]       [2]       [3]         [4]        [5]       [6]              [7]             [8]            [9]            [10]          [11]                    [12]
Field:  uuid   fullname  headline  smallPicUrl  flagsJSON  position  followByViewer  followsViewer   shortlistFlag  shortlistRank  smallPicToken  messageKeysInitialized  allowMessages
```

### Messaging-Specific Fields

| Index | Field | Type | Description |
|----|----|----|----|
| 11 | `messageKeysInitialized` | string | `"1"` if user has initialized messaging keys, `"0"` if not |
| 12 | `allowMessages` | string | `"all"`, `"followers"`, or `"none"` |

### Example Usage (Dart)

```dart
class UserSearchResult {
  final String uuid;
  final String fullname;
  final String headline;
  final String smallPicUrl;
  final bool canMessage;
  final String messageRestriction;
  
  factory UserSearchResult.fromP6Item(List<String> item) {
    final keysInitialized = item.length > 11 ? item[11] == "1" : false;
    final allowMessages = item.length > 12 ? item[12] : "all";
    
    return UserSearchResult(
      uuid: item[0],
      fullname: item[1],
      headline: item[2],
      smallPicUrl: item[3],
      // Pre-check: can we even start a conversation?
      canMessage: keysInitialized && allowMessages != "none",
      messageRestriction: allowMessages,
    );
  }
  
  /// Show appropriate UI based on messaging status
  Widget buildMessageButton() {
    if (!canMessage) {
      if (messageRestriction == "none") {
        return Tooltip(
          message: "This user has disabled messages",
          child: Icon(Icons.message_outlined, color: Colors.grey),
        );
      }
      return Tooltip(
        message: "This user hasn't set up messaging yet",
        child: Icon(Icons.message_outlined, color: Colors.grey),
      );
    }
    if (messageRestriction == "followers") {
      return Tooltip(
        message: "Only accepts messages from followers",
        child: Icon(Icons.message, color: Colors.blue),
      );
    }
    return Icon(Icons.message, color: Colors.blue);
  }
}
```

### Decision Matrix for Message Button UI

| `messageKeysInitialized` | `allowMessages` | UI Action |
|----|----|----|
| `"0"` | any | Show disabled "Set up messaging first" |
| `"1"` | `"none"` | Show disabled "Messages disabled" |
| `"1"` | `"followers"` | Show enabled with tooltip "Followers only" |
| `"1"` | `"all"` | Show enabled, no restrictions |


---

## Key Management & Recovery

### How Keys Work


1. **Client generates key pair** - X25519 private/public key pair generated on device
2. **Private key encrypted** - Client encrypts private key with user's password/PIN before sending
3. **Server stores both** - Public key (plaintext) + encrypted private key
4. **Session keys encrypted to public key** - Each conversation has a session key encrypted specifically to each participant's public key

### Key Loss Scenario

If a user loses their local keys (app reinstall, device change, storage cleared):


1. **Old conversations become undecryptable** - Session keys were encrypted to the old public key
2. **User can still start NEW conversations** - The server allows multiple conversations with the same person
3. **Old conversations are preserved** - They remain in the database for potential recovery

### Handling `isDecryptable` in the Client

```dart
Widget buildConversationTile(Conversation conv) {
  if (!conv.isDecryptable) {
    // Show locked state - user needs to import old keys
    return ListTile(
      leading: Icon(Icons.lock, color: Colors.orange),
      title: Text(conv.otherUserName ?? 'Unknown'),
      subtitle: Text('🔒 Encrypted with different keys'),
      onTap: () => showKeyRecoveryDialog(conv),
    );
  }
  // Normal conversation tile
  return ListTile(
    title: Text(conv.otherUserName ?? 'Unknown'),
    subtitle: Text(conv.lastMessagePreview ?? ''),
    onTap: () => openConversation(conv),
  );
}

void showKeyRecoveryDialog(Conversation conv) {
  showDialog(
    context: context,
    builder: (ctx) => AlertDialog(
      title: Text('Conversation Locked'),
      content: Text(
        'This conversation was created with different encryption keys. '
        'To access it, import your old keys from a backup. '
        'Alternatively, start a new conversation with this person.'
      ),
      actions: [
        TextButton(
          onPressed: () => importOldKeys(),
          child: Text('Import Keys'),
        ),
        TextButton(
          onPressed: () => startNewConversation(conv.otherUserUUID!),
          child: Text('Start New'),
        ),
      ],
    ),
  );
}
```

### Importing Old Keys

When user imports old keys, call `API_MessageInitKeys` with the old key pair:

```dart
Future<void> restoreOldKeys(List<int> oldPrivateKey) async {
  // Derive public key from private key
  final x25519 = X25519();
  final keyPair = await x25519.newKeyPairFromSeed(oldPrivateKey);
  final publicKey = await keyPair.extractPublicKey();
  
  final publicKeyB64 = base64.encode(publicKey.bytes);
  
  // Encrypt private key for storage
  final privateKeyEncB64 = await encryptPrivateKey(oldPrivateKey, userPassword);
  
  // Register with server - this updates the user's current key
  await api.call('API_MessageInitKeys', p3: [publicKeyB64, privateKeyEncB64]);
  
  // Refresh conversation list - old conversations will now show isDecryptable: true
  await refreshConversations();
}
```

### Multiple Conversations with Same Person

The server allows multiple 1:1 conversations with the same person if:

* The previous conversation was encrypted with a **different** public key
* This enables users to start fresh after key changes without being locked out

When starting a conversation (`API_MessageConversationStart`):

* Server checks for existing conversation encrypted to user's **current** public key
* If found: returns existing conversation (`isExisting: true`)
* If not found (keys changed or no prior conversation): creates new conversation


---

## Security Best Practices


1. **Never log or print decrypted messages or keys**
2. **Store private key encrypted** with user's password/PIN using secure storage
3. **Clear session keys from memory** when user logs out
4. **Rotate keys periodically** - call `API_MessageRotateKey` every 100 messages or 24 hours
5. **Validate message integrity** - XChaCha20-Poly1305 provides authentication
6. **Use secure random** for all nonce generation
7. **Re-fetch session key** after rotation notification
8. **Backup keys securely** - Allow users to export encrypted key backup for recovery
9. **Check** `isDecryptable` - Handle locked conversations gracefully in UI


---

## Error Handling

Common error responses in `P2`:

### General Errors

| Error | Meaning |
|----|----|
| `not a participant` | User is not in this conversation |
| `not logged in` | Authentication required |

### Blocking Errors

| Error | Meaning | User-Facing Message |
|----|----|----|
| `blocked` | Recipient has blocked you | "You cannot message this user" |
| `blocked by recipient` | Recipient has blocked you (in conversation) | "This user has blocked you" |
| `you_blocked_user` | You have blocked this user | "You have blocked this user" |
| `cannot block yourself` | Self-block attempted | "You cannot block yourself" |

### Preference Errors

| Error | Meaning | User-Facing Message |
|----|----|----|
| `messages_disabled` | User has disabled all messages | "This user has disabled messages" |
| `followers_only` | User only accepts from followers | "This user only accepts messages from people they follow" |

### Message Errors

| Error | Meaning |
|----|----|
| `recipient has no encryption keys` | User hasn't set up messaging yet |
| `message not found` | Message UUID doesn't exist |
| `only sender can delete message` | Not authorized to delete |
| `only sender can edit message` | Not authorized to edit |
| `message too old to edit` | Can only edit within 24 hours |

### Key Errors

| Error | Meaning |
|----|----|
| `initiator_keys_not_initialized` | You haven't initialized encryption keys yet |
| `recipient_keys_not_initialized` | Recipient hasn't initialized encryption keys yet |
| `user has no keys` | User hasn't initialized encryption |
| `recipient_not_found` | Recipient user does not exist |

### Error Handling Example

```dart
class MessagingError {
  final String code;
  final String? userMessage;
  
  MessagingError(this.code, {this.userMessage});
  
  static MessagingError fromResponse(String error) {
    final userMessages = {
      'blocked': 'You cannot message this user',
      'blocked by recipient': 'This user has blocked you',
      'you_blocked_user': 'You have blocked this user',
      'messages_disabled': 'This user has disabled messages',
      'followers_only': 'This user only accepts messages from people they follow',
      'not a participant': 'You are not in this conversation',
      'message not found': 'This message no longer exists',
      'only sender can delete message': 'You can only delete your own messages',
      'message too old to edit': 'Messages can only be edited within 24 hours',
      'recipient has no encryption keys': 'This user has not set up messaging',
      'initiator_keys_not_initialized': 'Please set up messaging encryption first',
      'recipient_keys_not_initialized': 'This user has not set up messaging yet',
      'recipient_not_found': 'This user does not exist',
    };
    
    return MessagingError(
      error,
      userMessage: userMessages[error],
    );
  }
  
  void showSnackbar(BuildContext context) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(userMessage ?? 'An error occurred')),
    );
  }
}

// Usage
try {
  await MessagingService.sendMessage(...);
} catch (e) {
  final error = MessagingError.fromResponse(e.toString());
  error.showSnackbar(context);
}
```


---

## Complete Messaging Service Example

Here's a complete service class that implements all messaging functionality:

```dart
import 'dart:convert';
import 'package:flutter/foundation.dart';

class MessagingService {
  static final ApiClient _api = ApiClient.instance;
  
  // ============================================================
  // Key Management
  // ============================================================
  
  /// Initialize user's encryption keys (call once on first use)
  static Future<void> initializeKeys(String publicKey, String privateKeyEnc) async {
    final response = await _api.call('API_MessageInitKeys', p3: [publicKey, privateKeyEnc]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Get own keys
  static Future<Map<String, dynamic>> getMyKeys() async {
    final response = await _api.call('API_MessageGetKeys');
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// Get another user's public key
  static Future<String> getUserPublicKey(String userUUID) async {
    final response = await _api.call('API_MessageGetUserPublicKey', p3: [userUUID]);
    if (response.p1 == 'Success') {
      return jsonDecode(response.p3[0])['publicKey'];
    }
    throw Exception(response.p2);
  }
  
  // ============================================================
  // Conversations
  // ============================================================
  
  /// Start or get existing conversation with a user
  static Future<Map<String, dynamic>> startConversation(String recipientUUID) async {
    final response = await _api.call('API_MessageConversationStart', p3: [recipientUUID]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// List all conversations
  static Future<List<dynamic>> listConversations({int limit = 20, int offset = 0}) async {
    final response = await _api.call('API_MessageConversationList', 
      p3: [limit.toString(), offset.toString()]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// Get single conversation
  static Future<Map<String, dynamic>> getConversation(String conversationUUID) async {
    final response = await _api.call('API_MessageConversationGet', p3: [conversationUUID]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// Delete (hide) a conversation
  static Future<void> deleteConversation(String conversationUUID) async {
    final response = await _api.call('API_MessageConversationDelete', p3: [conversationUUID]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Mute a conversation
  /// mutedUntil: 0 = forever, -1 = unmute, or Unix timestamp
  static Future<void> muteConversation(String conversationUUID, int mutedUntil) async {
    final response = await _api.call('API_MessageConversationMute', 
      p3: [conversationUUID, mutedUntil.toString()]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Mark conversation as read
  static Future<void> markConversationRead(String conversationUUID) async {
    final response = await _api.call('API_MessageConversationRead', p3: [conversationUUID]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Get session key for a conversation
  static Future<Map<String, dynamic>> getSessionKey(String conversationUUID) async {
    final response = await _api.call('API_MessageSessionKey', p3: [conversationUUID]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// Rotate session key (for forward secrecy)
  static Future<Map<String, dynamic>> rotateSessionKey(String conversationUUID) async {
    final response = await _api.call('API_MessageRotateKey', p3: [conversationUUID]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  // ============================================================
  // Messages
  // ============================================================
  
  /// Send an encrypted message
  static Future<Map<String, dynamic>> sendMessage({
    required String conversationUUID,
    required int messageType,
    required String contentEnc,
    required String contentNonce,
    String? replyToUUID,
    String? metadataEnc,
  }) async {
    final response = await _api.call('API_MessageSend', p3: [
      conversationUUID,
      messageType.toString(),
      contentEnc,
      contentNonce,
      replyToUUID ?? '',
      metadataEnc ?? '',
    ]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// Get messages from a conversation
  static Future<List<dynamic>> getMessages({
    required String conversationUUID,
    int limit = 50,
    int offset = 0,
    int? beforeTimestamp,
  }) async {
    final response = await _api.call('API_MessageGetMessages', p3: [
      conversationUUID,
      limit.toString(),
      offset.toString(),
      beforeTimestamp?.toString() ?? '',
    ]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// Delete a message
  static Future<void> deleteMessage(String messageUUID) async {
    final response = await _api.call('API_MessageDeleteMessage', p3: [messageUUID]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Edit a message (within 24 hours)
  static Future<void> editMessage(String messageUUID, String newContentEnc, String newNonce) async {
    final response = await _api.call('API_MessageEditMessage', 
      p3: [messageUUID, newContentEnc, newNonce]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Mark message as received or read
  static Future<void> markMessageReceipt(String messageUUID, {bool isRead = true}) async {
    final response = await _api.call('API_MessageReceipt', 
      p3: [messageUUID, isRead.toString()]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  // ============================================================
  // User Preferences
  // ============================================================
  
  /// Get messaging preferences
  static Future<Map<String, dynamic>> getPreferences() async {
    final response = await _api.call('API_MessageGetPrefs');
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// Update messaging preferences
  static Future<void> setPreferences({
    String? allowMessages,  // "all", "followers", "none"
    bool? messageNotifications,
    bool? readReceipts,
  }) async {
    final prefs = <String, dynamic>{};
    if (allowMessages != null) prefs['allowMessages'] = allowMessages;
    if (messageNotifications != null) prefs['messageNotifications'] = messageNotifications;
    if (readReceipts != null) prefs['readReceipts'] = readReceipts;
    
    final response = await _api.call('API_MessageSetPrefs', p3: [jsonEncode(prefs)]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Check if can message a user
  static Future<Map<String, dynamic>> canMessageUser(String targetUUID) async {
    final response = await _api.call('API_MessageCanMessage', p3: [targetUUID]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  // ============================================================
  // Blocking
  // ============================================================
  
  /// Block a user
  static Future<void> blockUser(String userUUID, {String? reason}) async {
    final response = await _api.call('API_MessageBlockUser', p3: [userUUID, reason ?? '']);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Unblock a user
  static Future<void> unblockUser(String userUUID) async {
    final response = await _api.call('API_MessageUnblockUser', p3: [userUUID]);
    if (response.p1 != 'Success') throw Exception(response.p2);
  }
  
  /// Get blocked users with profile details
  static Future<List<dynamic>> getBlockedUsers() async {
    final response = await _api.call('API_MessageGetBlocked');
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
  
  /// Check block status (bidirectional)
  static Future<Map<String, dynamic>> checkBlockStatus(String userUUID) async {
    final response = await _api.call('API_MessageIsBlocked', p3: [userUUID]);
    if (response.p1 == 'Success') return jsonDecode(response.p3[0]);
    throw Exception(response.p2);
  }
}
```


---

## Implementation Checklist

### Required for MVP

- [ ] Key generation and secure storage
- [ ] `API_MessageInitKeys` - Initialize encryption keys
- [ ] `API_MessageConversationStart` - Start conversations
- [ ] `API_MessageConversationList` - List conversations
- [ ] `API_MessageSend` - Send encrypted messages
- [ ] `API_MessageGetMessages` - Retrieve and decrypt messages
- [ ] `API_MessageSessionKey` - Get session key for encryption
- [ ] WebSocket handler for `SAPI_NewMessage`

### Important Features

- [ ] `API_MessageGetPrefs` / `API_MessageSetPrefs` - User preferences
- [ ] `API_MessageBlockUser` / `API_MessageUnblockUser` - Blocking
- [ ] `API_MessageGetBlocked` - Block management UI
- [ ] `API_MessageCanMessage` - Pre-flight check before messaging
- [ ] `API_MessageConversationRead` - Mark as read
- [ ] `API_MessageReceipt` - Read receipts
- [ ] `API_MessageDeleteMessage` - Delete messages
- [ ] `API_MessageEditMessage` - Edit messages

### Advanced Features

- [ ] `API_MessageRotateKey` - Forward secrecy key rotation
- [ ] `API_MessageKeyHistory` - Decrypt old messages after rotation
- [ ] `API_MessageConversationMute` - Mute notifications
- [ ] Push notifications integration
- [ ] Offline message queue
- [ ] Message search


