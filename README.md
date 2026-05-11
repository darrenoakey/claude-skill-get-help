![](banner.jpg)

# get-help

A skill for Claude Code (and other AI CLIs) that enables multi-model parallel consultation when you're stuck on a hard, ambiguous, or research-grade problem. Instead of guessing and iterating blindly, `get-help` queries multiple independent AI models and web sources simultaneously to produce a definitive, citation-grounded answer.

---

## Purpose

When debugging complex issues — especially ones involving niche platform bugs, upstream library issues, or domain-specific behaviour — guessing wastes time and trust. `get-help` solves this by:

- Consulting two peer AI CLIs **in parallel** for independent second opinions
- Running targeted web searches (including Reddit) at the same time
- Synthesizing the results into a single, citation-grounded recommendation

The goal is a real answer backed by documented prior art, not another guess.

---

## When to Use It

Use `get-help` when:

- You've already tried the obvious fix and it didn't work
- The problem is low-level (iOS, Metal, MLX, audio, codec, protocol) and likely has upstream precedent
- The user is losing patience with speculative fixes ("stop guessing", "this has to be definitive", "go and research")
- The bug pattern is specific enough that someone else has hit it (e.g. a precise frequency, a specific tensor shape, a numeric signature like NaN-at-step-N)
- You're about to do something irreversible or expensive and want a sanity check

**Do not use** for trivial bugs you can fix faster than a CLI round-trip.

---

## Installation

No separate installation is required. This is a Claude Code skill that is loaded automatically when present in your project or global skills directory.

The skill relies on the following peer CLI tools being available on your machine. Verify with:

```bash
which claude gemini codex
```

Expected locations: `/opt/homebrew/bin/` (or `~/.claude/local/claude` for the Claude CLI).

Install any missing peers via their respective setup instructions before using this skill.

---

## How to Use

### Step 1 — Trigger the skill

Ask Claude Code to consult peers on your hard problem. You can invoke it explicitly or Claude Code will apply it automatically when the situation warrants it.

Example prompts that trigger this skill:

```
This is still broken. Stop guessing and go research whether there's a known upstream issue.
```

```
I need a definitive answer on why CoreAudio drops frames at exactly 44100 Hz on M3. Go consult your peers.
```

```
I want more research, less testing. Find documented prior art for this NaN-at-step-47 pattern in MLX.
```

### Step 2 — The skill writes a shared prompt

Claude Code writes one concrete, well-formed prompt to `/tmp/help_prompt.txt` covering:

- Stack/runtime details (OS version, hardware, library versions)
- Exact symptom (measurements, error messages — not adjectives)
- What has already been ruled out
- A single, specific diagnostic question
- The requested output form (cited prior art, falsifiable hypothesis, etc.)

### Step 3 — Parallel consultation fires

All peer calls go out in a single parallel block — no sequential round-trips. For a Claude Code session this looks like:

```
[parallel]
  WebSearch  → "<symptom> site:reddit.com"
  WebSearch  → "<symptom> site:github.com OR stackoverflow.com"
  Bash       → gemini -p "$(cat /tmp/help_prompt.txt)"
  Bash       → codex exec --skip-git-repo-check - < /tmp/help_prompt.txt
[/parallel]
```

### Step 4 — Synthesized findings are reported

Claude Code synthesizes the responses and reports back with:

- The specific source(s) that grounded the answer (GitHub issue numbers, Reddit threads, Apple TN references)
- Direct quotes from those sources
- A concrete proposed change or next action
- A clear ask before taking any irreversible action

---

## Examples

### Example 1 — Known upstream library bug

**You say:**
```
Audio output is corrupted at T > 8000 on iPhone 15 Pro. I've ruled out sample rate mismatch and buffer sizing. Go research this.
```

**get-help returns:**
```
Codex + Gemini both pointed at mlx-swift#344 — ConvTransposed1d produces wrong
output on iOS for T > 8000. Verified via `gh issue view`: documented FAIL list
includes T=12,640 and T=50,560 (matches our chunk sizes). Fixed in mlx 3092,
released in mlx-swift 0.30.6. We're pinned to 0.30.2 in Package.resolved.
Bumping the pin is a documented one-line fix; PR #29 in mlalma/kokoro-ios is the
same change against the same dependency. Want me to apply it?
```

### Example 2 — Niche platform behaviour with Reddit prior art

**You say:**
```
NaN appears at exactly step 47 during fine-tuning on M2 Ultra. Deterministic. Not a learning-rate issue. Research this before we touch anything.
```

**get-help returns:**
```
Reddit thread r/MachineLearning (6 months ago) + a Gemini citation both describe
a deterministic NaN at step 47–52 tied to the Metal shader compiler on M2 when
accumulating bf16 gradients past a certain tensor rank. The Reddit OP confirmed
disabling Metal Performance Shaders fallback resolved it. Codex linked
apple/ml-stable-diffusion#501 which documents the same symptom. No official fix
yet. Proposed workaround: force fp32 accumulation for the affected layer. Want me
to add that flag?
```

### Example 3 — Architecture sanity check before a rewrite

**You say:**
```
I'm about to replace the streaming decoder with a batch approach to fix latency. Get a second opinion first.
```

**get-help returns:**
```
Gemini and Codex agree: the latency issue is not in the decoder architecture —
both cite a known AVAudioEngine scheduling jitter on iOS 17.4 (Apple Developer
Forums thread #1109823, April 2024). Switching to batch decoding would add 200–400ms
of buffering and likely make latency worse. The documented fix is pinning the
audio session IO buffer duration to 256 frames. Strongly recommend against the
rewrite. Want me to apply the IO buffer pin instead?
```

---

## Peer Selection Reference

The skill automatically picks the right peers based on which CLI host it's running inside:

| Running in | Peers consulted |
|---|---|
| Claude Code | `codex` + `gemini` |
| Codex CLI | `claude` + `gemini` |
| Gemini CLI | `claude` + `codex` |

This ensures you never consult the same model you're already running in — asking yourself twice produces no new signal.