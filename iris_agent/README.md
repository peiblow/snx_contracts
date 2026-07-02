# Iris — Personal Decision Keeper

Iris is a personal agent governed by Synx. She helps the user **capture, recall,
and act on decisions**, stored as Markdown notes in an Obsidian vault. Every read
and write goes through the Synx gate, which decides `APPROVED`/`DENY`
deterministically — Iris only acts when approved.

Two channels define how the world talks to her:

- **WhatsApp** — the user chats with Iris by message (inbound and reply).
- **Vault (filesystem)** — Iris reads and writes decision notes in Obsidian.

She is also **proactive**: a `cron` trigger has her review recent decisions every
morning, without the user asking.

> The `iris_agent.snx` here is the **final picture** of the contract. The VVM does
> not parse `trigger` or `skills` yet — those blocks document where the language
> is heading.

## Layout

| File | Kind | Contents |
|---|---|---|
| `iris_agent.snx` | contract | Imports + the `agent Iris` declaration (model, behavior, triggers, skills, tools) |
| `iris_types.snx` | library `IrisTypes` | `type DecisionNote` |
| `iris_decisions.snx` | library `IrisDecisions` | `readDecisions`, `recordDecision` |
| `iris_files.snx` | library `IrisFiles` | `readFile` |
| `iris_messaging.snx` | library `IrisMessaging` | `sendWppMessage` |
| `prompts/iris.md` | prompt | Iris's system prompt |
| `skills/*.md` | skill | Playbooks Iris loads on demand |

## How to read `agent Iris`

The block reads top to bottom like a sentence:

```
identity    → who she is          (version, owner, purpose)
model       → how she thinks      (provider, name, temperature, maxTokens)
behavior    → how she behaves     (systemPrompt, maxSteps, onDeny, onError)
trigger ×3  → how she wakes up    (http, filesystem, cron)
skills  ×2  → what she knows      (playbooks + the tools each one uses)
tool    ×4  → what she can touch  (the policy + the HTTP/filesystem action)
```

## Keywords

### Contract identity

| Keyword | What it is |
|---|---|
| `contract` | Compilation unit. Groups an `agent` with its functions and types. |
| `agent` | The governed agent. Has identity, model, behavior, triggers, skills, and tools. |
| `version` | Semantic version of the agent. |
| `owner` | Address (`0x…`) of the agent's owner. |
| `purpose` | Short purpose label; goes into the agent metadata. |

### `model` — how the LLM is called

| Keyword | What it is |
|---|---|
| `provider` | Model provider (`anthropic`, `groq`, `openai`, …). |
| `name` | Model name at the provider. |
| `temperature` | Generation randomness (0 = deterministic). |
| `maxTokens` | Cap on response tokens. |

### `behavior` — the agent loop

| Keyword | What it is |
|---|---|
| `systemPrompt` | Path to the `.md` holding the system prompt. |
| `maxSteps` | Max steps (tool calls) per turn; guards against infinite loops. |
| `onDeny` | What to do when the gate denies. `reflect` = reassess before retrying. |
| `onError` | What to do on execution error. `abort` = stop the turn. |

### `trigger` — how the world wakes Iris

Each `trigger` has an **infra** half (where/when to listen) and an **admission**
half (what it lets into the loop, before spending LLM cycles).

| Keyword | Trigger | Half | What it is |
|---|---|---|---|
| `type` | all | — | Trigger type: `http`, `filesystem`, or `cron`. |
| `route` | http | infra | Registered HTTP route (`/agent/{IRIS_HASH}/events`). |
| `method` | http | infra | HTTP verb (`POST`). |
| `auth` | http | admission | How to prove the webhook's origin. `hmac` = signed body. |
| `secret_ref` | http | admission | **Points** to the secret (env name), never the literal value. |
| `path` | filesystem | infra | Watched folder. |
| `ops` | filesystem | admission | Which operations wake Iris (`create`, `write`). |
| `recursive` | filesystem | infra | Descend into subfolders? Defaults to `false`. |
| `debounce` | filesystem | admission | Wait before reacting; coalesces repeated writes. |
| `pattern` | filesystem | admission | Glob of files that matter (`*.md`); filters out noise. |
| `schedule` | cron | infra | Schedule in 5 fields (`min hour day month weekday`). |
| `timezone` | cron | infra | Schedule timezone. Without it, runs in UTC. |
| `overlap` | cron | admission | What if the previous run hasn't finished? `skip` = skip it. |

**Why `secret_ref` and not `secret`:** the contract is versioned by hash and
auditable. A literal secret inside it would leak into the repo and the journal.
The contract points to *where* the secret lives; the value stays in the env,
outside the artifact.

### `skills` — playbooks loaded on demand

`skills` is an array. Each entry is a named capability: a `.md` with the "how to"
plus the list of tools it draws on.

| Keyword | What it is |
|---|---|
| `name` | The skill's name. |
| `content` | Path to the `.md` with the instructions (the playbook). |
| `uses` | The tools the skill may invoke (referenced by tool name). |

### `tool` — what Iris can touch

A `tool` joins **the policy Synx evaluates** (the `function`) with **the action to
run once approved** (the `action`).

| Keyword | What it is |
|---|---|
| `tool` | A capability exposed to the model (name + description + steps). |
| `description` | Text the model reads to decide when to use the tool. |
| `steps` | Ordered list of steps; each step has a `function` and, optionally, an `action`. |
| `function` | The library function Synx runs to decide `APPROVED`/`DENY`. |
| `input` | The shape of the arguments the model must produce. |
| `action` | The real call executed **after** approval (`filesystem` or `http`). |
| `operation` | On a filesystem action: `read` or `write`. |
| `method` / `url` | On an http action: verb and destination. |

### Libraries and builtins

| Keyword | What it is |
|---|---|
| `import` | Pulls a library into the compilation unit (FLAT model, no prefix). |
| `library` | Module of reusable functions and types. |
| `type` | Declares a structured input type (e.g. `DecisionNote`). |
| `fn` | A policy function; takes a typed input and emits an `Event`. |
| `require` | Validates a condition; if it fails, raises the given `Error`. |
| `Error` | Builds an error with `code` + `message`, caught in a `try/catch`. |
| `emit` | Emits a structured event (the observable result of the function). |
| `getEnv` | Reads an environment variable at runtime (never a secret in the contract). |

## Iris's tools

| Tool | Policy (`function`) | Action after approval |
|---|---|---|
| `read_decisions` | `IrisDecisions.readDecisions` | filesystem `read` on `DECISIONS_DIR` |
| `save_decision` | `IrisDecisions.recordDecision` | filesystem `write` in the vault |
| `read_file` | `IrisFiles.readFile` | filesystem `read` on the given path |
| `send_wpp_message` | `IrisMessaging.sendWppMessage` | http `POST` on `WPP_SEND_URL` |

## Iris's skills

| Skill | Playbook | `uses` | Ties to |
|---|---|---|---|
| `decision_logging` | `skills/decision_logging.md` | `read_decisions`, `save_decision`, `send_wpp_message` | reactive flow (WhatsApp) |
| `morning_review` | `skills/morning_review.md` | `read_decisions`, `send_wpp_message` | the 08:00 `cron` trigger |
