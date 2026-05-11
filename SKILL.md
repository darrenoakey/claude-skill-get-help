---
name: get-help
description: When stuck on a hard, ambiguous, or research-grade problem — especially one where guessing risks wasted iterations — consult the OTHER two locally-installed AI CLIs (and the web, especially Reddit) in parallel for independent second opinions. Pick peers based on the host you're running in: from Claude → consult `codex` + `gemini`; from Codex → consult `claude` + `gemini`; from Gemini → consult `claude` + `codex`. Use BEFORE writing speculative fixes. The goal is a definitive answer grounded in real prior art, not another guess.
---

# get-help — multi-model parallel consultation

## When to use

Reach for this skill when:

- You've already tried the obvious fix and it didn't work. Iterating blindly burns time and trust.
- The problem is iOS/Metal/MLX/audio/codec/protocol-level and likely has documented upstream issues you haven't found yet.
- The user says something like *"stop guessing"*, *"this has to be definitive"*, *"go and research"*, *"I want more research, less testing"*, or you sense they're losing patience with churn.
- The bug pattern looks specific enough that someone else has hit it (e.g. a precise frequency, a specific tensor shape, a chip-generation symptom, a numeric signature like NaN-at-step-N).
- You're about to do something irreversible or expensive (rewrite an architecture, ship a workaround) and want a sanity check.

Do NOT use for trivial bugs you can fix faster than a CLI round-trip — this is for hard cases.

## Pick your peers (don't consult yourself)

**First, detect which CLI you're running in**, then consult the OTHER two. Asking yourself the same question twice produces no new signal.

| You are running in | Consult these peers |
|---|---|
| Claude Code (this CLI) | `codex` + `gemini` |
| Codex CLI | `claude` + `gemini` |
| Gemini CLI | `claude` + `codex` |

Cheap detection: check `$CLAUDECODE` (set inside Claude Code), `$CODEX_*` env vars, or the originating process. If genuinely unsure, default to assuming you're in Claude Code and consulting `codex` + `gemini`.

## Available tools on this machine

Verify with `which claude gemini codex` before calling. All expected at `/opt/homebrew/bin/` (or `~/.claude/local/claude` for the Claude CLI).

- **`claude -p "<prompt>"`** — Anthropic Claude CLI (one-shot, non-interactive). Strong at synthesis and at refusing to speculate when grounding is thin. Use as a peer when you are NOT already running inside Claude Code. Inside Claude Code, skip this — you'd just be talking to yourself. Heads up: `claude --print` from *inside* a Claude Code session needs `env.pop("CLAUDECODE", None)` or it fails; the skill caller is responsible for stripping it.
- **`gemini -p "<prompt>"`** — Google Gemini CLI. Verbose, sometimes speculative, occasionally inventive in unhelpful ways. Treat its claims as hypotheses to verify, not facts. Quality is best on broad architectural questions and code-base scans.
- **`codex exec --skip-git-repo-check - < /tmp/prompt.txt`** — OpenAI Codex CLI. Reads stdin. Heavier emphasis on grounding in real upstream sources (GitHub issues, PRs, official docs). Empirically more reliable when the question is *"is there a known bug?"*. Always pass `--skip-git-repo-check` unless you're inside a trusted git repo.
- **`WebSearch` / `WebFetch`** — built-in. Use for direct hits on GitHub issues, Apple developer forum, Stack Overflow. **Especially search Reddit** — append `site:reddit.com` or `reddit` to at least one of your queries. Reddit threads (especially r/programming, r/MachineLearning, r/iOSProgramming, hardware-specific subs, and game-engine subs) frequently contain the *only* discussion of niche bugs that never made it to an official issue tracker.
- **`gh search issues`, `gh issue view`, `gh pr view`** — for repo-specific bug archaeology. Almost always faster than web search once you know which repo.

## The pattern

**1. Write ONE shared prompt to disk first.** All three (gemini / codex / WebSearch) get the same input. This makes their disagreements interpretable.

```bash
cat > /tmp/help_prompt.txt <<'EOF'
[Concrete context: stack, OS version, hardware]
[Precise symptom — measurements, not adjectives]
[What you've already ruled out — so they don't suggest those again]
[Specific question(s) — not "what should I do" but "what causes X?"]
[What kind of answer you want — pointer to prior art, falsifiable hypothesis, specific upstream issue]
EOF
```

The prompt quality dominates the response quality. Spend real time on this. The single most important sentence is usually *"I do NOT want 'try X' advice — I want a root cause grounded in documented prior art."*

**2. Fire all calls in a single message (parallel tool calls), not sequentially.** This is critical — round-trips are slow. Send the same prompt to your TWO peer CLIs (see the host-detection table above) plus two web searches. The pattern, assuming you're running in Claude Code (peers = codex + gemini):

```
[parallel block]
  WebSearch  query="<symptom> site:reddit.com"   (Reddit-targeted — do this one always)
  WebSearch  query="..."                          (second, narrower angle on GitHub/SO/forums)
  Bash       gemini -p "$(cat /tmp/help_prompt.txt)"
  Bash       codex exec --skip-git-repo-check - < /tmp/help_prompt.txt
[/parallel block]
```

If you're running in Codex, swap the `codex` call for `claude -p "$(cat /tmp/help_prompt.txt)"`. If you're running in Gemini, swap the `gemini` call for `claude -p ...`. The rule is simple: never call the CLI you're already inside.

Codex regularly hits rate limits — wrap it in a retry loop or be ready to re-issue. If it fails entirely, note the failure and proceed with the other sources.

**3. Synthesize, don't relay.** When the responses come back:

- **Citation-grounded peer first** (usually Codex, or Claude when run from a non-Claude host) — read its citations. Real GitHub PR/issue numbers? Real Apple TN references? Those are anchorable claims you can verify.
- **Gemini second** — read past its confident-sounding framing. Look for specific, falsifiable predictions. Discard the rest.
- **WebSearch third — and read the Reddit hits carefully.** Reddit threads often contain the lived-experience workaround that no AI cited and no official issue tracker recorded. Check for any source neither peer mentioned.
- Note where they AGREE (that's your high-confidence direction) and where they DISAGREE (that's where YOU need to make the call).
- Verify the top 1–2 cited sources directly with `gh` or `WebFetch` before acting. AIs hallucinate issue numbers regularly.

**4. Report findings before acting.** Show the user the actual cited sources. Quote the relevant text. Then propose the specific change. The user should never have to wonder whether you actually read the linked issue.

## What good output looks like

> *"Codex + Gemini both pointed at `mlx-swift#344` — `ConvTransposed1d` produces wrong output on iOS for T > 8000. Verified via `gh issue view`: documented FAIL list includes T=12,640 and T=50,560 (matches our chunk sizes). Fixed in mlx 3092, released in mlx-swift 0.30.6. We're pinned to 0.30.2 in `Package.resolved`. Bumping the pin is a documented one-line fix; PR #29 in `mlalma/kokoro-ios` is the same change against the same dependency. Want me to apply it?"*

That's a complete handoff: source, verification, change, ask. No guessing.

## What bad output looks like

> *"Gemini suggests it could be an MLX kernel bug. Codex thinks it might be ConvTransposed1d. I'll try increasing the LayerNorm epsilon to see if that helps."*

This is what you wrote BEFORE consulting them, dressed up with their names. Two failure modes here: synthesizing nothing, and proposing a speculative change anyway.

## Anti-patterns

- **Calling them sequentially.** Always one message with all calls.
- **Asking different questions.** Same prompt to all three; differences are the signal.
- **Trusting citations without verification.** Codex once mentioned `mlx-swift#344` — that issue is real. Gemini once mentioned `mlx-swift#354` for an unrelated fix — that PR is also real but for a different bug. Always `gh issue view` / `gh pr view` to confirm.
- **Pasting their full reply to the user.** The user wants the *answer*, not a transcript of two LLMs talking. Synthesize.
- **Using this for trivial bugs.** A round-trip costs ~30–60s. If you can fix it in 30s, just fix it.
- **Asking "what should I do?"** Ask "what causes X?" or "is there a known issue Y?". Action questions get speculation; diagnostic questions get citations.

## Prompt-writing checklist

Before sending, your prompt should contain:

- [ ] Stack/runtime (e.g. "iOS 26.5, iPhone 17 Pro Max, MLX-Swift 0.30.2")
- [ ] Exact symptom (measurements, frequencies, byte counts, error messages — not adjectives)
- [ ] Determinism statement (deterministic? flaky? environment-specific?)
- [ ] What you ruled out (list the obvious wrong directions so they skip them)
- [ ] One concrete question, not three vague ones
- [ ] Requested output form (cited prior art, falsifiable hypothesis, etc.)
- [ ] Explicit "no speculative `try X` advice" guardrail

If you can't fill these in, you don't know the problem well enough yet — keep diagnosing locally first.
