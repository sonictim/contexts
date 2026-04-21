# Subagent Orchestration Playbook
*Reference doc for any project requiring multi-agent scale — context window relief or parallelization.*

---

## 0. Claude Code Tool Mapping

This playbook describes roles and phases abstractly. Here's how they map to actual Claude Code primitives:

| Playbook Concept | Claude Code Implementation |
|---|---|
| Orchestrator | You (the main conversation agent) |
| Spawn a worker | `Agent` tool — one call per worker |
| Parallel workers | Multiple `Agent` calls in a single message |
| Background workers | `Agent` tool with `run_in_background: true` |
| Code-writing workers | `Agent` tool with `isolation: "worktree"` — gives each agent its own git worktree so parallel file edits don't collide |
| External file persistence | `Write` / `Edit` tools (plan files, master documents) |
| Context handoff to agent | The `prompt` parameter — must be self-contained since agents start cold |
| Model selection per agent | The `model` parameter on `Agent` (`"opus"`, `"sonnet"`, `"haiku"`) |
| Continue an existing agent | `SendMessage` with the agent's ID — resumes with full context |

**Worktree rule of thumb:** Use `isolation: "worktree"` whenever a worker agent will write or edit files. Without it, two parallel agents editing the same file will race. Worktrees are automatically cleaned up if the agent makes no changes; otherwise the path and branch are returned in the result for you to merge.

---

## 1. When to Deploy Subagents (Decision Gate)

Use subagents when **at least one** of these is true:

| Trigger | Signal |
|---|---|
| **Context overflow** | Task data + history exceeds a single context window |
| **Parallelizable work** | Dataset can be chunked into independent units |
| **Breadth-first scope** | Multiple independent research directions needed simultaneously |
| **High-value output** | Task value justifies ~15x token cost over a single-agent approach |

**Do not use subagents when:**
- Agents must share the same context to reason (tightly interdependent tasks)
- Task is simple enough for 1 agent with 3–10 tool calls
- Failure modes compound across agents and are hard to trace

> **Anthropic benchmark:** Multi-agent (Opus 4 orchestrator + Sonnet 4 workers) outperformed single-agent Opus 4 by 90.2% on breadth-first research tasks. Token volume alone explained 80% of performance variance — subagents work because they scale tokens across parallel context windows.

---

## 2. Orchestrator Setup

**Do this first — before spawning any agents.**

### Required Pre-Flight: Human Scoping Gate

**Always give a heads-up before entering a multi-agent workflow.** Subagent orchestration uses ~15x the tokens of a normal single-agent approach — that's the point (more tokens = more depth), but the human should know it's happening. Before spawning any agents:

1. **Flag the approach:** Briefly explain that this task would benefit from subagents and roughly how many you plan to spawn.
2. **Ask for a quick go-ahead:** A simple "yes, go ahead" is sufficient. Don't belabor the cost — just make sure it's not a surprise.
3. **Mention a lighter alternative only if the task is borderline:** If the task clearly needs subagents, don't waste time offering a worse option. Only flag the trade-off when it's genuinely close.

Once confirmed, resolve the following conditions. Proceed autonomously if all four are clearly resolved in the project brief. Pause and ask the human if any are uncertain — a wrong answer here cascades through every phase.

| Condition | Autonomous if... | Pause and ask if... |
|---|---|---|
| **Complexity tier** | Task maps clearly to simple / moderate / complex | Task scope is ambiguous or spans multiple tiers |
| **Output schema** | Final document structure is explicitly specified | Format or required fields are undefined |
| **Tool-to-data mapping** | Source data location is known and tools are matched | Unclear where data lives or which tools apply |
| **Irreversible actions** | System only reads and writes internal files | Outputs will be sent, published, or modify external state |

If pausing: ask one question per condition. Use each answer to inform the next. Do not batch all four into a single prompt.

Once all four are resolved — by the brief or by confirmation — proceed.

---

### Plan Persistence

```
1. Analyze the task
2. Assess complexity tier (table below)
3. Write plan to external file (e.g. _plan.md): objective, worker count, MAX_LOOPS, output schema, task boundaries per agent
4. THEN spawn subagents
```

Context windows fill fast in multi-agent runs (~15x token usage vs. chat). If the orchestrator's context truncates mid-task, an unpersisted plan is unrecoverable. This step is not optional.

**Complexity tier — embed this table verbatim in the orchestrator system prompt:**

| Task Complexity | Workers to Spawn | Tool Calls per Worker | MAX_LOOPS |
|---|---|---|---|
| Simple — single fact or lookup | 1 | 3–10 | 2 |
| Moderate — comparison, review, audit | 2–4 | 10–15 | 3 |
| Complex — broad dataset, multi-topic research | 5–10+ | 15–25 | 4 |

Agents cannot self-judge appropriate effort. Without explicit rules they default to over-spawning. The same complexity assessment that drives worker count drives MAX_LOOPS — one decision at plan time, both values written to the plan file.

> **MAX_LOOPS calibration:** Moderate defaults to 3 — empirically grounded as the minimum passes before structural stability is reliably achieved. Simple gets 2 not 1 because one loop rarely surfaces all gaps. Beyond 4, diminishing returns dominate; consistently hitting the ceiling means audit Phase 1 or Phase 2, not raise the count.

**Orchestrator prompt must also include:**
- Task objective (what "done" looks like, not just what to do)
- Output format for every subagent's return value
- Explicit task boundaries — what each agent is NOT responsible for
- Tool selection heuristics (which tools apply to this domain)
- Failure handling instruction: *"If a worker returns an error or fails to produce output in the required format, tell me explicitly: which chunk failed, what the error was, and whether you are retrying or flagging. Do not proceed silently."*

> **Failure mode:** Vague orchestrator instructions cause workers to duplicate work or leave gaps. One agent investigated 2021 supply chains while two others searched the same 2025 chains. Detailed briefs prevent this.

---

## 3. Agent Brief Template

Every subagent — Parse, Worker, Revision, Critic — must receive a brief in this format before starting. The orchestrator writes these; agents do not self-brief.

```markdown
## Agent Brief

**Role:** [Parse Agent | Worker Agent | Revision Agent | Critic Agent]
**Objective:** [One sentence — what done looks like]
**Input:** [Exact source — document path, dataset chunk, prior agent output]
**Output format:** [Exact schema — fields, types, structure]
**Effort level:** [N tool calls budgeted for this task — from complexity tier table]
**Task boundaries:**
  - IN SCOPE: [explicit list]
  - OUT OF SCOPE: [explicit list — what to leave for other agents]
**Tool selection rules:**
  - Examine ALL available tools before starting — do not default to the most familiar
  - Match tool to where the data actually lives (web search for external info, file tools for local data, etc.)
  - Prefer specialized tools over generic ones
  - Available tools for this task: [list each tool + one-line purpose]
  - Do NOT use: [any tools explicitly out of scope for this agent's task]
  - If a tool is failing or returning unexpected results, flag it — do not silently work around it (see §7)
**Exit condition:** [How agent knows it is done — must be explicit, not implied]

*For Critic Agent specifically, output must include:*
- Numeric score: 0.0–1.0 (score against the rubric dimensions in §8)
- Categorical rating: needs work / good / excellent
- Numbered feedback items (specific and actionable, not general)
```

> Vague briefs are the #1 cause of agent duplication and gaps. Every field in this template exists because Anthropic learned it the hard way.

---

## 4. Architecture and Phase Breakdown

### Overview

```
TASK AGENT (Orchestrator)
│
├── PHASE 0: ORCHESTRATOR SETUP (§2)
│   ├── Persists plan to _plan.md
│   └── Sets complexity tier → worker count + MAX_LOOPS
│
├── PHASE 1: PARSE AGENT
│   └── Retrieves/parses source data → working document
│       └── SELF-REVISION LOOP (until all 5 rubric dimensions Pass)
│
├── PHASE 2: WORKER AGENTS [run in parallel]
│   ├── Worker A → condensed chunk A result
│   ├── Worker B → condensed chunk B result
│   └── Worker N → condensed chunk N result
│       └── Orchestrator synthesizes → MASTER DOCUMENT (external file)
│
└── PHASE 3: QUALITY LOOP
    ├── REVISION AGENT → structural pass on master doc
    └── CRITIC AGENT → assumption challenge + numeric score
        └── Exit: score ≥ 0.90 (excellent)
               OR delta < 0.10 (diminishing returns)
               OR MAX_LOOPS hit (2/3/4 by complexity tier)
               → flagged handoff to human if not excellent
```

---

### Phase 1 — Parse Agent

**Purpose:** Convert raw source material into a structured working document.

**Inputs:** Source data, framework docs if applicable, output schema.

**Agent behavior:**
1. Retrieve or parse source data
2. Write output to working document
3. **Self-review pass:** Score each dimension below as Pass / Fail / Unknown
4. Revise any dimension that failed or returned Unknown
5. Repeat until all five dimensions return Pass

**Self-review rubric:**

| Dimension | Pass condition |
|---|---|
| Internal consistency | No section contradicts another |
| Coverage completeness | All required output schema fields are present |
| Source fidelity | All claims traceable to source material |
| Uniqueness | No section duplicates another |
| Scope compliance | Content stays within task boundaries — nothing missing, nothing extra |

**Exit condition:** Agent explicitly declares "structurally sound" only when all five dimensions return Pass. If any dimension returns Unknown, the agent flags it and loops rather than forcing a verdict. A forced pass on an Unknown is a failed exit.

Grade each dimension in isolation — don't collapse all five into a single judgment. After each self-review pass, write out gap analysis before revising — identify what failed and why before making changes.

**When to simplify Phase 1:** The full 5-dimension self-review loop is designed for messy, unstructured input (scraped web pages, raw research dumps, ambiguous briefs). When the source material is already well-structured — a codebase, a set of existing documents, an API with clear schema — skip the self-review loop. Have the Parse Agent do a single pass to organize the material into the working document format and move on. Don't add process overhead where the input doesn't need it.

---

### Phase 2 — Worker Agents (Parallel Execution)

**Purpose:** Execute the task described in the Phase 1 document across each chunk of the dataset.

**Key rules:**
- Workers run **in parallel**, not sequentially
- Each worker receives a brief (§3 — Agent Brief Template)
- Workers operate in isolated context windows — they do not communicate with each other
- Each worker **condenses its findings** before returning — no raw output dumps to the orchestrator
- Orchestrator synthesizes all condensed results into a Master Document (external file)
- Worker count set by complexity tier at plan time (§2)

**Worker failure protocol:**

When a worker returns malformed output, no output, or an error:
1. **Retry once** with the identical brief — transient failures are common
2. **Retry with a revised brief** if the first retry fails — the brief may have been ambiguous for this chunk
3. **Flag and continue** if the second retry fails — log the chunk as UNRESOLVED with the failure reason, proceed with remaining workers

Unresolved chunks must appear in the Master Document as explicit gaps, never silent omissions. The Revision Agent will see them; the Critic will flag them if unaddressed.

> **Do not over-spawn.** Match agent count to actual parallelizable units, not to a desire for thoroughness.

---

### Phase 3 — Quality Loop

Two agents run in sequence — Revision then Critic — looping until an exit condition fires.

#### Revision Agent (Structural Pass)
- Reads the Master Document
- Checks for: contradictions between sections, factual errors, format violations, missing required fields
- Makes direct edits — outputs a revised document
- Does NOT evaluate quality or challenge assumptions (that's the Critic's job)

#### Critic Agent (Assumption Challenge)
- Reads the revised document
- Challenges assumptions with specific, actionable feedback:
  - *"Section 3 assumes X but the data in section 7 suggests Y — reconcile or flag."*
  - *"This conclusion is not supported by the worker outputs — cite a specific source or remove."*
- Returns: a **numeric score (0.0–1.0)** + categorical rating + numbered feedback items
- Does NOT make edits — only issues a verdict and brief

Critic score thresholds: `0.0–0.59 = needs work` / `0.60–0.89 = good` / `0.90–1.0 = excellent`

#### Loop Logic

```
prev_score = 0
loop_count = 0

# Initial revision pass before first critic review
revision_agent.review(master_document)

while loop_count < MAX_LOOPS:
    score, feedback = critic_agent.review(revised_document)
    delta = score - prev_score

    if score >= 0.90:
        exit("excellent", score, feedback, loop_count)

    if loop_count > 0 AND delta < 0.10:
        exit("diminishing returns", score, feedback, loop_count)

    revision_agent.implement(feedback)
    prev_score = score
    loop_count += 1

exit("max loops hit", score, feedback, loop_count)
```

**Exit path behaviors:**

| Exit Condition | Meaning | Human Handoff |
|---|---|---|
| `excellent` (score ≥ 0.90) | Passed quality gate | Document + final score |
| `diminishing returns` (delta < 0.10) | Loop plateaued — further passes unlikely to help | Document + score + unresolved feedback |
| `max loops hit` | Hard ceiling reached | Document + score + unresolved feedback + loop count |

The second and third exit paths return the document **flagged**, not clean. The critic's last numbered feedback travels with it so the human knows exactly what wasn't resolved. Never return a document that didn't pass as if it did.

> **Diagnostic signal:** Consistently hitting `max loops hit` or `diminishing returns` means the problem is upstream — weak Parse Agent output, vague worker briefs, or poor master document aggregation. Raise the loop ceiling only as a last resort.

---

## 5. Model Selection

Model tier directly determines first-pass output quality. Higher quality workers reduce revision loop count — and quality loops are expensive (Revision + Critic agents running against the full master document each pass). The math often favors Opus workers over cheap workers plus extra loops.

**Default to Opus for any role involving judgment. Default to Haiku only when the task is mechanical.**

| Role | Model | Rationale |
|---|---|---|
| Orchestrator | Opus | Non-negotiable — coordination, decomposition, and synthesis quality determines everything downstream |
| Workers — analysis, review, synthesis, judgment | Opus | Default; higher first-pass quality reduces revision loop count |
| Workers — data gathering, structured parsing, clean extraction | Haiku | Judgment quality irrelevant; speed and cost efficiency dominate |
| Parse Agent | Sonnet | Source material is usually ambiguous enough to need real reasoning, not full Opus |
| Revision Agent | Sonnet | Structural edits don't require Opus-level reasoning |
| Critic Agent | Opus | Scoring consistency and assumption challenge quality directly determines loop exit — wrong place to economize |

**The mechanical test for Haiku:** If the task requires no judgment — fetch this, extract that field, format this structure — Haiku is appropriate. If the task requires the agent to evaluate, decide, synthesize, or assess quality in any way, default to Opus.

> **Anthropic benchmark:** Opus orchestrator + Sonnet workers outperformed single-agent Opus by 90.2%. Their cookbook further tiers to Haiku for high-volume mechanical subtasks. The worker upgrade to Opus reflects a deliberate trade: known per-worker cost increase against probabilistic reduction in quality loop count. On complex tasks running 3–4 loops, this math typically favors Opus workers.

---

## 6. Context Management

**Rules:**
- Plan persistence is handled in §2 — write to external file before spawning, period
- When orchestrator context approaches limits, spawn a **fresh subagent with a clean context** and pass only the essential state via handoff document
- Worker agents are already isolated — each has its own context window by design
- Master Document is built as an **external file**, never accumulated inside the orchestrator's context

**Handoff document template (for fresh subagent context):**
```
- Task objective (1–2 sentences)
- Decisions already made (bulleted)
- Output schema (exact format expected)
- This agent's specific scope (what chunk, what task)
- What NOT to do (explicit exclusions)
```

---

## 7. Handling Tool Failures

In Claude Code, tool descriptions are system-level — agents cannot rewrite them. When a tool is underperforming or failing for a specific task:

1. **Document the failure pattern** — what the tool returns vs. what was expected
2. **Add the pattern to subsequent agent briefs** — e.g., "WebSearch returns shallow results for this domain; use WebFetch on specific URLs instead"
3. **Flag persistent failures to the human** — if a tool is fundamentally wrong for the task, say so rather than working around it silently

The goal is the same as Anthropic's "self-improving tool descriptions" concept — agents learn from tool failures — but adapted to the Claude Code environment where tool definitions are fixed. The knowledge travels via agent briefs, not tool rewrites.

---

## 8. Evaluation Strategy

**During development — small samples first:**
- Start with 10–20 test cases representing real usage
- Early-stage changes have large effect sizes (30% → 80% success) — you don't need hundreds of cases to see signal
- Add more test cases as the system matures and gains are incremental

**Quality rubric for Critic Agent:**

The Critic returns a numeric score (0.0–1.0) plus categorical rating plus numbered feedback. Score thresholds: `< 0.60 = needs work` / `0.60–0.89 = good` / `≥ 0.90 = excellent`. The numeric score enables diminishing returns detection across loops — a delta of less than 0.10 between loops signals the document has plateaued.

| Dimension | Check |
|---|---|
| Factual accuracy | Do claims match source worker outputs? |
| Internal consistency | Do sections contradict each other? |
| Completeness | Are all required aspects covered? |
| Source fidelity | Are conclusions supported by cited data? |
| Format compliance | Does output match the specified schema? |
| Assumption validity | Are assumptions stated and defensible? |

---

## 9. Common Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Workers duplicate each other's work | Vague task boundaries in briefs | Add explicit IN SCOPE / OUT OF SCOPE per agent |
| Orchestrator over-spawns agents | No scaling rules in prompt | Embed complexity tier table in orchestrator prompt (§2) |
| Context truncation mid-task | Plan not persisted to file | Write plan to external file before spawning (§2) |
| Critic loop never exits | MAX_LOOPS not set at plan time | Set from complexity tier table in §2 |
| Workers use wrong tools | No tool heuristics in brief | Tool selection rules required in every Agent Brief (§3) |
| Self-revision loops forever | No explicit exit condition | Parse Agent must score all five rubric dimensions Pass (§4 Phase 1) |
| Quality degrades in aggregation | Master doc built in orchestrator context | Build master doc as external file (§4 Phase 2) |
| MAX_LOOPS hit without excellent | No defined exit behavior | Surface document flagged with score + unresolved feedback (§4 Phase 3) |
| First-pass worker output is weak | Under-powered model | Default workers to Opus for judgment tasks (§5) |
| Worker failure silently skipped | No failure protocol | Retry twice, flag chunk as UNRESOLVED in master doc (§4 Phase 2) |
| Parallel agents corrupt each other's files | Workers edit same files without isolation | Use `isolation: "worktree"` for code-writing workers (§0) |
| Tool keeps failing across agents | Failure pattern not communicated | Document pattern in agent briefs so downstream agents avoid it (§7) |

---

## 10. Eval Checkpoints — Run These Before Shipping Any Agent System

- [ ] Human given heads-up and confirmed go-ahead before spawning (§2)
- [ ] Complexity tier assessed — worker count and MAX_LOOPS set at plan time
- [ ] Plan written to external file before any agents spawn
- [ ] Model tier assigned to every agent role (§5)
- [ ] Every subagent receives a complete Agent Brief (§3)
- [ ] Tool descriptions are specific and unambiguous
- [ ] Workers running in parallel, not sequentially
- [ ] Code-writing workers use `isolation: "worktree"` (§0)
- [ ] Workers condense findings before returning — no raw output dumps
- [ ] Worker failure protocol in place (retry twice, flag and continue)
- [ ] Parse Agent complexity matches source material (full self-review for messy input, single pass for structured — §4 Phase 1)
- [ ] Master Document built as external file, not in-context accumulation
- [ ] All three quality loop exit paths defined (excellent / diminishing returns / max loops hit)
- [ ] Non-excellent exit surfaces document flagged, not clean
- [ ] Tool failure patterns documented in agent briefs, not silently worked around (§7)

---

## 11. Worked Example — Codebase Audit

**Task:** "Audit the pitchfreq codebase for memory safety issues, unused allocations, and potential buffer overflows."

### Phase 0: Plan (written to `_plan.md`)

```markdown
**Objective:** Produce a ranked list of memory safety concerns in ~/dev/pitchfreq, with severity, file location, and recommended fix for each.
**Complexity:** Moderate (codebase is ~15 files, multiple concern categories)
**Workers:** 3 (one per concern category)
**MAX_LOOPS:** 3
**Output schema:** Markdown table — severity (high/med/low), file:line, description, recommended fix
```

### Phase 1: Parse Agent (Sonnet)

Single-pass (source is a well-structured codebase — skip self-review loop per §4 note).
Agent reads the project structure, produces a working document listing all source files grouped by module with brief descriptions.

### Phase 2: Workers (3x Opus, parallel)

| Worker | Scope | NOT in scope |
|---|---|---|
| A — Memory safety | Use-after-free, double-free, uninitialized memory | Performance, style |
| B — Unused allocations | Allocated-but-never-read, leaked allocators | Correctness of used allocations |
| C — Buffer overflows | Slice bounds, unchecked indices, FFT buffer sizing | Allocation strategy |

Each worker returns: condensed findings as a markdown table (severity, file:line, description, fix).
Orchestrator merges into master document, deduplicates any overlapping findings.

### Phase 3: Quality Loop

- **Loop 1:** Revision Agent (Sonnet) consolidates duplicate findings, standardizes severity ratings. Critic Agent (Opus) scores 0.78 — flags that Worker C missed the resampling buffer edge case and two severity ratings seem inconsistent.
- **Loop 2:** Revision Agent addresses feedback. Critic scores 0.92 — excellent. Exit.

### Handoff

Final document returned to human with score (0.92) and note: "2 loops, all categories covered, no unresolved chunks."

---

*Last updated: April 2026. Source: Anthropic Engineering Blog, "How we built our multi-agent research system," June 13, 2025.*
