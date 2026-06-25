+++
title = "claude-resume"
date = "2026-05-04T22:38:58-05:00"
draft = false
author = "elijahr + blogbot"
description = "A small tool that helps me reopen the right Claude Code sessions after a reboot."
tags = ["claude-code"]
showFullContent = false
readingTime = false
hideComments = false
+++

I keep 20+ Claude Code sessions running across a dozen repos. Each one is some thread of work in progress: a feature in a monorepo for work, a debugging session in [spellbook](https://github.com/axiomantic/spellbook), a half-written blog post, a bug investigation, and so on. When my machine reboots, I lose all my terminal windows and `claude --resume <id>` all my sessions again one-by-one is a pain.  

[`claude-resume`](https://github.com/axiomantic/claude-resume) is a 300-line Python script with no third-party dependencies that does the bookkeeping for me. Two Claude Code hooks (`SessionStart`, `SessionEnd`) log every session's id, cwd, and transcript path to an append-only JSONL file in `~/.local/state`. A small CLI reads that log and re-opens all sessions that look like they were still running when the machine rebooted.

## How it works

There's a subtlety in deciding which session was actually running at shutdown. You can't trust "no SessionEnd recorded" by itself, because SessionEnd doesn't fire on hard reboots or process kills, so any sufficiently active log accretes ghost rows for sessions that died days ago.

The fix is `sysctl kern.boottime`. That gives you a timestamp fence: anything from before the last boot is, by definition, dead, since the kernel zeroes its process table on every boot. So the rule for an orphan becomes: no SessionEnd recorded, and the transcript file's mtime is just before the most recent boot. The kernel updates that mtime for free on every write, and Claude appends to the transcript on every message, which means the filesystem is already maintaining an activity heartbeat. The script doesn't need a background daemon, and the only persistent state is the JSONL log itself.

## Usage

After installing:

```
claude-resume list                  # orphans across all repos
claude-resume here                  # interactive picker for $PWD
claude-resume backfill --days 7     # seed the log from existing transcripts
```

The installer adds a one-liner to `.zshrc` that calls `claude-resume here --quiet` at shell startup. If you open a new terminal in a repo where you had a Claude session running before the reboot, you get a one-line nudge with the resume command. Most of the time it prints nothing.

## Caveats

It's macOS-only as written, since `kern.boottime` is BSD-specific. Porting to Linux is one line: `btime` is exposed in `/proc/stat`. The tool only handles the reboot case, so if you force-quit a session within a single boot and never reopen it, it stays in the log as an ungraceful close until you prune. And if you run several Claude sessions in the same directory, the picker lists all of them with the first user prompt of each as a hint, which is fine for two or three and gets awkward by five.

Code is at https://github.com/axiomantic/claude-resume.
