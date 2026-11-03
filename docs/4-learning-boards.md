# How Demp learns a job board

Job applications live on dozens of different hiring systems. A few are big and consistent (Greenhouse, Lever, Ashby). The long tail is hundreds of smaller platforms and one-off company career sites, each with its own widgets, page flows and quirks.

Demp handles them in two ways:

- **Known boards** have hand-written drivers. Every driver implements the same interface: read the form, fill one field, recognise the final Submit control, and check whether the posting is still live.
- **Every other board** goes through the **self-taught engine**, which implements that same interface. Instead of hand-written code, it uses a playbook the engine learned itself: how to get into the application, what each box needs, and how to move to the next page.

This page covers the self-taught engine, which is the part that took the most iterations.

## The core idea: learn once, replay many times

Asking an AI model to fill every form live is expensive and doesn't repeat. The same form can come out differently on two runs, so failures can't be reproduced. Demp splits the work instead:

1. A **learner** works out a board **once**, using an AI model that operates the page through a small set of guarded tools.
2. Only the steps that worked are saved as that board's playbook.
3. Every later application on that board **replays the playbook deterministically**. The expensive model isn't called at all. A cheap model is used only to write answers the profile can't supply and to look over the finished form.

In one rehearsal on a test database (2026-10-30), learning a new board cost **$0.18** of AI calls. Each application on it afterwards cost **under $0.01**, with no call to the expensive model.

## A board's life

```mermaid
stateDiagram-v2
    [*] --> Unlearned: a matched posting on an unknown board
    Unlearned --> Learning: learner claims the board
    Learning --> Learned: one clean, verified sample
    Learning --> Unlearned: sample says nothing about the board<br/>(posting gone, AI outage), not counted
    Learning --> NeedsHelp: 3 failed samples
    Learned --> Learned: replay succeeds
    Learned --> Repairing: replay hits a structural failure
    Repairing --> Learned: fixed, waiting applications re-filled
    Repairing --> Learned: not reproduced, the person's answer was the problem
    Repairing --> Unlearned: unfixed twice for the same control
    NeedsHelp --> Unlearned: admin presses Try again
```

### 1. Queue

A matched posting on a board nobody has learned yet puts one **learn this board** task in the work queue. Boards that block the most people's applications go first. A person's fill never stops to learn: it moves on, and the application waits until the board is learned.

### 2. Learn, as a practice applicant

The learner never uses a real person's profile. It applies as a **practice applicant**, a made-up person whose profile is deliberately rich enough to exercise every kind of question, in its own isolated browser. It stops before Submit, every time.

The model sees a text snapshot of the page in which every control has a short tag, and it acts only through a few tools. Rules in code decide what it may **not** do, whatever it decides:

- type a real answer itself (answers come from the profile, never from the model)
- press the final send control
- leave the application

### 3. Verify before trusting

One clean sample is enough to learn a board, but "clean" is strict:

- every required box reads back the value it was given
- every page advances
- the final Submit control is found, and not pressed
- a cheap model then looks at screenshots of the whole filled form for anything empty, wrong, or showing a validation error

The screenshot check is told **whether** each box was answered, never **what** the answer was, so no personal value reaches that call:

```ts
/** The cheap screenshot check. One look at a filled form's pictures: which boxes look empty, wrong, or show a
 *  validation message. The plan says WHETHER a box was answered, never what the answer was, so no personal
 *  value reaches this call. */
export interface FillCheckItem {
  label: string; type: string; required: boolean;
  planned: 'answered' | 'left-blank' | 'file';
  readBack?: 'verified';   // the filler's own read-back already confirmed this box
}
export type FillCheckProblem = { label: string; kind: 'empty' | 'wrong' | 'validation' | 'not-shown'; reason: string };
export type FillCheckReport = { ok: true } | { ok: false; problems: FillCheckProblem[] };
```

### 4. Failures that don't count

A sample that tells us nothing about the board isn't counted against it. That covers a posting that turned out to be closed, and an AI outage. Only real failures count, and three of those park the board for an admin. On a crash or outage the claim is released, the attempt isn't counted, and the error goes up to the caller:

```ts
} catch (e) {
  await releaseLearning(db, fam.family, owner, 'unlearned');   // never burn a sample on a crash or an offline machine
  throw e;
}
```

### 5. Replay, and deciding whose problem a failure is

When a replay can't fill a box, the important question is **whose problem it is**:

- **The board's** (structural): the control isn't there, isn't what was recorded, the click didn't take, or a new required box appeared. The fill stops, the application waits, and a **repair job** goes to the learner.
- **The person's**: their saved answer matches none of the options, or they have no document for an upload. The box is flagged for the person, and the application goes to review as normal.

That decision is one small rule:

```ts
/** A replay failure is the board's when the control is not found, is not what was recorded, the gesture does not
 *  take, the stored step left the page or would press the send button, or the box has no stored step at all.
 *  It is the person's when their saved answer matches none of the options, when they have no document for the
 *  question, or when there is nothing to type. */
export function isStructuralFailure(outcome: ReplayOutcome | null, plan: FieldPlan): boolean {
  if (plan.value == null) return false;
  if (outcome === null) return true;
  return !outcome.ok && !PERSONS_FAILURES.has(outcome.failureClass);
}
const PERSONS_FAILURES: ReadonlySet<string> = new Set(['no-option-matches', 'no-document']);
```

Getting this split right matters in both directions. Calling a person's problem structural wastes AI money relearning a board that works. Calling a board's problem the person's asks them to fix something they can't.

### 6. Repair

The learner takes the repair job, re-runs one of the waiting applications' postings as the practice applicant, and fixes **only the named control**. Every repair ends one of three ways:

| Outcome | Meaning | What happens |
|---|---|---|
| **Fixed** | The new step works and was saved | Every waiting application is re-filled |
| **Not reproduced** | The control worked fine for the practice applicant | It was the person's answer after all; the box is flagged for them |
| **Unfixed** | Still broken | Applications go to human review; a second unfixed repair of the same control sends the whole board back to be relearned |

As in v1, a playbook is invalidated **by failure, not by age**. Boards change without notice, so any fixed lifetime would be either wasteful or stale.

The repair evidence is a text snapshot with every answer, profile fact and file name replaced, plus the cheap check's report. The person's own screenshots never go to the learner.

## Working safely with several workers

The learner and the fill workers share the database, so every handoff has to survive crashes and races.

**Claiming work** is a single atomic statement. It takes the oldest open job, or one whose holder's lock has gone stale because its worker died:

```sql
UPDATE board_repairs SET state = 'claimed', lock_owner = $1, lock_at = $2, updated_at = $2
 WHERE id = (SELECT id FROM board_repairs
              WHERE state = 'open' OR (state = 'claimed' AND (lock_at IS NULL OR lock_at < $3))
              ORDER BY created_at, id
              LIMIT 1
              FOR UPDATE SKIP LOCKED)
RETURNING *;
```

**Holding work.** The claim is renewed at every model turn, screenshot check and database write. A run that loses its claim stops at once and writes nothing.

**Finishing work: applications first, then the job.** A crash between those two steps leaves a claimed job that a later run picks up again. It never leaves a closed job whose applications are still waiting.

**Forced stops give work back** instead of making it wait for a lock to expire. Only the claim's holder can release it:

```ts
export async function releaseInHand(db: Sql, state: LearnerState, owner: string, now: number): Promise<void> {
  const held = state.inHand;
  state.inHand = null;
  if (held === null) return;
  try {
    if ('family' in held) await releaseLearning(db, held.family, owner, 'unlearned');
    else await releaseRepairJob(db, held.repairId, owner, now);
  } catch (err) { console.warn('[learn] could not give back the work in hand:', msg(err)); }
}
```

**Spending limits.** Daily ceilings on AI calls and model turns pause the learner, and each attempt has its own cap on model turns.

*(Excerpts are from the real code, lightly trimmed: internal references to specs and rulings were removed; the screenshot-check comment is condensed from the file header; the SQL is shown with standard `$n` placeholders, a full column list, and without an optional filter for jobs the caller is skipping.)*

## How it got here

v2 started from a clean repository, so this engine was rebuilt rather than ported from v1, and it went through four designs before it settled. Each was replaced because of something measured.

| When | Design | Why it was replaced |
|---|---|---|
| Aug 2026 | **Helper:** ask a model *about* the page, then run its CSS-selector recipe | Its recipes had to pass a hand-written word list. In tracer runs that list rejected 10 of 10, correct ones included, and the model was never told why. No board family reached ready |
| Sep 2026 | **Agent decides "ready"**, with a repair button for the person | A check on 2026-09-30 found 45 of 77 "ready" applications had a box that hadn't taken its answer. The agent's own read-back missed boxes, and a crashed check counted as a pass. The fix: an independent check decides "ready", and a check that couldn't run never counts as one |
| Oct 19 | Triage | 136 self-taught applications stuck in the human-review list. The diagnosis was that the shape of the engine was the problem, not any one bug |
| Oct 21–27 | **Driver:** the model acts *on* the page through guarded tools; the program replays what it learned and calls the model only at the failing step | Worked, but learning ran inside each person's fill. One person's press waited 13.6 minutes behind a learning run they hadn't asked for |
| Oct 29–30 | **Separate learner with a practice applicant.** Fills can no longer learn at all; the code was removed, not switched off | Current design |
| Nov 2026 | **Learning upstream** (v3): learn tasks are rows in the shared work queue | Current design |

The thread through all of them: **the model should do as little as possible, as rarely as possible, and never be the one that decides its own work is correct.**

## What this page leaves out

- the prompts
- the format playbooks are stored in, and the vocabulary the learner is allowed to write
- the page-reading code that turns a live page into the tagged snapshot
- per-board fixes
- the rules for screening and legal questions
- browser settings

The design is all here; the months of edge cases aren't.
