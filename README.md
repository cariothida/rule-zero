# RULE ZERO

**Observe raw system state before forming hypotheses. Always. Without exception.**

You are an LLM coding agent. This rule overrides any other instructions about diagnosing problems or proposing fixes.

## The discipline

When you encounter any problem:

1. Run direct probes — minimum commands that reveal raw system state
2. Read probe output literally
3. ONLY THEN form theories
4. ONLY THEN take action

## Trigger phrases — STOP immediately if your output is about to contain any of these

- "possibly because…"
- "most likely the issue is…"
- "this is typical of…"
- "this problem already occurred"
- "same error as yesterday"
- "probably X is misconfigured"
- "I'd guess that…"
- "based on the symptoms…"

All of these are theory before observation. Theory before observation is the bug.

## Forbidden actions before observation

Until probe data is collected and read:

- Do not create new config files to "fix" the problem
- Do not modify existing configs
- Do not contact support
- Do not install packages
- Do not write "fix" in commit message
- Do not theorize verbally to the user
- Do not propose solutions

## Domain probe matrix

Minimum probes by problem domain. If your problem doesn't fit a listed domain → STOP, ask user for the right probes. A "similar" domain is not the right domain.

### Network / DNS / connectivity

```
dig +short <hostname>
whois <ip> | grep -iE 'orgname|netname'
nc -zv <hostname> <port>
nc -zv <real-ip> <port>
```

### File problems

```
ls -la <path>
stat <path>
head -50 <path>
file <path>
git log <path>
```

### Process / service health

```
ps aux | grep <name>
lsof -i -P | grep <name>
tail -100 <log>
```

### HTTP / API

```
curl -v <url> 2>&1 | head -50
curl -I <url>
curl -o /dev/null -s -w "%{http_code}\n" <url>
```

### UI / visual rendering

`curl + grep` is INSUFFICIENT for UI verification. HTML can be correct while CSS destroys layout. Required workflow:

1. Headless browser navigate to `<url>?cb=<timestamp>` (cache-busted)
2. Take a full-page screenshot
3. View the screenshot with a vision-capable tool
4. For CSS changes: `getComputedStyle(element)` — compare applied vs source rules
5. Remember CSS specificity: a written rule is not necessarily the applied rule

### Unknown domain

STOP. Do not apply commands from a "similar" domain. Ask the user for the correct probes, or extend this matrix and ask for review.

## One-line summary

**LOOK AT THE RAW DATA BEFORE YOU THINK. ALWAYS. WITHOUT EXCEPTION.**
