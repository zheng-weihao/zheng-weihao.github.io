---
layout: post
title: "Your AI agent works one repo at a time. Your work doesn't."
date: 2026-07-12
tags: ai devtools git llm
---

You're working on a service. It imports a shared library — different repo. Your agent writes clean code, ships the feature. Build fails: the library function it called doesn't exist. There's a different function, it takes different args, it was renamed three months ago. The agent didn't hallucinate out of stupidity. It answered from the only thing it had — training data — because the actual library lives in a repo it has never seen.

So you paste the library source into the prompt. One function, fine. Three files deep into a call chain, you're the file system, copying by hand.

Or vendor a copy of the dependency. Now the agent can read it — and it's already stale, drifting from upstream, polluting your diff.

Or point a RAG index at everything. The agent gets fragments — top-k chunks, out of context, read-only. Good enough to answer a question. Not good enough to trace a call across modules. Definitely not good enough to *change* anything.

Or run a separate agent per repo and become the integration layer between sessions that can't see each other.

Everyone who hits this reaches for the same patches. None fix the actual problem: **the agent's context is scoped to one repo. Real work spans multiple repos. You've become the context bridge — copy-pasting, re-explaining, manually keeping state consistent across sessions and repos.**

## What a fix looks like

Not cleverness. *Grounding* — give the agent the real source of everything the task touches, in one place it can grep, read, and edit.

Not an index. Not a snapshot. Not fragments. The actual repos — as real git worktrees with full history, in one directory tree.

Here's what that changes:

**The agent greps real source instead of guessing.** Need the signature of `BatchGet`? Open the file. Need to know why a config exists? `git blame`. API changed three months ago? Read the commit log — not stale training data.

**Read and write on the same substrate.** The same directory where the agent reads the library to verify an API is where it can edit the library, commit, and open a PR. When it changes the shared library, it immediately compiles the service against the change. No tool switching. Reading and writing aren't two mechanisms — they're one directory, used for both.

**Your toolchain doesn't know anything happened.** The workspace is a real directory tree — not symlinks, not editor virtual views. Drop a `go.work` at the root, `use ./service ./lib`, and `go build` and `gopls` resolve across repos. Cross-module jump-to-definition works. Editing the library and seeing it resolve in the service works. Same for Cargo, pnpm, Gradle. Zero adaptation — the toolchain sees a standard directory and works with it. That's not a feature you build. It's what falls out of using the real filesystem.

**Knowledge sticks around.** After the agent spends an hour learning that your handlers live in `internal/handler/`, that errors are wrapped a particular way, that the deploy path has one non-obvious gotcha — that understanding gets written back as plain markdown. Next session reads it and skips the re-exploration. Not a vector store. Not a database. Just files in git, with history, branches, and blame for free.

## What happened when I tried this

I built a tool called [Orbit](https://github.com/orbcli/orbit) — bash + git + markdown, no server, no embeddings, no runtime. v0.1, one person's weekends, deliberately minimal. Then I used it on the project it was written for.

The setup: Orbit's own code repo plus a separate knowledge repo where I keep design notes, decision records, and architecture rationale. Two repos, one workspace. The agent sees both.

**What worked.** I asked the agent to verify whether `orbit add` handles the case where a branch is already checked out by another worktree. Instead of guessing or citing training data, it opened the source, grepped for the relevant function, read the error path, and told me: "It detects this and falls back to a detached HEAD — here's the line." Then it suggested a clearer error message, edited the file, and committed. No copy-paste. No hallucination. The dependency source was *right there*, and the agent used it like a programmer would — open the file, read, decide, edit.

Another time, I was debugging why `orbit done` wasn't pruning a worktree correctly. The agent traced the call from `done` into the cleanup function, found a conditional that checked for uncommitted changes but not for untracked files, and fixed it — across two functions in different files. The kind of thing that takes ten minutes of manual `grep` + `git blame` when you're doing it by hand in a terminal. The agent did it in one pass because the full source was in its context.

**What didn't work.** The knowledge repo has short "memo" files — per-repo navigation notes that help the agent orient. After a few weeks of active development, some memos described code that had since been refactored. The agent read an outdated memo, assumed a module still had a certain entry point, and wrote code against a function that had been renamed. The exact same failure mode as the training-data problem — just moved one layer. The memo was stale.

I don't have a clean fix for this yet. I've added a staleness warning (if the memo is older than N commits, flag it), but that's a blunt heuristic. Content-level staleness detection — knowing that what the memo *says* no longer matches what the code *does* — is an open problem. The grounding solved the cross-repo blindness; it didn't solve the knowledge-freshness question.

## The honest boundaries

- **It's early.** v0.1.0 — a bash script leaning on git worktrees. One person's weekends.
- **Concurrent multi-writer is out of scope.** One person (or one agent at a time) works well. Many people PR-ing overlapping knowledge across repos — whose judgment wins, how you merge intent — deliberately unsolved.
- **It doesn't make the agent smarter.** It makes it *grounded*. The intelligence was never the bottleneck; the missing source was.

## The question this didn't answer

Fixing the reading problem — giving the agent real source across repos — turned out to be the easier half. The harder question crept in through the memo-staleness failure: once the agent has a knowledge repo alongside the code, how does knowledge flow *back* after the code changes? The wiki informs the code, sure. But after a refactor merges, the wiki is wrong — and nothing tells it. That bidirectional flow is what I'm still working through.

---

If your agent has ever confidently called an API that doesn't exist, the missing piece probably isn't a better model. It's the repo it couldn't see. Orbit is [here](https://github.com/orbcli/orbit) — if you want to experiment with this, I'd like to hear where it works and where it breaks.

---

*One more thing. This article was drafted inside an agentic workspace. The knowledge repo — my design notes, decision records, the concept definition itself — sat alongside the code repo, and the agent that helped me write it read from both and wrote back to both. When the agent wrote a paragraph that contradicted a design decision I'd already recorded, it caught itself by grepping the knowledge repo. The tool I'm describing is the one I used to describe it.*
