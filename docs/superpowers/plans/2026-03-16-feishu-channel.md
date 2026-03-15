# Feishu Channel Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Feishu (飞书) as a built-in messaging channel to NanoClaw, supporting group chat, private chat, @mention translation, non-text placeholders, and sender name resolution.

**Architecture:** Single-file channel implementation (`src/channels/feishu.ts`) following the self-registration pattern used by Telegram. WebSocket long connection via SDK's `WSClient` for receiving events; `Client` for API calls. JID format: `feishu:{chat_id}`.

**Tech Stack:** `@larksuiteoapi/node-sdk`, TypeScript, vitest

**Spec:** `docs/superpowers/specs/2026-03-16-feishu-channel-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `src/channels/feishu.ts` | Create | FeishuChannel class + registerChannel factory |
| `src/channels/feishu.test.ts` | Create | Unit tests (25 cases, fully mocked SDK) |
| `src/channels/index.ts` | Modify | Add `import './feishu.js'` |
| `.env.example` | Modify | Add `FEISHU_APP_ID`, `FEISHU_APP_SECRET` |
| `package.json` | Modify (via npm) | Add `@larksuiteoapi/node-sdk` dependency |

---

## Chunk 1: Setup & Test Infrastructure

### Task 1: Install SDK and update config files

- [ ] **Step 1: Install the Feishu SDK**

Run: `npm install @larksuiteoapi/node-sdk`
Expected: Package added to `package.json` dependencies.

- [ ] **Step 2: Update `.env.example`**

Add to `.env.example`:

```
FEISHU_APP_ID=
FEISHU_APP_SECRET=
```

- [ ] **Step 3: Add barrel import**

Add to `src/channels/index.ts` before the `// whatsapp` comment:

```typescript
// feishu
import './feishu.js';
```

- [ ] **Step 4: Commit**

```bash
git add package.json package-lock.json .env.example src/channels/index.ts
git commit -m "chore: add @larksuiteoapi/node-sdk dependency and feishu config"
```

### Task 2: Create test file with mock infrastructure

- [ ] **Step 1: Create `src/channels/feishu.test.ts` with full mock setup**

This sets up the entire mock infrastructure that all subsequent tests will use. The key patterns:
- `vi.hoisted()` for refs that mocks share with test code
- `vi.mock()` for all dependencies (SDK, registry, env, config, logger)
- Helper functions to create test opts and trigger events

```typescript
import { describe, it, expect, beforeEach, vi, afterEach } from 'vitest';

// --- Mocks (must be before imports) ---

vi.mock('./registry.js', () => ({ registerChannel: vi.fn() }));
vi.mock('../env.js', () => ({ readEnvFile: vi.fn(() => ({})) }));
vi.mock('../config.js', () => ({
  ASSISTANT_NAME: 'Andy',
  TRIGGER_PATTERN: /^@Andy\b/i,
}));
vi.mock('../logger.js', () => ({
  logger: {
    debug: vi.fn(),
    info: vi.fn(),
    warn: vi.fn(),
    error: vi.fn(),
  },
}));

// --- Feishu SDK mock ---

const sdkRef = vi.hoisted(() => ({
  messageCreate: null as any,
  userGet: null as any,
  botInfoGet: null as any,
  wsStart: null as any,
  eventHandlers: null as Record<string, Function> | null,
}));

vi.mock('@larksuiteoapi/node-sdk', () => ({
  Client: vi.fn(() => ({
    im: {
      v1: { message: { create: sdkRef.messageCreate } },
    },
    contact: {
      v3: { user: { get: sdkRef.userGet } },
    },
    bot: {
      v3: { info: { get: sdkRef.botInfoGet } },
    },
  })),
  WSClient: vi.fn(() => ({
    start: sdkRef.wsStart,
  })),
  EventDispatcher: vi.fn(() => ({
    register: vi.fn(function (this: any, handlers: Record<string, Function>) {
      sdkRef.eventHandlers = handlers;
      return this;
    }),
  })),
  AppType: { SelfBuild: 0 },
  Domain: { Feishu: 'https://open.feishu.cn' },
  LoggerLevel: { info: 1 },
}));

import { FeishuChannel, FeishuChannelOpts } from './feishu.js';

// --- Test helpers ---

function createTestOpts(
  overrides?: Partial<FeishuChannelOpts>,
): FeishuChannelOpts {
  return {
    onMessage: vi.fn(),
    onChatMetadata: vi.fn(),
    registeredGroups: vi.fn(() => ({
      'feishu:oc_test123': {
        name: 'Test Group',
        folder: 'test-group',
        trigger: '@Andy',
        added_at: '2024-01-01T00:00:00.000Z',
      },
    })),
    ...overrides,
  };
}

function createMessageEvent(overrides: {
  chatId?: string;
  chatType?: string;
  messageType?: string;
  content?: string;
  messageId?: string;
  senderOpenId?: string;
  senderType?: string;
  mentions?: any[];
  createTime?: string;
}) {
  return {
    message: {
      message_id: overrides.messageId ?? 'om_msg001',
      chat_id: overrides.chatId ?? 'oc_test123',
      chat_type: overrides.chatType ?? 'group',
      message_type: overrides.messageType ?? 'text',
      content: overrides.content ?? JSON.stringify({ text: 'Hello' }),
      mentions: overrides.mentions ?? [],
      create_time: overrides.createTime ?? '1704067200000',
    },
    sender: {
      sender_id: {
        open_id: overrides.senderOpenId ?? 'ou_sender001',
      },
      sender_type: overrides.senderType ?? 'user',
    },
  };
}

async function triggerEvent(eventName: string, data: any): Promise<void> {
  const handler = sdkRef.eventHandlers?.[eventName];
  if (handler) await handler(data);
}

// --- Tests ---

describe('FeishuChannel', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    sdkRef.messageCreate = vi.fn().mockResolvedValue({});
    sdkRef.userGet = vi
      .fn()
      .mockResolvedValue({ data: { user: { name: 'Test User' } } });
    sdkRef.botInfoGet = vi
      .fn()
      .mockResolvedValue({ data: { bot: { open_id: 'ou_bot123' } } });
    sdkRef.wsStart = vi.fn().mockResolvedValue(undefined);
    sdkRef.eventHandlers = null;
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  // Tests will be added in subsequent tasks
  it('placeholder', () => {
    expect(true).toBe(true);
  });
});
```

- [ ] **Step 2: Create `src/channels/feishu.ts` minimal skeleton**

Just enough for the test file to import without errors:

```typescript
import * as Lark from '@larksuiteoapi/node-sdk';

import { ASSISTANT_NAME, TRIGGER_PATTERN } from '../config.js';
import { readEnvFile } from '../env.js';
import { logger } from '../logger.js';
import { registerChannel, ChannelOpts } from './registry.js';
import {
  Channel,
  OnChatMetadata,
  OnInboundMessage,
  RegisteredGroup,
} from '../types.js';

export interface FeishuChannelOpts {
  onMessage: OnInboundMessage;
  onChatMetadata: OnChatMetadata;
  registeredGroups: () => Record<string, RegisteredGroup>;
}

export class FeishuChannel implements Channel {
  name = 'feishu';

  private client: Lark.Client | null = null;
  private wsClient: Lark.WSClient | null = null;
  private opts: FeishuChannelOpts;
  private appId: string;
  private appSecret: string;
  private connected = false;
  private botOpenId = '';
  private userNameCache = new Map<string, string>();

  constructor(appId: string, appSecret: string, opts: FeishuChannelOpts) {
    this.appId = appId;
    this.appSecret = appSecret;
    this.opts = opts;
  }

  async connect(): Promise<void> {
    throw new Error('Not implemented');
  }

  async sendMessage(_jid: string, _text: string): Promise<void> {
    throw new Error('Not implemented');
  }

  isConnected(): boolean {
    return this.connected;
  }

  ownsJid(_jid: string): boolean {
    throw new Error('Not implemented');
  }

  async disconnect(): Promise<void> {
    throw new Error('Not implemented');
  }

  async setTyping(_jid: string, _isTyping: boolean): Promise<void> {
    // Feishu API does not support typing indicators
  }
}

registerChannel('feishu', (opts: ChannelOpts) => {
  const envVars = readEnvFile(['FEISHU_APP_ID', 'FEISHU_APP_SECRET']);
  const appId = process.env.FEISHU_APP_ID || envVars.FEISHU_APP_ID || '';
  const appSecret =
    process.env.FEISHU_APP_SECRET || envVars.FEISHU_APP_SECRET || '';
  if (!appId || !appSecret) {
    logger.warn('Feishu: FEISHU_APP_ID or FEISHU_APP_SECRET not set');
    return null;
  }
  return new FeishuChannel(appId, appSecret, opts);
});
```

- [ ] **Step 3: Run tests to verify mock infrastructure works**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: 1 test passes (placeholder).

- [ ] **Step 4: Commit**

```bash
git add src/channels/feishu.ts src/channels/feishu.test.ts
git commit -m "feat(feishu): scaffold channel skeleton and test infrastructure"
```

---

## Chunk 2: Factory Registration & JID Routing

### Task 3: Factory registration

- [ ] **Step 1: Replace the placeholder test with registration tests**

In `src/channels/feishu.test.ts`, replace the placeholder test with:

```typescript
  describe('factory registration', () => {
    it('returns null when credentials are missing', async () => {
      const { registerChannel } = await import('./registry.js');
      // registerChannel was called at module load time
      expect(registerChannel).toHaveBeenCalledWith('feishu', expect.any(Function));

      // Extract the factory and call it with no env vars
      const factory = (registerChannel as any).mock.calls.find(
        (c: any) => c[0] === 'feishu',
      )?.[1];
      expect(factory).toBeDefined();

      const result = factory(createTestOpts());
      expect(result).toBeNull();
    });

    it('returns FeishuChannel instance when credentials are set', async () => {
      process.env.FEISHU_APP_ID = 'cli_test';
      process.env.FEISHU_APP_SECRET = 'secret_test';

      // Re-extract factory (it was registered at import time)
      const { registerChannel } = await import('./registry.js');
      const factory = (registerChannel as any).mock.calls.find(
        (c: any) => c[0] === 'feishu',
      )?.[1];

      const result = factory(createTestOpts());
      expect(result).toBeInstanceOf(FeishuChannel);
      expect(result.name).toBe('feishu');

      delete process.env.FEISHU_APP_ID;
      delete process.env.FEISHU_APP_SECRET;
    });
  });
```

**Important:** Since `registerChannel` is called at module import time, the factory is already captured in the mock's calls. We extract it and invoke it directly. The factory reads from `process.env`, so we set env vars before calling it.

- [ ] **Step 2: Run tests to verify they pass**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: 2 tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/channels/feishu.test.ts
git commit -m "test(feishu): add factory registration tests"
```

### Task 4: JID routing

- [ ] **Step 1: Write JID routing tests**

Add to the `describe('FeishuChannel')` block:

```typescript
  describe('ownsJid', () => {
    it('owns feishu: JIDs', () => {
      const channel = new FeishuChannel('id', 'secret', createTestOpts());
      expect(channel.ownsJid('feishu:oc_abc123')).toBe(true);
    });

    it('does not own telegram JIDs', () => {
      const channel = new FeishuChannel('id', 'secret', createTestOpts());
      expect(channel.ownsJid('tg:123456')).toBe(false);
    });
  });
```

- [ ] **Step 2: Run tests — expect ownsJid tests to FAIL**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: ownsJid tests fail with "Not implemented".

- [ ] **Step 3: Implement ownsJid**

In `src/channels/feishu.ts`, replace the `ownsJid` method:

```typescript
  ownsJid(jid: string): boolean {
    return jid.startsWith('feishu:');
  }
```

- [ ] **Step 4: Run tests — all pass**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/channels/feishu.ts src/channels/feishu.test.ts
git commit -m "feat(feishu): implement JID routing"
```

---

## Chunk 3: Connection Lifecycle

### Task 5: Connection lifecycle

- [ ] **Step 1: Write connection lifecycle tests**

Add to the `describe('FeishuChannel')` block:

```typescript
  describe('connection lifecycle', () => {
    it('isConnected() returns false before connect', () => {
      const channel = new FeishuChannel('id', 'secret', createTestOpts());
      expect(channel.isConnected()).toBe(false);
    });

    it('connect() creates Client and WSClient, starts WebSocket', async () => {
      const channel = new FeishuChannel('id', 'secret', createTestOpts());
      await channel.connect();

      expect(channel.isConnected()).toBe(true);
      expect(sdkRef.wsStart).toHaveBeenCalled();
      expect(sdkRef.botInfoGet).toHaveBeenCalled();
    });

    it('disconnect() sets connected to false', async () => {
      const channel = new FeishuChannel('id', 'secret', createTestOpts());
      await channel.connect();
      expect(channel.isConnected()).toBe(true);

      await channel.disconnect();
      expect(channel.isConnected()).toBe(false);
    });
  });
```

- [ ] **Step 2: Run tests — expect connection tests to FAIL**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: connect/disconnect tests fail.

- [ ] **Step 3: Implement connect() and disconnect()**

In `src/channels/feishu.ts`, replace `connect()` and `disconnect()`:

```typescript
  async connect(): Promise<void> {
    this.client = new Lark.Client({
      appId: this.appId,
      appSecret: this.appSecret,
      appType: Lark.AppType.SelfBuild,
      domain: Lark.Domain.Feishu,
    });

    // Fetch bot identity for @mention detection and is_from_me
    try {
      const botInfo = await (this.client as any).bot.v3.info.get();
      this.botOpenId = botInfo?.data?.bot?.open_id || '';
      logger.info(
        { botOpenId: this.botOpenId },
        'Feishu: bot identity resolved',
      );
    } catch (err) {
      logger.error(
        { err },
        'Feishu: failed to get bot info, @mention detection may not work',
      );
    }

    const eventDispatcher = new Lark.EventDispatcher({}).register({
      'im.message.receive_v1': async (data: any) => {
        if (!this.connected) return;
        await this.handleMessage(data);
      },
    });

    this.wsClient = new Lark.WSClient({
      appId: this.appId,
      appSecret: this.appSecret,
      loggerLevel: Lark.LoggerLevel.info,
    });

    await this.wsClient.start({ eventDispatcher });
    this.connected = true;
    logger.info('Feishu: connected via WebSocket');
  }

  // Also add the private handleMessage stub (empty for now):
  private async handleMessage(_data: any): Promise<void> {
    // Will be implemented in Task 6
  }

  async disconnect(): Promise<void> {
    this.connected = false;
    this.wsClient = null;
    this.client = null;
    logger.info('Feishu: disconnected');
  }
```

- [ ] **Step 4: Run tests — all pass**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/channels/feishu.ts src/channels/feishu.test.ts
git commit -m "feat(feishu): implement connection lifecycle"
```

---

## Chunk 4: Inbound Messages

### Task 6: Inbound text messages

- [ ] **Step 1: Write inbound text message tests**

Add to the `describe('FeishuChannel')` block:

```typescript
  describe('inbound text messages', () => {
    it('delivers text message for registered group', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({ content: JSON.stringify({ text: 'Hello world' }) });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({
          id: 'om_msg001',
          chat_jid: 'feishu:oc_test123',
          sender: 'ou_sender001',
          sender_name: 'Test User',
          content: 'Hello world',
          timestamp: '2024-01-01T00:00:00.000Z',
          is_from_me: false,
          is_bot_message: false,
        }),
      );
    });

    it('ignores messages from unregistered chats', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({ chatId: 'oc_unknown999' });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).not.toHaveBeenCalled();
    });

    it('tags bot messages with is_bot_message', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({ senderType: 'app' });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ is_bot_message: true }),
      );
    });

    it('sets is_from_me when sender is the bot', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({
        senderOpenId: 'ou_bot123', // matches botOpenId from mock
        senderType: 'app',
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ is_from_me: true, is_bot_message: true }),
      );
    });

    it('converts create_time milliseconds to ISO timestamp', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      // 1704067200000 ms = 2024-01-01T00:00:00.000Z
      const event = createMessageEvent({ createTime: '1704067200000' });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ timestamp: '2024-01-01T00:00:00.000Z' }),
      );
    });
  });
```

- [ ] **Step 2: Run tests — expect inbound message tests to FAIL**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: Inbound message tests fail (handleMessage is empty).

- [ ] **Step 3: Implement handleMessage for text messages**

In `src/channels/feishu.ts`, replace the `handleMessage` stub:

```typescript
  private async handleMessage(data: any): Promise<void> {
    const message = data.message;
    const sender = data.sender;

    const chatId = message.chat_id;
    const chatType = message.chat_type; // "group" | "p2p"
    const messageType = message.message_type;
    const chatJid = `feishu:${chatId}`;
    const senderId = sender.sender_id?.open_id || '';
    const senderType = sender.sender_type;
    const timestamp = new Date(parseInt(message.create_time)).toISOString();
    const isGroup = chatType === 'group';

    // Always emit metadata for chat discovery
    this.opts.onChatMetadata(chatJid, timestamp, undefined, 'feishu', isGroup);

    // Only process messages from registered groups
    const group = this.opts.registeredGroups()[chatJid];
    if (!group) {
      logger.debug({ chatJid }, 'Message from unregistered Feishu chat');
      return;
    }

    // Extract content based on message type
    let content: string;
    if (messageType === 'text') {
      try {
        const parsed = JSON.parse(message.content);
        content = parsed.text || '';
      } catch {
        content = message.content || '';
      }
    } else {
      content = `[Unsupported: ${messageType}]`;
    }

    // Resolve sender name
    const senderName = await this.resolveSenderName(senderId);

    this.opts.onMessage(chatJid, {
      id: message.message_id,
      chat_jid: chatJid,
      sender: senderId,
      sender_name: senderName,
      content,
      timestamp,
      is_from_me: senderId === this.botOpenId,
      is_bot_message: senderType !== 'user',
    });

    logger.info({ chatJid, sender: senderName }, 'Feishu message stored');
  }

  private async resolveSenderName(openId: string): Promise<string> {
    if (!openId) return 'Unknown';

    const cached = this.userNameCache.get(openId);
    if (cached) return cached;

    try {
      const res = await this.client!.contact.v3.user.get({
        path: { user_id: openId },
        params: { user_id_type: 'open_id' },
      } as any);
      const name = (res as any)?.data?.user?.name || openId;
      this.userNameCache.set(openId, name);
      return name;
    } catch (err) {
      logger.warn({ openId, err }, 'Feishu: failed to resolve sender name');
      this.userNameCache.set(openId, openId);
      return openId;
    }
  }
```

- [ ] **Step 4: Run tests — all pass**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/channels/feishu.ts src/channels/feishu.test.ts
git commit -m "feat(feishu): implement inbound text message handling"
```

### Task 7: @mention handling

- [ ] **Step 1: Write @mention tests**

Add to the `describe('FeishuChannel')` block:

```typescript
  describe('@mention handling', () => {
    it('replaces bot mention placeholder with trigger name', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({
        content: JSON.stringify({ text: '@_user_1 hello' }),
        mentions: [
          {
            key: '@_user_1',
            id: { open_id: 'ou_bot123' },
            name: 'TestBot',
          },
        ],
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({
          content: '@Andy hello',
        }),
      );
    });

    it('prepends trigger for private chat messages', async () => {
      const opts = createTestOpts({
        registeredGroups: vi.fn(() => ({
          'feishu:oc_private001': {
            name: 'Private',
            folder: 'private',
            trigger: '@Andy',
            added_at: '2024-01-01T00:00:00.000Z',
          },
        })),
      });
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({
        chatId: 'oc_private001',
        chatType: 'p2p',
        content: JSON.stringify({ text: 'What time is it?' }),
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_private001',
        expect.objectContaining({
          content: '@Andy What time is it?',
        }),
      );

      // Verify metadata emitted with isGroup = false for p2p chat
      expect(opts.onChatMetadata).toHaveBeenCalledWith(
        'feishu:oc_private001',
        expect.any(String),
        undefined,
        'feishu',
        false,
      );
    });
  });
```

- [ ] **Step 2: Run tests — expect @mention tests to FAIL**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: @mention tests fail (no mention handling in handleMessage yet).

- [ ] **Step 3: Add @mention handling to handleMessage**

In `src/channels/feishu.ts`, add these lines in `handleMessage` after content extraction, before `resolveSenderName`:

```typescript
    // @mention handling: replace bot mention placeholder with trigger name
    const mentions = message.mentions || [];
    if (isGroup && this.botOpenId) {
      for (const mention of mentions) {
        if (mention.id?.open_id === this.botOpenId) {
          content = content.replace(mention.key, `@${ASSISTANT_NAME}`);
          break;
        }
      }
    }

    // Private chat: prepend trigger so trigger pattern matches
    if (!isGroup && !TRIGGER_PATTERN.test(content)) {
      content = `@${ASSISTANT_NAME} ${content}`;
    }
```

- [ ] **Step 4: Run tests — all pass**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/channels/feishu.ts src/channels/feishu.test.ts
git commit -m "feat(feishu): implement @mention translation and p2p trigger"
```

### Task 8: Non-text message placeholders

- [ ] **Step 1: Write non-text placeholder tests**

Add to the `describe('FeishuChannel')` block:

```typescript
  describe('non-text messages', () => {
    it('returns [Image] placeholder for image messages', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({
        messageType: 'image',
        content: JSON.stringify({ image_key: 'img_xxx' }),
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ content: '[Image]' }),
      );
    });

    it('extracts plain text from post rich-text messages', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const postContent = {
        zh_cn: {
          title: 'My Title',
          content: [
            [
              { tag: 'text', text: 'Hello ' },
              { tag: 'a', text: 'link', href: 'https://example.com' },
            ],
            [
              { tag: 'at', user_id: 'ou_xxx' },
              { tag: 'text', text: ' world' },
            ],
          ],
        },
      };

      const event = createMessageEvent({
        messageType: 'post',
        content: JSON.stringify(postContent),
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({
          content: 'My Title\nHello link\n@ou_xxx world',
        }),
      );
    });

    it('returns [File] for file messages', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({ messageType: 'file', content: '{}' });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ content: '[File]' }),
      );
    });

    it('returns [Audio] for audio messages', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({
        messageType: 'audio',
        content: '{}',
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ content: '[Audio]' }),
      );
    });

    it('returns [Video] for media messages', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({
        messageType: 'media',
        content: '{}',
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ content: '[Video]' }),
      );
    });

    it('returns [Sticker] for sticker messages', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({
        messageType: 'sticker',
        content: '{}',
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ content: '[Sticker]' }),
      );
    });

    it('returns [Unsupported: share_chat] for unknown types', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({
        messageType: 'share_chat',
        content: '{}',
      });
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ content: '[Unsupported: share_chat]' }),
      );
    });
  });
```

- [ ] **Step 2: Run tests — expect post extraction test to FAIL**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: Post and other non-text tests fail (only text is handled, rest returns `[Unsupported: ...]`).

- [ ] **Step 3: Add extractPostText helper and expand content switch**

Add this function before the class definition in `src/channels/feishu.ts`:

```typescript
/** Extract plain text from a Feishu "post" rich-text content object. */
function extractPostText(content: any): string {
  const locale =
    content.zh_cn || content[Object.keys(content)[0]];
  if (!locale) return '[Post]';

  const parts: string[] = [];
  if (locale.title) parts.push(locale.title);

  if (Array.isArray(locale.content)) {
    for (const line of locale.content) {
      if (!Array.isArray(line)) continue;
      const lineText = line
        .map((seg: any) => {
          if (seg.tag === 'text' || seg.tag === 'a') return seg.text || '';
          if (seg.tag === 'at') return `@${seg.user_id || 'user'}`;
          return '';
        })
        .join('');
      parts.push(lineText);
    }
  }

  return parts.join('\n') || '[Post]';
}
```

Then expand the content extraction in `handleMessage`:

```typescript
    // Extract content based on message type
    let content: string;
    try {
      if (messageType === 'text') {
        const parsed = JSON.parse(message.content);
        content = parsed.text || '';
      } else if (messageType === 'post') {
        const parsed = JSON.parse(message.content);
        content = extractPostText(parsed);
      } else if (messageType === 'image') {
        content = '[Image]';
      } else if (messageType === 'file') {
        content = '[File]';
      } else if (messageType === 'audio') {
        content = '[Audio]';
      } else if (messageType === 'media') {
        content = '[Video]';
      } else if (messageType === 'sticker') {
        content = '[Sticker]';
      } else {
        content = `[Unsupported: ${messageType}]`;
      }
    } catch {
      content = message.content || `[${messageType}]`;
    }
```

- [ ] **Step 4: Run tests — all pass**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/channels/feishu.ts src/channels/feishu.test.ts
git commit -m "feat(feishu): implement non-text message placeholders and post extraction"
```

---

## Chunk 5: Outbound Messages & sender_name

### Task 9: Outbound messages (sendMessage)

- [ ] **Step 1: Write sendMessage tests**

Add to the `describe('FeishuChannel')` block:

```typescript
  describe('sendMessage', () => {
    it('sends a text message via API', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      await channel.sendMessage('feishu:oc_test123', 'Hello');

      expect(sdkRef.messageCreate).toHaveBeenCalledWith({
        params: { receive_id_type: 'chat_id' },
        data: {
          receive_id: 'oc_test123',
          msg_type: 'text',
          content: JSON.stringify({ text: 'Hello' }),
        },
      });
    });

    it('splits long messages into 4096-char chunks', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const longText = 'x'.repeat(5000);
      await channel.sendMessage('feishu:oc_test123', longText);

      expect(sdkRef.messageCreate).toHaveBeenCalledTimes(2);
      // First chunk: 4096 chars
      expect(sdkRef.messageCreate).toHaveBeenNthCalledWith(1, {
        params: { receive_id_type: 'chat_id' },
        data: {
          receive_id: 'oc_test123',
          msg_type: 'text',
          content: JSON.stringify({ text: 'x'.repeat(4096) }),
        },
      });
      // Second chunk: remaining 904 chars
      expect(sdkRef.messageCreate).toHaveBeenNthCalledWith(2, {
        params: { receive_id_type: 'chat_id' },
        data: {
          receive_id: 'oc_test123',
          msg_type: 'text',
          content: JSON.stringify({ text: 'x'.repeat(904) }),
        },
      });
    });

    it('handles send failure gracefully', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      sdkRef.messageCreate.mockRejectedValueOnce(new Error('Network error'));

      // Should not throw
      await expect(
        channel.sendMessage('feishu:oc_test123', 'Will fail'),
      ).resolves.toBeUndefined();
    });
  });
```

- [ ] **Step 2: Run tests — expect sendMessage tests to FAIL**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: sendMessage tests fail ("Not implemented").

- [ ] **Step 3: Implement sendMessage**

In `src/channels/feishu.ts`, replace the `sendMessage` method:

```typescript
  async sendMessage(jid: string, text: string): Promise<void> {
    if (!this.client) {
      logger.warn('Feishu client not initialized');
      return;
    }

    try {
      const chatId = jid.replace(/^feishu:/, '');
      const MAX_LENGTH = 4096;

      for (let i = 0; i < text.length; i += MAX_LENGTH) {
        await this.client.im.v1.message.create({
          params: { receive_id_type: 'chat_id' },
          data: {
            receive_id: chatId,
            msg_type: 'text',
            content: JSON.stringify({ text: text.slice(i, i + MAX_LENGTH) }),
          },
        });
      }

      logger.info({ jid, length: text.length }, 'Feishu message sent');
    } catch (err) {
      logger.error({ jid, err }, 'Failed to send Feishu message');
    }
  }
```

- [ ] **Step 4: Run tests — all pass**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/channels/feishu.ts src/channels/feishu.test.ts
git commit -m "feat(feishu): implement outbound message sending with chunking"
```

### Task 10: sender_name resolution

- [ ] **Step 1: Write sender_name tests**

Add to the `describe('FeishuChannel')` block:

```typescript
  describe('sender_name resolution', () => {
    it('resolves name via API on first lookup', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({});
      await triggerEvent('im.message.receive_v1', event);

      expect(sdkRef.userGet).toHaveBeenCalledWith(
        expect.objectContaining({
          path: { user_id: 'ou_sender001' },
          params: { user_id_type: 'open_id' },
        }),
      );
      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ sender_name: 'Test User' }),
      );
    });

    it('uses cache on repeated lookups', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      // Send two messages from the same sender
      const event1 = createMessageEvent({ messageId: 'om_1' });
      const event2 = createMessageEvent({ messageId: 'om_2' });
      await triggerEvent('im.message.receive_v1', event1);
      await triggerEvent('im.message.receive_v1', event2);

      // API should be called only once (second lookup hits cache)
      expect(sdkRef.userGet).toHaveBeenCalledTimes(1);
    });

    it('falls back to open_id when API fails', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      sdkRef.userGet.mockRejectedValueOnce(new Error('Permission denied'));

      const event = createMessageEvent({});
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onMessage).toHaveBeenCalledWith(
        'feishu:oc_test123',
        expect.objectContaining({ sender_name: 'ou_sender001' }),
      );
    });
  });
```

- [ ] **Step 2: Run tests — all pass (sender_name was already implemented in Task 6)**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: All tests pass (resolveSenderName was already implemented).

- [ ] **Step 3: Commit**

```bash
git add src/channels/feishu.test.ts
git commit -m "test(feishu): add sender_name resolution tests"
```

---

## Chunk 6: Chat Metadata & Final Integration

### Task 11: Chat metadata

- [ ] **Step 1: Write chat metadata tests**

Add to the `describe('FeishuChannel')` block:

```typescript
  describe('chat metadata', () => {
    it('emits metadata for registered group messages', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({});
      await triggerEvent('im.message.receive_v1', event);

      expect(opts.onChatMetadata).toHaveBeenCalledWith(
        'feishu:oc_test123',
        '2024-01-01T00:00:00.000Z',
        undefined,
        'feishu',
        true,
      );
    });

    it('emits metadata for unregistered chats (enables discovery)', async () => {
      const opts = createTestOpts();
      const channel = new FeishuChannel('id', 'secret', opts);
      await channel.connect();

      const event = createMessageEvent({ chatId: 'oc_unknown999' });
      await triggerEvent('im.message.receive_v1', event);

      // Metadata is emitted even though onMessage is not
      expect(opts.onChatMetadata).toHaveBeenCalledWith(
        'feishu:oc_unknown999',
        '2024-01-01T00:00:00.000Z',
        undefined,
        'feishu',
        true,
      );
      expect(opts.onMessage).not.toHaveBeenCalled();
    });
  });
```

- [ ] **Step 2: Run tests — all pass (metadata was already implemented in Task 6)**

Run: `npx vitest run src/channels/feishu.test.ts`
Expected: All tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/channels/feishu.test.ts
git commit -m "test(feishu): add chat metadata tests"
```

### Task 12: Full integration verification

- [ ] **Step 1: Run all tests**

Run: `npx vitest run`
Expected: All tests pass, including existing Telegram tests.

- [ ] **Step 2: Type-check**

Run: `npx tsc --noEmit`
Expected: No type errors. If there are SDK type issues, add targeted `as any` casts for SDK calls that don't have precise types.

- [ ] **Step 3: Build**

Run: `npm run build`
Expected: Clean compile. `dist/channels/feishu.js` exists.

- [ ] **Step 4: Format check**

Run: `npm run format:check`
Expected: All files formatted. If not, run `npm run format:fix` first.

- [ ] **Step 5: Final commit**

```bash
git add src/channels/feishu.ts src/channels/feishu.test.ts
git commit -m "feat(feishu): complete Feishu channel implementation

Add Feishu (飞书) as a built-in messaging channel supporting:
- WebSocket long connection via @larksuiteoapi/node-sdk
- Group chat and private chat with @mention translation
- Non-text message placeholders (image, file, audio, post, etc.)
- Post rich-text content extraction
- Sender name resolution with in-memory cache
- Message chunking for long outputs (4096 char limit)
- Chat metadata emission for discovery

25 unit tests with fully mocked SDK."
```

- [ ] **Step 6: Verify commit log**

Run: `git log --oneline feature/feishu-channel --not develop`
Expected: Clean sequence of commits from setup through implementation.
