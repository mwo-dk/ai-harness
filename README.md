# ai-harness

One Claude session plans, reviews and merges. Everything mechanical is pushed down to whatever
is cheaper and present: a local model on a machine you own, a smaller Claude tier, or nothing
extra at all. Work arrives as small pull requests. The plan lives in files, so a run survives a
closed laptop or a usage limit.

This repository is a Claude Code plugin plus a small layer you commit into each project. It is
built to be shared: a teammate with only Claude Code gets the same workflow with the local parts
switched off.

Status: design accepted, implementation in progress. See [the roadmap](docs/harness/roadmap.md)
for what exists today and [the design](docs/harness/design.md) for every decision and why.

## Quick start

Once per machine:

```sh
# 1. Claude Code and the plugin
claude plugin marketplace add mwo-dk/ai-harness
claude plugin install ai-harness@ai-harness

# 2. The command-line tools (Python, through uv)
uv tool install git+https://github.com/mwo-dk/ai-harness

# 3. Optional: tell this machine about local model servers and free-tier providers.
#    Skip both if you only have Claude. harness init writes these files for you through
#    a short wizard, so you do not have to hand-write them.
harness doctor
```

Once per project, new or existing:

```sh
gh repo create <you>/<project> --private --clone    # or clone an existing one
cd <project>
harness init          # detects, asks a few questions, writes the layer, protects main
claude                # open the coordinator session in the repository
```

Then, inside Claude Code:

```
/grill-with-docs      # one question at a time until spec, architecture, plan and bar exist
/harness-run          # builds the plan as small PRs, resumable, until the plan is empty
```

Joining a project someone else started is the same: clone, `harness init`, `claude`. The
command notices the layer is already there and only sets up your machine.

## The idea in one picture

```
            you
             |
   +---------v----------+      plans, writes failing tests, reviews, merges
   |  Coordinator       |      (interactive Claude Code, Opus or Fable)
   |  Claude Code       |
   +----+----+-----+----+
        |    |     |
        |    |     +--------------------------+
        |    +------------+                   |
        v                 v                   v
   host A             host B             Claude subagents
   local model        local model        haiku, then sonnet
   full Claude Code   full Claude Code   (fallback rungs)
   in its own         in its own
   checkout           checkout
        |                 |                   |
        +--------+--------+-------------------+
                 v
          one branch, one PR per step
                 v
          critic gate (fresh context, blind, against the bar)
                 v
          merge
```

Three roles:

- **Coordinator.** The Claude Code session you sit in. It decides what to build, writes the
  failing test that defines each step, hands the step down a ladder of runners, reviews the diff
  that comes back, and merges. It spends subscription tokens on judgement, not typing.
- **Workers.** Headless Claude Code processes that make a failing test pass. On a host they run
  against a local model with the project's skills loaded. Without a host they are Claude subagents
  on a cheaper tier. Same task text, different runner.
- **Critic.** A fresh-context reviewer that gates every pull request against the step's acceptance
  criteria and a concrete quality bar, and sends back the single biggest gap until it passes.

## What "zero, one or many hosts" means

A **host** is a machine that runs a local model server (Ollama or Lemonade, both work), has
Claude Code installed, and holds a checkout of the project. Hosts are described per machine in
`~/.config/harness/hosts.yaml`, never in the project. A project only names roles.

| Your setup | What happens |
|---|---|
| Claude Code only | The ladder starts at a Haiku subagent, then Sonnet, then the coordinator. The two local MCP servers answer "unavailable". Everything else is identical. |
| One host | The host is rung one. Mechanical steps go there first. Bench results decide which of its models fills which role. |
| Several hosts | Each host is a rung in order. Parallel steps can run one per host, on different files and branches. A weak host can be limited to embeddings only. |
| Free cloud tiers | OpenCode Zen, Gemini, Kimi, DeepSeek and others are reached through OpenCode as a second runner. You opt in per provider in `~/.config/harness/providers.yaml` with a data-policy flag; providers that train on your code are skipped on private repositories unless the project allows it. |
| Mixed team | Each person's machine has its own `hosts.yaml` and `providers.yaml`. The project repository is the same for all. Each step's PR carries a stats file saying which rung landed it, so the split is visible to everyone without any shared service. |

Example `hosts.yaml` with two hosts, one strong and one weak:

```yaml
hosts:
  - name: workstation          # any name you like, never committed
    ssh: workstation            # an entry in ~/.ssh/config, or user@address
    launcher: lemonade          # lemonade | ollama | none
    base_url: http://127.0.0.1:13305
    worktree_root: ~/wt
  - name: laptop
    ssh: laptop
    launcher: ollama
    base_url: http://127.0.0.1:11434
    worktree_root: ~/wt
    roles: [embed]              # optional: restrict what this host is used for
```

Example `harness.yaml` in a project (generated by `harness init`, edited by hand when you like):

```yaml
runner: claude
ladder: [hosts, haiku, sonnet, coordinator]   # "hosts" expands to this machine's hosts.yaml entries, in order
roles:
  coder: local            # make a red test green, mechanical refactors, DTOs
  fast_draft: local       # first drafts the coordinator rewrites
  embed: local
  critic: sonnet          # opus for security and infrastructure paths
  planner: coordinator
bar: docs/harness/bar.md
budget_threshold: 0.8
plugins:
  superpowers: 6.2.0
  compound-engineering: 2.x
  mattpocock-skills: 1.2.3
  ponytail: 1.x
  caveman: 1.x
  i-have-adhd: 1.x
```

## The flow, start to finish

1. **Install** the plugin once per machine. Add a `hosts.yaml` if the machine can reach any
   local model servers, or skip it.
2. **New project:** run `harness init` in the repository. It detects stack, hosts and plugins,
   asks a few questions (what kind of project, which quality bar, is there literature worth
   turning into skills), writes the committed layer, protects the default branch, and offers to
   bench the local models. Rerunning it is safe.
3. **Think:** run `/grill-with-docs`. It asks one question at a time until nothing is assumed,
   offers three candidate bars, and writes `docs/harness/spec.md`, `architecture.md`, `plan.md`
   and `bar.md` as a pull request. Every plan step has files, an acceptance command, an owner
   tag and a PR boundary.
4. **Build:** run `/harness-run`. Each tick takes the next unblocked steps, writes the failing
   test, sends the step down the ladder, gates the PR with the critic, merges, ticks the box in
   `docs/harness/run.md`, and sleeps. Stop and restart whenever you like.
5. **Watch:** `harness status` or the live page shows every step, its rung, its PR, and the
   running split of lines and tokens between local models and Claude tiers.

## Rules the kit enforces, so you do not have to remember them

- Nothing goes to a worker without a committed failing test on its branch.
- One step, one branch, one PR, off the default branch. Never stacked, never pushed to `main`.
- Workers may edit files, run the build and test tools, and push their own branch. Nothing else.
- The diff and the acceptance command are the evidence. A worker's report is not.
- Two failed attempts on a rung means the next rung. The coordinator is the last rung.
- The critic never shares a model tier with the builder it judges.
- Local models do embeddings, summaries, extraction, classification and first drafts. They do
  not plan, review or judge.
- When subscription usage passes the threshold, only local rungs run and critics pause. When a
  free tier or the subscription is exhausted, that rung cools down until its window resets and
  the loop carries on with the rest.

## Where things live

```
~/.config/harness/hosts.yaml      per machine, never committed
~/.config/harness/providers.yaml  per user, free and paid cloud tiers, never committed
<project>/harness.yaml          roles, ladder, bar, pinned plugin versions
<project>/AGENTS.md             the brief every agent reads, 60 lines at most
<project>/.agents/skills/       project skills; .claude/skills links here
<project>/docs/harness/         spec, architecture, plan, bar, run state, model bench
<project>/docs/harness/stats/   one small file per step: rung, model, tokens, lines
~/wt/<repo>/<branch>            worktrees on every host
~/.local/state/harness/         run logs
```

## For people who only have Claude Code

Clone the project, install the plugin, run `harness doctor`. It reports the plugin and skills as
present and the hosts as skipped. Run `/harness-run` or any skill as usual; the ladder starts at
Haiku. If you later install Ollama and a coder model on your own machine, add a `hosts.yaml`,
run `harness bench`, and your machine becomes rung one for you only. Nothing in the project
changes.

## For people on Windows

WSL is the recommended path and is plain Linux. Native Windows works through Git Bash: the only
difference is that `.claude/skills` becomes a copy of `.agents/skills` instead of a link when
symlinks are unavailable, and `harness doctor` warns when the copy drifts.

## Why not one big autonomous prompt

The published gauntlet-loop method runs one huge prompt in a single session and keeps all state
in that session's memory. It is fast to start and expensive to keep alive, and a crash loses the
reasoning. This kit uses the same roles, builder and blind critic against a bar, but keeps the
plan and progress in files and ships work as many small pull requests. `/grill-with-docs` still
writes the paste-ready prompt as a document, in case you want the other way for a one-off.

## Repository layout

```
.claude-plugin/      plugin and marketplace manifests
skills/              harness, delegate-local, harness-init, harness-run, grill-with-docs, gauntlet-loop
src/ai_harness/      Python: config, hosts, ladder, worktree, local-run, bench, doctor, status, init, MCP servers
templates/           harness.yaml, hosts.yaml, AGENTS.md, critic prompt, run.md, bench tasks
tests/               pytest, with a fake OpenAI-compatible endpoint and fake hosts
docs/harness/        design decisions, roadmap, per-PR plans, runbook
```

## Licence

MIT.
