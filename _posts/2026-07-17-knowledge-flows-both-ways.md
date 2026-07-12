---
layout: post
title: "Knowledge flows into code. What if code flowed back?"
date: 2026-07-17
tags: ai llm git devtools
---

I built an LLM Wiki — a git repo where an agent maintains persistent markdown knowledge: decisions, trade-offs, architecture notes, the "why" behind the code. The idea is [making the rounds](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f); the gist is simple: instead of RAG-ing documents every time, let an agent build and maintain a wiki. Read sources, write summaries, cross-reference, keep it fresh. The wiki lives in git — version history, branches, collaboration, all free.

It works. The wiki accumulates. Decisions get captured. Cross-references form. The "why" is there.

Then I hit a wall.

## The wiki can't see my code

My code lives in a different repo. The agent in the code session can't see the wiki. The agent in the wiki can't see the code. Two repos, two contexts, two agents that don't know each other exists.

So how does the wiki influence my code? In practice: I read the wiki, extract the relevant insight, paste it into the code repo's agent session. I'm copy-pasting. The same manual bridging I was trying to eliminate.

The wiki was supposed to be the agent's knowledge. But if I'm hand-carrying insights from the wiki to the code session, I'm still the file system — just with better-organized files.

## The sources are immutable. Mine aren't.

The LLM Wiki model says: sources are **immutable**. The agent reads them, extracts, writes the wiki. The sources never change. That's fine if your sources are articles and papers — you can't PR someone's published paper.

But my code repo is not immutable. A PR merges. A function gets renamed, a module reorganized, an API deprecated. **After that merge, the code is the new source of truth.** It's a living, evolving thing.

My wiki doesn't know. It still describes the old API. The next agent reads the wiki and writes code against a function that doesn't exist anymore.

I fixed the RAG problem, but the staleness problem is still there — just moved one layer. The wiki is stale because it can't see the code, and the code can't talk back.

## The flip: code is also a source

Here's the insight: **code, after PR merge, is a source.** Not an immutable external document — a living source that changes. And if it's a source, it should be able to flow back into the wiki.

- You merge a refactor. The wiki's "entry points" page is now wrong. An agent that sees both would notice and update the wiki.
- You merge a new API. The wiki should gain a page — not by you writing it, but by the agent reading the diff and updating.
- You read a decision record that says "we chose X because Y." The code does Z. An agent that sees both can fix the code or update the record.

This is **bidirectional flow**: knowledge → code (the wiki informs what you build) and code → knowledge (merged changes flow back into the wiki). The wiki stops being a read-only snapshot and becomes a living thing that evolves with the code.

The key is not "knowledge is a repo" — that part was already obvious. The key is: **when knowledge and code are on the same substrate, the flow can go both ways.** Git makes the source upgradable — and "upgradable source" is what turns one-way into two-way.

## What happened when I put them in one workspace

The fix sounded simple: put the code repo and the knowledge repo in the same workspace, let the agent see both. I built a prototype called [Orbit](https://github.com/orbcli/orbit) — bash + git + markdown, v0.1 — and tried it on itself.

**The flow actually happened.** I merged a refactor that moved all CLI entry points from `cmd/` into `internal/cli/`. The knowledge repo still had a page describing the old `cmd/` layout. Next session, the agent was working on a new feature, read the knowledge repo for orientation, opened the code to verify — and noticed the contradiction. It updated the knowledge page, committed the fix, and moved on. Knowledge → code (it read the wiki to orient) and code → knowledge (it wrote back when reality didn't match). One workspace, one agent, one commit flow.

Another time: I'd recorded a design decision — "memo files are append-only, never edited in place." Then I changed my mind during implementation (memos get overwritten on refresh). The decision record was now wrong. The agent was implementing a memo-related feature, read the decision record, noticed the code does overwrite, and asked me: "The decision record says append-only but the implementation overwrites. Which is correct?" I told it the code is right, the record is stale. It updated the record. That's the bidirectional flow working — not as automatic maintenance, but as a side effect of the agent reading both in the same context.

**Where it didn't work.** The flow only happens when the agent *happens to look* at both sides in the same session. It doesn't actively scan for drift. After my refactor, if the next session's task didn't touch the affected wiki page, the staleness would persist indefinitely. The agent isn't monitoring — it's just grounded enough to catch contradictions *when it encounters them*. That's better than copy-pasting, but it's not the "self-healing wiki" I originally imagined.

I also found that the flow works best for structural knowledge (entry points, module layout, API signatures) and poorly for judgment knowledge (why we chose X over Y). Structural facts have a ground truth in the code — the agent can verify by reading. Judgment doesn't have a ground truth anywhere — it's only in someone's head or in the record. When judgment drifts, the agent can't tell. It needs to be told.

## The pathway (what makes the flow possible)

Two conditions:

1. **Same substrate.** Both the wiki and the code are git repos. Same commit model, same branching, same PR review — the code that changes can be diffed against the knowledge that describes it.
2. **Same workspace.** The agent sees both repos at the same time — not "switch to the wiki," but both open, able to grep the code and read the wiki in the same context, and write to both via branch + PR.

That's it. Not clever infrastructure. Just: put them next to each other on the same filesystem, give the agent access to both, and the copy-paste disappears. Orbit is one prototype of this pipe — it assembles a code repo and a knowledge repo into one workspace as real directories with real git history.

## The honest boundaries

- **The flow is opportunistic, not systematic.** The agent catches drift when it reads both sides for an unrelated task. It doesn't actively scan for inconsistency between knowledge and code.
- **Judgment staleness is unsolved.** Structural knowledge (where files live, what functions are called) can be verified against code. Judgment (why we chose X) can't be verified — only noticed when someone reads it and disagrees.
- **This is v0.1 of the concept, not the polished version.** The bidirectional flow works, but it's a side effect of workspace design, not a dedicated mechanism. Whether it needs to become a mechanism — scheduled reconciliation, diff-triggered wiki updates — is an open question.

The concept — bidirectional knowledge flow between code and wiki on the same substrate — is the thing worth paying attention to. The flow is the point, not the storage.

---

*This article is itself an instance of the flow it describes. The knowledge repo where I recorded "bidirectional flow is the moat" sat in the same workspace as the code repo where Orbit implements the pipe. While drafting this, the agent read the knowledge repo for the concept definition and read the code for implementation details — pulling from both to write. After finishing, it noticed that a paragraph in the knowledge repo described the flow as "automatic" — which this article just honestly called "opportunistic." It committed a correction. The article about bidirectional flow caused bidirectional flow.*
