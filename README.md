# Hive: Bio-Inspired Swarm Orchestrator for Claude Code

A multi-agent orchestration skill that coordinates parallel Claude Code agents using mechanisms from ant colonies, honeybee swarms, and AI research.

## What It Does

`/hive <task>` breaks your task into subtasks, spawns agents in parallel waves, coordinates them through shared findings, resolves conflicts, and learns from each run.

## Key Features

- **19 orchestration mechanisms** from nature and AI research (pheromone scoring, stigmergy, semantic quorum, cross-inhibition, and more)
- **Tested to 25 concurrent agents** with zero failures (April 2026 stress test, 161 agent launches)
- **Data-driven model selection**: haiku for scouts (7s avg), sonnet for reviewers (95s avg), opus for leads (105s avg)
- **Strict prompt discipline**: proven 44% faster and 10x less code bloat than loose prompts
- **Cross-session learning**: pheromone-weighted history means strategy selection improves over time
- **Adaptive concurrency**: TCP-inspired scaling adjusts agent count based on completion velocity
- **Worktree isolation**: parallel code changes without merge conflicts
- **3 execution modes**: Lite (1-3 tasks), Standard (4-8), Full (9+) automatically scales mechanisms to task size
- **5 strategies**: wide-parallel, deep-pipeline, fan-out-gather, hybrid, iterative
- **Checkpoint/resume**: save and resume interrupted runs with `--resume`

## Performance (April 2026 benchmarks)

| Scenario | Agents | Wall Time | Speedup vs Sequential |
|----------|--------|-----------|----------------------|
| 10-file audit | 10 haiku | ~60s | 8x |
| 15-fix code wave | 2 opus + 4 sonnet + 9 haiku | ~200s | 4.4x |
| 109-file catalog | 5 waves x 25 haiku | ~3 min | 18x |
| 20-file write batch | 20 haiku | ~73s | 10x |

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
2. **Strategy Selection**: Reads past run history, applies pheromone decay, picks best strategy
3. **Planning**: Breaks task into waves with dependency ordering and reserve pool
4. **Pre-flight**: Context ceiling check, confidence calibration, agent TTL calculation
5. **Execution**: Spawns agents in waves with stigmergy (shared findings board), velocity scaling, and error handling
6. **Synthesis**: Merges results using decision protocols (vote, consensus, or AAD)
7. **Learning**: Appends scored results to history, updates playbook with time-decay

## Mechanisms

| # | Mechanism | Origin | Purpose |
|---|-----------|--------|---------|
| 1 | Pheromone Evaporation | Ants | Time-weighted history prevents strategy lock-in |
| 2 | Self-Validation Gates | Leaf-cutter ants | Agents validate own output before returning |
| 3 | Reasoning Tree Conflicts | AgentAuditor | Find exact divergence point in reasoning |
| 4 | Stigmergy | Ants | Shared findings for indirect coordination |
| 5 | Completion Velocity | Harvester ants (TCP) | Scale concurrency on completion rate |
| 6 | Semantic Quorum | Honeybees + LLM | Early commitment via semantic similarity |
| 7 | Scout Retirement (TTL) | Honeybees | Kill stuck agents proactively |
| 8 | Decision Protocols | ACL 2025 | Vote / consensus / AAD per task type |
| 9 | Swarm Playbook | Ant tandem running | Winning approaches persist across runs |
| 10 | Ready-Up Signal | Bee piping | Pre-flight verification before stages |
| 11 | Cross-Inhibition | Honeybee stop signals | Competing proposals dampen each other |
| 12 | Inspector Agents | Honeybees | Re-check rejected options against new findings |
| 13 | Assembly Line QC | Leaf-cutter ants | Pipeline handoff quality gates |
| 14 | Checkpoint/Resume | LangGraph-inspired | Save state, resume after failures |
| 15 | Adaptive Mode | Ant response thresholds | Auto-scale mechanisms to task size |
| 16 | Worktree Isolation | Termite chambers | Conflict-free parallel file writes |

## Requirements

- Claude Code CLI
- Git (for worktree isolation)

## License

MIT