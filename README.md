# Hindsight memory for Hermes Agent: setup and best practices

A field guide for giving a [Hermes Agent](https://hermes-agent.nousresearch.com/docs) long-term memory with [Hindsight](https://hindsight.vectorize.io), on a local machine or a cloud server.

How to read the labels:

- **Tested**: run end to end on one Mac Studio (Apple M5 Max, 128 GB) in November 2026 with the Hermes Hindsight plugin 1.2.4, Hindsight server and client 1.2.3, and Ollama 0.40.17. One machine, small banks. Treat these as field notes, not benchmarks.
- **Docs**: stated in the official Hindsight or Hermes docs (see [Sources](#sources)).
- **Recipe**: the author's own combination of documented pieces. Sensible, but not an official recommendation.

Versions move fast. Check the sources before relying on any value here.

---

## Contents

1. [Choose a deployment](#1-choose-a-deployment)
2. [Install the plugin (all deployments)](#2-install-the-plugin-all-deployments)
3. [Local machine with Ollama](#3-local-machine-with-ollama)
4. [Cloud server](#4-cloud-server)
5. [Privacy and secrets (read before storing real data)](#5-privacy-and-secrets-read-before-storing-real-data)
6. [Configure the memory bank](#6-configure-the-memory-bank)
7. [Tune recall](#7-tune-recall)
8. [Verify it works](#8-verify-it-works)
9. [Backups and maintenance](#9-backups-and-maintenance)
10. [Pitfalls we hit](#10-pitfalls-we-hit)
11. [Sources](#sources)

---

## 1. Choose a deployment

Hermes talks to Hindsight in one of three modes (`mode` in the plugin config). **Docs.**

| Mode | What runs where | Cost | Good fit |
| --- | --- | --- | --- |
| `local_embedded` | Hermes starts a Hindsight daemon on the same machine, with an embedded PostgreSQL (pg0). You supply an LLM. | Free with a local LLM. Token cost with a hosted LLM. | A machine with enough RAM for a local model, or a small cloud box using a hosted LLM |
| `local_external` | Hindsight runs as its own service (Docker or pip). Hermes connects over HTTP. | Your server plus LLM tokens | Cloud servers, several agents sharing one memory, production |
| `cloud` | Vectorize's hosted Hindsight Cloud | Hindsight Cloud pricing | No infrastructure to run |

Quick decision (**Recipe**):

- **Machine with 16 GB+ RAM and Apple Silicon or a GPU:** `local_embedded` + Ollama. See [section 3](#3-local-machine-with-ollama).
- **Cloud VPS without a GPU:** `local_embedded` or `local_external` with a hosted LLM API. Running the LLM on the CPU is possible. We expect it to be too slow for memory that updates every turn, but we did not measure it. See [section 4](#4-cloud-server).
- **No servers to manage:** `cloud`.

**Who shares a bank.** Memory lives in a *bank*, named by `bank_id` (default `hermes`). Two separate embedded installs keep separate databases. But any installs that point at the same external server or the same Cloud account **with the same `bank_id` share one memory**. So do two Hermes profiles on one machine that use the same daemon profile. Give each agent its own `bank_id` unless sharing is the goal.

**"Local" covers only the memory engine.** Ollama in this guide runs Hindsight's extraction model. Hermes' own chat model is separate: if that is a hosted API, your conversation and the recalled memories still go to that provider. For a fully private setup, Hermes' main model must be local too.

---

## 2. Install the plugin (all deployments)

Hindsight is in the Hermes plugin catalog. **Docs.**

```bash
hermes plugins install hindsight --enable
hermes memory setup          # choose "hindsight", then a mode; prompts for keys without echoing them
hermes memory status         # expect: Provider: hindsight
```

Notes:

- `memory.provider: hindsight` in `config.yaml` is what activates the provider (the setup wizard writes it). `hermes plugins enable` alone does **not** activate it. **Docs.**
- Only one external memory provider can be active. Hermes' built-in `MEMORY.md` / `USER.md` stay on alongside it. When the agent **adds or replaces** a built-in memory entry, the plugin copies it into Hindsight. **Removing** an entry does not delete the Hindsight copy. **Docs** + plugin source.
- Plugin config lives in `$HERMES_HOME/hindsight/config.json` (one per Hermes profile). Keys under `memory.hindsight` in `config.yaml` are ignored. **Docs.**
- After changing plugin config, restart the gateway from a normal shell with `hermes gateway restart`. A running gateway keeps the config it started with. **Tested.**
- Update the plugin with `hermes plugins update hindsight`. `hermes update` does not move it. **Docs.**
- `hermes memory status` confirms the provider is selected. It does not prove the server is reachable. Use the checks in [section 8](#8-verify-it-works) for that.

---

## 3. Local machine with Ollama

### 3.1 Pick and test the local model

```bash
brew install ollama            # macOS; see ollama.com for Linux and Windows
brew services start ollama
ollama pull gemma3:12b         # Hindsight's documented Ollama default, about 8 GB
```

Check two requirements before you trust a model.

**a) It must support Ollama structured output.** Hindsight sends LLM calls with a JSON schema. A model that cannot honor the schema fails every save. **Tested:** with Ollama 0.40.17, the default `gemma4:26b` and `qwen3.8:27b` tags (served by Ollama's MLX runner) returned `501 structured output is unavailable`. The llama.cpp-served tag `gemma4:26b-a4b-it-q4_K_M` worked. Test any model like this:

```bash
curl -s localhost:11434/api/chat -d '{
  "model": "gemma3:12b", "stream": false,
  "format": {"type":"object","properties":{"ok":{"type":"boolean"}},"required":["ok"]},
  "messages": [{"role":"user","content":"Return ok true as JSON."}]
}'
# Pass: message.content is {"ok": true}.   Fail: {"error":"structured output is unavailable"}
```

**b) It must allow enough output tokens.** Hindsight's models guide calls for a 65,000-token output budget for fact extraction, or a lower `HINDSIGHT_API_RETAIN_MAX_COMPLETION_TOKENS` that your model accepts. A lowered limit must still be larger than `HINDSIGHT_API_RETAIN_CHUNK_SIZE`, or the server refuses to start. The tiny test above does not check this. **Docs.**

**Test candidates on your own data.** Hindsight's [leaderboard](https://benchmarks.hindsight.vectorize.io/leaderboards) scores models as served by their providers, across the operations that use an LLM. A local quantized build can rank differently. **Tested:** on one five-question test conversation saved through the real pipeline into throwaway banks, `gemma3:12b` recalled 5/5 with no junk facts (about 16 s per save). `gemma4:26b-a4b-it-q4_K_M` recalled 4/5 because it dropped an email address. That is one small sample: a sanity check, not a ranking.

### 3.2 Pre-install the Hindsight server

On first use, `local_embedded` downloads the server through `uvx`. The integration docs give the download as about 482 MB on macOS ARM and about 3.2 GB on Linux x86-64. **Tested:** this download made our first daemon start time out. Install it once ahead of time, matching the version your plugin expects:

```bash
uv tool install 'hindsight-api==1.2.3'    # use the version your plugin runs; check with: uv tool list | grep hindsight
```

### 3.3 Plugin config

`$HERMES_HOME/hindsight/config.json`. The example below holds no secrets, but a file written by the setup wizard can contain an API key, so keep it `chmod 600` and out of any repo:

```json
{
  "mode": "local_embedded",
  "bank_id": "hermes",
  "llm_provider": "ollama",
  "llm_model": "gemma3:12b",
  "llm_base_url": "http://localhost:11434/v1",
  "memory_mode": "hybrid",
  "auto_retain": true,
  "auto_recall": true,
  "recall_types": "observation,world,experience",
  "recall_budget": "mid",
  "recall_min_scores": {"reranker": 0.02},
  "retain_context": "Conversation between the user and their AI assistant",
  "retain_source": "hermes",
  "recall_indicator": false,
  "retain_indicator": false,
  "idle_timeout": 0
}
```

What each non-default setting does:

| Key | Plugin default | Why change it |
| --- | --- | --- |
| `recall_types` | observations only | Until consolidation has run there are no observations, so recall returns nothing. Including raw facts makes recall work from the first save. Pair it with the bank setting `recall_prefer_observations` ([section 6](#6-configure-the-memory-bank)) to collapse duplicates. **Tested.** |
| `recall_min_scores` | no floor | Without a floor, recall fills up to the token budget whether or not the results are relevant. **Calibrate your own value** ([section 7](#7-tune-recall)). 0.02 fit our bank and may not fit yours. |
| `retain_context` | `conversation between Hermes Agent and the User` | The docs call context "high-impact on extraction quality". Describe what the conversations are. |
| `retain_source` | empty | A label for where memories came from. Helps when several sources share a bank. |
| `recall_indicator`, `retain_indicator` | on | Off hides the "👁️ Hindsight" status lines in chat apps. |
| `idle_timeout` | 300 s | `0` keeps the daemon running. **Tested:** a cold start took about 40 s, and the running daemon used about 2 GB of RAM. |

### 3.4 Daemon settings

The daemon builds its environment from three layers: the inherited process environment, the profile file `~/.hindsight/profiles/<profile>.env`, and settings the plugin generates from `config.json`. Set daemon-only options in the profile file with the `hindsight-embed` binary that ships in the plugin's environment. **Docs** + **Tested.**

```bash
# If hindsight-embed is not on PATH:  ls ~/.hermes/installs/*/environments/*/venv/bin/hindsight-embed
hindsight-embed profile set-env hermes HINDSIGHT_API_LLM_MAX_CONCURRENT 1
hindsight-embed profile set-env hermes HINDSIGHT_API_ENABLE_OBSERVATIONS true
```

**A running daemon does not reload this file.** After changing it, stop the daemon (`hindsight-embed -p hermes daemon stop`) and let Hermes start it again on the next memory call. **Tested.** This matters more with `idle_timeout: 0`, because then the daemon never stops on its own.

- `LLM_MAX_CONCURRENT=1` comes from Hindsight's Ollama tutorial. Current Ollama can serve parallel requests when memory allows, so treat `1` as a safe starting point and raise it if saves queue up. **Docs.**
- `ENABLE_OBSERVATIONS`: the Ollama tutorial suggests `false` on slow machines, because consolidation adds background LLM calls. **Tested:** on an M5 Max, consolidating 15 facts took under 2 minutes with zero failures, so we turned it on. The server switch and the per-bank `enable_observations` setting ([section 6](#6-configure-the-memory-bank)) should match. If you turn it off here, also set the bank setting to `False`.

### 3.5 Memory budget

**Tested:** the daemon used about 2 GB and `gemma3:12b` about 8 GB. Ollama keeps a model loaded for 5 minutes after its last request (its default keep-alive), then unloads it. Plan for both if the machine also runs other large models.

---

## 4. Cloud server

### Option A: hosted LLM on a small VPS (simplest)

Use `local_embedded` as in [section 3](#3-local-machine-with-ollama), but send extraction to a hosted LLM. The easiest path is `hermes memory setup`, which sets the OpenRouter endpoint and stores the key without echoing it. When switching from Ollama, the wizard suggests your old model name as the default: type the OpenRouter model ID instead of accepting it. A manual equivalent for OpenRouter:

```json
{
  "mode": "local_embedded",
  "bank_id": "hermes",
  "llm_provider": "openrouter",
  "llm_base_url": "https://openrouter.ai/api/v1",
  "llm_model": "<provider/model-id from openrouter.ai>"
}
```

Remove any leftover Ollama `llm_base_url`. The key goes in `$HERMES_HOME/.env` as `HINDSIGHT_LLM_API_KEY`, never in `config.json`. Use the wizard, or add it with an editor to a file already set to `chmod 600`. Typing it into an `echo` command stores it in your shell history.

**Choosing a model.** At the time of writing, the top three rows of the leaderboard's [All operations board](https://benchmarks.hindsight.vectorize.io/leaderboards/all-operations) paired an LLM with Hindsight's decision model `hs-decide-1-flash`. Those LLMs were `glm-5.3-flash` (score 90.7), `gpt-6-luna` and `claude-haiku-5.5`. Without the decision model, `glm-5.3-flash` scored 81.7. Using the decision model takes extra `HINDSIGHT_API_DECISION_MODEL_*` settings; see the [models guide](https://hindsight.vectorize.io/developer/models). Rankings move, so check the live board.

**Hardware (Docs):** 1.5 to 2 GB of RAM for the full server, which runs its own local embedder and reranker. 2 vCPUs without a GPU is fine for development and light use. The embedded pg0 database needs memory on top of that.

**Cost:** you pay LLM tokens for every save, consolidation and `reflect`. Embeddings and reranking run locally at no token cost.

### Option B: dedicated Hindsight service (production or shared memory)

Run Hindsight as its own service and connect with `local_external`.

**1. Secrets in a protected env file**, not on the command line (**Recipe**):

```bash
sudo mkdir -p /etc/hindsight && sudo chmod 700 /etc/hindsight
sudo sh -c 'umask 077; touch /etc/hindsight/hindsight.env; chmod 600 /etc/hindsight/hindsight.env'   # touch never empties an existing file
# then edit it and add:
#   HINDSIGHT_API_LLM_PROVIDER=...
#   HINDSIGHT_API_LLM_API_KEY=...
#   HINDSIGHT_API_DATABASE_URL=postgresql://USER:PASS@HOST:5432/DB
#   HINDSIGHT_API_TENANT_EXTENSION=hindsight_api.extensions.builtin.tenant:ApiKeyTenantExtension
#   HINDSIGHT_API_TENANT_API_KEY=<long random string, e.g. from: openssl rand -hex 32>
#   HINDSIGHT_API_WORKER_ID=hindsight-1
```

**2. Run a pinned image** (**Recipe** built from the installation docs):

```bash
docker run -d --name hindsight --restart unless-stopped \
  --shm-size=1g --stop-timeout 75 \
  --env-file /etc/hindsight/hindsight.env \
  -p 127.0.0.1:8888:8888 \
  ghcr.io/vectorize-io/hindsight:1.2.3      # pin a version you tested, not :latest
```

Run `docker` as root (`sudo docker ...`) or as an account that can read `/etc/hindsight/hindsight.env`: the Docker CLI reads the env file, not the daemon.

**3. Point Hermes at it.** `config.json`:

```json
{ "mode": "local_external", "api_url": "http://127.0.0.1:8888", "bank_id": "hermes" }
```

Put the tenant key in `$HERMES_HOME/.env` as `HINDSIGHT_API_KEY` (wizard or editor, as above).

**Reaching it from another machine.** `-p 127.0.0.1:8888:8888` accepts connections only from the server itself, which is the safe default. For a remote Hermes, pick one:

- SSH tunnel from the Hermes machine: `ssh -N -L 8888:127.0.0.1:8888 user@server`, keeping `api_url` at `http://127.0.0.1:8888`.
- Publish on a private interface only, for example a Tailscale or VPC address: `-p <private-ip>:8888:8888`, with `api_url` set to `http://<private-ip>:8888`.
- If it must face the internet, put it behind a TLS reverse proxy and keep authentication on.

Production checklist:

- **External PostgreSQL 14+ with pgvector** (Supabase, Neon, RDS, Cloud SQL or self-hosted). The docs say embedded pg0 is "not recommended for production". Back up that database with its own tooling, and use the bank export in [section 9](#9-backups-and-maintenance) as well. **Docs.**
- **Authentication on.** Hindsight has no auth by default. The tenant extension requires a Bearer key on the bank API. `/health` stays open, so test auth separately ([section 8](#8-verify-it-works)). This extension accepts one key and gives every holder the same view of every bank, so it is a server password, not per-user access control. **Docs** + server source.
- **Stable worker ID.** Set `HINDSIGHT_API_WORKER_ID` even with one container, so a replacement container can recover unfinished work. **Docs.**
- **Persistence.** With external PostgreSQL, the database is your storage. If you run pg0 inside the container instead, mount the named volume from the installation docs. A host bind mount must be owned by UID 1000. **Docs.**
- **Sizing.** Full image: 1.5 to 2 GB of RAM. Slim image: 512 MB to 1 GB, but it needs external embedding and reranker providers. Under real traffic the local reranker is the bottleneck, so give it a GPU or use a hosted reranker. **Docs.**
- **Control plane UI** (port 9999) only if needed: publish it on `127.0.0.1:9999:9999`, set `HINDSIGHT_CP_DATAPLANE_API_KEY` so it can call the protected API, and reach it through a tunnel. **Docs.**

### Option C: Hindsight Cloud

Run `hermes memory setup`, choose `hindsight`, then `cloud`. The wizard first asks for the API key (from [ui.hindsight.vectorize.io](https://ui.hindsight.vectorize.io)), then the API URL. At the URL prompt, **type `https://api.hindsight.vectorize.io` yourself** rather than accepting the default: when switching from `local_external`, the default can still be your old server, and the wizard may send your new Cloud key there (for example while loading starter templates). Afterwards, open `config.json`: it should say `"mode": "cloud"` with the Cloud URL, and it should have **no** `apiKey` or `api_key` field. A leftover key field there overrides the new key in `.env`, so delete it before restarting the gateway. **Plugin source.** Bank configuration ([section 6](#6-configure-the-memory-bank)) still applies.

### Scheduled jobs and multiple users

- **Cron jobs.** The installed plugin (1.2.4) skips automatic saving for scheduled jobs unless `retain_cron` is `true`. It has **no** `recall_cron` switch, even though the live docs list one, so cron runs still auto-recall if `auto_recall` is on. The explicit memory tools stay available to cron jobs either way. **Plugin source** (checked against the docs).
- **Several people on one server.** Bank templates like `bank_id_template: "hermes-{user}"` separate memory by name, but they are not a security boundary. An empty `{user}` falls back to the plain `hermes` bank. Tag matching only applies when recall tags are actually set. `reflect` has its own tag settings, and observation consolidation is shared by default. For people who must not see each other's memories, give each one a separate deployment with its own storage, or use a tenant extension that maps each key to its own tenant. The built-in single-key extension does neither. If one trusted app does share a bank, scope saves, recall, reflect and consolidation. Then test with two users and with a missing user ID before going live. **Docs** + plugin source.

---

## 5. Privacy and secrets (read before storing real data)

A line in the retain mission like "never store passwords" only steers **fact extraction**. It does not stop raw text from being stored or sent:

- **Raw text is kept.** `HINDSIGHT_API_STORE_DOCUMENT_TEXT` defaults to `true`, so the original conversation text is saved next to the extracted facts. **Docs** + server default.
- **LLM calls are logged.** `HINDSIGHT_API_LLM_TRACE_ENABLED` defaults to `true`, with a retention set by `HINDSIGHT_API_LLM_TRACE_RETENTION_DAYS`. Deleting a document does not immediately delete its traces. **Server default.**
- **A hosted extraction model sees everything** you save, before any mission instruction applies.

Practical rules (**Recipe**):

1. Don't paste secrets into a chat with an agent that auto-saves. Redact before saving.
2. Consider `HINDSIGHT_API_STORE_DOCUMENT_TEXT=false` if you don't need the raw text, and shorten or disable LLM traces on servers that handle personal data. Check the [configuration reference](https://hindsight.vectorize.io/developer/configuration) for the current behavior first.
3. Keep keys in `$HERMES_HOME/.env` or the server env file (`chmod 600`), never in `config.json`, a repo or a command line.
4. Treat bank exports and backups as personal data: store them in a private folder (`chmod 700`).

---

## 6. Configure the memory bank

Hindsight's best-practices page says: "Misconfigured missions are the single biggest cause of low-quality memories." New banks start with empty missions, so set them **before** storing much. **Docs.**

Bank settings are not in `config.json`. Set them through the API, the Python client or the control plane UI. Find your server address first:

- **local_embedded:** `hindsight-embed -p hermes profile show -o json` shows the daemon port. For a profile named `hermes` it is usually `9177`.
- **local_external:** your `api_url`.
- **cloud:** the Cloud API URL plus your key.

Set these once in your shell; every example below uses them:

```bash
export HS_API=http://127.0.0.1:9177      # your server URL from above
export HS_BANK=hermes                     # your effective bank_id
# Cloud or a server with auth: load the key without echoing it (leave blank for local_embedded)
read -rsp "Hindsight API key (blank if none): " HINDSIGHT_API_KEY; echo; export HINDSIGHT_API_KEY
```

Run the Python below with an interpreter that has `hindsight_client` installed. In a Hermes install, the `python` next to `hindsight-embed` has it; plain `python3` may not.

```python
import os
from hindsight_client import Hindsight

# The client does not read Hermes' .env file. Pass the key explicitly when auth is on.
c = Hindsight(base_url=os.environ["HS_API"], api_key=os.environ.get("HINDSIGHT_API_KEY"))
BANK = os.environ["HS_BANK"]

c.update_bank_config(
    BANK,
    retain_mission=(
        "Long-term memory for <name>'s AI assistant. Extract durable facts: people and how they relate to "
        "<name>, their companies, roles and verified contact details; decisions and the reasons for them; "
        "stated preferences, corrections and standing rules; commitments, deadlines and dates; where files "
        "and documents live; project status changes. Keep names exactly as written. Ignore greetings, "
        "small talk, status chatter, tool output and temporary debugging steps. Never extract passwords, "
        "API keys, tokens, card numbers or one-time codes."
    ),
    observations_mission=(
        "Observations are durable beliefs about <name>, the people they work with, and how they want work "
        "done. Track evolving preferences, recurring patterns and relationships. Focus on durable patterns, "
        "not transient states. When new evidence contradicts an earlier observation, keep the history and "
        "state what changed and when. Keep different people with similar names apart."
    ),
    reflect_mission=(
        "You are <name>'s assistant. Answer from what memory shows, cite supporting facts, and say plainly "
        "when memory is thin, stale or contradictory. Prefer the most recent evidence."
    ),
    retain_extraction_mode="concise",
    default_timezone="America/New_York",       # your IANA time zone
    enable_observations=True,                  # match HINDSIGHT_API_ENABLE_OBSERVATIONS (section 3.4)
    recall_prefer_observations=True,
    disposition_skepticism=4,                  # dispositions shape reflect answers only (scale 1-5)
    disposition_literalism=3,
    disposition_empathy=3,
)
```

Mission-writing rules from the docs:

- List the **kinds of facts** you want, and list what to **ignore**. The docs treat the ignore list as just as important.
- Match the mission to the data you actually store: conversations, documents or tickets.
- The docs' own examples of bad missions: "Extract all information" and "Be helpful".

For `cloud` and `local_external`, `hermes memory setup` also offers starter **bank templates** with preset missions and dispositions.

If you turn observations on for a bank that already holds facts, run consolidation once:

Shell setup. The auth header goes in an owner-only file so the key never appears in the process list:

```bash
API="$HS_API"
BANK=$(python3 -c 'import os,urllib.parse;print(urllib.parse.quote(os.environ["HS_BANK"],safe=""))')
AUTH=()
if [ -n "${HINDSIGHT_API_KEY:-}" ]; then
  [ -n "${HDR:-}" ] && rm -f "$HDR"                   # rerunning this block: remove the previous file
  HDR=$(mktemp "${TMPDIR:-/tmp}/hs-auth.XXXXXX")      # new owner-only file for this shell only
  trap 'rm -f "$HDR"' EXIT
  printf 'Authorization: Bearer %s\n' "$HINDSIGHT_API_KEY" > "$HDR"
  AUTH=(-H "@$HDR")              # curl reads the header from the file (curl 7.55+)
fi
curl -s "${AUTH[@]}" -X POST "$API/v1/default/banks/$BANK/consolidate" -H 'Content-Type: application/json' -d '{}'
curl -s "${AUTH[@]}" "$API/v1/default/banks/$BANK/stats"   # wait for pending_consolidation 0; failed_consolidation should stay 0
```

The later shell examples assume these `API`, `BANK` and `AUTH` variables. The header file is deleted when the shell exits.

---

## 7. Tune recall

Hindsight ranks results but does not decide what counts as relevant. Without a floor, a question about pizza can return your tax notes. The docs give `{"reranker": 0.25}` as an example and say to check scores on your own bank first. **Docs.**

**Tested calibration:** on our bank, correct answers scored between 0.06 and 1.00 on the reranker. Unrelated questions scored 0.000 to 0.001. A 0.25 floor would have dropped two correct answers (0.06 and 0.16), so we used 0.02. Measure your own:

```bash
q() { curl -s "${AUTH[@]}" -X POST "$API/v1/default/banks/$BANK/memories/recall" \
  -H 'Content-Type: application/json' \
  -d "$(python3 -c 'import json,sys;print(json.dumps({"query":sys.argv[1],"types":["world","experience","observation"]}))' "$1")" \
  | python3 -c '
import sys, json
for r in json.load(sys.stdin)["results"][:5]:
    s = (r.get("scores") or {}).get("reranker")
    print("missing" if s is None else f"{s:.3f}", "|", r["text"][:70])'; }
q "a question your memory should answer"
q "a question it should know nothing about"
```

Run about 8 questions memory should answer and 6 it shouldn't. Set the floor between the lowest correct score and the highest unrelated one. If scores show as `missing`, your reranker does not produce them; don't set a reranker floor until it does. `q()` deliberately ignores your floor so you can see raw scores. If you later switch recall to a decision-model or TypeSafe reranker, remove the `reranker` floor: server 1.2.3 rejects that combination with HTTP 400. **Server source.**

Other recall settings:

- `recall_budget`: `mid` by default. The docs call `high` on every recall an anti-pattern, because it is slow and expensive. **Docs.**
- `recall_sync: false` (default) fetches memories in the background and uses them on the next turn. That keeps most of the delay out of replies, though the plugin can wait up to about 3 seconds for an unfinished fetch. Don't save and recall the same fact in one request: new memories take time to be indexed. **Docs** + plugin source.
- **Short facts can be invisible to topical questions.** **Tested:** a question about a topic scored 0.0 against a fact that only said *where* a list on that topic was saved. Rewriting the fact to name its topics raised the score to 0.98. To edit a fact: `curl "${AUTH[@]}" -X PATCH "$API/v1/default/banks/$BANK/memories/<memory_id>" -H 'Content-Type: application/json' -d '{"text":"..."}'`.

---

## 8. Verify it works

Run these after setup and after any change:

1. **Provider selected:** `hermes memory status` shows `Provider: hindsight`.
2. **Server up:** `curl -s "$API/health"` returns `"status":"healthy"` and `"database":"connected"`.
3. **Auth enforced (servers with auth):** `curl -s -o /dev/null -w '%{http_code}\n' "$API/v1/default/banks/$BANK/stats"` without a key should fail (401 or 403). With `"${AUTH[@]}"` it should return 200. `/health` doesn't prove this, because it is always open.
4. **Real round trip.** Tell the agent a distinctive made-up fact ("My test parrot is named Quillbert"). Wait about a minute for extraction. Then, **in a new session**, call the `hindsight_recall` tool or the recall API with "What is my test parrot called?". Asking in the same chat proves nothing, because the fact is still in the conversation. Delete the test fact afterward.
5. **Unrelated question:** ask through the `hindsight_recall` tool (which applies your configured floor), or add `"min_scores":{"reranker":<your floor>}` to the API request. It should return nothing. The `q()` helper skips the floor, so it can't test this.
6. **No failures:** `curl -s "${AUTH[@]}" "$API/v1/default/banks/$BANK/stats"` shows `failed_operations: 0` and `failed_consolidation: 0`.
7. **Browse what was stored, locally only (`local_embedded`):**
   ```bash
   UI_PORT=19177                                              # daemon port + 10000 by default
   hindsight-embed -p hermes ui stop                          # an already-running UI ignores --hostname
   hindsight-embed -p hermes ui start --hostname 127.0.0.1 --port "$UI_PORT"
   lsof -nP -iTCP:"$UI_PORT" -sTCP:LISTEN                     # every line must show 127.0.0.1, never * or 0.0.0.0
   ```
   Without `--hostname`, the UI listens on all network interfaces with no password. **CLI source.**

---

## 9. Backups and maintenance

**Full bank backup.** `memories/list` returns only one page of facts. It leaves out documents, directives, mental models and bank config, so it is not a backup. Use the transfer export, which covers data and bank config. The audit log and LLM request traces are left out unless you add `include_history=true`, and if you need them you must set it on **both** export and restore (see the [memory banks API](https://hindsight.vectorize.io/developer/api/memory-banks)). **Tested** on 1.2.3:

```bash
# Wait for an operation to finish. Retries only while it is pending or processing.
wait_op() {   # usage: wait_op <operation_id>; prints the final JSON on success
  local res st
  while :; do
    res=$(curl -sf "${AUTH[@]}" "$API/v1/default/banks/$BANK/operations/$1") || { echo "status request failed" >&2; return 1; }
    st=$(printf '%s' "$res" | python3 -c 'import sys,json;print(json.load(sys.stdin).get("status",""))') || { echo "bad status reply" >&2; return 1; }
    case "$st" in
      completed) printf '%s' "$res"; return 0 ;;
      pending|processing) sleep 5 ;;
      *) echo "operation ended as '$st': $res" >&2; return 1 ;;
    esac
  done
}

OP=$(curl -sf "${AUTH[@]}" -X POST "$API/v1/default/banks/$BANK/transfer/export" \
     | python3 -c 'import sys,json;print(json.load(sys.stdin)["operation_id"])') || { echo "export request failed"; exit 1; }
RES=$(wait_op "$OP") || exit 1
URL=$(printf '%s' "$RES" | python3 -c 'import sys,json;print(json.load(sys.stdin)["result_metadata"]["download_url"])') || exit 1

# Download to a private temp file, check it, then replace the old backup
TMP=$(mktemp "./backup.XXXXXX")                  # mktemp creates it owner-only
case "$URL" in
  http*) printf 'url = "%s"\n' "$URL" | curl -sf -o "$TMP" --config - ;;   # signed URL via stdin, kept out of ps; no Hindsight key
  *)     curl -sf "${AUTH[@]}" -o "$TMP" "$API$URL" ;;
esac && unzip -tq "$TMP" >/dev/null && mv "$TMP" ./bank-backup.zip && echo "backup OK: ./bank-backup.zip" \
  || { rm -f "$TMP"; echo "download or zip check failed; previous backup untouched"; exit 1; }
```

(These snippets use `exit 1` on failure: run them as a script, or replace `exit` with `return` inside a function.)

**Test the restore before you depend on it.** A restore writes into a **new** bank and never merges into an existing one. It re-embeds the data without LLM calls, so it costs no LLM tokens; a hosted embedding provider may still bill for the embeddings. **Tested:** a 132-memory bank restored completely and recall worked.

```bash
TARGET="restore-test-$(python3 -c 'import uuid;print(uuid.uuid4().hex[:12])')"   # unique, so nothing existing is touched
OP=$(curl -sf "${AUTH[@]}" -X POST "$API/v1/default/banks/$BANK/transfer/import?target_bank_id=$TARGET" \
       -F 'file=@-;filename=bank-backup.zip' < ./bank-backup.zip \
     | python3 -c 'import sys,json;print(json.load(sys.stdin)["operation_id"])') || { echo "import refused"; exit 1; }
wait_op "$OP" >/dev/null || exit 1               # the import is tracked on the submitting bank
curl -s "${AUTH[@]}" "$API/v1/default/banks/$BANK/stats"     # compare counts...
curl -s "${AUTH[@]}" "$API/v1/default/banks/$TARGET/stats"   # ...with the restored copy
# Only after you've compared them, remove the copy this script created:
# curl -s "${AUTH[@]}" -X DELETE "$API/v1/default/banks/$TARGET"
```

The restore block reuses `wait_op` from the backup block, so run them in the same shell or script.

Other maintenance:

- **Remove junk by conversation.** Setup and debugging chats get saved too, for example as "Recall returns too much". List documents with `GET $API/v1/default/banks/$BANK/documents`. Then delete one conversation and all its facts with `DELETE $API/v1/default/banks/$BANK/documents/<document_id>`.
- **Moving facts in from another memory store:** back up both sides first. Save each fact with the `hindsight_retain` tool, or `POST $API/v1/default/banks/$BANK/memories` with `{"items":[{"content":"..."}]}`. Recall-test every one, and only then delete the old copy.
- **Built-in memory.** The docs suggest turning off Hermes' built-in memory so the model doesn't favor it over Hindsight: `hermes config set memory.memory_enabled false` for `MEMORY.md`, plus `hermes config set memory.user_profile_enabled false` for `USER.md`. Keeping them on for rules that must apply in every turn is also reasonable. Choose deliberately. **Docs.**
- **Logs:** `~/.hermes/logs/hindsight-embed.log` (daemon startup) and `~/.hindsight/profiles/<profile>.log` (runtime).

---

## 10. Pitfalls we hit

All **Tested**.

| Symptom | Cause | Fix |
| --- | --- | --- |
| Daemon start times out on first use | `uvx` downloading the server | `uv tool install 'hindsight-api==1.2.3'` (your version) first |
| Every save fails, with `501 structured output is unavailable` in the log | Ollama model or runner without JSON-schema support | Use a llama.cpp-served tag; run the test in 3.1 |
| Facts are saved, but recall says "No relevant memories found" | `recall_types` defaults to observations only, and none exist yet | `"recall_types": "observation,world,experience"` |
| Every question returns every memory | No relevance floor | Calibrated `recall_min_scores` ([section 7](#7-tune-recall)) |
| A config change has no effect in Telegram or Discord | The gateway is still running the old config | `hermes gateway restart` from a normal shell |
| A profile env change has no effect | The running daemon doesn't reload its environment | `hindsight-embed -p hermes daemon stop`, then let Hermes restart it |
| Memory fills with notes about the setup itself | Auto-save stored the setup conversation | Delete that conversation's document ([section 9](#9-backups-and-maintenance)) |
| Restore fails with "Target bank already exists" | Restore never merges | Use `?target_bank_id=<new, unused name>` |

---

## Sources

Read on November 22, 2026. Check them for changes.

- Hindsight × Hermes integration: https://hindsight.vectorize.io/sdks/integrations/hermes
- Hindsight best practices: https://hindsight.vectorize.io/best-practices
- Memory banks, bank config, export and import: https://hindsight.vectorize.io/developer/api/memory-banks
- Recall API: https://hindsight.vectorize.io/developer/api/recall
- Observations: https://hindsight.vectorize.io/developer/observations
- Models and providers: https://hindsight.vectorize.io/developer/models
- Installation (Docker, PostgreSQL, hardware, worker ID): https://hindsight.vectorize.io/developer/installation
- Configuration (auth, document text, traces, control plane): https://hindsight.vectorize.io/developer/configuration
- Running Hindsight with Ollama: https://hindsight.vectorize.io/blog/2026/03/10/run-hindsight-with-ollama
- Leaderboards: https://benchmarks.hindsight.vectorize.io/leaderboards
- Hermes memory providers: https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers
- Hermes plugin catalog entry: https://github.com/NousResearch/hermes-agent/blob/main/plugin-catalog/hindsight.yaml
- Ollama model tags: https://ollama.com/library/gemma4/tags · Ollama FAQ (keep-alive, parallel requests): https://docs.ollama.com/faq

Not affiliated with Vectorize or Nous Research. Released under the MIT License.
