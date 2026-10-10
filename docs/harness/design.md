# ai-harness: design decisions

Status: accepted 2026-10-10. Each numbered decision below was settled in a grilling session
before any code was written. Change one by opening a PR against this file; the roadmap in
[roadmap.md](./roadmap.md) and the README follow from it.

## What this is

A reusable kit that turns any repository into a project where **one Claude session on a
subscription coordinates**, and **as much typing as possible is pushed down** to local models,
cheaper Claude tiers, or whatever is available on the machine running it. It ships as a Claude
Code plugin plus a small committed layer per project, and it is built so that a collaborator with
nothing but Claude Code gets the same workflow with the local parts switched off.

The end state for a project is: run `/grill-with-docs`, get spec, architecture, plan and quality
bar as documents, then run `/harness-run` and watch small pull requests arrive until the plan is
empty.

## Vocabulary

| Term | Meaning |
|---|---|
| Coordinator | The interactive Claude Code session on the subscription. Plans, reviews, merges. Opus or Fable. |
| Host | A machine that can run a local model server and has a checkout of the project. Zero or more per setup. |
| Worker | A headless Claude Code process (`claude -p`) running one plan step on a host against a local model, or a subagent on a cheaper Claude tier. |
| Rung | One position in the fallback ladder: where a step is tried next after the previous runner failed. |
| Step | The smallest unit of a plan: one branch, one PR, one failing test, one acceptance command, an owner tag. |
| Critic | A fresh-context reviewer that gates every PR against the step's acceptance criteria and the project bar. |
| Bar | The concrete thing the critic compares against: a test suite, a reference implementation, exemplar text, a latency target. |
| Ladder | The ordered list of rungs for a project, in `harness.yaml`. |

## Decisions

### D1. Shape: plugin plus a committed project layer

The kit is a Claude Code plugin (`mwo-dk/ai-harness` marketplace) containing skills, two MCP
servers and Python command-line tools. A project that adopts it commits a small layer:

```
harness.yaml           roles, ladder, bar, pinned plugin versions
AGENTS.md              the 60-line brief every agent and worker reads
.agents/skills/        project skills (literature, project-specific), canonical path
.claude/skills         symlink to .agents/skills (copy on machines without symlinks)
docs/harness/          spec, architecture, plan, bar, run state, model bench
```

Why both: the plugin is versioned and installed once per machine; the layer is what makes a
clone self-describing for someone who has never seen the plugin.

### D2. Skill path: `.agents/skills` canonical, `.claude/skills` a symlink

Claude Code reads `.claude/skills`. OpenCode and other Agent Skills hosts read `.agents/skills`.
Model servers (Ollama, Lemonade) read nothing: skills reach a model only through the agent
harness that sends the prompt. So the canonical location is the standard one, and the Claude
path is a link. On machines where symlinks are unavailable (Windows without Developer Mode),
`harness-init` copies instead, adds the copy to `.gitignore` locally, and `harness doctor`
warns when the copy drifts.

### D3. Worker shape: vendor launcher on the host, full skills

Both Ollama (`ollama launch claude --model <m>`) and Lemonade (`lemonade launch claude`) start a
full Claude Code session against a local model. In print mode (`-p`) that session is headless:
it runs to completion and prints one JSON result. The worker therefore runs **on the host**, in
the host's own checkout, with the project's skills, `AGENTS.md`, CLAUDE.md and hooks loaded.

The coordinator never calls a model API for a worker. It runs a process:

```
ssh <host> "cd ~/wt/<repo>/<branch> && <launcher> claude -- -p '<task>' --output-format json ..."
```

and reads three things back: exit code, the JSON `result` text, and the branch the worker pushed.
The diff is the contract; the worker's own report is never evidence.

Fallback when a host has a model server but no checkout or launcher: the previous design, a
`claude --bare` worker on the coordinator's machine with the model remote and the `AGENTS.md`
brief injected into the prompt.

### D4. The ladder

A task is self-contained text (goal, files, conventions, acceptance command), so it ports
unchanged between runners. Only the runner changes:

| Rung | Runner | Cost |
|---|---|---|
| 1..n | each configured host, in order | electricity |
| n+1 | Agent subagent, `model: haiku`, same worktree | cheap subscription |
| n+2 | Agent subagent, `model: sonnet` | mid |
| last | coordinator does it | Opus/Fable |

A rung is skipped when: host unreachable, nonzero exit, time box hit (default 15 minutes), or the
acceptance command fails after one retry. Before each retry the worktree is reset to the committed
red test (`git restore . && git clean -fd`) so no rung inherits the previous rung's mess. The
plan records which rung landed each step. A setup with zero hosts has a ladder that starts at
Haiku; nothing else changes.

### D5. Code on hosts: bare clone, worktrees, GitHub as hub

Each host keeps one bare clone per project and one worktree per branch under
`~/wt/<repo>/<branch>`. Flow per step: coordinator pushes the step branch to origin, the host
fetches and adds a worktree, the worker commits and pushes the same branch, the coordinator
fetches and reviews. Worktrees live outside the repo; `.worktrees/` is ignored anyway.

Branch protection on the default branch is set by `harness-init` via `gh api`: pull request
required, no direct push, no force push. Workers are allowed `git push origin HEAD:<own branch>`
and nothing else in git beyond `status`, `diff`, `add`, `commit`.

Not chosen: host-to-host fetch over the tailnet (three copies, confusing), shared filesystem
(breaks worktree metadata and locks).

### D6. Steps, branches, PRs

One step, one branch, one worktree, one PR, all off the default branch. Never stacked. The plan
partitions steps by file ownership so parallel steps rarely conflict; a step that touches another
open step's files waits. The coordinator rebases and merges; a conflict is resolved by the
coordinator, never by a worker. Each step must be mergeable alone, with its own test, behind a
flag if it is a slice of a larger feature. Step size is the main control knob of the whole system.

### D7. TDD per step, enforced

1. Red: the coordinator writes the failing test and signatures and commits. The red commit is the
   task specification.
2. Green: the ladder makes it pass.
3. Refactor: the coordinator, with `ponytail-review`, commits.

`local-run` refuses a task whose branch has no failing-test commit ahead of the default branch.

### D8. Critic gate per PR

Before merge, a fresh-context critic subagent reviews the PR blind: diff, red test, the step's
acceptance criteria, the project bar, and the CI result. Output: pass, or the single biggest gap
returned to the builder rung. Up to three rounds, then the coordinator takes over.

- Critic model: Sonnet by default, Opus when the diff touches security or infrastructure paths.
- Critic is never the same tier that built the step.
- `security-review` is added automatically when the diff touches auth, secrets or input parsing.
- The critic prompt is generated by `harness-init` from the project bar, so it is the same critic
  that `harness-run` uses later.

### D9. Model roles: measured, not guessed

`harness-bench` runs a fixed micro-task set per stack (make a red test green, mechanical rename,
write a DTO or mapper, summarise a chapter) against every downloaded coder model on every host.
It scores pass rate, wall time and tokens per second, writes `docs/harness/models.md` and fills
the `roles:` section of `harness.yaml`. A model below two of three passes is not in the ladder.
Running stats from `local-run` refine the choice over time.

### D10. Only Claude Code as runner, for now

Hosts can also launch OpenCode, Codex and others against the same models. One runner means one
prompt format, one JSON contract, one allowed-tools model. `harness.yaml` has a `runner:` field
so a second runner is a one-line change after the bench shows a reason.

### D11. Two MCP servers, with an honest split of what local models may do

`local-llm` exposes one-shot tools over any OpenAI-compatible local server. Only tasks a 27B to
35B class model does well are exposed:

| Tool | Local? | Reason |
|---|---|---|
| embed | yes | small embedding models are near state of the art |
| summarise, extract, classify (bounded input) | yes | long context, instruction following is fine |
| draft (doc, test skeleton, DTO) | yes, coordinator rewrites | cheaper to edit than to type |
| literature chapter notes, glossary, cheatsheet | mostly | per chapter, mechanical |
| literature SKILL.md synthesis | no | needs judgement about what matters |
| critic, review, planning | no | existing harness rule |

`literature` exposes search over book-derived skills and chapter text using local embeddings.

Both answer "unavailable" cleanly when no local server is reachable, so a Claude-only setup does
not break. The task-type to runner table lives in `harness.yaml` under `roles:`; the `harness`
skill reads it rather than carrying the knowledge in prose.

### D12. Literature pipeline

`harness-init` asks whether literature is relevant and offers a picker over a configured books
directory plus free paths and URLs. Per book: extraction local (`book-to-skill`), chapter notes
local, SKILL.md written by the coordinator. Output goes to `.agents/skills/lit-<slug>/` when the
project repository is private, otherwise to a private literature repository referenced from
`harness.yaml`. PDFs are never committed. Results are cached by book hash so a second project
reuses a skill in seconds. Generated skills are listed in `AGENTS.md` so every agent is told to
consult them.

### D13. Pinned skill set

Installed by the plugin and named in `AGENTS.md`: superpowers, compound-engineering,
mattpocock-skills, ponytail, caveman, i-have-adhd, skill-creator, code-review, security-guidance,
plus this repository's own skills. Versions are pinned in `harness.yaml`; `harness-init` warns on
drift. Workers receive only caveman as an output style (one rule set, less drift). The canonical
TDD skill in the harness rule is mattpocock `tdd`.

### D14. `grill-with-docs`

A skill that chains: `ce-brainstorm` when the idea is vague, `grilling` one question at a time,
`domain-modeling` for `CONTEXT.md`, `ce-plan` with every step tagged `[local]` or `[claude]` and
given files, acceptance command and PR boundary, `ce-doc-review`, then a choice between three
candidate bars, then the literature question. It writes `docs/harness/spec.md`,
`architecture.md`, `plan.md`, `bar.md` and a paste-ready gauntlet prompt, all on a branch via PR.

### D15. `harness-run`: a durable loop, not one giant prompt

The published gauntlet loop is one large prompt with all state in a single session. Here the
state lives in `docs/harness/run.md`: steps, status, rung used, critic rounds, PR URLs. Each
`/loop` tick: pick ready steps by file ownership, red, green via ladder, critic gate, PR, merge,
mark done, repeat until the plan is empty. A crash, a usage limit or a closed laptop loses nothing;
the next tick resumes from the file. Ultracode and `lfg` are off by default.

### D16. Budget awareness

Each tick reads subscription usage. Above a threshold in `harness.yaml` (default 80% of the
five-hour window) only local rungs run, critics pause and the loop sleeps longer. A manual
`budget: low` in `run.md` overrides. If usage data turns out unreadable, the loop falls back to
surviving limit errors, which the durable state already allows.

### D17. Watching a run and the local/Claude split

Every step logs rung, host, model, tokens in and out, wall time and committed lines.
`harness status` prints a table; a live page is regenerated each tick. Both show, per project,
steps, commits and lines by local versus Claude tier, token totals, and failures and fallbacks
per model.

### D18. Hosts per machine, roles per project

Nothing in a project names a machine. Hosts live in `~/.config/harness/hosts.yaml` on each
machine. A project only names roles. A collaborator with no hosts gets a ladder that starts at
Haiku; one with their own Ollama is offered a bench and the host is added.

### D19. Ownership split with the machines repository

This repository owns the AI harness: skills, `local-run`, MCP servers, bench, the `hosts.yaml`
format. A machine-provisioning repository owns the operating system, model server services and
network, installs this plugin, and writes its `hosts.yaml`. Old copies of the skills there are
deleted after migration.

### D20. Language, platforms, tests

All command-line tools are Python run through `uv`; no shell scripts. Linux and macOS first
class; Windows supported through WSL (recommended) and tolerated natively (symlink fallback,
Git Bash). Tests: `pytest` with a fake OpenAI-compatible endpoint and a `HARNESS_FAKE_HOSTS=1`
switch so nothing in CI needs a real host; skills checked with `claude plugin eval`; GitHub
Actions on Linux and Windows. The kit is built with its own rules: TDD, one step per PR, local
workers from the moment `local-run` v2 exists.

### D21. Public, MIT

Nothing secret lives in the repository by design: hosts in `~/.config`, books in a private
literature repository.

## Facts to verify during the build, not decisions

- `lemonade launch claude` and `ollama launch claude` pass `-p`, `--output-format json` and
  `--allowedTools` through to `claude`. If not, set the environment on the host and call
  `claude -p` directly; the process shape is unchanged.
- Claude Code usage data is readable locally for D16.
- `claude plugin eval` can drive a skill against a temporary repository.
