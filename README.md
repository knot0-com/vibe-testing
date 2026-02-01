# Vibe Testing

**Pressure-test your specs with LLM reasoning before writing code.**

Vibe testing is a technique for validating specification documents by simulating real-world scenarios against them. An LLM reads your spec docs, traces through a concrete user scenario step by step, and flags every gap, conflict, and ambiguity — before anyone writes a line of implementation.

## Why

We test code obsessively. Unit tests, integration tests, E2E tests. But specifications? We "review" them in a meeting.

Vibe testing moves the discovery of design flaws to the cheapest possible moment: before implementation begins.

## How It Works

```
1. Write a scenario: a named persona, a concrete goal, step-by-step interaction
2. Give an LLM all your spec docs + the scenario
3. The LLM traces each step, identifies governing specs, flags gaps
4. You get a structured gap report with severity ratings
```

No code. No test harness. Just reasoning.

## Install as Agent Skill

Works with Claude Code, OpenAI Codex, Gemini CLI, Cursor, GitHub Copilot, OpenCode, and any tool supporting the Agent Skills open standard.

### Claude Code

```bash
git clone https://github.com/knot0-com/vibe-testing.git ~/.claude/skills/vibe-testing
```

### OpenAI Codex

```bash
git clone https://github.com/knot0-com/vibe-testing.git ~/.codex/skills/vibe-testing
```

### Gemini CLI

```bash
git clone https://github.com/knot0-com/vibe-testing.git ~/.gemini/skills/vibe-testing
```

### Universal (works with most agents)

```bash
git clone https://github.com/knot0-com/vibe-testing.git ~/.agent/skills/vibe-testing
```

### Project-level (shared with team)

```bash
git clone https://github.com/knot0-com/vibe-testing.git .claude/skills/vibe-testing
```

## Usage

Once installed, the skill activates when you ask your coding agent to validate specs:

```
> /vibe-testing

> "Test my specs against a realistic scenario"

> "Find gaps in the architecture docs before we start building"

> "Vibe test the design docs in docs/v2/"
```

## What's Included

```
vibe-testing/
├── SKILL.md                          # The skill definition (Agent Skills standard)
├── references/
│   └── simulator-prompt.md           # Copy-paste prompt templates
└── examples/
    └── example-vibe-test.md          # Complete example: e-commerce checkout flow
```

## The Gap Report

Vibe tests produce a structured gap report:

| Severity | Meaning |
|----------|---------|
| **BLOCKING** | Spec cannot answer. Implementation impossible without resolution. |
| **DEGRADED** | Workaround exists but it's fragile. |
| **COSMETIC** | Missing convenience. Not a correctness issue. |

## Example

The included example tests an e-commerce checkout against specs for auth, payments, inventory, orders, notifications, and shipping. A single scenario — "first-time buyer, payment declined, retries with new card" — found:

- **Payment retry timing exceeds inventory hold** — stock can be sold to another customer while the buyer is entering a new card number
- **Auth token expires mid-checkout** — 15-minute JWT TTL vs. potentially longer checkout flow on slow connections
- **Payment succeeds but order confirmation fails** — customer is charged with no order record (no saga/compensation defined)
- **Guest checkout order access undefined** — no spec for how a guest views their order status

Each would have been a rewrite-level discovery weeks into implementation.

## License

MIT
