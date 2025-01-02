# freellmpool

<!-- mcp-name: io.github.0xzr/freellmpool -->

![freellmpool tokenmax terminal demo](assets/demo.svg)

![178 enabled chat routes, 23 LLM providers cataloged, keyless start when available](assets/tokenmax-results.svg)

freellmpool catalogs 23 LLM providers as distinct groups spanning recurring
free tiers, keyless endpoints, finite trials, pin-only routes, and disabled
candidates. It exposes 178 enabled chat routes and 443 cataloged chat models,
and automatically pools only
enabled routes you can access behind one OpenAI-compatible endpoint — as a CLI,
a Python library, or a local proxy. It can start without credentials when an
enabled keyless route is available.

[![PyPI](https://img.shields.io/pypi/v/freellmpool.svg)](https://pypi.org/project/freellmpool/)
[![CI](https://github.com/0xzr/freellmpool/actions/workflows/ci.yml/badge.svg)](https://github.com/0xzr/freellmpool/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Website](https://img.shields.io/badge/docs-0xzr.github.io%2Ffreellmpool-6ea8ff)](https://0xzr.github.io/freellmpool/)

[FAQ](FAQ.md): where prompts go, ToS posture, failover, bans, and comparisons.

## Release and distribution status

- **Latest release: 0.13.0.** The GitHub release and PyPI package are both
  0.13.0; `pip install freellmpool` and `uvx freellmpool` install the audited
  provider catalog, bounded streaming and sentinel hardening, Hermes profile,
  proxy readiness/provider APIs, `spread` routing, and OpenCode
  registry-readiness hardening.

- **Registry publication status: pending.** `opencode-freellmpool` and
  `opencode-freellmpool-tui` are tested but not published on npm as of
  2026-08-29. Use their repository-local installation instructions for now.

## 30-second quickstart

Fresh install to first free-model reply is measured at about 19 seconds under
the 30-second target on a clean Linux/Python 3.12 environment, with no API keys
when a keyless provider is up:

```bash
# One command when uv is installed; no checkout or manually managed virtualenv:
uvx freellmpool ask --max-tokens 32 "Reply with one short sentence: freellmpool is ready."
```

Portable virtual-environment path:

```bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install freellmpool
freellmpool ask --max-tokens 32 "Reply with one short sentence: freellmpool is ready."
```

CI runs the same path from this checkout with
`FREELLMPOOL_QUICKSTART_PACKAGE=. scripts/quickstart-test.sh`.

The catalog covers provider-operated free tiers, keyless routes, and a few
explicit finite-trial or disabled candidates. Eligibility and limits differ by
provider. freellmpool automatically uses only enabled routes you can access,
fails over when one is rate limited or down, and tracks local per-day usage.

Several providers (Pollinations, OVHcloud, and Kilo Gateway) need no API key,
and LLM7 works without one, so the quickstart can answer without signup when a
keyless provider is available.

To inspect your local provider keys, agent CLIs, proxy config, and Tailscale
state before wiring tools, run the print-only init wizard:

```bash
freellmpool init --yes
freellmpool init --yes --agent opencode
freellmpool init --yes --agent metaswarm --tailnet
```

Add keys for the other providers to unlock more models and higher limits.

### Local runtimes (explicit, pin-only)

LM Studio, Ollama, and llama.cpp can be previewed from a fixed list of literal
loopback endpoints without scanning your LAN or processes:

```bash
freellmpool local discover
freellmpool local discover --name lm_studio
freellmpool local import --name lm_studio --yes
freellmpool ask --providers local_lm_studio --model "<model-from-discover>" "hello"
```

Discovery is read-only. Import is a separate affirmative step, writes no
credential, and keeps every imported model out of automatic routing
(`auto = false`). `freellmpool local remove local_lm_studio --yes` reverses only
the block managed by the importer. Custom endpoints must be canonical literal
loopback URLs such as `http://127.0.0.1:1234/v1`; hostnames, LAN addresses,
redirects, and broad network scans are rejected.

## First-run setup with `freellmpool init`

`freellmpool init` inspects provider keys, installed agent CLIs, Tailscale
state, and proxy config, then prints one copy-pastable next step without editing
files. Run it detect-only first:

```bash
freellmpool init --yes
```

`--json` emits the same detection as versioned JSON for scripts and agents.

### Tailnet / remote agent gateway

Serve the proxy on your Tailscale 100.x address with a generated API key:

```bash
freellmpool tailnet serve --port 8080
```

From a remote machine:

```bash
freellmpool tailnet connect <tailnet-ip> --port 8080
```

Both sides support `--api-key <shared-secret>` if you want to pin a key instead
of using a generated token. Tailnet serving requires auth by default; do not
run unauthenticated over non-loopback interfaces.

### Metaswarm agent lanes

This project uses one Umans/Kimi K2.7 worker lane, one MiniMax M3 lane, Codex as
escalation, and Claude Opus only for final pre-ship review. The installable
Metaswarm profile mirrors that posture: one free/cheap worker lane through the
local proxy, one larger freellmpool reviewer lane, and Codex/Opus as explicit
user-owned paid escalation/final-review lanes only (never silent).

```bash
freellmpool init --yes --agent metaswarm --tailnet
freellmpool profile install metaswarm
freellmpool tailnet serve --port 8080
freellmpool profile doctor metaswarm --dry-run
```

## Run a coding agent on free models

freellmpool's proxy speaks the OpenAI API and includes an experimental
Anthropic-compatible path, so coding agents can run against pooled free tiers —
just point them at the proxy:

```bash
freellmpool proxy                       # starts http://localhost:8080
freellmpool code claude                 # prints the one-line setup for Claude Code
freellmpool profile list                # richer installable profiles
freellmpool profile show metaswarm      # Tailnet-aware Metaswarm profile
freellmpool profile install hermes       # Hermes custom-endpoint config
# (also: codex, aider, cline, continue, cursor, hermes, opencode, metaswarm)
```

The Hermes profile prints (and never writes) this supported custom-endpoint
block; `hermes model` provides the interactive equivalent:

```yaml
model:
  provider: custom
  default: quality
  base_url: http://localhost:8080/v1
  api_key: anything
```

Claude Code gateway mode can also be launched directly:

```bash
ANTHROPIC_BASE_URL=http://localhost:8080 \
ANTHROPIC_AUTH_TOKEN=dummy \
ANTHROPIC_API_KEY=dummy \
ANTHROPIC_MODEL=auto \
ANTHROPIC_SMALL_FAST_MODEL=auto \
CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1 \
claude
```

Existing OpenAI-compatible apps work the same way: set
`OPENAI_BASE_URL=http://localhost:8080/v1` and keep your code unchanged.
Anthropic-compatible tools can use the experimental bridge with
`ANTHROPIC_BASE_URL=http://localhost:8080`.

Text-only `/v1/responses` and `/v1/messages` requests stream incrementally from
the selected provider. Tool calls and richer content intentionally stay on the
buffered compatibility path. The browser dashboard and playground share one
public, data-free shell; when proxy auth is enabled it prompts for the bearer
token and keeps it only in page memory while protected status, inventory,
model, and battle calls continue to require the `Authorization` header.

**OpenCode** gets a deeper integration in 0.12.0: a live in-editor **dashboard** (routing mode,
estimated savings, tokens served free, provider race, latency), per-request
**agent routing** via the model picker (`freellmpool/agent|spread|auto|fast|quality|fair`), and `freellmpool_status`
/ `freellmpool_models` tools — see [integrations/opencode-tui](integrations/opencode-tui)
and the [guide](https://0xzr.github.io/freellmpool/run-opencode-on-free-models.html).
The plugin registers its routing aliases automatically on supported OpenCode
versions without rewriting user configuration. Restart OpenCode and check
`opencode models freellmpool`.
The package tarballs are validated in CI, but npm publication remains pending;
the linked local-file instructions remain the working install path.

**New in 0.11:** capacity tools — `freellmpool capacity status` shows which free
tiers are usable right now, `freellmpool providers health` live-probes them, and
`freellmpool keys add` walks you through configuring more (see
[Capacity & provider health](#capacity--provider-health) and
[docs/CAPACITY.md](docs/CAPACITY.md)).

**New in 0.10:** an async API (`AsyncPool`), an MCP server (`freellmpool mcp`),
latency-aware routing with `freellmpool benchmark`, observability hooks, and a
plugin system for custom providers. See the [changelog](CHANGELOG.md).

## Install

```bash
pip install freellmpool      # or: pipx install freellmpool
```

Only dependency is `httpx`. Python 3.11+.

### Docker and Open WebUI

The published container runs the local proxy as an unprivileged user. Persist
quota, stats, route-health, cache, and configuration state in a named volume:

```bash
docker volume create freellmpool-data
docker run --rm -p 127.0.0.1:8080:8080 \
  --volume freellmpool-data:/home/freellmpool/.config/freellmpool \
  --env GROQ_API_KEY \
  ghcr.io/0xzr/freellmpool:0.13.0
```

Omit `--env GROQ_API_KEY` if you want a credential-free start; the proxy can
answer only while an enabled keyless route is available. Keep the published
port on loopback. If you deliberately expose it, set
`FREELLMPOOL_PROXY_KEY` and require that Bearer token from clients.

For the bundled Open WebUI stack, place any provider credentials in `.env` and
run `docker compose up -d`. Compose waits for the proxy health check and keeps
both freellmpool and Open WebUI state in named volumes; open
`http://localhost:3000` after both services are healthy.

## Command line

```bash
freellmpool ask "Write a haiku about sqlite"
git diff | freellmpool ask "Write a commit message for this"
freellmpool tokenmax "Hardest question you've got"  # 🌈 blast models, print answers, optional synthesis
freellmpool providers        # which providers are configured
freellmpool models           # cataloged provider/model ids
freellmpool stats            # lifetime tokens served free + estimated cost avoided
freellmpool badge -o badge.svg   # a shareable SVG badge of that total
freellmpool badge --summary -o summary.svg   # a larger usage summary card
```

`freellmpool tokenmax` is the tongue-in-cheek maximum-effort mode: it fans your
prompt out to many available models at once and prints each answer. The CLI adds
a synthesized verdict by default unless you pass `--no-synthesize`; the MCP tool
returns the model answers for the calling agent to synthesize. (See
[docs/MCP.md](docs/MCP.md).)

`freellmpool stats` is a running, **persistent** lifetime total (it survives restarts
and upgrades). Embed `freellmpool badge` in a README, or use
`freellmpool badge --summary -o summary.svg` for a larger card with tokens,
requests, estimated savings, and provider mix. The proxy can serve both
`/badge.svg` and `/summary.svg` live when `FREELLMPOOL_PUBLIC_BADGE=1` makes
public badges embeddable.

Pin a provider or model; common OpenAI/Anthropic model names are mapped to a free
equivalent so existing scripts keep working:

```bash
freellmpool ask -m groq/openai/gpt-oss-20b "hi"
freellmpool ask -p cerebras,groq "hi"
freellmpool ask -m gpt-4o-mini "hi"      # routed to a free model
```

### Roles

`freellmpool roles` lists ask-role presets (`coder`, `critic`, `summarizer`,
`grounded-reader`, `long-context`, `cheap`, `fast`, `second-opinion`, ...). Each
role sets routing, token budget, temperature, task intent, and system-prompt hints
without inventing a second routing engine. Explicit flags (`--model`, `--providers`,
`--routing`, `--task`, `--max-tokens`) win over role defaults.

```bash
freellmpool ask --role coder "write a pytest for this function"
freellmpool ask --role grounded-reader "read this Markdown file"
freellmpool ask --routing quality --task general "ignore automatic task classification"
FREELLMPOOL_MODE=wise freellmpool ask --role cheap "summarize this patch"
```

## As a proxy

Run a local server that speaks the OpenAI API, then point any OpenAI-compatible
tool at it. On loopback, any placeholder API key works unless you configured
`FREELLMPOOL_PROXY_KEY` or passed `--api-key`; Tailnet/LAN serving requires a
real proxy bearer token by default.

```bash
freellmpool proxy
export OPENAI_BASE_URL=http://localhost:8080/v1
export OPENAI_API_KEY=unused
```

```python
from openai import OpenAI
client = OpenAI()
print(client.chat.completions.create(
    model="auto",
    messages=[{"role": "user", "content": "hi"}],
).choices[0].message.content)

# audio → text (Whisper), same client:
print(client.audio.transcriptions.create(
    model="auto", file=open("audio.mp3", "rb"),
).text)
```

Or with `curl` (multipart upload):

```bash
curl -s http://localhost:8080/v1/audio/transcriptions \
  -F file=@audio.mp3 -F model=auto
```

The proxy also implements the OpenAI Responses API (for the Codex CLI) and an
experimental Anthropic Messages API path (for Claude Code), so coding agents can
run on free models too. `freellmpool code <agent>` prints the exact setup, while
`freellmpool profile install <agent>` prints the fuller copy-pastable profile
without mutating third-party config:

```bash
freellmpool code aider       # also: claude, codex, cline, continue, cursor, hermes, opencode
freellmpool profile show opencode
freellmpool profile doctor opencode --dry-run
```

Release 0.12.0 proxy surfaces:

- `/v1/chat/completions` — OpenAI-compatible chat, token streaming, tool calling.
- `/v1/responses` — Responses API shim with incremental text streaming; tools
  and rich content use the buffered compatibility path.
- `/v1/messages` — experimental Anthropic-compatible Messages path with the
  same incremental-text / buffered-tool split.
- `/v1/embeddings` and `/v1/audio/transcriptions` — OpenAI-compatible embedding
  and Whisper-style multipart transcription.
- `/v1/models` — routing aliases plus concrete `provider/model` ids.
- `/v1/models?ready=true` — the same shape, limited to locally ready targets.
- `/v1/providers` — authenticated, secret-free provider/model readiness details.
- `/freellmpool/battle` and `/playground` — bounded browser/JSON model comparisons.
- `/healthz` and `/livez` — public process liveness aliases.
- `/readyz` — public advisory local-capacity readiness (200 when at least one
  provider is ready, otherwise 503); it never live-probes an upstream.
- `/dashboard`, `/status`, `/badge.svg`, and `/summary.svg` — local operations surfaces.

`/`, `/dashboard`, and `/playground` serve one public, data-free shell. When a
proxy key is set, the shell prompts for it and keeps it only in page memory;
status, provider/model inventory, battle, and model API calls remain header-auth
protected. Liveness/readiness stay public for orchestrator probes; readiness is
a local quota/cooldown snapshot, not an upstream health guarantee.
Setup snippets for specific tools are in [docs/INTEGRATIONS.md](docs/INTEGRATIONS.md)
and [docs/AGENTS.md](docs/AGENTS.md). The repo also includes an experimental
[metaswarm review adapter](integrations/metaswarm) for using `freellmpool` as an
external-tools reviewer/second opinion. `freellmpool profile show metaswarm`
documents a free/cheap worker lane, a larger reviewer lane, Tailnet client setup,
and paid Codex/Opus lanes as explicit user-owned escalation paths only.

## As a library

```python
from freellmpool import Pool

pool = Pool.from_default_config()
reply = pool.ask("Summarize the plot of Hamlet in 20 words.")
print(reply.text, "—", reply.provider_id)

vectors = pool.embed(["first document", "second document"]).vectors

with open("audio.mp3", "rb") as f:
    text = pool.transcribe(f.read(), "audio.mp3").text   # Whisper, failover across providers
```

Async is the same API with `await`:

```python
from freellmpool import AsyncPool

async with AsyncPool.from_default_config() as pool:
    reply = await pool.aask("Summarize the plot of Hamlet in 20 words.")
```

Pass `on_event=...` to either pool to receive structured routing/cache events
(`attempt`/`success`/`error`/`cooldown`/`cache_hit`/`cache_miss`/`exhausted`) for logging or tracing. Add
your own endpoint with `register_provider(...)`, or a new request shape with
`register_adapter(name, fn)`.

## Benchmark your providers

`freellmpool benchmark` times one call per configured provider and prints
latency and success, so you can see which of your free tiers are fastest right
now. The router learns the same latency/success signal from real traffic as it
runs; set `FREELLMPOOL_ROUTING=fast` to prefer the lowest-latency provider
instead of the default least-used-first.

```
$ freellmpool benchmark
  provider/model            status   latency  note
  cerebras/gpt-oss-120b     ok        180 ms  6 tok
  groq/openai/gpt-oss-20b   ok        240 ms  6 tok
  ovh/Meta-Llama-3_3-70B-Instruct  FAIL    -  HTTP 429
```

## Capacity & provider health

Free tiers drift through the day — keys expire, providers go down, daily caps
fill. These commands tell you what's usable right now and what to set up next:

```bash
freellmpool capacity status --target 5   # who's healthy / near quota / missing a key
freellmpool capacity status --refresh    # explicitly refresh advisory provider metadata
freellmpool quota-wise status            # local headroom + recommended mode
freellmpool providers health             # send one tiny request to each, time it
freellmpool keys checklist --target 5    # which keys to add to reach N healthy providers
freellmpool keys add groq                # configure a key (and record metadata)
```

Protocol support is verified separately from basic provider health. Run
`freellmpool conformance run` to check chat, streaming, tools, JSON object/schema,
vision, Responses, and Anthropic Messages with bounded synthetic requests, then inspect
the sanitized evidence with `freellmpool conformance status --json`.
Feature-specific auto-routing uses only verified targets; an exact
provider/model pin is the explicit override. Full operator and privacy contract:
[docs/PROTOCOL_CONFORMANCE.md](docs/PROTOCOL_CONFORMANCE.md).

`capacity status` is offline and cache-first: it reads your catalog, environment, and
per-day quota counters and labels each provider `healthy`, `low_quota`,
`exhausted`, `invalid_key`, or `missing`. Cached metadata from the advisory
external catalog ([mnfst/awesome-free-llm-apis](https://github.com/mnfst/awesome-free-llm-apis))
can suggest free providers you could add; only an explicit
`freellmpool capacity status --refresh` or `freellmpool catalog sync` uses the
network for that metadata. It remains advisory only; your `providers.toml`
stays the source of truth for routing. `keys add <name>` can import a
suggested provider from that catalog or create an OpenAI-compatible stub and
autodiscover its models. The proxy `/dashboard` shows the same capacity at a
glance. Full reference: [docs/CAPACITY.md](docs/CAPACITY.md).

`FREELLMPOOL_MODE=wise` is the conservative quota mode: `ask` defaults to a
smaller output budget and spread routing, `tokenmax` narrows its default fan-out,
and broad multi-model calls require confirmation unless you pass `--yes`.
Per-command `--mode normal|wise` overrides the environment, and
`[settings] mode = "wise"` works from `config.toml`. The `conserve` role is a
quota-conscious shorthand for small, spread-routed answers.

For a bounded second opinion instead of a full `tokenmax` blast:

```bash
freellmpool ask --second-opinion --opinions 3 "is this implementation plan sound?"
freellmpool ask --role second-opinion --synthesize "which release note is clearer?"
```

The shared panel asks a few diverse providers, keeps individual failures vi