# Feishu Channel Design

Add Feishu (飞书) as a messaging channel to NanoClaw, parallel to Telegram, Slack, Discord, and WhatsApp.

## Decisions

- **Approach**: Built-in channel (方案 A). Same pattern as Telegram/Slack/Discord — single file in `src/channels/`, self-registration via factory.
- **Connection**: WebSocket long connection via SDK's `WSClient`. No public URL required.
- **Domain**: Domestic Feishu (`Lark.Domain.Feishu`).
- **Outbound format**: Plain text (`msg_type: 'text'`).
- **Scope**: Group chat + private chat + non-text message placeholders.

## File Changes

### New Files

| File | Purpose |
|------|---------|
| `src/channels/feishu.ts` | FeishuChannel class + `registerChannel()` call |
| `src/channels/feishu.test.ts` | Unit tests (mocked SDK) |

### Modified Files

| File | Change |
|------|--------|
| `src/channels/index.ts` | Add `import './feishu.js'` |
| `.env.example` | Add `FEISHU_APP_ID`, `FEISHU_APP_SECRET` |
| `package.json` | Add `@larksuiteoapi/node-sdk` dependency |

## JID Format

```
feishu:{chat_id}
```

Examples: `feishu:oc_abc123def456` (group), `feishu:oc_xyz789` (private).

`ownsJid()` checks `jid.startsWith('feishu:')`.

## Environment Variables

```
FEISHU_APP_ID=cli_xxxxxxxxxxxx
FEISHU_APP_SECRET=your_app_secret
```

The SDK automatically manages `tenant_access_token` acquisition and refresh.

## Class Interface

```typescript
export class FeishuChannel implements Channel {
  name = 'feishu';

  constructor(appId: string, appSecret: string, opts: ChannelOpts);

  async connect(): Promise<void>;
  async sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  async disconnect(): Promise<void>;
  async setTyping?(jid: string, isTyping: boolean): Promise<void>; // noop
}
```

## Inbound Message Flow

```
Feishu user sends message
    ↓
WSClient receives im.message.receive_v1 event
    ↓
EventDispatcher callback fires
    ↓
Extract fields:
  - chatId = data.message.chat_id           → "oc_xxx"
  - chatType = data.message.chat_type       → "group" | "p2p"
  - messageType = data.message.message_type → "text" | "image" | ...
  - content = JSON.parse(data.message.content)
  - senderId = data.sender.sender_id.open_id
  - mentions = data.message.mentions || []
    ↓
Build JID: feishu:{chatId}
    ↓
Security filter: only registered groups pass through
    ↓
Build NewMessage:
  {
    id: message_id,
    chat_jid: "feishu:{chatId}",
    sender: senderId,
    sender_name: resolved via API (cached),
    content: parsed text,
    timestamp: ISO from create_time,
    is_from_me: false,
    is_bot_message: false
  }
    ↓
opts.onMessage(chatJid, message)
    ↓
SQLite → message loop → agent invocation
```

## @Mention Handling

Feishu uses placeholder keys in message text (`@_user_1`) with a `mentions` array mapping keys to user IDs.

1. Check if `mentions` contains the bot's own `open_id`.
2. If yes, replace the placeholder with `@{ASSISTANT_NAME}` (e.g., `@Andy`) so the trigger pattern matches.
3. Private chat (`chat_type === "p2p"`) triggers without @mention.

## Outbound Message Flow

```typescript
async sendMessage(jid: string, text: string): Promise<void> {
  const chatId = jid.replace(/^feishu:/, '');
  const chunks = splitMessage(text, 4000);

  for (const chunk of chunks) {
    await this.client.im.v1.message.create({
      params: { receive_id_type: 'chat_id' },
      data: {
        receive_id: chatId,
        msg_type: 'text',
        content: JSON.stringify({ text: chunk }),
      },
    });
  }
}
```

Split at 4000 characters per chunk (consistent with Telegram strategy).

## Non-Text Message Placeholders

| message_type | Placeholder |
|---|---|
| `image` | `[Image]` |
| `file` | `[File]` |
| `audio` | `[Audio]` |
| `media` | `[Video]` |
| `sticker` | `[Sticker]` |
| `post` | Extract plain text (iterate content arrays, concatenate text segments) |
| other | `[Unsupported: {type}]` |

## sender_name Resolution

The `im.message.receive_v1` event does not include the sender's display name.

1. **Primary**: Call `client.contact.v3.user.get()` with the sender's `open_id`.
2. **Cache**: In-memory `Map<string, string>` mapping `open_id → name`. Avoids repeated API calls.
3. **Fallback**: On API failure, use `open_id` as the sender_name.

## Connection Lifecycle

### connect()

1. Create `Lark.Client` with appId, appSecret, `AppType.SelfBuild`, `Domain.Feishu`.
2. Create `EventDispatcher`, register `im.message.receive_v1` handler.
3. Create `WSClient`, call `wsClient.start({ eventDispatcher })`.
4. Set `connected = true`.

### disconnect()

Set `connected = false`. SDK's WSClient has no explicit stop method; the flag prevents event processing.

### setTyping()

Noop. Feishu IM API does not support typing indicators.

## Error Handling

| Scenario | Strategy |
|---|---|
| WSClient connection failure | SDK built-in reconnection; log warn |
| Send message API failure | Catch, log error, do not break message loop |
| User name lookup failure | Fallback to open_id, log warn |
| Unregistered group message | Silent ignore (consistent with other channels) |
| Rate limit (429) | Log warn, no retry (avoid cascading), natural interval on next message |

## Test Plan

17 test cases in `src/channels/feishu.test.ts`, fully mocking `@larksuiteoapi/node-sdk`.

### Registration (2)
- No credentials → factory returns null
- With credentials → factory returns FeishuChannel instance

### JID Routing (2)
- `ownsJid('feishu:oc_xxx')` → true
- `ownsJid('tg:123')` → false

### Connection (1)
- `connect()` calls `WSClient.start()`

### Inbound Messages (4)
- Text message → onMessage callback with correct fields
- Image message → placeholder `[Image]`
- Post (rich text) → extracted plain text
- Unregistered group → silently ignored

### @Mention (2)
- Bot @mentioned → placeholder replaced with `@{ASSISTANT_NAME}`
- Private chat → trigger prefix auto-added

### Outbound Messages (3)
- Short text → single API call
- Long text → chunked into multiple calls
- API failure → error logged, no throw

### sender_name (3)
- First lookup → API called
- Repeat lookup → cache hit, API called once
- API failure → fallback to open_id
