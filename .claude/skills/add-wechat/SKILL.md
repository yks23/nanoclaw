---
name: add-wechat
description: Add WeChat (微信) as a channel. Uses Wechaty with a pluggable puppet system for WeChat connectivity. Authenticates via QR code scan. Can run alongside other channels.
---

# Add WeChat Channel (微信)

This skill adds WeChat (微信) support to NanoClaw using the Wechaty framework. Wechaty provides a universal bot SDK with pluggable "puppet" implementations for WeChat connectivity.

## Important Notice

> **⚠️ WeChat 个人号接口说明 / WeChat Personal Account API Notice**
>
> 微信没有官方的个人号机器人 API。此集成使用 Wechaty 框架，通过不同的 puppet 实现连接微信。
> 请注意：使用非官方接口可能违反微信使用条款，存在封号风险。建议使用专用的微信号，不要使用主力账号。
>
> WeChat does not provide an official bot API for personal accounts. This integration uses the Wechaty framework with pluggable puppet implementations.
> Note: Using unofficial interfaces may violate WeChat's Terms of Service and risk account suspension. Use a dedicated WeChat account, not your primary one.

## Phase 1: Pre-flight

### Check if already applied

Check if `src/channels/wechat.ts` exists. If it does, skip to Phase 3 (Setup).

### Ask the user

AskUserQuestion: 你想使用哪种 Wechaty Puppet？/ Which Wechaty puppet do you want to use?

- **wechaty-puppet-wechat4u**（免费，基于网页版协议，功能有限）/ Free, web protocol based, limited features
- **wechaty-puppet-padlocal**（付费，iPad 协议，功能完整，需要 token）/ Paid, iPad protocol, full features, requires token
- **wechaty-puppet-xp**（免费，Windows 桌面协议，需要 Windows 环境）/ Free, Windows desktop protocol, requires Windows

If they chose padlocal: AskUserQuestion: 请提供你的 PadLocal Token / Please provide your PadLocal token

## Phase 2: Apply Code Changes

### Install dependencies

Based on the chosen puppet:

**For wechaty-puppet-wechat4u (recommended for simplicity):**

```bash
npm install wechaty wechaty-puppet-wechat4u
```

**For wechaty-puppet-padlocal:**

```bash
npm install wechaty wechaty-puppet-padlocal
```

**For wechaty-puppet-xp:**

```bash
npm install wechaty wechaty-puppet-xp
```

### Create the channel file

Create `src/channels/wechat.ts` implementing the `Channel` interface. The implementation must:

1. **Self-register** via `registerChannel('wechat', factory)` at module load time
2. **Read credentials** from `.env` via `readEnvFile()` — key: `WECHATY_PUPPET` (puppet name), optionally `WECHATY_PUPPET_TOKEN` (for paid puppets like padlocal)
3. **Factory returns null** when `WECHATY_PUPPET` is missing
4. **JID format**: `wx:<contact_id>` for DMs, `wx:room_<room_id>` for group chats (rooms)
5. **`ownsJid()`**: return `jid.startsWith('wx:')`
6. **`connect()`**: Initialize and start the Wechaty bot:

```typescript
import { WechatyBuilder, ScanStatus } from 'wechaty';

const bot = WechatyBuilder.build({
  name: 'nanoclaw-wechat',
  puppet: puppetName,      // e.g., 'wechaty-puppet-wechat4u'
  puppetOptions: {
    token: puppetToken,    // only for paid puppets
  },
});

bot.on('scan', (qrcode, status) => {
  if (status === ScanStatus.Waiting || status === ScanStatus.Timeout) {
    const qrcodeUrl = `https://wechaty.js.org/qrcode/${encodeURIComponent(qrcode)}`;
    logger.info({ qrcodeUrl }, 'WeChat: Scan QR code to log in');
    // Also log the URL so users can find it
  }
});

bot.on('login', (user) => {
  logger.info({ user: user.name() }, 'WeChat: Logged in');
  connected = true;
});

bot.on('logout', (user) => {
  logger.info({ user: user.name() }, 'WeChat: Logged out');
  connected = false;
});

bot.on('message', async (message) => {
  // Skip self-sent messages and non-text
  if (message.self()) return;
  if (message.type() !== bot.Message.Type.Text) return;

  const room = message.room();
  const talker = message.talker();
  const isGroup = !!room;

  const jid = isGroup
    ? `wx:room_${room!.id}`
    : `wx:${talker.id}`;

  const senderName = talker.name() || talker.id;
  const chatName = isGroup
    ? (await room!.topic() || room!.id)
    : senderName;

  const msg: NewMessage = {
    id: message.id || `wx-${Date.now()}`,
    chat_jid: jid,
    sender: talker.id,
    sender_name: senderName,
    content: message.text(),
    timestamp: new Date().toISOString(),
    is_from_me: false,
  };

  opts.onMessage(jid, msg);
  opts.onChatMetadata(jid, msg.timestamp, chatName, 'wechat', isGroup);
});

await bot.start();
```

7. **`sendMessage(jid, text)`**: Extract the ID from JID. For room JIDs (`wx:room_xxx`), find the room and send. For DM JIDs (`wx:xxx`), find the contact and send:

```typescript
if (jid.startsWith('wx:room_')) {
  const roomId = jid.slice('wx:room_'.length);
  const room = await bot.Room.find({ id: roomId });
  if (room) await room.say(text);
} else {
  const contactId = jid.slice('wx:'.length);
  const contact = await bot.Contact.find({ id: contactId });
  if (contact) await contact.say(text);
}
```

Handle long messages by splitting at 2000 characters.

8. **`setTyping()`**: No-op (WeChat does not support typing indicators via Wechaty)
9. **`disconnect()`**: `await bot.stop()`
10. **`isConnected()`**: Return the tracked connection state

**Key imports:**

```typescript
import { WechatyBuilder, ScanStatus } from 'wechaty';
import { registerChannel } from './registry.js';
import { readEnvFile } from '../env.js';
import { logger } from '../logger.js';
import type { ChannelOpts } from './registry.js';
import type { Channel, NewMessage } from '../types.js';
```

### Create the test file

Create `src/channels/wechat.test.ts` with unit tests. Mock the `wechaty` module. Tests should:

- Verify factory returns null when `WECHATY_PUPPET` env var is missing
- Verify `ownsJid()` correctly identifies `wx:` prefixed JIDs
- Verify `sendMessage()` calls the correct Wechaty methods for rooms vs contacts
- Verify incoming message events are transformed into `NewMessage` objects correctly
- Verify self-sent messages are ignored
- Verify non-text messages are ignored

### Register in barrel file

Append to `src/channels/index.ts`:

```typescript
import './wechat.js';
```

### Update .env.example

Add to `.env.example`:

```bash
WECHATY_PUPPET=wechaty-puppet-wechat4u
WECHATY_PUPPET_TOKEN=
```

### Validate code changes

```bash
npm install
npm run build
npx vitest run src/channels/wechat.test.ts
```

All tests must pass and build must be clean before proceeding.

## Phase 3: Setup

### QR Code Authentication

Tell the user:

> 微信登录需要扫描二维码。启动服务后，日志中会出现二维码 URL。
>
> 1. 启动 NanoClaw：`npm run dev`（开发模式）
> 2. 在日志中找到 QR code URL（格式：`https://wechaty.js.org/qrcode/...`）
> 3. 用手机微信扫描该二维码
> 4. 在手机上确认登录
>
> ---
>
> WeChat login requires scanning a QR code:
>
> 1. Start NanoClaw: `npm run dev` (development mode)
> 2. Find the QR code URL in the logs (format: `https://wechaty.js.org/qrcode/...`)
> 3. Open the URL in a browser to see the QR code, then scan with WeChat on your phone
> 4. Confirm login on your phone
>
> **Note:** If using wechaty-puppet-wechat4u, the web WeChat protocol requires your account to have web login enabled. Some accounts created after 2017 may not support this — in that case, use the padlocal puppet.

### Configure environment

Add to `.env`:

```bash
WECHATY_PUPPET=<chosen-puppet>
WECHATY_PUPPET_TOKEN=<token-if-needed>
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

### Get Contact / Room IDs

Tell the user:

> 获取对话 ID：
>
> **私聊：** 给机器人微信号发送一条消息，查看 NanoClaw 日志中的 contact ID。
>
> **群聊：** 将机器人拉入群聊，发送 `@Andy hello`（或配置的触发词），查看日志中的 room ID。
>
> ---
>
> To get chat IDs:
>
> **DM:** Send a message to the bot's WeChat account. Check NanoClaw logs for the contact ID.
>
> **Group:** Add the bot to a group chat, send `@Andy hello`. Check logs for the room ID.

Wait for the user to provide the ID (format: `wx:<contact_id>` or `wx:room_<room_id>`).

### Register the chat

For a main chat (responds to all messages):

```typescript
registerGroup("wx:<id>", {
  name: "<chat-name>",
  folder: "wechat_main",
  trigger: `@${ASSISTANT_NAME}`,
  added_at: new Date().toISOString(),
  requiresTrigger: false,
  isMain: true,
});
```

For additional chats (trigger-only):

```typescript
registerGroup("wx:<id>", {
  name: "<chat-name>",
  folder: "wechat_<group-name>",
  trigger: `@${ASSISTANT_NAME}`,
  added_at: new Date().toISOString(),
  requiresTrigger: true,
});
```

## Phase 5: Verify

### Test the connection

Tell the user:

> 在注册的微信对话中发送消息测试：
> - 主对话：直接发送任意消息
> - 其他群聊：使用触发词（如 `@Andy hello`）
>
> 机器人应在几秒内回复。
>
> ---
>
> Send a message in your registered WeChat chat:
> - Main chat: Any message works
> - Other groups: Use the trigger word (e.g., `@Andy hello`)
>
> The bot should respond within a few seconds.

### Check logs if needed

```bash
tail -f logs/nanoclaw.log
```

## Troubleshooting

### QR code not appearing

1. Check `WECHATY_PUPPET` is set in `.env`
2. Check the puppet package is installed: `npm ls wechaty`
3. Check logs for errors: `tail -50 logs/nanoclaw.log`

### "Web login is not supported" error

Some WeChat accounts (especially those created after 2017) cannot use web-based login. Solutions:
1. Use `wechaty-puppet-padlocal` (paid, most reliable) or `wechaty-puppet-xp` (Windows only)
2. Try enabling web login by logging into `wx.qq.com` in a browser first

### Session expired / logged out

WeChat sessions expire periodically. When this happens:
1. The bot will emit a `logout` event
2. A new QR code will be generated in the logs
3. Scan the new QR code to re-authenticate

### Bot not receiving group messages

1. In WeChat groups, the bot may need to be @mentioned to trigger Wechaty's message handler (depends on the puppet)
2. Check that the group is registered in NanoClaw
3. Verify the trigger pattern matches

### Bot responds twice or in wrong chat

1. Verify the correct JID is registered (check room ID vs contact ID)
2. Ensure only one NanoClaw instance is running with this WeChat account

## Known Limitations

- **Text messages only** — Images, voice messages, mini-programs, and other rich content types are not processed.
- **No typing indicator** — WeChat does not expose a typing status API through Wechaty.
- **Session persistence varies** — Depending on the puppet, sessions may expire and require re-scanning the QR code.
- **Web protocol limitations** — The free `wechaty-puppet-wechat4u` puppet uses the web WeChat protocol, which has limited features and may not work on all accounts.
- **No official API** — This integration relies on reverse-engineered protocols. WeChat may change these at any time.
- **Single account** — Only one WeChat account can be connected per NanoClaw instance.

## Removal

To remove WeChat integration:

1. Delete `src/channels/wechat.ts` and `src/channels/wechat.test.ts`
2. Remove `import './wechat.js'` from `src/channels/index.ts`
3. Remove `WECHATY_PUPPET` and `WECHATY_PUPPET_TOKEN` from `.env`
4. Remove registrations: `sqlite3 store/messages.db "DELETE FROM registered_groups WHERE jid LIKE 'wx:%'"`
5. Uninstall: `npm uninstall wechaty wechaty-puppet-wechat4u` (or whichever puppet was installed)
6. Rebuild: `npm run build && launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `npm run build && systemctl --user restart nanoclaw` (Linux)
