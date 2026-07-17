# Token Budget Lab — Cut the Bill


Shipping fast is the cheap part; running it is where the money goes. This lab makes that visible: you'll measure, on the same TaskFlow repo you've used all day, exactly what a naive request costs versus a deliberate one — then grow the repo for real so the gap stops being a rounding error.

> **In this lab** you'll use nothing but `/context` and `/cost` — no plugin install, no new repo, no bespoke tooling. Three levers (tight payload, prompt caching, model routing), each one measured on your own screen, not read off a slide.
> This guide is written to be run solo, start to finish. If you're facilitating a room instead, everything here still works read aloud — there's nothing instructor-only held back.

---

## On this page

1. [The idea in one minute](#01--the-idea-in-one-minute)
2. [How this connects to today](#02--how-this-connects-to-today)
3. [Pre-flight](#03--pre-flight)
4. [Walkthrough — measure it, cut it](#04--walkthrough--measure-it-cut-it-30-min)
5. [Stretch — token budget at scale](#05--stretch--token-budget-at-scale-20-min-optional)
6. [Honest caveats](#06--honest-caveats)
7. [Cheat sheet](#07--cheat-sheet)

---

## 01 — The idea in one minute

In AI-assisted work, OpEx is now a token bill, and context is a line item, not a technical detail. Three levers bring that bill down, and all three are measurable on this same repo, in this same lab:

- **A tight payload** — `CLAUDE.md` plus only the files a task touches, instead of dumping the whole repo.
- **Prompt caching** — the stable parts of your context (the ones that don't change between turns) get read at a steep discount after the first turn.
- **Model routing** — heavy models for the hard thinking, cheap models for tests, review, and CI.

Nothing here needs a bigger repo or a custom tool. `/context` and `/cost` are already sitting in your session, showing you the real number instead of a slide's estimate.

## 02 — How this connects to today

- **Demo 1 (SubAgents):** you already saw model routing in action — `doc-writer` on Haiku, `code-reviewer` on the reasoning tier. This lab puts a dollar shape on why that split exists.
- **Demo 2 (Agent Teams):** the fix you make in Step 3 below (if you're running this cold) is the same PATCH truthiness bug from that demo.
- **Demo 3 (Dynamic Workflows):** a workflow fanning out *N* subagents is *N* copies of whatever payload size you choose. The stretch section in this lab measures that multiplication for real.

## 03 — Pre-flight

- [ ] Fresh Claude Code session from the TaskFlow repo root (continuing from Demos 1–2 works too).
- [ ] `npm test` green before you start.

## 04 — Walkthrough — measure it, cut it (~30 min)

### Step 1 — Measure the naive way

Point the agent at the whole repo and read the context meter:

```
Read every file in src/ and tests/, then tell me whether the PATCH
/api/tasks/:id route validates its request body correctly.
```

```
/context
```

**Why this matters.** That's the default habit — "give the agent the folder so it has context." Note the number on screen. Everything in `src/services`, `src/store`, every test file — none of it touches body validation, and you just paid to load all of it anyway.

### Step 2 — Measure the tight way

Fresh turn (or `/clear` for a clean comparison), give it `CLAUDE.md` plus the two files this specific task actually touches:

```
Using CLAUDE.md's architecture note, check src/routes/tasks.js and
src/services/taskService.js for request body validation on the PATCH
handler.
```

```
/context
```

**Checkpoint:** on this repo, `CLAUDE.md` + the two touched files runs about **60% smaller** than the whole-repo dump — measured here as 10.6KB vs. 4.2KB. Your own number will land somewhere in that range, not necessarily exactly this one — session overhead and tool definitions vary. The ratio is the lesson, not the exact percentage.

> **Honest note.** The ~60–70% figure is verified for *this* repo, *this* file selection (`CLAUDE.md` + `tasks.js` + `taskService.js`). Pick a different file set and the number moves — expected, not a bug.

### Step 3 — Run a short multi-turn edit both ways, compare `/cost`

Pick one path and actually make the fix (the PATCH truthiness bug from Demo 2, if you're running this cold: `if (req.body[field])` → an explicit `in` / `!== undefined` check), across 2–3 turns — ask for the fix, ask for a regression test, ask to run the suite. Do this once naive, once tight (two sessions or a `/clear` between them), then:

```
/cost
```

**Why this matters.** With the tight context, `CLAUDE.md` is the same stable prefix on every turn — that's exactly what prompt caching is for. First read costs 1.25×; every read after that, within the cache window, costs 0.10× — a 90% discount on that portion. Compounded with the smaller starting payload, the tight path's session bill drops well below the naive one. The gap between your two `/cost` numbers is the real number, not a slide.

### Step 4 — The scale line to keep

60–70% on a toy repo like TaskFlow is a rounding error — a few thousand tokens, cents either way. The same 60–70% on a 100,000-token monorepo is 60,000–70,000 tokens saved *every single turn*. The mechanism you just watched work here is the same mechanism that turns a five-figure monthly bill into a four-figure one at real scale. Nothing about the ratio changes with size — only the number of zeros on the check. (Section 05 makes this literal instead of hypothetical.)

### Step 5 — Glimpse: model routing (~2 min, optional)

No new setup — Demo 1 already put this in the repo:

```
Use the doc-writer subagent to document src/routes/tasks.js.
```

```
/cost
```

Then, on the same file:

```
Use the code-reviewer subagent to review src/routes/tasks.js.
```

```
/cost
```

**Checkpoint:** `doc-writer` (Haiku) should cost noticeably less than `code-reviewer` (this session's own tier) for a similarly-sized task. Documentation is deterministic, routine work; judging correctness and security is the hard thinking. Same output quality on the routine work, a fraction of the token cost — one line of YAML.

## 05 — Stretch — Token Budget at Scale (~20 min, optional)

TaskFlow is deliberately tiny so the mechanism above is easy to see — which also means 60–70% of it is a rounding error in real dollars. This stretch gets the *same* lever to behave like it would on a repo your team actually ships, without leaving TaskFlow or standing up a separate large codebase.

### Stretch Step 1 — Start from the real, shared baseline (2 min)

Clone the pushed repo cold instead of reusing your locally-modified copy — treat it the way you'd treat a teammate's repo you're opening for the first time:

```bash
git clone https://github.com/nsr-19/acceler-lvt-labs.git
cd acceler-lvt-labs/lvt-day4-code-demo && npm install && npm test
```

### Stretch Step 2 — Grow the repo, then feel the tax (8 min)

Ask Claude Code to scaffold two more resources that mirror the existing pattern exactly:

```
Add two new resources to this API, following the exact pattern in
src/routes/tasks.js, src/services/taskService.js, and tests/tasks.test.js:
a "projects" resource and a "notifications" resource. Each needs a route
file, a service file, and a test file with the same coverage shape as
tasks.js has. Do not touch the existing tasks or users code.
```

Run `npm test` to confirm the suite is still green, then repeat Steps 1–2 above on a question that spans the *new* code:

```
Read every file in src/ and tests/, then tell me whether the new
notifications resource validates its request body the same way tasks does.
```

```
/context
```

Fresh turn or `/clear`, then tight:

```
Using CLAUDE.md's architecture note, check src/routes/notifications.js and
src/services/notificationService.js for request body validation.
```

```
/context
```

**Checkpoint:** the naive number should have grown — more files under `src/` to dump. The tight number should have barely moved — still just `CLAUDE.md` plus two files. The *percentage* reduction gets **bigger**, not smaller, as the repo grows: the naive path pays for every file that exists, the tight path pays for every file the task touches, and those are different curves.

> **Honest caveat.** Don't repeat the original ~60% figure after this step — record the number you actually measured on the grown repo. If it didn't move much, that's worth noting honestly too: two extra resources on top of three existing ones is still a small repo in absolute terms.

### Stretch Step 3 — Multiply it with a workflow, for real numbers (6 min)

This is Demo 3's fan-out, pointed at the cost question instead of a code question:

```
use a workflow to audit every route file under src/routes/ for missing
input validation. Run two passes: in the first, give each auditor the
whole src/ folder as context; in the second, give each auditor only
CLAUDE.md plus its one route file. Report the total token cost of each
pass and the per-agent average.
```

Watch `/workflows`, drill into a phase, and read the token totals per agent.

**Why this matters.** This is the Step 4 multiplication (*N* agents × naive payload vs. *N* agents × tight payload) except every number on screen came from an actual run, not a projection. A workflow auditing five route files is nothing. A workflow auditing five hundred is the same ratio, paid five hundred times.

### Stretch Step 4 — Price it out (4 min)

Take your own measured `/cost` numbers — either the single-turn delta from Step 3 or the grown-repo delta from Stretch Step 2 — and do the arithmetic:

1. **Per-question cost, naive:** naive input tokens × your model's input price.
2. **Per-question cost, tight:** tight input tokens × input price for the first turn, then × (input price × 0.1) for every turn after.
3. **Multiply by a realistic cadence:** pick a number you recognize from your own team — e.g. 15 engineers × 8 similar questions a day × 20 workdays a month.

**Checkpoint:** pull the current per-token price from whatever pricing page is live when you run this — the exact numbers move over time, and a stale figure is worse than pointing at the real one. What doesn't move is the shape of the math: naive cost scales with repo size × question volume; tight cost scales with task size × question volume.

## 06 — Honest caveats

- The core-lab figure (~60–70%) is verified for TaskFlow at its current size and this exact file selection — it moves if either changes.
- The stretch's grown-repo figure is whatever you actually measured — don't default back to the original 60% once the repo has changed shape.
- `/cost`'s cache breakdown varies by Claude Code version — some surface `cache_read_input_tokens` directly, some show one total. Either way, the gap between two `/cost` readings is the real number.
- Pricing numbers move over time — pull the current rate from whatever pricing page is live, not from a cached figure in this guide.

## 07 — Cheat sheet

| Command | What it does |
| --- | --- |
| `/context` | Shows what's currently loaded into the session and its size |
| `/cost` | Shows token spend and (where supported) cache read/write breakdown |
| `/clear` | Starts a clean context within the same session, for a fair before/after |
| `git clone https://github.com/nsr-19/acceler-lvt-labs.git` | Pulls the pushed, pristine TaskFlow baseline |
| `use a workflow to ...` | Fans a task out across subagents — multiplies whatever payload you give it |
| `CLAUDE.md` | The tight-payload anchor: architecture notes read once, cached, reused every turn |

---

## Self-check before you move on

1. In your own words, why did `/context` drop between Step 1 and Step 2 — what specifically changed about the prompt, not just "it got smaller"?
2. Which turn in Step 3 showed the biggest `/cost` gap between naive and tight — the first, or a later one? Why that one?
3. In Stretch Step 2, did the percentage reduction get bigger or smaller as you grew the repo? Does that match the "tight cost doesn't scale with repo size" claim?
4. Can you explain the Stretch Step 3 multiplication (*N* agents × payload size) using a number other than the one in this guide?
5. Using your own team's headcount and question volume, is the Stretch Step 4 monthly gap big enough to be worth a sentence in a planning doc?
