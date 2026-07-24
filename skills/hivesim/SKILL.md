---
name: hivesim
description: Strategic wargame engine. 7-phase simulation pipeline that runs parallel market simulations, attacks its own conclusions, then imagines how the decision fails. Use for big strategic decisions, pricing changes, market entry, partnership evaluation, or any question where being wrong is expensive. Triggers when user says /hivesim or asks to wargame a decision.
user_invocable: true
---

# /hivesim — Strategic Wargame Engine

A 7-phase strategic simulation pipeline that runs parallel market simulations, attacks its own conclusions, then imagines how the decision fails. Produces a one-page Decision Brief with a Go/No-Go recommendation, kill criteria, and next action.

**What it combines:**
- **Hive** (parallel agent orchestration, spawns and coordinates sub-agents)
- **Sim** (quantitative buyer simulation, 10-15 AI personas make independent BUY/PASS decisions)
- **Miro** (deep qualitative simulation, models multi-turn buying journeys with social dynamics)
- **Red Team** (adversarial agent that attacks the recommendation)
- **Pre-Mortem** (assumes the decision failed and writes the post-mortem)

Use when: big strategic decisions, pricing changes, market entry, accelerator prep, partnership evaluation, competitive positioning, or any question where being wrong is expensive.

**Example:** `/hivesim should we enter the enterprise vertical at $499/mo before or after core-market traction?`

## Ultracode Mode (Workflow-tool orchestration)

When the Workflow tool is available (under ultracode/xhigh it always is), run /hivesim as ONE Workflow script instead of hand-rolling the agent waves and the `.hive/findings` disk choreography. The 7-phase pipeline maps almost 1:1 onto Workflow primitives, and the runtime gives you journaling, checkpoint/resume, the concurrency cap, the budget ceiling, and schema-validated agent I/O for free. The phase vocabulary is unchanged: you still run Phase 0 Memory Callback, Phase 1 Scout, Phase 2 Hive Decomposition, Phase 3 Sim, Phase 4 Miro, Phase 5 Red Team with its 3 adversaries plus synth, Phase 6 Pre-Mortem with its 6b verifier, and Phase 7 Decision Brief. Workflow is the substrate under those names, not a replacement for them.

**Detection rule:** Workflow available AND ultracode/xhigh on -> use the Workflow path below. Otherwise (tool absent, or a single-turn zero-or-one-agent gut check) use the Manual Agent path documented in the rest of this file. The two paths are behaviorally equivalent: any FATAL the manual path would surface, the Workflow path surfaces too.

**Concept -> primitive map (specific to this skill):**
- Phase boundaries (Phase 3 needs the full Phase 2 SIMULATION PLAN before any persona votes) -> `phase(title)` groups, each phase's fan-out awaited in sequence. `meta.phases` titles MUST match the `phase()` calls exactly.
- Phase 3 Sim persona pool (10-15 personas each voting BUY/MAYBE/PASS) -> `parallel()` fan-out with `opts.schema` enforcing `{decision: BUY|MAYBE|PASS, confidence, reasoning}`. The schema replaces the per-persona disk-write durability hack; the runtime journals each vote.
- Phase 4 Miro deep journeys (top 2-3 scenarios, multi-turn) -> `pipeline(scenarios, deepJourney)` so each scenario's narrative streams independently with no barrier. Give the journey a schema with a `recommend: GO|NO` field so the Sim/Miro agreement bonus is computable in JS.
- Phase 5 Red Team, 3 specialized adversaries (Financial / Technical / Market) -> `parallel()` of 3 schema'd `agent()` calls (a real barrier), then ONE synthesis `agent()` that merges and de-duplicates into the RED TEAM REPORT. Cross-adversary corroboration and FATAL escalation are computed in plain deterministic JS over the schema'd findings, not prose-parsed. Every FATAL then passes Phase 5b verification (one skeptic per finding tries to refute it against the live product, repo, and memory); only VERIFIED FATALs may gate the verdict.
- Phase 6 Pre-Mortem -> 6b verifier -> a `pipeline()` stage: write the narrative, then a low-effort verifier returns `{verdict: PASS|RETRY|ACCEPT_WITH_NOTE}`; on RETRY re-run the pre-mortem ONCE with missing anchors injected (the documented single-retry ceiling).
- Confidence Calculation + Phase 7 Decision Brief assembly -> plain JS for the confidence math (`base + red_team_penalty + sim/miro agreement_bonus`, clamp 0.3-0.95) feeding a final synthesis `agent()`.
- Adaptive Depth (Quick / Standard / HighStakes) -> plain JS branching on `args` at the top: it sets persona count (8/12/15), scenario count, and whether the 3-adversary split runs (Quick collapses to one combined adversary).
- Concurrency ceilings, the `total_agents<=13` math, 529 backoff -> in the script you do not re-implement them; the runtime auto-caps LOCAL parallelism at `min(16, cores-2)` and queues the rest. That local cap is not the API 529 ceiling, so chunk a large persona fan-out (HighStakes is 15 x 5) into batches of <= 13 concurrent agents if you ever see 529s (Execution Rules: personas in waves of 10-15 are proven stable). The manual ceilings stay as the fallback.
- Phase-outputs-to-disk + `--resume` + snapshots -> Workflow journaling; relaunch with `{ scriptPath, resumeFromRunId }` and the unchanged `agent()` prefix returns cached results on resume per the runtime's documented behavior, so completed personas and adversaries are not re-run when the runtime supports it. Keep a thin human-facing report write if you still want the audit trail, but it is a side effect, not the coordination channel.

The AskUserQuestion depth gate still belongs in the skill's pre-script step, not inside the script. Keep all in-script logic deterministic: no wall-clock reads (pass the date in via `args`), no RNG (vary per-item by index). Never fabricate persona reactions or numbers, and never pass API keys in agent prompts.

**Phase 8 is NOT replaced by the Workflow journal.** The MANDATORY Cross-Skill Learning Sync exit gate (Phase 8) still runs after the script returns the Decision Brief: append to `~/.claude/hive-history.jsonl`, `~/.claude/hive-calibration.jsonl`, and `~/.claude/hive-muscle.jsonl`, then end your user-facing message with the exact `Self-improvement: history+1, calibration+{N}, muscle+{K}/-{C}.` line. That happens on BOTH paths; the Workflow journal is run state, not the cross-skill muscle/calibration write.

```javascript
export const meta = {
  name: "hivesim",
  description: "7-phase strategic wargame: scout, decompose, sim, miro, red team, pre-mortem, decision brief",
  phases: ["Scout", "Decompose", "Sim", "Miro", "Red Team", "Pre-Mortem", "Decision Brief"]
};

// HARD RULE (hard-won lesson, learned twice): the Workflow `args` input can arrive as a
// JSON STRING, not an object — the first `A.x.map` then crashes the whole run.
// Normalize FIRST, or skip args entirely and inline the data as a const.
const A = typeof args === "string" ? JSON.parse(args) : (args || {});

// Adaptive Depth: plain JS branch on args (date passed in for determinism)
const depth = A.depth || "Standard";
const cfg = {
  Quick:      { personas: 8,  scenarios: 2, splitAdversaries: false },
  Standard:   { personas: 12, scenarios: 3, splitAdversaries: true },
  HighStakes: { personas: 15, scenarios: 5, splitAdversaries: true }
}[depth] || { personas: 12, scenarios: 3, splitAdversaries: true };
const ctx = "Question: " + A.question + "\nProduct context is mandatory in every persona prompt.";

// Phase 1: Scout (Phase 0 memory callback folded into the brief)
phase("Scout");
const intel = await agent("Read the product vision doc in full, gather fresh competitive intel, check prior hivesim runs for this topic. Collect REALITY ANCHORS: every observed real-world number the business has bearing on the question (real conversion, reply rates, pilot usage, churn, volumes), each tagged [observed]; personas must not contradict an observed number. Produce INTEL BRIEF (with REALITY ANCHORS section) + PRODUCT BRIEF. " + ctx, { phase: "Scout" });

// Phase 2: Hive Decomposition (barrier: Phase 3 needs the full plan)
phase("Decompose");
const plan = await agent("Decompose into " + cfg.scenarios + " scenarios and the variables that matter. Return the SIMULATION PLAN. " + ctx + "\nIntel: " + intel, {
  phase: "Decompose",
  schema: { type: "object", required: ["scenarios"], properties: { scenarios: { type: "array", items: { type: "string" } } } }
});
const scenarios = plan.scenarios.slice(0, cfg.scenarios);

// Phase 3: Sim. Persona pool votes BUY/MAYBE/PASS (parallel barrier so quorum sees all).
// The runtime caps local concurrency at min(16, cores-2) and queues the rest; no manual
// batching. If API 529s ever reappear, reintroduce chunking in waves of <=13.
phase("Sim");
// Integrity rule (hard-won attribution lesson): never require the model to echo back an
// identifier used for aggregation. The scenario is attached from the closure after the call resolves.
const voteSchema = { type: "object", required: ["decision"], properties: {
  decision: { type: "string", enum: ["BUY", "MAYBE", "PASS"] },
  confidence: { type: "number" },
  reasoning: { type: "string" }
} };
const personaJobs = scenarios.flatMap((sc) =>
  Array.from({ length: cfg.personas }, (_, p) => () =>
    agent("You are buyer persona #" + p + " evaluating scenario: " + sc + ". Vote BUY/MAYBE/PASS with reasoning. " + intel,
      { phase: "Sim", effort: "low", schema: voteSchema })
      .then(v => v ? { ...v, scenario: sc } : v)));
const votes = (await parallel(personaJobs)).filter(Boolean);
const buyRate = sc => {
  const v = votes.filter(x => x.scenario === sc);
  return v.length ? v.filter(x => x.decision === "BUY").length / v.length : 0;
};
// Uniformity guard (hard-won lesson): zero-vote scenarios or all-identical rates signal
// an aggregation bug or a uniform confound, not a dead heat. Flag before trusting the ranking.
const voteCounts = scenarios.map(sc => votes.filter(x => x.scenario === sc).length);
if (voteCounts.some(c => c === 0)) log("INTEGRITY WARNING: scenario(s) with 0 attributed votes " + JSON.stringify(voteCounts) + ". Inspect the journal before trusting the ranking.");
if (scenarios.length > 1 && new Set(scenarios.map(sc => buyRate(sc).toFixed(2))).size === 1) log("INTEGRITY WARNING: every scenario scored identically. Check for a uniform confound (e.g. unrendered merge fields) before trusting the ranking.");
const top = [...scenarios].sort((a, b) => buyRate(b) - buyRate(a)).slice(0, 3);

// Phase 4: Miro deep journeys for top scenarios (pipeline, no barrier). Schema'd and kept
// index-aligned to `top` (do NOT filter here) so the Sim/Miro agreement bonus is deterministic.
phase("Miro");
const deep = await pipeline(top,
  (prev, sc) => agent("Simulate the full multi-turn buying journey for scenario: " + sc + ". Initial reaction, internal pitch, objections, final decision, would they refer peers. Then give your verdict. " + intel,
    { phase: "Miro", schema: { type: "object", required: ["scenario", "recommend"], properties: {
        scenario: { type: "string" }, recommend: { type: "string", enum: ["GO", "NO"] }, narrative: { type: "string" } } } })
);

// Phase 5: Red Team. 3 specialized adversaries in parallel (barrier), then synth.
phase("Red Team");
const findSchema = { type: "object", required: ["findings"], properties: { findings: { type: "array", items: {
  type: "object", required: ["claim", "severity"], properties: {
    claim: { type: "string" },
    severity: { type: "string", enum: ["FATAL", "SERIOUS", "MINOR"] },
    lens: { type: "string" }
  } } } } };
const lenses = cfg.splitAdversaries
  ? ["hostile CFO, attack only the math", "hostile engineering lead, attack only feasibility", "hostile rival CEO, attack only market dynamics"]
  : ["combined adversary, attack math, feasibility, and market"];
const reports = (await parallel(
  lenses.map((lens) => () =>
    agent(lens + ". Rate each finding FATAL/SERIOUS/MINOR and set its lens. Recommendation so far rests on: " + JSON.stringify({ buyRates: top.map(buyRate), deep: deep.filter(Boolean) }),
      { phase: "Red Team", effort: "high", schema: findSchema }))
)).filter(Boolean);
const allFindings = reports.flatMap(r => r.findings);
const fatal = allFindings.filter(f => f.severity === "FATAL");

// Phase 5b: FATAL verification (no-alarmism rule: verify red-team claims against the live
// product BEFORE they gate the verdict). One skeptic per FATAL tries to REFUTE it against
// the repo, live product, and memory. A failed verifier keeps the FATAL conservatively.
const checkedFatal = (await parallel(fatal.map(f => () =>
  agent("Adversarially verify this FATAL wargame claim before it can force a NO-GO. Check the live product, the repo, and memory files for hard evidence. Try to REFUTE it. Claim: " + f.claim + " (lens: " + (f.lens || "combined") + "). " + ctx,
    { phase: "Red Team", effort: "high", schema: { type: "object", required: ["refuted"], properties: {
      refuted: { type: "boolean" }, evidence: { type: "string" } } } })
    .then(v => v ? { ...f, refuted: v.refuted, evidence: v.evidence } : { ...f, refuted: false, evidence: "verifier failed; FATAL kept conservatively" })
))).filter(Boolean);
const verifiedFatal = checkedFatal.filter(f => !f.refuted);
const refutedFatal = checkedFatal.filter(f => f.refuted);

// Deterministic FATAL gate: any VERIFIED FATAL forces CONDITIONAL; a verified FATAL
// corroborated by 2+ distinct lenses forces automatic NO-GO. No prose-parsing.
const norm = s => String(s).toLowerCase().replace(/[^a-z0-9]+/g, " ").trim().slice(0, 60);
const lensesByClaim = {};
for (const f of verifiedFatal) {
  const k = norm(f.claim);
  (lensesByClaim[k] = lensesByClaim[k] || new Set()).add(f.lens || "combined");
}
const corroboratedFatal = Object.keys(lensesByClaim).some(k => lensesByClaim[k].size >= 2);
const forcedVerdict = corroboratedFatal ? "NO-GO" : (verifiedFatal.length ? "CONDITIONAL" : null);

const redReport = await agent("Merge and de-duplicate these adversary findings into one RED TEAM REPORT, note cross-adversary corroboration, pick top 3 risks. Refuted FATALs were killed in verification, demote them to context only: " + JSON.stringify(refutedFatal) + ". Findings: " + JSON.stringify(allFindings), { phase: "Red Team", effort: "high" });

// Phase 6 + 6b: Pre-Mortem then specificity verifier, single retry on RETRY.
phase("Pre-Mortem");
const premortem = (await pipeline([{ inject: "" }],
  (prev, item) => agent("It is 12 months from " + A.date + ". The decision was made and FAILED. Write a specific post-mortem with named competitors, dollar amounts, dates, a named churned customer. " + item.inject + " " + ctx, { phase: "Pre-Mortem", effort: "high" }),
  async (narrative) => {
    const check = await agent("Verify this pre-mortem meets all specificity anchors (2 competitors, 3 dollar/percent, 2 dates, 1 named customer). Narrative: " + narrative,
      { phase: "Pre-Mortem", effort: "low", schema: { type: "object", required: ["verdict", "missing"], properties: {
        verdict: { type: "string", enum: ["PASS", "RETRY", "ACCEPT_WITH_NOTE"] },
        missing: { type: "array", items: { type: "string" } } } } });
    if (check.verdict !== "RETRY") return { narrative, note: check.verdict === "ACCEPT_WITH_NOTE" ? check.missing : null };
    const retry = await agent("Rewrite the pre-mortem. You previously omitted: " + check.missing.join(", ") + ". Include each. " + ctx, { phase: "Pre-Mortem", effort: "high" });
    return { narrative: retry, note: null };
  }
)).filter(Boolean)[0];

// Confidence Calculation (deterministic JS): base + red_team_penalty + sim/miro agreement_bonus.
phase("Decision Brief");
const spread = Math.max(...top.map(buyRate)) - Math.min(...top.map(buyRate));
const base = spread < 0.15 ? 0.8 : 0.6;
const penalty = verifiedFatal.length * -0.15 + allFindings.filter(f => f.severity === "SERIOUS").length * -0.05;
const miroAgrees = !!(deep[0] && deep[0].recommend === "GO"); // deep is index-aligned to top; top[0] is the Sim winner
const agreementBonus = miroAgrees ? 0.1 : -0.1;
const confidence = Math.min(0.95, Math.max(0.3, base + penalty + agreementBonus));

const brief = await agent("Write the one-page Decision Brief. Required recommendation: " + (forcedVerdict || "GO / NO-GO / CONDITIONAL as the evidence warrants") + ". Lead with any verified FATAL. Tag every figure [observed] or [simulated] per the intel's REALITY ANCHORS; if none are observed, state 'UNANCHORED: all figures simulated' under the numbers. Confidence: " + Math.round(confidence * 100) + "%.\n" +
  "Votes: " + JSON.stringify(top.map(s => ({ s, buyRate: buyRate(s) }))) + "\n" +
  "Red Team: " + redReport + "\n" +
  "Verified FATAL findings: " + JSON.stringify(verifiedFatal) + "\n" +
  (refutedFatal.length ? "Refuted FATALs (verification killed these; one line of context at most): " + JSON.stringify(refutedFatal) + "\n" : "") +
  "Pre-mortem: " + premortem.narrative + (premortem.note ? "\nPre-mortem confidence: medium (gaps: " + premortem.note.join(", ") + ")" : ""),
  { phase: "Decision Brief", effort: "high" });

// Phase 8 (MANDATORY exit gate) is NOT replaced by the Workflow journal: after this returns,
// append to hive-history/calibration/muscle and end your message with the 'Self-improvement:' line.
return brief;
```

**FALLBACK:** When the Workflow tool is unavailable, the manual Agent-tool path documented in the rest of this file (the `.hive/findings` phase files, the snapshot/`--resume` protocol, the concurrency ceilings, and the stigmergy merge) is authoritative and still applies in full. Do not delete it; it is the spec this plugin ships, and it must stay behaviorally equivalent to the Workflow path above.

**Two-path equivalence checklist (update BOTH paths and this list in the same edit, or don't make the edit):**
1. Persona votes carry attribution attached by the orchestrator (closure / orchestrator-assigned ID), never model-echoed identifiers.
2. Uniformity guard before ranking: zero-vote scenarios or all-identical scores are flagged, not trusted.
3. Phase 5b FATAL verification: only verified FATALs gate the verdict; refuted FATALs demote to context; a failed verifier keeps its FATAL.
4. Reality anchors from Phase 1 reach every persona prompt; observed vs simulated figures stay tagged through to the brief.
5. Confidence math: base + red-team penalty (verified FATALs only) + sim/miro agreement bonus, clamped 0.30-0.95.
6. Pre-mortem verifier with single-retry ceiling.
7. Phase 8 exit gate incl. the outcomes-ledger row.

This checklist exists because of a real drift finding: a documented integrity lesson sat in memory for four days while the script kept the bug. The checklist is the tripwire.

## Arguments

$ARGUMENTS -- The strategic question or scenario to wargame. Can be a single question or a structured scenario with parameters.

## Pipeline (7 phases, executed in order)

### Phase 0: Memory Callback
Before doing anything new, check if this topic has been analyzed before:
1. Read `~/.claude/hive-history.jsonl` for related past runs
2. Read memory files for prior decisions on this topic
3. If found, note: "Previously concluded: [X] on [date]. Here's what changed since then: [Y]"
4. This prevents re-litigating settled decisions and shows evolution of thinking

### Phase 0b: Outcome Review (reality feedback loop)

The single biggest epistemic gap in a simulation engine is never checking whether its past calls were right. Close it here:

1. Read `~/.claude/hivesim-outcomes.jsonl`. Each row: `{run_id, date, question, verdict, confidence, kill_criteria, review_after, status}`.
2. Any row with `status: "pending"` and `review_after` in the past is DUE. Surface it to the user in ONE compact question before the new run starts: "Past call due for review: [verdict] on [question] at [confidence]%. Did reality confirm it? (right / wrong / too early / abandoned)".
3. Record the answer: update the row's `status` to `right | wrong | too_early | abandoned` (rewrite the line; this file is the one exception to append-only). `too_early` pushes `review_after` out by the same interval.
4. Feed calibration: append `{run_id, reported_confidence, outcome}` to `~/.claude/hive-calibration.jsonl`. A `wrong` at high confidence is the most valuable row this system ever writes: extract WHY into a muscle entry (what did the sim miss: persona pool gap, unmodeled competitor, fabricated anchor?).
5. If the user does not answer, leave the row pending. Never self-grade an outcome; only reality (the user) grades.

Skip this phase silently only when the outcomes file is empty or absent.

### Phase 1: Scout Pre-Load
**Product knowledge gate (MANDATORY):** Before building ANY personas, read your product vision document in full (wherever the project keeps it: README, CLAUDE.md, a product brief). It must cover what the product actually is, every user type, all features built, and the competitive positioning. Personas that don't know the product produce garbage scores (in testing, v1 without product knowledge scored 4.5/10, v2 with it scored 6.2/10 on the same question). Every persona prompt MUST include a "Product Context" block summarizing what the product offers. Never let a persona evaluate the product based only on the question text.

Gather fresh external context before simulating. Use web search to find:
- Latest competitive intel relevant to the question
- Recent industry news or market shifts
- Any data points that should ground the simulation
- Check memory for known competitor data (scout history)

**Reality anchors (MANDATORY):** Synthetic votes must never contradict observed reality. Before simulating, collect every REAL number the business already has that bears on the question: actual conversion rates (analytics/billing), actual outreach reply rates, actual pilot-customer behavior, actual churn/retention, actual usage volumes. List them in the INTEL BRIEF under `REALITY ANCHORS`, each tagged `[observed]`. Rules downstream:
- Persona prompts include the anchors. A persona may not assume a conversion or willingness-to-pay that an observed number already falsifies.
- The Decision Brief's "The Numbers" table marks each figure `[observed]` or `[simulated]`. Never let a simulated number sit next to an observed one without the tag.
- If ZERO observed numbers exist for the question, say so in the brief: "UNANCHORED: all figures simulated." That sentence alone prevents synthetic precision from masquerading as data.

Output: `INTEL BRIEF` (5-10 bullet points of relevant fresh data, REALITY ANCHORS section) + `PRODUCT BRIEF` (key features/pricing to inject into persona prompts)

### Phase 2: Hive Decomposition
Break the strategic question into parallel simulation tracks. Use Hive's orchestration to:
1. Identify 3-5 distinct scenarios or variants to test
2. Identify which variables matter most (price, timing, positioning, audience)
3. Plan the agent waves: which scenarios run in parallel, which are sequential
4. Apply Hive's concurrency rules (proven ceiling: 25 read / 20 write agents, zero failures)

Output: `SIMULATION PLAN` with scenarios, variables, and wave structure

### Phase 3: Sim (Market Numbers)
For each scenario, run /sim-style quantitative analysis:
- Build 10-15 buyer personas (use /sim's persona pool, adapted to the question)
- Each persona independently evaluates each scenario: BUY / MAYBE / PASS
- Calculate: conversion rate, projected ARR, risk-adjusted revenue, willingness to pay
- Spawn personas in waves of 10-15 low-effort agents (proven stable; inherit the session model, low effort for persona simulation, high effort for synthesis)

Output: `MARKET DATA` table with per-scenario metrics

### Phase 4: Miro (Deep Persona Simulation)
For the top 2-3 scenarios from Phase 3, run deep simulations:
- Select 3-5 key personas (highest-stakes buyers, most representative)
- Simulate multi-turn decision processes: initial reaction, internal pitch to team, objections raised, final decision
- Model social dynamics: would this buyer recommend to peers? Would they tweet about it?
- Test emotional triggers: urgency, FOMO, trust, skepticism

Unlike /sim's one-shot decisions, Miro simulates the full buying journey.

Output: `DEEP INSIGHTS` with narrative per key persona

### Phase 5: Red Team (parallel, 3 specialized adversaries)

**Old approach (deprecated):** One agent attacked from all angles. Single attack vector — agents tend to fixate on whichever attack feels strongest and underweight the rest. Audit found assumption/data attacks dominated, competition/timing attacks were weak.

**New approach:** Three high-effort adversaries run in parallel, each with a narrow remit (inherit the session model; do not pin an older model name). A 4th high-effort call merges and de-duplicates. Same total cost as the prior single-agent approach for Standard/HighStakes depth (3 + 1 = 4 calls instead of 1 long one with deeper context); attack diversity is the win.

**Adversary 1 — Financial / Numbers Critic (high effort)**
```
You are a hostile CFO reviewing this recommendation. Your only target: the math.

Attack the numbers, not the strategy. Focus on:
- Conversion-rate assumptions: are they grounded in the persona pool or aspirational?
- ARR projection: what's the true CAC, churn, and gross margin? Where do they break?
- Hidden costs: support burden, onboarding cost, bad-debt, refunds, infra
- Sensitivity: if conversion drops 30% what's the picture?
- Cash-flow timing: when do we actually see revenue vs spend?
- Unit economics: is each customer profitable on payback at month 6/12/18?

Find at minimum:
- 2 numerical assumptions that are probably wrong, with the corrected estimate
- 1 cost the simulation missed entirely
- 1 sensitivity scenario that flips the recommendation

Rate each: FATAL / SERIOUS / MINOR. Be specific with numbers, not adjectives.
```

**Adversary 2 — Technical / Execution Critic (high effort)**
```
You are a hostile engineering lead reviewing this recommendation. Your only target: feasibility.

Attack the build/operate plan, not the market opportunity. Focus on:
- What new infrastructure does this need? What's the real engineering cost?
- What scaling cliffs hit between 10/100/1000 customers?
- Operational load: support tickets, on-call burden, fraud handling, compliance
- Integration risk: which third parties can break this and at what frequency?
- Team capacity: do current humans have bandwidth, or does this require hiring?
- Failure modes: what breaks first under load and who notices?

Find at minimum:
- 2 execution blockers that the recommendation hand-waves
- 1 scaling cliff with the customer count where it hits
- 1 third-party dependency that can kill the offering

Rate each: FATAL / SERIOUS / MINOR. Be specific.
```

**Adversary 3 — Market / Competitor Critic (high effort)**
```
You are a hostile rival CEO reviewing this recommendation. Your only target: market dynamics.

Attack the positioning and competitive moat. Focus on:
- How would the named incumbents in your market / a well-funded competitor neutralize this?
- Survivorship bias: who's NOT in the persona pool that should be?
- Timing: why is now wrong? What changes in 6/12/18 months?
- Substitutes: what existing tool, even if mediocre, satisfies the same job-to-be-done?
- Distribution: how does this customer actually find out about the product, and is that channel saturated?
- Differentiation: what stops a 5-eng-team from cloning the surface in a quarter?

Find at minimum:
- 2 specific competitor responses (named companies, named features)
- 1 survivorship-bias gap in the persona pool
- 1 timing argument for waiting

Rate each: FATAL / SERIOUS / MINOR. Be specific.
```

**Synthesis (high effort, 1 call):** Reads all 3 adversary reports, merges and de-duplicates findings, produces unified `RED TEAM REPORT` with:
- All findings grouped by severity (FATAL → SERIOUS → MINOR)
- Per-finding attribution (which adversary raised it)
- Cross-adversary corroboration noted (`[corroborated: Financial+Market]` carries more weight than a solo finding)
- Top 3 risks selected for the Decision Brief

**Phase 5b — FATAL verification (mandatory):** Before any FATAL can gate the verdict, spawn one skeptic per FATAL finding that tries to REFUTE it against the live product, the repo, and memory files. This encodes the no-alarmism rule directly into the pipeline: verify red-team claims against reality before treating them as fatal. Refuted FATALs are demoted to context and get at most one line in the brief. A verifier that errors keeps its FATAL conservatively (never drop a FATAL because verification failed to run).

**FATAL escalation:** If a VERIFIED FATAL survives, the Decision Brief MUST lead with it. The recommendation becomes CONDITIONAL or NO-GO until the finding is mitigated. Do not bury verified FATALs in a list. Cross-adversary corroboration of the same verified FATAL triggers automatic NO-GO.

**Cost note:** For Quick-check depth, fall back to a single combined adversary (the prior single-agent prompt) to keep the run cheap. Standard and HighStakes use the 3-adversary split.

Output: `RED TEAM REPORT` (synthesized) with rated attacks

### Phase 6: Pre-Mortem
Spawn a separate agent (high effort) that assumes the decision was made AND FAILED:

Pre-Mortem Agent Prompt:
```
It is 12 months from today ({current_date + 12 months}). The decision described below was made and it FAILED. The company wasted time, money, and opportunity cost.

Write a post-mortem explaining WHY it failed. Be specific:
- What went wrong first?
- What was the cascading effect?
- What early warning signs were ignored?
- What alternative would have worked better?
- In hindsight, what was the fatal flaw in the original analysis?

SPECIFICITY REQUIREMENTS (non-negotiable):
- Name at least 2 specific competitors by company name (not "a competitor")
- Include at least 3 specific dollar amounts or percentages (MRR, churn rate, CAC)
- Include at least 2 specific dates/timelines (not "eventually" or "soon")
- Name at least 1 specific customer or customer type who churned, with their reason
- Reference the actual pricing/feature decisions from the scenario, not generic SaaS failure

Do not write a generic failure. Write a SPECIFIC, plausible failure narrative grounded in the actual scenario details. A good pre-mortem reads like a real incident report, not a business school case study.
```

### Phase 6b: Pre-Mortem Specificity Verifier

The pre-mortem requirements list specific anchors (named competitors, dollar amounts, dates, named customers). Without verification these are aspirational — agents drift to generic failure narratives. Spawn ONE low-effort checker against the just-written pre-mortem:

```
You are a fact-anchor checker. Your only job: verify the pre-mortem below meets ALL specificity requirements.

REQUIREMENTS:
1. At least 2 SPECIFIC competitor company names (not "a competitor")
2. At least 3 SPECIFIC dollar amounts or percentages (with units)
3. At least 2 SPECIFIC dates or timelines (not "eventually", "soon")
4. At least 1 named customer or customer type with a stated reason for churning
5. References actual scenario pricing/feature decisions (not generic SaaS failure)

Return JSON only:
{
  "passes": true|false,
  "found": {"competitors": [...], "dollar_amounts": [...], "dates": [...], "customers": [...]},
  "missing": ["list of unmet requirements"],
  "verdict": "PASS | RETRY | ACCEPT_WITH_NOTE"
}

Use ACCEPT_WITH_NOTE if it's borderline (e.g. 2 dates but one is fuzzy). Use RETRY only if 2+ requirements are unmet.
```

Action by verdict:
- **PASS**: Continue to Phase 7
- **ACCEPT_WITH_NOTE**: Continue, but the Decision Brief must include `Pre-mortem confidence: medium (specificity gaps: {missing})`
- **RETRY**: Re-run Phase 6 ONCE with the missing requirements injected directly into the prompt: `You previously failed to include: {list}. Specifically include each missing item this run.` If retry still fails, accept with note. Never retry more than once (cost ceiling).

Log: `Phase 6b: pre-mortem verifier {verdict}, {N} retries`. This addresses an audit finding that Phase 6 had requirements but no enforcement.

Output: `PRE-MORTEM NARRATIVE` (500 words max)

### Phase 7: Decision Brief Synthesis
Synthesize everything into a one-page decision brief:

```
# Decision Brief: {question}
Date: {date} | Scenarios tested: {N} | Personas: {N} | Confidence: {X}%

## Recommendation
**{GO / NO-GO / CONDITIONAL}**: {one sentence}

## Why
{2-3 sentences summarizing the strongest evidence}

## The Numbers
| Scenario | Conversion | Projected ARR | Risk-Adjusted |
|----------|-----------|---------------|---------------|
| A        | X% [simulated] | $X       | $X            |
| B        | X% [observed]  | $X       | $X            |

(Every figure tagged [observed] or [simulated]. If zero figures are [observed], state "UNANCHORED: all figures simulated" directly under the table.)

## Top 3 Risks (from Red Team)
1. {risk} — {FATAL/SERIOUS/MINOR} — {mitigation}
2. {risk} — {FATAL/SERIOUS/MINOR} — {mitigation}
3. {risk} — {FATAL/SERIOUS/MINOR} — {mitigation}

## Pre-Mortem: How This Fails
{2-3 sentence summary of the failure narrative}

## Kill Criteria
Abandon this if:
- {specific measurable condition 1}
- {specific measurable condition 2}

## Next Action
{One specific thing to do this week. Not "think about it." A concrete action.}

## vs. Last Time
{If topic was analyzed before: what changed and why the conclusion may differ}
```

### Phase 8: Cross-Skill Learning Sync (MANDATORY exit)

**Audit gap this fixes:** /hivesim runs appended to `hive-history.jsonl` but never wrote to `hive-muscle.jsonl`. Result: powerful lessons (e.g. "personas without product context score 4.5/10") never became reusable techniques. /hive's Step 10d only ran from /hive invocations.

**Fix:** /hivesim has its own muscle-memory extraction step, mirroring /hive's Step 10d. Same file (`~/.claude/hive-muscle.jsonl`), same format, but `task_type: "hivesim"` for hivesim-specific entries and `task_type: "universal"` when the technique generalizes.

After the Decision Brief is written, before returning to user:

1. **Bootstrap** (if missing): `touch ~/.claude/hive-muscle.jsonl` and `touch ~/.claude/hive-calibration.jsonl`
2. **Extract from this run:**
   - Persona-pool quality: did the personas have product context? Did any score below 5/10? → technique entry
   - Red Team adversary corroboration: when 2+ adversaries flagged the same FATAL, that's a high-signal pattern
   - Pre-mortem verifier verdict: PASS / RETRY / ACCEPT_WITH_NOTE → reinforces or contradicts pre-mortem-prompt techniques
   - Sim vs Miro agreement: agreement at >0.85 reinforces the persona pool; disagreement contradicts it
   - Resume protocol used: if a checkpoint resume succeeded, reinforce "checkpoint snapshots survive compaction"
3. **Reinforce or create** muscle entries (same rules as /hive Step 10d):
   - Match found + score ≥ 7.0: reinforce (`strength = min(1.0, strength + 0.2*(1-strength))`)
   - Match found + score < 6.0: contradict (`strength = max(0.0, strength - 0.3)`)
   - No match + score ≥ 7.0: create (strength 0.5, hits 1)
   - No match + score < 7.0: discard
4. **Append to history**: `hive-history.jsonl` entry with `task_type: "hivesim"`, `score`, `lessons` (1-2 sentence summary)
5. **Append to calibration**: one entry per persona/adversary call with reported_confidence and outcome
6. **Append to the outcome ledger** (reality feedback loop, feeds Phase 0b): add a row to `~/.claude/hivesim-outcomes.jsonl`:
   `{run_id, date, question, verdict: GO|NO-GO|CONDITIONAL, confidence, kill_criteria: [...], review_after, status: "pending"}`.
   Set `review_after` from the kill criteria's natural horizon; default 60 days out if the criteria imply none. Every brief becomes a scored prediction, not a fire-and-forget document.
7. **Run consolidation if entry count > 50** (same as /hive Step 10d)

**Exit gate (mirror /hive Step 10e):** Before returning the Decision Brief, log:
```
[HIVESIM EXIT GATE]
hive-history.jsonl       → +1 entry (task_type=hivesim, score={X})
hive-calibration.jsonl   → +{N} entries (personas + adversaries)
hive-muscle.jsonl        → +{K}/-{C} (new/contradicted)
hivesim-outcomes.jsonl   → +1 pending prediction (review after {date})
hivesim-reports/         → wrote {filename}
```

If any write fails, surface it: `[HIVESIM EXIT GATE FAIL] {file}: {reason}`.

The user-facing final message MUST end with one line:
```
Self-improvement: history+1, calibration+{N}, muscle+{K}/-{C}.
```

This matches /hive's exit-gate confirmation so the loop is visible across both skills.

## Execution Rules

1. **Concurrency**: Proven ceiling: 15 mixed, 20 write, 25 read. Sim personas in waves of 10-15 at low effort (proven stable). Keep high-effort phases (red team, synthesis) to 2-3 concurrent. NOTE: these ceilings are prior-model-generation numbers — see the hive skill's "STALE-DATA NOTICE". The flat `total_agents <= 13` safety rule is model-independent and still holds; refresh the rest via a hive calibration run. On the Workflow path the runtime's own cap (min(16, cores-2)) makes manual batching unnecessary unless API 529s reappear.
2. **Models/effort**: Do NOT pin model names (hardcoded model tiers go stale and pin agents to a weaker generation). Omit `model` so agents inherit the session model, and scale with `effort` instead: low for sim personas and mechanical checks, high for Red Team, FATAL verification, Pre-Mortem, and final synthesis. Red Team adversaries and personas are business roles, so they stay `general-purpose`. The exception: if a scenario hinges on a technical-feasibility call, spawn Adversary 2 (execution critic) as `feature-dev:code-architect` so it reasons against real build constraints. See the hive skill's "Agent type selection" section for the full specialized-`subagent_type` map.
3. **Context management**: Each phase writes to `.hive/findings/hivesim-{RUN_ID}-phase{N}-{name}.json` (see Compaction Resilience section). Next phase reads previous findings from disk, not memory.
4. **Checkpoints**: Snapshot after every phase to `.hive/checkpoints/hivesim-{RUN_ID}-snapshot.json`. Resume-able with `/hivesim --resume`.
5. **Budget**: A full /hivesim run uses significant context. Warn user if running on a short context budget.
6. **No fabrication**: All numbers must come from the simulation, not be made up. If data is uncertain, say so with confidence intervals.
7. **No em dashes**: Never use em dashes in output. Use commas, periods, or restructured sentences.
8. **History**: Append to `~/.claude/hive-history.jsonl` with `task_type: "hivesim"` after completion.
9. **Shared findings**: Use Hive's stigmergy pattern. Each phase writes to its own file, orchestrator merges between phases.
10. **Lesson-to-patch rule**: If a run produces an integrity lesson that implicates this skill's script or pipeline, patch THIS FILE in the same session and tick the two-path equivalence checklist. A memory entry alone is not a fix; a past attribution bug sat live for four days that way.
11. **Grade the grader**: Phase 0b outcome reviews are the only ground truth this engine gets. Never skip a due review to save time, and never self-grade; only the user's answer counts.

## Adaptive Depth

Not every question needs all 7 phases at full intensity:

| Question Complexity | Personas | Scenarios | Red Team | Pre-Mortem |
|---|---|---|---|---|
| Quick check ("should I raise prices 10%?") | 8 | 2 | 1 combined adversary | Skip |
| Standard ("enter a new vertical?") | 12 | 3 | 3 specialized adversaries + 1 synth | 1 agent + 1 verifier |
| High stakes ("pivot the whole company?") | 15 | 5 | 3 specialized adversaries + 1 synth + 1 reviewer | 1 agent + 1 verifier + 1 reviewer |

Auto-detect based on the question scope. If the scope is genuinely ambiguous (the question could be a cheap gut-check or a bet-the-company call), gate via the `AskUserQuestion` tool before spending the budget — header "Depth", options "Quick check (~40K, 8 personas)", "Standard (~80K, 12 personas)", "High stakes (~120K, 15 personas)". Otherwise auto-select and default to Standard when unsure.

## Output Location

Save the full Decision Brief to: `~/.claude/hivesim-reports/{date}-{slugified-question}.md`

Start the file with machine-readable YAML frontmatter so the outcome loop and future Phase 0 callbacks can parse briefs deterministically instead of prose-scraping:

```yaml
---
run_id: hivesim-...
question: <verbatim>
verdict: GO | NO-GO | CONDITIONAL
confidence: 0.30-0.95
kill_criteria: ["...", "..."]
review_after: YYYY-MM-DD
reality_anchors: <count of [observed] figures used; 0 means UNANCHORED>
---
```

Also display the Decision Brief directly in the terminal.

## Confidence Calculation

The Decision Brief includes a confidence percentage. Derive it as follows:

```
base_confidence = (sim_conversion_spread < 15%) ? 0.8 : 0.6
red_team_penalty = (FATAL_count * -0.15) + (SERIOUS_count * -0.05)
agreement_bonus = (sim_recommendation == miro_recommendation) ? +0.1 : -0.1
confidence = clamp(base_confidence + red_team_penalty + agreement_bonus, 0.3, 0.95)
```

- If Sim and Miro agree and Red Team found no FATAL issues: ~80-90%
- If they disagree or FATAL issues exist: ~40-60%
- Never display 95%+ (false precision) or below 30% (if it's that uncertain, recommend NO-GO)

## Resume (--resume)

Snapshots are written to `.hive/checkpoints/hivesim-{RUN_ID}-snapshot.json` after every phase, plus a stable copy at `.hive/state/latest-hivesim-snapshot.json` for easy lookup. See Compaction Resilience for details.

When `/hivesim --resume` is called:
1. Read `.hive/state/current-hivesim-run.txt` for the active run ID (if absent, fall back to `ls -t .hive/checkpoints/hivesim-*-snapshot.json | head -1`)
2. Load `.hive/state/latest-hivesim-snapshot.json` for orchestrator state
3. Read all `.hive/findings/hivesim-{RUN_ID}-phase*.json` files for completed phase outputs
4. Display what was completed and what remains
5. Skip to the next uncompleted phase (`next_action` in the snapshot)
6. Re-run Phase 1 (Scout) if checkpoint is >48 hours old (intel may be stale)
7. Re-run Phase 0 (Memory Callback) if any new hivesim reports landed since the snapshot

If no checkpoint exists, warn the user and start fresh.

## Error Handling

| Failure | Action |
|---|---|
| Phase 1 web search returns nothing | Proceed with memory-only context. Note "UNGROUNDED: no fresh intel available" in the Decision Brief. |
| Phase 3 scenarios within 5% of each other | Flag as "INCONCLUSIVE: scenarios are statistically equivalent." Red Team should focus on timing/positioning rather than which scenario wins. |
| Phase 5 Red Team flags FATAL | Verify first (Phase 5b: one skeptic per FATAL vs live product/repo/memory). A verified FATAL makes the recommendation CONDITIONAL or NO-GO and leads the brief; a refuted FATAL is demoted to context. |
| Sim and Miro contradict each other | Note the contradiction explicitly in the Decision Brief. Spawn a tiebreaker agent that reads only the two conflicting conclusions and rules on which has stronger evidence. |
| Agent timeout or failure | Log the failure, skip the agent, note reduced confidence in the Brief. Do not retry more than once. |
| Context budget running low | Complete current phase, save checkpoint, output a partial Decision Brief with what's available, note "PARTIAL: phases {X-Y} not completed due to context limits." |

## Context Budget Estimate

| Depth | Personas | High-effort calls | Estimated Total |
|---|---|---|---|
| Quick check | 8 | 1 (synthesis only) | ~40K tokens |
| Standard | 12 | 3 (red team + pre-mortem + synthesis) | ~80K tokens |
| High stakes | 15 | 4 (2x red team + pre-mortem + synthesis) | ~120K tokens |

Phase 5b FATAL verification adds one high-effort call per FATAL finding (typically 0-3).

Warn the user before starting if context is above 50% used. At 70%+ context, auto-downgrade to Quick check depth.

## Compaction Resilience (durable state, not blocking)

**Old approach (deprecated):** A lock file told a pre-compact hook to block /compact mid-run. This was wrong: locks orphan across sessions, blocking compaction is hostile when context is genuinely full, and context can hit hard limits anyway.

**New approach:** Assume compaction WILL happen. Make state survive it. Same pattern as /hive — durable state on disk.

### 1. Initialize a run directory (replaces the lock)

```bash
RUN_ID="hivesim-$(date '+%Y%m%dT%H%M%S')-$$"
mkdir -p .hive/{checkpoints,findings,state}
echo "$RUN_ID" > .hive/state/current-hivesim-run.txt
```

No lock file. No PreCompact hook integration. The run ID and findings on disk are the only durable state.

### 2. Phase outputs go to disk as JSON (not memory)

Every phase writes its full output to disk **as soon as it completes**, before the orchestrator does anything else with it. Compaction can happen between phases.

```
.hive/findings/hivesim-{RUN_ID}-phase0-memory-callback.json
.hive/findings/hivesim-{RUN_ID}-phase1-intel-brief.json
.hive/findings/hivesim-{RUN_ID}-phase2-simulation-plan.json
.hive/findings/hivesim-{RUN_ID}-phase3-market-data.json
.hive/findings/hivesim-{RUN_ID}-phase4-deep-insights.json
.hive/findings/hivesim-{RUN_ID}-phase5-red-team.json
.hive/findings/hivesim-{RUN_ID}-phase6-pre-mortem.json
.hive/findings/hivesim-{RUN_ID}-phase7-decision-brief.json
```

Each file is append-only; never overwrite once written.

### 3. Pre-compaction snapshot (proactive handoff)

At ~65% context usage OR after each phase completes, write a snapshot:

```json
{
  "run_id": "hivesim-...",
  "started_at": "...",
  "depth": "Quick|Standard|HighStakes",
  "question": "<original user question verbatim>",
  "completed_phases": [0, 1, 2],
  "current_phase": 3,
  "phase_outputs": {
    "0": "<path>", "1": "<path>", "2": "<path>"
  },
  "in_flight_agents": [],
  "next_action": "Run Phase 3 sim with 12 personas across 3 scenarios",
  "user_constraints": ["no em dashes", "honest reporting"]
}
```

Save to `.hive/checkpoints/hivesim-{RUN_ID}-snapshot.json` AND `.hive/state/latest-hivesim-snapshot.json`.

### 4. Resume protocol

If a session starts with a `hivesim-{RUN_ID}` directory and an unfinished snapshot:
1. Read `.hive/state/current-hivesim-run.txt` for run ID
2. Read `.hive/state/latest-hivesim-snapshot.json` for orchestrator state
3. Read all `.hive/findings/hivesim-{RUN_ID}-phase*.json` files
4. Resume from `next_action`. Do NOT re-run completed phases — persona votes are expensive.

`/hivesim --resume` follows the same protocol.

### 5. Persona durability

Each persona must write its decision to disk before returning, same pattern as /hive agents:

```
Before you return your final summary, write your full result to:
  .hive/findings/hivesim-{RUN_ID}-phase{N}-persona-{ID}.json

Format: {"persona_id":"...","scenario":"...","decision":"BUY|MAYBE|PASS","reasoning":"...","confidence":0.0-1.0,"ts":"..."}

Do this even if the orchestrator never asks. If conversation gets compacted, this file is the only way your vote survives.
```

### 6. Cleanup (on confirmed completion only)

When the Decision Brief is delivered:

```bash
mv .hive/state/current-hivesim-run.txt .hive/state/last-hivesim-run-$(date '+%Y%m%dT%H%M%S').txt
```

Keep `.hive/findings/hivesim-*` and `.hive/checkpoints/hivesim-*` — they're the audit trail and feed Phase 0 memory callback on future runs.

If the run aborts (Phase 5 FATAL, fatal error, user interrupt), do NOT delete the run state — leave it so the next session can offer to resume.
