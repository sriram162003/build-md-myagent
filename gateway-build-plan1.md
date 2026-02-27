# AURA Gateway

Build the complete AURA Gateway — a self-hosted, always-on AI agent brain that runs on any home PC or Mac. Seven messaging channels (Telegram, WhatsApp, Signal, Slack, Discord, Google Chat, Microsoft Teams), fully switchable LLM (Claude, OpenAI, Ollama, Gemini), 11 skill integrations, ElevenLabs voice, multi-agent routing, proactive outreach, self-improving skills, and a live Canvas workspace — all over the original AURA Node Protocol (ANP).

## Problem Statement

Personal AI agents are either cloud-locked, model-specific, or too fragile to run 24/7 at home. AURA Gateway solves this with a single always-on daemon that:

- **Owns the protocol** — every client (body disc, phone, browser, messaging app) connects over ANP, one unified interface
- **Owns the models** — switch Claude/OpenAI/Ollama/Gemini by editing one line in config.yaml, zero code changes
- **Owns the data** — memory, skills, conversations are plain files on your disk, inspectable and git-backable
- **Acts without being asked** — proactive heartbeat, scheduled outreach, reminders, agent-initiated messages across any channel
- **Grows itself** — self-improving skills engine lets the agent write and install new capabilities from a conversation

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 20 LTS |
| Language | TypeScript 5 strict mode |
| ANP server | `ws` npm |
| LLM — Claude | `@anthropic-ai/sdk` |
| LLM — OpenAI | `openai` npm |
| LLM — Ollama | Ollama REST (`fetch`) |
| LLM — Gemini | `@google/generative-ai` |
| Channel — Telegram | `telegraf` |
| Channel — WhatsApp | `whatsapp-web.js` |
| Channel — Signal | `signal-cli` (Java, spawned child process) |
| Channel — Slack | `@slack/bolt` |
| Channel — Discord | `discord.js` |
| Channel — Google Chat | `googleapis` REST |
| Channel — MS Teams | `botbuilder` (Microsoft Bot Framework) |
| Channel — WebChat | built-in WebSocket (browser client) |
| Voice TTS | ElevenLabs REST API |
| Voice STT | Whisper API (`openai` sdk) |
| Memory DB | `better-sqlite3` FTS5 |
| Skill watcher | `chokidar` |
| Scheduler | `node-cron` |
| Config | `js-yaml` |
| mDNS | `mdns` npm |
| Canvas | WebSocket + JSON (port 3001) |
| Self-write | LLM-generated TS + `tsc` sandbox validation |
| Dev | `tsx` watch mode, `vitest` |

---

## Project Structure

```
gateway/
├── src/
│   ├── server.ts                      # entry: starts all subsystems
│   ├── anp/
│   │   ├── server.ts                  # WebSocket server port 8765
│   │   ├── auth.ts                    # token validation
│   │   ├── router.ts                  # route ANP messages to handlers
│   │   └── types.ts                   # all ANP message type definitions
│   ├── llm/
│   │   ├── router.ts                  # model selection per tier + agent
│   │   ├── context.ts                 # system prompt + memory assembly
│   │   ├── types.ts                   # LLMAdapter, LLMResponse interfaces
│   │   └── adapters/
│   │       ├── claude.ts
│   │       ├── openai.ts
│   │       ├── ollama.ts
│   │       └── gemini.ts
│   ├── agents/
│   │   ├── registry.ts                # load + hot-reload agents.yaml
│   │   ├── resolver.ts                # pick correct agent per node_id
│   │   └── types.ts                   # AgentConfig, AgentProfile
│   ├── skills/
│   │   ├── engine.ts                  # load YAML, build tool defs, hot-reload
│   │   ├── executor.ts                # run TS tool calls, return results
│   │   ├── registry.ts                # in-memory loaded skill map
│   │   ├── self_write.ts              # LLM writes + installs new skills
│   │   └── types.ts                   # SkillDefinition, SkillContext
│   ├── memory/
│   │   ├── manager.ts                 # facade: all memory tiers
│   │   ├── short_term.ts              # RAM ring buffer, per session
│   │   ├── episodic.ts                # daily .md files per agent namespace
│   │   └── semantic.ts                # SQLite FTS5 search
│   ├── scheduler/
│   │   ├── engine.ts                  # node-cron + SQLite reminder queue
│   │   ├── heartbeat.ts               # reads HEARTBEAT.md, runs LLM, acts
│   │   └── proactive.ts               # agent-initiated outreach tools
│   ├── channels/
│   │   ├── interface.ts               # ChannelAdapter interface
│   │   ├── manager.ts                 # start/stop/route all adapters
│   │   ├── telegram.ts
│   │   ├── whatsapp.ts
│   │   ├── signal.ts
│   │   ├── slack.ts
│   │   ├── discord.ts
│   │   ├── google_chat.ts
│   │   ├── teams.ts
│   │   └── webchat.ts
│   ├── canvas/
│   │   ├── server.ts                  # Canvas WebSocket server port 3001
│   │   ├── renderer.ts                # Canvas state + persistence
│   │   └── types.ts                   # CanvasBlock, CanvasEvent
│   ├── voice/
│   │   ├── elevenlabs.ts              # TTS: text to PCM stream
│   │   └── whisper.ts                 # STT: audio to text
│   └── api/
│       └── rest.ts                    # REST for Web UI, port 3002
├── ~/.aura/
│   ├── config.yaml
│   ├── nodes.yaml
│   ├── agents.yaml
│   ├── HEARTBEAT.md
│   ├── skills/
│   │   ├── web_search.yaml + .ts
│   │   ├── notion.yaml + .ts
│   │   ├── spotify.yaml + .ts
│   │   ├── apple_notes.yaml + .ts
│   │   ├── twitter.yaml + .ts
│   │   ├── webhooks.yaml + .ts
│   │   ├── whoop.yaml + .ts
│   │   ├── home_assistant.yaml + .ts
│   │   ├── google_calendar.yaml + .ts
│   │   ├── reminders.yaml + .ts
│   │   └── body_control.yaml + .ts
│   └── memory/
│       ├── aura.db
│       ├── canvas.json
│       ├── preferences.yaml
│       ├── routine.json
│       ├── personal/YYYY-MM-DD.md
│       ├── dev/YYYY-MM-DD.md
│       └── social/YYYY-MM-DD.md
├── tests/
├── package.json
├── tsconfig.json
└── .env
```

---

## Agent Build Order

Build in strict dependency order. Each step has a validation gate — do not proceed until the gate passes.

### Step 1: Types + Config Loader
Define all TypeScript types before writing any logic. Every subsequent step imports from these.

**Deliverables:**
- `src/anp/types.ts` — all ANP message shapes
- `src/llm/types.ts` — LLMAdapter, LLMParams, LLMResponse, ToolDefinition, ToolCall
- `src/agents/types.ts` — AgentConfig, AgentProfile
- `src/skills/types.ts` — SkillDefinition, SkillContext
- `src/canvas/types.ts` — CanvasBlock, CanvasEvent, CanvasState
- `src/config/loader.ts` — reads + validates config.yaml, nodes.yaml, agents.yaml
- **Gate:** `npx tsc --noEmit` passes with zero errors

### Step 2: ANP Server
**Deliverables:**
- WebSocket server on `127.0.0.1:8765` path `/anp`
- HELLO → WELCOME / REJECT handshake with 5s timeout
- Connected node registry `Map<node_id, NodeSession>`
- PING → PONG keepalive
- Internal EventBus: `anp.on('utterance', handler)`
- `sendCommand(node_id, cmd, payload)` callable by all subsystems
- mDNS: advertise `_aura-gw._tcp.local` port 8765
- chokidar watch on `nodes.yaml` → close revoked node WebSockets within 5s
- **Gate:** valid WELCOME, invalid REJECT 401, duplicate REJECT 409, 5s timeout, PING/PONG

### Step 3: LLM Router — All Four Adapters
**Deliverables:**
- Claude adapter — `@anthropic-ai/sdk`, tool_use, vision, streaming
- OpenAI adapter — same tool_use shape, vision on gpt-4o
- Ollama adapter — REST `http://localhost:11434`, tool_use via format param
- Gemini adapter — `@google/generative-ai`, functionDeclarations for tools
- `LLMRouter.complete(tier, params, agent_id?)` — picks adapter from config
- **Gate:** all four adapters return valid responses

### Step 4: Multi-Agent Registry
**Deliverables:**
- `agents.yaml` loader + validator
- `AgentRegistry.resolve(node_id)` — exact match → prefix match → default fallback
- Each agent: id, name, persona, channels[], skills[], llm_tier, voice_id, memory_ns
- chokidar hot-reload on `agents.yaml`
- **Gate:** resolver picks correct agent for body, Telegram, Slack, Discord, unknown

### Step 5: Skills Engine + All 11 Skills
**Deliverables:**
- YAML loader, TS dynamic importer, chokidar hot-reload
- Tool definition builder → tools[] array for LLM
- Tool call executor with 5-iteration loop guard
- All 11 skills (YAML + TS): web_search, notion, spotify, apple_notes, twitter, webhooks, whoop, home_assistant, google_calendar, reminders, body_control
- **Gate:** all 11 load; web_search and reminders execute without API keys

### Step 6: Self-Improving Skills
**Deliverables:**
- `skills/self_write.ts` — `create_skill(name, description, tools_needed, notes)` tool
- Sonnet generates YAML + TS content
- `tsc --noEmit --strict` validates before save
- chokidar auto-loads on save — no manual step
- Self-written skills tagged `source: self_written`
- shell and self_write tools absent from self-written skill contexts
- **Gate:** ask agent to create a skill, confirm it installs and works in same session

### Step 7: Memory Manager + Context Builder
**Deliverables:**
- Short-term: per-session RAM ring buffer, 20-turn limit, auto-compress oldest 10 via Haiku
- Episodic: daily .md files at `~/.aura/memory/<agent_ns>/YYYY-MM-DD.md`
- Semantic: SQLite FTS5 search scoped per agent namespace
- `ContextBuilder.build(event, agent)` → {system, messages, tools} ready for LLM
- Token budget: > 6000 tokens → compress via Haiku before main call
- **Gate:** context has correct persona, agent-scoped memory, time context, tool defs; dev cannot see personal memory

### Step 8: Core Event Loop
**Deliverables:**
- Wire: ANP utterance → AgentResolver → ContextBuilder → LLMRouter → SkillExecutor → response
- Tool call loop: execute → inject result → re-call LLM → repeat up to 5x
- Vision path: image_b64 present → force vision tier (Sonnet / GPT-4o)
- Hardware node path → reply via sendCommand('speak')
- Channel node path → reply via ChannelAdapter.send()
- MemoryManager.addTurn() after every completed exchange
- **Gate:** full utterance-in → speak-command-out round trip with tool use

### Step 9: Channel Adapters — All Seven + WebChat
**Deliverables:**
- ChannelAdapter interface + ChannelManager
- All 7 adapters: Telegram, WhatsApp, Signal, Slack, Discord, Google Chat, Teams
- WebChat (browser WebSocket)
- Auto-register channel nodes on first message
- **Gate:** message in on each channel → response arrives back on same channel

### Step 10: Scheduler + Proactive Outreach
**Deliverables:**
- node-cron scheduler + SQLite reminder queue
- Heartbeat: reads HEARTBEAT.md, LLM + tools, acts or responds HEARTBEAT_OK
- `send_message(node_id, text)` and `send_to_agent_channels(agent_id, text)` tools in heartbeat context
- Max 10 send_message calls per heartbeat run
- Nightly summary at 23:30
- **Gate:** 1-minute reminder fires; heartbeat sends HEARTBEAT_OK when nothing to do

### Step 11: ElevenLabs Voice
**Deliverables:**
- `voice/elevenlabs.ts` — POST to ElevenLabs `/v1/text-to-speech/:voice_id/stream`
- Returns PCM 16kHz buffer, streamed back to body as audio_chunk ANP commands (1KB chunks)
- voice_id per agent from agents.yaml
- First audio byte target < 400ms
- **Gate:** body utterance → ElevenLabs audio → body speaker plays response

### Step 12: Live Canvas
**Deliverables:**
- WebSocket server on `127.0.0.1:3001` path `/canvas`
- Canvas state: ordered CanvasBlock[] — text, code, table, image, chart, embed
- On client connect: send full current state as {event:'state', blocks:[]}
- Agent tools: canvas_clear, canvas_append, canvas_update, canvas_delete
- Persist to `~/.aura/memory/canvas.json` on every change, restore on startup
- **Gate:** agent utterance triggers canvas update; browser WS client sees live blocks

### Step 13: REST API
**Deliverables:**
- HTTP on `127.0.0.1:3002` — all endpoints listed in REST contract below
- **Gate:** all curl checks return expected shapes

---

## Authoritative Contracts

### ANP Types (`src/anp/types.ts`)

```typescript
export type NodeCap =
  | 'audio_in' | 'audio_out' | 'camera' | 'display'
  | 'led' | 'imu' | 'haptic' | 'battery'
  | 'text_in' | 'text_out' | 'notifications' | 'ble_relay';

export type ANPHello = {
  anp:     '1.0';
  type:    'hello';
  node_id: string;
  token:   string;
  caps:    NodeCap[];
  meta:    Record<string, unknown>;
};

export type ANPWelcome = {
  type:                   'welcome';
  node_id:                string;
  session_id:             string;
  gateway_v:              string;
  heartbeat_interval_sec: number;
};

export type ANPReject = {
  type:   'reject';
  code:   401 | 409;
  reason: string;
};

export type RoutingHint = 'simple' | 'complex' | 'vision' | 'creative';

export type UtterancePayload = {
  text:          string;
  confidence?:   number;
  image_b64?:    string | null;
  routing_hint?: RoutingHint;
  context?: { battery?: number; activity?: string; time_of_day?: string };
};

export type ANPEvent = {
  type:       'event';
  event:      'utterance' | 'camera_frame' | 'sensor' | 'status';
  node_id:    string;
  session_id: string;
  ts:         number;
  payload:    UtterancePayload | Record<string, unknown>;
};

export type ANPCommand = {
  type:    'command';
  target:  string;
  cmd:     'speak' | 'led' | 'display' | 'alert' | 'ota_update' | 'audio_chunk';
  payload: SpeakPayload | LEDPayload | AlertPayload | AudioChunkPayload | Record<string, unknown>;
};

export type SpeakPayload      = { text: string; voice?: string; display?: string; led?: LEDColor };
export type LEDPayload        = { pattern: LEDPattern; color: LEDColor; duration_ms?: number };
export type AlertPayload      = { text: string; led?: LEDColor; haptic?: HapticType; priority?: Priority };
export type AudioChunkPayload = { chunk_b64: string; seq: number; final: boolean };

export type LEDPattern = 'solid' | 'pulse' | 'breathing' | 'spin' | 'off';
export type LEDColor   = 'green' | 'blue' | 'amber' | 'red' | 'white' | 'off';
export type HapticType = 'single' | 'double_pulse' | 'long' | 'none';
export type Priority   = 'high' | 'normal' | 'silent';
```

### LLM Adapter Interface (`src/llm/types.ts`)

```typescript
export interface LLMAdapter {
  complete(params: LLMParams): Promise<LLMResponse>;
  readonly provider: string;
  readonly model:    string;
}

export interface LLMParams {
  system:      string;
  messages:    LLMMessage[];
  tools?:      ToolDefinition[];
  max_tokens?: number;
}

export interface LLMMessage {
  role:          'user' | 'assistant' | 'tool';
  content:       string;
  tool_call_id?: string;
}

export interface LLMResponse {
  text:        string;
  tool_calls?: ToolCall[];
  usage:       { input_tokens: number; output_tokens: number };
  model:       string;
  provider:    string;
}

export interface ToolCall {
  id:   string;
  name: string;
  args: Record<string, unknown>;
}

export interface ToolDefinition {
  name:        string;
  description: string;
  parameters:  Record<string, unknown>;
}
```

### config.yaml (authoritative schema)

```yaml
agent:
  name: "AURA"
  persona: >
    You are AURA, a personal AI agent. Warm, direct, never verbose.
    For voice surfaces: respond in under 3 sentences.
    For text surfaces: you may be more detailed.

llm:
  default: claude-haiku-4-5

  routing:
    simple:   claude-haiku-4-5
    complex:  claude-sonnet-4-6
    vision:   claude-sonnet-4-6
    creative: claude-opus-4
    offline:  ollama/llama3.2:3b

  providers:
    claude:
      api_key:  ${ANTHROPIC_API_KEY}
      models:   [claude-haiku-4-5, claude-sonnet-4-6, claude-opus-4]
    openai:
      api_key:  ${OPENAI_API_KEY}
      models:   [gpt-4o, gpt-4o-mini]
    ollama:
      base_url: http://localhost:11434
      models:   [llama3.2:3b, llama3.2:1b, mistral:7b, phi3:mini]
    lm_studio:
      base_url: http://localhost:1234
      models:   [any]
    gemini:
      api_key:  ${GOOGLE_API_KEY}
      models:   [gemini-1.5-pro, gemini-1.5-flash]

channels:
  telegram:
    enabled: true
    token:   ${TELEGRAM_BOT_TOKEN}
  whatsapp:
    enabled: true
  signal:
    enabled:      true
    phone_number: ${SIGNAL_PHONE_NUMBER}
    signal_cli:   /usr/local/bin/signal-cli
  slack:
    enabled:        true
    bot_token:      ${SLACK_BOT_TOKEN}
    signing_secret: ${SLACK_SIGNING_SECRET}
    app_token:      ${SLACK_APP_TOKEN}
  discord:
    enabled: true
    token:   ${DISCORD_BOT_TOKEN}
  google_chat:
    enabled:     true
    credentials: ${GOOGLE_CHAT_CREDENTIALS_PATH}
  teams:
    enabled:    true
    app_id:     ${TEAMS_APP_ID}
    app_secret: ${TEAMS_APP_SECRET}
  webchat:
    enabled: true
    port:    3000

voice:
  tts:
    provider: elevenlabs
    api_key:  ${ELEVENLABS_API_KEY}
    voice_id: 21m00Tcm4TlvDq8ikWAM
  stt:
    provider: whisper_api
    api_key:  ${OPENAI_API_KEY}

canvas:
  enabled: true
  port:    3001

scheduler:
  heartbeat_interval_min: 30
  reminder_check_sec:     60
  nightly_summary_time:   "23:30"

security:
  bind_address: 127.0.0.1
  anp_port:     8765
  rest_port:    3002
```

### agents.yaml (authoritative schema)

```yaml
agents:
  - id:          personal
    name:        "AURA"
    description: "Primary personal assistant"
    persona: >
      You are AURA, a warm and direct personal AI.
      Keep voice responses under 3 sentences.
    channels:
      - telegram_personal
      - whatsapp_personal
      - aura_body_01
      - aura_mobile_01
    skills:
      - web_search
      - reminders
      - google_calendar
      - notion
      - spotify
      - home_assistant
      - apple_notes
      - whoop
      - body_control
    llm_tier:  simple
    voice_id:  21m00Tcm4TlvDq8ikWAM
    memory_ns: personal

  - id:          dev
    name:        "AURA Dev"
    description: "Developer assistant"
    persona: >
      You are AURA Dev, a technical assistant.
      Be precise. Prefer code over prose. Use markdown for all code.
    channels:
      - slack_dev_dm
      - discord_dm
    skills:
      - web_search
      - webhooks
      - notion
    llm_tier:  complex
    voice_id:  null
    memory_ns: dev

  - id:          social
    name:        "AURA Social"
    description: "Twitter/X and social content"
    persona: >
      You are AURA Social. Draft posts that are sharp and original.
      Never cringe. Max 280 chars for tweets unless thread requested.
    channels:
      - telegram_social
    skills:
      - twitter
      - web_search
      - notion
    llm_tier:  creative
    voice_id:  null
    memory_ns: social

  - id:          default
    name:        "AURA"
    description: "Fallback for unrecognised channels"
    persona:     "You are AURA. Be helpful and brief."
    channels:    [__default__]
    skills:      [web_search, reminders]
    llm_tier:    simple
    voice_id:    null
    memory_ns:   default
```

**Agent resolution rules (`src/agents/resolver.ts`):**
1. Exact `node_id` match in any agent's `channels[]`
2. Prefix match: `node_id` starts with `slack_` → agent with any `slack_*` channel
3. No match → `default` agent

### Channel Adapter Interface (`src/channels/interface.ts`)

```typescript
export interface ChannelAdapter {
  readonly channel_id: string;
  init(config: ChannelConfig): Promise<void>;
  onMessage(handler: (event: ANPEvent) => void): void;
  send(message: OutboundMessage): Promise<void>;
  isHealthy(): Promise<boolean>;
  destroy(): Promise<void>;
}

export interface OutboundMessage {
  node_id:   string;
  text:      string;
  markdown?: boolean;
}
```

**node_id naming convention (all adapters must follow this exactly):**

| Channel | Format | Example |
|---------|--------|---------|
| Telegram | `telegram_<chat_id>` | `telegram_123456789` |
| WhatsApp | `whatsapp_<phone>` | `whatsapp_6591234567` |
| Signal | `signal_<phone>` | `signal_6591234567` |
| Slack | `slack_<user_id>` | `slack_U0123ABCDEF` |
| Discord | `discord_<user_id>` | `discord_123456789012` |
| Google Chat | `gchat_<space_id>` | `gchat_AAAA1234` |
| Teams | `teams_<conversation_id>` | `teams_19:abc123` |
| WebChat | `browser_<session_uuid>` | `browser_uuid-v4` |

All channel nodes auto-register on first message — no nodes.yaml entry required.

### Skill YAML Schema

```yaml
name:         skill_name
version:      1.0.0
description:  What this skill does
executor:     skill_name.ts
enabled:      true
source:       human              # or 'self_written'
requires_env: [SOME_API_KEY]

tools:
  - name:        tool_function_name
    description: When the LLM should call this
    parameters:
      type: object
      properties:
        param:
          type:        string
          description: What this is
      required: [param]
```

### Skill Executor Signature

```typescript
export async function tool_function_name(
  args: { param: string },
  ctx:  SkillContext
): Promise<unknown> { /* ... */ }

export interface SkillContext {
  node_id:    string;
  session_id: string;
  agent_id:   string;
  anp:        { sendCommand: (node_id: string, cmd: string, payload: unknown) => void };
  memory:     { search: (q: string) => Promise<string[]> };
  channel:    { send: (node_id: string, text: string) => Promise<void> };
  canvas:     { append: (block: CanvasBlock) => void; clear: () => void };
}
```

---

## All 11 Skills

### 1. web_search
```
tools: [search]
requires_env: []  # DuckDuckGo free; SERPAPI_KEY optional

search(query: string, num?: number) → {title, snippet, url}[]
Primary: DuckDuckGo Instant Answer API (no key)
Fallback: SerpAPI if SERPAPI_KEY set
```

### 2. notion
```
tools: [notion_search, notion_get_page, notion_create_page, notion_append_block, notion_query_database]
requires_env: [NOTION_API_KEY]

notion_search(query) → {id, title, url}[]
notion_get_page(page_id) → Markdown string
notion_create_page(parent_id, title, content) → URL
notion_append_block(page_id, content) → void
notion_query_database(database_id, filter?) → rows[]
Uses: @notionhq/client
```

### 3. spotify
```
tools: [spotify_play, spotify_pause, spotify_next, spotify_search, spotify_queue, spotify_current, spotify_volume]
requires_env: [SPOTIFY_CLIENT_ID, SPOTIFY_CLIENT_SECRET]

spotify_play(uri?) → void
spotify_pause() → void
spotify_next() → void
spotify_search(query, type?) → results[]
spotify_current() → {track, artist, album, progress_ms, is_playing}
spotify_volume(percent) → void
Auth: OAuth2 PKCE, token cached at ~/.aura/tokens/spotify.json
```

### 4. apple_notes
```
tools: [notes_list, notes_read, notes_create, notes_append,
        apple_reminders_list, apple_reminders_create, apple_reminders_complete]
requires_env: []  # JXA/AppleScript — macOS only

notes_list(search?) → {id, title, modified}[]
notes_read(note_id) → string
notes_create(title, body, folder?) → void
notes_append(note_id, content) → void
apple_reminders_list(list?) → {id, title, due, completed}[]
apple_reminders_create(title, due_iso?, list?) → void
apple_reminders_complete(reminder_id) → void
Non-macOS: stubs returning 'Not available on this OS'
```

### 5. twitter
```
tools: [twitter_post, twitter_thread, twitter_reply, twitter_search, twitter_get_timeline]
requires_env: [TWITTER_API_KEY, TWITTER_API_SECRET, TWITTER_ACCESS_TOKEN, TWITTER_ACCESS_SECRET]

twitter_post(text) → tweet URL
twitter_thread(tweets: string[]) → first tweet URL
twitter_reply(tweet_id, text) → tweet URL
twitter_search(query, count?) → Tweet[]
twitter_get_timeline(count?) → Tweet[]
Uses: twitter-api-v2
Also posts to Bluesky if BLUESKY_IDENTIFIER + BLUESKY_PASSWORD set
```

### 6. webhooks
```
tools: [webhook_send, webhook_list_received, webhook_clear]
requires_env: []

webhook_send(url, payload, method?) → {status, body}
webhook_list_received(key, limit?) → payload[]
webhook_clear(key) → void

Inbound: POST /webhook/:key (port 3002) → stored in SQLite
```

### 7. whoop
```
tools: [whoop_recovery, whoop_sleep, whoop_strain]
requires_env: [WHOOP_CLIENT_ID, WHOOP_CLIENT_SECRET]

whoop_recovery(date?) → {score, hrv, resting_hr, sleep_quality}
whoop_sleep(date?) → {performance, duration_hours, disturbances}
whoop_strain(date?) → {score, avg_hr, max_hr, calories}
date: 'YYYY-MM-DD' or omit for today
Auth: OAuth2, token at ~/.aura/tokens/whoop.json
```

### 8. home_assistant
```
tools: [ha_get_state, ha_call_service, ha_list_entities]
requires_env: [HA_URL, HA_TOKEN]

ha_get_state(entity_id) → {state, attributes}
ha_call_service(domain, service, data) → void
  e.g. ('light', 'turn_off', {entity_id: 'light.living_room'})
ha_list_entities(domain?) → {entity_id, state, name}[]
HA_URL: http://homeassistant.local:8123
HA_TOKEN: Long-Lived Access Token from HA profile
```

### 9. google_calendar
```
tools: [calendar_list_events, calendar_create_event, calendar_update_event, calendar_delete_event]
requires_env: [GOOGLE_OAUTH_CREDENTIALS]

calendar_list_events(days?, calendar_id?) → Event[]
calendar_create_event(title, start_iso, end_iso, description?, location?) → URL
calendar_update_event(event_id, updates) → void
calendar_delete_event(event_id) → void
Auth: OAuth2, shared credentials with gmail
```

### 10. reminders
```
tools: [set_reminder, list_reminders, cancel_reminder]
requires_env: []  # SQLite only

set_reminder(text, fire_at_iso, target_node?) → void
  target_node defaults to sender's node_id
  Fires as ANP 'alert' command at fire_at

list_reminders() → {id, text, fire_at, target_node, fired}[]
cancel_reminder(id) → void
```

### 11. body_control
```
tools: [speak, set_led, send_alert, set_display]
requires_env: []  # uses ANP directly

speak(text, node_id?) → void   (defaults to aura_body_01)
set_led(pattern, color, node_id?) → void
send_alert(text, haptic?, node_id?) → void
set_display(line1, line2?, node_id?) → void
```

---

## Self-Improving Skills

Tool the LLM calls: `create_skill(skill_name, description, tools_needed, implementation_notes)`

```typescript
// src/skills/self_write.ts
async function create_skill(args, ctx) {
  // 1. Prompt Sonnet to generate YAML + TS
  const { yaml_content, ts_content } = await generateSkillFiles(args);

  // 2. Write to temp files
  fs.writeFileSync(`/tmp/${args.skill_name}.yaml`, yaml_content);
  fs.writeFileSync(`/tmp/${args.skill_name}.ts`, ts_content);

  // 3. Validate — must pass before saving
  execSync(`npx tsc --noEmit --strict /tmp/${args.skill_name}.ts`);
  // throws if invalid — skill is NOT installed

  // 4. Move to ~/.aura/skills/ — chokidar auto-loads, no restart
  fs.copyFileSync(`/tmp/${args.skill_name}.yaml`, `${SKILLS_DIR}/${args.skill_name}.yaml`);
  fs.copyFileSync(`/tmp/${args.skill_name}.ts`,   `${SKILLS_DIR}/${args.skill_name}.ts`);

  return { installed: true, skill_name: args.skill_name };
}
```

**Safety rules:**
- `tsc --noEmit --strict` must pass before any file is saved to skills dir
- Self-written skills tagged `source: self_written` in YAML
- `shell` and `self_write` tools never available inside self-written skill executors
- All files human-reviewable at `~/.aura/skills/`

---

## Proactive Outreach

Two tools available to agents **in heartbeat context only**:

```typescript
// send_message — agent initiates contact
async function send_message(args: { node_id: string; text: string }, ctx) {
  await ctx.channel.send(args.node_id, args.text);
}

// send_to_agent_channels — broadcast to all nodes of an agent
async function send_to_agent_channels(args: { agent_id: string; text: string }, ctx) {
  const agent = agentRegistry.get(args.agent_id);
  for (const node_id of agent.channels) {
    await ctx.channel.send(node_id, args.text);
  }
}
```

Rate limit: max 10 `send_message` calls per heartbeat run.

---

## HEARTBEAT.md Format

```markdown
## Every 30 minutes
- Check current time against routine.json schedule
- If calendar event starts in < 15 minutes: use body_control to send alert
- If body node last status > 10 minutes ago: log warning
- Check reminders table — fire any with fire_at <= now

## Morning (07:00–09:00 weekdays)
- Get weather and today's calendar
- Check whoop_recovery score
- Compose spoken briefing under 30 words
- Use body_control to speak briefing to aura_body_01
- Use send_message to send full briefing to Telegram personal channel

## Evening (21:00–21:30 weekdays)
- Summarise day from episodic memory
- Note open reminders
- Use send_message to send summary to Telegram only

## WHOOP alert
- If whoop_recovery < 33: send_message to telegram_personal with warning

## Nightly (23:30)
- Write today's episodic summary to ~/.aura/memory/<agent_ns>/YYYY-MM-DD.md
- Update memory_fts index
- Update routine.json with observed schedule changes

## Silent rule (CRITICAL)
- If none of the above apply: respond ONLY "HEARTBEAT_OK"
- No message is always better than a pointless message
```

---

## Live Canvas Types

```typescript
export type CanvasBlock =
  | { id: string; type: 'text';  content: string }
  | { id: string; type: 'code';  language: string; content: string }
  | { id: string; type: 'table'; headers: string[]; rows: string[][] }
  | { id: string; type: 'image'; url: string; alt: string }
  | { id: string; type: 'chart'; chart_type: 'bar'|'line'|'pie'; data: ChartData }
  | { id: string; type: 'embed'; url: string; title: string };

export type CanvasEvent =
  | { event: 'state';  blocks: CanvasBlock[] }   // full state on connect
  | { event: 'append'; block:  CanvasBlock }
  | { event: 'update'; id: string; block: CanvasBlock }
  | { event: 'delete'; id: string }
  | { event: 'clear' };
```

Agent tools: `canvas_clear()`, `canvas_append(type, fields)`, `canvas_update(id, fields)`, `canvas_delete(id)`

Canvas WS: `ws://127.0.0.1:3001/canvas`
Canvas persists to `~/.aura/memory/canvas.json`, restored on startup.

---

## ElevenLabs Voice Contract

```typescript
export async function textToSpeech(
  text:     string,
  voice_id: string,
  format:   'pcm_16000' | 'pcm_22050' = 'pcm_16000'
): Promise<Buffer> {
  const res = await fetch(
    `https://api.elevenlabs.io/v1/text-to-speech/${voice_id}/stream`,
    {
      method: 'POST',
      headers: {
        'xi-api-key':   process.env.ELEVENLABS_API_KEY!,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        text,
        model_id:       'eleven_turbo_v2_5',
        output_format:  format,
        voice_settings: { stability: 0.5, similarity_boost: 0.75 }
      })
    }
  );
  return Buffer.from(await res.arrayBuffer());
}

// Streamed to body as audio_chunk ANP commands (1KB each):
// { type:'command', target: node_id, cmd:'audio_chunk',
//   payload: { chunk_b64: chunk.toString('base64'), seq: N, final: false } }
// Last chunk: final: true
```

Target: first audio byte from ElevenLabs < 400ms. Never buffer full audio before sending.

---

## SQLite Schema (`aura.db`)

```sql
CREATE TABLE IF NOT EXISTS reminders (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    text        TEXT NOT NULL,
    fire_at     DATETIME NOT NULL,
    target_node TEXT NOT NULL,
    agent_id    TEXT NOT NULL DEFAULT 'personal',
    fired       INTEGER DEFAULT 0,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_reminders_fire ON reminders(fire_at, fired);

CREATE TABLE IF NOT EXISTS webhooks (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    key         TEXT NOT NULL,
    payload     TEXT NOT NULL,
    processed   INTEGER DEFAULT 0,
    received_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX IF NOT EXISTS idx_webhooks_key ON webhooks(key, processed);

CREATE VIRTUAL TABLE IF NOT EXISTS memory_fts USING fts5(
    date,
    agent_id,
    content,
    tokenize = 'porter ascii'
);
```

---

## REST API Contract

**Base URL:** `http://127.0.0.1:3002`

| Method | Endpoint | Response |
|--------|----------|----------|
| GET | `/api/health` | `{status, uptime_s, version, nodes_connected, agents_loaded}` |
| GET | `/api/nodes` | `NodeStatus[]` |
| GET | `/api/nodes/:id/events` | `ANPEvent[]` last 100 |
| POST | `/api/nodes/:id/command` | `{cmd, payload}` → 204 |
| POST | `/api/nodes/:id/rotate-token` | `{token: "sk-aura-..."}` |
| GET | `/api/agents` | `AgentProfile[]` |
| GET | `/api/agents/:id/memory/search?q=` | `{results: string[]}` |
| GET | `/api/agents/:id/memory/:date` | episodic .md as plain text |
| DELETE | `/api/agents/:id/memory/:date` | 204 |
| GET | `/api/skills` | `SkillDefinition[]` |
| POST | `/api/skills/:name/toggle` | `{enabled: bool}` → 200 |
| GET | `/api/canvas` | `CanvasBlock[]` |
| DELETE | `/api/canvas` | 204 |
| POST | `/api/heartbeat/run` | triggers run → 200 |
| GET | `/api/heartbeat/log` | `HeartbeatLogEntry[]` last 50 |
| GET | `/api/webhooks/:key` | `WebhookPayload[]` |
| DELETE | `/api/webhooks/:key` | 204 |
| POST | `/webhook/:key` | inbound from Zapier/n8n → stored → 200 |

---

## Cross-Cutting Concerns

| Concern | File | Detail |
|---------|------|--------|
| Tool call loop guard | `server.ts` | Max 5 LLM iterations per utterance. After 5 → throw. |
| Vision routing override | `llm/router.ts` | `image_b64` present → force vision tier. Only Sonnet + GPT-4o. |
| Agent memory isolation | `memory/manager.ts` | All reads/writes scoped to `agent.memory_ns`. No cross-agent access. |
| Channel auto-register | `channels/manager.ts` | First message → generate token, store in-memory, log. |
| Heartbeat silence | `scheduler/heartbeat.ts` | LLM must return exactly `HEARTBEAT_OK` to exit silently. Log every run. |
| Proactive rate limit | `scheduler/proactive.ts` | Max 10 `send_message` per heartbeat run. |
| Skill hot-reload | `skills/engine.ts` | chokidar on `~/.aura/skills/`. Change → clear cache, re-import, update registry. |
| Self-write safety | `skills/self_write.ts` | `tsc --noEmit --strict` before save. shell/self_write absent from self-written contexts. |
| Token budget | `llm/context.ts` | > 6000 tokens → compress oldest 10 turns via Haiku before main call. |
| Config hot-reload | `config/loader.ts` | chokidar on all config files. Reload + notify subsystems. No restart. |
| mDNS | `anp/server.ts` | Advertise `_aura-gw._tcp.local:8765` on startup. Unpublish on shutdown. |
| Node revocation | `anp/server.ts` | nodes.yaml change → close WebSocket of removed node_id within 5s. |
| ElevenLabs streaming | `voice/elevenlabs.ts` | Stream 1KB chunks immediately. Never buffer full audio. |
| Canvas persistence | `canvas/renderer.ts` | Write canvas.json after every block op. Load on startup. |
| Webhook security | `api/rest.ts` | Rate-limited 60 req/min per key. Payload stored as string, never eval'd. |
| WhatsApp session | `channels/whatsapp.ts` | QR scan first run. Session at `~/.aura/tokens/whatsapp/`. Auto-restores. |
| Signal CLI | `channels/signal.ts` | Java binary, JSON-RPC over stdin/stdout. Requires Java 17+. |

---

## Validation

### Step 1–2: Config + ANP Server
```bash
cd gateway && npm install && npx tsx src/server.ts &
sleep 2

wscat -c ws://localhost:8765/anp
# Valid → WELCOME:
# {"anp":"1.0","type":"hello","node_id":"test_01","token":"<valid>","caps":["text_in","text_out"],"meta":{}}

# Invalid → REJECT 401:
# {"anp":"1.0","type":"hello","node_id":"bad","token":"sk-aura-invalid","caps":[],"meta":{}}

# Duplicate → REJECT 409: connect test_01 again while connected

# 5s timeout: connect, send nothing, confirm close

# PING → PONG:
# {"type":"ping","ts":1234567890}

# mDNS:
dns-sd -B _aura-gw._tcp local
# Expect: gateway appears within 3s
```

### Step 3: LLM Router
```bash
npx tsx -e "
import { LLMRouter } from './src/llm/router.ts';
import { loadConfig } from './src/config/loader.ts';
const router = new LLMRouter(loadConfig());
const p = { system: 'Be brief.', messages: [{role:'user' as const, content:'Say: pong'}] };
for (const tier of ['simple','complex','offline'] as const) {
  const r = await router.complete(tier, p);
  console.log('✓', tier, r.provider+'/'+r.model, r.text.slice(0,20));
}
"
# ✓ simple  claude/claude-haiku-4-5   pong
# ✓ complex claude/claude-sonnet-4-6  pong
# ✓ offline ollama/llama3.2:3b        pong
```

### Step 4: Multi-Agent Registry
```bash
npx tsx -e "
import { AgentRegistry } from './src/agents/registry.ts';
const reg = new AgentRegistry();
await reg.load();
const cases = [
  ['aura_body_01','personal'],['telegram_999','personal'],
  ['slack_U0123','dev'],['discord_dm_1','dev'],
  ['telegram_social','social'],['unknown_xyz','default'],
];
for (const [node_id, expected] of cases) {
  const a = reg.resolve(node_id);
  console.assert(a.id === expected, node_id + ' expected ' + expected + ' got ' + a.id);
  console.log('✓', node_id, '→', a.id);
}
"
```

### Step 5: All 11 Skills
```bash
npx tsx -e "
import { SkillsEngine } from './src/skills/engine.ts';
const engine = new SkillsEngine();
await engine.load();
const required = ['web_search','notion','spotify','apple_notes','twitter',
  'webhooks','whoop','home_assistant','google_calendar','reminders','body_control'];
const loaded = engine.listSkills().map(s => s.name);
for (const r of required) {
  console.assert(loaded.includes(r), 'MISSING: ' + r);
  console.log('✓ Loaded:', r);
}
const results = await engine.execute('search', {query:'test'}, mockCtx);
console.assert(Array.isArray(results));
console.log('✓ web_search works');
"
# Hot-reload: edit any skill YAML, watch for "[Skills] Reloaded: X" in logs
```

### Step 6: Self-Improving Skills
```bash
# Start gateway, connect test node, send:
# "Create a skill that fetches my current public IP address"

# Expected logs:
# [LLM] tool_call: create_skill
# [SelfWrite] Generating files for: ip_address
# [SelfWrite] TypeScript validation passed
# [SelfWrite] Saved ~/.aura/skills/ip_address.yaml + .ts
# [Skills] Reloaded: ip_address (source: self_written)

# Then ask: "What is my public IP?"
# Confirm: agent calls ip_address tool, returns IP

ls ~/.aura/skills/ip_address.* && echo "✓ Files exist"
grep 'source: self_written' ~/.aura/skills/ip_address.yaml && echo "✓ Tagged"
```

### Step 7: Memory + Context
```bash
npx tsx -e "
import { MemoryManager } from './src/memory/manager.ts';
const mem = new MemoryManager();
await mem.init();

await mem.addTurn('s1','personal','user','What time is standup?');
await mem.addTurn('s1','personal','assistant','Standup is at 10am.');
console.assert(mem.getShortTerm('s1').length === 2);
console.log('✓ Short-term: 2 turns');

await mem.writeEpisodic('personal','2026-02-27','User asked about standup at 10am.');
const ep = await mem.readEpisodic('personal','2026-02-27');
console.assert(ep.includes('standup'));
console.log('✓ Episodic written');

await mem.indexEpisodic('personal','2026-02-27', ep);
const hits = await mem.search('personal','standup');
console.assert(hits.length > 0);
console.log('✓ FTS5 search hit');

const devHits = await mem.search('dev','standup');
console.assert(devHits.length === 0, 'dev should not see personal memory');
console.log('✓ Agent isolation confirmed');
"
```

### Step 8: Full Event Loop
```bash
npx tsx src/server.ts & && sleep 2
wscat -c ws://localhost:8765/anp
# HELLO → WELCOME
# Send: {"type":"event","event":"utterance","node_id":"test_01","session_id":"s1","ts":0,"payload":{"text":"what time is it","routing_hint":"simple"}}
# Expect within 4s: speak command with current time

# Tool use: "search the web for latest AI news"
# Expect: complex tier, search tool called, news in speak command

# Vision: utterance with image_b64 field
# Expect: vision tier forced, description in speak command
```

### Step 9: All Channel Adapters
```bash
# Telegram: message bot "hello AURA" → confirm personal agent reply within 5s
# Slack: DM bot → confirm dev agent persona (technical tone)
# Discord: DM bot → confirm reply
curl -s http://localhost:3002/api/nodes | jq '[.[] | select(.type=="channel") | {id:.node_id,ok:.connected}]'
# All enabled channels: connected: true
```

### Step 10: Scheduler + Proactive
```bash
# Set heartbeat_interval_min: 1 for testing
# Connect as aura_body_01, send: "remind me in 30 seconds to drink water"
# Wait 30s → expect alert command on body node

# Heartbeat silence: wait 60s
# Expect: "[Heartbeat] Running... HEARTBEAT_OK"

curl -X POST http://localhost:3002/api/heartbeat/run
# If morning window: morning briefing sent
# Else: HEARTBEAT_OK in logs
```

### Step 11: ElevenLabs Voice
```bash
npx tsx -e "
import { textToSpeech } from './src/voice/elevenlabs.ts';
const start = Date.now();
const pcm = await textToSpeech('Hello, this is AURA.', '21m00Tcm4TlvDq8ikWAM');
const ms = Date.now() - start;
console.assert(pcm.length > 0);
console.assert(ms < 2000, 'Too slow: ' + ms + 'ms');
console.log('✓ ElevenLabs:', pcm.length, 'bytes in', ms, 'ms');
"
```

### Step 12: Canvas
```bash
# Browser WS → ws://localhost:3001/canvas
# Send utterance: "show me a canvas with today's date and my top 3 tasks"
# Browser receives: state → append → append → append (live)
curl -s http://localhost:3002/api/canvas | jq 'length'  # 3+ blocks

# Persistence: kill gateway, restart
curl -s http://localhost:3002/api/canvas | jq 'length'  # same blocks
```

### Step 13: Full End-to-End
```bash
npx tsx src/server.ts & && sleep 3

# 1. Body voice → ElevenLabs → body speaker
# HELLO as aura_body_01, send weather utterance
# Expect: speak cmd + audio_chunk stream

# 2. Telegram → personal agent → reply with web_search
# 3. Slack → dev agent → different persona confirmed
# 4. Self-written skill survives gateway restart
# 5. Node revocation: delete from nodes.yaml → WS closes within 5s
# 6. Canvas: utterance → live browser blocks
# 7. Webhook: POST /webhook/test → ask agent to read it → lists payload
# 8. Memory across restart: converse → kill → restart → ask recall → confirmed

echo "All E2E paths complete"
```

**Success criteria:**
- All 4 LLM adapters return valid responses
- All 7 channels deliver messages and receive replies with correct agent persona
- All 11 skills load; web_search executes without API key
- Self-written skill installs and runs in same session, tagged source: self_written
- ElevenLabs first byte < 2s, PCM chunks arrive at body node
- Canvas live updates in browser WS, persists across restart
- Heartbeat silent when nothing to do; reminder fires within 5s of scheduled time
- Node revocation within 5s of nodes.yaml change
- Memory persists across restart; dev agent cannot read personal memory

---

## Security Checklist

- [ ] `security.bind_address: 127.0.0.1` — never 0.0.0.0
- [ ] REST (3002) and Canvas (3001) also bind to 127.0.0.1 only
- [ ] Remote access only via Tailscale — zero open firewall ports
- [ ] All tokens: `sk-aura-` + `crypto.randomBytes(32).toString('hex')`
- [ ] `.env` and `nodes.yaml` in `.gitignore`
- [ ] Self-write: `tsc --noEmit --strict` must pass before save
- [ ] Self-written skills cannot call shell or self_write tools
- [ ] Tool loop: max 5 iterations, hardcoded, not configurable
- [ ] Proactive: max 10 send_message per heartbeat, hardcoded
- [ ] Webhook payloads stored as JSON string, never eval'd
- [ ] WhatsApp session at `~/.aura/tokens/whatsapp/` — not in source
- [ ] Signal phone number in .env — never hardcoded

---

## Environment Variables

```bash
# LLMs
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=...

# Voice
ELEVENLABS_API_KEY=...

# Channels
TELEGRAM_BOT_TOKEN=...
SLACK_BOT_TOKEN=xoxb-...
SLACK_SIGNING_SECRET=...
SLACK_APP_TOKEN=xapp-...
DISCORD_BOT_TOKEN=...
SIGNAL_PHONE_NUMBER=+6591234567
GOOGLE_CHAT_CREDENTIALS_PATH=~/.aura/tokens/google_chat.json
TEAMS_APP_ID=...
TEAMS_APP_SECRET=...

# Skills
SERPAPI_KEY=...                          # optional
NOTION_API_KEY=secret_...
SPOTIFY_CLIENT_ID=...
SPOTIFY_CLIENT_SECRET=...
GOOGLE_OAUTH_CREDENTIALS=~/.aura/tokens/google.json
TWITTER_API_KEY=...
TWITTER_API_SECRET=...
TWITTER_ACCESS_TOKEN=...
TWITTER_ACCESS_SECRET=...
BLUESKY_IDENTIFIER=user.bsky.social      # optional
BLUESKY_PASSWORD=...                     # optional
WHOOP_CLIENT_ID=...
WHOOP_CLIENT_SECRET=...
HA_URL=http://homeassistant.local:8123
HA_TOKEN=...
```
