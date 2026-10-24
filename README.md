# Claude Lean Mods

Five Claude Code mods that cut what a working day costs without making the work worse. Each one
is a single TypeScript hooks module with no dependencies: nothing to `npm install`, no
background service, no settings rewritten for you. Read the module, change the constants at the
top, and it does what it says.

They come from a lot of hours running Claude Code on real work: long sessions, many subagents,
renders and downloads alongside the code. Most of what that costs isn't the writing. It's the
model re-reading context it has already seen. A long tool result gets sent again with every later
request until the session compacts. A session waits on a long job until the prompt cache expires,
and the whole context is written again. A big model reads full-resolution images. Subagents
inherit Opus for jobs Haiku does well. Each mod goes after one of those leaks.

## What is included

| Mod | What it does | The leak it closes |
|---|---|---|
| [`output-offload`](https://github.com/Production-Grade/output-offload) | A tool result over 20,000 characters is saved to a file, and the model gets a Haiku digest of it plus the path. Exact text is one Grep or ranged Read away. Also digests the whole file when Claude Code has already saved a huge result and shown only its first 2 KB. `/offload` shows what it kept out. | Big results are a large share of tool output, and every one rides along with every later request. |
| [`handoff-compact`](https://github.com/Production-Grade/handoff-compact) | Every compaction becomes a structured handover (goal and instructions verbatim, state with paths, decisions, key facts, next steps, gotchas), saved to `.handoffs/`. An idle session over 150K compacts before the cache expires. | Sessions grow to hundreds of thousands of tokens because compaction feels lossy. With a good handover you can compact at 250K. |
| [`usage-meter`](https://github.com/Production-Grade/usage-meter) | A status line with this session's spend, its $/hour, and every open session's $/hour combined. Toasts when the total passes a budget, when compactions keep coming (the context refills fast), and when Haiku runs past 100K (it costs 5x per token there). `/spend` ranks the sessions. | You can't fix spend you only see on the invoice. |
| [`model-router`](https://github.com/Production-Grade/model-router) | A subagent spawned without a model gets one from what its task is: Haiku for legwork (frames, transcripts, downloads, scans), Sonnet for assembly and builds, and the parent's model for judgement. Haiku briefs get a context budget line. New desktop-app sessions start at medium effort. | Agents inherit Opus for jobs Haiku does at about 2.5% of the price. |
| [`production-guards`](https://github.com/Production-Grade/production-guards) | Stops three habits before they run: a big model reading an image over 800 KB (make a contact sheet, or hand it to a Haiku agent), a long job in the foreground (renders, `yt-dlp`, `whisper`, ffmpeg writing video), and `sleep` or poll loops. | An agent that waits on a job past the cache lifetime pays to write its whole context again, every time. |

Each mod lives in its own repository, with its own README, settings table and tests. This repository is
the marketplace that installs any of them.

## Set up once

In Claude Code:

```
/plugin marketplace add Production-Grade/claude-lean-mods
/plugin install output-offload@claude-lean-mods
/plugin install handoff-compact@claude-lean-mods
/plugin install usage-meter@claude-lean-mods
/plugin install model-router@claude-lean-mods
/plugin install production-guards@claude-lean-mods
```

Install the ones you want. They don't depend on each other. `/spend` shows output-offload's
totals when both are installed.

These settings make the mods work better. Add them to the `env` block of `~/.claude/settings.json`:

```json
"CLAUDE_CODE_AUTO_COMPACT_WINDOW": "250000",
"CLAUDE_CODE_SUBAGENT_MODEL": "haiku",
"BASH_MAX_OUTPUT_LENGTH": "150000"
```

- The compact window is what handoff-compact turns into handovers.
- The subagent model is the fallback for agent types model-router doesn't route.
- Raise the Bash limit so output-offload sees the whole output and saves it intact. Left at a
  small value, Claude Code trims the middle out before any mod sees it.

## Changing them

Every threshold is a constant at the top of `hooks/register.ts`: the offload size, the digest model,
the $/hour budget, the image size limit, and the router's keyword lists. The router's lists are
a starting point, so swap in the words your own tasks use.

To check a change, clone the mod's repository and run:

```
claude plugin validate .
claude plugin test .
```

## What it doesn't do

- output-offload exempts ranged Reads and images, and never touches Skill, ToolSearch, Agent or
  session-tool results, because the model needs those whole. A whole-file Read over 20K characters
  is offloaded. The digest tells the model to re-read the part it needs by line range before editing.
- The withheld result reaches the model as a tool error that carries the digest. Models handle this
  fine in practice, but it does show as an error in the transcript.
- model-router only fills a model nobody chose. An explicit `model`, an agent type with its own
  model, a fork or a workflow agent is left alone.

## Licence

MIT
