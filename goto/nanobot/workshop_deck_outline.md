# Workshop Deck Outline (Self-Paced)
## Build Your Own Second Brain with Nanobot + Telegram

> **Audience:** A student going through this alone with the worksheet open beside them.
> **Format:** ~25 slides. Each slide has a title, bullet content, and a "notes" block written *for the student* — what to think, what to do, when to pause.
> **Companion:** Pairs with `workshop_worksheet.md` — slides give the *narrative and intuition*, the worksheet has the *exact commands*. Don't copy commands onto slides.
> **Visual recommendation:** Plenty of whitespace. Big type. Use diagrams over bullet walls where possible. The Karpathy wiki diagram and the architecture diagram are the most important visuals.

---

## Section 1 — Hook (slides 1–4)

Goal: convince the student why this matters before they spend an hour installing things.

---

### Slide 1 — Title

**Title:** Build Your Own Second Brain
**Subtitle:** Nanobot + Telegram, on hardware you already own

**Visual:** A photo of a Raspberry Pi or laptop with a phone next to it showing a Telegram chat.

**Notes:** Welcome. This deck is a guide, not a substitute for the worksheet — keep `workshop_worksheet.md` open in another tab. Slides explain *why* and *what to expect*; the worksheet has the *exact commands*. Plan for ~90 minutes end to end.

---

### Slide 2 — The Problem with Vanilla LLMs

**Bullets:**
- ChatGPT / Claude conversations have **no memory between sessions**
- Every chat starts from zero
- RAG (retrieval-augmented generation) helps but **re-derives** knowledge every query — nothing accumulates
- You've been doing the same Googling, the same re-explaining, for years

**Notes:** This slide names the pain. Take 30 seconds and think of three times this week you wished an AI remembered something you'd told it before. That frustration is what we're fixing.

---

### Slide 3 — The LLM Wiki Pattern (Karpathy)

**Visual (most important diagram in the deck):**
```
                                              
   YOU  ──pastes──►  RAW SOURCES               
                          │                    
                          ▼                    
                    ┌──────────┐               
                    │   LLM    │  reads + writes
                    └──────────┘               
                          │                    
                          ▼                    
                   ╔══════════════╗            
                   ║   THE WIKI   ║ ◄── compounds
                   ║ (markdown)   ║     over time
                   ╚══════════════╝            
                          ▲                    
   YOU  ──asks──►   reads from                 
```

**Bullets:**
- The LLM **incrementally builds and maintains** a persistent wiki between you and the raw sources
- The wiki is just markdown files on your machine
- Each interaction **adds to** or **refines** the wiki — it doesn't reset

**Notes:** Andrej Karpathy described this pattern in a 2025 gist. The key insight: humans abandon wikis because the bookkeeping burden outgrows the value. LLMs don't get bored. They'll touch fifteen files in one pass to update a cross-reference, and they don't mind. Maintenance cost → near zero. So the wiki actually stays maintained.

---

### Slide 4 — Real-World Reference

**Quote (big, centered):**
> *"The diplomat who learns to work with AI will have a meaningful edge. I think that edge is now."*
> — Vivian Balakrishnan, Singapore Foreign Minister, April 2026

**Subtitle bullets:**
- Published his personal AI second brain setup: **Claude (cloud) + Raspberry Pi + WhatsApp + the LLM Wiki pattern**
- The Pi is the always-on orchestrator. The cloud does the thinking. The wiki lives at home.
- Today you build a near-identical system on whatever hardware you have

**Notes:** This isn't a toy. Real public figures use this exact architecture. The composition you'll learn today is the durable skill — the specific tools (nanobot, OpenRouter, Telegram) will be obsolete in 18 months; the pattern won't be.

---

## Section 2 — What We're Building (slides 5–7)

---

### Slide 5 — Architecture

**Visual (the second-most-important diagram):**
```
   PHONE                YOUR MACHINE                  CLOUD
   ┌────┐              ┌──────────────┐              ┌─────┐
   │ TG ├──messages───►│              │──prompt─────►│ LLM │
   └────┘              │   NANOBOT    │              │     │
                       │   GATEWAY    │◄─reply───────┤     │
                       │              │              └─────┘
                       │  reads/writes│
                       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │  MEMORY.md   │  ← YOUR brain
                       │  AGENTS.md   │     YOUR rules
                       └──────────────┘
```

**Notes:** Three boxes. Your phone (Telegram), your machine (nanobot gateway, the wiki, the rules), and the LLM — which can be either cloud (OpenRouter) or local (Ollama running on your same machine). Both are supported as equal paths in this workshop. **The wiki and rules NEVER leave your machine** regardless of which LLM path you choose. With cloud, only the prompts themselves are sent out — same as ChatGPT. With local, nothing leaves your machine at all.

---

### Slide 6 — Hardware Reality Check

**Table:**

| Your machine | Local (Path B) | Cloud (Path A) |
|---|---|---|
| Laptop with NVIDIA GPU (≥6 GB) | ✅ ~5-15s/reply | ✅ ~5-15s/reply |
| Apple Silicon (M1/M2/M3) | ✅ ~10-30s/reply | ✅ ~5-15s/reply |
| CPU-only laptop | 😐 ~30-90s/reply | ✅ ~5-15s/reply |
| Raspberry Pi 5 (16 GB) | 😐 ~1-4 min/reply | ✅ ~5-15s/reply |
| Raspberry Pi 4/5 (8 GB) | 😐 ~30-90s/reply | ✅ ~5-15s/reply |
| Raspberry Pi 4 (4 GB) | ⚠️ Small model only, weaker quality | ✅ ~5-15s/reply |

**Notes:** Both paths reach the same finish line. Local is genuinely usable even on a Pi after the worksheet's Path B Step B.3 (which raises nanobot's default 120s LLM timeout — GitHub issue #3455). The trade-off is reply *speed*, not whether it works. Pick the path that matches your priorities: **cloud for speed and ease, local for privacy/offline/no API costs**. A Pi user who values privacy should go local without apology; a Pi user who just wants to feel the workflow fast should go cloud.

---

### Slide 7 — What "Done" Looks Like

**Visual:** Screenshot of a Telegram chat showing:
- User: *Ingest this as a source: [paragraph about an AI tool or concept]*
- Bot: *✓ Added entries under Mental Models (LLM Persistent Wiki Pattern) and Sources*
- User: *What's the LLM Wiki pattern?*
- Bot: *[Answer pulled from MEMORY.md, with the definition / key ideas / why it matters / source citation]*

**Notes:** This is the finish line. A bot on your phone that you feed sources, that organizes those sources into a structured journal on your machine, that answers questions citing what *you* taught it. The "AI" is in the cloud; the *intelligence about your topic* is on your machine. Take a moment to picture this working — that's what we're building toward. The default topic in the worksheet is AI fluency, but the pattern works for any narrowly-scoped subject you care about.

---

## Section 3 — Plan & Setup Choice (slides 8–9)

---

### Slide 8 — The Plan (5 Activities)

**Numbered list:**
1. **Install** — Ollama (optional) + Nanobot
2. **Configure** — Tell nanobot which LLM to use (cloud or local)
3. **Telegram gateway** — Connect to a bot you create with @BotFather
4. **Customize the schema** — Make `AGENTS.md` define a *wiki maintainer* on your topic
5. **Ingest + query** — Feed it sources, ask questions, see the wiki populate

**Notes:** Each activity has explicit checkpoints in the worksheet (✅ marks). When you hit a checkpoint, **stop and verify** before moving on. The checkpoints are designed so that if you've passed checkpoint N, you can recover from any disaster in checkpoint N+1 without redoing earlier work.

---

### Slide 9 — Choose Your LLM Path (Decision Point)

**Two columns:**

| **Path A — Cloud (OpenRouter)** | **Path B — Local (Ollama)** |
|---|---|
| Seconds per reply | Minutes per reply on a Pi (seconds with GPU) |
| Free tier available; ~$0.001/msg if paid | No API key, no usage cost ever |
| Prompts travel to OpenRouter | Nothing leaves your machine |
| Easier setup (Activity 2 is shorter) | More setup (3 substeps), one-time |
| **Pick this if:** you want speed and least friction | **Pick this if:** privacy matters, or no network, or you love local inference |

**Notes:** Stop and decide now — picking both creates confusion. You can switch later by editing two lines in `config.json`. Pi users: both paths work for you. If you don't know which you want, ask yourself: *do I care more about quick replies, or about my data staying on my device?* No wrong answer — the workshop is identical from Activity 3 onward. The worksheet's Activity 2 has both paths laid out side by side; just follow the column you picked.

---

## Section 4 — Activity Walkthroughs (slides 10–18)

> ⚠️ **Style note:** These slides don't list commands. The worksheet has commands. Slides explain what each activity is *for* and what to *watch out for*.

---

### Slide 10 — Activity 1: Install (Section divider)

**Big title:** ACTIVITY 1 — INSTALL

**Subtitle:** Ollama (optional) + Nanobot + workspace
**Time:** ~10 min (mostly waiting for downloads)

**Notes:** Open the worksheet to Activity 1 now. Follow the commands there. **Don't skim** — the prerequisite check (Python version, RAM, disk) takes 30 seconds and saves an hour of debugging. Come back to the deck when you hit the ✅ Checkpoint at the end.

---

### Slide 11 — Activity 1: What You're Installing

**Two boxes:**
- **Ollama** = the AI engine. Loads models into RAM and answers prompts. Skip if going cloud-only.
- **Nanobot** = the "agent loop". Reads `AGENTS.md`, calls tools, writes to `MEMORY.md`, talks to channels (Telegram, etc.).

**Notes:** Ollama is one thing — an inference engine. Nanobot is doing much more: routing messages, maintaining memory, calling tools, managing the conversation. Even if you skip Ollama (cloud path), nanobot is still the brain of the operation. After this activity you'll have both installed but nothing connected yet — that's normal.

---

### Slide 12 — Activity 2: Configure (Section divider)

**Big title:** ACTIVITY 2 — CONFIGURE & FIRST CHAT
**Time:** ~10 min (Path A) or ~15 min (Path B — one-time setup)

**Two boxes side-by-side:**
- **If you picked Path A (cloud):** Get OpenRouter key → configure nanobot → sanity-test with curl
- **If you picked Path B (local):** Tune Ollama service (B.1) → point nanobot at it with a RAM-aware model picker (B.2) → raise LLM timeouts (B.3) → sanity-test with `ollama run`

**Notes:** Path B has three substeps, each one a one-time fix that solves a real Pi failure mode: (1) Ollama's default 4096-token context is smaller than nanobot's ~6500-token system prompt → garbled replies. (2) You need the *right model* for your RAM. (3) Nanobot's default 120s LLM timeout is too short for Pi-class generation. The worksheet's B.1/B.2/B.3 solve all three before you hit them — together they make local Ollama a reliable path, not a workaround. Don't skip any; each addresses a real problem.

---

### Slide 13 — Activity 2: The Sanity Test (Why It Matters)

**Bullet content:**
- Before involving nanobot, **test the LLM directly** with `curl` (cloud) or `ollama run` (local)
- If the sanity test fails, the bug is in the LLM setup, not nanobot
- If the sanity test passes but nanobot times out, the bug is in nanobot or its config
- This 30 seconds of testing saves ~hours of debugging the wrong layer

**Notes:** This is a general lesson, not just for this workshop: **always test each layer of the stack independently** before debugging composite systems. The worksheet shows you the exact curl command for cloud and the `ollama run` for local. Run it. Confirm it works. Then move on.

---

### Slide 14 — Activity 3: Telegram (Section divider)

**Big title:** ACTIVITY 3 — TELEGRAM GATEWAY
**Time:** ~15 min

**Three things to collect:**
1. A bot token from @BotFather
2. Your numeric user ID from @userinfobot
3. (Critical) Add your ID to `channels.telegram.allowFrom`

**Notes:** Telegram bots are public by default. Without `allowFrom`, anyone who finds your bot's username can chat with it — they'd be using *your* OpenRouter quota or *your* Pi's CPU to answer their questions. **This is the single most-important security setting in the workshop.** Don't skip it.

---

### Slide 15 — Activity 3: The Gateway Concept

**Bullet content:**
- The "gateway" is a **long-running process**: it polls Telegram for new messages and replies as they come
- Must stay running for the bot to be online
- If you close the terminal, the bot goes offline
- For permanent always-on operation: run as a systemd service (see worksheet Reference Card)

**Notes:** When you run `nanobot gateway`, that terminal is "consumed" by the gateway. Open a second terminal for everything else. Later, set it up as a systemd service so it survives reboots and you don't need a terminal open at all. The worksheet has the systemd snippet — it's 6 lines of config.

---

### Slide 16 — Activity 4: Schema (Section divider)

**Big title:** ACTIVITY 4 — CONFIGURE THE WIKI SCHEMA
**Time:** ~15 min

**Subtitle:** This is the workshop. Everything else is plumbing.

**Notes:** Stop. Read this slide twice. The previous activities were setup; this one is where your bot transforms from a generic chatbot into *your* wiki maintainer. The whole quality of your second brain comes from this one file: `AGENTS.md`. Don't rush through it.

---

### Slide 17 — Activity 4: The Schema is the Product

**Three principles:**
1. **Pick a narrow topic.** "Tech" fails. "Premier League tactics 2025–26" works.
2. **Write rules like a programmer** — but in English. Use MANDATORY, ALWAYS, NEVER.
3. **Include a worked example.** Show the right *and* wrong behaviour. Small models follow examples better than rules.

**Notes:** A loose schema like *"answer from the wiki when possible"* will be ignored by smaller free-tier models — they'll fall back to training data. The schema in the worksheet uses **MANDATORY**, **EXCLUSIVELY**, **NEVER**, plus a worked example showing both correct and incorrect behaviour. This makes even smaller models comply consistently. Imitate the pattern; just swap the topic.

---

### Slide 18 — Activity 5: Ingest + Query (Section divider)

**Big title:** ACTIVITY 5 — FEED THE BRAIN
**Time:** ~10 min

**Three things to try:**
1. **Ingest** — paste a source, watch the wiki populate
2. **Query** — ask about something in the source
3. **Compounding test** — ask about a *concept* from the source you didn't directly ingest

**Notes:** The third test is where compounding starts. After your first ingest, the bot has *one* mental model in the wiki. Ask about a *different* concept from the same source, and the bot honestly says "the journal doesn't cover that yet" — and that's correct minimalist behaviour. After 5-10 ingests of related sources, you'll start asking questions like "how do these two ideas relate?" and the bot will synthesize from multiple entries. PDF drag-drop also works (Telegram channel extracts text at ingest time) — see worksheet Activity 5.6 for the constraints.

---

## Section 5 — The Aha Moment (slides 19–20)

---

### Slide 19 — Watch the Wiki Self-Populate

**Visual:** Before/after of `MEMORY.md`:

**BEFORE ingest:**
```
## Mental Models I'm Internalizing
*(no entries yet)*

## Sources
*(no entries yet)*
```

**AFTER ingest of Karpathy's LLM Wiki gist:**
```
## Mental Models I'm Internalizing
- **LLM Persistent Wiki Pattern** – A mental model where an LLM
  incrementally builds and maintains a structured, interlinked
  markdown wiki from raw sources...
  - *Source*: LLM Wiki (Karpathy gist)
  - *Quotes*: "the bookkeeping cost is what kills wikis" / ...
  - *Key ideas*: persistent wiki, incremental updates, compounding
  - *Why it matters*: avoids repeated reconstruction
```

**Notes:** This is the workshop's moment of clarity. Until now, you've been configuring; now you're watching a curated knowledge base grow in real time, written by the AI according to *your* rules. The file is just markdown — open it in any editor and read every word the AI wrote. Nothing is hidden, nothing is magic. Note the `*Quotes*:` line — that's the schema's audit trail, verbatim phrases from the source so months later you can verify what the AI wrote vs. what the source actually said.

---

### Slide 20 — Lint: The Wiki Self-Heals

**Sequence:**
1. **You:** *lint the journal*
2. **Bot:** *Found 1 thin entry + 1 missing cross-reference. Recommend: expand the Vibe Coding entry, link it to LLM Wiki pattern.*
3. **You:** *Apply both fixes.*
4. **Bot:** *[Tool calls to edit MEMORY.md]*
5. **Wiki:** ✅ Cleaned up.

**Notes:** This loop is what makes the wiki sustainable long-term. The AI catches its own messes when you ask. You approve fixes — it doesn't apply them silently. Schedule a daily lint and your wiki stays clean forever, with zero work from you. This is the difference between "AI tool" and "AI employee."

---

## Section 6 — Multi-Agent + Going Further (slides 21–23)

---

### Slide 21 — From One Agent to Many

**Visual:**
```
         ┌─────────────────┐
         │ CURATOR (you)   │  ← talks to you
         └────┬───────┬────┘
              │       │
       ┌──────▼───┐ ┌─▼──────┐
       │RESEARCHER│ │ LINTER │
       └──────────┘ └────────┘
```

**Bullets:**
- Today: one agent does everything (ingest, query, lint)
- Tomorrow: split into specialists — a researcher (web search), a linter (qa), an ingester (large pdfs)
- All coordinated by a curator that talks to you

**Notes:** You don't need this on day one. But once your wiki gets bigger (~50+ sources), you'll notice the single agent gets slow because every operation re-reads the whole MEMORY.md. Sub-agents let you parallelize — and let different operations use different models (cheap fast model for ingest, smart slow model for synthesis). The worksheet has the config skeleton.

---

### Slide 22 — Going Further (Checklist)

**Checkboxes:**
- [ ] **Run gateway as a systemd service** (survives reboots)
- [ ] **Pin a known-good model** once you know what works for your topic
- [ ] **Add web search tool** (Brave, Serper, Tavily — free tiers exist)
- [ ] **Add MCP servers** for Gmail, Calendar, Notion — see modelcontextprotocol.io
- [ ] **Schedule a daily lint** via nanobot cron
- [ ] **Back up your wiki** — workspace is already a git repo, push to GitHub

**Notes:** These are 1-day projects each. Do them at your own pace. The first two are quick wins; the MCP integrations are where it gets genuinely powerful — your bot can now read your email, check your calendar, summarize PDFs in your Drive, all funneled into the same wiki.

---

### Slide 23 — The Bigger Idea

**Two quotes, stacked:**

> *"The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. LLMs don't get bored. The wiki stays maintained because the cost of maintenance is near zero."*
> — Karpathy

> *"The specific composition you used today will be obsolete in months. The builder's ability to compose the right pieces — that's the durable advantage."*
> — Balakrishnan

**Notes:** Both quotes land in the same place: today's tools are scaffolding. The skill of *composing* them — picking the right LLM, the right glue, the right schema — is what scales. You just demonstrated that skill end-to-end. Now go do it on a topic that actually matters to you.

---

## Section 7 — Resources + Wrap (slides 24–25)

---

### Slide 24 — Resources

**Links list:**
- **Worksheet** (commands & troubleshooting) — `workshop_worksheet.md`
- **Nanobot** — https://github.com/HKUDS/nanobot
- **Nanobot docs** — https://nanobot.wiki
- **Karpathy's gist on the LLM Wiki pattern** — https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- **OpenRouter** — https://openrouter.ai (free models at /collections/free-models)
- **Ollama models** — https://ollama.com/library
- **MCP servers (tool extensions)** — https://modelcontextprotocol.io
- **Nanobot 120s timeout (configurable)** — https://github.com/HKUDS/nanobot/issues/3455

**Notes:** Bookmark the nanobot wiki and the MCP server directory; those two are the most actively-updated and most useful long-term. The Karpathy gist is short — read it once after the workshop and the whole pattern will lock in.

---

### Slide 25 — Now Build Something

**Big text, centered:**
**"What's a topic you've always meant to organize?"**

**Small text below:**
- Recreate `AGENTS.md` for it
- Spend 30 minutes pasting in sources you already have
- Notice what becomes queryable
- Notice what the wiki *doesn't* yet cover — those are your next sources

**Notes:** The worksheet's running example is AI fluency. Your real second brain is whatever you genuinely care about and have sources for — hobby, work project, learning goal, family knowledge, your craft. Don't ask the bot to be a Wikipedia replacement — ask it to be a structured memory for *one specific thing you care about*. Start narrow. Let it grow.

---

## Appendix — Optional Diagnostic Slides (if you want to add them)

Skip during normal workflow; useful if you're stuck.

---

### A1 — "I sent a message and nothing happened"

- Check the gateway terminal — do you see *"Processing message from telegram:..."*? If no → message never reached the gateway. Likely cause: `allowFrom` doesn't include your user ID, or the bot username is wrong.
- If yes but no reply → LLM call is hanging. See A2.

---

### A2 — "LLM transient error: request timed out (~2 min)"

- This is nanobot's default 120-second HTTP timeout (GitHub issue #3455).
- **If on Path B (local):** the `NANOBOT_*_TIMEOUT_S` env vars from Step B.3 didn't reach the shell that ran nanobot. Run `echo $NANOBOT_OPENAI_COMPAT_TIMEOUT_S` — should print `900`. If blank, `source ~/.bashrc` and retry. If you've set up the gateway as a systemd service, see the worksheet Appendix (timeouts need a `systemctl edit` override).
- **If on Path A (cloud):** rare; usually the free model upstream is hung. Switch to `openrouter/free` (auto-router) to fall through to another provider.

---

### A3 — "Reply contains weird `<think>` or `reasoning:` text"

- The model is a "thinking" model and its chain-of-thought is leaking into the reply.
- **Solution:** pin to a non-thinking model. `meta-llama/llama-3.3-70b-instruct:free` is a safe bet.

---

### A4 — "Bot rattles off Wikipedia facts instead of saying 'wiki empty'"

- The schema isn't being followed by the current model.
- **Solution 1:** Tighten the schema — make sure MANDATORY/EXCLUSIVELY/NEVER are in `AGENTS.md`.
- **Solution 2:** Switch to a stricter model — Llama 3.3 follows instructions well, Claude Haiku follows them very strictly (paid, but cheap).

---

### A5 — "Maximum tool iterations reached"

- A single agent task needed more `edit_file` / `read_file` calls than your `maxToolIterations` budget.
- **Solution:** Bump `agents.defaults.maxToolIterations` in `config.json` from 10 to 15 or 20. Restart the gateway.
