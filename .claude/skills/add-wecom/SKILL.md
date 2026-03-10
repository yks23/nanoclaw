---
name: add-wecom
description: Add WeCom (企业微信/WeWork) as a channel. Uses the official WeCom Server API with callback URL or long-polling for event delivery. Suitable for enterprise use with official API support.
---

# Add WeCom Channel (企业微信)

This skill adds WeCom (企业微信, also known as WeWork internationally) support to NanoClaw. WeCom provides an official, well-documented Server API for building bots and integrations.

## Phase 1: Pre-flight

### Check if already applied

Check if `src/channels/wecom.ts` exists. If it does, skip to Phase 3 (Setup). The code changes are already in place.

### Ask the user

AskUserQuestion: 你是否已经有企业微信自建应用的凭证？/ Do you already have WeCom custom app credentials?

- **已有凭证 / Yes** — Collect Corp ID, Agent ID, Secret, Token, and EncodingAESKey now
- **还没有 / No** — We'll create one in Phase 3

AskUserQuestion: 你希望如何接收消息？/ How do you want to receive messages?

- **回调模式 / Callback mode**（需要公网可访问的 URL）— Requires a publicly accessible URL. More reliable, real-time delivery. Use with ngrok or a VPS.
- **轮询模式 / Polling mode**（无需公网地址）— No public URL needed. Uses the WeCom "sync message" API to poll for new messages periodically. Simpler setup, slight delivery delay.

## Phase 2: Apply Code Changes

### Install dependencies

```bash
npm install @wecom/crypto xml2js
npm install --save-dev @types/xml2js
```

- `@wecom/crypto`: Official WeCom message encryption/decryption library
- `xml2js`: XML parser for WeCom's XML message format

### Create the channel file

Create `src/channels/wecom.ts` implementing the `Channel` interface. The implementation must:

1. **Self-register** via `registerChannel('wecom', factory)` at module load time
2. **Read credentials** from `.env` via `readEnvFile()` — required keys: `WECOM_CORP_ID`, `WECOM_AGENT_ID`, `WECOM_SECRET`, `WECOM_TOKEN`, `WECOM_ENCODING_AES_KEY`; optional: `WECOM_CALLBACK_PORT` (default 8080), `WECOM_MODE` ('callback' or 'poll', default 'poll')
3. **Factory returns null** when required credentials (`WECOM_CORP_ID`, `WECOM_SECRET`) are missing
4. **JID format**: `wecom:<userid>` for DMs (single chat), `wecom:group_<chatid>` for group chats
5. **`ownsJid()`**: return `jid.startsWith('wecom:')`

### Access Token Management

WeCom API requires an `access_token` that expires every 2 hours. Implement a token manager:

```typescript
let accessToken = '';
let tokenExpiresAt = 0;

async function getAccessToken(): Promise<string> {
  if (accessToken && Date.now() < tokenExpiresAt - 60000) {
    return accessToken;
  }
  const url = `https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=${corpId}&corpsecret=${secret}`;
  const res = await fetch(url);
  const data = await res.json();
  if (data.errcode !== 0) {
    throw new Error(`WeCom token error: ${data.errmsg}`);
  }
  accessToken = data.access_token;
  tokenExpiresAt = Date.now() + data.expires_in * 1000;
  return accessToken;
}
```

### Callback Mode Implementation

6a. **`connect()` (callback mode)**: Start an HTTP server to receive WeCom event callbacks:

```typescript
import { createServer } from 'http';
import { parseStringPromise } from 'xml2js';
import { getSignature, decrypt } from '@wecom/crypto';

const server = createServer(async (req, res) => {
  const url = new URL(req.url!, `http://localhost`);
  const params = url.searchParams;

  // URL verification (GET request from WeCom during callback setup)
  if (req.method === 'GET') {
    const { msg_signature, timestamp, nonce, echostr } = Object.fromEntries(params);
    // Verify signature and decrypt echostr
    const { message } = decrypt(encodingAESKey, echostr);
    res.end(message);
    return;
  }

  // Message callback (POST request)
  if (req.method === 'POST') {
    const body = await readBody(req);
    const xml = await parseStringPromise(body);
    const encrypted = xml.xml.Encrypt[0];

    // Verify signature
    const timestamp = params.get('timestamp')!;
    const nonce = params.get('nonce')!;
    const expectedSig = params.get('msg_signature')!;
    const sig = getSignature(token, timestamp, nonce, encrypted);
    if (sig !== expectedSig) {
      res.writeHead(403);
      res.end('Invalid signature');
      return;
    }

    // Decrypt message
    const { message } = decrypt(encodingAESKey, encrypted);
    const msgXml = await parseStringPromise(message);
    const msgType = msgXml.xml.MsgType[0];

    if (msgType === 'text') {
      const content = msgXml.xml.Content[0];
      const fromUser = msgXml.xml.FromUserName[0];
      // Determine if group or DM based on message structure
      const jid = `wecom:${fromUser}`;

      // Fetch sender name via WeCom user API
      const senderName = await getUserName(fromUser);

      const msg: NewMessage = {
        id: msgXml.xml.MsgId?.[0] || `wecom-${Date.now()}`,
        chat_jid: jid,
        sender: fromUser,
        sender_name: senderName,
        content: content,
        timestamp: new Date().toISOString(),
        is_from_me: false,
      };

      opts.onMessage(jid, msg);
      opts.onChatMetadata(jid, msg.timestamp, senderName, 'wecom', false);
    }

    res.writeHead(200);
    res.end('success');
  }
});

server.listen(callbackPort, () => {
  logger.info({ port: callbackPort }, 'WeCom callback server started');
  connected = true;
});
```

### Polling Mode Implementation

6b. **`connect()` (polling mode)**: Poll the WeCom sync API periodically:

```typescript
// WeCom provides a customer-service message sync API for polling:
// https://qyapi.weixin.qq.com/cgi-bin/kf/sync_msg

// For internal apps, use the chat/message API or simply
// rely on callback mode. For polling-only mode without
// a public URL, implement a timer that fetches recent messages.

let pollCursor = '';

async function pollMessages(): Promise<void> {
  const token = await getAccessToken();
  // Use the external contact message sync API or application message sync
  const url = `https://qyapi.weixin.qq.com/cgi-bin/message/get_statistics?access_token=${token}`;
  // Note: WeCom's internal messaging API is callback-based.
  // For polling mode, the recommended approach is:
  // 1. Use the WeCom "receive messages" callback to a local server
  // 2. Use ngrok or similar to expose the local server
  // Alternatively, for simpler setups without a public URL,
  // implement the WeCom customer service (微信客服) API
  // which supports the sync_msg polling endpoint.
}

const pollInterval = setInterval(pollMessages, 5000);
```

**Important Note for Claude Code**: WeCom's internal app messaging API is fundamentally callback-based. For true polling without a public URL, the recommended approach is to use the **WeCom Customer Service (微信客服)** module which provides a `sync_msg` API. If the user cannot set up a callback URL, implement the customer service message sync endpoint instead. Inform the user about this limitation during setup.

7. **`sendMessage(jid, text)`**: Use the WeCom message send API:

```typescript
async function sendMessage(jid: string, text: string): Promise<void> {
  const token = await getAccessToken();
  const id = jid.replace(/^wecom:/, '').replace(/^group_/, '');
  const isGroup = jid.startsWith('wecom:group_');

  // Split long messages
  const chunks = splitMessage(text, 2048);

  for (const chunk of chunks) {
    const body = isGroup
      ? {
          chatid: id,
          msgtype: 'text',
          text: { content: chunk },
          safe: 0,
        }
      : {
          touser: id,
          msgtype: 'text',
          agentid: parseInt(agentId, 10),
          text: { content: chunk },
          safe: 0,
        };

    const url = isGroup
      ? `https://qyapi.weixin.qq.com/cgi-bin/appchat/send?access_token=${token}`
      : `https://qyapi.weixin.qq.com/cgi-bin/message/send?access_token=${token}`;

    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });

    const data = await res.json();
    if (data.errcode !== 0) {
      logger.error({ errcode: data.errcode, errmsg: data.errmsg, jid }, 'WeCom send failed');
    }
  }
}
```

8. **`setTyping()`**: No-op (WeCom does not support typing indicators)
9. **`disconnect()`**: Stop the callback server or polling timer
10. **`isConnected()`**: Return tracked connection state

**Helper: Get user name from WeCom API:**

```typescript
const userNameCache = new Map<string, string>();

async function getUserName(userId: string): Promise<string> {
  if (userNameCache.has(userId)) return userNameCache.get(userId)!;
  try {
    const token = await getAccessToken();
    const res = await fetch(
      `https://qyapi.weixin.qq.com/cgi-bin/user/get?access_token=${token}&userid=${userId}`
    );
    const data = await res.json();
    const name = data.name || userId;
    userNameCache.set(userId, name);
    return name;
  } catch {
    return userId;
  }
}
```

### Create the test file

Create `src/channels/wecom.test.ts` with unit tests. Mock `fetch` and `@wecom/crypto`. Tests should:

- Verify factory returns null when `WECOM_CORP_ID` or `WECOM_SECRET` is missing
- Verify `ownsJid()` correctly identifies `wecom:` prefixed JIDs
- Verify access token fetching and caching
- Verify `sendMessage()` constructs correct API calls for DMs vs groups
- Verify callback signature verification
- Verify XML message parsing and transformation into `NewMessage`

### Register in barrel file

Append to `src/channels/index.ts`:

```typescript
import './wecom.js';
```

### Update .env.example

Add to `.env.example`:

```bash
WECOM_CORP_ID=
WECOM_AGENT_ID=
WECOM_SECRET=
WECOM_TOKEN=
WECOM_ENCODING_AES_KEY=
WECOM_MODE=callback
WECOM_CALLBACK_PORT=8080
```

### Validate code changes

```bash
npm install
npm run build
npx vitest run src/channels/wecom.test.ts
```

All tests must pass and build must be clean before proceeding.

## Phase 3: Setup

### Create WeCom Custom App (if needed)

If the user doesn't have credentials, tell them:

> 我需要你创建一个企业微信自建应用：
>
> 1. 打开 [企业微信管理后台](https://work.weixin.qq.com/wework_admin/frame) 并登录
> 2. 进入 **应用管理** > **自建** > **创建应用**
> 3. 填写应用名称（如 "Andy 助手"）、应用 logo 和可见范围
> 4. 创建后，记录以下信息：
>    - **AgentId**：应用详情页面顶部
>    - **Secret**：应用详情页面，点击查看
> 5. 回到 **我的企业** 页面（左上角），记录 **企业 ID (Corp ID)**
> 6. 在应用详情页面，进入 **接收消息** 设置：
>    - 点击 **设置API接收**
>    - 记录自动生成的 **Token** 和 **EncodingAESKey**
>    - URL 填写你的回调地址（如 `https://your-domain.com/wecom/callback`）
>    - 如果没有公网地址，可以使用 ngrok：`ngrok http 8080`
>
> ---
>
> I need you to create a WeCom custom app:
>
> 1. Open [WeCom Admin Console](https://work.weixin.qq.com/wework_admin/frame) and sign in
> 2. Go to **App Management** > **Custom** > **Create App**
> 3. Fill in app name (e.g., "Andy Assistant"), logo, and visibility scope
> 4. After creation, note the following:
>    - **AgentId**: At the top of the app detail page
>    - **Secret**: On the app detail page, click to view
> 5. Go to **My Enterprise** (top-left), note the **Corp ID**
> 6. On the app detail page, go to **Receive Messages** settings:
>    - Click **Set API Receive**
>    - Note the auto-generated **Token** and **EncodingAESKey**
>    - Enter your callback URL (e.g., `https://your-domain.com/wecom/callback`)
>    - If you don't have a public URL, use ngrok: `ngrok http 8080`

Wait for the user to provide all credentials.

### Configure environment

Add to `.env`:

```bash
WECOM_CORP_ID=<corp-id>
WECOM_AGENT_ID=<agent-id>
WECOM_SECRET=<secret>
WECOM_TOKEN=<token>
WECOM_ENCODING_AES_KEY=<encoding-aes-key>
WECOM_MODE=callback
WECOM_CALLBACK_PORT=8080
```

If using ngrok:

```bash
# Start ngrok (in a separate terminal or as background process)
ngrok http 8080
```

Copy the ngrok HTTPS URL and configure it as the callback URL in WeCom admin console.

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

### Get User / Chat IDs

Tell the user:

> 获取对话 ID：
>
> **单聊（个人消息）：** 在企业微信中向自建应用发送一条消息。NanoClaw 日志会显示发送者的 UserID。
> JID 格式：`wecom:<userid>`
>
> **群聊：** 通过企业微信 API 创建群聊，或在管理后台查看群聊 ID。
> JID 格式：`wecom:group_<chatid>`
>
> 你也可以通过 [企业微信 API 调试工具](https://developer.work.weixin.qq.com/resource/devtool) 获取 UserID 和 ChatID。
>
> ---
>
> To get chat IDs:
>
> **DM (personal message):** Send a message to the custom app in WeCom. NanoClaw logs will show the sender's UserID.
> JID format: `wecom:<userid>`
>
> **Group:** Create a group via WeCom API, or find the chat ID in the admin console.
> JID format: `wecom:group_<chatid>`
>
> You can also use the [WeCom API Debug Tool](https://developer.work.weixin.qq.com/resource/devtool) to look up UserIDs and ChatIDs.

Wait for the user to provide the ID.

### Register the chat

For a main chat (responds to all messages):

```typescript
registerGroup("wecom:<id>", {
  name: "<chat-name>",
  folder: "wecom_main",
  trigger: `@${ASSISTANT_NAME}`,
  added_at: new Date().toISOString(),
  requiresTrigger: false,
  isMain: true,
});
```

For additional chats (trigger-only):

```typescript
registerGroup("wecom:<id>", {
  name: "<chat-name>",
  folder: "wecom_<group-name>",
  trigger: `@${ASSISTANT_NAME}`,
  added_at: new Date().toISOString(),
  requiresTrigger: true,
});
```

## Phase 5: Verify

### Test the connection

Tell the user:

> 在企业微信中向注册的对话发送消息测试：
> - 主对话：直接发送任意消息
> - 其他群聊：使用触发词（如 `@Andy hello`）
>
> 机器人应在几秒内回复。
>
> ---
>
> Send a message in your registered WeCom chat:
> - Main chat: Any message works
> - Other groups: Use the trigger word (e.g., `@Andy hello`)
>
> The bot should respond within a few seconds.

### Check logs if needed

```bash
tail -f logs/nanoclaw.log
```

## Troubleshooting

### Bot not responding

Check:
1. All WeCom credentials are set in `.env` AND synced to `data/env/env`
2. Chat is registered: `sqlite3 store/messages.db "SELECT * FROM registered_groups WHERE jid LIKE 'wecom:%'"`
3. Callback URL is accessible from the internet (test with `curl`)
4. Service is running: `launchctl list | grep nanoclaw` (macOS) or `systemctl --user status nanoclaw` (Linux)

### Callback verification fails

1. Check `WECOM_TOKEN` and `WECOM_ENCODING_AES_KEY` match the values in WeCom admin console exactly
2. Make sure the callback port matches `WECOM_CALLBACK_PORT` in `.env`
3. If using ngrok, verify the ngrok tunnel is active: `curl -s localhost:4040/api/tunnels`

### Access token errors

1. Check `WECOM_CORP_ID` and `WECOM_SECRET` are correct
2. Verify the app's IP whitelist allows your server's IP (if configured)
3. Token errors often indicate the secret was copied incorrectly — try re-copying from the admin console

### Messages received but not sent

1. Check the agent ID matches the app that has send permissions
2. Verify the `touser` value matches a valid WeCom user ID
3. Check API response for error codes: `tail -50 logs/nanoclaw.log | grep "WeCom send"`

### Encryption / decryption errors

1. Verify `WECOM_ENCODING_AES_KEY` is exactly 43 characters (Base64 encoded)
2. Check that `@wecom/crypto` is installed: `npm ls @wecom/crypto`
3. The key must match what's shown in the WeCom admin console — re-copy if needed

### ngrok tunnel expired

Free ngrok tunnels have session limits. For production use:
1. Get a paid ngrok plan, or
2. Use a VPS with a static IP, or
3. Use Cloudflare Tunnel (free): `cloudflared tunnel --url http://localhost:8080`

## Known Limitations

- **Text messages only** — Images, voice, video, files, and location messages are not processed.
- **No typing indicator** — WeCom API does not support typing status.
- **Callback mode requires public URL** — The callback server must be accessible from WeCom servers. Use ngrok, Cloudflare Tunnel, or a VPS.
- **Access token expiry** — Tokens expire every 2 hours and are auto-refreshed, but network issues may cause brief outages.
- **Group chat limitations** — WeCom group chats (appchat) have a member limit and require API creation. They differ from regular WeCom groups.
- **No message history API for internal apps** — WeCom internal apps can only receive messages via callbacks, not pull historical messages. Messages sent while the service is offline are lost.
- **IP whitelist** — Some WeCom configurations require adding your server IP to an allowlist in the admin console.

## Removal

To remove WeCom integration:

1. Delete `src/channels/wecom.ts` and `src/channels/wecom.test.ts`
2. Remove `import './wecom.js'` from `src/channels/index.ts`
3. Remove all `WECOM_*` variables from `.env`
4. Remove registrations: `sqlite3 store/messages.db "DELETE FROM registered_groups WHERE jid LIKE 'wecom:%'"`
5. Uninstall: `npm uninstall @wecom/crypto xml2js @types/xml2js`
6. Rebuild: `npm run build && launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `npm run build && systemctl --user restart nanoclaw` (Linux)
