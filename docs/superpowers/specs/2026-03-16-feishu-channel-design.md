# Feishu Channel Design

Add Feishu (飞书) as a messaging channel to NanoClaw, parallel to Telegram, Slack, Discord, and WhatsApp.

## Decisions

- **Approach**: Built-in channel (方案 A). Same pattern as Telegram/Slack/Discord — single file in `src/channels/`, self-registration via factory.
- **Connection**: WebSocket long connection via SDK's `WSClient`. No public URL required.
- **Domain**: Domestic Feishu (`Lark.Domain.Feishu`).
- **Outbound format**: Plain text (`msg_type: 'text'`). Future enhancement: upgrade to `post` rich text for markdown/links support.
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

The SDK automatically manages `tenant_access_token` acquisition and refresh. The factory also reads from `.env` file via `readEnvFile()` as fallback (consistent with Telegram).

## Factory Registration

```typescript
registerChannel('feishu', (opts: ChannelOpts) => {
  const envVars = readEnvFile(['FEISHU_APP_ID', 'FEISHU_APP_SECRET']);
  const appId = process.env.FEISHU_APP_ID || envVars.FEISHU_APP_ID || '';
  const appSecret = process.env.FEISHU_APP_SECRET || envVars.FEISHU_APP_SECRET || '';
  if (!appId || !appSecret) {
    logger.warn('Feishu: FEISHU_APP_ID or FEISHU_APP_SECRET not set');
    return null;
  }
  return new FeishuChannel(appId, appSecret, opts);
});
```

## Class Interface

```typescript
export class FeishuChannel implements Channel {
  name = 'feishu';
  private client: Lark.Client;        // For API calls (send messages, user lookup)
  private wsClient: Lark.WSClient;    // For receiving events (separate object)
  private connected = false;
  private botOpenId: string = '';      // Resolved during connect()
  private userNameCache = new Map<string, string>();

  constructor(appId: string, appSecret: string, opts: ChannelOpts);

  async connect(): Promise<void>;
  async sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  async disconnect(): Promise<void>;
  async setTyping?(jid: string, isTyping: boolean): Promise<void>; // noop
  // syncGroups intentionally omitted — Feishu has no equivalent bulk sync API
}
```

Note: `Lark.Client` and `Lark.WSClient` are two separate SDK objects. `Client` handles API calls (sending messages, querying users). `WSClient` handles the WebSocket event subscription. Both require `appId`/`appSecret` independently.

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
  - senderType = data.sender.sender_type    → "user" | "bot" | ...
  - mentions = data.message.mentions || []
    ↓
Build JID: feishu:{chatId}
    ↓
Emit chat metadata (ALWAYS, even for unregistered chats):
  opts.onChatMetadata(chatJid, timestamp, chatName, 'feishu', chatType === 'group')
  This enables chat discovery — unregistered chats appear in admin UI.
    ↓
Security filter: only registered groups pass through
  (unregistered chats: emit metadata above, then return early)
    ↓
Filter bot messages:
  if senderType !== 'user', set is_bot_message = true
    ↓
Build NewMessage:
  {
    id: message_id,
    chat_jid: "feishu:{chatId}",
    sender: senderId,
    sender_name: resolved via API (cached),
    content: parsed text (with @mention replacement),
    timestamp: new Date(parseInt(data.message.create_time)).toISOString(),
    is_from_me: senderId === this.botOpenId
  }
    ↓
opts.onMessage(chatJid, message)
    ↓
SQLite → message loop → agent invocation
```

Key details:

- `create_time` is **milliseconds** (string). Convert via `new Date(parseInt(create_time)).toISOString()`.
- `onChatMetadata` is called for ALL incoming messages (registered and unregistered) to enable chat discovery.
- Bot messages (`sender_type !== 'user'`) are tagged so the router can filter them from agent input.

## @Mention Handling

Feishu uses placeholder keys in message text (`@_user_1`) with a `mentions` array mapping keys to user IDs.

1. During `connect()`, fetch the bot's own `open_id` via `client.bot.v3.botInfo.get()` and cache it as `this.botOpenId`.
2. On incoming message, check if `mentions` contains an entry where `id.open_id === this.botOpenId`.
3. If yes, replace that placeholder (e.g., `@_user_1`) with `@{ASSISTANT_NAME}` (e.g., `@Andy`) so the trigger pattern matches.
4. For private chat (`chat_type === "p2p"`): prepend `@{ASSISTANT_NAME}` to the message content so the trigger pattern matches without explicit @mention. This is consistent with how Telegram handles DMs.

## Outbound Message Flow

```typescript
async sendMessage(jid: string, text: string): Promise<void> {
  const chatId = jid.replace(/^feishu:/, '');

  // Feishu text messages support up to ~30KB content.
  // Split at 4096 chars for UX consistency with Telegram.
  for (let i = 0; i < text.length; i += 4096) {
    const chunk = text.slice(i, i + 4096);
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

Uses inline slicing (same approach as Telegram), not a separate `splitMessage` utility.

## Non-Text Message Placeholders

| message_type | Placeholder |
|---|---|
| `image` | `[Image]` |
| `file` | `[File]` |
| `audio` | `[Audio]` |
| `media` | `[Video]` |
| `sticker` | `[Sticker]` |
| `post` | Extract plain text (see below) |
| other | `[Unsupported: {type}]` |

### Post (rich text) extraction

Feishu `post` content structure:

```json
{
  "zh_cn": {
    "title": "Title",
    "content": [
      [{"tag": "text", "text": "Hello "}, {"tag": "a", "text": "link", "href": "..."}],
      [{"tag": "at", "user_id": "ou_xxx"}, {"tag": "text", "text": " world"}]
    ]
  }
}
```

Extraction logic:

1. Get the first available locale key (try `zh_cn`, then first key).
2. If `title` exists, prepend it.
3. Iterate `content` (array of arrays). For each segment:
   - `tag === "text"` or `tag === "a"`: append `.text`
   - `tag === "at"`: append `@{user_id}`
   - Other tags: skip
4. Join lines with `\n`.

## sender_name Resolution

The `im.message.receive_v1` event does not include the sender's display name.

1. **Primary**: Call `client.contact.v3.user.get({ path: { user_id: open_id }, params: { user_id_type: 'open_id' } })` and extract `.data.user.name`.
2. **Cache**: In-memory `Map<string, string>` mapping `open_id → name`. Avoids repeated API calls.
3. **Fallback**: On API failure, use `open_id` as the sender_name.
4. **Permission**: Requires `contact:user.base:readonly` scope (add to setup instructions).

## Bot Identity

During `connect()`, call `client.bot.v3.botInfo.get()` to obtain the bot's `open_id`. Cache as `this.botOpenId`. Used for:

- `is_from_me` detection (filter out bot's own messages from agent input)
- @mention detection (match bot's open_id in mentions array)

## Connection Lifecycle

### connect()

1. Create `Lark.Client` with appId, appSecret, `AppType.SelfBuild`, `Domain.Feishu` — for API calls.
2. Fetch bot identity via `client.bot.v3.botInfo.get()`, cache `botOpenId`.
3. Create `EventDispatcher`, register `im.message.receive_v1` handler.
4. Create `Lark.WSClient` with appId, appSecret — for event subscription (separate from Client).
5. Call `wsClient.start({ eventDispatcher })`.
6. Set `connected = true`.

### disconnect()

Set `connected = false` and `wsClient = null`. SDK's WSClient has no explicit `stop()` method — this is a known SDK limitation. Setting the reference to null allows garbage collection. Process exit is the true cleanup mechanism.

### setTyping()

Noop. Feishu IM API does not support typing indicators.

## Error Handling

| Scenario | Strategy |
|---|---|
| WSClient connection failure | SDK built-in reconnection; log warn |
| Send message API failure | Catch, log error, do not break message loop |
| User name lookup failure | Fallback to open_id, log warn |
| Bot identity fetch failure | Log error, use empty string (is_from_me will never match, @mention detection degrades gracefully) |
| Unregistered group message | Emit onChatMetadata for discovery, then silent ignore |
| Rate limit (429) | Log warn, no retry. Feishu rate limit is ~50 msg/sec — generous for personal assistant use case |
| Bot-originated message | Tag as is_bot_message, let router filter |

## Test Plan

22 test cases in `src/channels/feishu.test.ts`, fully mocking `@larksuiteoapi/node-sdk`.

### Registration (2)
- No credentials → factory returns null
- With credentials → factory returns FeishuChannel instance

### JID Routing (2)
- `ownsJid('feishu:oc_xxx')` → true
- `ownsJid('tg:123')` → false

### Connection (3)

- `connect()` creates Client + WSClient separately, calls WSClient.start()
- `isConnected()` returns false before connect, true after
- `disconnect()` sets connected to false

### Inbound Messages (5)

- Text message → onMessage callback with correct fields
- Image message → placeholder `[Image]`
- Post (rich text) → extracted plain text with title
- Unregistered group → onChatMetadata called, onMessage NOT called
- Bot message (sender_type !== 'user') → tagged as is_bot_message

### @Mention (2)

- Bot @mentioned in group → placeholder replaced with `@{ASSISTANT_NAME}`
- Private chat → `@{ASSISTANT_NAME}` prepended to content

### Outbound Messages (3)

- Short text → single API call
- Long text (>4096 chars) → chunked into multiple calls
- API failure → error logged, no throw

### sender_name (3)

- First lookup → API called, name returned
- Repeat lookup → cache hit, API called only once
- API failure → fallback to open_id

### Chat Metadata (2)

- Registered group message → onChatMetadata called with channel='feishu'
- Unregistered group → onChatMetadata still called (enables discovery)
