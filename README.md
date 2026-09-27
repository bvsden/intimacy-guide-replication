# Hidden Continuous Intimacy Guide

## A privacy-safe replication specification for another AI

This document describes a configurable hidden guidance system for ordinary chat replies. It is an implementation specification: another AI should be able to reproduce the behavior without access to private conversations, personal names, real presets, account data, server paths, deployment details, or production state.

The system has five separate responsibilities:

1. Decide whether a flow has genuinely started.
2. Maintain a user-configurable multi-turn state machine.
3. Select private creative anchors from configurable pools.
4. Preserve continuity across retries, edits, recovery, and device sync.
5. Keep all controls and private guidance outside the visible conversation.

Private metadata is application protocol state, not hidden chain-of-thought. The model must never expose or explain it.

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

A tag containing only `keys` is never an implicit start:

```xml
<intimacy_control keys="context-a"/>
```

In continuous-flow mode this is context-only and must not create a flow.

### Recent-climax re-entry

A later flow always requires a new explicit `start`. Before selecting its first stage, inspect the latest four eligible ordinary assistant replies:

- if one of them has a saved applied snapshot proving that it used the semantic climax stage, start from the configured pre-climax stage;
- otherwise start from the first configured stage.

Prefer a stage explicitly named or identified as pre-climax. If none exists, fall back to the stage immediately before climax. Automatically generated messages, reminders, voice sessions, side-channel posts, hidden messages, and background artifacts do not count among the four ordinary replies.

This rule supports a natural near-term restart without keeping one endless post-climax cycle alive.

---

## 2. Hidden control protocol

Primary actions:

```xml
<intimacy_control action="start" keys="context-a|context-b"/>
<intimacy_control action="advance"/>
<intimacy_control action="stop"/>
```

Compatibility actions that an upgraded parser may continue to accept:

```xml
<intimacy_control action="hold"/>
<intimacy_control action="continue"/>
```

| Action | Current meaning |
| --- | --- |
| `start` | Start a new flow after the visible reply has already crossed the activation threshold. |
| `advance` | After the current stage minimum is satisfied, move to the next stage. Advancing the final non-climax stage ends the flow. |
| `stop` | End the entire flow immediately. |
| `hold` | Legacy compatibility. Equivalent to the default behavior of remaining in the current stage. |
| `continue` | Legacy compatibility. It no longer creates another cycle; it is treated as remaining in the current stage. |

While a flow is active, absence of a stage action means **stay in the current stage**. Reaching a minimum never advances automatically.

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
<intimacy_control action="advance"/>
<scheduler_control delay_min="30"/>
```

Arbitrary visible text after the intimacy tag invalidates the control. A supported scheduler or exclusive external-flow suffix may be allowed by the parser, but the control must never become visible.

These controls are small machine-readable decisions. They do not represent private reasoning.

---

## 3. Current continuous state machine

The flow is not a hard-coded list of stages. A preset owns an ordered stage plan with up to 20 stages. Each stage has its own minimum of 1–99 ordinary replies.

Conceptually:

```text
inactive
  └─ explicit start after genuine visible activation
       └─ first configured stage
            ├─ below minimum: stay and increment
            ├─ minimum reached, no advance: stay and increment
            ├─ explicit advance: next configured stage
            └─ explicit stop or scene boundary: inactive

semantic climax stage
  └─ once its minimum reply completes: inactive
```

If recent-climax re-entry applies, replace `first configured stage` with the configured pre-climax stage.

### Stage rules

- Every stage stays active by default.
- Before the minimum is reached, even a premature `advance` is ignored and the stage counter increments normally.
- At or after the minimum, only explicit `advance` moves to the next stage.
- A semantic climax stage is terminal. Once its minimum is completed, the flow ends mechanically.
- There is no post-climax review round and no same-flow repeat cycle.
- If a custom flow has no semantic climax stage, explicit `advance` from its final stage ends the flow.
- `stop` cancels the entire flow. It is not a shortcut for skipping a stage while pretending the flow continued.

### Pre-climax decision contract

When the next stage is semantic climax, the assistant may emit `advance` only when the current continuous scene contains a sufficiently clear, observable verbal or behavioral signal. A satisfied minimum, rising intensity, or the assistant's unilateral preference is not enough.

This is a model decision contract supplied in the hidden prompt. The deterministic state machine verifies the minimum and executes a valid trailing `advance`; it does not independently infer the meaning of the visible conversation.

### Scene boundary and withdrawal

The flow belongs to one uninterrupted scene.

- A user stop, pause, refusal, or boundary change overrides every configured minimum immediately.
- The assistant may also use `stop` after recognizing a mistaken start or deciding that continuation is no longer suitable.
- Leaving the scene, returning to ordinary activity, or postponing continuation until later requires `stop`.
- If the same scene continues but the assistant does not explicitly choose to advance, the stage simply remains active.

### Ineligible paths

Only ordinary user-triggered chat replies consume or advance a guide. These paths do not:

- automatically generated messages;
- reminders and scheduled events;
- voice or other exclusive external sessions;
- posts, comments, and side-channel interactions;
- background jobs;
- unrelated system events.

If an ordinary reply chooses an exclusive external path instead of completing as chat, release its consumed guide back to `pending` so the next eligible ordinary reply can use the same snapshot.

---

## 4. Configuration schema

The normalized configuration is schema version 3 and supports up to 20 complete presets.

```js
{
  schemaVersion: 3,
  enabled: true,
  activePresetId: "preset-a",

  presets: [
    {
      id: "preset-a",
      name: "Preset A",

      flow: {
        enabled: true,
        stages: [
          { id: "stage-a", name: "Stage A", minTurns: 1 },
          { id: "stage-b", name: "Stage B", minTurns: 3 },
          { id: "pre-terminal", name: "Pre-terminal", minTurns: 1 },
          { id: "climax", name: "Terminal", minTurns: 1 }
        ]
      },

      guidancePrompt: "Treat selected entries as private creative anchors.",

      pools: [
        {
          id: "pool-a",
          name: "Pool A",
          enabled: true,
          drawMode: "turn", // "turn" or legacy-named "cycle"
          drawCount: 2,
          entries: [
            { id: "entry-a", text: "anchor-a", enabled: true }
          ]
        }
      ],

      cues: [
        {
          id: "cue-a",
          key: "stage-a-key",
          kind: "stage", // "stage", "place", or "other"
          flowRole: "stage-a", // "none" or a stage ID in this preset
          description: "Optional generic description",
          enabled: true,
          poolIds: ["pool-a"]
        }
      ]
    }
  ]
}
```

For backward compatibility, an implementation may also project the active preset's `flow`, `guidancePrompt`, `pools`, and `cues` onto the configuration root. The preset array remains the canonical owner of complete preset content.

### Preset semantics

- Exactly one preset is active for future starts.
- Creating a preset copies the entire currently selected preset.
- A started flow records its launch preset ID and name.
- Switching the active preset affects only later starts.
- The active flow keeps resolving cues, pools, and the editable guidance prompt from the preset identified at launch.
- The stage order, labels, and minimums are copied into the flow snapshot, so later stage-plan edits cannot reroute an in-flight turn.
- Prevent deletion of presets that might still be referenced by active flows, or implement an explicit archival strategy.

### Normalization limits

- maximum 20 presets;
- maximum 20 stages per preset;
- stage minimum clamped to 1–99;
- maximum 80 pools per preset;
- maximum 80 cue words per preset;
- maximum 160 entries per pool;
- maximum 1200 characters per entry;
- maximum 6000 characters in the editable guidance prompt;
- maximum 4 context keys per control;
- maximum 12 selected entries per composite guide;
- maximum 8000 characters of selected private guidance;
- draw count clamped to 1–5 per pool.

Normalize IDs, trim whitespace, deduplicate cue keys case-insensitively, discard invalid pool references, and neutralize `<` and `>` before inserting user-configured text into private XML-like prompt blocks.

Treat semantic climax as terminal during normalization. Remove configured stages after it and remove cues that target discarded post-climax stages. If localized names are used to infer semantic roles, keep that inference narrow and deterministic; explicit stable IDs are preferable.

---

## 5. Pool selection and determinism

Pool entries should be compact creative anchors, not complete private conversations or long scripts. The examples in a public specification must always remain placeholders.

Selection algorithm:

1. Resolve only enabled cue keys by normalized exact match.
2. Keep at most four non-stage context keys.
3. Add the cue associated with the current stage.
4. Collect enabled pools from context cues and the stage cue.
5. Deduplicate pool IDs in first-seen order.
6. Ignore disabled pools and disabled or empty entries.
7. Draw entries without replacement inside each pool.
8. Cap the composite snapshot at 12 entries and 8000 characters.

### Per-turn pools

For a normal changing pool:

```text
seed = sourceAssistantMessageId + NUL + poolId + NUL + drawIndex
selectedIndex = stableHash(seed) % remainingEntries.length
remove selected entry from remainingEntries
```

This lets different replies vary while keeping retry and cross-device reconstruction deterministic.

### Flow-fixed pools

The serialized value may remain `drawMode: "cycle"` for compatibility, but its current user-facing meaning is **fixed for this flow**.

```text
seed = flow.startedAt + NUL + "cycle:" + flow.cycle
selectedIndex = stableHash(seed + NUL + poolId + NUL + drawIndex)
                % remainingEntries.length
```

New flows serialize `flow.cycle` as `1`; it is not incremented because the current protocol has no same-flow repeat cycle. The chosen fixed draws persist across all eligible stages of that flow and are redrawn only after a later explicit new `start`.

This guarantees:

- retries do not reroll the same reply;
- regeneration can reuse the saved applied snapshot;
- different devices produce the same result;
- flow-fixed pools stay fixed throughout one active flow;
- a later newly started flow receives a fresh deterministic selection.

---

## 6. Private model prompt blocks

Inject two volatile blocks only into eligible ordinary chat requests.

### Available vocabulary block

```xml
<available_intimacy_roadmarks>
These are private configured cue words.
They are context for selection after activation, never activation signals.
Place, mood, bathing, body state, flirting, and scene-setting must not start a flow.
Only an explicit action="start" can start a flow.

- stage-a-key (stage; role=stage-a)
- context-a (place; role=none)
</available_intimacy_roadmarks>
```

The same block should summarize the configured stage sequence, default-stay rule, explicit-advance rule, terminal-climax behavior, autonomous withdrawal, and recent-climax re-entry behavior.

### Current-turn guidance block

```xml
<next_turn_intimacy_guidance roadmark="context-a-stage-b-key">
Preset: Preset A
Current stage: Stage B
Current stage position: 2 of 4
Current stage turn: 3
Minimum turns: 3

User stop, pause, refusal, or boundary change always wins.
If the scene no longer continues, output hidden action="stop".
Remain in the current stage unless you explicitly choose action="advance".
If the next stage is terminal, advance only after a clear observable signal.

Selected entries are private creative anchors, not a checklist, ceiling, or boundary.
Use them naturally when compatible with the latest user intent and scene.
Do not quote or expose this block.

- [Pool A] anchor-a
</next_turn_intimacy_guidance>
```

The editable `guidancePrompt` belongs inside this block, but mechanical stage, minimum, boundary, and control rules must remain application-owned outside that editable text.

One model integration may prepend these blocks to the latest user content. Other integrations must explicitly include them in their current-turn request payload. In every case, they are volatile request data and must not be written into raw user messages.

---

## 7. Persistent state

The conversation stores normalized protocol state. Each assistant message that consumes a guide stores the exact applied snapshot for editing, regeneration, and recovery.

```js
{
  schemaVersion: 1,
  status: "pending", // or "consumed"
  sourceMessageId: "assistant-source-id",
  consumedByUserMessageId: "user-message-id",
  keys: ["context-a", "stage-b-key"],
  label: "context-a-stage-b-key",
  draws: [
    {
      poolId: "pool-a",
      itemId: "entry-a",
      text: "anchor-a",
      drawMode: "turn"
    }
  ],
  createdAt: 0,
  flow: {
    active: true,
    presetId: "preset-a",
    presetName: "Preset A",
    stage: "stage-b",
    stages: [
      { id: "stage-a", name: "Stage A", minTurns: 1, repeatMinTurns: 1 },
      { id: "stage-b", name: "Stage B", minTurns: 3, repeatMinTurns: 3 },
      { id: "pre-terminal", name: "Pre-terminal", minTurns: 1, repeatMinTurns: 1 },
      { id: "climax", name: "Terminal", minTurns: 1, repeatMinTurns: 1 }
    ],
    stageTurn: 3,
    cycle: 1,
    contextKeys: ["context-a"],
    fixedDraws: [],
    startedAt: 0
  }
}
```

`cycle`, `repeatMinTurns`, and an optional `repeatFromStageId` may remain in normalized snapshots solely to recover older in-flight state. New flows do not loop or increment a cycle.

### Lifecycle

1. Create a `pending` guide after a valid start or completed eligible reply.
2. Bind it to exactly one triggering user message.
3. Mark it `consumed` while generating that reply.
4. Save the exact applied snapshot on the assistant message.
5. Parse the assistant's hidden action and calculate the next guide.
6. Commit the transition atomically as `expected -> next`.

The atomic transition prevents stale devices, retries, queued replies, and delayed state deltas from double-consuming or resurrecting an older guide.

### Editing, regeneration, and new windows

- Editing a consumed user message restores the immutable applied snapshot used by its original assistant reply, releases it to `pending`, and binds it to the replacement user message.
- Regenerating an assistant reply is replacement of the same turn, not a new intimacy turn. Reuse the saved applied snapshot and re-anchor the resulting pending guide to the replacement assistant message without rerolling draws or changing the stage counter.
- Deleting the source of a derived guide must invalidate or explicitly re-anchor that guide as part of the same authoritative operation.
- Opening a normal new conversation window may copy the current guide as `pending` so the next eligible reply continues from the same state. The original conversation keeps its own state.
- Incognito or private windows remain isolated and must not upload pending flow state.
- Failed regeneration or interrupted delivery must not silently clear the flow.

---

## 8. Privacy boundaries

The feature must never place private guidance into:

- raw user messages;
- visible assistant messages;
- long-term memory;
- handoff summaries;
- compaction summaries;
- generic push payloads;
- ordinary logs;
- public documentation, examples, fixtures, or screenshots.

Normal conversations may synchronize the minimum flow metadata and the user's configuration when the product explicitly supports that synchronization. Private or incognito conversations may use the feature locally but must not upload their pending flow state or include it in memory pipelines.

Logs should record only safe metadata such as transition success or failure and opaque message identifiers. Avoid logging preset names, stage names, pool text, control-tag contents, credentials, or conversation text.

Before publishing a replication guide:

1. Replace all real preset, stage, cue, pool, and entry text with placeholders.
2. Remove people, product-instance names, domains, IP addresses, file paths, account identifiers, timestamps, and message IDs from production.
3. Do not copy private prompts or conversation excerpts, even if they seem generic.
4. Scan the final diff for secrets, URLs, email addresses, filesystem paths, and high-entropy tokens.
5. Review only the staged public diff before pushing.

---

## 9. Parser and sanitizer pseudocode

```js
function parseTrailingControl(raw) {
  const value = removeOnlyOrphanedSupportedSuffix(raw);
  const matches = findAllSelfClosingIntimacyTags(value);
  const candidate = lastMatchWithActionOrKeys(matches);

  if (!candidate) return null;
  if (!isSupportedSuffix(value.slice(candidate.end))) return null;

  return {
    action: oneOf(candidate.action, [
      "start", "advance", "stop", "hold", "continue"
    ]),
    keys: splitAndNormalizeKeys(candidate.keys).slice(0, 4)
  };
}

function visibleText(raw) {
  return stripAllProtocolControlsAndOrphanedControlTails(raw).trimEnd();
}
```

Important parser rules:

- Never default a key-only control to `start`.
- A control in the middle of ordinary prose is invalid.
- A partial `<intimacy_control...` suffix must be hidden during streaming.
- Sanitization must happen independently at stream, final-message, state-save, and recovery boundaries.
- Scheduler and exclusive external-flow markers must not accidentally advance the state.
- Compatibility actions must not restore the removed post-climax loop.

---

## 10. Transition pseudocode

```js
function nextGuide(config, rawReply, appliedGuide, sourceAssistantId, now, recentMessages) {
  const control = parseTrailingControl(rawReply);

  if (appliedGuide?.flow) {
    return advanceFlow(config, appliedGuide, control, sourceAssistantId, now);
  }

  if (!control || control.action !== "start") return null;

  return startFlow(
    config,
    control.keys,
    sourceAssistantId,
    now,
    recentMessages
  );
}

function startFlow(config, requestedKeys, sourceAssistantId, now, recentMessages) {
  const preset = activePreset(config);
  const stages = immutableNormalizedStageSnapshot(preset.flow.stages);
  const startStage = recentEligibleClimax(recentMessages, 4)
    ? preClimaxStage(stages)
    : stages[0];

  const flow = {
    active: true,
    presetId: preset.id,
    presetName: preset.name,
    stage: startStage.id,
    stages,
    stageTurn: 1,
    cycle: 1,
    contextKeys: validContextKeys(preset, requestedKeys),
    fixedDraws: [],
    startedAt: now
  };

  return buildGuide(preset, flow, sourceAssistantId, now);
}

function advanceFlow(config, applied, control, sourceAssistantId, now) {
  const flow = clone(applied.flow);
  const preset = presetById(config, flow.presetId);
  const action = control?.action || "";

  if (action === "stop") return null;

  const updatedContext = validContextKeys(preset, control?.keys);
  if (updatedContext.length) flow.contextKeys = updatedContext;

  const index = flow.stages.findIndex(stage => stage.id === flow.stage);
  const stage = flow.stages[index];
  const minimum = stage.minTurns;
  const isLast = index === flow.stages.length - 1;
  const isClimax = isSemanticClimax(stage, flow.stages);

  if (flow.stageTurn < minimum) {
    flow.stageTurn += 1;
  } else if (isClimax) {
    return null;
  } else if (action === "advance" && !isLast) {
    flow.stage = flow.stages[index + 1].id;
    flow.stageTurn = 1;
  } else if (action === "advance") {
    return null;
  } else {
    flow.stageTurn += 1;
  }

  return buildGuide(preset, flow, sourceAssistantId, now);
}
```

The model prompt must separately enforce the clear-signal contract before entering climax. The state transition function should remain deterministic and should not perform a second free-form interpretation of the conversation.

---

## 11. UI and editor requirements

The configuration editor should provide:

- a master enable switch that pauses without deleting configuration;
- a compact active-preset selector;
- up to 20 complete named presets;
- copy-current-preset creation;
- composition-safe preset-name editing for input methods such as Chinese Pinyin;
- an ordered stage editor with add, remove, rename, reorder, and 1–99 minimum controls;
- cue-word cards with type and arbitrary stage-role selection;
- pool cards with enabled state, draw count, draw mode, and entries;
- explicit per-turn / flow-fixed selection per pool;
- an editable private draw-guidance prompt with restore-default support;
- a test-draw preview that never starts or mutates a real flow;
- a visible warning when a formal stage has no enabled role cue or usable pool;
- a live-flow summary that shows safe stage metadata without exposing private pool entries unnecessarily;
- draft state separate from saved state;
- explicit save confirmation only after the server returns the normalized configuration;
- unsaved-change confirmation on exit.

Do not infer draw mode from pool names. Existing pools remain per-turn unless the user explicitly changes them.

For touch interfaces, reorder only from a dedicated drag handle. Do not disable page scrolling or text input across the entire editor.

---

## 12. Migration from the earlier protocol

An implementation upgrading from the original fixed-cycle specification should migrate conservatively.

### Configuration

1. Wrap the former flat `flow`, `pools`, `cues`, and guidance fields in one complete preset.
2. Set that preset as `activePresetId`.
3. Convert fixed process counters into an ordered `stages` array.
4. Clamp every stage minimum to 1–99.
5. Treat semantic climax as terminal and discard obsolete post-climax review stages and their cue bindings.
6. Preserve pool IDs, entry IDs, cue IDs, draw modes, and enabled states where valid.
7. Keep legacy-named `drawMode: "cycle"` values, but present them as flow-fixed in the UI.

### In-flight state

1. Normalize old snapshots on read instead of clearing them.
2. Preserve the current stage, stage turn, context, fixed draws, timestamps, and source/consumer bindings.
3. Retain legacy `cycle`, repeat-minimum, or repeat-entry fields only as needed to finish recovering old snapshots.
4. Do not create any new review or repeat cycle after the migrated flow reaches climax.
5. Save the normalized form through the same authoritative atomic state path.

### Model protocol

1. Add `advance` to the accepted action set.
2. Change default progression to stay-in-stage.
3. Stop requesting `continue` after climax.
4. Add the clear-signal instruction for pre-climax advancement.
5. Add the recent-four-ordinary-replies re-entry rule.
6. Ensure older `hold` and `continue` outputs cannot revive removed behavior.

---

## 13. Verification checklist

At minimum, test:

- context alone does not start a flow;
- explicit `start` does start one;
- the transition reply is not counted as a formal-stage reply;
- key-only controls never imply start;
- every configured stage minimum is enforced;
- reaching a minimum without `advance` stays in the same stage;
- premature `advance` cannot bypass a minimum;
- valid `advance` moves exactly one stage;
- `stop` works in every formal stage;
- user boundary changes override minimums;
- same-scene default stay and scene-boundary stop are both represented in the model prompt;
- semantic climax ends immediately after its minimum, with no review round;
- `hold` and `continue` cannot create a post-climax cycle;
- a recent saved climax snapshot restarts at pre-climax;
- a climax older than the latest four eligible ordinary replies restarts at the first stage;
- ineligible automated or external replies do not displace the four-reply history window;
- stages after semantic climax are removed during normalization;
- cues targeting removed stages are removed;
- a custom flow without semantic climax ends only through explicit advance from its final stage;
- switching the active preset does not reroute an already active flow;
- the in-flight stage-plan snapshot remains immutable;
- flow-fixed pools remain constant throughout one flow;
- a later explicit new start rerolls flow-fixed pools deterministically;
- per-turn pools vary between source message IDs but remain stable on retry;
- shared pools are drawn only once per composite guide;
- composite draw and text caps are enforced;
- editable guidance text cannot close the private prompt block;
- complete and partial controls stay invisible at every output boundary;
- scheduler and exclusive external-flow paths do not advance the flow;
- editing a consumed user message reuses the original applied snapshot;
- assistant regeneration preserves stage, counters, context, and draws;
- failed regeneration does not clear the pending flow;
- an ordinary new window can carry a released pending snapshot without clearing the source window;
- private windows do not upload pending flow state;
- stale state cannot overwrite a newer atomic transition;
- private guidance never enters memory, handoff, compaction, logs, fixtures, or public documentation;
- the final public diff contains no real names, presets, pool text, conversation excerpts, domains, addresses, paths, credentials, or production identifiers.

The central invariant is:

> Only an explicit `action="start"` following genuine visible activation can start a flow. Every stage then stays active by default until an explicit, valid `advance`; semantic climax ends the flow, private pools provide deterministic creative anchors, and every internal control remains outside the user-visible and memory boundaries.
