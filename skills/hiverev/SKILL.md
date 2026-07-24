---
name: hiverev
description: Persona-driven review of anything, from any angle. Point it at a UI, a landing page, a cold email, a README, a pitch deck, an onboarding flow, an API design, or any artifact, pick who is looking at it and what lens they bring, and get a prioritized friction punch list. Lighter than /hivesim. Triggers when user says /hiverev, "review this as [persona]", "what would [persona] think", "find friction in this", or "critique this from the angle of [X]".
user_invocable: true
---

# /hiverev — Persona-Driven Review (any artifact, any angle)

A focused critique skill. Spawns no sub-agents. Runs persona-driven review in-context on whatever you point it at: a UI surface, a document, marketing copy, an email, a deck, a flow, an API design. Returns a prioritized punch list with severity ratings.

The core move is always the same: pick WHO is looking at the artifact and WHAT ANGLE they bring (a skeptical first-time visitor judging trust, a busy operator judging usability, an investor judging the story, a new developer judging the docs, a hostile competitor judging the positioning), then walk the artifact as that persona and report only the friction that survives a red team.

## When to use /hiverev vs /hivesim

- **/hiverev** — Persona-driven friction-finding on an existing artifact. Asks "what would this specific person actually do or think here?" Returns a fix list.
- **/hivesim** — Strategic go/no-go for big decisions (pricing, market entry). Spawns 15 buyer agents. Heavyweight.

Use /hiverev when the question is "does this artifact work on its audience", not "should this thing exist."

## Inputs (prompt should include)

1. **Target** — anything reviewable: a Paper artboard ID, a screenshot path, a live or localhost URL, a component/file path, a document, or pasted text. At least one.
2. **Persona + angle** — who is looking at it, and through what lens? Examples: "Dana the dispatcher judging day-1 usability", "a skeptical CFO reading this pricing page", "an investor skimming this deck in 3 minutes", "a new hire following this README", "a tired customer reading this cold email on their phone." Default to "the actual primary audience of the artifact" if not specified.
3. **What "good" means** — what would success look like for this persona on this artifact? E.g. "Dana completes onboarding without getting stuck or asking support for help", "the CFO forwards the email instead of deleting it."

If any input is missing, ask the user before proceeding. Don't guess the persona.

## Pipeline (lightweight, 5 phases, all in-context — no sub-agents)

### Phase 0: Prior-review callback

Before reviewing, check `~/.claude/hiverev-reports/` for earlier reviews of the same target (match on the slug). If one exists:

1. Read the latest one's findings (they carry stable IDs, see Phase 5).
2. Classify each prior P0/P1 against the current artifact: `fixed`, `still-open`, or `regressed`.
3. Re-read its "What's working" list; if something that was working is now broken, that is automatically a P1 (regressions of known-good outrank new polish).
4. The new report includes a "vs last review" section with those classifications.

A review that cannot remember its own past findings re-litigates instead of compounding. Skip silently only when no prior report exists.

### Phase 1: Ground the persona (mandatory)

Before judging anything, build the persona properly:

1. Read the project's product docs (README, CLAUDE.md, a product vision note — wherever the project keeps them) to know what the product or artifact is actually for.
2. Check project memory or notes for persona-specific context (e.g. "the pilot customer runs a small fleet, is skeptical of AI, and is loyal to the tool they use today").
3. If the persona is a real person (named pilot customer, a specific recipient), look for any project notes about them.
4. Write a 3-bullet "Persona brief" that captures:
   - What they want to accomplish (or what they're deciding while looking at this)
   - What they bring (skills, prior tools, mental models, how much time and attention they'll give it)
   - What they fear or distrust

A persona that doesn't know the product will produce garbage critiques. Skip this and the review is worthless.

### Phase 2: Capture the artifact

Match the capture method to the target:

- **Paper artboard**: first confirm the Paper MCP server is actually connected (ToolSearch for `mcp__paper__`); if it is not, ask for a screenshot or fall back to Playwright against the running app. When connected, call `mcp__paper__get_screenshot` with the node ID. If tree summary is needed first, use `mcp__paper__get_node_info`.
- **Live or localhost URL**: use Playwright `browser_navigate` + `browser_take_screenshot`. Capture multiple scroll positions if the page is long.
- **Screenshot / image / PDF**: use the Read tool.
- **Document, email draft, deck outline, or pasted text**: read it in full. Note anything the persona would see that the text alone doesn't carry (subject line, sender, channel, formatting) and ask for it if it matters.
- **Component or code file (e.g. TSX)**: FIRST try to look at rendered truth, in this order: (a) a dev server already running (Playwright to the route), (b) the production URL if the component is live, (c) a screenshot from the user. Only if all three are unavailable, review the code as written, and then two hard rules apply: the TL;DR must open with "(code-intent review, not rendered)" and confidence is capped at 60%. A persona cannot bounce off code; findings about spacing, hierarchy, or affordance from unrendered code are hypotheses, not observations.

The general rule: review what the audience would actually experience, in the form they'd experience it. The further the captured form is from that, the lower the confidence cap.

### Phase 3: Walk the artifact as the persona

**Live-signal anchor (when available):** Before walking, check whether real audience behavior exists for this artifact. If the target is a live product surface and the PostHog MCP is connected, pull the cheap high-signal queries: rage clicks, dead clicks, drop-off on the route, session replays flagged for it. For other artifact types, use whatever real signal exists (email reply rates, doc search queries, support tickets quoting the page). Real friction outranks imagined friction: a finding backed by live data is tagged `[observed]` and cannot be dropped by the Phase 4 red team as "subjective taste". If no live data exists (pre-launch, draft, artboard), proceed purely persona-driven; the report notes "persona-only review, no live signals".

This is where the actual review happens. Walk through the artifact the way the persona would. Three passes, adapted to the artifact:

1. **First-glance pass (5 seconds):** What does the persona see or read first? Where do their eyes go? What do they think this is for? Do they get the value prop, the ask, or the point? For an email: subject line + first two lines. For a deck: the title slide and slide 2. For a UI: the viewport before scrolling.
2. **Engagement pass:** What does the persona try to do, or want to know, next? Where do they click, what do they skim to, what question forms in their head? Where does the artifact diverge from that expectation? For copy and docs, this is the comprehension pass: where do they stumble, reread, or misread?
3. **Edge pass:** What happens when the happy path breaks? Empty states, error states, slow loading, missing permissions for a UI. Objections, "sounds too good", "what's the catch", deletion-finger for an email or landing page. Missing prerequisites, version drift, "step 3 doesn't work on my machine" for docs.

For each pass, note specific friction points with **exact location** (e.g. "the 'Approve & send' button in the Today card", "the second sentence of paragraph 3") and **why it's a problem for this persona** (e.g. "this persona is skeptical of automation making decisions for them, and this copy implies the system is proposing while they rubber-stamp").

### Phase 4: Red team the review itself

Before writing the punch list, attack your own findings:

- Which findings are subjective taste vs. real friction?
- Which findings would the persona actually mention vs. silently work around?
- Which findings are P0 (blocks the goal) vs P1 (annoying) vs P2 (cosmetic)?
- Which findings have a specific, cheap fix vs. require a structural rebuild?

Drop any finding that doesn't survive the red team. A good /hiverev review has 3-7 findings, not 20.

### Phase 5: Output the decision brief

Same shape as `/hivesim` Decision Brief, scoped to this review. Save to `~/.claude/hiverev-reports/{date}-{slugified-target}.md` and display in terminal.

```
---
target: {slug}
date: {date}
persona: {name}
angle: {lens}
confidence: {0-100}
mode: rendered | code-intent | artboard | screenshot | document
live_signals: true | false
findings: [{id: "{slug}-{n}", severity: P0|P1|P2, status: open}]
---

# Review: {target}
Persona: {name} | Angle: {lens} | Date: {date} | Confidence: {%}

## TL;DR
{One sentence: "Ship as is" / "Ship after P0 fixes" / "Rework"}

## vs last review
{Only when a prior report exists: each prior P0/P1 as fixed / still-open / regressed. Regressions of "what's working" items are automatic P1s.}

## Persona brief
- Wants: ...
- Brings: ...
- Fears: ...

## Findings (prioritized)

### P0 — Blockers ({n})
1. **{specific location}** — {what's wrong} — {why it matters for this persona} — **Fix:** {concrete change}

### P1 — Serious ({n})
...

### P2 — Cosmetic ({n})
...

## What's working
{Don't only critique. Name 2-3 things to keep doing — this is feedback memory for future passes.}

## Pre-mortem
"It's {date + 30 days}. {Persona} bounced / deleted it / churned. Why?"
{One paragraph, specific scenario.}

## Next actions
1. {Concrete action, owner, timeline}
2. ...
```

## Execution rules

1. **Stay in-context by default.** /hiverev spawns no sub-agents for a single artifact or a small set. The ONLY exception is the top tier of the Adaptive depth table (Full product or document set, 10+ pieces, 2-3 personas) under ultracode, where you MAY fan out just the capture-and-walk step per the Ultracode note. Synthesis, the red team, and the brief always stay in-context. For anything bigger, use /hivesim.
2. **Never fabricate persona reactions.** Every finding must trace to a specific element the persona would encounter and a specific reason it would fail them.
3. **Severity discipline.** P0 = blocks the goal. P1 = serious annoyance. P2 = cosmetic. Don't inflate severity to seem thorough.
4. **No em dashes** in any output. Use commas, periods, or restructured sentences.
5. **Cap output at 600 words.** Force prioritization. If the review is longer than that, the persona didn't get used as a filter.
6. **Always include "what's working"**. If you only ship corrections, the team drifts away from things that were already right. Save 2-3 wins as feedback memory.
7. **History.** Append to `~/.claude/hive-history.jsonl` with `task_type: "hiverev"` after completion.
8. **Stable finding IDs.** Findings are numbered `{target-slug}-{n}` in the report frontmatter. On a re-review, never renumber old IDs; new findings continue the sequence. IDs are what make the "vs last review" section computable instead of vibes.
9. **Experienced truth beats intent.** Never present a code-intent or outline-only finding with the same confidence as one from a rendered surface, a real document, or live data. The mode tag in the frontmatter is not optional.
10. **Lesson-to-patch rule.** If a run exposes a flaw in this skill's process or fanout script, patch this file in the same session. A memory entry alone is not a fix.

## Adaptive depth

| Artifact size | Personas | Findings cap | Time budget |
|---|---|---|---|
| Single artifact (one page, one email, one artboard) | 1 persona | 5 findings | ~5 min |
| Small set (3-5 screens, a short doc, a deck) | 1-2 personas | 8 findings | ~10 min |
| Full product or document set (10+ pieces) | 2-3 personas | 12 findings | ~20 min |

If reviewing more than 10 pieces with 3+ personas, recommend /hivesim instead. At the top tier only (10+ pieces, 2-3 personas) under ultracode you MAY fan out just the capture-and-walk step via the small Workflow in the Ultracode note below; synthesis and the brief stay in-context.

### Ultracode note

Under ultracode the Workflow tool is available, but /hiverev stays in-context by default. The whole point of this skill is no sub-agents, lighter than /hivesim. Keep it that way for a single artifact or a small set.

ONE opt-in escape hatch: only at the TOP tier of the Adaptive depth table (10+ pieces, 2-3 personas) MAY you use a small Workflow that fans out ONLY the capture-and-walk step (Phase 2 + Phase 3), one agent per persona x artifact. Phase 1 (persona brief), Phase 4 (red team the review), and Phase 5 (decision brief) stay in-context. The mapping is narrow: each persona-x-artifact pair flows through a `pipeline()` (Phase 2 capture, then Phase 3 walk, no barrier between them), each walk returning structured findings instead of free prose; the runtime's journaling and auto concurrency cap replace any hand-rolled batching. This is not the new default and never fires for one artifact. For anything bigger than 10 pieces with 3+ personas, the doc still points you to /hivesim. Do not grow /hiverev into a swarm.

```javascript
export const meta = {
  name: 'hiverev-fanout',
  description: 'Top-tier hiverev only: fan out Phase 2 capture + Phase 3 persona walk, one agent per persona x artifact. Phase 4 red team and Phase 5 brief stay in-context.',
  phases: ['Capture the artifact', 'Walk the artifact as the persona']
};

// args: { artifacts:[{id,target}], personas:[{name,brief}] }  (persona briefs built in-context in Phase 1)
// Integrity rule (hard-won attribution lesson): persona/artifact attribution is attached from the
// closure after the call resolves, never echoed back by the model.
const walkSchema = {
  type: 'object',
  required: ['findings'],
  properties: {
    findings: {
      type: 'array',
      items: {
        type: 'object',
        required: ['location', 'problem', 'pass'],
        properties: {
          location: { type: 'string' },
          problem: { type: 'string' },
          pass: { type: 'string', enum: ['first-glance', 'engagement', 'edge'] }
        }
      }
    }
  }
};

const pairs = [];
for (const a of args.artifacts) {
  for (const p of args.personas) {
    pairs.push({ artifact: a, persona: p });
  }
}

// Each pair flows independently: Phase 2 captures the artifact, Phase 3 walks it as the persona.
// pipeline() = no barrier between stages; a throwing stage drops that pair to null.
const walks = (await pipeline(
  pairs,
  (prev, pair) => agent(
    'You are ' + pair.persona.name + '. Persona brief: ' + pair.persona.brief + '\n' +
    'Phase 2 (Capture the artifact): capture ' + pair.artifact.target + '. Describe the captured state factually. Do not fabricate.',
    { label: 'capture ' + pair.persona.name + ' x ' + pair.artifact.id, phase: 'Capture the artifact' }
  ),
  (capture, pair) => agent(
    'You are ' + pair.persona.name + '. Persona brief: ' + pair.persona.brief + '\n' +
    'Captured state of ' + pair.artifact.target + ':\n' + capture + '\n\n' +
    'Phase 3 (Walk the artifact as the persona): walk it in three passes, first-glance 5s, engagement, edge. ' +
    'Return only friction findings, each with exact location, the problem, and which pass it came from. Do not fabricate reactions.',
    { label: 'walk ' + pair.persona.name + ' x ' + pair.artifact.id, phase: 'Walk the artifact as the persona', schema: walkSchema }
  ).then(w => w ? { ...w, persona: pair.persona.name, artifact: pair.artifact.id } : w)
)).filter(Boolean);

// Hand the raw walks back. Phase 4 (red team the review) and Phase 5 (decision brief,
// 3-7 findings, <=600 words) run IN-CONTEXT, exactly as the manual path below specifies.
return { walks };
```

FALLBACK: when the Workflow tool is unavailable, or for anything at or below the top tier (single artifact, small set), run all 5 phases in-context exactly as the manual Pipeline section documents. That manual in-context path is the default and always applies.

## What /hiverev is NOT for

- Strategic decisions about whether to build or ship the thing at all. Use /hivesim.
- Formal accessibility or performance audits. Use dedicated tooling for those; /hiverev can flag what the persona trips over, but it is not an audit.
- Code-level review. Use a code reviewer agent; /hiverev reviews what the audience experiences, not the implementation.

## Example invocations

```
/hiverev review the onboarding wizard in Paper (artboard 3A-1) as the pilot customer
/hiverev walk the landing page at localhost:8080 as a first-time visitor who has never heard of the product
/hiverev critique this cold email as a busy ops manager reading it on their phone between meetings
/hiverev review README.md as a new developer trying to get to a running app in 15 minutes
/hiverev read this pitch deck as a skeptical seed investor giving it 3 minutes
/hiverev compare the empty state vs the populated dashboard for a new user on day 1 vs day 14
```

## Output location

Save the full review to: `~/.claude/hiverev-reports/{date}-{slugified-target}.md`

Display the review directly in the terminal alongside the file path.
