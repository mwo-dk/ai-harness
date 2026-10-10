# ai-harness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the kit described in [design.md](./design.md) as ten small pull requests, each mergeable alone, each built with the kit's own rules.

**Architecture:** A Claude Code plugin (skills, two MCP servers, Python CLI under `uv`) plus a committed per-project layer (`harness.yaml`, `AGENTS.md`, `.agents/skills/`, `docs/harness/`). Workers are headless Claude Code processes on hosts; the coordinator is the interactive session. State for long runs lives in files in the repository.

**Tech Stack:** Python 3.12, `uv`, `pytest`, `pyyaml`, `httpx`, `mcp` (Python SDK), Claude Code plugin manifest, GitHub Actions, `gh`.

## Global Constraints

- No shell scripts in `bin/`; every tool is `uv run <tool>` (D20).
- Nothing in the repository or a project layer names a machine; hosts come from `~/.config/harness/hosts.yaml` (D18).
- Every tool works with zero hosts and degrades to a clear "unavailable" (D11, D18).
- One PR per task below; off `main`; never stacked; branch protection on `main` (D5, D6).
- Each PR carries its own tests; CI must pass on `ubuntu-latest` and `windows-latest` with `HARNESS_FAKE_HOSTS=1` (D20).
- Prose in committed files is plain English; `AGENTS.md` is at most 60 lines (D1, D13).
- Plugin and skill names: `ai-harness` marketplace, skills `harness`, `delegate-local`, `harness-init`, `harness-bench`, `harness-run`, `grill-with-docs`, `gauntlet-loop`; CLI commands `harness`, `local-run`.

## How this plan is executed

Each task below is one PR and is right-sized for a reviewer to accept or reject alone. Before starting a task, the coordinator runs `/ce-plan` on that task to produce the bite-sized red/green/refactor steps with code, saved as `docs/harness/plans/<nn>-<slug>.md`. From Task 3 onward, steps tagged `[local]` are handed to a host through the new `local-run`; the plan records the rung that landed each step (D4). This document tracks only PR-level status.

## File structure

```
.claude-plugin/plugin.json       plugin manifest (name, version, skills, mcp servers)
.claude-plugin/marketplace.json  marketplace entry for `claude plugin marketplace add`
pyproject.toml                   package `ai_harness`, console scripts `harness`, `local-run`
src/ai_harness/
  config.py                      load/validate harness.yaml and hosts.yaml (pydantic models)
  hosts.py                       reach a host: ssh, launcher detection, model list, health
  ladder.py                      rung iteration, reset-between-rungs, time box
  worktree.py                    bare clone, worktree add/remove, push/fetch helpers
  local_run.py                   `local-run` CLI: task guard, run on rung, log JSONL
  bench.py                       `harness bench`
  doctor.py                      `harness doctor`
  status.py                      `harness status` and live page render
  usage.py                       subscription usage reader (D16)
  init/                          `harness init` phases: detect, ask, provision, verify
  mcp/local_llm.py               MCP server: embed, summarise, extract, classify, draft
  mcp/literature.py              MCP server: index and search book skills
  literature/                    book-to-skill wrapper, cache by hash, placement rules
skills/<name>/SKILL.md           plugin skills
templates/                       harness.yaml, AGENTS.md, hosts.yaml, critic prompt, run.md
tests/                           pytest; tests/fake_llm.py is the OpenAI-compatible fake
docs/harness/                    design, roadmap, per-PR plans, runbook pages
README.md                        the human guide
```

---

### Task 1: Repository skeleton and schemas

**Files:**
- Create: `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `pyproject.toml`, `src/ai_harness/__init__.py`, `src/ai_harness/config.py`, `templates/harness.yaml`, `templates/hosts.yaml`, `templates/providers.yaml`, `.github/workflows/ci.yml`
- Test: `tests/test_config.py`

**Interfaces:**
- Produces: `config.load_project(path) -> ProjectConfig` with fields `roles: dict[str, Runner]`, `ladder: list[Rung]`, `bar: str`, `budget_threshold: float`, `plugins: dict[str, str]`, `runner: Literal["claude"]`.
- Produces: `config.load_hosts(path=None) -> list[Host]` with `Host(name, ssh, launcher: Literal["lemonade","ollama","none"], base_url, worktree_root, roles: list[str] | None)`; missing file returns `[]`.
- Produces: `config.load_providers(path=None) -> list[Provider]` with `Provider(name, runner: Literal["opencode"], model, data_policy: Literal["private","trains"], quota: Quota | None)`; missing file returns `[]`.
- Produces: `config.effective_ladder(project, hosts, providers, repo_private: bool) -> list[Rung]` applying D4 order and the `trains` gate.

**Acceptance:** `uv run pytest tests/test_config.py` passes on both CI runners; `claude plugin validate .` passes; a `hosts.yaml` with zero hosts loads to an empty list and a `harness.yaml` without `ladder:` gets the default ladder `[haiku, sonnet, coordinator]`; a `trains` provider is absent from the effective ladder when `repo_private=True` and `allow_training_providers` is unset.

- [ ] Plan with `/ce-plan`, build, PR, critic gate, merge.

### Task 2: Migrate existing skills and tools

**Files:**
- Create: `skills/harness/SKILL.md`, `skills/delegate-local/SKILL.md`, `skills/gauntlet-loop/SKILL.md`, `src/ai_harness/doctor.py`, `docs/harness/migration.md`
- Test: `tests/test_doctor.py`

**Interfaces:**
- Consumes: `config.load_hosts`.
- Produces: `harness doctor` exit 0/1 with checks `plugin installed`, `skills linked`, `hosts.yaml present`, per host `reachable`, `launcher found`, `model loaded`; all host checks SKIP when `HARNESS_FAKE_HOSTS=1`.

**Acceptance:** skills copied verbatim from the machines repository with only path and name edits; `uv run harness doctor` on a machine with no hosts prints PASS for plugin checks and SKIP for host checks; a PR in the machines repository removes its copies and installs the plugin, referenced by URL in `migration.md`.

- [ ] Plan, build, PR, gate, merge. Companion PR in the machines repository.

### Task 3: `local-run` v2

**Files:**
- Create: `src/ai_harness/hosts.py`, `src/ai_harness/worktree.py`, `src/ai_harness/ladder.py`, `src/ai_harness/local_run.py`, `src/ai_harness/runners/{claude.py,opencode.py}`, `src/ai_harness/quota.py`
- Modify: `skills/delegate-local/SKILL.md`
- Test: `tests/test_ladder.py`, `tests/test_worktree.py`, `tests/test_local_run.py`, `tests/fake_llm.py`

**Interfaces:**
- Produces: `local-run "<task>" [--branch B] [--timebox 900]`: guard (branch has a failing-test commit ahead of default, worktree clean), push branch, for each rung: ensure host worktree, run launcher in `-p` JSON mode with the allowed-tools list, time box, fetch, run acceptance, on failure reset and next rung; log one JSON line per attempt to `~/.local/state/harness/runs.jsonl` with `step, rung, host, model, tokens_in, tokens_out, ms, lines, result`.
- Produces: `ladder.run(task, rungs, on_attempt) -> Attempt` used by Task 9.
- Produces: `runners.Runner` protocol `run(task, cwd, model, allowed_tools, timebox) -> RunResult(exit_code, text, tokens_in, tokens_out, ms)`, implemented by `ClaudeRunner` (`claude -p --output-format json`) and `OpenCodeRunner` (`opencode run --model <provider/model> --format json`).
- Produces: `quota.cooldown(rung, until)` and `quota.is_cool(rung) -> bool` backed by `~/.local/state/harness/quota.json`; a 429 or connection error cools the rung until its window reset or 30 minutes when unknown.
- Allowed tools for workers: `Bash(cargo *)`, `Bash(dotnet *)`, `Bash(npm *)`, `Bash(pnpm *)`, `Bash(npx *)`, `Bash(uv *)`, `Bash(git status*)`, `Bash(git diff*)`, `Bash(git add *)`, `Bash(git commit *)`, `Bash(git push origin HEAD:<branch>)`, `Bash(ls *)`.

**Acceptance:** with a fake host (`HARNESS_FAKE_HOSTS=1` points ssh at a local subprocess and the launcher at `tests/fake_llm.py`), a task whose fake worker fails on rung 1 and succeeds on rung 2 ends with the branch green, the worktree reset between rungs, and two log lines. A fake provider rung returning 429 is skipped, recorded in `quota.json` with a reset time, and skipped again without a call on the next run until that time. `OpenCodeRunner` passes the same task text and produces the same `RunResult` shape as `ClaudeRunner` against the fake. The guard refuses a branch with no red commit with the message `local-run: no failing test committed on <branch>; write the red test first`. Verified by hand once against a real host: `lemonade launch claude -- -p` passes flags through, or the documented fallback is used and recorded in `design.md`.

- [ ] Plan, build, PR, gate, merge. First task where `[local]` steps run through the previous `local-run`.

### Task 4: `harness bench`

**Files:**
- Create: `src/ai_harness/bench.py`, `templates/bench/<stack>/` (python, rust, dotnet, node, prose), `templates/models.md.j2`
- Test: `tests/test_bench.py`

**Interfaces:**
- Consumes: `hosts.py`, `ladder.py`, `fake_llm.py`.
- Produces: `harness bench [--stack S] [--host H] [--provider P]` writes `docs/harness/models.md` and updates `roles:` in `harness.yaml` with `coder`, `fast_draft`, `embed`, `critic_local` (empty when nothing passes). Score per model: `passes/3`, median wall seconds, tokens per second, backend (`cuda`, `rocm`, `vulkan`, `cpu`, `cloud`). Providers are benched through `OpenCodeRunner` with the same tasks. Threshold: fewer than 2 of 3 passes excludes the model from the ladder.

**Acceptance:** against the fake endpoint configured to pass 3, 2 and 1 tasks for three fake models, the generated `models.md` lists all three and `harness.yaml` includes exactly the first two in `ladder:`; with zero hosts the command prints `bench: no hosts configured` and exits 0 without writing.

- [ ] Plan, build, PR, gate, merge.

### Task 5: `local-llm` MCP server

**Files:**
- Create: `src/ai_harness/mcp/local_llm.py`, `.claude-plugin/plugin.json` (mcp entry)
- Test: `tests/test_local_llm.py`

**Interfaces:**
- Produces tools: `embed(texts: list[str]) -> list[list[float]]`, `summarise(text, max_words)`, `extract(text, schema: dict) -> dict`, `classify(text, labels: list[str]) -> str`, `draft(brief, kind: Literal["doc","test","dto"]) -> str`. Each resolves its model from `roles:` in `harness.yaml` and the first reachable host; when none, returns `{"unavailable": "no local host; use a Claude subagent"}`.

**Acceptance:** all five tools pass against `fake_llm.py`; with `HARNESS_FAKE_HOSTS=1 HARNESS_NO_HOSTS=1` every tool returns the unavailable payload and the server stays up; `claude mcp list` shows the server after plugin install.

- [ ] Plan, build, PR, gate, merge.

### Task 6: `literature` MCP server and book pipeline

**Files:**
- Create: `src/ai_harness/literature/{extract.py,notes.py,place.py,cache.py}`, `src/ai_harness/mcp/literature.py`, `templates/lit-skill/SKILL.md.j2`
- Test: `tests/test_literature.py`, `tests/fixtures/tiny-book.md`

**Interfaces:**
- Consumes: `book-to-skill` extractor (vendored call, not a fork), `local_llm.embed`, `local_llm.summarise`.
- Produces: `harness lit add <path|url> [--slug S]`: extract, chapter notes via local summarise, SKILL.md left as a draft for the coordinator to finish, placement by repository visibility (`gh repo view --json visibility`) into `.agents/skills/lit-<slug>/` or the private literature repository path from `harness.yaml`, cache key sha256 of the source file. Tools: `lit_search(query, k) -> list[Hit(skill, chapter, score, excerpt)]`, `lit_list()`.
- Produces: `AGENTS.md` section `## Literature` regenerated from `lit_list()`.

**Acceptance:** `tiny-book.md` becomes a skill directory with `SKILL.md`, `chapters/ch01-*.md`, `glossary.md`; running `lit add` twice hits the cache; `lit_search("the term in chapter one")` returns chapter one first against the fake embedder; placement picks the private repository path when visibility is `PUBLIC`.

- [ ] Plan, build, PR, gate, merge.

### Task 7: `harness init`

**Files:**
- Create: `src/ai_harness/init/{detect.py,ask.py,provision.py,verify.py,__main__.py}`, `skills/harness-init/SKILL.md`, `templates/AGENTS.md.j2`, `templates/critic.md.j2`
- Test: `tests/test_init.py`

**Interfaces:**
- Consumes: everything above.
- Produces: `harness init` in a repository: detect (stack, hosts, launchers, accelerator and server backend per D22, plugin versions, symlink support, repository visibility, existing project layer), ask (project kind, bar candidates, literature yes/no, collaborators; on a machine with no `hosts.yaml` or `providers.yaml`, a short wizard that detects Ollama or Lemonade, offers OpenCode Zen signup with the data-policy flag, and prints install hints), provision (`harness.yaml`, `AGENTS.md` under 60 lines, `.agents/skills/` plus link or copy, `docs/harness/`, branch protection via `gh api`, bench offer), verify (`harness doctor`, print the effective ladder for this machine). Idempotent: a second run changes nothing and prints `init: up to date`. Joiner mode (D23): when the layer exists, only the machine side runs.

**Acceptance:** in a temporary git repository with zero hosts, `harness init --yes` produces the layer, `harness doctor` passes, and a second run is a no-op; with one fake host the ladder in `harness.yaml` starts with that host; the generated `AGENTS.md` is at most 60 lines and names every pinned skill; in a repository that already has the layer, `harness init` touches no tracked file and writes only under `~/.config/harness/`; a fake Ollama reporting a model with zero VRAM produces the CPU-spill warning.

- [ ] Plan, build, PR, gate, merge.

### Task 8: `grill-with-docs` skill

**Files:**
- Create: `skills/grill-with-docs/SKILL.md`, `skills/grill-with-docs/references/{bar-picking.md,plan-step-format.md}`, `templates/docs/{spec,architecture,plan,bar}.md.j2`
- Test: `evals/grill-with-docs/` (plugin eval suite)

**Interfaces:**
- Produces: on invocation, runs the chain in D14, asks one question at a time, offers three bars, writes the four documents plus `docs/harness/gauntlet-prompt.md` on a branch and opens a PR. Every plan step carries: `id`, `title`, `owner: local|claude`, `files`, `acceptance`, `pr_boundary`, `depends_on`.

**Acceptance:** the eval suite starts from a repository prepared by Task 7 with a one-line idea and checks that the five documents exist, every step in `plan.md` has all seven fields, and `bar.md` names a concrete inspectable bar.

- [ ] Plan, build, PR, gate, merge.

### Task 9: `harness run`, critic gate, budget

**Files:**
- Create: `src/ai_harness/{run.py,critic.py,usage.py,status.py}`, `skills/harness-run/SKILL.md`, `templates/run.md.j2`, `templates/status.html.j2`
- Test: `tests/test_run.py`, `tests/test_critic.py`, `tests/test_usage.py`

**Interfaces:**
- Consumes: `plan.md` step format from Task 8, `ladder.run` from Task 3.
- Produces: `/harness-run` as a `/loop` skill; per tick: read `run.md`, pick ready steps (dependencies done, no file overlap with open steps), red by coordinator, green via `local-run`, critic gate (`critic.review(pr) -> Pass | Gap(text)`, up to three rounds, model per D8), PR via `gh`, merge, update `run.md`. `usage.read() -> fraction` and the threshold rule from D16; manual `budget: low` honoured. Each step writes `docs/harness/stats/<step-id>.json` on its own branch (rung, runner, host class, model, tokens in/out, ms, lines, critic rounds) per D17. `harness status` aggregates the directory into a table and `docs/harness/status.html` with the local versus free tier versus Claude split. On an exhausted subscription window the loop sleeps until the reset time.

**Acceptance:** with a three-step fake plan where step 2 depends on step 1 and step 3 overlaps step 1's files, one tick opens exactly one PR; a critic returning two gaps then pass results in three rounds and a merge; usage above threshold skips critic calls and only local rungs run; killing the process mid-tick and running again resumes with no duplicate PR; two steps merged from parallel branches leave two stats files and no conflict; `harness status` shows the split by rung class.

- [ ] Plan, build, PR, gate, merge.

### Task 10: Runbook and README

**Files:**
- Modify: `README.md`
- Create: `docs/harness/runbook/{00-concepts,10-install,20-hosts,30-new-project,40-grill,50-run,60-collaborators,70-troubleshooting}.md`

**Acceptance:** a reader with Claude Code and no hosts can follow `10-install` and `30-new-project` to a passing `harness doctor` and a first `/grill-with-docs`; a reader with one host can add it in `20-hosts` and see it appear in `harness bench`; no machine names anywhere (`grep -ri` for the author's host names returns nothing).

- [ ] Plan, build, PR, gate, merge.

---

## Status

| Task | PR | Status | Rung split (local / claude lines) |
|---|---|---|---|
| 0 Plan and README guide | this PR | open | n/a |
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |
| 6 | | | |
| 7 | | | |
| 8 | | | |
| 9 | | | |
| 10 | | | |
