# Workshop Worksheet
## Build Your Own Second Brain with Nanobot + Telegram

> **Goal by end of workshop:** A working AI agent running on your machine, connected to your Telegram, configured as a second brain on a topic of your choice. You can ingest sources into it and query it from anywhere.

> ⚠️ **Reality check up front.** Nanobot v0.2.0 has a hardcoded 120-second HTTP timeout when talking to the LLM (GitHub issue [HKUDS/nanobot#3455](https://github.com/HKUDS/nanobot/issues/3455)). On a Raspberry Pi or any CPU-only machine, local Ollama models will time out for all but the smallest prompts. The reliable path on modest hardware is **Nanobot on your machine + OpenRouter for the LLM** — same wiki, same memory, same Telegram experience, just delegating the "thinking" step to the cloud where it takes 2 seconds instead of 2+ minutes. Both paths are covered below. Pick based on your hardware.

---

## Before We Start — Prerequisites Checklist

- [ ] Machine with **Python ≥ 3.11** (`python3 --version` to check)
- [ ] Mac, Linux, Windows (WSL), or Raspberry Pi 4/5 (4 GB RAM minimum, 8+ recommended)
- [ ] **Telegram account** installed on your phone
- [ ] ~8 GB free disk space if running local Ollama; ~1 GB if cloud-only
- [ ] Internet access (for OpenRouter, model downloads, Telegram polling)

Note your machine details below:
- OS / hardware: _______________
- Python version: _______________
- RAM: _______________
- Telegram username: _______________

> 💡 **Hardware sanity table** — what to expect:
>
> | Machine | Local Ollama? | Cloud LLM via OpenRouter? |
> |---|---|---|
> | Modern laptop with NVIDIA GPU (≥6 GB VRAM) | ✅ Fast | ✅ Faster |
> | Apple Silicon Mac (M1/M2/M3) | ✅ Usable | ✅ Faster |
> | CPU-only laptop / Intel Mac | 😐 Slow but works for small models | ✅ Recommended |
> | **Raspberry Pi 5 (16 GB)** | ❌ Times out (use cloud) | ✅ **Recommended** |
> | Raspberry Pi 4 (4–8 GB) | ❌ Times out | ✅ Required |
> | Pi 3 or older | ❌ Don't bother | ✅ Required |

---

## Part 1 — Concepts (Fill in as we go)

**1.1 The problem with vanilla LLMs:**

LLMs forget everything between sessions. Standard RAG re-derives knowledge on every query. Nothing _____________________.

**1.2 The LLM Wiki Pattern (Karpathy):**

The LLM **incrementally builds and maintains a persistent wiki** between you and the raw sources. The wiki **compounds**.

Three layers:
1. **Raw sources** — _______________ (articles, notes, PDFs you drop in)
2. **The wiki** — _______________ (LLM-generated markdown, summaries, entity pages)
3. **The schema** — _______________ (rules in `AGENTS.md` for how the LLM maintains the wiki)

**1.3 Three operations:**
- **Ingest:** _____________________________________________
- **Query:** _____________________________________________
- **Lint:** _____________________________________________

**1.4 Why local matters (and when it doesn't):**

Running a personal AI on your own hardware gives you privacy, no per-message cost, no rate limits. Trade-off: speed and quality of replies are bounded by your hardware.

A pragmatic hybrid: **the wiki, memory, and rules all live on your machine** (the parts that make it *your* brain), while the LLM "thinking" step happens wherever it's fastest. Singapore's Foreign Minister Vivian Balakrishnan published his personal AI second brain in April 2026 — Claude (cloud) + Raspberry Pi (orchestration) + WhatsApp + the LLM Wiki pattern. His quote:

> *"The diplomat who learns to work with AI will have a meaningful edge. I think that edge is now."*

Today you build the same pattern. You decide where the LLM runs based on your hardware.

---

## Part 2 — Hands-On

### Activity 1: Install Ollama + Nanobot (10 min)

> Even if you plan to use cloud LLMs (OpenRouter), installing Ollama is harmless — it just won't be the active provider. You can skip Ollama entirely if you're certain you're staying cloud-only.

**Step 1.1 — Install Ollama** *(skip if going cloud-only or already installed)*

- **Linux / Pi:** `curl -fsSL https://ollama.com/install.sh | sh`
- **Mac:** Download from https://ollama.com/download (installs `Ollama.app` + the CLI)
- **Windows:** Download installer from https://ollama.com/download (WSL works too)

Verify: `ollama --version`

On Linux/Pi, Ollama installs itself as a system service. Confirm:
```
systemctl status ollama
```
Look for `active (running)`. Press `q` to exit. If not running: `sudo systemctl start ollama`.

> 🤓 **Why a service?** A "service" is a program managed by the OS — it auto-starts on boot, restarts on crash, and runs in the background without owning a terminal. Ollama needs to be listening constantly to answer requests.

**Step 1.2 — Pull a model** *(skip if going cloud-only)*

```
ollama pull ministral-3:latest
ollama pull ministral-3:3b
```

> 💡 **Pick what's current.** Model names age fast. Check what's available with `ollama list` after pulling, or browse https://ollama.com/library. Older guides may reference `mistral` or `gemma:2b`; in 2026 the better defaults are `ministral-3:latest` (6 GB, smart), `ministral-3:3b` (3 GB, faster), or `qwen3.5:4b` (3.4 GB, a different family worth comparing).

> ⚠️ **Pi users:** Stick to the 3B-sized models. The 6 GB Mistral-3 will load but be too slow to fit nanobot's 120s timeout window. We'll use OpenRouter instead.

Verify both pulled: `ollama list`

**Step 1.3 — Install Nanobot**

```
pip install nanobot-ai
```

If you hit **`error: externally-managed-environment`** (modern Linux/Pi OS), do one of:

- **Recommended:** Use `pipx` for isolated install
  ```
  sudo apt install pipx -y
  pipx ensurepath
  pipx install nanobot-ai
  ```
  Then **open a new terminal** so PATH changes take effect.
- **Quick & dirty:** `pip install nanobot-ai --break-system-packages`

Verify: `nanobot --version` → should print `🐈 nanobot v0.2.x`

**Step 1.4 — Initialize**

```
nanobot onboard
```

Creates `~/.nanobot/config.json` and a workspace at `~/.nanobot/workspace/` containing:

| File | Purpose |
|---|---|
| `AGENTS.md` | **The rules** — defines how the AI behaves (you'll edit this in Activity 4) |
| `SOUL.md` | The AI's identity / persona prompt |
| `USER.md` | What the AI knows about you |
| `TOOLS.md` | Description of available tools (v0.2.0+) |
| `HEARTBEAT.md` | Periodic self-check prompt (v0.2.0+) |
| `memory/MEMORY.md` | **The wiki** — your second brain's persistent notebook |
| `memory/history.jsonl` | Chronological log of all interactions |
| `.git/` | Workspace is auto-versioned — every change is a git commit |

> 💡 **The `.git/` is genuinely useful.** Every wiki edit becomes a git commit. If the AI ever corrupts something, `cd ~/.nanobot/workspace && git log` shows the history, and you can roll back with standard git commands.

✅ **Checkpoint:** Run `nanobot --version` and (if you installed Ollama) `ollama list`. Both work.

---

### Activity 2: Configure & First Chat (10 min)

> The v0.2.0 config file (~11 KB) ships with **every provider pre-listed** with empty fields. You just fill in the ones you want — you don't add new sections. Older guides showed creating a custom `"local"` provider; that's no longer the pattern.

**Step 2.1 — Back up the config** (good habit, costs nothing)

```
cp ~/.nanobot/config.json ~/.nanobot/config.json.backup
```

**Step 2.2 — Choose your path: Cloud (recommended for Pi/CPU) or Local**

#### Path A — Cloud LLM via OpenRouter (recommended for Pi/CPU)

1. Create a free account at https://openrouter.ai
2. Go to https://openrouter.ai/keys → **Create Key**
3. Copy the key (starts with `sk-or-v1-`). Treat it like a password.

Then run this script. Save it to a file (here-docs and `getpass` don't mix when piped to Python directly):

```
cat > /tmp/configure_openrouter.py <<'PYEOF'
import json, getpass
from pathlib import Path

config_path = Path.home() / ".nanobot" / "config.json"
config = json.loads(config_path.read_text())

key = getpass.getpass("Paste OpenRouter API key (hidden): ").strip()

config["providers"]["openrouter"]["apiKey"] = key
config["providers"]["openrouter"]["apiBase"] = "https://openrouter.ai/api/v1"

config["agents"]["defaults"]["provider"] = "openrouter"
config["agents"]["defaults"]["model"] = "openrouter/free"
config["agents"]["defaults"]["maxTokens"] = 1024
config["agents"]["defaults"]["maxToolIterations"] = 10

config_path.write_text(json.dumps(config, indent=2))
print("✓ Configured for OpenRouter")
PYEOF

python3 /tmp/configure_openrouter.py
```

> 💡 **Why `openrouter/free`?** It's an auto-router that picks whichever free model is currently available. Individual free models (e.g. `deepseek/deepseek-v4-flash:free`) often hit 429 rate-limited; the auto-router falls through to whatever has capacity. Once you know which models work for you, you can pin a specific one. If free tier becomes too jammed, top up $10 (one-time, never expires) and switch to `anthropic/claude-haiku-4-5` — roughly $0.001 per workshop-style message.

#### Path B — Local Ollama (laptop with GPU, or Apple Silicon)

```
python3 <<'PYEOF'
import json
from pathlib import Path
config_path = Path.home() / ".nanobot" / "config.json"
config = json.loads(config_path.read_text())

config["providers"]["ollama"]["apiKey"] = "ollama"
config["providers"]["ollama"]["apiBase"] = "http://localhost:11434/v1"

config["agents"]["defaults"]["provider"] = "ollama"
config["agents"]["defaults"]["model"] = "ministral-3:latest"
config["agents"]["defaults"]["maxTokens"] = 1024
config["agents"]["defaults"]["maxToolIterations"] = 10
config["agents"]["defaults"]["contextWindowTokens"] = 16384

config_path.write_text(json.dumps(config, indent=2))
print("✓ Configured for local Ollama")
PYEOF
```

**Step 2.3 (Local path only) — Tell Ollama to load with a bigger context window**

Nanobot's default system prompt is ~6500 tokens. Ollama's default model context is 4096 → silent truncation → garbled replies. Critical fix:

```
sudo systemctl edit ollama
```

In the empty `[Service]` section, paste:
```
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=16384"
Environment="OLLAMA_KEEP_ALIVE=24h"
Environment="OLLAMA_NUM_PARALLEL=1"
```

Save (`Ctrl+O`, Enter, `Ctrl+X`). Reload:
```
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

> 🤓 **What these do:**
> - `OLLAMA_CONTEXT_LENGTH=16384` — quadruples the model's context window so nanobot's system prompts fit without truncation
> - `OLLAMA_KEEP_ALIVE=24h` — keeps loaded models warm in RAM (default unloads after 5 min of inactivity)
> - `OLLAMA_NUM_PARALLEL=1` — one request at a time, so each gets full CPU/GPU. Trade-off: no concurrency.

**Step 2.4 — Sanity-test the LLM independent of nanobot**

This isolates "is the LLM reachable?" from "is nanobot working?" — invaluable for debugging.

**Cloud path:**
```
KEY=$(python3 -c "import json; print(json.load(open('/home/$USER/.nanobot/config.json'))['providers']['openrouter']['apiKey'])")
curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"openrouter/free","messages":[{"role":"user","content":"say hi in 3 words"}]}'
```

You should see JSON with a `"content"` field containing the reply, in 2–10 seconds.

**Local path:**
```
ollama run ministral-3:latest "say hi briefly"
```

You should see a short reply within ~60 seconds (CPU) or ~5 seconds (GPU).

Then verify the model is loaded with the new settings:
```
ollama ps
```
**`CONTEXT` column should show 16384** (not 4096), and `UNTIL` should be 24 hours from now.

**Step 2.5 — First chat via nanobot itself**

```
nanobot agent -m "Hello — tell me what you are in one sentence."
```

Be patient on first run (loads workspace, calls LLM). Should reply within ~15s cloud, ~60s local-with-GPU.

✅ **Checkpoint:** Both the sanity-test curl/`ollama run` AND `nanobot agent -m` return a response.

Notes / observations:
___________________________________________________
___________________________________________________

---

### Activity 3: Telegram Gateway (15 min)

The gateway is a long-running process that polls Telegram for new messages, routes them through your AI, and replies. It must stay running while you want the bot online.

**Step 3.1 — Create a bot with BotFather** (on your phone or desktop Telegram)

1. Search Telegram for **`@BotFather`** (look for the blue verified ✓)
2. Send `/newbot`
3. Pick a display name (e.g. `My Second Brain`)
4. Pick a username ending in `bot` (e.g. `my_second_brain_bot`) — must be unique across Telegram
5. **Copy the bot token** — looks like `1234567890:ABCdef...`. Treat as a password.

Paste your token here (privately): _____________________________________

**Step 3.2 — Get your numeric user ID** (also in Telegram)

1. Search Telegram for **`@userinfobot`** → tap Start
2. It immediately replies with your info including a numeric `Id:`
3. Copy the numeric ID (e.g. `7045522516`)

> ⚠️ **Why this is critical:** Without your ID in `allowFrom`, your bot ignores you. **More importantly:** without `allowFrom` set at all, your bot answers anyone who finds it. That's a free LLM proxy for the world.

Paste your user ID here: _____________________________________

**Step 3.3 — Plug into config (safely)**

Save the script to a file so `getpass` works:

```
cat > /tmp/configure_telegram.py <<'PYEOF'
import json, getpass
from pathlib import Path

config_path = Path.home() / ".nanobot" / "config.json"
config = json.loads(config_path.read_text())

token = getpass.getpass("Paste bot token (hidden): ").strip()
user_id = input("Paste numeric user ID: ").strip()

config["channels"]["telegram"]["enabled"] = True
config["channels"]["telegram"]["token"] = token
config["channels"]["telegram"]["allowFrom"] = [user_id]

config_path.write_text(json.dumps(config, indent=2))
print("✓ Telegram configured")
PYEOF

python3 /tmp/configure_telegram.py
```

Verify (without exposing the token):
```
python3 -c "
import json
c = json.load(open('/home/$USER/.nanobot/config.json'))['channels']['telegram']
print('enabled:', c['enabled'])
print('token has colon:', ':' in c['token'])
print('allowFrom:', c['allowFrom'])
"
```

Should show `enabled: True`, `token has colon: True`, and your ID in `allowFrom`.

**Step 3.4 — Start the gateway**

```
nanobot gateway
```

Wait for:
```
✓ Channels enabled: telegram
... | INFO | telegram | bot @your_bot_name connected
```

**Leave this terminal alone** — it's the bot's lifeline.

**Step 3.5 — Test from Telegram**

In your Telegram app, search for your bot → tap **Start**. Send `hi`.

In the gateway terminal you should see:
```
... | INFO | Processing message from telegram:<your-id>|<session>: hi
... | INFO | Response to telegram:<your-id>|<session>: <the reply>
```

And on your phone, the bot's reply.

> 💡 **Subtle thing:** The bot may take 30s+ on the first message even on cloud, because the workspace files have to be parsed and the conversation session initialized. Subsequent messages are much faster.

✅ **Checkpoint:** Bot replies to `hi` in Telegram. Your phone is now a terminal for your AI.

**Troubleshooting:**

| Symptom | Likely cause | Fix |
|---|---|---|
| Bot online but silent | `allowFrom` doesn't match your user ID, or wrong format | Re-run step 3.3, ensure ID is in quoted-string form inside the list |
| "Connection refused" | Ollama not running (local path) | `sudo systemctl start ollama` |
| `LLM transient error: request timed out` after exactly 2 min | Nanobot's hardcoded 120s HTTP timeout (issue #3455) on a slow local LLM | Switch to OpenRouter (Path A in Activity 2) |
| `LLM truncating input prompt` in `journalctl -u ollama` | `OLLAMA_CONTEXT_LENGTH` not applied | Re-do Step 2.3 carefully, then `sudo systemctl restart ollama` |
| 429 `rate-limited upstream` from OpenRouter | Free model upstream provider busy | Use `openrouter/free` (auto-router) instead of a pinned model |
| Reply contains weird `reasoning` text | Free reasoning model leaking chain-of-thought | Pin to a non-reasoning model like `meta-llama/llama-3.3-70b-instruct:free` |

---

### Activity 4: Configure Your Second Brain Schema (15 min)

This is the most important step. `AGENTS.md` is what turns a generic chatbot into a disciplined wiki maintainer.

**Step 4.1 — Pick your topic** (2 min)

Pick something narrow enough that "the wiki" makes sense, broad enough that sources exist. Examples:
- *"Stoic philosophy"*
- *"Singapore foreign policy 2020–present"*
- *"My fitness training program"*
- *"Premier League tactics 2025–26"*
- *"My PhD thesis on protein folding"*
- *"The Mandalorian (TV show) lore"*

Bad picks: too broad ("AI", "philosophy"), no sources available ("my deepest thoughts").

My topic: _____________________________________________

**Step 4.2 — Write a tight AGENTS.md** (13 min)

> ⚠️ **The strength of the schema matters a lot.** Loose schemas like *"answer from the wiki when possible"* are routinely ignored by smaller models. Schemas with **MANDATORY**, **EXCLUSIVELY**, **NEVER**, plus a worked example showing right vs. wrong behaviour, get consistent compliance.

Use Python to write the file (avoids the multiline-here-doc trap):

```
python3 <<'PYEOF'
from pathlib import Path

TOPIC = "Stoic Philosophy"  # ← change this

content = f"""# Agent Schema — Second Brain for {TOPIC}

You are a strict **wiki maintainer**, not a general chatbot.
You answer EXCLUSIVELY from the wiki at `memory/MEMORY.md`.
You do NOT answer from general knowledge, ever.

## MANDATORY first step on every message

Before responding to ANY user question, you MUST:
1. Call the `read_file` tool with path `memory/MEMORY.md`
2. Read its contents
3. ONLY answer based on what's in the file

If the wiki has no relevant entry, you say so. You do NOT fall back to training data.

## Wiki structure (inside MEMORY.md)
- `## Index`          — catalog of every entity/concept page with one-line summary
- `## Entities`       — people, organizations, named things relevant to {TOPIC}
- `## Concepts`       — ideas, frameworks, recurring themes
- `## Sources`        — what you've ingested, with date and 2-3 sentence summary
- `## Open Questions` — contradictions and things to investigate
- `## Synthesis`      — your evolving overall picture

## When the user pastes a source or says "ingest"
1. Read the source carefully.
2. Extract: entities, concepts, key claims, dates.
3. Use the `edit_file` tool to update MEMORY.md:
   - Add an entry under `## Sources` (with title and date)
   - For each entity/concept mentioned, update or create its entry
   - If new info contradicts existing info, note it under `## Open Questions`
   - Update `## Index`
4. Reply with a 3-bullet summary of what changed.

## When the user asks a question (the normal case)
1. ALWAYS read MEMORY.md first using `read_file`.
2. If the wiki has the answer, respond with it. Cite the source/entity/concept page.
3. If the wiki does NOT have the answer, respond EXACTLY in this format:
   "The wiki doesn't cover [topic] yet. Would you like to ingest a source about it?"
4. Do NOT add general knowledge to fill the gap. Do NOT recite facts from training.

## Example interaction (FOLLOW THIS PATTERN)

User: "Who was [some figure relevant to {TOPIC}]?"

You (correct):
[calls read_file on memory/MEMORY.md]
[finds nothing]
"The wiki doesn't cover [that figure] yet. Would you like to ingest a source about them?"

You (WRONG — never do this):
"[Recites a Wikipedia-style summary from training data]"

## When the user says "lint" or "health check"
Read MEMORY.md. Report:
- Contradictions between entries
- Concepts mentioned without their own page
- Open questions that have since been answered

Suggest fixes. Do NOT apply them without explicit confirmation.

## Style
- Concise. Markdown. Bullets over prose.
- Cite sources by their title in `## Sources`.
- Scoped to {TOPIC} ONLY. Refuse all other topics politely.
"""

Path.home().joinpath(".nanobot/workspace/AGENTS.md").write_text(content)
print(f"✓ AGENTS.md written: {{len(content)}} chars")
PYEOF
```

(Edit the `TOPIC = ...` line for your own subject.)

**Step 4.3 — Initialize MEMORY.md with the scaffold**

The AI works better with a properly-formed wiki on first read than with the boilerplate `nanobot onboard` placeholder:

```
python3 <<'PYEOF'
from pathlib import Path

TOPIC = "Stoic Philosophy"  # ← match above

content = f"""# Second Brain: {TOPIC}

## Index
*(no entries yet — ingest sources to populate this)*

## Entities
*(no entries yet)*

## Concepts
*(no entries yet)*

## Sources
*(no entries yet)*

## Open Questions
*(no entries yet)*

## Synthesis
*(no entries yet — synthesis emerges after a few ingests)*
"""
Path.home().joinpath(".nanobot/workspace/memory/MEMORY.md").write_text(content)
print(f"✓ MEMORY.md scaffolded: {{len(content)}} chars")
PYEOF
```

**Step 4.4 — Restart the gateway**

In the gateway terminal: `Ctrl+C`. Then:
```
nanobot gateway
```

**Step 4.5 — Test the new personality**

Ask via Telegram a question on your topic that the empty wiki couldn't possibly answer:
> *Who was [some named figure]?*

You're looking for two signs of correctness in the gateway logs:
1. A `Tool call: read_file({"path": "memory/MEMORY.md"})` line
2. A reply matching: `The wiki doesn't cover [topic] yet. Would you like to ingest a source about it?`

If instead the bot rattles off facts from training data — the schema isn't being followed. Likely culprits: the model is a small/loose one (try pinning to `meta-llama/llama-3.3-70b-instruct:free` or upgrading to a paid model), or the schema's MANDATORY/EXCLUSIVELY language was softened during your edit.

Then test scoping with an off-topic message:
> *What's the weather in Singapore?*

Expected: polite refusal.

✅ **Checkpoint:** Bot reads MEMORY.md before answering, admits empty wiki, refuses off-topic.

---

### Activity 5: Ingest & Query via Telegram (10 min)

**Step 5.1 — Ingest a source**

From Telegram, send a single message in this format (include both the trigger phrase and the title/date — they go straight into the Sources section):

```
Ingest this as a source:

[paste a paragraph or article relevant to your topic — your own notes, a Wikipedia
intro, a tweet thread, a section from a PDF, whatever]

Source title: [a short title]
Date: [today's date]
```

In the gateway terminal, watch the sequence:
- `Tool call: read_file({"path": "memory/MEMORY.md"})`
- `Tool call: edit_file({...})` — possibly multiple
- `Response to telegram: ...` — a 3-bullet summary of what changed

> ⚠️ **Watch the iteration count.** Complex sources may trigger many `edit_file` calls. If you hit *"reached maximum number of tool call iterations"*, bump `maxToolIterations` in config — 10 is reasonable, 15 generous, 20+ if you regularly ingest large multi-entity sources.

**Step 5.2 — Verify the wiki was updated**

In a second terminal:
```
cat ~/.nanobot/workspace/memory/MEMORY.md
```

You should see real entries under `## Sources`, `## Entities`, `## Concepts`, and matching index lines. The `*(no entries yet)*` placeholders should be gone from any section the source touched.

**Step 5.3 — Query the wiki**

In Telegram, ask the **exact same question** you asked in step 4.5 (the one that previously returned "wiki doesn't cover this yet"):
> *Who was [the figure you just ingested about]?*

This time the bot should:
- `read_file` on MEMORY.md
- Find the entry
- Answer from it, citing the Source title

**The answer should now reference YOUR sources, not generic knowledge.**

**Step 5.4 — Test compounding (the real magic)**

Ask a question about a *concept* or *secondary entity* mentioned in your source but that you didn't explicitly query for before:

> *What is [a concept from the ingested source]?*

If the schema worked, the AI extracted that concept into its own page during ingestion. So even though you only ingested *one* source, *multiple* things became queryable. That's the compounding.

**Step 5.5 — Try a lint**

Send to the bot:
> *Run a lint pass on the wiki. Report any issues, but do not apply fixes yet.*

The bot should read MEMORY.md and flag any duplications, orphan index entries, or missing pages. You can then approve specific fixes:
> *Apply the fix for [issue X].*

Watch the gateway logs — multiple `edit_file` calls, then a summary.

✅ **Checkpoint:** Your second brain has at least one source ingested, answers queries grounded in the wiki, and the lint+approve loop works.

---

## Part 3 — From Single Agent to Multi-Agent (10 min)

A single agent handles ingest, query, and lint sequentially. For larger second brains, split the work across **sub-agents** specialized for each task.

The pattern:
```
        ┌─────────────────────┐
        │  CURATOR (default)  │  ← talks to you on Telegram
        │  - owns the wiki    │
        │  - delegates work   │
        └──┬───────┬────────┬─┘
           │       │        │
   ┌───────▼───┐ ┌─▼─────┐ ┌▼──────────┐
   │RESEARCHER │ │LINTER │ │ INGESTER  │
   │(web tool) │ │       │ │           │
   └───────────┘ └───────┘ └───────────┘
```

In nanobot, sub-agents are defined as additional entries under the `agents` block in `config.json`. **Note:** v0.2.0 nanobot uses a slightly different sub-agent format than older versions — consult `https://nanobot.wiki` for the current canonical form. Skeleton concept (don't copy verbatim — adapt to current docs):

```
"agents": {
  "defaults": {
    "provider": "openrouter",
    "model": "openrouter/free"
  },
  "researcher": {
    "model": "meta-llama/llama-3.3-70b-instruct:free",
    "systemPrompt": "You research topics on the web. Return findings as bullet points with URLs."
  },
  "linter": {
    "model": "qwen/qwen3-coder:free",
    "systemPrompt": "You audit a markdown wiki for contradictions, orphans, and stale claims."
  }
}
```

Then in your **curator's AGENTS.md** you'd add:
```
When you need fresh facts not in the wiki, delegate to the `researcher` sub-agent.
When the user asks for a lint, delegate to the `linter` sub-agent.
Always file results back into MEMORY.md yourself — the sub-agents do not write to the wiki.
```

**Discussion (5 min):**
- What sub-agents would make sense for *your* topic?
- Note them here:
    ___________________________________________________
    ___________________________________________________

---

## Part 4 — Going Further

**Easy wins after the workshop:**
- [ ] **Run the gateway as a service** so it survives reboots. See systemd snippet in the Reference Card below.
- [ ] **Pin a known-good model** once you've found which free or paid model best follows your schema. Auto-router is forgiving but inconsistent.
- [ ] **Add web search** as a tool (Brave / Serper / Tavily API key → free tier exists). Lets your agent pull live info before ingesting.
- [ ] **Add an MCP server** — Gmail, Calendar, Notion, GitHub. See `docs/configuration.md` in the nanobot repo. Lots of MCP servers exist in the wild.
- [ ] **Schedule a daily lint** via nanobot cron. The cron config in v0.2.0 is at the top level; check current docs for syntax.
- [ ] **Back up your wiki off-machine** — `~/.nanobot/workspace/` is already a git repo. Add a remote (GitHub, Gitea, anything) and push regularly. Free version history.

**The bigger idea (Karpathy):**
> *The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch many files in one pass. The wiki stays maintained because the cost of maintenance is near zero.*

**The bigger idea (Balakrishnan):**
> *The specific composition you used today will be obsolete in months. The builder's ability to compose the right pieces — that's the durable advantage.*

You now have that ability. **Keep building.**

---

## Reference Card — Commands

```
# Daily start (only the gateway needs starting on most systems;
# ollama auto-starts via systemd if you went the local route)
nanobot gateway

# One-shot test from the terminal (no Telegram needed)
nanobot agent -m "your message"

# View the wiki
cat ~/.nanobot/workspace/memory/MEMORY.md

# View the conversation history
cat ~/.nanobot/workspace/memory/history.jsonl

# Wiki git history (workspace is auto-versioned)
cd ~/.nanobot/workspace && git log --oneline

# Roll back a bad AI edit
cd ~/.nanobot/workspace && git checkout HEAD~1 -- memory/MEMORY.md

# Reset memory entirely (nuclear)
rm ~/.nanobot/workspace/memory/MEMORY.md
nanobot onboard   # recreates default

# List local Ollama models (local path only)
ollama list

# See what's loaded in RAM right now
ollama ps

# Switch to a different model — edit ~/.nanobot/config.json
# Change agents.defaults.model to the new model id

# Quick LLM sanity-test (cloud)
KEY=$(python3 -c "import json,os; print(json.load(open(os.path.expanduser('~/.nanobot/config.json')))['providers']['openrouter']['apiKey'])")
curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"openrouter/free","messages":[{"role":"user","content":"hi"}]}'

# Quick LLM sanity-test (local)
ollama run ministral-3:latest "hi"
```

---

## Reference Card — Auto-start the gateway on boot (systemd)

```
sudo tee /etc/systemd/system/nanobot-gateway.service > /dev/null <<EOF
[Unit]
Description=Nanobot Gateway
After=network.target ollama.service
Wants=ollama.service

[Service]
Type=simple
User=$USER
ExecStart=$HOME/.local/bin/nanobot gateway
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable nanobot-gateway
sudo systemctl start nanobot-gateway
sudo systemctl status nanobot-gateway
sudo journalctl -u nanobot-gateway -f   # tail live logs
```

---

## Reference Card — File Locations

| What | Where |
|---|---|
| Config | `~/.nanobot/config.json` |
| Workspace | `~/.nanobot/workspace/` |
| **Schema (your edits)** | `~/.nanobot/workspace/AGENTS.md` |
| **The wiki** | `~/.nanobot/workspace/memory/MEMORY.md` |
| Conversation log | `~/.nanobot/workspace/memory/history.jsonl` |
| AI identity prompt | `~/.nanobot/workspace/SOUL.md` |
| What the AI knows about you | `~/.nanobot/workspace/USER.md` |
| Tools description (v0.2.0+) | `~/.nanobot/workspace/TOOLS.md` |
| Heartbeat prompt (v0.2.0+) | `~/.nanobot/workspace/HEARTBEAT.md` |
| Git history of workspace | `~/.nanobot/workspace/.git/` |
| Ollama systemd override (local path) | `/etc/systemd/system/ollama.service.d/override.conf` |

---

## Reference Card — Resources

- **Nanobot:** https://github.com/HKUDS/nanobot
- **Nanobot docs:** https://nanobot.wiki
- **Karpathy's LLM Wiki gist:** https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- **Ollama:** https://ollama.com
- **Ollama model library:** https://ollama.com/library
- **OpenRouter:** https://openrouter.ai (free models at https://openrouter.ai/collections/free-models)
- **MCP (for adding tools):** https://modelcontextprotocol.io
- **Known issue: nanobot 120s timeout:** https://github.com/HKUDS/nanobot/issues/3455

---

## Notes & Questions

*Use this space for your own notes, things to try later, and questions for the instructor.*

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________

___________________________________________________
