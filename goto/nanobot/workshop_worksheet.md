# Workshop Worksheet
## Build Your Own Second Brain with Nanobot + Telegram

> **Goal by end of workshop:** A working AI agent running on your machine, connected to your Telegram, configured as a second brain on a topic of your choice. You can ingest sources into it and query it from anywhere.

> ⚠️ **Reality check up front.** Nanobot defaults to a 120-second HTTP timeout on LLM calls (GitHub issue [HKUDS/nanobot#3455](https://github.com/HKUDS/nanobot/issues/3455)). On a Raspberry Pi this default kills any non-trivial local Ollama request. **The worksheet handles this for you in Path B (Step B.3)** by raising the timeout via env vars before you ever hit it — so local Ollama is a first-class path here, not a hack. The trade-off is genuine, though: local replies on a Pi take **minutes**, where cloud replies take **seconds**. Pick the path that matches your priorities: cloud for speed and ease, local for privacy and offline operation.

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
> | Machine | Local Ollama (Path B) | Cloud via OpenRouter (Path A) |
> |---|---|---|
> | Modern laptop with NVIDIA GPU (≥6 GB VRAM) | ✅ Fast (~5-15s/reply) | ✅ Fast (~5-15s) |
> | Apple Silicon Mac (M1/M2/M3) | ✅ Usable (~10-30s/reply) | ✅ Fast (~5-15s) |
> | CPU-only laptop / Intel Mac | 😐 Slow (~30-90s/reply) | ✅ Fast (~5-15s) |
> | **Raspberry Pi 5 (16 GB)** | 😐 Works, slow (~1-4 min/reply) | ✅ Fast (~5-15s) |
> | Raspberry Pi 4/5 (8 GB) | 😐 Works, slow (~30-90s/reply) | ✅ Fast (~5-15s) |
> | Raspberry Pi 4 (4 GB) | ⚠️ Tight — small model only (~30-90s/reply, weaker quality) | ✅ Fast (~5-15s) |
> | Pi 3 or older | ❌ Don't bother | ✅ Required |
>
> Path B (local) is genuinely usable on a Pi after the Step B.3 timeout fix. The trade-off is reply *speed*, not whether it works at all. Pick local for privacy/offline; pick cloud for snappiness.

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

Which model to pull depends on your machine's RAM. Pick **one** row from this table:

| Machine | Model | Disk size | Approx. RAM when loaded |
|---|---|---|---|
| Pi 4 (4 GB) / very tight | `qwen2.5:1.5b` | ~1 GB | ~2 GB |
| Pi 4 (8 GB) / Pi 5 (8 GB) | `qwen2.5:3b` | ~2 GB | ~3 GB |
| **Pi 5 (16 GB)** | `qwen2.5:7b` | ~4.7 GB | ~6 GB |
| Laptop GPU (≥6 GB VRAM) or Mac (≥16 GB) | `qwen2.5:7b` | ~4.7 GB | ~6 GB |
| Apple Silicon (32 GB+) / big GPU | `qwen2.5:14b` | ~9 GB | ~11 GB |

Pull just the one row that matches you. Example for a Pi 5 16 GB:

```
ollama pull qwen2.5:7b
```

> 💡 **Why Qwen 2.5?** Good instruction-following at every size, solid tool-use behaviour, available across the size range so the same family scales with your hardware. The `:Nb` tag in Ollama is the parameter count — bigger numbers are smarter but need more RAM and run slower. Don't pick a bigger size than your table row suggests; you'll either OOM or wait minutes per token.

> ⚠️ **Pi 4 with 4 GB RAM:** You can run `qwen2.5:1.5b` but quality will be visibly weaker — the 1.5B model often misses tool calls or rambles. If your topic is important to you, the cloud path (Path A) is the better experience on this hardware. If you specifically want offline operation, accept the trade-off.

Verify the pull: `ollama list` — your model should appear.

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
| `TOOLS.md` | Description of available tools |
| `HEARTBEAT.md` | Periodic self-check prompt |
| `memory/MEMORY.md` | **The wiki** — your second brain's persistent notebook |
| `memory/history.jsonl` | Chronological log of all interactions |
| `.git/` | Workspace is auto-versioned — every change is a git commit |

> 💡 **The `.git/` is genuinely useful.** Every wiki edit becomes a git commit. If the AI ever corrupts something, `cd ~/.nanobot/workspace && git log` shows the history, and you can roll back with standard git commands.

✅ **Checkpoint:** Run `nanobot --version` and (if you installed Ollama) `ollama list`. Both work.

---

### Activity 2: Configure & First Chat (10 min)

> The current nanobot config file (~11 KB) ships with **every provider pre-listed** with empty fields. You just fill in the ones you want — you don't add new sections. Older guides showed creating a custom `"local"` provider; that's no longer the pattern.

**Step 2.1 — Back up the config** (good habit, costs nothing)

```
cp ~/.nanobot/config.json ~/.nanobot/config.json.backup
```

**Step 2.2 — Choose your path: Cloud (OpenRouter) or Local (Ollama)**

> Both paths reach the same finish line. **Pick one and follow it through Activity 2 only**; the rest of the workshop is identical. You can switch later by editing two lines in `config.json`.

#### Path A — Cloud LLM via OpenRouter

Good for: getting started quickly, slow CPUs, intermittent local capacity, learning the workshop pattern without a multi-GB model download.

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

#### Path B — Local Ollama

Good for: privacy (nothing leaves your machine), offline operation, no API costs, building skill with local inference. Slower than cloud — minutes per reply on a Pi vs. seconds on cloud. That's a real trade-off, not a bug.

> ⚠️ **Three things must work together** for the local path to be reliable: (1) Ollama loads the model with a *big enough context window*, (2) nanobot points at it correctly, and (3) nanobot's request timeouts are *raised above what Pi-class generation actually takes*. Skipping any of the three causes the most common Pi failure modes (silent truncation, 2-minute timeouts, garbled replies). This is why Path B has more steps than Path A — but each step is a one-time setup.

##### B.1 — Tune Ollama to keep the model loaded with full context

Without this, Ollama uses a 4096-token context (smaller than nanobot's system prompt → garbled replies) and unloads your model after 5 idle minutes (every "first chat" pays a 30-60s reload cost).

```
sudo systemctl edit ollama
```

This opens an editor (nano by default) on an **empty** override file. Add these lines (yes, including the `[Service]` header — there's no pre-existing section to extend):

```
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=16384"
Environment="OLLAMA_KEEP_ALIVE=24h"
Environment="OLLAMA_NUM_PARALLEL=1"
```

Save (`Ctrl+O`, Enter, `Ctrl+X`). Reload Ollama so the new settings take effect:

```
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

> 🤓 **What each variable does:**
> - `OLLAMA_CONTEXT_LENGTH=16384` — quadruples the model's context window so nanobot's system prompts fit without silent truncation.
> - `OLLAMA_KEEP_ALIVE=24h` — keeps loaded models warm in RAM (default unloads after 5 idle minutes).
> - `OLLAMA_NUM_PARALLEL=1` — one request at a time, so each gets full CPU/GPU. Trade-off: no concurrency. Right for a personal Pi.

##### B.2 — Point nanobot at Ollama (with the right model for your hardware)

This script detects your machine's RAM and picks a model that fits. Run it as-is on a Pi; it will pick the right size:

```
python3 <<'PYEOF'
import json
from pathlib import Path

# Detect total RAM. Works on Linux/Pi (reads /proc/meminfo).
def total_ram_gb():
    try:
        with open("/proc/meminfo") as f:
            for line in f:
                if line.startswith("MemTotal:"):
                    return int(line.split()[1]) / 1024 / 1024
    except FileNotFoundError:
        return None
    return None

ram = total_ram_gb()

# Pick a model that fits comfortably in available RAM, leaving headroom
# for the OS, nanobot, and the context window itself.
if ram is None:                # non-Linux (Mac) — assume capable
    model = "qwen2.5:7b"
elif ram >= 14:                # Pi 5 16 GB reports ~15.x
    model = "qwen2.5:7b"
elif ram >= 7:                 # Pi 4/5 8 GB
    model = "qwen2.5:3b"
else:                          # Pi 4 4 GB or tight
    model = "qwen2.5:1.5b"

# Write the config
config_path = Path.home() / ".nanobot" / "config.json"
config = json.loads(config_path.read_text())

config["providers"]["ollama"]["apiKey"] = "ollama"
config["providers"]["ollama"]["apiBase"] = "http://localhost:11434/v1"

config["agents"]["defaults"]["provider"] = "ollama"
config["agents"]["defaults"]["model"] = model
config["agents"]["defaults"]["maxTokens"] = 1024
config["agents"]["defaults"]["maxToolIterations"] = 10
config["agents"]["defaults"]["contextWindowTokens"] = 16384

config_path.write_text(json.dumps(config, indent=2))
print(f"✓ Configured for local Ollama with model: {model}")
print(f"  (Detected RAM: {ram:.1f} GB)" if ram else "  (RAM not detected — using default model)")
PYEOF
```

> ⚠️ **Important:** the model the script picks **must match** what you pulled in Step 1.2. If you pulled a different size, edit the config now: open `~/.nanobot/config.json` in any text editor and change the `model` field under `agents.defaults` to the model you actually pulled. Then save.

Verify config:
```
python3 -c "import json, os; c = json.load(open(os.path.expanduser('~/.nanobot/config.json'))); print('model:', c['agents']['defaults']['model'])"
```

You should see `model: qwen2.5:3b` (or whichever size you pulled).

##### B.3 — Raise nanobot's LLM timeouts (essential on CPU)

Nanobot's default is 120s for the HTTP call to the LLM and 300s for the overall agent turn. On a Pi CPU, generating a real reply often takes 2-5 minutes. Without raising these, your bot will time out before the model finishes its first token.

Add the env vars permanently to your shell profile so every `nanobot` invocation inherits them:

```
cat >> ~/.bashrc <<'EOF'

# Nanobot LLM timeouts — needed for slower local inference on CPU
export NANOBOT_OPENAI_COMPAT_TIMEOUT_S=900
export NANOBOT_LLM_TIMEOUT_S=900
EOF

source ~/.bashrc
```

Verify they're set in the current shell:
```
echo "compat=$NANOBOT_OPENAI_COMPAT_TIMEOUT_S llm=$NANOBOT_LLM_TIMEOUT_S"
```

You should see `compat=900 llm=900`. (15 minutes per LLM call. If your Pi can't generate a reply in 15 minutes, you have a different problem — see Troubleshooting at the end of Activity 2.)

> 🤓 **Why both vars?** `NANOBOT_OPENAI_COMPAT_TIMEOUT_S` is the inner HTTP read timeout (httpx → Ollama). `NANOBOT_LLM_TIMEOUT_S` is the outer wait that wraps the whole LLM iteration including retries. Setting only one leaves the other as a cap.

> 💡 **Using zsh (Mac default since Catalina)?** Replace `~/.bashrc` with `~/.zshrc`. Same content, different file.

**Step 2.3 — Sanity-test the LLM independent of nanobot**

This isolates "is the LLM reachable?" from "is nanobot working?" — invaluable for debugging.

**Cloud path:**
```
KEY=$(python3 -c "import json, os; print(json.load(open(os.path.expanduser('~/.nanobot/config.json')))['providers']['openrouter']['apiKey'])")
curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"openrouter/free","messages":[{"role":"user","content":"say hi in 3 words"}]}'
```

You should see JSON with a `"content"` field containing the reply, in 2–10 seconds.

**Local path:**

Read the model your config is set to use, then run it directly. (Hardcoding the model name here would diverge from what your config picked.)

```
MODEL=$(python3 -c "import json, os; print(json.load(open(os.path.expanduser('~/.nanobot/config.json')))['agents']['defaults']['model'])")
echo "Testing with model: $MODEL"
ollama run "$MODEL" "say hi briefly"
```

Wait for the reply. Expected timing:
- Pi 5 (16 GB) running `qwen2.5:7b`: **30-90 seconds** for first response (cold load + generation)
- Pi 4/5 (8 GB) running `qwen2.5:3b`: **15-45 seconds**
- Laptop with GPU: **2-10 seconds**

If nothing happens for 2+ minutes, hit `Ctrl+C` and skip to the Troubleshooting matrix at the end of Activity 2.

Then verify Ollama loaded the model with the new context window:
```
ollama ps
```

Look at the **`CONTEXT`** column in the output. There are three possible outcomes:

| What you see | What it means | Fix |
|---|---|---|
| `CONTEXT` shows **16384** | ✅ Working as intended | Continue to Step 2.5 |
| `CONTEXT` shows **4096** | The `OLLAMA_CONTEXT_LENGTH` env var didn't take effect | Re-do Step B.1 carefully. After `sudo systemctl restart ollama`, confirm with: `systemctl show ollama \| grep Environment` — should list your three variables |
| No model listed (empty table) | Model isn't currently loaded in RAM | Re-run the `ollama run "$MODEL" "say hi"` above. The act of running it loads it. |

**Step 2.4 — First chat via nanobot itself**

```
nanobot agent -m "Hello — tell me what you are in one sentence."
```

Be patient on first run. Expected timing:

| Setup | First reply | Subsequent replies |
|---|---|---|
| Cloud (Path A) | ~10-20s | ~5-15s |
| Local Pi 5 16 GB (`qwen2.5:7b`) | **2-4 min** | 1-2 min |
| Local Pi 4/5 8 GB (`qwen2.5:3b`) | **1-3 min** | 30-90s |
| Local laptop with GPU | ~15-30s | ~5-15s |

The "first reply takes longer" pattern is because nanobot loads workspace files, builds the system prompt, and (for local) Ollama may still be warming up the model on first use. Subsequent messages reuse the warm state and are faster.

> ⚠️ **If you get `LLM transient error: request timed out`** on a Pi: your `NANOBOT_*_TIMEOUT_S` env vars from Step B.3 didn't propagate to nanobot. Verify they're set in *this* shell: `echo $NANOBOT_OPENAI_COMPAT_TIMEOUT_S` should print `900`. If empty, run `source ~/.bashrc` and retry. If still failing, skip to the Troubleshooting matrix at the end of Activity 2.

✅ **Checkpoint:** Both the sanity-test curl/`ollama run` AND `nanobot agent -m` return a response.

Notes / observations:
___________________________________________________
___________________________________________________

**Path B Troubleshooting — Common Local-Ollama Issues**

If you're on Path B and something didn't work, find your symptom here. (Cloud-path issues are in the Activity 3 troubleshooting table.)

| Symptom | Likely cause | Fix |
|---|---|---|
| `connection refused` from `ollama run` or nanobot | Ollama service isn't running | `systemctl status ollama` → if not active, `sudo systemctl start ollama` |
| `model 'X' not found` | The model in `config.json` wasn't pulled | Either `ollama pull <model from config>`, or edit config to match what you did pull |
| `ollama ps` shows `CONTEXT 4096` after restart | The `OLLAMA_CONTEXT_LENGTH` override didn't apply | `systemctl show ollama \| grep Environment` should list your 3 vars. If empty, re-do Step B.1 — the override file must contain `[Service]` as its first line |
| `ollama ps` returns nothing | Model not loaded into RAM yet | Run `ollama run <model> "test"` once; that loads it. `OLLAMA_KEEP_ALIVE=24h` keeps it warm thereafter |
| Replies appear but are garbled / nonsense | Context truncation (system prompt was cut to fit) | Confirm `CONTEXT 16384` in `ollama ps`. If correct, your model size may be too small — try the next size up if your RAM allows |
| Replies take >5 min, but eventually arrive | Normal for Pi-class hardware with `qwen2.5:7b` | Stick with the smaller model (`qwen2.5:3b`) or accept the wait — local inference on a Pi is genuinely slow |
| `nanobot agent -m` times out at exactly 2:00 | `NANOBOT_OPENAI_COMPAT_TIMEOUT_S` not in the shell that ran nanobot | `echo $NANOBOT_OPENAI_COMPAT_TIMEOUT_S` should print `900`. If empty: `source ~/.bashrc` then retry. (You'll need this set in *every* terminal you use for nanobot — adding it to `~/.bashrc` makes new terminals inherit it) |
| Out-of-memory (OOM) — Pi reboots or freezes mid-reply | Model too big for your RAM | Use a smaller model from the Step 1.2 table. Pi 4 8 GB → `qwen2.5:3b`; Pi 4 4 GB → `qwen2.5:1.5b` |
| Ollama logs say `truncating input prompt limit=4096` | Same as the garbled-replies row | Re-do Step B.1, then restart Ollama AND nanobot |

> 💡 **Reading Ollama's logs is genuinely useful** when things go wrong: `journalctl -u ollama -f` shows live logs. Truncation warnings, GPU detection, model loading times — all there.

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
import json, os
c = json.load(open(os.path.expanduser('~/.nanobot/config.json')))['channels']['telegram']
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
| `LLM transient error: request timed out` after ~2 min | Default 120s HTTP timeout (issue #3455) on a slow local LLM | If on Path B: confirm `NANOBOT_*_TIMEOUT_S` env vars from Step B.3 are set (`echo $NANOBOT_OPENAI_COMPAT_TIMEOUT_S`); if blank, `source ~/.bashrc`. If on Path A: try `openrouter/free` auto-router |
| `LLM truncating input prompt` in `journalctl -u ollama` | `OLLAMA_CONTEXT_LENGTH` not applied | Re-do Step 2.3 carefully, then `sudo systemctl restart ollama` |
| 429 `rate-limited upstream` from OpenRouter | Free model upstream provider busy | Use `openrouter/free` (auto-router) instead of a pinned model |
| Reply contains weird `reasoning` text | Free reasoning model leaking chain-of-thought | Pin to a non-reasoning model like `meta-llama/llama-3.3-70b-instruct:free` |

---

### Activity 4: Configure Your Second Brain Schema (15 min)

This is the most important step. `AGENTS.md` is what turns a generic chatbot into a disciplined wiki maintainer.

**Step 4.1 — Pick your topic** (2 min)

Pick something narrow enough that "the wiki" makes sense, broad enough that sources exist. The default we'll use throughout this worksheet is **AI fluency** — your evolving understanding of AI tools, techniques, and the broader landscape. Other tested examples:

- *"Personal AI fluency journal"* (the running example below)
- *"Singapore foreign policy 2020–present"*
- *"My fitness training program"*
- *"Premier League tactics 2025–26"*
- *"My PhD thesis on protein folding"*
- *"The Mandalorian (TV show) lore"*
- *"Stoic philosophy"* (good starter — concrete entities, well-bounded sources)

Bad picks: too broad ("AI" by itself, "philosophy"), too vague ("my deepest thoughts"), no source material available.

> 💡 **The "all of it" trap.** If you're picking AI fluency, resist the urge to make it "everything about AI" — tools AND tech AND policy AND your domain. That breadth duplicates entities across categories and the wiki never compounds. Pick a slice (e.g. "AI tools and workflows I'm actually using") OR adopt the **learner-journal** structure shown below (where the bot maintains *your* evolving understanding rather than an encyclopedia).

My topic: _____________________________________________

**Step 4.2 — Write a tight AGENTS.md** (13 min)

> ⚠️ **The strength of the schema matters a lot.** Loose schemas like *"answer from the wiki when possible"* are routinely ignored by smaller models. Schemas with **MANDATORY**, **EXCLUSIVELY**, **NEVER**, plus a worked example showing right vs. wrong behaviour, get consistent compliance.

> 🤓 **Two schema flavors.** This worksheet ships the **learner-journal** schema (best for "my understanding of X is evolving"-type topics like AI fluency, a craft you're learning, your strategy at work). If your topic is more **encyclopedia**-flavored (a fandom, a historical era, a body of literature where entities and concepts have stable definitions), see the alternative schema in the Reference Card at the end. For the worksheet flow we use learner-journal throughout.

Use Python to write the file — saved to `/tmp` first to avoid heredoc traps (multiline pastes into bash with `<<EOF` are fragile; saving the script to a file then running it is bulletproof):

```
nano /tmp/write_agents.py
```

Paste the following (just the Python, no shell wrappers):

```python
from pathlib import Path

TOPIC = "AI Fluency"  # ← change this to your topic if different

content = f"""# Agent Schema — Personal {TOPIC} Journal

You are a **learning journal maintainer** for the user's evolving {TOPIC}, not a general chatbot.
You answer EXCLUSIVELY from the wiki at `memory/MEMORY.md`.
You do NOT answer from general knowledge, ever.

## MANDATORY first step on every message

Before responding to ANY user question, you MUST:
1. Call the `read_file` tool with path `memory/MEMORY.md`
2. Read its contents
3. ONLY answer based on what's in the file

If the wiki has no relevant entry, you say so. You do NOT fall back to training data.

## SCOPE — refuse anything outside {TOPIC}

If a question is NOT about {TOPIC} — refuse politely, point back at what the journal covers.

DO NOT attempt to use tools that don't exist in your toolset (no weather, no search, no calendar — these are not available).

EXAMPLE — off-topic refusal (FOLLOW THIS PATTERN):

User: "What's the weather like?"

You (correct):
"I'm scoped to your {TOPIC} journal — weather isn't something I track. Want to ask about something in the journal, or ingest a source?"

You (WRONG — never do this):
[tries to call a weather tool that doesn't exist]
[answers from training data]

## Wiki structure (inside MEMORY.md, use these top-level headers)

- `## Index` — one-line catalog of every entry across all sections
- `## Tools I'm Using` — tools/products with my actual usage notes. Per tool: what it does, my use cases, what works, what doesn't.
- `## Workflows I'm Building` — recipes, pipelines, repeated patterns I've put into practice. Each entry: goal, steps, tools involved, how it's evolving.
- `## Mental Models I'm Internalizing` — frameworks/concepts I want to retain. Each entry: 2-3 sentence definition, my plain-language take, why it matters to me.
- `## Things I'm Watching` — developments I'm tracking but don't yet have a strong view on. Lower-effort entries; one-liners with a date are fine.
- `## My Open Questions` — gaps in my own understanding. Things I want to dig into.
- `## Insights & Synthesis` — connections, surprises, patterns I've noticed across the above. First-person voice; my own thinking, not summaries of others.

## When the user pastes a source or says "ingest"

1. Read the source carefully.
2. Decide which section(s) it touches. A source rarely lives in just one — a blog post about a new tool might add an entry under `Tools I'm Using` AND a `Mental Model` for the underlying technique AND a `Watch` for the broader trend.
3. **Extract 1-3 verbatim quotes** from the source that best capture its key arguments. These MUST be exact quotes — preserve the source's wording exactly, in quotation marks. Do not paraphrase quoted material. These quotes make the entry auditable later: when re-reading the wiki months later, the user can see what the source actually said versus your summary.
4. Use the `edit_file` or `apply_patch` tool to update MEMORY.md:
   - For tools: capture *the user's perspective*, not a marketing summary. Ask "what would I want to remember about this in 6 months?"
   - For mental models: write the 2-3 sentence definition clearly enough that the user could re-read it cold and understand
   - For workflows: capture the actual sequence of steps, not just the gist
   - For every entry, include a sub-bullet `*Source*:` with title, author, and date
   - For every entry, include a sub-bullet `*Quotes*:` with the 1-3 verbatim quotes (each in quotation marks, separated by ` / ` if multiple)
   - Update `## Index` to include any new entries
5. **If the source content is not actually available to you** (e.g. you only see a filename or metadata, not the full text) — STOP. Reply: "I can see the source's metadata but not its full text. Can you paste the content directly, or describe what's in it?" Do NOT invent quotes or content from the filename alone.
6. Reply with a 3-bullet summary: which entries got created/updated, the verbatim quotes you captured, and any cross-references to existing entries you noticed.

## When the user asks a question (the normal case)

1. ALWAYS read MEMORY.md first using `read_file`.
2. If the wiki has the answer, respond with it. Cite the section/entry.
3. If the wiki does NOT have the answer, respond EXACTLY in this format:
   "The journal doesn't cover [topic] yet. Want to file something on it? You can ingest a source or just describe your current thinking."
4. Do NOT add general knowledge to fill the gap. Do NOT recite facts from training.

## Example interaction (FOLLOW THIS PATTERN)

User: "What is a transformer?"

You (correct):
[calls read_file on memory/MEMORY.md]
[finds nothing in Mental Models]
"The journal doesn't cover transformers yet. Want to file something on it? You can ingest a source or just describe your current thinking."

You (WRONG — never do this):
"A transformer is a neural network architecture introduced in 2017..."

## When the user shares a personal observation or insight

Sometimes the user will say things like "I noticed today that..." or "I think the real value of X is...". These belong in `## Insights & Synthesis`.
1. Capture them in the user's own voice — don't smooth them into encyclopedia prose.
2. Cross-link to related Tools, Workflows, or Mental Models if relevant.
3. Date the entry.

## When the user says "lint" or "health check"

Read MEMORY.md. Report:
- Entries that exist in the Index but have no detailed section
- Sections that have grown thin or stale
- Cross-references that could be made between entries
- Mental Models that haven't been touched in a long time

Suggest fixes. Do NOT apply them without explicit confirmation.

## Style

- Concise. Markdown. Bullets over prose for factual content; full sentences for Insights.
- Always cite sources by their title.
- Scoped to {TOPIC} ONLY. Politely refuse unrelated topics.
- It's OK to say "the journal doesn't cover this yet" or "I don't know" — those are the right answers when true.
"""

Path.home().joinpath(".nanobot/workspace/AGENTS.md").write_text(content)
print(f"✓ AGENTS.md written: {len(content)} chars")
```

Save (`Ctrl+O`, Enter, `Ctrl+X`), then run:

```
python3 /tmp/write_agents.py
```

You should see `✓ AGENTS.md written: NNNN chars` (around 5500).

> 🤓 **Why save-to-file instead of heredoc?** Pasting multi-line heredocs (`python3 <<'EOF' ... EOF`) into SSH terminals is fragile — the closing `EOF` marker can get eaten, indented, or stripped of its trailing newline. You'll see the shell stuck at a `>` continuation prompt forever. Save to a file with nano, run with python3 — never fails.

**Step 4.3 — Initialize MEMORY.md with the scaffold**

The AI works better with a properly-formed wiki on first read than with the boilerplate `nanobot onboard` placeholder. Same save-to-file pattern:

```
nano /tmp/scaffold_memory.py
```

Paste:

```python
from pathlib import Path

TOPIC = "AI Fluency"  # ← match your AGENTS.md

content = f"""# Personal {TOPIC} Journal

*A learning journal maintained between me and the things I read, watch, and try.
Not a Wikipedia replacement — my own evolving understanding, in my own voice.*

## Index
*(no entries yet — ingest sources or share observations to populate this)*

## Tools I'm Using
*(no entries yet)*

## Workflows I'm Building
*(no entries yet)*

## Mental Models I'm Internalizing
*(no entries yet)*

## Things I'm Watching
*(no entries yet)*

## My Open Questions
*(no entries yet)*

## Insights & Synthesis
*(no entries yet — these grow over time as patterns emerge across the other sections)*
"""

Path.home().joinpath(".nanobot/workspace/memory/MEMORY.md").write_text(content)
print(f"✓ MEMORY.md scaffolded: {len(content)} chars")
```

Save and run:

```
python3 /tmp/scaffold_memory.py
```

You should see `✓ MEMORY.md scaffolded: NNNN chars`.

**Step 4.4 — Restart the gateway**

In the gateway terminal: `Ctrl+C`. Then:
```
nanobot gateway
```

**Step 4.5 — Test the new personality**

Ask via Telegram a question on your topic that the empty wiki couldn't possibly answer. For AI fluency:
> *What is a transformer?*

You're looking for two signs of correctness in the gateway logs:
1. A `Tool call: read_file({"path": "memory/MEMORY.md"})` line
2. A reply matching: `The journal doesn't cover [topic] yet. Want to file something on it? You can ingest a source or just describe your current thinking.`

If instead the bot rattles off facts from training data — the schema isn't being followed. Likely culprits: the model is a small/loose one (try pinning to a stricter free model below, or upgrading to paid), or the schema's MANDATORY/EXCLUSIVELY language was softened during your edit.

Then test scoping with an off-topic message:
> *What's the weather in Singapore?*

Expected: polite refusal, redirecting back to your topic. Heads-up on what "polite refusal" actually looks like in practice — exact phrasing varies by which model the auto-router picks. Acceptable signs of correctness: no phantom tool calls (no `Tool call: weather(...)` in logs), no factual answers from training data. Looser phrasing of "I can't help with that, try a weather app" is OK; what matters is the *behaviour*, not the words.

> 🤓 **If refusals are loose with `openrouter/free`** — that's expected. Auto-router picks a different model each request, and free-tier models vary in instruction-following. Strict refusals are possible but require either: (a) pinning a known-strict model like `meta-llama/llama-3.3-70b-instruct:free` (which may 429 when popular), or (b) topping up OpenRouter $10 and pinning a paid model like `anthropic/claude-haiku-4-5` (~$0.001/msg). For workshop purposes the looser behaviour is acceptable; the journal-maintenance behaviour is what matters and that's surprisingly robust across models.

✅ **Checkpoint:** Bot reads MEMORY.md before answering, admits empty wiki using something close to the schema's phrasing, refuses off-topic queries (any phrasing) without phantom tool calls.

---

### Activity 5: Ingest & Query via Telegram (10 min)

**Step 5.1 — Ingest your first source (paste-text version)**

From Telegram, send a single message in this format. The trigger phrase tells the bot to enter ingest mode rather than treating your text as a chat message:

```
Ingest this as a source:

[paste a paragraph or article relevant to your topic — your own notes, a blog
post, a paper abstract + intro, whatever]

Source title: [a short, memorable title]
Date: [today's date]
URL (optional): [link]
```

For an AI fluency journal, good first sources to try:
- Karpathy's "LLM Wiki" gist (the conceptual basis for this whole workshop)
- A Simon Willison blog post about a tool you actually use
- An MCP server's README that you've skimmed
- Your own workflow notes about, say, how you use Claude Code

Pick something **you've actually read** — you'll be able to verify the bot's summary is accurate. Length 500-2000 words is the sweet spot for first ingest; longer sources risk tool-iteration limits.

In the gateway terminal, watch the sequence:
- `Tool call: read_file({"path": "memory/MEMORY.md"})`
- `Tool call: apply_patch({...})` or `Tool call: edit_file({...})` — possibly multiple
- `Response to telegram: ...` — a 3-bullet summary of what changed

> ⚠️ **Watch the iteration count.** Complex sources may trigger many edits. If you hit *"reached maximum number of tool call iterations"*, bump `maxToolIterations` in config — 10 is reasonable, 15 generous, 20+ if you regularly ingest large multi-section sources.

> 🤓 **You may see new log lines on the source build:** `Routed follow-up message to pending queue` (nanobot buffering messages received during a tool call), `Empty response on turn N, retrying` (model paused mid-decision, nanobot retried), `Token consolidation round` (context window approaching limit, nanobot auto-summarized older messages). All of these are healthy behaviours, not errors.

**Step 5.2 — Verify the wiki was updated**

In a second terminal:
```
cat ~/.nanobot/workspace/memory/MEMORY.md
```

You should see real entries under one or more sections (depending on what your source was about — likely `## Mental Models I'm Internalizing` for a conceptual source, `## Tools I'm Using` for a tool-focused one), and matching index lines. The `*(no entries yet)*` placeholders should be gone from any section the source touched.

**Step 5.3 — Spot-check the entry against the actual source** (genuinely important)

This step exists because the LLM is prone to a specific failure mode: **plausible structure with invented content**. The bot is excellent at producing wiki-shaped output. It's less reliable about whether that output actually reflects what the source said — especially for PDFs or sources where text extraction may have failed silently (see Step 5.6).

Compare:
- Does the entry's definition match the source's actual thesis?
- Are the `*Quotes*:` sub-bullets actually present in the source? Do a text-search of the source for one of them.
- Is the cited author/date accurate?

If quotes don't match the source, you have a hallucinated entry. Treat as a real bug: delete the entry, investigate why text extraction failed, ingest again with paste-text instead of attachment.

**Step 5.4 — Query the wiki**

Ask via Telegram about something that's now in the wiki. For example, if you ingested Karpathy's gist:
> *What's the LLM Wiki pattern?*

This time the bot should:
- `read_file` on MEMORY.md
- Find the entry
- Answer from it, citing the section ("This is in your Mental Models section")
- Include the structured details — definition, key ideas, why it matters

**The answer should reference YOUR ingested source, not generic knowledge.** If the bot suddenly answers from training data after the wiki has content, the schema isn't holding under query — re-check that the entry was actually written.

**Step 5.5 — Test compounding (the real magic)**

Ask a question about something *related to* the source but that you didn't explicitly query for:

> *What is RAG?* (after ingesting Karpathy's LLM Wiki gist)

Two outcomes, both informative:

| What you see | What it means |
|---|---|
| `The journal doesn't cover RAG yet. Want to file something on it?` | The bot was minimalist on first ingest — RAG was mentioned but not extracted as its own entry. **Correct behaviour** — the schema only files entries on what the source is *about*. After several ingests of related sources, common concepts get cross-referenced. |
| Bot answers about RAG from the wiki | The bot did broader extraction during ingest. Check MEMORY.md to confirm there's actually a RAG entry. |

**Step 5.6 — Ingesting PDFs via Telegram drag-drop**

Nanobot's Telegram channel supports PDF attachments directly. Drag a PDF into your chat with the bot — it will treat the attachment as a source and ingest it.

This works **but with three important constraints** to understand before relying on it:

1. **Text extraction happens at the channel layer, once.** When the PDF arrives, nanobot's channel handler extracts text and passes it to the agent for that one ingest call. The raw PDF text is **not persisted** — only the wiki entry the bot creates from it survives.

2. **The agent can't re-read the PDF after ingest.** If you later ask the bot to "summarize that PDF I sent earlier" or "quote the section about X," it will tell you it can't read PDFs (truthfully — it lost access to the text after ingest completed). This means **the wiki entry IS the only persistence**. Make sure ingest entries are detailed enough to answer your future questions.

3. **If text extraction fails or returns junk, the bot may still produce a confident-looking entry from metadata alone** (filename, author from PDF metadata, etc.). This is the most concerning failure mode — entries that look right but were inferred from breadcrumbs. The schema's *"if the source content is not actually available to you... STOP and ask the user to paste it"* clause is designed to catch this, but it's not 100% reliable. **Always spot-check PDF ingests against the actual source.**

**Workflow recommendation:**
- For **short, important** sources (papers you care about, articles you'll want to query in depth): copy-paste the text into a Telegram message using the Step 5.1 format. Slower to set up, but you control exactly what the bot sees, and you can verify the ingest exactly captured the text.
- For **bulk, lower-stakes** sources (newsletters, blog posts you want to skim later): PDF drag-drop is fine. Spot-check entries occasionally.
- For **highly technical PDFs with tables, equations, or unusual formatting** where Telegram's text extraction may garble: pre-convert on the Pi with `pdftotext file.pdf -` (faster, lossy) or `marker_single file.pdf` (slower, much cleaner markdown output) and paste the result.

**Step 5.7 — Try a lint**

Send to the bot:
> *Run a lint pass on the journal. Report any issues, but do not apply fixes yet.*

The bot should read MEMORY.md and flag:
- Entries in the Index without matching detailed sections
- Thin or stub entries that could be expanded
- Cross-references that aren't being made
- Mental Models that haven't been touched recently

You can then approve specific fixes:
> *Apply the fix for [issue X].*

Watch the gateway logs — multiple `edit_file` or `apply_patch` calls, then a summary.

✅ **Checkpoint:** Your second brain has at least one source ingested (text and/or PDF), answers queries grounded in the wiki, you've spot-checked one entry against its actual source, and the lint+approve loop works.

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

In nanobot, sub-agents are defined as additional entries under the `agents` block in `config.json`. **Note:** the sub-agent format has been evolving across versions — consult `https://nanobot.wiki` for the current canonical form before relying on this skeleton. Concept (don't copy verbatim — adapt to current docs):

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
- [ ] **Schedule a daily lint** via nanobot cron. The cron config is at the top level of `config.json`; check current docs for syntax (it has evolved across releases).
- [ ] **Back up your wiki off-machine** — `~/.nanobot/workspace/` is already a git repo. Add a remote (GitHub, Gitea, anything) and push regularly. Free version history.

**The bigger idea (Karpathy):**
> *The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch many files in one pass. The wiki stays maintained because the cost of maintenance is near zero.*

**The bigger idea (Balakrishnan):**
> *The specific composition you used today will be obsolete in months. The builder's ability to compose the right pieces — that's the durable advantage.*

You now have that ability. **Keep building.**

---

## Part 5 — Lessons from a Real Run (as of late May 2026)

*These are point-in-time observations from running this workshop on a Raspberry Pi 5 16 GB, building a real AI fluency journal. Future readers — some of these may be stale; treat as breadcrumbs, not gospel.*

### What worked unexpectedly well

- **PDF ingestion via Telegram drag-drop.** Nanobot's channel layer extracts text from PDFs at ingest time without any extra setup. Constraints in Activity 5.6 — read those — but the basic workflow is "drop PDF in chat, get wiki entry." Worth knowing.
- **The learner-journal schema** (Tools / Workflows / Mental Models / Watch / Open Questions / Insights) handled the same source the encyclopedia schema would have, but produced entries in *the user's voice* rather than Wikipedia voice. For evolving-topic subjects (AI fluency, a craft, a strategy at work), this matters a lot.
- **The bot pattern-matching its own entry structure across ingests.** After the first source set a Definition / Key Ideas / Why It Matters layout, subsequent sources followed the same structure without being told. Consistency emerges from the bot, not just the schema.

### What didn't work (and why)

- **Pinning `meta-llama/llama-3.3-70b-instruct:free` directly.** Popular free models get throttled hard (429s on every call). May work on quieter days. The provider (Venice) returns a `retry_after_seconds` between 7 and 60+ seconds depending on load.
- **OpenRouter's `models` fallback array.** Setting `"models": [primary, fallback1, fallback2]` in your config doesn't work — nanobot's openai-compat client doesn't forward this OpenRouter-specific field. Curl honors it; nanobot doesn't. Verified by hammering Llama 3.3 four times with no fallback to Qwen or auto-router. Durable fix would require a proxy layer between nanobot and OpenRouter, or upstream changes to nanobot.
- **Strict refusal phrasing from `openrouter/free` auto-router.** Behaviour is correct (no phantom tools, no training-data dumps) but exact wording varies per model the router picks. If you need consistent phrasing, pay OpenRouter $10 and pin Claude Haiku.

### What surprised us

- **The newer source build added three tools we didn't expect:** `apply_patch` (unified-diff edits), `run_cli_app` (agent can shell out), `find_files` (proper search vs. list_dir + grep). Removed `notebook_edit`. Run `nanobot --help` after a source upgrade — new subcommands may have appeared (`serve`, `status` are recent additions).
- **`nanobot status`** is genuinely useful — gives a one-shot view of your config, which providers have keys/OAuth configured, your current workspace location.
- **Token consolidation rounds** kick in automatically when context fills (~16k tokens for default config). Older messages get summarized. You'll see `Token consolidation round 0` in logs as a regular occurrence after a long ingest session — it's not an error.
- **The "I lack pymupdf" failure mode.** After a successful PDF ingest, if you ask the bot to "summarize that PDF you read," it'll say it can't read PDFs. That's because the channel layer extracted text once and discarded it — the agent itself doesn't have PDF tools. The wiki entry IS the persistence. Make ingest entries detailed enough to satisfy future questions.
- **Plausible structure ≠ accurate content.** The bot can write entries that *look* right from filename + metadata alone if the text extraction fails silently. Activity 5.3 (spot-checking against the source) is the only defense. The verbatim-quotes requirement in the schema helps a lot — entries with `*Quotes*:` sub-bullets are auditable.

### What we'd do differently next time

- **Skip the model-pinning detour.** Start with `openrouter/free` auto-router. If refusals are too loose for your taste after using it for a week, then pay $10 to OpenRouter and pin a paid model. The middle ground (free pinned models with retry backoff) costs more time than it saves.
- **Start with the learner-journal schema** for topics that involve learning or evolving understanding. Reach for the encyclopedia schema (next Reference Card) only when entities and concepts have stable definitions worth cataloging.
- **Test PDF ingest with a known PDF early.** Spot-check one ingest carefully — verify quotes against the actual source. If text extraction is working on your setup, you can rely on it for bulk; if it's not, you'll know to pre-convert.

---

## Reference Card — Alternative Schema (Encyclopedia Flavor)

The Activity 4 worksheet ships the **learner-journal** schema, which works best when the topic is something you're learning or your understanding is evolving. For topics with stable entities and concepts — a fandom, a historical era, a body of literature — an **encyclopedia** schema may fit better. Same `read_file` MANDATORY clause, same scoped refusals, just different top-level sections:

```
## Index           — one-line catalog of every entity/concept page
## Entities        — people, organizations, named things relevant to {TOPIC}
## Concepts        — ideas, frameworks, recurring themes
## Sources         — what's been ingested (title + date + 2-3 sentence summary)
## Open Questions  — contradictions and things to investigate
## Synthesis       — overall picture, emergent observations
```

Per-entry structure stays the same (definition + verbatim quotes + source citation + why it matters). The differences are:

- **Voice**: encyclopedia entries tend to be written about the thing; learner-journal entries are written from the user's perspective on the thing.
- **What gets a page**: encyclopedia gives every named entity its own page (Marcus Aurelius, Epictetus, *Meditations*); learner-journal only files what you find personally significant.
- **Cross-references**: encyclopedia naturally creates a dense graph (Aurelius → *Meditations* → Stoicism → ...); learner-journal stays sparser, with links forming around your insights.

Both work. Pick one and commit — mixing them creates confusion. The same `AGENTS.md` Python template in Activity 4.2 can be edited to use the encyclopedia sections by swapping the `## Wiki structure` block and removing the `## When the user shares a personal observation or insight` section.

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

> ⚠️ **Before you start: stop any foreground gateway.** If you have `nanobot gateway` running in a terminal, hit `Ctrl+C` first. Then verify nothing's running:
> ```
> pgrep -af nanobot
> ```
> Should return **no output**. If a process is still listed, the systemd service will fail to start with a port conflict on `:18790` (the health endpoint) and you'll spend twenty minutes confused.

First, find where your `nanobot` binary lives — varies by how you installed it:

```
which nanobot
```

Common results: `~/.local/bin/nanobot` (pipx or `pip install --user`), `/usr/local/bin/nanobot` (pip with `--break-system-packages`), or a Python venv path. **Use the path from `which` in the `ExecStart=` line below.** The snippet below auto-substitutes it via `$(which nanobot)`.

```
NANOBOT_BIN="$(which nanobot)"
sudo tee /etc/systemd/system/nanobot-gateway.service > /dev/null <<EOF
[Unit]
Description=Nanobot Gateway
After=network.target ollama.service
Wants=ollama.service

[Service]
Type=simple
User=$USER
WorkingDirectory=$HOME
Environment=HOME=$HOME
ExecStart=$NANOBOT_BIN gateway
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

### Day-to-day commands

Once the service is set up, these are the ones you'll actually use:

| Task | Command |
|---|---|
| Check current status | `sudo systemctl status nanobot-gateway` |
| Tail live logs (Ctrl+C to exit) | `sudo journalctl -u nanobot-gateway -f` |
| Last 50 log lines (no follow) | `sudo journalctl -u nanobot-gateway -n 50` |
| Restart (after config or upgrade) | `sudo systemctl restart nanobot-gateway` |
| Stop temporarily | `sudo systemctl stop nanobot-gateway` |
| Start again | `sudo systemctl start nanobot-gateway` |
| Disable auto-start on boot | `sudo systemctl disable nanobot-gateway` |
| Re-enable auto-start on boot | `sudo systemctl enable nanobot-gateway` |

> 💡 **After `pipx upgrade nanobot-ai` or any source reinstall** — the running service still holds the *old* code in memory until you restart it. Always `sudo systemctl restart nanobot-gateway` after an upgrade. (See also the next Reference Card on keeping nanobot up to date.)

> 🤓 **If the service won't start:** check `sudo journalctl -u nanobot-gateway -n 50` for the actual error. Most common causes: (1) the `nanobot` binary path in `ExecStart` is wrong (re-run `which nanobot` and edit `/etc/systemd/system/nanobot-gateway.service`), (2) a foreground nanobot is still running on port `:18790` (kill it with `pgrep -af nanobot` then `kill <pid>`), or (3) `User=` references a user that doesn't exist.

---

## Reference Card — Laptop users: shutdown, sleep, and "when you're done for the day"

The systemd auto-start card above assumes you're on an always-on Linux machine (Pi, server, plugged-in workstation). **Laptops are different physics**: they sleep, they shut down, they get carried places, their network drops. The bot can't be online when the laptop is asleep — and that's fine. This card explains what actually happens and what you should do.

### The honest reality

Your nanobot is **only online when** (a) the machine is awake AND (b) `nanobot gateway` is running on it. Close your laptop → bot offline. Shut down → bot offline. Boot the laptop fresh → bot offline until you start it again. Telegram queues incoming messages briefly (~24h) and delivers them when your bot comes back online, so you won't *lose* messages — just delay them.

If you need a truly always-on bot, a laptop isn't the right substrate. See "If you want always-on" at the bottom of this card.

### Clean shutdown procedure (recommended default)

When you're done for the day:

1. Go to the terminal running `nanobot gateway`
2. Press `Ctrl+C` — the gateway shuts down gracefully (you'll see goodbye messages)
3. Confirm cleanup: `pgrep -af nanobot` should show no output
4. Close the terminal. Shut down or sleep your laptop normally.

That's it. Your wiki at `~/.nanobot/workspace/` is on disk; nothing is lost. The git history is intact.

### Resume procedure (next time you want the bot)

```
nanobot gateway
```

Wait for `bot @<your_bot_username> connected`. Send a `hi` from Telegram to confirm. You're back online.

If you set up the env vars from Path B Step B.3 (local Ollama), they're already in your `~/.bashrc` and will load automatically when you open a fresh terminal.

### What happens when you close the lid (sleep/suspend)

- The OS suspends your nanobot process along with everything else
- Telegram's polling stops (no network)
- On wake: network reconnects, polling resumes within seconds, queued Telegram messages arrive in a burst
- **Usually recovers cleanly** without you doing anything. If it doesn't (very rare — sometimes a stuck HTTP connection from before sleep), restart the gateway: `Ctrl+C` in the terminal, then `nanobot gateway` again.

### Auto-start on Mac (if you want the bot back automatically after every boot)

macOS uses **launchd**, not systemd. The launchd plist equivalent of our systemd service:

Save as `~/Library/LaunchAgents/com.nanobot.gateway.plist` (create the dir if it doesn't exist):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.nanobot.gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.local/bin/nanobot</string>
        <string>gateway</string>
    </array>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
    <key>StandardOutPath</key>
    <string>/tmp/nanobot-gateway.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/nanobot-gateway.log</string>
</dict>
</plist>
```

Replace `YOUR_USERNAME` with your Mac username (check via `whoami`). If your `nanobot` binary lives elsewhere (`which nanobot`), update the path. Then load it:

```
launchctl load ~/Library/LaunchAgents/com.nanobot.gateway.plist
```

To unload (disable auto-start): `launchctl unload ~/Library/LaunchAgents/com.nanobot.gateway.plist`. To check status: `launchctl list | grep nanobot`.

> ⚠️ **Mac caveat**: launchd will *not* keep the bot running when the lid is closed unless you have "Prevent automatic sleeping on power adapter when the display is off" enabled (System Settings → Battery → Options). Even then, the lid-closed-on-battery behaviour is unreliable. Mac auto-start is for "bot starts when I boot," not "bot stays running while I'm carrying the laptop in my bag."

### Auto-start on Windows (WSL or native)

**If you installed via WSL** (recommended for Windows): WSL itself stops when you close the terminal or shut down Windows. Auto-starting nanobot inside WSL across reboots requires either (a) keeping a WSL terminal pinned and the bot running in it via `tmux`, or (b) registering nanobot as a Windows scheduled task that invokes WSL.

**Task Scheduler approach** (more reliable):
1. Open Task Scheduler (Windows search → "Task Scheduler")
2. Create Task → name it "Nanobot Gateway"
3. Trigger: "At log on"
4. Action: Start a program → `wsl.exe` with arguments `-d Ubuntu -- /home/YOUR_WSL_USER/.local/bin/nanobot gateway`
5. Check "Run whether user is logged on or not" if you want it running when locked

> ⚠️ **Windows caveat**: similar to Mac, sleep and hibernation interrupt WSL processes. Reliable always-on operation on Windows really wants a server, not a laptop.

### If you want always-on

If your bot needs to be reachable when you're not at your laptop (e.g., you message it from your phone during the day), laptop deployment is the wrong choice. Three good alternatives, in order of cost:

| Option | Cost | Setup time |
|---|---|---|
| Raspberry Pi 4/5 at home | ~$80 one-time | ~30 min |
| Cheap cloud VM (Hetzner CX11, Oracle Free Tier, DigitalOcean Droplet) | $0-5/month | ~20 min |
| Existing always-on home server (NAS, mini-PC) | $0 (you already have it) | ~10 min |

All three run the same nanobot you ran on your laptop — same config file, same workspace, same Telegram bot. To migrate: `scp -r ~/.nanobot/ user@new-host:~/` then run `nanobot gateway` on the new host. Stop the laptop one; the same Telegram bot now talks to the new host.

> 💡 **Tradeoff to know**: cloud VMs are the cheapest always-on option but your wiki lives on someone else's hardware. If privacy was your reason for choosing local Path B, a home Pi or self-hosted server is better.

---

## Reference Card — Keeping nanobot up to date

Nanobot is pre-1.0 and ships fast. New tools, providers, and bug fixes land on GitHub regularly — often weeks before they make it to a tagged PyPI release. This card covers both upgrade paths.

### Before any upgrade

1. **Stop the gateway.** `Ctrl+C` if running interactively; `sudo systemctl stop nanobot-gateway` if running as a service.
2. **Back up your config.** Cheap insurance:
   ```
   cp ~/.nanobot/config.json ~/.nanobot/config.json.pre-upgrade.$(date +%Y%m%d-%H%M)
   ```
3. **Record the current version** for later comparison: `nanobot --version`

> 🤓 **Your wiki is always safe.** Config and workspace live in `~/.nanobot/`, separate from the package install. Upgrading nanobot can never lose your wiki — the `~/.nanobot/workspace/.git/` is your additional safety net.

### Upgrade Path 1 — Latest PyPI release (recommended default)

```
pipx upgrade nanobot-ai
```

If pipx insists you're current but you want it to rebuild anyway:
```
pipx upgrade nanobot-ai --force
```

After upgrade, restart the gateway and watch logs for any new env-var warnings, schema migrations, or tool-count changes:
```
nanobot gateway   # or: sudo systemctl start nanobot-gateway
```

### Upgrade Path 2 — Install from GitHub `main` (bleeding edge)

Use this when PyPI lags behind a fix you need (common, since publishes are batched). It clones nanobot's main branch and builds from source inside the same pipx-managed venv:

```
pipx install nanobot-ai --force --pip-args="git+https://github.com/HKUDS/nanobot.git@main"
```

Expect 30-90 seconds on a Pi (clone + build wheels + install deps). Pin to a specific commit for reproducibility:

```
pipx install nanobot-ai --force --pip-args="git+https://github.com/HKUDS/nanobot.git@<commit-sha>"
```

> ⚠️ **`main` is a moving target.** It may be mid-refactor or have failing CI on any given day. If you need stability, prefer a tagged PyPI release (Path 1).

### Verify what you actually installed

`nanobot --version` shows the version string the source code declares — which lags real changes when you're on the source build. The truth lives in pip's install metadata:

```
cat ~/.local/share/pipx/venvs/nanobot-ai/lib/python*/site-packages/nanobot_ai-*.dist-info/direct_url.json
```

Look for the `commit_id` field — that's the exact commit you're running. The `nanobot status` command also gives a current snapshot of your config, configured providers, and workspace health.

### Reading what changed

```
# Latest release notes from PyPI side
curl -s https://api.github.com/repos/HKUDS/nanobot/releases/latest \
  | python3 -c "import json, sys; r = json.load(sys.stdin); print(r['name']); print('---'); print(r['body'])"

# Or browse the commit log directly
# https://github.com/HKUDS/nanobot/commits/main
```

Also compare the "Registered N tools" line in your gateway startup logs before vs after — new tools are usually the most visible sign of new capability.

### Rollback

If anything breaks:
```
pipx install nanobot-ai==<previous-version> --force
```

Restore your config too if needed:
```
cp ~/.nanobot/config.json.pre-upgrade.<timestamp> ~/.nanobot/config.json
```

### After upgrade — restart the systemd service too

If you have the gateway running as a systemd service (from the Reference Card above), the old process holds the old code until you restart it:
```
sudo systemctl restart nanobot-gateway
sudo systemctl status nanobot-gateway   # confirm active (running)
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
| Tools description | `~/.nanobot/workspace/TOOLS.md` |
| Heartbeat prompt | `~/.nanobot/workspace/HEARTBEAT.md` |
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

## Appendix — Timeout env vars when running nanobot under systemd

> **You need this** if you've set up the gateway as a systemd service (from the Reference Card above) AND you're on Path B (local Ollama). The `~/.bashrc` exports from Step B.3 only affect interactive shells — a systemd-managed nanobot doesn't see them. This appendix gets the same env vars into the service.

### Background — the two timeouts

Nanobot has two LLM timeouts:

| Variable | Default | What it bounds |
|---|---|---|
| `NANOBOT_OPENAI_COMPAT_TIMEOUT_S` | 120 | Inner HTTP read/write timeout (httpx, talking to Ollama or OpenRouter) |
| `NANOBOT_LLM_TIMEOUT_S` | 300 | Outer `asyncio.wait_for` guard around one LLM iteration |

Path B Step B.3 raised both to 900 in your shell. We need to mirror that into the systemd service so the always-on gateway has the same budget.

### Add timeouts to the nanobot-gateway service

```
sudo systemctl edit nanobot-gateway
```

This opens an empty override file. Add:

```
[Service]
Environment=NANOBOT_OPENAI_COMPAT_TIMEOUT_S=900
Environment=NANOBOT_LLM_TIMEOUT_S=900
```

Save (`Ctrl+O`, Enter, `Ctrl+X`). Reload:

```
sudo systemctl daemon-reload
sudo systemctl restart nanobot-gateway
```

Verify the new env reached the running process:

```
sudo systemctl show nanobot-gateway | grep -i timeout
```

You should see both variables listed. If you don't, the override didn't save — re-run `sudo systemctl edit nanobot-gateway` and confirm the file has the `[Service]` header at the top.

### What raising the timeout still can't fix

- **CPU prefill is slow.** A 6500-token system prompt on a Pi 5 CPU takes ~80 seconds *before* generating the first reply token. Bigger timeouts let that complete; they don't speed it up.
- **OpenAI SDK client-level retries.** The SDK retries failed requests by default (issue [#2511](https://github.com/HKUDS/nanobot/issues/2511)). On really slow links you can stack 3× SDK retries × your full timeout, which is why some students see apparent 6-minute hangs. `NANOBOT_LLM_TIMEOUT_S` is the catch-all outer guard for this.
- **Telegram's own message delivery timing.** If a reply takes >60s, Telegram may show a stale "typing..." indicator. Nanobot's streaming usually masks this, but not always.

### When to give up and go cloud

If you find yourself raising timeouts above ~900s just to get basic replies, the cloud path is genuinely better. The wiki, memory, and rules still live entirely on your machine; only the LLM prompts (which you typed anyway) travel to OpenRouter. For most workshop topics that's not a meaningful privacy regression.

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
