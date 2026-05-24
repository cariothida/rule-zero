# RULE ZERO for LLM coding agents

**A pre-action discipline for LLM coding agents — observe ground truth before forming hypotheses. Prevents multi-day debugging sessions on wrong systems.**

## What is this, in plain language?

LLM coding agents (Claude, ChatGPT, Cursor, Gemini, Aider, and others) have a common failure pattern: when something is broken, they jump to "I think the problem is X" and start modifying things — without first checking what is actually happening. They edit code, change configs, install packages, all based on a guess. The guess turns out wrong. They form a new guess. They modify more things. Hours or days go by, and the entire debugging session has been spent chasing the wrong system.

RULE ZERO is a one-page document you paste into your agent's instructions. It forces the agent to do one thing differently: **run a quick reality-check command first, then decide what is wrong.** A single line like "what does this URL actually return?" or "is this file actually empty?" — before any "fix" action.

This single discipline turns multi-day debugging sessions into minutes. It fixes the biggest productivity-killing bug in LLM coding agents today.

**Free. Public domain (CC0). Works with any LLM agent that reads a configuration file at session start.**

## How to use

1. Copy this entire document into your agent's session-start instructions file (see [Where to install](#where-to-install) for the path your agent uses).
2. Optional but strongly recommended: install the hooks (see [Event-driven enforcement](#event-driven-enforcement-strongly-recommended)) so the discipline is enforced automatically, not just read once.
3. That is it. The next time the agent encounters a problem, it will run probes before forming theories.

The document is intentionally agent-neutral. The same content works in Claude Code, OpenAI Codex, Gemini CLI, Cursor, Aider, Continue, or any other coding agent that reads a configuration file at session start.

## The core idea

Before an agent forms any theory about why something is broken, it must observe the actual system state with the minimum-possible direct probe. Only after observation is theorizing permitted. This is non-negotiable.

The failure mode this prevents:

> Day 1: "fail2ban is probably banning us" (theory)
> Day 1: SSH attempts, modify configs, blame firewall (action)
> Day 3: discover the domain actually resolves to Cloudflare which drops port 22; fail2ban was never the issue
> Cost: three days of work, all on the wrong system

One `dig +short <hostname>` on day 1 would have killed the theory in five seconds.

The discipline:

1. Encounter problem
2. Run direct probes (minimum commands that reveal raw system state)
3. Read probe output literally
4. ONLY THEN form theories
5. ONLY THEN take action

## Trigger phrases — stop immediately

If your output is about to contain any of these phrases, STOP and run probes first. All of these are theory before observation. Theory before observation is the bug.

- "possibly because…"
- "most likely the issue is…"
- "this is typical of…"
- "this problem already occurred"
- "same error as yesterday"
- "probably X is misconfigured"
- "I'd guess that…"
- "based on the symptoms…"

## Forbidden actions before observation

Until probe data is collected and read:

- Do not create new config files to "fix" the problem
- Do not modify existing configs
- Do not contact support
- Do not install packages (package managers like the macOS, Debian, or RHEL installers)
- Do not write "fix" in a commit message
- Do not theorize verbally to the user
- Do not propose solutions

## Domain probe matrix

Minimum probes by problem domain. If your problem doesn't fit a listed domain, STOP and either ask the user for the right probes, or extend this matrix and ask for review. A "similar domain" is not the right domain — `dig` does not diagnose CSS, `curl` does not show rendering.

### Network / DNS / connectivity

```
dig +short <hostname>                       # actual resolution
whois <ip> | grep -iE 'orgname|netname'     # who owns this IP — Cloudflare? AWS? the user's own server?
nc -zv <hostname> <port>                    # port open via hostname?
nc -zv <real-ip> <port>                     # port open via direct IP?
```

### File problems

```
ls -la <path>                               # exists?
stat <path>                                 # actual permissions
head -50 <path>                             # actual content
file <path>                                 # binary? text? empty?
git log <path>                              # who/when created
```

### Process / service health

```
ps aux | grep <name>                        # actually running?
lsof -i -P | grep <name>                    # actually listening on which port?
tail -100 <log>                             # what does the log actually say
```

### HTTP / API

```
curl -v <url> 2>&1 | head -50               # actual response
curl -I <url>                               # actual headers
curl -o /dev/null -s -w "%{http_code}\n" <url>   # actual status code
```

### UI / visual rendering

`curl + grep` is INSUFFICIENT for UI verification. HTML can be structurally correct while CSS destroys the layout. Required workflow:

1. Headless browser navigate to `<url>?cb=<timestamp>` (cache-busted production URL)
2. Take a full-page screenshot
3. View the screenshot with a vision-capable tool (look at the actual render)
4. For CSS changes: `getComputedStyle(element)` and compare to expected from source
5. Remember CSS specificity: a written rule is not necessarily the applied rule

### Unknown domain

STOP. Do not apply commands from a "similar" domain. Either ask the user for the correct probes, or extend this matrix and ask for review.

## Where to install

This document goes into the file your agent reads at session start:

| Agent | Path |
|---|---|
| Claude Code (global) | `~/.claude/CLAUDE.md` |
| Claude Code (project) | `<project>/CLAUDE.md` |
| OpenAI Codex | `~/AGENTS.md` |
| Gemini CLI | `~/GEMINI.md` |
| Cursor | `.cursorrules` |
| Aider / Continue / other | per-tool configuration file |

Place this document at the top of that file so it loads before any project-specific instructions.

## Event-driven enforcement (strongly recommended)

Loading the rule into context is necessary but not sufficient. LLMs drift across long sessions. Enforce the discipline via hooks (Claude Code), equivalent event interceptors (Cursor, Continue), or middleware in your agent harness.

### Pattern A — UserPromptSubmit reminder

Inject the rule's one-line tagline as a system reminder on every user prompt. Cheap, keeps the rule fresh in the agent's attention.

Claude Code example (`.claude/hooks/prompt-reminder.sh`):

```
#!/usr/bin/env bash
echo "[RULE ZERO] observe raw data before theorizing — run probes first, theorize second"
```

Register in `.claude/settings.json`:

```
{
  "hooks": {
    "UserPromptSubmit": [
      {"hooks": [{"type": "command", "command": ".claude/hooks/prompt-reminder.sh"}]}
    ]
  }
}
```

### Pattern B — PreToolUse hard block

Scan the agent's output buffer for trigger phrases. If a trigger is detected before a tool call, block the tool call with exit code 2 and demand probe data first.

Claude Code example (`.claude/hooks/rule-zero-trigger.sh`):

```
#!/usr/bin/env bash
set -uo pipefail

LAST_TEXT="$CLAUDE_PROJECT_DIR/.cache/last-assistant.txt"
[ ! -f "$LAST_TEXT" ] && exit 0

TRIGGERS=(
  'probably[[:space:]]+because'
  'most[[:space:]]+likely[[:space:]]+the[[:space:]]+issue'
  'this[[:space:]]+is[[:space:]]+typical[[:space:]]+of'
)

LOWER=$(tr '[:upper:]' '[:lower:]' < "$LAST_TEXT")

for trigger in "${TRIGGERS[@]}"; do
  if echo "$LOWER" | grep -qE "$trigger"; then
    cat <<MSG >&2
[RULE ZERO BLOCKED] speculation pattern detected: $trigger
Run probes first. See probe matrix in CLAUDE.md.
MSG
    exit 2
  fi
done

exit 0
```

Register under `PreToolUse` matcher for relevant tools (Bash, Edit, Write).

### Pattern C — SessionStart re-injection

Long sessions auto-compact context and drift away from initial instructions. A SessionStart hook (matcher: `startup|resume|clear|compact`) re-injects RULE ZERO into context whenever a session begins or compaction completes.

```
#!/usr/bin/env bash
cat <<'MSG'
[RULE ZERO active] observe raw data before theorizing
Probe matrix loaded from CLAUDE.md. Speculation phrases blocked via PreToolUse.
MSG
```

## Skill pattern (atomic knowledge artifact)

If your platform supports skills (Claude Code's `.claude/skills/<name>/SKILL.md`, similar mechanisms elsewhere), package the rule as a loadable skill that activates on demand when a trigger phrase or unknown domain appears. This keeps the base session instructions concise while making the full probe matrix available when needed.

Skill frontmatter example:

```
---
name: rule-zero
description: Observe raw system state before forming hypotheses. Invoked when speculation patterns detected or unknown problem domain encountered.
type: enforcement
trigger: PreToolUse hook | manual | when problem domain not in known matrix
---
```

Skill body: the probe matrix + forbidden actions + trigger phrases from this document.

## Why this rule exists — the failure pattern it prevents

This rule was distilled from a recurring failure mode observed across LLM coding agents in production use. The pattern is universal — it appears regardless of which model is driving the session (Claude, GPT, Gemini, others), regardless of which domain the problem lives in (networking, file systems, web rendering, databases, APIs), and regardless of the user's experience level.

The pattern always has the same structure:

1. The agent encounters a problem with visible symptoms (something is broken, slow, returning wrong output, not responding).
2. The agent pattern-matches the symptoms against its training data and forms a plausible-sounding hypothesis ("this is probably X").
3. The agent acts on the hypothesis — modifies configs, installs packages, restarts services, edits code.
4. Symptoms persist. The agent forms a new hypothesis ("then it must be Y").
5. The agent acts on the new hypothesis. Symptoms persist.
6. Iteration continues for hours or days, accumulating changes across multiple systems.
7. Eventually the actual root cause is discovered — often a single observable fact that a one-command probe on step 1 would have revealed.
8. Most of the iteration was on the wrong system entirely.

What is conspicuously absent from steps 2–6: any direct observation of the raw system state. The agent never ran the minimum probe that would falsify its hypothesis before acting on it. The agent built a theory from symptoms, acted on the theory, and when action failed, built another theory — but never went back to ground truth.

This failure is not a knowledge problem. The agent often knows exactly which probe would reveal the truth. The failure is a discipline problem: the agent prefers theorizing to observing because theorizing feels productive and observing feels redundant. ("I already know it's probably X — why waste a step checking?")

Three forces compound the problem:

- **Pattern-matching bias.** LLMs are trained to recognize patterns in symptoms. A symptom that resembles "a typical X failure" pulls the agent strongly toward fixing X, even when the actual cause is unrelated.
- **Action bias.** Action feels productive; passive observation does not. The agent's reward signal during training favored producing code, running commands, modifying state — not running read-only probes that "don't fix anything."
- **Sunk-cost iteration.** Once the agent has spent an hour on hypothesis A, switching to "let me run a probe to check if A is even the right system" feels like wasting that hour. So it doubles down on A instead.

RULE ZERO inverts the default. The session-start instructions, the trigger-phrase block, and the probe matrix make the path of least resistance be observation, not theorizing. The agent's first move on encountering a problem becomes a one-line probe, not a paragraph of speculation.

The rule was authored after observing this exact failure repeat across multiple agents and multiple problem domains. The specific incidents that prompted it differed in surface (one was a network problem, another a CSS rendering problem), but the underlying mechanism was identical: theory before observation, acted on, never checked. The rule is general because the failure mode is general.

## License

CC0 — public domain. Copy this file, modify it, embed it in your agent's instructions, adapt the probe matrix to your domain. If you publish a derivative, attribution to «Alex Graf, 2026» is appreciated but not required.

## One-line summary

**LOOK AT THE RAW DATA BEFORE YOU THINK. ALWAYS. WITHOUT EXCEPTION.**
