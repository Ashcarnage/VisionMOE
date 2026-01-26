# LisnAI - Technical Architecture & Implementation Plan v2

## Learnings from Clawdbot Analysis

After deep analysis of Clawdbot (8,000+ stars, built by Peter Steinberger), we've identified key architectural patterns to adopt for LisnAI:

### Key Patterns to Adopt

| Pattern | Clawdbot Implementation | LisnAI Adaptation |
|---------|------------------------|-------------------|
| **Gateway Architecture** | Central WebSocket control plane (`ws://127.0.0.1:18789`) that all clients connect to | WebSocket server for real-time iOS sync, action streaming, and live transcription updates |
| **Multi-Channel Messaging** | Gateway owns ALL channel connections (WhatsApp via Baileys, Discord via Bot API, Telegram via grammY). iOS does NOT connect directly. | Gateway connects to WhatsApp, Discord, Telegram, Instagram. iOS sends `chat.send` with channel param. |
| **Node Pattern** | iOS/Android/macOS apps connect as "nodes" with capabilities (camera, screen, voice, location) | iOS app as a node that exposes device capabilities to the backend |
| **Skills System** | Plugins defined via SKILL.md with install gating and runtime discovery | Modular action handlers that can be extended without code changes |
| **Memory System** | Hybrid vector + keyword search with markdown files, session transcripts as JSONL | Combine Qdrant vectors with Firestore keyword search, store transcripts as structured JSONL |
| **Tool Architecture** | TypeBox schemas with strict validation, ~50 tools covering browser/canvas/nodes/sessions | Zod schemas with strict validation, tools for iOS actions |
| **Session Routing** | Per-peer sessions with routing bindings, session keys like `agent:main:per-peer` | Per-device sessions with time-based partitioning |
| **Proactive Features** | Cron jobs for briefings, scheduled notifications | Background jobs via Inngest for daily summaries, proactive reminders |
| **Wake Word** | Swabble daemon with on-device Speech.framework detection | Integrate with iOS SpeechAnalyzer for wake word |
| **Device Commands** | `node.invoke()` pattern for camera_snap, screen_record, notify, location_get, system.run | Deep link actions + Shortcuts integration for iOS |

### Critical Insight: Multi-Channel Architecture

**iOS does NOT connect directly to WhatsApp, Discord, Telegram, etc.**

Instead:
1. **Gateway owns ALL channel connections** (tokens, SDKs, sockets)
2. **iOS connects ONLY to Gateway** via WebSocket
3. iOS sends `chat.send` with `channel` parameter
4. Gateway routes to appropriate platform

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MESSAGING PLATFORMS                                   │
│    WhatsApp    Discord    Telegram    Instagram    Signal    iMessage        │
│       │           │           │           │           │          │           │
│   (Baileys)  (discord.js) (grammY)    (API)    (signal-cli)  (native)       │
│       │           │           │           │           │          │           │
│       └───────────┴───────────┴─────┬─────┴───────────┴──────────┘           │
│                                     │                                        │
│                          GATEWAY OWNS ALL                                    │
│                        CHANNEL CONNECTIONS                                   │
│                                     │                                        │
└─────────────────────────────────────┼────────────────────────────────────────┘
                                      │
                                 WebSocket
                                      │
┌─────────────────────────────────────┴────────────────────────────────────────┐
│                              iOS APP (Node)                                   │
│                                                                               │
│   • Connects ONLY to Gateway (single WebSocket)                              │
│   • Sends: chat.send { channel: "whatsapp", message: "Hello" }               │
│   • Receives: messages from ALL configured channels                           │
│   • NO WhatsApp SDK, NO Discord SDK in iOS app                               │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Architecture Comparison

```
CLAWDBOT                                    LISNAI (Updated)
=========                                   =================

Gateway (WebSocket)                         Gateway (WebSocket + REST)
    ↓                                           ↓
Multiple Channels                           iOS Node (primary)
(WhatsApp, Telegram, Discord, etc.)         (Future: watchOS, macOS, web)
    ↓                                           ↓
Pi Agent (Claude/OpenAI)                    Agent Pipeline
    ↓                                       (Entity → Memory → Router → Action)
Skills + Tools                                  ↓
    ↓                                       Skills (extensible handlers)
Nodes (iOS/Android/macOS)                       ↓
    ↓                                       Firebase + Qdrant
Memory (sqlite-vec + markdown)
```

---

## Updated Tech Stack

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              LISNAI TECH STACK v2                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┬────────────────────────────────────────────────────────────────┐
│      LAYER          │                         TECHNOLOGY                             │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  iOS Node           │  Swift 6.0, SwiftUI, FluidAudio, SpeechAnalyzer                │
│                     │  Gateway WebSocket client (NO direct channel SDKs)             │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Runtime            │  Bun (primary) / Node.js 22+ (fallback)                        │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  API Framework      │  Hono (REST) + Bun WebSocket (real-time)                       │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Gateway Protocol   │  WebSocket JSON-RPC for iOS ↔ Backend sync                     │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  MESSAGING CHANNELS │  Gateway owns ALL connections:                                 │
│  (Gateway-side)     │  • WhatsApp: @whiskeysockets/baileys (Web protocol)            │
│                     │  • Discord: discord.js (Bot API + Gateway)                     │
│                     │  • Telegram: grammY (Bot API)                                  │
│                     │  • Instagram: instagram-private-api (unofficial)               │
│                     │  • Signal: signal-cli (wrapper)                                │
│                     │  • iMessage: via BlueBubbles or native macOS                   │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Validation         │  Zod (TypeScript-first, strict schemas like Clawdbot TypeBox)  │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  AI/Agents          │  Vercel AI SDK 6 (ToolLoopAgent, generateObject)               │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  LLM Gateway        │  LiteLLM Proxy (unified API, model fallbacks)                  │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Database           │  Firebase Firestore (documents) + Qdrant (vectors)             │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Memory Search      │  Hybrid: Qdrant vector + Firestore keyword (like Clawdbot)     │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Authentication     │  Firebase Auth + Gateway token pairing (like Clawdbot nodes)   │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Background Jobs    │  Inngest (daily summaries, proactive briefings)                │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Observability      │  Langfuse TypeScript SDK v4                                    │
├─────────────────────┼────────────────────────────────────────────────────────────────┤
│  Encryption         │  TweetNaCl.js (E2EE), TLS 1.3                                  │
└─────────────────────┴────────────────────────────────────────────────────────────────┘
```

---

## Updated Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           MESSAGING PLATFORMS (External)                             │
│                        Gateway maintains ALL these connections                       │
│                                                                                      │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│   │WhatsApp │ │ Discord │ │Telegram │ │Instagram│ │ Signal  │ │iMessage │          │
│   │(Baileys)│ │(disc.js)│ │(grammY) │ │ (API)   │ │(sig-cli)│ │(BlueBub)│          │
│   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
│        │           │           │           │           │           │                │
│        └───────────┴───────────┴─────┬─────┴───────────┴───────────┘                │
│                                      │                                               │
│                            GATEWAY OWNS ALL                                          │
│                          (tokens, SDKs, sockets)                                     │
│                                      │                                               │
└──────────────────────────────────────┼───────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            LISNAI GATEWAY (Hono + Bun)                               │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                           CHANNEL MANAGER                                      │  │
│  │                                                                                │  │
│  │   • Maintains connections to WhatsApp, Discord, Telegram, Instagram, etc.     │  │
│  │   • Routes chat.send { channel: "whatsapp", to: "+1234..." } to platforms     │  │
│  │   • Receives messages from ALL channels → forwards to agent pipeline          │  │
│  │   • Multi-account support per channel                                          │  │
│  │   • Channel status & presence tracking                                         │  │
│  │                                                                                │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                          │                                           │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         WEBSOCKET CONTROL PLANE                                │  │
│  │                                                                                │  │
│  │   • Node registration & pairing (like Clawdbot node.pair.*)                   │  │
│  │   • Real-time transcript streaming                                             │  │
│  │   • Action result delivery                                                     │  │
│  │   • Session sync                                                               │  │
│  │   • Device capability discovery                                                │  │
│  │                                                                                │  │
│  │   Protocol: JSON-RPC 2.0 over WebSocket                                       │  │
│  │   Port: 18790 (different from Clawdbot to avoid conflicts)                    │  │
│  │                                                                                │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                          │                                           │
│  ┌───────────────────────────────────────┼───────────────────────────────────────┐  │
│  │                              REST API ROUTES                                   │  │
│  │                                                                                │  │
│  │   POST /api/v1/transcripts/process   - Process transcript                     │  │
│  │   POST /api/v1/transcripts/stream    - WebSocket upgrade for streaming        │  │
│  │   POST /api/v1/chat/send             - Send to any channel                    │  │
│  │   GET  /api/v1/channels/status       - Get channel connection status          │  │
│  │   GET  /api/v1/memories              - List memories                          │  │
│  │   POST /api/v1/memories/search       - Hybrid search (vector + keyword)       │  │
│  │   POST /api/v1/chat                  - Chat with memory context               │  │
│  │   GET  /api/v1/actions/pending       - Pending iOS actions                    │  │
│  │   POST /api/v1/node/invoke           - Invoke device capabilities             │  │
│  │                                                                                │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                          │                                           │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                      AGENT PIPELINE (Vercel AI SDK 6)                         │  │
│  │                                                                                │  │
│  │  ┌─────────────────┐                                                          │  │
│  │  │  Entity Agent   │ Extract people, dates, places, topics                    │  │
│  │  └────────┬────────┘                                                          │  │
│  │           │                                                                    │  │
│  │           ▼                                                                    │  │
│  │  ┌─────────────────────────────────────────────────────────┐                  │  │
│  │  │          PARALLEL EXECUTION (like Clawdbot)             │                  │  │
│  │  │                                                         │                  │  │
│  │  │   ┌─────────────────┐      ┌─────────────────┐         │                  │  │
│  │  │   │  Memory Agent   │      │  Router Agent   │         │                  │  │
│  │  │   │                 │      │                 │         │                  │  │
│  │  │   │ ALWAYS stores   │      │ Classifies:     │         │                  │  │
│  │  │   │ everything as   │      │ • action        │         │                  │  │
│  │  │   │ diary entries   │      │ • query         │         │                  │  │
│  │  │   │                 │      │ • passive       │         │                  │  │
│  │  │   └────────┬────────┘      └────────┬────────┘         │                  │  │
│  │  │            │                        │                   │                  │  │
│  │  └────────────┼────────────────────────┼───────────────────┘                  │  │
│  │               │                        │                                       │  │
│  │               │         ┌──────────────┘                                       │  │
│  │               │         │                                                      │  │
│  │               │         ▼                                                      │  │
│  │               │  ┌─────────────────┐                                          │  │
│  │               │  │  Action Agent   │ (if intent == action)                    │  │
│  │               │  │  ToolLoopAgent  │                                          │  │
│  │               │  │                 │                                          │  │
│  │               │  │  Skills:        │                                          │  │
│  │               │  │  • calendar     │                                          │  │
│  │               │  │  • reminder     │                                          │  │
│  │               │  │  • message      │                                          │  │
│  │               │  │  • email        │                                          │  │
│  │               │  │  • note         │                                          │  │
│  │               │  │  • navigation   │                                          │  │
│  │               │  └────────┬────────┘                                          │  │
│  │               │           │                                                    │  │
│  │               ▼           ▼                                                    │  │
│  │  ┌─────────────────────────────────────────────────────────┐                  │  │
│  │  │                    Query Agent                          │                  │  │
│  │  │                                                         │                  │  │
│  │  │   Hybrid search: Qdrant vectors + Firestore keywords   │                  │  │
│  │  │   (Like Clawdbot's memory_search + memory_get tools)   │                  │  │
│  │  │                                                         │                  │  │
│  │  └─────────────────────────────────────────────────────────┘                  │  │
│  │                                                                                │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                              │
│                                                                                      │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                  │
│   │    FIREBASE      │  │     QDRANT       │  │    LITELLM       │                  │
│   │                  │  │                  │  │                  │                  │
│   │ • users/{uid}/   │  │ • memories       │  │ • gpt-4o         │                  │
│   │   memories/      │  │   collection     │  │ • gpt-4o-mini    │                  │
│   │   sessions/      │  │ • Cosine         │  │ • claude-sonnet  │                  │
│   │   pendingActions/│  │   similarity     │  │ • mistral-large  │                  │
│   │   contacts/      │  │ • 1536 dims      │  │ • embeddings     │                  │
│   │   profile/       │  │                  │  │                  │                  │
│   │                  │  │ Hybrid fallback: │  │ Fallback chain:  │                  │
│   │ Real-time sync   │  │ BM25 keyword     │  │ gpt-4o → claude  │                  │
│   │ with iOS         │  │ (like Clawdbot)  │  │ → mistral        │                  │
│   │                  │  │                  │  │                  │                  │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘                  │
│                                                                                      │
│   ┌──────────────────┐  ┌──────────────────┐                                        │
│   │    INNGEST       │  │    LANGFUSE      │                                        │
│   │                  │  │                  │                                        │
│   │ • Daily briefings│  │ • Trace all LLM  │                                        │
│   │ • Session summary│  │   calls          │                                        │
│   │ • Proactive      │  │ • Cost tracking  │                                        │
│   │   reminders      │  │ • Evaluations    │                                        │
│   │ • Cron jobs      │  │                  │                                        │
│   │   (like Clawdbot)│  │                  │                                        │
│   │                  │  │                  │                                        │
│   └──────────────────┘  └──────────────────┘                                        │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## New Components from Clawdbot

### 1. Multi-Channel Manager (Like Clawdbot)

The Gateway owns ALL messaging channel connections. iOS never touches these SDKs directly.

```typescript
// src/channels/manager.ts
import { z } from 'zod';

export const ChannelIdSchema = z.enum([
  'whatsapp',
  'discord',
  'telegram',
  'instagram',
  'signal',
  'imessage',
]);

export type ChannelId = z.infer<typeof ChannelIdSchema>;

export interface ChannelAccount {
  accountId: string;
  channelId: ChannelId;
  enabled: boolean;
  configured: boolean;
  linked: boolean;
  running: boolean;
  lastError?: string;
  displayName?: string;
}

export interface ChannelManager {
  // Lifecycle
  startChannel(channelId: ChannelId, accountId?: string): Promise<void>;
  stopChannel(channelId: ChannelId, accountId?: string): Promise<void>;

  // Status
  getChannelStatus(channelId: ChannelId): ChannelAccount[];
  getAllChannelStatus(): Record<ChannelId, ChannelAccount[]>;

  // Messaging
  sendMessage(params: {
    channelId: ChannelId;
    accountId?: string;
    to: string;
    message: string;
    attachments?: Attachment[];
  }): Promise<SendResult>;

  // Incoming messages handler
  onMessage(handler: (msg: IncomingMessage) => void): void;
}

// Example: WhatsApp via Baileys
// src/channels/whatsapp/provider.ts
import makeWASocket, { useMultiFileAuthState } from '@whiskeysockets/baileys';

export async function createWhatsAppProvider(config: WhatsAppConfig) {
  const { state, saveCreds } = await useMultiFileAuthState(config.authDir);

  const socket = makeWASocket({
    auth: state,
    printQRInTerminal: true, // For initial pairing
  });

  socket.ev.on('creds.update', saveCreds);

  socket.ev.on('messages.upsert', async ({ messages }) => {
    for (const msg of messages) {
      if (!msg.key.fromMe) {
        // Forward to agent pipeline
        await handleIncomingMessage({
          channelId: 'whatsapp',
          from: msg.key.remoteJid,
          text: msg.message?.conversation || '',
          timestamp: new Date(msg.messageTimestamp * 1000),
        });
      }
    }
  });

  return {
    send: async (to: string, message: string) => {
      await socket.sendMessage(to, { text: message });
    },
    socket,
  };
}

// Example: Discord via discord.js
// src/channels/discord/provider.ts
import { Client, GatewayIntentBits } from 'discord.js';

export async function createDiscordProvider(config: DiscordConfig) {
  const client = new Client({
    intents: [
      GatewayIntentBits.Guilds,
      GatewayIntentBits.GuildMessages,
      GatewayIntentBits.DirectMessages,
      GatewayIntentBits.MessageContent,
    ],
  });

  client.on('messageCreate', async (message) => {
    if (message.author.bot) return;

    // Forward to agent pipeline
    await handleIncomingMessage({
      channelId: 'discord',
      from: message.author.id,
      text: message.content,
      guildId: message.guildId,
      channelId: message.channelId,
      timestamp: message.createdAt,
    });
  });

  await client.login(config.botToken);

  return {
    send: async (channelId: string, message: string) => {
      const channel = await client.channels.fetch(channelId);
      if (channel?.isTextBased()) {
        await channel.send(message);
      }
    },
    client,
  };
}

// Example: Telegram via grammY
// src/channels/telegram/provider.ts
import { Bot } from 'grammy';

export async function createTelegramProvider(config: TelegramConfig) {
  const bot = new Bot(config.botToken);

  bot.on('message:text', async (ctx) => {
    // Forward to agent pipeline
    await handleIncomingMessage({
      channelId: 'telegram',
      from: String(ctx.from?.id),
      text: ctx.message.text,
      chatId: String(ctx.chat.id),
      timestamp: new Date(ctx.message.date * 1000),
    });
  });

  bot.start();

  return {
    send: async (chatId: string, message: string) => {
      await bot.api.sendMessage(chatId, message);
    },
    bot,
  };
}
```

### 2. iOS Chat Transport (Routes Through Gateway)

iOS sends ALL messages through the Gateway, specifying the target channel:

```typescript
// iOS sends this via WebSocket:
{
  "type": "req",
  "id": "unique-id",
  "method": "chat.send",
  "params": {
    "channel": "whatsapp",           // Target platform
    "accountId": "default",          // Which account (if multi-account)
    "to": "+1234567890",             // Recipient
    "message": "Hello from LisnAI!",
    "sessionKey": "main"
  }
}

// Gateway receives, routes to WhatsApp provider, sends response:
{
  "type": "res",
  "id": "unique-id",
  "ok": true,
  "payload": {
    "status": "sent",
    "messageId": "wa-msg-123",
    "timestamp": "2026-01-26T01:00:00Z"
  }
}
```

### 3. Channel Status & Presence

iOS receives real-time channel status via Gateway events:

```typescript
// Gateway sends presence event to all connected nodes:
{
  "type": "event",
  "event": "channels.presence",
  "payload": {
    "channels": {
      "whatsapp": {
        "accountId": "default",
        "running": true,
        "linked": true,
        "displayName": "+1 234 567 8900"
      },
      "discord": {
        "accountId": "mybot",
        "running": true,
        "configured": true,
        "displayName": "LisnAI Bot#1234"
      },
      "telegram": {
        "accountId": "default",
        "running": false,
        "lastError": "Bot token invalid"
      }
    }
  }
}
```

### 4. Gateway WebSocket Protocol

Inspired by Clawdbot's Gateway at `ws://127.0.0.1:18789`, we add real-time bidirectional communication:

```typescript
// src/gateway/protocol.ts
import { z } from 'zod';

// Gateway message types (JSON-RPC 2.0 style)
export const GatewayMessageSchema = z.discriminatedUnion('type', [
  // Node registration
  z.object({
    type: z.literal('node.register'),
    id: z.string(),
    params: z.object({
      deviceId: z.string(),
      displayName: z.string(),
      platform: z.enum(['ios', 'android', 'macos', 'web']),
      capabilities: z.array(z.enum(['camera', 'screen', 'location', 'voiceWake', 'canvas'])),
      commands: z.array(z.string()),
    }),
  }),

  // Transcript streaming
  z.object({
    type: z.literal('transcript.stream'),
    id: z.string(),
    params: z.object({
      sessionId: z.string(),
      text: z.string(),
      isFinal: z.boolean(),
      timestamp: z.string().datetime(),
    }),
  }),

  // Action result from iOS
  z.object({
    type: z.literal('action.result'),
    id: z.string(),
    params: z.object({
      actionId: z.string(),
      status: z.enum(['completed', 'failed', 'cancelled']),
      result: z.record(z.unknown()).optional(),
      error: z.string().optional(),
    }),
  }),

  // Node invoke (backend → iOS)
  z.object({
    type: z.literal('node.invoke'),
    id: z.string(),
    params: z.object({
      command: z.enum(['notify', 'camera_snap', 'location_get', 'screen_record', 'system.run']),
      args: z.record(z.unknown()),
    }),
  }),

  // Ping/pong for keepalive
  z.object({
    type: z.literal('ping'),
    id: z.string(),
  }),
  z.object({
    type: z.literal('pong'),
    id: z.string(),
  }),
]);

export type GatewayMessage = z.infer<typeof GatewayMessageSchema>;
```

### 2. iOS Node Capabilities

Inspired by Clawdbot's node capabilities pattern:

```typescript
// src/gateway/node-capabilities.ts
import { z } from 'zod';

export const NodeCapabilitySchema = z.enum([
  'canvas',      // Visual workspace
  'camera',      // Camera snap/clip
  'screen',      // Screen recording
  'location',    // GPS location
  'voiceWake',   // Wake word detection
  'notify',      // Push notifications
]);

export const NodeCommandSchema = z.enum([
  // Notifications
  'system.notify',

  // Camera
  'camera.list',
  'camera.snap',
  'camera.clip',

  // Screen
  'screen.record',

  // Location
  'location.get',

  // Canvas (future)
  'canvas.present',
  'canvas.hide',

  // System
  'system.run',           // Execute Shortcuts
  'system.openUrl',       // Deep links
]);

export interface ConnectedNode {
  deviceId: string;
  displayName: string;
  platform: 'ios' | 'android' | 'macos' | 'web';
  capabilities: NodeCapability[];
  commands: NodeCommand[];
  connectedAt: Date;
  lastSeen: Date;
  userId: string;
}

// Node registry
export class NodeRegistry {
  private nodes = new Map<string, ConnectedNode>();

  register(node: ConnectedNode): void {
    this.nodes.set(node.deviceId, node);
  }

  unregister(deviceId: string): void {
    this.nodes.delete(deviceId);
  }

  getNodesByUser(userId: string): ConnectedNode[] {
    return Array.from(this.nodes.values()).filter(n => n.userId === userId);
  }

  getNodeWithCapability(userId: string, capability: NodeCapability): ConnectedNode | null {
    return this.getNodesByUser(userId).find(n => n.capabilities.includes(capability)) || null;
  }
}
```

### 3. Skills System

Inspired by Clawdbot's 54+ skills with SKILL.md documentation:

```typescript
// src/skills/types.ts
import { z } from 'zod';

export interface Skill {
  name: string;
  description: string;
  version: string;
  os: ('ios' | 'android' | 'macos' | 'web')[];
  requires?: {
    capabilities?: string[];
    apps?: string[];
  };
  tools: SkillTool[];
}

export interface SkillTool {
  name: string;
  description: string;
  parameters: z.ZodSchema;
  execute: (params: unknown, deps: AgentDeps) => Promise<ToolResult>;
}

// Example: Apple Reminders skill (like Clawdbot's apple-reminders)
export const appleRemindersSkill: Skill = {
  name: 'apple-reminders',
  description: 'Create and manage Apple Reminders via iOS',
  version: '1.0.0',
  os: ['ios', 'macos'],
  requires: {
    capabilities: ['system.run'],
  },
  tools: [
    {
      name: 'create_reminder',
      description: 'Create a new reminder in Apple Reminders',
      parameters: z.object({
        title: z.string(),
        dueDate: z.string().datetime().optional(),
        list: z.string().optional(),
        priority: z.enum(['low', 'medium', 'high']).optional(),
        notes: z.string().optional(),
      }),
      execute: async (params, deps) => {
        // Store as pending action for iOS to execute
        const actionRef = await deps.firestore
          .collection(`users/${deps.userId}/pendingActions`)
          .add({
            type: 'reminder',
            skill: 'apple-reminders',
            params,
            status: 'pending',
            createdAt: new Date(),
          });

        return {
          success: true,
          message: `Reminder "${params.title}" ready to create`,
          actionId: actionRef.id,
          iosDeepLink: `lisnai://action/reminder/${actionRef.id}`,
        };
      },
    },
  ],
};
```

### 4. Hybrid Memory Search

Inspired by Clawdbot's hybrid vector + keyword search:

```typescript
// src/memory/hybrid-search.ts
import { embed } from 'ai';
import { openai } from '@ai-sdk/openai';
import type { AgentDeps } from '../agents/deps';

interface SearchResult {
  memoryId: string;
  content: string;
  score: number;
  source: 'vector' | 'keyword';
  timestamp: Date;
  topics: string[];
}

export async function hybridSearch(
  query: string,
  deps: AgentDeps,
  options: {
    limit?: number;
    minScore?: number;
    dateFrom?: Date;
    dateTo?: Date;
  } = {}
): Promise<SearchResult[]> {
  const { limit = 10, minScore = 0.5 } = options;

  // Parallel: vector search + keyword search
  const [vectorResults, keywordResults] = await Promise.all([
    // 1. Vector search via Qdrant
    (async () => {
      const { embedding } = await embed({
        model: openai.embedding('text-embedding-3-small'),
        value: query,
      });

      const qdrantResults = await deps.qdrant.search('memories', {
        vector: embedding,
        filter: {
          must: [
            { key: 'userId', match: { value: deps.userId } },
          ],
        },
        limit,
        score_threshold: minScore,
      });

      return qdrantResults.map(r => ({
        memoryId: r.id as string,
        score: r.score,
        source: 'vector' as const,
      }));
    })(),

    // 2. Keyword search via Firestore
    (async () => {
      // Extract keywords from query
      const keywords = query.toLowerCase().split(/\s+/).filter(w => w.length > 3);

      if (keywords.length === 0) return [];

      // Search in topics and rawTranscript
      const snapshot = await deps.firestore
        .collection(`users/${deps.userId}/memories`)
        .where('topics', 'array-contains-any', keywords.slice(0, 10))
        .orderBy('timestamp', 'desc')
        .limit(limit)
        .get();

      return snapshot.docs.map(doc => ({
        memoryId: doc.id,
        score: 0.7, // Keyword matches get base score
        source: 'keyword' as const,
      }));
    })(),
  ]);

  // Merge and deduplicate results
  const mergedMap = new Map<string, SearchResult>();

  for (const r of vectorResults) {
    mergedMap.set(r.memoryId, { ...r, content: '', timestamp: new Date(), topics: [] });
  }

  for (const r of keywordResults) {
    const existing = mergedMap.get(r.memoryId);
    if (existing) {
      // Boost score if found in both
      existing.score = Math.min(1, existing.score + 0.2);
    } else {
      mergedMap.set(r.memoryId, { ...r, content: '', timestamp: new Date(), topics: [] });
    }
  }

  // Fetch full memory documents
  const memoryIds = Array.from(mergedMap.keys()).slice(0, limit);
  const memoryDocs = await Promise.all(
    memoryIds.map(id => deps.firestore.doc(`users/${deps.userId}/memories/${id}`).get())
  );

  const results: SearchResult[] = [];
  for (const doc of memoryDocs) {
    if (!doc.exists) continue;
    const data = doc.data()!;
    const scoreData = mergedMap.get(doc.id)!;

    results.push({
      memoryId: doc.id,
      content: data.rawTranscript,
      score: scoreData.score,
      source: scoreData.source,
      timestamp: data.timestamp?.toDate() || new Date(),
      topics: data.topics || [],
    });
  }

  // Sort by score descending
  return results.sort((a, b) => b.score - a.score);
}
```

### 5. Proactive Briefings (Cron Jobs)

Inspired by Clawdbot's cron tool:

```typescript
// src/jobs/daily-briefing.ts
import { inngest } from '../inngest';
import { createDeps } from '../agents/deps';
import { generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

// Daily morning briefing job
export const dailyBriefing = inngest.createFunction(
  { id: 'daily-briefing' },
  { cron: '0 7 * * *' }, // 7 AM daily
  async ({ step }) => {
    // Get all users who have enabled briefings
    const usersSnapshot = await step.run('get-users', async () => {
      const { firestore } = await import('../services/firebase');
      return firestore
        .collectionGroup('profile')
        .where('settings.dailyBriefing.enabled', '==', true)
        .get();
    });

    for (const userDoc of usersSnapshot.docs) {
      const userId = userDoc.ref.parent.parent?.id;
      if (!userId) continue;

      await step.run(`briefing-${userId}`, async () => {
        // Get yesterday's memories
        const yesterday = new Date();
        yesterday.setDate(yesterday.getDate() - 1);

        const memories = await firestore
          .collection(`users/${userId}/memories`)
          .where('timestamp', '>=', yesterday)
          .orderBy('timestamp', 'asc')
          .get();

        if (memories.empty) return;

        // Generate briefing
        const { object: briefing } = await generateObject({
          model: openai('gpt-4o-mini'),
          schema: z.object({
            summary: z.string(),
            keyMoments: z.array(z.string()).max(5),
            pendingActions: z.array(z.string()),
            reminders: z.array(z.string()),
          }),
          prompt: `Generate a morning briefing from yesterday's memories:
${memories.docs.map(d => d.data().rawTranscript).join('\n')}`,
        });

        // Send notification to iOS
        await sendBriefingNotification(userId, briefing);
      });
    }
  }
);
```

---

## Updated Project Structure

```
backend/
├── package.json
├── tsconfig.json
├── biome.json
├── docker-compose.yml
├── litellm-config.yaml
│
├── src/
│   ├── index.ts                    # Main entry + REST routes
│   ├── config.ts                   # Environment configuration
│   │
│   ├── gateway/                    # NEW: WebSocket control plane
│   │   ├── server.ts               # WebSocket server
│   │   ├── protocol.ts             # JSON-RPC message types
│   │   ├── node-registry.ts        # Connected nodes tracking
│   │   ├── handlers/
│   │   │   ├── node.ts             # Node registration/invoke
│   │   │   ├── transcript.ts       # Transcript streaming
│   │   │   └── action.ts           # Action results
│   │   └── index.ts
│   │
│   ├── schemas/                    # Zod schemas
│   │   ├── entities.ts
│   │   ├── router.ts
│   │   ├── memory.ts
│   │   ├── actions.ts
│   │   ├── gateway.ts              # NEW: Gateway messages
│   │   └── index.ts
│   │
│   ├── agents/                     # Vercel AI SDK agents
│   │   ├── deps.ts
│   │   ├── entity-agent.ts
│   │   ├── router-agent.ts
│   │   ├── memory-agent.ts
│   │   ├── action-agent.ts
│   │   ├── query-agent.ts
│   │   └── index.ts
│   │
│   ├── skills/                     # NEW: Extensible skill system
│   │   ├── types.ts
│   │   ├── registry.ts
│   │   ├── apple-reminders.ts
│   │   ├── apple-calendar.ts
│   │   ├── apple-notes.ts
│   │   ├── messaging.ts
│   │   └── index.ts
│   │
│   ├── memory/                     # NEW: Enhanced memory system
│   │   ├── hybrid-search.ts        # Vector + keyword search
│   │   ├── session-manager.ts      # Session lifecycle
│   │   └── index.ts
│   │
│   ├── jobs/                       # NEW: Background jobs (Inngest)
│   │   ├── daily-briefing.ts
│   │   ├── session-summary.ts
│   │   ├── proactive-reminders.ts
│   │   └── index.ts
│   │
│   ├── pipeline/
│   │   ├── main.ts
│   │   └── streaming.ts            # NEW: Streaming pipeline
│   │
│   ├── services/
│   │   ├── firebase.ts
│   │   ├── qdrant.ts
│   │   ├── langfuse.ts
│   │   ├── litellm.ts
│   │   └── index.ts
│   │
│   ├── middleware/
│   │   ├── auth.ts
│   │   └── index.ts
│   │
│   └── utils/
│       ├── temporal.ts
│       └── text.ts
│
└── tests/
```

---

## Updated Implementation Phases

### Phase 1: Foundation ✅ (COMPLETED)
- [x] Set up Hono + Bun project structure
- [x] Create Zod schemas
- [x] Implement Firebase service client
- [x] Set up Qdrant vector DB client
- [x] Set up Langfuse observability
- [x] Create basic Hono API with health check and auth middleware
- [x] Create Docker compose for LiteLLM + Qdrant + Langfuse

### Phase 2: Gateway + Agents ✅ (COMPLETED)
- [x] Implement WebSocket gateway server (`src/gateway/server.ts`)
- [x] Create gateway protocol - JSON-RPC messages (`src/gateway/protocol.ts`)
- [x] Implement node registration and capability discovery (`src/gateway/registry.ts`)
- [x] Build Entity Agent - Vercel AI SDK + Zod (`src/agents/entity-agent.ts`)
- [x] Build Memory Agent - always stores everything (`src/agents/memory-agent.ts`)
- [x] Build Router Agent - intent classification (`src/agents/router-agent.ts`)
- [x] Implement main processing pipeline (`src/pipeline/process.ts`)
- [x] Add POST /api/transcripts/process endpoint (`src/routes/transcripts.ts`)

**Ports:**
- REST API: `http://localhost:3000`
- Gateway WebSocket: `ws://localhost:18790`

### Phase 3: Skills + Actions ✅ (COMPLETED)
```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: SKILLS + ACTIONS                                       │
│                                                                 │
│  ✅ Create skill registry and types                              │
│  ✅ Implement apple-reminders skill                              │
│  ✅ Implement apple-calendar skill                               │
│  ✅ Implement apple-notes skill                                  │
│  ✅ Implement messaging skill                                    │
│  ✅ Build Action Agent with ToolLoopAgent                       │
│  ✅ Add node.invoke for iOS capabilities                         │
│  ✅ Test action execution flow                                   │
│                                                                 │
│  DELIVERABLE: Voice commands → iOS actions working               │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 4: Hybrid Search + Chat ✅ (COMPLETED)
```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4: HYBRID SEARCH + CHAT                                   │
│                                                                 │
│  ✅ Implement hybrid search (vector + keyword)                   │
│  ✅ Build Query Agent with memory context                       │
│  ✅ Implement semantic search endpoint                           │
│  ✅ Build Chat Agent                                             │
│  □ Add streaming chat responses (future enhancement)             │
│                                                                 │
│  DELIVERABLE: "What did I say about X?" working                  │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 5: Proactive + Background Jobs ✅ (COMPLETED)
```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 5: PROACTIVE + BACKGROUND JOBS                            │
│                                                                 │
│  ✅ Set up Inngest for background jobs                           │
│  ✅ Implement daily briefing job (7 AM default)                  │
│  ✅ Implement session summary job (auto on idle)                 │
│  ✅ Implement proactive reminder detection                       │
│  ✅ Add cron scheduling (briefings, reminders, overdue)          │
│  ✅ Push notifications via Firebase Cloud Messaging              │
│  ✅ Commitment detection from transcripts                        │
│                                                                 │
│  DELIVERABLE: Proactive morning briefings working                │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 6: iOS Integration
```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 6: iOS INTEGRATION                                        │
│                                                                 │
│  □ Create iOS Gateway client (WebSocket)                         │
│  □ Implement node registration in iOS                            │
│  □ Add capability exposure (camera, location, etc.)              │
│  □ Implement pending action sync                                 │
│  □ Add deep linking for actions                                  │
│  □ Integrate with Apple Shortcuts                                │
│                                                                 │
│  DELIVERABLE: Full iOS ↔ Backend integration                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Differences from Original Plan

| Aspect | Original Plan | Updated Plan (from Clawdbot) |
|--------|--------------|------------------------------|
| **Communication** | REST only | REST + WebSocket gateway |
| **iOS Role** | API client | "Node" with capabilities |
| **Search** | Vector only | Hybrid vector + keyword |
| **Skills** | Hardcoded tools | Extensible skill plugins |
| **Proactive** | Not specified | Daily briefings, cron jobs |
| **Session Mgmt** | Basic | Per-device with routing |
| **Device Control** | Deep links only | node.invoke + deep links |

---

## Sources

- [Clawdbot GitHub](https://github.com/clawdbot/clawdbot) - 8,000+ stars
- [Clawdbot Documentation](https://docs.clawd.bot)
- [MacStories: Future of Personal AI](https://www.macstories.net/stories/clawdbot-showed-me-what-the-future-of-personal-ai-assistants-looks-like/)
- [VelvetShark: Clawdbot vs Siri](https://velvetshark.com/clawdbot-the-self-hosted-ai-that-siri-should-have-been)
- [WebProNews: Local Lobster](https://www.webpronews.com/clawdbots-local-lobster-the-open-source-agent-redefining-personal-ai/)
