---
title: "Agent works one repo at a time. Work doesn't."
date: 2026-07-12
tags: ai devtools git llm
image: /assets/img/one-repo-og-1200.jpg
---

I support features across multiple repos. Backend, contract, frontend — three repos, one feature. Monorepo? Too expensive. Too much migration. I had to make polyrepo work with an agent. This is the path I took to get there — it ended in a small open-source tool called [Orbit](https://github.com/orbcli/orbit), but the tool is the last step, not the point.

## Attempt 1: copy-paste

One repo, one session — that's the agent default. So I copy the API signature from the backend session, paste it into the frontend session, re-explain the context. Every session starts from zero. I'm the proxy. Works for simple changes, breaks the moment you need to trace a call chain across repos.

## Attempt 2: start the session at the project root

Put all repos under one directory — `~/project/`. Start the agent there. It sees everything.

The problem: "everything" is 36 repos — my 3, plus other project repos, the CI/CD toolchain, and the community repos I pull in for verification like Kubernetes and gRPC. Start the agent there and the first thing it does is crawl. It walks the directories trying to work out what's what, and I sit and wait. When it's done it hands me a flat list — no idea which repos are the target, which are mine, which are community repos I only want to grep, not touch. I'm still the middleman, just shuffling paths around instead of code.

Worse: there's one working copy per repo, so a repo can only serve one task at a time. A second request comes in while I'm mid-change — I can't just start on it. The repo is locked to the first task's branch and its uncommitted edits. To touch the second task I have to `git stash` the work in flight, switch branch, do the thing, switch back, unstash, and hope nothing collided. Two tasks, one working directory, constant context-swapping by hand.

## Attempt 3: `--add-dir`

The locking was the dealbreaker — parallel tasks aren't a nice-to-have for me, they're the job. So I dropped the single-directory idea and left each repo in its own separate location, wiring them together per session instead. Claude Code has `--add-dir` — add extra directories to the session. I start the agent in the backend, `--add-dir` the contract and frontend. The agent can read all three.

But now the repos sit at scattered absolute paths, and that's where the toolchain breaks. Compilers, bundlers, language servers — they all assume the modules they resolve against live at real relative paths inside one project layout. Point them at directories miles apart and cross-repo resolution falls apart: jump-to-definition misses, the build can't find the sibling module locally. So I push the change to a remote, pull the new version back down, rebuild. A one-line edit becomes a publish-test cycle.

The agent can see the repos. It can't work with them as one thing.

## Where I landed

The answer turned out to be unglamorous: the repos need to be real directories in a real directory tree. Not views. Not symlinks. Actual git worktrees, sitting side by side, addressable by relative path.

The shape:

```
project-root/
  .repos/                ← all repos live here, one copy, shared across workspaces
    backend/
    frontend/
    contract/
    kubernetes/          ← community repo, grep for feature + version verification
    toolchain/           ← CI/CD, linter configs, agent skills
    docs/                ← design notes, decisions, ADRs
  task-01/               ← workspace: isolated environment for one task
    backend/             ← worktree (actual development directory)
    frontend/
    contract/
    kubernetes/          ← worktree (pulled in only when this task needs it)
```

The worktrees are real directories, but they share the pool's git objects — so spinning up a workspace is near-instant and throwing one away costs nothing.

When the backend, contract, and frontend sit in the same real tree, four things unlock:

**Cross-repo context consistency.** One agent, one workspace — every repo it pulls in shares the same context. The agent changes the backend, sees the result immediately in the frontend. No copy-pasting, no re-explaining, no switching tools. Full git history — blame, log, branch topology — across every repo. The context bridge is gone.

**The agent pulls in the repos it needs — not the ones I name.** Nothing loads up front. I give it a goal; it reads the pool roster, and pulls repos in one at a time as it discovers it needs them — the contract to verify an API, the frontend to check the call site, Kubernetes to grep an API against its source. No up-front crawl, no waiting while it walks 36 directories, no context bloated with repos this task never touches. The agent isn't just reading more repos. It's deciding which ones matter, and paying for only those.

**Real directory tree, zero toolchain adaptation.** The go workspace file just works — real relative paths, cross-repo build and jump-to-definition, no remote round-trips. Same for Cargo, pnpm, Gradle. The toolchain doesn't know anything special happened — it sees a standard directory and works with it. That's not a feature you build. It's what falls out of using the real filesystem.

**Task isolation.** One workspace = one task. When the agent finishes the API change, the workspace is done — reclaim it, start a new one for the next task. No context leaking, no branch contamination.

## Building it, and what building it surfaced

I built a prototype that assembles repos into this shape. It's called [Orbit](https://github.com/orbcli/orbit) — bash + git + markdown. No server, no runtime, no embeddings, no index to rebuild. The whole thing is a directory tree and some git plumbing.

Running real tasks on it surfaced the things the design never told me. Some I've fixed, some I haven't:

- **Fixed — the per-repo notes were too heavy.** First cut had the agent write a 40–80 line survey of each repo. Maintaining it cost more than the work it saved, so I cut it: each repo now gets a compact card answering two questions — when to add this repo, and where to start reading — captured as a byproduct of working, not a separate chore.
- **Partly fixed — those notes go stale.** The agent will happily follow a note describing a function that got renamed last week. It now tracks how far behind each note is and flags it on cold start. Still open: the refresh is advisory, not enforced — an agent can see the flag and ignore it. TODO.
- **Open — it's bash, and "bash script" makes people flinch.** The zero-infra story is a feature: no runtime, no daemon, just git and files. But "reliability of a bash script" is a fair question, and it's the first thing a contributor asks. The spec is detailed enough to be implementation-independent, so the open test is whether the whole thing reimplements in Go with zero friction — a single static binary, still zero-infra. If it does, that proves the spec is a model, not a script.

The concept — a workspace where agent context spans multiple repos as real, editable, history-bearing source — is the thing. Orbit is one prototype of it.

If you want to see the shape for yourself, this drops you into a demo workspace where one change has to land in two repos in lockstep — the exact problem this article started with:

```bash
curl -sL https://raw.githubusercontent.com/orbcli/orbit/main/examples/demo/try.sh \
  | bash -s -- --claude
```

Swap `--claude` for `--qodercli`, or drop it entirely to install just the runtime for any other agent. Clean up with `rm -rf ~/orbit-try`.

The source, issues, and roadmap all live on GitHub: [github.com/orbcli/orbit](https://github.com/orbcli/orbit).

---

*One more thing. This article was drafted inside an agentic workspace — the concept definition and the code sat in the same directory tree, and the agent that helped me write it could grep both. The setup I'm describing is the one I used to describe it.*