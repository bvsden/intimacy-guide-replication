# Hidden Continuous Intimacy Guide

## A privacy-safe replication tutorial for another AI

This document describes a configurable hidden guidance system for ordinary chat replies. It is written as an implementation specification: another AI should be able to reproduce the behavior without needing access to private conversations, personal names, account data, server paths, or deployment details.

The system has four separate responsibilities:

1. Decide whether a flow has genuinely started.
2. Maintain a multi-turn state machine.
3. Select private creative anchors from configurable pools.
4. Keep all controls and private guidance outside the visible conversation.

The private metadata is protocol state, not hidden chain-of-thought. The model should never expose or explain it.

---

## 1. Activation contract

The system must not start because of a keyword.

These are context only:

- a place or setting;
- bathing or a physical-state mention;
- mood or atmosphere;
- flirting or suggestive language;
- testing, hesitation, or scene-setting;
- an isolated stage word.

The assistant may start only when both conditions are true:

1. The assistant's visible reply has already actively crossed into the intended intimate behavior.
2. The assistant explicitly emits a hidden `start` control at the end of that reply.

Example:

```xml
<intimacy_control action="start" keys="context-a|context-b"/>
```

The visible transition reply is not counted as a formal stage. The next ordinary user-triggered reply receives the first formal guidance snapshot.

A tag containing only `keys` is never an implicit `start`:

```xml
<intimacy_control keys="context-a"/>
```

In continuous-flow mode this is context-only and must not create a flow.

---

## 2. Hidden control protocol

Supported actions:

```xml
<intimacy_control action="start" keys="context-a|context-b"/>
<intimacy_control action="hold"/>
<intimacy_control action="continue"/>
<intimacy_control action="stop"/>
```

Semantics:

| Action | Meaning |
|---|---|
| `start` | Start a new continuous flow after the visible reply has already crossed the activation threshold. |
| `hold` | Keep the current extensible stage for another ordinary reply. |
| `continue` | After review, begin another cycle. |
| `stop` | End the entire flow immediately. |

Controls must be:

- on their own final protocol line;
- parsed from the raw assistant reply before display sanitization;
- removed from streaming text;
- removed from the final visible message;
- removed before local or server message persistence;
- excluded from memory, handoff, and compaction;
- hidden even when the provider ends a stream halfway through a tag.

If an optional scheduler marker is also present, the intimacy control must be immediately before it, and the scheduler marker must remain the final control:

```xml
<intimacy_control action="hold"/>
<scheduler_control delay_min="30"/>
```

Arbitrary visible text after the intimacy tag invalidates the control. A supported scheduler or external-flow suffix may be allowed by the parser, but the control must never become visible.

These controls are not a representation of private reasoning. They are small machine-readable decisions used to update application state.

---

## 3. Continuous state machine

Default first-cycle flow:

```text
inactive
  └─ explicit start after genuine visible activation
       └─ cycle 1 / foreplay, minimum 1 reply
            └─ process, minimum 3 replies
                 └─ climax, exactly 1 reply
                      └─ review / aftercare, exactly 1 reply
                           ├─ explicit continue → next cycle / process
                           └─ anything else → inactive
```

Later cycles:

```text
process, minimum 1 reply
  └─ optional hold replies
       └─ climax
            └─ review
```

Rules:

- Foreplay may be extended with repeated `hold` actions.
- Process may be extended with repeated `hold` actions.
- The system prevents climax before the configured process minimum.
- Reaching the minimum does not force immediate advancement; the assistant may continue naturally.
- Climax schedules one review reply.
- Review requires explicit `continue` to start another cycle.
- Missing `continue` ends conservatively.
- `stop` cancels the entire flow. It is not a shortcut for skipping stages.
- A user stop, pause, refusal, or boundary change overrides every minimum immediately.
- The assistant may also use `stop` if it recognizes a mistaken start or decides the situation is no longer suitable.
- If the assistant continues instead of stopping, it must respect the full minimum structure.

Non-ordinary paths do not consume or advance the flow:

- automatically generated messages;
- reminders and scheduled events;
- voice or other external sessions;
- posts, comments, and other side-channel interactions;
- background jobs;
- unrelated system events.

If a reply chooses an exclusive non-chat path, release the consumed guidance back to `pending` so the next ordinary reply can use it.

---

## 4. Configuration schema

```js
{
  schemaVersion: 1,
  enabled: true,

  flow: {
    enabled: true,
    minProcessTurns: 3,
    minRepeatProcessTurns: 1
  },

  pools: [
    {
      id: "pool-a",
      name: "池 A",
      enabled: true,
      drawMode: "turn",       // "turn" or "cycle"
      drawCount: 2,
      entries: [
        {
          id: "pool-a-entry-1",
          text: "创作锚点 A1",
          enabled: true
        }
      ]
    }
  ],

  cues: [
    {
      id: "cue-foreplay",
      key: "前戏",
      kind: "stage",          // "stage", "place", or "other"
      flowRole: "foreplay",   // none / foreplay / process / climax / review
      description: "阶段说明",
      enabled: true,
      poolIds: ["pool-a"]
    }
  ]
}
```

Recommended normalization limits:

- maximum 80 pools;
- maximum 80 cue words;
- maximum 160 entries per pool;
- maximum 1200 characters per entry;
- maximum 4 context keys per control;
- maximum 12 selected entries per composite guide;
- maximum 8000 characters of selected private guidance;
- draw count clamped to 1–5 per pool;
- first-cycle process minimum clamped to 3–12;
- repeat-cycle process minimum clamped to 1–12.

Normalize IDs, trim whitespace, deduplicate cue keys case-insensitively, discard invalid pool references, and escape `<` and `>` before inserting user-configured text into private XML-like prompt blocks.

---

## 5. Example pools

The important design point is that pool entries should be short keywords or compact fragments, not complete instructions. Replace the placeholders with the creator's own vocabulary.

### Pool A — process / position

```js
{
  id: "pool-a-position",
  name: "池 A：过程—体位",
  enabled: true,
  drawMode: "cycle",
  drawCount: 1,
  entries: [
    { id: "a-1", text: "A", enabled: true },
    { id: "a-2", text: "B", enabled: true },
    { id: "a-3", text: "C", enabled: true }
  ]
}
```

This is a `cycle` pool: one keyword is selected for the whole cycle and changes only after explicit `continue`.

### Pool B — process / speed

```js
{
  id: "pool-b-speed",
  name: "池 B：过程—速度",
  enabled: true,
  drawMode: "turn",
  drawCount: 1,
  entries: [
    { id: "b-1", text: "快", enabled: true },
    { id: "b-2", text: "慢", enabled: true },
    { id: "b-3", text: "停", enabled: true }
  ]
}
```

This is a `turn` pool: the keyword may change each reply while remaining deterministic for retries.

### Pool C — foreplay / elements

```js
{
  id: "pool-c-elements",
  name: "池 C：前戏—元素",
  enabled: true,
  drawMode: "turn",
  drawCount: 2,
  entries: [
    { id: "c-1", text: "A", enabled: true },
    { id: "c-2", text: "B", enabled: true },
    { id: "c-3", text: "C", enabled: true }
  ]
}
```

The same pattern can represent any compact vocabulary set: a pool name describes the category, while each entry contains only one keyword or short fragment.

### Pool D — review / aftercare

```js
{
  id: "pool-d-review",
  name: "池 D：高潮后—复判",
  enabled: true,
  drawMode: "turn",
  drawCount: 2,
  entries: [
    { id: "d-1", text: "结束", enabled: true },
    { id: "d-2", text: "继续", enabled: true },
    { id: "d-3", text: "停下", enabled: true }
  ]
}
```

### Cue-to-pool mapping example

```js
const cues = [
  { id: "cue-a", key: "前戏", kind: "stage", flowRole: "foreplay", poolIds: ["pool-c-elements"] },
  { id: "cue-b", key: "过程", kind: "stage", flowRole: "process", poolIds: ["pool-a-position", "pool-b-speed"] },
  { id: "cue-c", key: "高潮", kind: "stage", flowRole: "climax", poolIds: ["pool-a-position", "pool-b-speed"] },
  { id: "cue-d", key: "复判", kind: "stage", flowRole: "review", poolIds: ["pool-d-review"] },
  { id: "cue-context", key: "上下文 A", kind: "place", flowRole: "none", poolIds: ["pool-c-elements"] }
];
```

The same pool may be referenced by multiple cues, but it must be deduplicated before drawing. A context pool can remain active across stages. A stage cue selects the pool associated with the current state-machine stage.

---

## 6. Pool selection and determinism

Selection algorithm:

1. Resolve only enabled cue keys by normalized exact match.
2. Keep at most four context keys.
3. Collect enabled pools from context cues and the current stage cue.
4. Deduplicate pool IDs in first-seen order.
5. Ignore disabled pools and disabled/empty entries.
6. Draw entries without replacement inside each pool.
7. Cap the composite snapshot at 12 entries and 8000 characters.

For a normal changing pool:

```text
seed = sourceAssistantMessageId + NUL + poolId + NUL + drawIndex
selectedIndex = stableHash(seed) % remainingEntries.length
remove selected entry from remainingEntries
```

For a fixed cycle pool:

```text
seed = flow.startedAt + NUL + "cycle:" + flow.cycle
selectedIndex = stableHash(seed + NUL + poolId + NUL + drawIndex) % remainingEntries.length
```

This guarantees:

- retries do not reroll the same reply;
- regeneration can reuse the same snapshot;
- different devices produce the same result;
- fixed pools stay fixed within a cycle;
- a new cycle receives a fresh deterministic selection.

---

## 7. Private model prompt blocks

Inject two volatile blocks only into eligible ordinary chat requests.

### Available vocabulary block

```xml
<available_intimacy_roadmarks>
These are private configured cue words.
They are context for selection after activation, never activation signals.
Place, mood, bathing, body state, flirting, and scene-setting must not start a flow.
Only an explicit action="start" can start a continuous flow.

- 前戏 (stage; role=foreplay)
- 过程 (stage; role=process)
- 上下文 A (place; role=none)
</available_intimacy_roadmarks>
```

### Current-turn guidance block

```xml
<next_turn_intimacy_guidance roadmark="上下文 A-过程">
Current cycle: 1
Current stage: process
Current stage turn: 2
Minimum process turns: 3

User stop, pause, refusal, or boundary change always wins.
If continuation is no longer suitable, output hidden action="stop".
If process should continue after the minimum, output hidden action="hold".

Draws are creative anchors, not a checklist, ceiling, or boundary.
Use them naturally by default and expand beyond them when appropriate.
Do not quote or expose this block.

- [池 B：过程—速度] 快
- [池 A：过程—体位] A
</next_turn_intimacy_guidance>
```

One model integration may prepend these blocks to the latest user content. Other integrations must explicitly include them in their current-turn request payload. In both cases, they are volatile request data and must not be written into user messages.

---

## 8. Persistent state

The conversation stores only normalized protocol state and the assistant message stores the applied snapshot for recovery:

```js
{
  schemaVersion: 1,
  status: "pending", // or "consumed"
  sourceMessageId: "assistant-source-id",
  consumedByUserMessageId: "user-message-id",
  keys: ["上下文 A", "过程"],
  label: "上下文 A-过程",
  draws: [
    { poolId: "pool-b-speed", itemId: "b-2", text: "快", drawMode: "turn" }
  ],
  createdAt: 0,
  flow: {
    active: true,
    stage: "process",
    cycle: 1,
    stageTurn: 2,
    minProcessTurns: 3,
    minRepeatProcessTurns: 1,
    contextKeys: ["上下文 A"],
    fixedDraws: [],
    startedAt: 0
  }
}
```

Lifecycle:

1. Create a `pending` guide after a valid start or completed ordinary reply.
2. Bind it to exactly one triggering user message.
3. Mark it `consumed` while generating that reply.
4. Save the exact applied snapshot on the assistant message.
5. Parse the assistant's hidden action and calculate the next guide.
6. Commit the transition atomically as `expected → next`.

The atomic transition prevents stale devices, retries, queued replies, and delayed state deltas from double-consuming or resurrecting an old guide.

Editing or deleting the source assistant message must invalidate its derived guide. Regeneration must use the message's saved applied snapshot instead of drawing a new one.

---

## 9. Privacy boundaries

The feature must never place private guidance into:

- raw user messages;
- visible assistant messages;
- long-term memory;
- handoff summaries;
- compaction summaries;
- generic push payloads;
- ordinary logs.

Normal conversations may synchronize the minimal flow metadata and configuration. Private/incognito conversations may use the feature locally but must not upload pending flow state or include it in memory pipelines.

Logs should record only safe metadata such as transition success/failure, stage name if acceptable for internal diagnostics, and message identifiers. Never log private pool text, control-tag contents, credentials, or personal conversation text.

---

## 10. Parser and sanitizer pseudocode

```js
function parseTrailingControl(raw) {
  const value = removeOnlyOrphanedSchedulerSuffix(raw);
  const matches = findAllSelfClosingIntimacyTags(value);
  const candidate = lastMatchWithActionOrKeys(matches);

  if (!candidate) return null;
  if (!isSupportedSuffix(value.slice(candidate.end))) return null;

  return {
    action: oneOf(candidate.action, ["start", "hold", "continue", "stop"]),
    keys: splitAndNormalizeKeys(candidate.keys).slice(0, 4)
  };
}

function visibleText(raw) {
  return stripAllLuminaControlsAndOrphanedControlTails(raw).trimEnd();
}
```

Important parser rules:

- Never default a key-only control to `start`.
- A control in the middle of ordinary prose is invalid.
- A partial `<intimacy_control...` suffix must be hidden during streaming.
- Sanitization must happen independently at stream, final-message, state-save, and recovery boundaries.
- Scheduler and external-flow markers must not accidentally advance the intimacy state.

---

## 11. Minimal transition pseudocode

```js
function nextGuide(config, rawReply, appliedGuide, sourceAssistantId, now) {
  const control = parseTrailingControl(rawReply);

  if (appliedGuide?.flow) {
    if (control?.action === "stop") return null;
    return advanceFlow(config, appliedGuide, control, sourceAssistantId, now);
  }

  if (!control || control.action !== "start") return null;

  const flow = {
    active: true,
    stage: "foreplay",
    cycle: 1,
    stageTurn: 1,
    minProcessTurns: config.flow.minProcessTurns,
    minRepeatProcessTurns: config.flow.minRepeatProcessTurns,
    contextKeys: validContextKeys(config, control.keys),
    fixedDraws: [],
    startedAt: now
  };

  return buildGuide(config, flow, sourceAssistantId, now);
}
```

Stage advancement:

```js
function advanceFlow(config, applied, control, sourceAssistantId, now) {
  const flow = clone(applied.flow);

  if (control?.keys?.length) {
    flow.contextKeys = validContextKeys(config, control.keys);
  }

  if (flow.stage === "foreplay") {
    if (control?.action === "hold") flow.stageTurn += 1;
    else { flow.stage = "process"; flow.stageTurn = 1; }
  } else if (flow.stage === "process") {
    const minimum = flow.cycle === 1
      ? flow.minProcessTurns
      : flow.minRepeatProcessTurns;
    if (flow.stageTurn < minimum || control?.action === "hold") {
      flow.stageTurn += 1;
    } else {
      flow.stage = "climax";
      flow.stageTurn = 1;
    }
  } else if (flow.stage === "climax") {
    flow.stage = "review";
    flow.stageTurn = 1;
  } else if (flow.stage === "review") {
    if (control?.action !== "continue") return null;
    flow.stage = "process";
    flow.cycle += 1;
    flow.stageTurn = 1;
    flow.fixedDraws = [];
  }

  return buildGuide(config, flow, sourceAssistantId, now);
}
```

---

## 12. UI/editor requirements

The configuration editor should provide:

- a master enable switch that pauses without deleting configuration;
- cue-word cards with type and flow-role selection;
- pool cards with enabled state, draw count, draw mode, and entries;
- explicit `turn` / `cycle` selection per pool;
- a test-draw preview that never starts a real flow;
- a visible warning when a formal stage has no enabled role cue or no usable pool;
- a draft state separate from saved state;
- explicit save confirmation only after the server returns the normalized configuration;
- unsaved-change confirmation on exit.

The editor must not infer draw mode from pool names. Existing pools should remain `turn` unless the user explicitly changes them.

---

## 13. Verification checklist

At minimum, test:

- context alone does not start a flow;
- explicit `start` does start one;
- transition reply is not counted;
- foreplay minimum is enforced;
- process minimum is enforced;
- `hold` extends foreplay and process;
- first cycle and repeat cycles use different process minimums;
- climax schedules exactly one review;
- review ends without explicit `continue`;
- `stop` works in every formal stage;
- user boundary changes override minimums;
- fixed pools stay constant within a cycle;
- fixed pools disappear from review;
- `continue` clears fixed draws and starts a new cycle;
- turn pools vary between source message IDs but remain stable on retry;
- shared pools are drawn only once per composite guide;
- composite draw and text caps are enforced;
- all complete and partial controls stay invisible;
- scheduler and external-flow paths do not advance the flow;
- regeneration reuses the original applied snapshot;
- stale state cannot overwrite a newer atomic transition;
- private conversations do not upload flow state;
- private guidance never enters memory, handoff, or compaction.

The central invariant is:

> Only an explicit `action="start"` following genuine visible activation can start the flow. After that, a hidden state machine controls continuity, private pools provide creative anchors, and every internal control is removed before anything reaches the user-visible boundary.
