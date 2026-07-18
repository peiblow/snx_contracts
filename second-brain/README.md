# Second Brain — three governed agents over an Obsidian vault

A complete, real-world Synx system: a personal knowledge assistant reachable over Telegram
that answers questions from your Obsidian vault, captures new notes on request, and
automatically enriches every capture with tags and wikilinks.

It is also a tour of the whole Synx surface: the three trigger types (`http`, `filesystem`,
and dispatch handoff), all four action types (`http`, `shell`, `filesystem`, `dispatch`),
and write isolation enforced by contract — the agent that talks to you can never write to
your vault.

## The agents

| Agent | Contract | Woken by | Model | What it does |
| --- | --- | --- | --- | --- |
| **Ask** | `ask.snx` | `http` trigger on `POST /chat` (Telegram webhook) | groq / llama-3.3-70b | Answers questions strictly from vault notes; on an explicit "save this" request, hands the note off to Writer |
| **Writer** | `writer/writer.snx` | dispatch from Ask — no trigger block at all | ollama / qwen3:8b | Persists the capture under `Captures/`. Its only tool is `write_note`, and the path is pinned to the Captures folder by the contract |
| **Enricher** | `enricher/enricher.snx` | `filesystem` trigger — `create` of `*.md` in `Captures/` | ollama / qwen3:8b | Rewrites the new capture with YAML frontmatter tags, `[[wikilinks]]` to related notes, and expanded detail |

## Flow

```
Telegram message
      │  webhook → POST /chat (EEAPI)
      ▼
    Ask ── question ──> list_notes / read_note ──> reply (Telegram)
      │
      └── "save this" ──> save (dispatch) ──> Writer ──> Captures/<note>.md
                                                              │  fs trigger (create *.md)
                                                              ▼
                                                          Enricher ──> enrich_note (overwrite in place)
```

Every tool call above crosses the gate before it executes. The isolation is topological, not
prompt-based: Ask has no write action in its contract, so no amount of prompt injection can
make it write; Writer can only write inside `Captures/` because the contract concatenates the
path itself.

## Configuration

All config is declared with `getEnv(...)` — the contracts carry only the keys, values live in
your `contract.env`. Start from the example file:

```sh
cp contract.env.example contract.env
```

(or let `synx env --scaffold .` generate the checklist for you.)

| Key | Used by | Value |
| --- | --- | --- |
| `VAULT_DIR` | Ask, Writer, Enricher | Absolute path of your Obsidian vault |
| `CAPTURES_DIR` | Enricher (trigger) | `<VAULT_DIR>/Captures` |
| `TG_SEND_URL` | Ask | `https://api.telegram.org/bot<TOKEN>/sendMessage` |
| `TG_CHAT_ID` | Ask | Your Telegram chat id |
| `WRITER_HASH` | Ask (dispatch target) | Writer's `agent_hash`, printed by its deploy |

Because the `filesystem` trigger runs inside the EEAPI container, the vault must be mounted
in the container at the **same absolute path** as on the host — see
`docker-compose.override.yaml.example` in [synx-init](https://github.com/peiblow/synx-init).

## Deploy

Order matters only once: Writer first, so its hash can feed Ask's config. The `agent_hash`
is deterministic — the same source produces the same hash on any machine.

```sh
cd writer   && synx ship .        # copy the printed agent_hash into WRITER_HASH
cd ..       && synx ship .        # Ask
cd enricher && synx ship .
```

Triggers are mounted at EEAPI startup, so restart it after deploying (`make restart-eeapi`
in synx-init). Note that changes to `contract.env` consumed by EEAPI (trigger paths) need a
container recreate (`docker compose up -d`), not just a restart.

Finally, point your Telegram bot's webhook at the public URL of the `/chat` route (e.g. via
an HTTPS tunnel in local setups):

```sh
curl "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://<public-host>/chat"
```

## Layout

Each agent is a self-contained deploy unit: its own `synx.toml`, its own `policies.snx`
(the gate library), and its system prompt as an external `.md` embedded at build time.

```
second-brain/
├── ask.snx            # Ask agent + its tools
├── policies.snx       # AskGates — verdict functions for Ask's tools
├── prompts/ask.md     # Ask's system prompt
├── synx.toml
├── writer/            # Writer agent (same structure)
└── enricher/          # Enricher agent (same structure)
```
