---
name: add-feishu
description: Add Feishu (飞书/Lark) as a channel. Uses the official Lark Node.js SDK with long-polling event subscription (no public URL needed). Can run alongside other channels.
---

# Add Feishu Channel (飞书/Lark)

This skill adds Feishu (飞书) support to NanoClaw, then walks through interactive setup. Feishu is ByteDance's enterprise collaboration platform (international version: Lark).

## Phase 1: Pre-flight

### Check if already applied

Check if `src/channels/feishu.ts` exists. If it does, skip to Phase 3 (Setup). The code changes are already in place.

### Ask the user

AskUserQuestion: 你是否已经有飞书自建应用的凭证（App ID 和 App Secret）？/ Do you already have Feishu custom app credentials (App ID and App Secret)?

- **已有凭证 / Yes** — Collect App ID and App Secret now
- **还没有 / No** — We'll create one in Phase 3

## Phase 2: Apply Code Changes

Since there is no pre-built remote branch for Feishu, Claude Code generates the channel code directly.

### Install dependencies

```bash
npm install @larksuiteoapi/node-sdk
```

The `@larksuiteoapi/node-sdk` is the official Lark/Feishu Node.js SDK. It handles authentication, API calls, and event subscription.

### Create the channel file

Create `src/channels/feishu.ts` implementing the `Channel` interface. The implementation must:

1. **Self-register** via `registerChannel('feishu', factory)` at module load time
2. **Read credentials** from `.env` via `readEnvFile()` — required keys: `FEISHU_APP_ID`, `FEISHU_APP_SECRET`
3. **Factory returns null** when credentials are missing (channel is skipped)
4. **JID format**: `feishu:<chat_id>` for group chats, `feishu:<open_id>` for direct messages
5. **`ownsJid()`**: return `jid.startsWith('feishu:')`
6. **`connect()`**: Initialize the Lark SDK client, start the long-polling event dispatcher using `lark.EventDispatcher` with WebSocket mode (`lark.ws.WSClient`). Subscribe to `im.message.receive_v1` events. On receiving a message:
   - Extract `chat_id`, `open_id`, `message_id`, `msg_type`, `content` from the event
   - Only process `text` type messages (ignore images, files, etc. for now)
   - Parse JSON content body: `JSON.parse(event.message.content).text`
   - Determine sender name from `event.sender.sender_id.open_id` — fetch user info via `client.contact.user.get()` or use open_id as fallback
   - Determine if it's a group chat: `event.message.chat_type === 'group'`
   - Construct JID: `feishu:${event.message.chat_id}` for groups, `feishu:${event.sender.sender_id.open_id}` for P2P
   - Call `opts.onMessage(jid, msg)` with a `NewMessage` object
   - Call `opts.onChatMetadata(jid, timestamp, chatName, 'feishu', isGroup)`
7. **`sendMessage(jid, text)`**: Extract the chat_id or open_id from JID. Use `client.im.message.create()` to send:
   - `receive_id_type`: `'chat_id'` for group JIDs, `'open_id'` for DM JIDs
   - `receive_id`: the ID extracted from the JID
   - `msg_type`: `'text'`
   - `content`: `JSON.stringify({ text })`
   - Handle messages over 4000 characters by splitting
8. **`setTyping()`**: No-op (Feishu API does not support typing indicators)
9. **`disconnect()`**: Stop the WebSocket client
10. **`isConnected()`**: Track connection state via a boolean flag

**Key implementation details:**

```typescript
import * as lark from '@larksuiteoapi/node-sdk';
import { registerChannel } from './registry.js';
import { readEnvFile } from '../env.js';
import { logger } from '../logger.js';
import type { ChannelOpts } from './registry.js';
import type { Channel, NewMessage } from '../types.js';
```

The Lark SDK client is created as:

```typescript
const client = new lark.Client({
  appId: appId,
  appSecret: appSecret,
  loggerLevel: lark.LoggerLevel.WARN,
});
```

Event subscription uses WebSocket mode (no public URL required):

```typescript
const eventDispatcher = new lark.EventDispatcher({}).register({
  'im.message.receive_v1': async (data) => {
    // Handle incoming messages
  },
});
const wsClient = new lark.WSClient({
  appId,
  appSecret,
  eventDispatcher,
  loggerLevel: lark.LoggerLevel.WARN,
});
await wsClient.start();
```

### Create the test file

Create `src/channels/feishu.test.ts` with unit tests following the patterns in existing channel tests (e.g., `telegram.test.ts`). Tests should:

- Mock `@larksuiteoapi/node-sdk`
- Verify factory returns null when credentials are missing
- Verify `ownsJid()` correctly identifies `feishu:` prefixed JIDs
- Verify `sendMessage()` calls the Lark API with correct parameters
- Verify incoming message events are transformed into `NewMessage` objects correctly

### Register in barrel file

Append to `src/channels/index.ts`:

```typescript
import './feishu.js';
```

### Update .env.example

Add to `.env.example`:

```bash
FEISHU_APP_ID=
FEISHU_APP_SECRET=
```

### Validate code changes

```bash
npm install
npm run build
npx vitest run src/channels/feishu.test.ts
```

All tests must pass and build must be clean before proceeding.

## Phase 3: Setup

### Create Feishu Custom App (if needed)

If the user doesn't have credentials, tell them:

> 我需要你创建一个飞书自建应用：
>
> 1. 打开 [飞书开放平台](https://open.feishu.cn/app) 并登录
> 2. 点击 **创建自建应用**
> 3. 填写应用名称（如 "Andy 助手"）和描述
> 4. 创建后，在 **凭证与基础信息** 页面，复制 **App ID** 和 **App Secret**
> 5. 在左侧菜单进入 **添加应用能力** > **机器人**，开启机器人能力
> 6. 在 **权限管理** 中，申请以下权限：
>    - `im:message` — 获取与发送消息
>    - `im:message:send_as_bot` — 以机器人身份发送消息
>    - `im:chat:readonly` — 获取群组信息
>    - `contact:user.base:readonly` — 获取用户基本信息
> 7. 在 **事件订阅** 中：
>    - 选择 **使用长连接（WebSocket）接收事件**（无需配置公网地址）
>    - 订阅事件：`im.message.receive_v1`（接收消息）
> 8. 在 **版本管理与发布** 中，创建并发布一个版本
> 9. 如果是企业内部使用，需要管理员在 **管理后台** 审核通过
>
> ---
>
> I need you to create a Feishu custom app:
>
> 1. Open [Feishu Open Platform](https://open.feishu.cn/app) and sign in
> 2. Click **Create Custom App**
> 3. Fill in app name (e.g., "Andy Assistant") and description
> 4. After creation, go to **Credentials & Basic Info**, copy the **App ID** and **App Secret**
> 5. Go to **Add App Capability** > **Bot**, enable bot capability
> 6. In **Permissions**, request: `im:message`, `im:message:send_as_bot`, `im:chat:readonly`, `contact:user.base:readonly`
> 7. In **Event Subscriptions**: choose **Long Connection (WebSocket)** mode, subscribe to `im.message.receive_v1`
> 8. Create and publish a version in **Version Management**
> 9. For internal use, admin approval may be required

Wait for the user to provide the App ID and App Secret.

### Configure environment

Add to `.env`:

```bash
FEISHU_APP_ID=<their-app-id>
FEISHU_APP_SECRET=<their-app-secret>
```

Sync to container environment:

```bash
mkdir -p data/env && cp .env data/env/env
```

### Build and restart

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

## Phase 4: Registration

### Get Chat ID

Tell the user:

> 获取对话 ID：
>
> **私聊机器人：** 在飞书中搜索并打开刚创建的机器人，发送任意消息。查看应用日志获取 open_id。
>
> **群组：** 将机器人添加到群组中，发送 `@机器人名 hello`。NanoClaw 日志会显示 chat_id。
>
> 你也可以通过飞书开放平台的 [API 调试台](https://open.feishu.cn/api-explorer/) 调用 `im/v1/chats` 接口获取群组 chat_id。
>
> ---
>
> To get chat IDs:
>
> **DM the bot:** Search for your bot in Feishu and send any message. Check logs for the open_id.
>
> **Group:** Add the bot to a group, send `@BotName hello`. NanoClaw logs will show the chat_id.
>
> You can also use the [API Explorer](https://open.feishu.cn/api-explorer/) to call `im/v1/chats` to list chat IDs.

Wait for the user to provide the chat ID (format: `feishu:oc_xxxxx` for groups, `feishu:ou_xxxxx` for DMs).

### Register the chat

For a main chat (responds to all messages):

```typescript
registerGroup("feishu:<chat-id>", {
  name: "<chat-name>",
  folder: "feishu_main",
  trigger: `@${ASSISTANT_NAME}`,
  added_at: new Date().toISOString(),
  requiresTrigger: false,
  isMain: true,
});
```

For additional chats (trigger-only):

```typescript
registerGroup("feishu:<chat-id>", {
  name: "<chat-name>",
  folder: "feishu_<group-name>",
  trigger: `@${ASSISTANT_NAME}`,
  added_at: new Date().toISOString(),
  requiresTrigger: true,
});
```

## Phase 5: Verify

### Test the connection

Tell the user:

> 在注册的飞书对话中发送消息测试：
> - 主对话：直接发送任意消息
> - 其他群组：使用 `@Andy hello` 或 @机器人名 触发
>
> 机器人应在几秒内回复。
>
> ---
>
> Send a message in your registered Feishu chat:
> - Main chat: Any message works
> - Other groups: Use `@Andy hello` or @mention the bot
>
> The bot should respond within a few seconds.

### Check logs if needed

```bash
tail -f logs/nanoclaw.log
```

## Troubleshooting

### Bot not responding

Check:
1. `FEISHU_APP_ID` and `FEISHU_APP_SECRET` are set in `.env` AND synced to `data/env/env`
2. Chat is registered: `sqlite3 store/messages.db "SELECT * FROM registered_groups WHERE jid LIKE 'feishu:%'"`
3. App is published and approved in Feishu admin console
4. Bot capability is enabled on the app
5. Event subscription is set to WebSocket mode with `im.message.receive_v1` subscribed
6. Service is running: `launchctl list | grep nanoclaw` (macOS) or `systemctl --user status nanoclaw` (Linux)

### Bot connects but no messages received

1. Check event subscription mode is **WebSocket (长连接)**, not HTTP callback
2. Verify `im.message.receive_v1` event is subscribed
3. For group messages, the bot must be added to the group AND have `im:message` permission
4. Check if the app version is published and approved

### Permission denied errors

1. Verify all required permissions are approved in the Feishu admin console
2. Some permissions require admin approval — check the app's permission status page
3. After adding new permissions, you may need to publish a new app version

### Message content is empty

Feishu encrypts message content by default in some configurations. Ensure:
1. The app does NOT have an Encrypt Key configured (or handle decryption in the channel code)
2. The message type is `text` — other types (image, file, etc.) are not processed

## Known Limitations

- **Text messages only** — Images, files, rich text, and interactive cards are not processed. The bot only handles plain text messages.
- **No typing indicator** — Feishu API does not expose a typing status endpoint.
- **WebSocket mode only** — This implementation uses long-polling via WebSocket, which requires the app to be configured for WebSocket event delivery.
- **No thread support** — Feishu thread/reply messages are flattened into the channel context.

## Removal

To remove Feishu integration:

1. Delete `src/channels/feishu.ts` and `src/channels/feishu.test.ts`
2. Remove `import './feishu.js'` from `src/channels/index.ts`
3. Remove `FEISHU_APP_ID` and `FEISHU_APP_SECRET` from `.env`
4. Remove registrations: `sqlite3 store/messages.db "DELETE FROM registered_groups WHERE jid LIKE 'feishu:%'"`
5. Uninstall: `npm uninstall @larksuiteoapi/node-sdk`
6. Rebuild: `npm run build && launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `npm run build && systemctl --user restart nanoclaw` (Linux)
