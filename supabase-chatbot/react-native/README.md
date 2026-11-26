# Supabase chatbot — React Native client

## Overview

A small Expo (React Native) app for the
[graphrt `examples/supabase-chatbot`](https://github.com/Graphrt/graphrt/tree/main/examples/supabase-chatbot)
backend. Users sign in with Supabase Auth; each message goes straight to
`graphrt serve` with the user's Supabase access token, and the answer streams
back over SSE token by token. Conversations are stored in Supabase
(`chat_sessions` / `chat_messages`, row level security per user).

```text
app ── Supabase Auth (email + password) ──► access token (JWT)
app ── POST {SERVE_URL}/v1/workflows/agent/runs ──► graphrt serve ──► agent + supabase tool (as the user)
       X-User-Authorization: Bearer <jwt>
       {"stream":true,"inputs":{message,session_id},"sys":{history,today}}
    ◄── text/event-stream: response {text} … run_succeeded
app ── rpc start_chat_session / add_chat_message / list_chat_messages ──► Supabase (RLS)
```

## Features

- Supabase email + password sign-in / sign-up; session persisted and refreshed automatically.
- Streaming answers rendered as `response` events arrive, with a "Looking up your data…"
  indicator while the agent runs tools (`node_progress` `tool_call` / `tool_result`).
- History saved to and loaded from `chat_messages`; the last N turns go out as `sys.history`
  (trimmed to stay under serve's 64 KiB body limit).
- Resilience:
  - **401** (expired / invalid token): refresh the Supabase session and retry once; sign out
    if that fails too.
  - **429 with `Retry-After`** (per-user rate limit): wait, then retry (up to 30 s).
  - **429 without `Retry-After`, 502/503/504, network error before the stream starts**:
    exponential backoff with jitter (0.5 s, 1 s, 2 s; 3 retries).
  - **Connection dropped mid-stream / idle 60 s / Stop**: the partial answer stays on screen,
    is not saved, and gets a **Retry** button (no silent re-run: the agent may already have
    acted).
  - `run_failed` shows the server error; `invalid_input` is not retryable.
- No secrets in the app: only the Supabase URL, the anon/publishable key and the serve URL.

Small on purpose: four runtime dependencies besides Expo/React Native
(`@supabase/supabase-js`, AsyncStorage for the session, the URL polyfill Supabase needs on
React Native, safe-area insets). SSE is read with `XMLHttpRequest` progress events — no SSE
or native streaming library.

## Prerequisites

- Node.js ≥ 22.18 (the unit tests run TypeScript directly with `node --test`)
- The Expo Go app on a device, or an Android emulator / iOS simulator
- The backend from graphrt `examples/supabase-chatbot`: a Supabase project with its migration
  applied and `graphrt serve` running with `--verify-jwt` (see that README)

## Installation

```bash
cd supabase-chatbot/react-native
npm install
cp .env.example .env.local   # then fill in the three values
```

## Configuration

Set in `.env.local` (Expo loads it and inlines `EXPO_PUBLIC_*` at build time):

| Variable | Required | Meaning |
| --- | --- | --- |
| `EXPO_PUBLIC_SUPABASE_URL` | yes | `https://<project>.supabase.co` |
| `EXPO_PUBLIC_SUPABASE_ANON_KEY` | yes | The anon / publishable key (`sb_publishable_…`). Public by design; RLS protects the data. |
| `EXPO_PUBLIC_SERVE_URL` | yes | Base URL of `graphrt serve`, e.g. `https://chat.example.com`. Android emulator → host: `http://10.0.2.2:8080`. |
| `EXPO_PUBLIC_WORKFLOW` | no | Workflow name = file stem of the served yaml (default `agent`). |
| `EXPO_PUBLIC_HISTORY_TURNS` | no | Earlier exchanges sent as `sys.history` (default `8`; serve keeps the last 16 messages). |

**Never** put an LLM API key, the Supabase service role key or the `graphrt serve` API key
(`GRAPHRT_API_KEY`) in the app — everything in the bundle can be extracted. The app does
not send an `Authorization` header to serve at all.

A missing or malformed value shows a "Configuration missing" screen listing the problem.

## Quick start

```bash
npm start          # then press i / a, or scan the QR code with Expo Go
```

Sign in (or create an account), ask "What's my display name?" and watch the answer stream.
Plain `http://` serve URLs work in Expo Go during development; release builds need HTTPS
(or an explicit cleartext exception).

## Usage

- **Send** streams a new answer; **Stop** cancels it (the partial text stays, with Retry).
- **Retry** under an incomplete answer resends the same message with the same history; the
  user message is not stored twice.
- **New chat** starts a new `chat_sessions` row on the next message (titled after it).
  On launch the most recent session is reopened (`list_chat_messages`, last 50 messages).
- Only finished answers are saved (`add_chat_message`) and used as history.

## Tests

```bash
npm test           # SSE parser, serve event mapping, retry policy, XHR transport, chat turn, history
npm run typecheck
```

The tests cover chunking at every byte, CR/CRLF, keep-alives, a recorded serve stream,
401 → refresh → retry, 429 `Retry-After`, backoff limits, mid-stream drops and idle timeouts.

## Project layout

```text
App.tsx                  config check, auth gate
src/config.ts            EXPO_PUBLIC_* parsing and validation
src/auth/                Supabase client, access-token get / refresh
src/sse/                 SSE parser, serve event → chat event mapping
src/net/                 XHR event stream, retry policy, errors, abortable sleep
src/chat/chatTurn.ts     one turn: stream + 401/429/backoff rules (I/O injected)
src/chat/history.ts      sys.history selection, request body, validation
src/chat/chatRepository.ts  Supabase RPCs for sessions and messages
src/chat/useChat.ts      UI state: bubbles, streaming, saving, retry
src/ui/                  Login and chat screens
test/                    node:test unit tests
```

## License

Apache-2.0, see [LICENSE](../../LICENSE).
