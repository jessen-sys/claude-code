# Claude Code Message Processing Pipeline — Complete Summary

A deep-dive into how Claude Code handles user message submission, queueing,
streaming, tool execution, and concurrent input — based on the leaked source
in `src/`.

---

## 1. Core Components

| Component | File | Role |
|---|---|---|
| Input UI | `src/components/PromptInput/PromptInput.tsx` | Terminal input box; fires `onSubmit` on Enter |
| Dispatcher | `src/utils/handlePromptSubmit.ts` | Decides "execute now" vs "enqueue" |
| State lock | `src/utils/QueryGuard.ts` | Three-state machine: `idle / dispatching / running` |
| Message queue | `src/utils/messageQueueManager.ts` | Three-priority FIFO: `now / next / later` |
| Queue driver | `src/hooks/useQueueProcessor.ts` | Auto-drains queue when idle |
| Main loop | `src/query.ts` (lines 307, 700-863, 1570-1643, 1728) | Streaming + tool loop + queue folding |
| Cancellation | `src/hooks/useCancelRequest.ts` | Esc → abort + clear queue |

---

## 2. State Machine

```
idle ── reserve() ──▶ dispatching ── tryStart() ──▶ running ── end() ──▶ idle
```

**Only one query can be in `running` at any time.** This single lock is the
root cause of all queueing behavior.

---

## 3. Submit Path

```
User presses Enter
   │
   ▼
handlePromptSubmit
   ├─ QueryGuard idle    → execute immediately (reserve → running)
   └─ QueryGuard running → enqueue with 'next' priority
                          (slash commands are an exception → dropped)
```

---

## 4. The Per-Turn Main Loop (`while (true)` in `query.ts`)

```
                 ┌──────────────────────────────────────┐
                 │                                       │
                 ▼                                       │
        Model streams reply ──▶ Tool calls (parallel) ──▶ Tool results
                                                  │      │
                                                  ▼      │
                                          ★ Fold Point ★ │
                                          Runs every     │
                                          iteration.     │
                                          Snapshot queue,│
                                          convert all to │
                                          queued_command │
                                          attachments,   │
                                          inject into    │
                                          conversation.  │
                                                  │      │
                                                  ▼      │
                      needsFollowUp || fold pulled msgs? │
                                  │                      │
                                  yes ────────────────────┘
                                  │
                                  no → turn ends
                                       QueryGuard → idle
                                       useQueueProcessor checks leftovers
```

---

## 5. Fold Trigger Mechanism

The fold is **not timer-based**. It's a structural step in `while(true)` that
runs once at the end of every "model reply + tool execution" round.
**The fold itself always runs.** The real question is whether the loop iterates
again.

### Signals controlling continuation

| Signal | Location | Effect |
|---|---|---|
| `needsFollowUp` | `query.ts:834` | Assistant produced `tool_use` blocks → must iterate to send tool results back |
| Fold pulled queued msgs | `query.ts:1570-1590` | Queued messages injected as attachments → new user content to respond to |
| `sleepRan` | `query.ts:1566` | Sleep tool used → fold priority expands from `next` to `later` (pulls task notifications) |
| `signal.aborted` | `query.ts:839, 849` | Esc/interrupt → skip fold, exit with `aborted_streaming` |
| `max_turns` cap | `query.ts:1711` | Hard upper bound → exit with `max_turns` |

### Filtering rules at the fold (`query.ts:1572-1578`)

- **Slash commands are never folded** — they stay in the queue and are handled
  by `useQueueProcessor` after the turn ends (they need `processSlashCommand`,
  not raw text injection).
- **Agent isolation** — the main thread only pulls items with
  `agentId === undefined` (real user prompts); sub-agents only pull
  `task-notification` items addressed to their own `agentId`, never user
  prompts.

### One-line understanding

> After every "model speaks + tools finish" round, the system forces a pause
> and asks three things: Is there anything new in the queue? Does the model
> still want to call tools? Did the user hit Esc? No timers — purely driven
> by model behavior and queue contents.

---

## 6. Key Principle: M1 Never Merges Replies With Later Messages

**When M1 starts, the queue is empty. The model only sees M1, so reply1
addresses only M1.** Later messages must wait for the fold point to be
injected, and the model then produces reply2 to address them.

### Timing → reply count

| Timing | Reply count | Why |
|---|---|---|
| M2 sent after M1 fully finishes | 2 (two independent turns) | Each starts its own turn |
| M2 sent while M1 is running | **2** (same turn: reply1 + fold + reply2) | M1 locks the turn before M2 is visible |
| M2 sent in the tiny `dispatching` window before M1 truly starts (practically unreachable) | 1 | M2 joins M1's initial conversation |

---

## 7. Typical Scenarios

### Scenario 1: Send M1, then immediately send M2

```
You:         M1
Assistant:   reply1  ← only addresses M1
You(queued): M2
Assistant:   reply2  ← addresses M2
```

**Result: 2 replies.**

### Scenario 2: Rapid-fire 4 messages while M1 is running

```
t=0   M1 enters → locks running, starts
t<1s  M2 M3 M4 enter → queued [M2, M3, M4]
```

- **M1 is plain text**: M1 finishes → fold pulls M2 M3 M4 → merged reply →
  **2 replies** (reply1 for M1; reply2 merges M2+M3+M4).
- **M1 calls one tool**: tool finishes → fold pulls M2 M3 M4 → merged reply →
  **2 replies**.
- **M1 calls multiple slow tools, M2/M3/M4 arrive at different times**: each
  fold point pulls whoever has arrived → **possibly 3–4 separate replies**.

### What does NOT happen

- ❌ No parallel queries (global lock)
- ❌ No message dropping (except slash commands)
- ❌ No new conversation per message (all share one context)
- ❌ No waiting for "all messages to arrive" before processing
- ❌ **M1 never merges its reply with later-arriving messages**

### Esc behavior

- Current stream is aborted.
- Any remaining queued messages are cleared.
- Lock returns to idle.

---

## 8. Design Takeaways

1. **Single-threaded turns** — only one query runs at a time.
2. **Queue, don't drop** — input during busy state is preserved (slash commands excepted).
3. **Batch folding** — fold is a one-time snapshot; everything eligible is pulled in one shot.
4. **Same conversation context** — queued messages are injected as follow-up user messages, so they can refer back to earlier replies.
5. **Structurally triggered, not periodic** — fold timing depends on turn progress, not a fixed interval.
6. **Reply count ≠ message count** — M messages may produce 1–M assistant replies.
7. **M1 always gets its own reply** — the first message has its own response; later ones merge at the fold point.
8. **Two-layer interrupt** — Esc both stops the current stream and clears the queue.
9. **Tool-level parallelism is separate** — `streamingToolExecutor` runs multiple tools concurrently within a turn; that's orthogonal to the serial message queue.
10. **Agent isolation** — fold filters by `agentId`; sub-agents never see the user's raw prompt.
11. **Sleep is an explicit checkpoint** — using the Sleep tool widens the fold's priority to `later`, deliberately pulling in background task notifications.

---

## 9. Ultimate One-Liner

> Claude Code uses a global lock so only one conversation turn runs at a time;
> when M1 starts, it locks the turn and **reply1 addresses only M1**; new
> messages sent while busy enter a FIFO queue; after each "model reply + tool
> execution" round, the loop reaches a **fold point** that injects every
> eligible queued message into the same conversation as follow-up user
> messages, after which the model produces reply2 to address them; whether
> the loop continues is decided by "did the model call tools?" and "did the
> fold pull in new messages?"; Esc both aborts the active stream and clears
> the queue.
