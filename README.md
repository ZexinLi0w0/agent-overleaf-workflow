# Agent-Driven Overleaf Workflow

A reference guide for using terminal-based AI coding agents to author and revise LaTeX papers hosted on Overleaf — without ever leaving your editor.

---

## 1. Common Agent Tools

A quick side-by-side of four popular agent CLIs you can use as the "editor" for your Overleaf project.

| Tool | Vendor | Open Source | Pricing (input, $/M tokens)¹ | GUI |
|---|---|---|---|---|
| **[Claude Code CLI](https://docs.claude.com/claude-code)** | Anthropic | ❌ Closed (binary) | Sonnet 4.5: **$3** · Opus 4.x: **$15** · Haiku 4.5: **$1** | Terminal only (official VS Code extension) |
| **[Codex CLI](https://github.com/openai/codex)** | OpenAI | ✅ MIT | GPT-5-Codex: **~$1.25** · GPT-5.3-Codex: **~$1.75** | Terminal only (also web + IDE versions) |
| **[Antigravity CLI](https://ai.google.dev/gemini-api/docs/antigravity-agent)** | Google | ❌ Closed (preview) | Gemini 3 Pro: **~$2** · Gemini 3 Flash: **$0.50** | Has GUI (Antigravity IDE / agent workspace) |
| **[OpenClaw](https://github.com/openclaw/openclaw)** | Community (openclaw.ai) | ✅ MIT | BYOK — depends on chosen model² | Web UI + multi-channel (Telegram / Discord / Slack / WhatsApp / iMessage / …) |

¹ Pricing as of early 2026, list price for direct API usage. Subscription tiers (Claude Pro, ChatGPT Plus, Google AI Ultra) bundle quotas differently — see each vendor's pricing page for current rates.
² OpenClaw routes to any supported model provider (Anthropic, OpenAI, Google Vertex, Bedrock, OpenRouter, local Ollama, …). You pay the underlying model's rate.

---

## 2. Abstract Pipeline: Agent-Driven Overleaf Editing

The trick is: **Overleaf gives every project a private Git remote.** Once you've cloned that remote locally, any agent that can `commit` and `push` becomes an Overleaf editor. Here is the full loop.

```
┌──────────────┐      git clone      ┌──────────────┐    agent edits     ┌──────────────┐
│   Overleaf   │ ───────────────────▶│ Local clone  │ ──────────────────▶│  Agent CLI   │
│   project    │                     │   (folder)   │                    │  (Claude/    │
│              │◀────────────────────│              │◀───────────────────│   Codex/…)   │
└──────────────┘    git push          └──────────────┘    commit changes  └──────────────┘
       │                                                                          ▲
       │                                                                          │
       └───────────── PDF auto-rebuilds in Overleaf web UI ──────────────────────┘
```

### Step 1 — Get an Overleaf Git token

> ⚠️ **Requires an Overleaf paid plan** (Standard / Professional / institutional license). The free tier does **not** include Git integration.

1. In your Overleaf project, click **Menu → Integrations → Git** (sometimes labelled *Sync → Git*).
2. Click **Generate authentication token** under Account Settings → Git Integration. Copy it once — you cannot view it again.
3. Each Overleaf project has its own clone URL of the form:
   ```
   https://git.overleaf.com/<PROJECT_ID>
   ```
   where `<PROJECT_ID>` is the hex string in the project URL.

### Step 2 — Clone the project locally

Use the token as your HTTP Basic-Auth password (username can be anything, conventionally `git`):

```bash
git clone https://git@git.overleaf.com/<PROJECT_ID> my-paper
cd my-paper
```

Or stash the token via Git's credential helper so you don't paste it every push:

```bash
git config credential.helper osxkeychain        # macOS
# or: git config credential.helper store         # Linux (plaintext)
```

Optional: also push the local clone to a **private GitHub mirror** for branch-based collaboration or CI:

```bash
gh repo create my-paper --private --source=. --remote=github
git push github master
```

This is useful because most agent toolkits (Claude Code, Codex, OpenClaw) ship a **`github` skill / built-in tool** that streamlines PRs, issues, branch management — your Overleaf project effectively gains GitHub-flavoured workflow without disrupting the Overleaf rebuild loop.

### Step 3 — Hand the folder to an agent

Inside the cloned folder, launch your agent of choice. The agent reads `*.tex`, modifies content, then commits and pushes back to the Overleaf remote.

**Claude Code CLI:**
```bash
cd my-paper
claude --dangerously-skip-permissions -p "Fix all overfull hbox warnings in main.tex; rebuild and commit."
```

**Codex CLI:**
```bash
cd my-paper
codex exec "Rewrite §3.2 to emphasize the adaptive component; commit with a descriptive message."
```

**Antigravity CLI:**
```bash
cd my-paper
antigravity run "Add a related-work paragraph comparing to Smith 2024 (see ref.bib); commit and push."
```

**OpenClaw** (web UI / Telegram / Discord — agent operates on the cloned folder via its workspace):
> Paste the path `/path/to/my-paper` into chat and ask:
> *"Refactor §4.1 in main.tex to remove all `\textcolor{red}{…}` wrappers; verify it still compiles with `pdflatex`; commit and push to origin master."*

### Step 4 — Commit & push closes the loop

Every push to the Overleaf remote (`origin master` by default) triggers Overleaf's PDF rebuild on the web side. Refresh your browser tab to see the new PDF. If the agent broke a `\cite{}` or `\ref{}`, Overleaf surfaces it in the same compile-log pane you've always used.

```bash
git add -A
git commit -m "Refactor §4.1 per agent edits"
git push origin master
```

> 💡 **Big PDFs (>5 MB)?** Overleaf's Git endpoint can return HTTP 500 on the first push. Bump the buffer once:
> ```bash
> git -c http.postBuffer=524288000 push origin master
> ```

---

## 3. Tips & Gotchas

- **Conflicts.** If you also edit in the Overleaf web UI between agent sessions, you may get merge conflicts on the next pull. Always `git pull --rebase origin master` before starting an agent session.
- **Build verification.** Have the agent compile locally before pushing — install [TinyTeX](https://yihui.org/tinytex/) (~150 MB) so `pdflatex` is on `$PATH`. This catches errors the agent introduced before they show up in Overleaf's UI.
- **Sanitizing diffs.** For arXiv-style cleanup (strip `% comments`, remove `\textcolor{red}{…}` review marks, prune unused files), a 5-minute agent run beats hours of manual scrubbing.
- **Token cost reality.** A full paper edit (read main.tex + a couple of body sections + recompile + commit) is typically 10–50K input tokens per round. At Sonnet/Codex rates that's $0.03–0.15 per substantive revision pass.
- **Bring-your-own-key vs. subscription.** Subscription bundles (Claude Pro $20/mo, ChatGPT Plus $20/mo, Google AI Ultra $200/mo, Codex $20/$100/$200 tiers) cap usage by request quota rather than tokens — cheaper if you edit many small things, more expensive than direct API if you do occasional heavy refactors.

---

## License

This documentation is released into the public domain (CC0).
