# Conversational Agent — Message Pipeline Design

**Audience:** Frontend and backend engineers building a chat/agent product where
the assistant streams responses and may invoke tools.

**Goal:** Define how user input is accepted, queued, processed, and responded to
when the assistant is already busy. This is the same pattern Claude Code uses.

---

## TL;DR

We run a **single-threaded conversation loop** with a **side queue** managed by
the frontend. The first message runs alone. Anything the user sends while the
assistant is busy goes into the queue. At the end of each internal step of the
loop, the backend sweeps the queue and folds all waiting messages into the
**same** conversation as follow-up user messages. This produces fewer model
turns, preserves context, and gives the user a deterministic experience.

Counterintuitive behaviors this produces — call them out to the team early:

- 4 messages typed in quick succession typically produce **2** assistant
  replies, not 4.
- The very first in-flight message **never** merges with later messages.
- Pressing Cancel/Esc must clear **both** the in-flight request and the queue.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                          FRONTEND                                │
│                                                                  │
│  ┌──────────┐   ┌─────────────┐   ┌──────────────────────────┐  │
│  │  Input   │──▶│ Dispatcher  │──▶│  Queue (FIFO, in-memory) │  │
│  │  Box     │   │             │   │                          │  │
│  └──────────┘   └─────────────┘   └────────────┬─────────────┘  │
│                       │                         │                │
│                       │ (lock idle)             │ (drain)        │
│                       ▼                         ▼                │
│                ┌───────────────────────────────────────┐         │
│                │  StreamClient (one in-flight request) │         │
│                └─────────────────┬─────────────────────┘         │
└──────────────────────────────────┼──────────────────────────────┘
                                   │  SSE / WebSocket
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                          BACKEND                                 │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │   Turn Loop:                                            │   │
│   │     while (true) {                                      │   │
│   │       streamModelReply()                                │   │
│   │       executeTools()                                    │   │
│   │       ★ fold(): pull queued user msgs from FE ★         │   │
│   │       if (!needsFollowUp && noFolded) break             │   │
│   │     }                                                   │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

Key property: **the queue lives on the frontend, but the backend loop decides
when to read it.** This is what makes batching deterministic.

---

## 2. State Machine (Frontend)

A single global lock with three states:

```
idle ── reserve() ──▶ dispatching ── tryStart() ──▶ running ── end() ──▶ idle
```

- `idle` — no active request; queue drainer is free to start one.
- `dispatching` — request being built, not yet on the wire. Brief; prevents
  double-fire.
- `running` — request streaming. New user input goes to the queue.

Only one transition path. No concurrent `running` states.

---

## 3. The Queue (Frontend)

In-memory FIFO. For most products a single priority is enough. If you need
priorities (e.g. system-injected events vs. user messages), use three levels:

```ts
type Priority = 'now' | 'next' | 'later'  // 0 = highest
```

- `now` — internal/system-injected, head of line.
- `next` — user messages (default).
- `later` — background/notification messages.

FIFO within each priority. The queue is **not** persisted; it's per-session.

### What goes in the queue
- Plain user prompts sent while the lock is `running`.

### What does NOT go in the queue
- Slash/local commands (they need client-side handling, not model injection) —
  reject these with a UI hint if sent while busy, or hold them for after the
  turn.
- Anything sent while the lock is `idle` — those execute immediately.

---

## 4. Frontend Responsibilities

### 4.1 Input submission
```ts
function onSubmit(text: string) {
  if (isSlashCommand(text)) {
    if (lock.state === 'running') { rejectWithToast(); return }
    runLocalCommand(text); return
  }

  if (lock.state === 'idle') {
    lock.reserve()
    startRequest(text)
  } else {
    queue.push({ text, priority: 'next', uuid: uuid() })
    // Optional: if a cancelable tool is in flight, abort it before queuing
    // so the user feels their new input is acknowledged faster.
  }
}
```

### 4.2 Queue draining
A subscription to `(lock.state, queue.length)`:
```ts
useEffect(() => {
  if (lock.state !== 'idle') return
  if (queue.length === 0) return
  const next = queue.shift()
  lock.reserve()
  startRequest(next.text)
}, [lockState, queueLength])
```

### 4.3 Streaming
Open a single SSE/WebSocket per turn. Render:
- Assistant tokens as they stream.
- Tool calls and their results inline (or in a collapsed view).
- Queued messages in the transcript with a distinct "queued" style as soon
  as they're enqueued — don't wait for the backend to acknowledge them.

### 4.4 Mid-turn queue exposure to backend
There are two acceptable transport patterns. Pick one and document it.

**Pattern A — Push:** the frontend opens a separate channel (e.g. a small
HTTP POST per queued message, or a multiplexed WS frame) to deliver queued
messages to the backend, which appends them to a server-side per-turn buffer.

**Pattern B — Pull:** the backend, at each fold point, sends a "queue?" frame
on the stream and the frontend replies with the current snapshot. Simpler
state, more round-trips.

Claude Code uses Pattern A in spirit (queued messages are made available to
the in-flight turn via shared in-process state because the loop runs in the
same process as the UI). For a client/server split, **Pattern A is recommended**.

### 4.5 Cancellation
Esc / Cancel must do **two** things:
1. Send abort to the in-flight request (`AbortController.abort()` + signal
   the backend to stop streaming and kill running tools).
2. Clear the queue.

Do not do only one of these — users will be confused by ghost messages
appearing after cancellation.

---

## 5. Backend Responsibilities

### 5.1 The turn loop
Pseudocode:
```ts
async function* runTurn(initialMessages, ctx) {
  let messages = initialMessages
  while (true) {
    let needsFollowUp = false
    const assistantMsg = await streamModelReply(messages)  // yields chunks
    yield assistantMsg

    const toolCalls = extractToolCalls(assistantMsg)
    if (toolCalls.length > 0) {
      needsFollowUp = true
      const results = await executeTools(toolCalls, ctx)   // may run in parallel
      yield* results
      messages = [...messages, assistantMsg, ...results]
    } else {
      messages = [...messages, assistantMsg]
    }

    // ★ FOLD POINT ★
    const queued = await fetchQueuedUserMessages(ctx)      // see §4.4
    if (queued.length > 0) {
      const injected = queued.map(toUserMessage)
      yield* injected
      messages = [...messages, ...injected]
      // Loop continues — model will see queued messages on next iteration.
      continue
    }

    if (!needsFollowUp) break
    if (ctx.signal.aborted) break
    if (ctx.turnCount++ > MAX_TURNS) break
  }
}
```

### 5.2 The fold point — when and what
- **When:** at the end of every iteration of the turn loop, after model reply
  + tool execution.
- **What:** snapshot whatever queued user messages exist at that instant, in
  FIFO order, and inject them as `user` messages into the conversation.
- **Result:** the next iteration of the loop sends them to the model as
  continuation, not as a new conversation.

This is the single most important detail. The fold is **structural** (tied to
loop iterations), not **time-based** (no polling interval).

### 5.3 Continuation rules
The loop continues if **any** of:
- Model produced tool calls (needs results sent back).
- Fold injected new user messages (something new to respond to).

The loop ends if:
- No tools requested AND no queued messages AND abort not signaled.
- Abort signaled.
- Turn count cap hit.

### 5.4 Tool execution
Within a single turn iteration, tool calls can run in parallel via a worker
pool. This is a separate concern from message queueing: tool parallelism is
intra-turn; message queueing is inter-message. Don't conflate them.

### 5.5 Cancellation handling
- Wire an `AbortSignal` through every async boundary (model stream, tool
  workers, fold fetch).
- On abort: stop reading from the model stream, signal tool workers to stop
  (cooperatively if possible), do **not** run the fold step, return with
  reason `aborted`.

---

## 6. The FE↔BE Contract

A minimal contract is enough. Pick concrete shapes for your transport.

### Request (FE → BE, start turn)
```ts
POST /turn
{
  conversationId: string,
  userMessage: { text: string, attachments?: [...] },
  options: { model: string, tools: [...], maxTurns?: number }
}
```

### Stream events (BE → FE)
```
event: assistant_chunk        // streaming tokens of model reply
event: tool_call              // model invoked a tool
event: tool_result            // tool completed
event: queued_user_injected   // the BE folded a queued message in
event: turn_end               // turn complete
event: error
```

### Queue delivery (FE → BE, mid-turn)
```ts
POST /turn/{turnId}/queue
{ messages: [{ uuid, text, attachments? }] }
```
The backend appends these to a per-turn buffer that `fetchQueuedUserMessages`
reads at the next fold point.

### Cancel (FE → BE)
```ts
POST /turn/{turnId}/cancel
```

---

## 7. Behavioral Specification

These are the user-visible behaviors that must hold. Add them to the test plan.

| # | Behavior |
|---|---|
| B1 | Submitting while idle starts a new turn immediately. |
| B2 | Submitting while running enqueues without blocking the UI; input box clears; queued message is visible in the transcript with a "waiting" style. |
| B3 | The first in-flight message never merges with later messages — it always gets its own reply. |
| B4 | Multiple messages queued while busy are batched at the next fold point and produce a single combined follow-up reply. |
| B5 | If queued messages arrive at different fold points (e.g. between tool rounds), they may produce separate replies. |
| B6 | Cancel aborts the in-flight stream **and** clears the queue in one action. |
| B7 | Slash/local commands sent while busy are not queued; they are either rejected with a hint or held for after the turn. |
| B8 | Order is preserved: queued messages are injected FIFO; the model sees them in the order the user sent them. |
| B9 | No parallel turns: at most one model request is in flight per conversation at any time. |
| B10 | Reconnecting mid-turn (FE refresh) does not double-process queued messages. (Backend deduplicates by `uuid`.) |

---

## 8. Edge Cases & Pitfalls

- **Race at the fold:** If the FE pushes a queued message at the exact moment
  the BE is reading the buffer, that message must either be in this fold or
  the next — never lost, never duplicated. Use server-side append + atomic
  drain.
- **Long-running tools:** Don't block the fold on slow tools. The fold should
  read whatever buffer exists; the next iteration's fold will pick up newer
  messages.
- **Abort during fold:** If abort fires after the stream chunk but before the
  fold completes, treat as aborted; do **not** inject queued messages into a
  dying turn.
- **UUID-based dedup:** Every queued message carries a client-generated UUID.
  The backend deduplicates on append. This protects against retries and
  reconnects.
- **Slash commands:** They are not text-for-the-model; they're client actions.
  Do not put them through the same pipeline.
- **Multi-agent / sub-agent fan-out (if applicable):** Each agent's loop has
  its own fold filter — typically scoped by an agent ID so messages addressed
  to one agent don't leak into another's conversation.

---

## 9. What This Design Buys You

- **Determinism:** one turn at a time means no interleaved tool calls
  mutating shared state mid-response.
- **Context preservation:** queued messages join the same conversation, so
  follow-ups like "and also do X" work naturally.
- **Fewer model calls:** batching 3 quick messages into one combined
  follow-up reply saves a model round-trip vs. running each as its own turn.
- **Predictable cancellation:** a single lock + a single queue means "stop"
  has an unambiguous meaning.
- **Simple mental model for users:** "the assistant is busy → my message
  is waiting → it'll be answered next."

---

## 10. What NOT to Do

- ❌ Don't run multiple model requests in parallel per conversation. It
  destroys context and creates merge headaches.
- ❌ Don't drop messages silently when busy. Users will retype and you'll
  end up processing duplicates.
- ❌ Don't merge queued messages into the in-flight first message. The
  model has already started replying; you can't retroactively change its
  input without canceling and restarting (which is a different feature —
  "edit your message").
- ❌ Don't poll the queue on a timer. Tie fold timing to the loop structure
  so the system has zero idle CPU when the queue is empty.
- ❌ Don't put slash commands through the same path. They are not model
  input.

---

## 11. Implementation Checklist

**Frontend**
- [ ] Three-state lock with `reserve` / `tryStart` / `end`.
- [ ] FIFO queue with UUID-tagged items, rendered in transcript.
- [ ] Drainer effect watching `(lock, queue)`.
- [ ] Cancel handler that aborts request AND clears queue.
- [ ] Slash command rejection-or-deferral when busy.
- [ ] Queue delivery transport to backend (Pattern A recommended).

**Backend**
- [ ] Turn loop with explicit fold point at end of each iteration.
- [ ] Per-turn queued-message buffer with atomic append + drain.
- [ ] UUID dedup on enqueue.
- [ ] AbortSignal threaded through model stream and tool workers.
- [ ] Continuation rule: `needsFollowUp || foldedAny`.
- [ ] `MAX_TURNS` safety cap.
- [ ] Stream event schema with explicit `queued_user_injected` event so the
      FE can show the user "your queued message is now being processed."

---

## 12. One-Line Summary for the Whole Team

> Single-threaded conversation loop on the backend, FIFO queue on the
> frontend; at the end of each loop iteration the backend folds whatever
> the user queued into the same conversation as follow-up user messages,
> so quick successive inputs collapse into one combined assistant reply,
> and Cancel always wipes both the active stream and the queue.
