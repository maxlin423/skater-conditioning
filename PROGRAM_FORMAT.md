# Training Program Format

This is the contract between a program *author* — a person or an agent — and the
off-ice training app. A program is one JSON document. The app imports it as
pasted text, validates it, and runs it.

`program.schema.json` is the machine-checkable version. This document is the one
to reason from: it explains what the fields are *for*, and encodes judgment the
schema can't express.

---

## The shape of a program

```json
{
  "schemaVersion": 1,
  "id": "atg-jump",
  "name": "Off-Ice Jump Training",
  "version": "1.0.0",
  "coreLibrary": 1,
  "disclaimer": "…",
  "phases": [ … ]
}
```

Those seven fields are required. Everything else is optional and falls back to an
app default.

---

## Exercises: three ways to get one

**1. Reference the core library.** The app ships a versioned library of
bodyweight exercises. Reference them with a `core:` prefix:

```json
"plan": ["core:warmup", "core:calfStraight", "core:cooldown"]
```

**2. Reference a core exercise and change some fields.** Put the changes in
`overrides`. Everything you don't mention comes from the library.

```json
"overrides": {
  "core:split": { "base": 6, "sets": 2 }
}
```

`name` and `mode` can't be overridden — the app's session UI is built around
them staying put. If you need a genuinely different movement, define a new one.

**3. Define your own.** Keys go in `exercises`, without a prefix:

```json
"exercises": {
  "box-step-up": {
    "name": "Box step-up",
    "mode": "reps", "sets": 3, "base": 8, "step": 1, "max": 15,
    "perSide": true,
    "why": "…", "how": ["…"], "tips": ["…"],
    "equipment": [{ "id": "jump-box", "label": "Jump box", "params": { "height_in": 12 } }],
    "alternatives": ["core:split"]
  }
}
```

### Core library, version 1

| Reference | Movement | Mode |
|---|---|---|
| `core:warmup` | Warm-up sequence | hold |
| `core:tib` | Tibialis raises | reps |
| `core:calfStraight` | Calf raises, straight leg | reps |
| `core:calfBent` | Calf raises, bent knee | reps |
| `core:bridge` | Single-leg glute bridge | reps |
| `core:wallsit` | Wall sit | hold |
| `core:split` | ATG split squat | reps |
| `core:backwalk` | Backward walks | reps |
| `core:minijump` | Mini jumps | reps, `jump` |
| `core:balance` | Single-leg balance with arms | hold, `balanceScore` |
| `core:hops` | Forward hops | reps, `jump` |
| `core:rotate` | Standing rotations | reps |
| `core:lateral` | Lateral lunges | reps |
| `core:stepdown` | Controlled step-downs | reps |
| `core:cooldown` | Cool-down sequence | hold |

All of them are bodyweight and need no equipment.

`coreLibrary` pins which version you resolved against. It's fixed at install, so
a later app update can never silently change an installed program. If you author
against a library version the target app doesn't have, the import fails with a
clear message rather than guessing.

### Exercise fields

| Field | Meaning |
|---|---|
| `mode` | `reps` counts repetitions, `hold` counts seconds |
| `sets` | 1–6 |
| `base` | Starting target, and the floor progression never goes below |
| `step` | How far a target moves per progression event. `0` pins it (warm-ups, cool-downs) |
| `max` | Progression ceiling. Must be ≥ `base` |
| `perSide` | Target applies to each side separately |
| `jump` | Plyometric. Skipped entirely during a heavy ice week |
| `balanceScore` | `hold` only. Attaches a gyroscope steadiness score when measure mode is on |
| `why` | One or two sentences on what it's for. The skater reads this |
| `how` | Ordered setup and execution steps |
| `tips` | Form cues, common mistakes, how to make it harder or easier |

---

## Equipment

**There is no fixed list of equipment.** Use whatever id and label describe the
thing. The app renders the label and any params next to the exercise; it never
validates against an enum.

```json
"equipment": [
  { "id": "ankle-weight", "label": "Ankle weights", "params": { "weight_lb": 2 } },
  { "id": "slant-board",  "label": "Slant board", "optional": true }
]
```

`params` is free-form — height, weight, incline, length, whatever the setup
needs. Mark `optional: true` when the exercise works fine without it and the
equipment only adds load or variety.

### Always give alternatives

Any exercise with non-optional equipment should carry `alternatives`. The skater
taps to swap, and **the swap is recorded and reported back to you** in the
export. If you prescribe something she can't do and give her no way out, that
exercise silently becomes a hole in her session.

```json
"alternatives": ["core:calfStraight", "my-tempo-raise"]
```

### Using equipment well

Equipment earns its place two ways: it loads a movement that bodyweight has
outgrown, or it makes a session less boring. Both are legitimate — a program the
skater has stopped enjoying is one she stops doing.

What it should not do is add complexity for its own sake. If a weighted vest and
bodyweight produce the same training effect this week, use bodyweight.

---

## Phases and days

A program is an ordered list of phases. Each phase holds one or more day
variants.

Single-day phase — use the `plan` shorthand:

```json
{ "id": "foundation", "name": "Foundation", "weeks": "Weeks 1–3",
  "blurb": "…", "unlockAfter": 9,
  "plan": ["core:warmup", "core:tib", "core:cooldown"] }
```

Multi-day phase — use `days`:

```json
{ "id": "power", "name": "Power", "unlockAfter": 9,
  "rotate": "sequential",
  "days": [
    { "id": "legs", "name": "Legs", "plan": ["core:warmup", "core:split", "core:cooldown"] },
    { "id": "spring", "name": "Spring", "plan": ["core:warmup", "core:minijump", "core:cooldown"] }
  ] }
```

Use one or the other, never both. `rotate: "sequential"` advances a day each
session; `"choose"` lets the skater pick.

`unlockAfter` is how many sessions in this phase before the next one unlocks.
**Advancing is always a manual choice** — this only lights up the button. Never
build a program that advances anyone automatically.

`weeks` is a display label with no logic behind it.

---

## Strategies

Progression, unlocking, and volume scaling are **named strategies with
parameters**. A program is data; it cannot carry executable code, and the app
will never evaluate an expression from one.

### `progression`

```json
"progression": { "strategy": "linear-step", "params": { "shortfall": 0.7 } }
```

| Strategy | Behaviour |
|---|---|
| `linear-step` | Every set completed at target, session not marked tough, no pain → target rises by `step`. Marked tough, or any set below `shortfall` × target → falls by `step`. Clamped to `base`/`max`. **This is the default.** |
| `double-progression` | Reps climb through `repRange` before difficulty steps up and reps reset to the bottom |
| `hold-until-clean` | Requires `cleanSessions` consecutive clean sessions before a target rises. Conservative — good coming back from pain |
| `none` | Targets never move |

### `unlock`

```json
"unlock": { "strategy": "sessions-in-phase", "params": { "n": 9 } }
```

`sessions-in-phase`, `weeks-in-phase`, or `manual` (always unlocked). A phase's
own `unlockAfter` overrides `n` for that phase.

### `volumeScaling`

Optional, defaults to `none`. Scales prescribed volume from tracked metrics:

```json
"volumeScaling": {
  "strategy": "age-banded",
  "params": { "bands": [ { "maxAge": 11, "setsMultiplier": 0.8, "jumpRepsMax": 6 } ] }
}
```

`bodyweight-scaled` uses a `metric` id, a `referenceValue`, and an `exponent`.

---

## Metrics

Body metrics the program wants tracked over time. They feed `volumeScaling` and
appear in the export you read on the next revision.

```json
"metrics": [
  { "id": "height", "label": "Height", "unit": "cm", "cadence": "monthly" },
  { "id": "bodyweight", "label": "Weight", "unit": "kg", "cadence": "monthly" }
]
```

`cadence` is `weekly`, `monthly`, or `quarterly` — it controls how often the app
*invites* an entry. The app never nags, never blocks a session on a missing
measurement, and never sets a goal or target value for any body metric.

Ask for the slowest cadence that answers your question. Height monthly is enough
to catch a growth spurt; weekly measurement of anything that moves this slowly
produces noise the skater has to go collect.

---

## Checklist and badges

`checklist` replaces the app's default readiness list when present. Plain
statements the skater can honestly tick or not.

`badges` are program-specific. They're namespaced `programId:badgeId` when
awarded and evaluated against **this program's sessions only**. The app's own
career badges (session milestones, weeks landed) are separate and always there,
counting across every program — switching programs never costs her a badge.

```json
"badges": [
  { "id": "deep-split", "ic": "◆", "nm": "Deep split squat",
    "rule": { "type": "exerciseTarget", "params": { "ex": "core:split", "value": 12 } } }
]
```

Rule types: `sessionCount`, `weeksLanded`, `phaseReached`, `checklistComplete`,
`exerciseTarget`, `exerciseSessions`, `jumpCount`, `jumpImprovement`,
`painFreeSessions`.

---

## What the app rejects

Validation is not a formality — a generated program is untrusted input.

- **Missing or too-short `disclaimer`.** Required, shown on first activation.
- **Volume over ceiling.** Max 6 sets, 12 exercises per day, 100 reps, 600
  second holds, 7 days per phase, 12 phases. Plyometrics (`jump: true`) are
  capped harder: 20 reps, 4 sets.
- **Markup in any string.** All copy is stripped of tags on import.
- **Unresolvable references.** Every id in every `plan`, every `alternatives`
  entry, and every `overrides` key must resolve.
- **Unknown strategy or badge rule names.**
- **A `coreLibrary` version this app build doesn't carry.**

---

## Revisions

To revise a program the skater is already running, **keep the same `id`** and
bump `version`:

```json
{ "id": "atg-jump", "version": "1.1.0", "revisionOf": "1.0.0",
  "revisionNote": "Split squats stalled for five weeks — dropped back to 8 and added a slant board variation for variety." }
```

The app recognises it as a revision and carries progress forward:

- Exercises that survive keep their current adaptive targets
- New exercises start at their `base`
- Removed exercises are archived, not deleted — history stays readable
- Phase position and session count continue uninterrupted

To deliberately reset a stalled exercise, say so explicitly:

```json
"overrides": { "core:split": { "base": 8, "resetTarget": true } }
```

A new `id` installs a **second, separate program** and starts its progress from
nothing. Only do that when it genuinely is a different program.

---

## The training export

This is the other half of the loop — what the app hands *back*. Settings →
**Copy training report** produces it. It's the input for revising a program.

```json
{
  "kind": "training-export", "exportVersion": 1, "targetSchema": 1,
  "skater": { "ageYears": 11.9, "metrics": { "height": {…}, "bodyweight": {…} } },
  "equipment": "jump rope, 12 and 18 inch boxes, 2lb ankle weights, slant board",
  "program": { "id": "atg-jump", "version": "1.1.0", "phase": "power",
               "sessionsInPhase": 8, "unlockMet": false },
  "adherence": { "weeklyGoal": 3, "last8Weeks": [2,3,3,0,2,3,1,2], "painSessions": 2 },
  "exercises": [ … ],
  "signals": { … },
  "checklist": [ … ], "jumps": [ … ], "sessionNotes": [ … ]
}
```

Each entry in `exercises` carries `target`/`base`/`max`, `atCap`, `neverMoved`,
`daysSinceIncrease`, `appearances`, `skipped`, `weeksInProgram`, `effortMix`,
`funMix`, `painDates`, `swappedInFor` / `swappedAwayTo`, `recentSets` and any
notes she wrote.

### Signals

Derived conclusions, precomputed so they can't be missed:

| Signal | Means |
|---|---|
| `stalled` | Target hasn't risen in 21+ days over 5+ sessions. **The most actionable one.** |
| `atCap` | Sitting at `max` — the ceiling needs raising, or the exercise needs replacing |
| `consistentlyHard` | Rated hard repeatedly — likely prescribed too heavy |
| `tooEasy` | Rated easy repeatedly and not at cap — under-prescribed |
| `dullest` | Rated boring more than fun. **Variety, not intensity, is the fix** |
| `painLinked` | Pain flagged against this exercise, with dates. Never re-prescribe without changing something |
| `frequentlySkipped` | Skipped a third of the time or more — usually equipment she doesn't have, or avoidance |
| `alternativePreferred` | She keeps swapping it out for the same substitute. **That substitute is the exercise she actually needs** |
| `unchangedWeeks` | How long each exercise has been in the program, for judging when to rotate |

Exercises with `step: 0` are marked `fixed` and are excluded from `stalled` and
`atCap` — a warm-up was never going to progress.

### Revising from an export

1. Read `signals` first; they're the summary of everything below them.
2. Keep the same `id`, bump `version`, set `revisionOf` and write a
   `revisionNote` in plain language — she sees it on import.
3. Address stalls by reducing, not pushing: lower the target with
   `resetTarget`, or swap the movement.
4. Anything in `painLinked` changes or comes out. Never leave it as it was.
5. Prescribe only against what `equipment` says she has, and keep
   `alternatives` on anything that needs kit.
6. Rotate what's in `dullest` even when it's working physically.

---

## Writing the copy

The skater reads `why`, `how`, and `tips` alone, on a phone, usually right
before doing the movement.

- Write to her, not about her. Short sentences.
- `why` should answer "why am I doing this" in terms of skating, not physiology.
- `how` is what to do, in order. `tips` is what usually goes wrong.
- Encouraging, never nagging. No guilt about gaps, no pressure to train more
  than the weekly goal.
- Pain is never something to push through, and no copy should imply otherwise.
  "Burning in the shin is the point. Sharp pain is not." is the register.
- Don't reference body weight, size, or appearance in exercise copy.

---

## Worked example

A minimal but complete two-phase program using core exercises, one custom
equipment exercise with an alternative, and a program badge.

```json
{
  "schemaVersion": 1,
  "id": "summer-base",
  "name": "Summer Base Block",
  "version": "1.0.0",
  "coreLibrary": 1,
  "author": "training-program-author",
  "summary": "Six-week off-season base block. Ankle and calf focus, low plyometric volume.",
  "basedOn": ["Ben Patrick / Knees Over Toes — gradual loading, full range of motion"],
  "disclaimer": "General training guidance, not medical or coaching advice. Volume and jump work should be cleared with a skating coach or a trainer who works with young athletes, particularly during a growth spurt.",
  "weeklyGoal": 3,
  "audience": { "minAge": 9, "maxAge": 14 },

  "progression": { "strategy": "linear-step", "params": { "shortfall": 0.7 } },
  "unlock": { "strategy": "sessions-in-phase", "params": { "n": 9 } },

  "metrics": [
    { "id": "height", "label": "Height", "unit": "cm", "cadence": "monthly" }
  ],

  "overrides": {
    "core:split": { "base": 6 }
  },

  "exercises": {
    "box-stepup": {
      "name": "Box step-up",
      "mode": "reps", "sets": 3, "base": 8, "step": 1, "max": 15,
      "perSide": true,
      "why": "Teaches one leg to lift your whole body, which is what a takeoff asks for.",
      "how": [
        "Stand facing a low box, about knee height or lower.",
        "Step up with one foot, driving through the whole foot.",
        "Stand tall at the top, then lower slowly back down."
      ],
      "tips": [
        "Don't push off the back foot. The working leg does everything.",
        "Lower slowly. That half is the point.",
        "Lower box if the knee caves inward."
      ],
      "equipment": [
        { "id": "jump-box", "label": "Jump box", "params": { "height_in": 12 } }
      ],
      "alternatives": ["core:split"]
    }
  },

  "phases": [
    {
      "id": "base",
      "name": "Base",
      "weeks": "Weeks 1–3",
      "blurb": "Ankles and calves. Nothing explosive yet.",
      "unlockAfter": 9,
      "plan": ["core:warmup", "core:tib", "core:calfStraight", "core:calfBent", "core:wallsit", "core:cooldown"]
    },
    {
      "id": "build",
      "name": "Build",
      "weeks": "Weeks 4–6",
      "blurb": "Single-leg loading gets added on top of the base work.",
      "rotate": "sequential",
      "days": [
        {
          "id": "strength",
          "name": "Strength",
          "plan": ["core:warmup", "core:tib", "core:split", "box-stepup", "core:cooldown"]
        },
        {
          "id": "control",
          "name": "Control",
          "plan": ["core:warmup", "core:calfStraight", "core:backwalk", "core:balance", "core:cooldown"]
        }
      ]
    }
  ],

  "checklist": [
    { "id": "c1", "t": "20 single-leg calf raises on each leg, full height" },
    { "id": "c2", "t": "45-second wall sit without sliding down" },
    { "id": "c3", "t": "No ankle or knee pain during any of it" }
  ],

  "badges": [
    { "id": "base-done", "ic": "◈", "nm": "Base block landed",
      "rule": { "type": "sessionCount", "params": { "n": 18 } } }
  ]
}
```
