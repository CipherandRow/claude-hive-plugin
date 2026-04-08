# Hive v4.0: Bio-Inspired Swarm Orchestrator for Claude Code

A multi-agent orchestration skill that coordinates parallel Claude Code agents using mechanisms from ant colonies, honeybee swarms, and AI research.

## What It Does

`/hive <task>` breaks your task into subtasks, spawns agents in parallel waves, coordinates them through shared findings, resolves conflicts, and learns from each run.

## What's New in v4.0 (April 8, 2026)

- **Concurrency unlocked**: Old limit was 2 heavy agents (from a March 2026 incident). Stress-tested to 25 read / 20 write / 15 mixed agents with zero failures across 161 launches. Default start concurrency raised from 5 to 10.
- **Mechanism audit**: Honest assessment of all 27 mechanisms. 10 fire regularly, 17 are theoretical/broken. Dead mechanisms documented, not removed (platform may support them in future).
- **Cross-skill sync**: Updated concurrency limits across /hivesim, /sim, /gtm, /hivefront. All skills now use proven ceilings.
- **Hivesim personas**: Now spawn 10-15 personas per wave instead of 2. ~4x faster wall clock time for strategic analysis.

## Performance (April 2026 benchmarks)

| Scenario | Agents | Wall Time | Speedup vs Sequential |
|----------|--------|-----------|----------------------|
| 10-file audit | 10 haiku | ~60s | 8x |
| 15-fix code wave | 2 opus + 4 sonnet + 9 haiku | ~200s | 4.4x |
| 109-file catalog | 5 waves x 25 haiku | ~3 min | 18x |
| 20-file write batch | 20 haiku | ~73s | 10x |

## Proven Concurrency Tiers

| Workload | Config | Agents | Proven Time |
|----------|--------|--------|-------------|
| Quick audit | 10 haiku | 10 | ~60s |
| Security scan | 1 opus + 8 haiku | 9 | ~90s |
| Fix wave | 2 opus + 4 sonnet + 9 haiku | 15 | ~200s |
| Full catalog | 25 haiku | 25 | ~50s |
| Write batch | 20 haiku | 20 | ~73s |

Mean latency does NOT degrade with more agents. Tail latency (spread) widens at scale (3x at 25 agents).

## Installation

```
/plugin install hive@claude-plugins-official
```

Or manually copy the skill into your `~/.claude/skills/hive/` directory.

## Usage

```
/hive research the top 5 competitors and compare pricing
/hive fix all failing tests in src/
/hive --dry-run refactor the auth module
/hive --resume
/hive --isolate update all API endpoints to v2
```

### Flags

| Flag | Description |
|------|-------------|
| `--resume` | Resume from last checkpoint |
| `--isolate` | Force git worktree isolation for all agents |
| `--dry-run` | Show execution plan without launching agents |
| `--verbose` | Show mechanism trace after each wave (default: on) |
| `--quiet` | Suppress execution trace |

## How It Works

1. **Mode Detection**: Counts subtasks, selects Lite/Standard/Full mode
2. **Strategy Selection**: Reads past run history, picks best strategy
3. **Planning**: Breaks task into waves with dependency ordering and reserve pool
4. **Pre-flight**: Context ceiling check, confidence calibration, agent TTL calculation
5. **Execution**: Spawns agents in waves with stigmergy (shared findings board), velocity scaling, and error handling
6. **Synthesis**: Merges results using decision protocols (vote, consensus, or AAD)
7. **Learning**: Appends scored results to history, updates playbook with time-decay

## Mechanisms

### Active (fire regularly, proven in production)

| # | Mechanism | Origin | Purpose |
|---|-----------|--------|---------|
| 2 | Self-Validation Gates | Leaf-cutter ants | Agents validate own output before returning |
| 3 | Reasoning Tree Conflicts | AgentAuditor | Find exact divergence point in reasoning |
| 4 | Stigmergy | Ants | Shared findings for indirect coordination |
| 6 | Semantic Quorum | Honeybees + LLM | Early commitment via semantic similarity |
| 7 | Scout Retirement (TTL) | Honeybees | Kill stuck agents proactively |
| 11 | Cross-Inhibition | Honeybee stop signals | Competing proposals dampen each other |
| 13 | Assembly Line QC | Leaf-cutter ants | Pipeline handoff quality gates |
| 15 | Adaptive Mode | Ant response thresholds | Auto-scale mechanisms to task size |
| 17 | Auto-Verify Gate | Manufacturing QC | Run tsc + tests after code-writing waves |
| 27 | Debug Trace | Distributed tracing | JSONL event log for every decision |

### Theoretical (defined but rarely/never fire in current Claude Code)

| # | Mechanism | Why Inactive |
|---|-----------|-------------|
| 1 | Pheromone Evaporation | History is read but scoring never changes agent routing |
| 5 | Completion Velocity | Concurrency is now fixed at proven tiers, not dynamically scaled |
| 8 | Decision Protocols | Vote/consensus/AAD never explicitly activates as distinct step |
| 9 | Swarm Playbook | No evidence of playbook influencing future runs |
| 10 | Ready-Up Signal | Pre-flight exists but not as a distinct mechanism |
| 12 | Inspector Agents | Never fires in production |
| 14 | Checkpoint/Resume | Defined but never used across 50 runs |
| 16 | Worktree Isolation | Broken on Windows (file delivery fails on cleanup) |
| 18-26 | Advanced mechanisms | Platform doesn't yet support (JSON schema enforcement, dynamic tool loading, hot reassignment, etc.) |

These remain in the skill definition for forward compatibility. As Claude Code and the Agent SDK evolve, more mechanisms may activate.

## Model Selection (data-driven)

| Model | Avg Duration | Avg Tool Calls | Best For |
|-------|-------------|----------------|----------|
| Haiku | 7-29s | 1-5 | File reads, grep, single-file fixes, summaries |
| Sonnet | 85-95s | 20-25 | Multi-file fixes, code review, medium complexity |
| Opus | 94-115s | 10-12 | Complex rewrites, architecture decisions, synthesis |

## Prompt Discipline (proven by A/B test)

Strict prompts beat loose prompts in every dimension:
- 44% faster per-agent execution
- 10x less code bloat
- 0 scope violations vs 1

Always include in write agent prompts:
```
ONLY EDIT THIS FILE: [path]. DO NOT edit any other file.
DO NOT add inline comments, refactor, or change code outside the task.
```

## Requirements

- Claude Code CLI
- Git (for worktree isolation, optional)

## Version History

| Version | Date | Changes |
|---------|------|---------|
| v4.0 | April 8, 2026 | Concurrency unlocked (2->25), mechanism audit, cross-skill sync |
| v3.0 | March 25, 2026 | 14 mechanisms, semantic quorum, reasoning tree, A2A protocol |
| v2.0 | March 2026 | Adaptive mode, worktree isolation, stigmergy |
| v1.0 | March 2026 | Basic parallel agent orchestration |

## License

MIT
