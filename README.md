# SafeDrain

**Usage-aware graceful drain for long-running Codex / agent sessions.**

SafeDrain is a small behavioral skill that tells an agent to use the runtime's **native usage/quota capability** proactively, reduce work-unit size as quota pressure increases, and checkpoint safely before a quota window is exhausted.

It deliberately does **not** add a daemon, watchdog, custom telemetry transport, polling script, usage parser, or persistent SafeDrain state. The design principle is simple: use the capability the runtime already has, and build extra machinery only if a concrete gap is observed.

> **Status:** v0.2 beta. Tested in naturalistic Codex workflows; not a claim of universal reliability across every model, account, runtime, or provider.

## What SafeDrain does

SafeDrain instructs the active agent to:

- check all relevant usage windows exposed by the runtime;
- evaluate each window independently against its own thresholds;
- increase usage-check frequency as a window approaches its limits;
- reduce the size of newly started work units under quota pressure;
- preflight usage before long operations that may not return control for a while;
- stop starting substantive work at the configured drain threshold;
- finish only the current atomic step needed to reach a recoverable boundary;
- create/update the normal project checkpoint or handoff;
- stop immediately after a minimal recovery checkpoint at hard drain.

## Requirement

SafeDrain depends on the active runtime exposing a **native usage / rate-limit capability that the agent can query**. If the runtime cannot expose current quota information to the agent, SafeDrain does not invent an alternative telemetry stack.

If native behavior proves insufficient, the skill instructs the agent to record the concrete gap and stop rather than silently creating monitoring infrastructure.

## Install as a standalone skill

Current OpenAI Codex documentation supports user skills under:

```text
$HOME/.agents/skills
```

Copy this repository's skill directory to:

```text
$HOME/.agents/skills/safe-drain/
```

The resulting file should be:

```text
$HOME/.agents/skills/safe-drain/SKILL.md
```

On Windows PowerShell, from the cloned repository:

```powershell
New-Item -ItemType Directory -Force "$HOME\.agents\skills\safe-drain" | Out-Null
Copy-Item ".\skills\safe-drain\SKILL.md" "$HOME\.agents\skills\safe-drain\SKILL.md" -Force
```

Then invoke it explicitly in Codex with:

```text
$safe-drain
```

## Set the quota thresholds in the prompt

SafeDrain intentionally contains **no permanent universal thresholds**. Supply the thresholds for the current workflow/session.

Use this prompt template:

```text
$safe-drain

SafeDrain session thresholds:

5h window:
- monitoring/caution: <N>% remaining
- drain: <N>% remaining
- hard drain: <N>% remaining

Weekly window:
- monitoring/caution: <N>% remaining
- drain: <N>% remaining
- hard drain: <N>% remaining

[Your task here]
```

Each usage window is evaluated against **its own thresholds**. SafeDrain must not convert one window into another or apply one window's thresholds to a different window.

### Example profile: long engineering session

This profile was used in one of our naturalistic engineering workflows:

```text
$safe-drain

SafeDrain session thresholds:

5h window:
- monitoring/caution: 25% remaining
- drain: 20% remaining
- hard drain: 15% remaining

Weekly window:
- monitoring/caution: 5% remaining
- drain: 3% remaining
- hard drain: 2% remaining
```

### Example profile: intentionally aggressive low-quota finish

This profile was used for a bounded image/asset workflow where we deliberately accepted a smaller reserve:

```text
$safe-drain

SafeDrain session thresholds:

5h window:
- monitoring/caution: 8% remaining
- drain: 5% remaining
- hard drain: 3% remaining

Weekly window:
- monitoring/caution: 3% remaining
- drain: 2% remaining
- hard drain: 1% remaining
```

These are **examples, not universal recommendations**. Choose thresholds based on the cost and recoverability of the work you are about to start.

## Observed behavior in naturalistic tests

SafeDrain v0.2 has been exercised in multiple real workflows rather than only synthetic demos. Observed examples include:

- an engineering session that entered caution near the configured threshold, avoided starting new large work, and checkpointed when the drain threshold was crossed;
- an asset-generation session that stopped before launching a new expensive generation step when the configured drain threshold was reached;
- a longer asset-generation session where the agent progressively reduced parallelism/work-unit size as weekly quota fell, entered caution at 3%, then checkpointed and stopped at the 2% weekly drain threshold.

See [`docs/VALIDATION.md`](docs/VALIDATION.md) for the compact evidence summary and limitations.

## What SafeDrain is not

SafeDrain is not:

- a background daemon;
- a quota scraper;
- a UI parser;
- an event subscription service;
- an external telemetry server;
- a guarantee that the agent can be interrupted during an operation that never returns control;
- a replacement for project-specific recovery/checkpoint logic.

SafeDrain changes **agent behavior around native quota information**. Your existing project checkpoint/handoff mechanism remains the recovery authority.

## Native Capability First

SafeDrain follows a narrow design rule:

```text
need capability
→ try the native capability first
→ characterize reliability and conditions
→ identify the actual gap
→ build only the gap
```

Capability existence and reliable capability utilization are different problems. SafeDrain addresses the latter: the runtime may already know its usage state, but the agent may fail to check it proactively or fail to change behavior after a low-quota observation.

## Repository layout

```text
.
├── plugin.json
├── README.md
├── LICENSE
├── CHANGELOG.md
├── docs/
│   └── VALIDATION.md
└── skills/
    └── safe-drain/
        └── SKILL.md
```

The root `plugin.json` makes the repository ready to package as a minimal skills-only plugin if/when you want to distribute it that way. The plugin contains no MCP server or custom telemetry implementation.

## Contributing / feedback

The most useful feedback is a **concrete failure case**:

- what runtime/model was used;
- which quota windows were exposed;
- thresholds supplied;
- what SafeDrain did;
- what you expected it to do;
- whether control returned between the last successful usage check and the failure.

Please avoid proposing a daemon or custom telemetry layer unless a reproducible native-capability gap requires one.

## License

MIT. See [`LICENSE`](LICENSE).
