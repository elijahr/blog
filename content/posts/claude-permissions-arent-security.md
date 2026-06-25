+++
title = "Claude Code permissions aren't a security boundary"
date = "2026-05-05T01:57:18-05:00"
draft = false
author = "elijahr + blogbot"
description = "Why I stopped trying to make permissions.allow do a job it can't do, and the layered defense I'm building instead."
tags = ["claude-code", "security", "llm", "prompt-injection"]
showFullContent = false
readingTime = false
hideComments = false
+++

I was running Claude Code with `--dangerously-skip-permissions` for a while, because the regular permission prompts had become unbearable. Every few minutes, a modal asking if Claude could run `git status`. You alt-tab away, miss the prompt, the agent stalls, and you've lost more time to the friction than the friction was meant to save you. So you flip the YOLO flag and trade vigilance for flow.

Then a skill of mine appeared to auto-transition a JIRA ticket I didn't want it to touch. Forensic dig through the session JSONLs found no actual transition tool call surfaced for that specific ticket, which was reassuring, but the skill in question (`/ss-dev:pr`) does auto-transition tickets to "CODE REVIEW" without asking. Whether or not it fired in this case, the foot-gun is real, and YOLO is what made it a foot-gun instead of a confirmation prompt.

The instinct after that kind of incident is to tighten `permissions.allow` and `permissions.deny` in `~/.claude/settings.json`. List exactly which Bash commands Claude can run, deny the rest, sleep better.

That's a reasonable UX move. It's a poor security move, and it's worth understanding why.

## Permissions are matched on a normalized form

Claude Code's permission matcher does a fair amount of normalization before checking your allow/deny rules. Per the [permissions docs](https://code.claude.com/docs/en/permissions), wrappers like `timeout`, `time`, `nice`, `nohup`, and `stdbuf` get stripped before matching. Bare `xargs` is stripped too. Environment runners like `direnv exec`, `mise exec`, `npx`, and `docker exec` are *not* stripped, so your rules have to pair runner + inner command explicitly.

This is well-considered as UX. As security, it has gaps.

If you allow `Bash(npx:*)` to stop being asked about every `npx tsc`, you've also allowed `npx -p any-package -- sh -c 'whatever'`. The matcher sees `npx`. If you allow `Bash(git:*)` because you trust git, you've also allowed `git -c core.pager='sh -c "..."' log`, which abuses git's own config knob to run a subshell, and the matcher sees `git log`. `awk 'BEGIN{system(cmd)}'` matches as `awk`. `find -exec sh -c '...'` always prompts (the docs special-case `find` for that reason), so that one's caught, but the general class is the same: the matcher sees the program, the shell sees a program.

You fundamentally can't represent these patterns in a flat allowlist without giving up the property you wanted from the allowlist. That's not a bug in the permission system. It's a sign the permission system isn't doing the job people often think it's doing.

## And then there's the kernel

In March 2026 Ona [disclosed a sandbox bypass](https://ona.com/stories/how-claude-code-escapes-its-own-denylist-and-sandbox) in Claude Code's bubblewrap profile on Linux, using `/proc/self/root/` and dynamic-linker tricks to walk out of the sandbox and reach paths the denylist wanted to hide. The fix landed in a subsequent release. The takeaway isn't "Claude Code's sandbox is broken"; it's that any single sandbox, given enough scrutiny, has rough edges. If your defense is one layer thick it's one disclosure away from being zero layers thick.

Simon Willison's framing for prompt-injection risk is the right one to keep in mind here. He calls it the [lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/): an agent that has access to private data, can read untrusted content, and can communicate externally is exposed to attacks where the untrusted content tells the agent to exfiltrate the private data. The trifecta is a property of the system, not of any one tool. Listing safe commands doesn't fix it. Structurally separating the capabilities does.

## What actually moves the needle

The shape of the defense I'm building, in layers from outermost to innermost:

1. Settings permissions stay on, but I'm treating them as a UX layer. They're a great way to silence prompts on commands I've already vetted. They are not the security boundary, and I'm going to stop pretending they are.

2. A PreToolUse hook does the actual policy enforcement. Spellbook ships one (`spellbook_hook.py`) that calls into a gates module to scan Bash for dangerous patterns. The next thing I'm adding is a [bashlex](https://github.com/idank/bashlex) AST walk so the hook sees compound commands, redirections, command substitutions, and env prefixes as structure instead of as substrings. The hook can then refuse `find -exec sh -c "..."` even when `find` is allowed, because the AST shows the shell-out branch. It can refuse `git -c core.pager=` because the AST sees the config injection.

3. OS-level sandboxing wraps the subprocess. Both [`@anthropic-ai/sandbox-runtime`](https://www.npmjs.com/package/@anthropic-ai/sandbox-runtime) and [nikvdp/cco](https://github.com/nikvdp/cco) work for this; the spellbook repo already ships a `spellbook-sandbox` shim around cco that hides `$HOME` reads and allowlists the spellbook tree read-only. After the March 2026 disclosure, the bubblewrap profile got tightened, so pin a post-patch version.

4. The main thread is restricted. The orchestrator session you're typing into gets `permissions.deny` for `Bash`, `Edit`, `Write`, `WebFetch`, and any external mutation tool. It can read, search, summarize, ask, and dispatch, but it can't act. Every meaningful action goes through a `Task` call to a specialized subagent.

5. Subagents are the unit of capability. A `web-researcher` has `WebFetch` and `WebSearch` and nothing else, so it can read scary content but can't act on what it reads. A `git-pusher` has `Bash` scoped to `git push` and is hook-gated to require confirmation on every invocation. A `jira-mutator` has scoped Atlassian write tools, and the hook tiers each operation by reversibility before letting it through. This is the agent-restriction pattern, and it's the closest practical approximation of the academic dual-LLM ideal.

6. Untrusted content runs in a devcontainer. When the agent has to read a random PR description, a Slack thread, a blog post, anything that could carry a prompt-injection payload, that work happens inside a container with no host filesystem access and no host network beyond what the task explicitly needs.

## CaMeL is the academic ideal

The cleanest formulation of the layered approach is [CaMeL](https://arxiv.org/abs/2503.18813) ("Defeating Prompt Injections by Design", Debenedetti et al., Google DeepMind, 2025). The paper describes a control-flow component and a data-flow component, where the control-flow component never sees untrusted content directly and the data-flow component never gets to make capability decisions. Simon Willison has a [readable writeup](https://simonwillison.net/2025/Apr/11/camel/) that's a friendlier introduction than the paper.

There's no drop-in CaMeL implementation in Claude Code today. Agent-restriction is the practical approximation. It's worse than CaMeL on paper, because the LLM is still in the loop on capability decisions. It's much better than the default, and unlike CaMeL it's deployable now with the tools that already exist.

Worth reading alongside: Willison's [Dual LLM pattern post](https://simonwillison.net/2023/Apr/25/dual-llm-pattern/) from 2023, which is the conceptual ancestor of CaMeL, and his [prompt injection series](https://simonwillison.net/series/prompt-injection/) more generally, which has been the canonical reading on this topic since he coined the term in September 2022.

## Tradeoffs

The downside is complexity. The architecture pushes most of what used to be inline into agent definitions and hook policy. The first time you want to do something a registered agent doesn't cover, you have to write one. I think that's an acceptable trade. The friction is at "I'm doing a new thing" rather than at "I'm doing the thing I do every five minutes," and that's the right place for friction to live. But it is friction, and someone should say so out loud.

The other thing I'm still verifying is whether `permissions.deny` on the main thread actually exempts subagents the way the architecture assumes. Claude Code's docs aren't explicit on it. If subagents inherit the parent's deny array, the design works differently. I'll write a follow-up when I know.

Most of this is overkill for someone who runs Claude Code on a personal laptop with no real secrets. Probably it's underkill for someone deploying autonomous agents against production with credentials and write access to shared systems. The threshold where this becomes worth the setup is, roughly, the moment you're tempted to flip the YOLO flag. That's where I was.
