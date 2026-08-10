+++
title = "Claude Code permissions: parse the command, don't match the string"
date = 2026-08-10T15:24:13+08:00
description = "Claude Code's permission allowlist matches command strings by prefix, and a shell command isn't a string. Four months of running a PreToolUse hook that parses commands instead: the mechanics, the numbers from its own log, what it can't do — and the hole in it that reviewing this very post uncovered."
+++

Anthropic made auto mode the default in Claude Code, and the
[Hacker News thread about it](https://news.ycombinator.com/item?id=49239021)
is full of a complaint I recognise:

> These models generate the most unreadable bash commands I've ever seen.
> Utilising every single option available and piping the result through
> multiple layers of regex and junk.
>
> The mental load of having to actually review these commands beyond the most
> surface level glance is too much. — SchemaLoad

And its companion:

> Not just that, the commands also have often slight variations in each new
> session. They still do  the same, but the variations are enough so it isn't
> matched by the allowlist any more. — xg15

Anthropic's answer is an LLM classifier that vets commands before they run.
Their announcement says users approve 97% of permission prompts, and that in
a paid study human testers caught a planted dangerous command 13.6% of the
time while the classifier blocked 89% of them. I believe all of that. A
prompt that is approved 97% of the time carries almost no information, and
nobody carefully reads the fifteenth `find`-with-eight-flags of the
afternoon.

But before reaching for a second model to judge the first model's shell
commands, it's worth asking why the prompts are so noisy in the first place.
I think the built-in allowlist has the wrong shape: it treats a shell command
as a *string*, matched by prefix. And a shell command isn't a string, it's a
program.

String matching fails in both directions at once. It's too strict: `git diff`
approved once doesn't cover `nice git diff`, and any pipe or `&&` glues two
commands into a fresh unique string that no allowlist entry will ever match
again — that's xg15's complaint. And it's too loose: the day you get tired
and approve `git *` wholesale, you've approved `git push --force` along with
`git log`.

## First attempt: ban the shell tricks

My first fix worked on the model's side of the problem. I put a rule in my
global `CLAUDE.md`: no `&&`, no `;`, no pipes — one command per Bash tool
call, temp files instead of pipelines, no command substitution. The reason
was exactly the allowlist mechanics above: separate plain calls reuse
existing approvals; anything chained never matches.

It half worked: the commands got shorter and more readable, and the
allowlist matched more often. But it never worked completely, because a
`CLAUDE.md` rule is advisory: the model weighs it against its own judgement
and writes a pipeline anyway when a pipeline seems obviously right. And it
was backwards — I was constraining perfectly good shell to fit a matcher
that couldn't read it, and paying for that with clunkier commands and more
tool-call round trips. The instruction was a prose patch for a structural
problem. I've since deleted it, because the real fix makes it unnecessary.

## The fix: a hook that parses

Claude Code has a `PreToolUse` hook: a script that sees every tool call
before the permission system does, and can answer "allow", "deny", or stay
silent and let the normal prompt happen. It gets the tool call as JSON on
stdin and answers with JSON on stdout — the whole mechanism is documented
in the [hooks reference](https://code.claude.com/docs/en/hooks), and it's
one of Claude Code's least-advertised features. Mine is a single Python
file, about 900 lines, standard library only, no second LLM.

The core decision is to *parse* the command instead of matching it:

1. **Tokenise** with `shlex` in punctuation mode, so `git diff; echo hi`
   splits into tokens rather than producing an unsplittable `diff;`. If
   shlex can't parse it, fall through to the human.

2. **Split at shell operators** (`&&`, `||`, `;`, `|`, `&`) and classify
   every sub-command independently. The whole call is auto-allowed only if
   *every* part is. A pipeline of three safe commands is safe; a pipeline
   with one unknown command in the middle goes to the prompt. Chaining
   stops defeating the allowlist, because nothing is matched on the
   combined string.

3. **Peel wrappers.** `nice`, `ionice`, `env VAR=x`, `timeout 30` — these
   wrap the real command, so the classifier skips over them (consuming
   their flags) and judges what's underneath. `nice make` is `make`.

4. **Judge programs, not prefixes.** Tools I've decided to trust regardless
   of arguments (`ls`, `grep`, `wc`, `stat`, …) are always allowed.
   Programs whose safety depends on
   their arguments get their own small checker. The git one knows that
   `git commit` is fine but `git commit --amend` rewrites history, that
   `git add file.c` is fine but `git add .` violates a house rule, that
   `push`, `reset` and `rebase` always go to the human:

   ```python
   if subcmd in GIT_DANGEROUS_SUBCOMMANDS:   # push, reset, rebase
       return "ask"
   if subcmd == "commit":
       return "ask" if "--amend" in args else "allow"
   if subcmd == "add":
       if "--all" in args or "-A" in args:
           return "ask"
   ```

   No string-prefix allowlist can express "commit yes, amend no". A parser
   expresses it in two lines.

5. **Recurse into substitutions.** `echo $(rm -rf /)` runs `rm -rf /`,
   whatever `echo` then does with the result. So the inner command of
   every `$(...)`, backtick and `<(...)` outside single quotes is
   classified like any other command, recursively. Heredoc bodies with
   quoted delimiters (`<<'EOF'`) are literal, so they're stripped first —
   which keeps the everyday `git commit -m "$(cat <<'EOF' ...` idiom
   prompt-free.

6. **Fail towards the human.** `sudo` always asks. `kill` always asks.
   Anything unparseable or that the scanner can't follow — subshells,
   unquoted heredocs, arithmetic expansion, an unknown program anywhere —
   falls through to the ordinary permission prompt. The hook never
   auto-allows a command it can't classify under its rules; the worst case
   is the status quo: you get asked.

Here is a stripped-down but working version. The real one adds many more
programs and checkers, consumes each wrapper's own flags, parses git's
global flags, recurses into substitutions, logs every decision with its
reason, and handles other tools (Read, Edit, MCP calls) — but nothing
structurally different:

```python
#!/usr/bin/env python3
"""PreToolUse hook, proof of concept: auto-allow Bash commands that
parse as safe.

Claude Code sends {"tool_name": ..., "tool_input": ...} as JSON on
stdin. Print a permissionDecision to skip the prompt; print nothing to
fall through to the normal permission flow.
"""
import json
import os
import shlex
import sys

ALWAYS_SAFE = {"ls", "cat", "grep", "rg", "head", "tail", "wc", "sort",
               "uniq", "find", "diff", "stat", "file", "echo", "which"}
WRAPPERS = {"nice", "ionice", "env", "time", "nohup"}
OPERATORS = {"&&", "||", ";", "|", "&"}
GIT_SAFE = {"status", "diff", "log", "show", "branch", "add", "commit",
            "stash", "fetch"}


def classify_git(tokens):
    # The real version also parses git's global flags (-C, -c, ...).
    sub = next((t for t in tokens[1:] if not t.startswith("-")), None)
    if sub == "commit" and "--amend" in tokens:
        return "ask"
    if sub == "add" and any(t in ("-A", "--all", ".") for t in tokens[2:]):
        return "ask"
    return "allow" if sub in GIT_SAFE else "ask"


def classify_one(tokens):
    # Peel wrappers and VAR=val prefixes to find the real program.
    # (The real version also consumes each wrapper's own flags.)
    while tokens and (os.path.basename(tokens[0]) in WRAPPERS
                      or ("=" in tokens[0]
                          and tokens[0].split("=")[0].isidentifier())):
        tokens = tokens[1:]
    if not tokens:
        return "ask"
    prog = os.path.basename(tokens[0])
    if prog == "git":
        return classify_git(tokens)
    return "allow" if prog in ALWAYS_SAFE else "ask"


def classify(command):
    lex = shlex.shlex(command, posix=True, punctuation_chars=True)
    lex.whitespace_split = True
    lex.commenters = ""
    try:
        tokens = list(lex)
    except ValueError:          # unparseable: let the human decide
        return "ask"
    if not tokens:
        return "ask"
    group = []
    for tok in tokens + ["&&"]:         # sentinel flushes the last group
        if tok in OPERATORS:
            if group and classify_one(group) != "allow":
                return "ask"            # every part must pass
            group = []
        else:
            group.append(tok)
    return "allow"


def main():
    data = json.load(sys.stdin)
    if data.get("tool_name") != "Bash":
        return
    if classify(data["tool_input"].get("command", "")) == "allow":
        json.dump({"hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "allow"}}, sys.stdout)


if __name__ == "__main__":
    main()
```

To run it yourself: save the script somewhere stable (mine lives at
`~/.claude/hooks/pre-tool-use.py`) and register it in
`~/.claude/settings.json` — or in a project's `.claude/settings.json` for
per-repo scope:

```json
"hooks": {
  "PreToolUse": [{"hooks": [
    {"type": "command", "command": "python3 ~/.claude/hooks/pre-tool-use.py"}
  ]}]
}
```

Claude Code snapshots hook configuration at session start, so an
already-running session won't pick it up until restarted; the `/hooks`
command shows what's active.

Even this toy version gets the interesting cases right: `git diff; echo
done` is allowed, `git push; echo done` asks, `git add file.c` is allowed,
`git add .` asks, `nice grep -r foo src` is allowed, and an unterminated
quote or an unknown program falls through to the prompt. It omits the
substitution recursion, though — `echo $(rm -rf /)` fools it; the real
version classifies the inner command.

## What four months of the log says

Every decision is appended to a log file with its reason. That log is how
the allowlist grows — when the same command keeps falling through, I add a
checker for it, from observed use rather than guesswork — and it's where
the following numbers come from.

Since mid-April the hook has classified 32,381 Bash calls. It auto-allowed
23,172 of them (72%) and denied 658 (2%); the remaining 26% fell through to
the normal permission flow, where Claude Code's own allowlist catches a
further chunk before a prompt ever reaches me. The two mechanisms compose:
the hook handles the structural cases, the built-in allowlist remembers the
specific one-off approvals.

Those 23,172 auto-allows are prompts I didn't see and didn't rubber-stamp.
That's the point. The prompts I do see now are `sudo`, `git
push`, `rm -rf`, package installs, and commands weird enough that a parser
refused to guess. Those I actually read.

## A hook can teach, not just gate

The `deny` verdict turned out to be the most interesting one, because the
model reads the denial reason and reacts to it. Those 658 denials are almost
all one rule that has nothing to do with safety:

```python
# Reject piping to head/tail without tee — it silently discards data.
return _block("Use '| tee <file> | head/tail' instead of bare "
              "'| head/tail' to avoid discarding data. "
              "Append '# no-tee' to override.")
```

`build 2>&1 | tail -20` throws away everything but the last twenty lines,
and half an hour later the model re-runs an expensive command to see output
it already had. The hook rejects the call with that message; the model reads
it, rewrites the command with `tee`, and the full output lands in a file.
There's an escape hatch (`# no-tee`) for the cases where discarding really
is fine. It is a style rule enforced mechanically: a `CLAUDE.md`
instruction saying the same thing gets weighed against the model's
judgement; a deny doesn't.

## What this is not

The hook is not a security boundary, and I want to be precise about the
threat model, because the HN thread mixes two different problems.

My hook defends against a *well-meaning* model's accidents: the reflexive
`git push`, the confident `rm -rf`, the history rewrite that seemed like a
good idea. It does nothing against a malicious or prompt-injected model.
`python`, `perl`, `curl` and `ssh` are on my always-allow list because I use
them constantly — and any one of them is arbitrary code execution or network
egress. Writing this post also found two real holes. `cat .env` sailed
straight through even though the Read tool is denied `.env`; the hook now
asks when any argument looks like a secret path (a `.env`, `.pem` or
`.key` file, or anything under `.ssh` or `.aws`) — a net for accidents,
not a boundary; the principled fix for secrets is a sandbox that never
mounts them. Worse, `echo $(rm -rf /)` was auto-allowed: the substitution
hid inside an argument token of a safe program. How that surfaced is the
addendum at the end of this post. Redirections are still not gated: `echo x > important-file` truncates
whatever it likes. Another HN commenter (kingstnap) put the general
version well: you can't regex your way around `python <<PY`; whether the
Python is safe is not a question about the command line.

If your threat model includes the model actively working against you, the
people in that thread running bubblewrap, podman, devcontainers or
throwaway VMs have the real answer: constrain the process, not the
prompt stream. I do that too where it matters — reproducers never run
against the host, agents that install packages get a container. The hook and
the sandbox solve different problems, and you can run both.

## An open problem: judgement calls

Between the two clean cases — commands a parser can classify, and malice
only a sandbox can contain — sits a third class I don't have a good answer
for: actions whose safety is a judgement call about content and context, not
about command shape.

Outbound messages are the canonical example. Back when I was on Slack, I
wanted Claude to be able to send *some* messages — posting a build result to
my own channel is fine — but not others, and "which others" is not a
predicate I can write in Python. A parser can see that a call sends a
message; it cannot see that this particular message is a bad idea. And
unlike nearly everything else in this post, a sent message is unrecoverable:
I have backups against `rm` and the reflog against `git reset`, but there
is no reflog for a dumb message to your boss. The actions where a structural check helps
least are exactly the ones you can't undo.

I'm off Slack now, but the same class is live in my setup today: `gh` is on
my always-allow list because I read PRs and issues with it constantly — and
the same binary posts comments. As far as the hook can see, `gh pr diff`
and `gh pr comment` have the same shape; only one of them publishes.

My actual solution is instructions to Claude — show me anything destined for
GitHub before posting it — which is the same advisory patch I spent the
first half of this post retiring, and I know it. The musing I keep returning
to: the hook is just a program, so nothing stops it shelling out to another
model for exactly this residue. Not the model that proposed the action —
asking the proposer to judge its own proposal buys you correlated failures —
but a different family, codex or DeepSeek, with a narrow question: here is
the message and its context; should a human see this first?

This shape isn't hypothetical. One commenter in the thread (prtmnth) ran
it before auto mode existed: a script that ran before every permission
request, asked Haiku for a safe/unsafe verdict, and logged the answers. Auto mode is the built-in
version of the same idea, and its classifier is now free. What I'd want is
the layered combination: the parser keeps answering the structural
majority deterministically, and a classifier only ever sees the
fall-through band — the judgement calls and the genuinely weird — with a
human prompt when the classifier is negative or unsure. The cross-family
version appeals to me over a same-family one because two families are less
likely to fail the same way, and because the verdicts would land in the
same log as every other decision, where I can audit them. I haven't built it. For now the
honest description of my outbound-message policy is: instructions, plus
paying attention.

## What's left

What the hook solves is the problem the 97% number describes: a prompt
stream so dense with obviously-fine commands that approval becomes a reflex,
at which point the prompts protect nothing. Anthropic's fix is a classifier
model watching the command stream. Mine is a parser — deterministic,
auditable via the log, as fast as any small local script (milliseconds,
against a server round trip for a classifier), and wrong in ways I can read
in the source rather than ways I have to discover empirically. For the residue that
neither structure nor allowlist covers, I'm still in the loop — and with so
much less noise, I'm actually reading.

## Addendum: reviewing the post fixed the hook

The substitution recursion in point 5 exists because this post was
reviewed, not because I knew to write it. Before publishing, I sent the
draft, the hook's source, and the HN thread to two other model families —
codex and DeepSeek — with instructions to check the post's claims against
the code. Codex refuted one directly: the draft claimed anything
syntactically odd falls through to the human, and codex pointed out that
process substitution doesn't — `cat <(id)` tokenises with `cat` in front,
and the hook judged the leading program only. Measuring that claim against
the live hook confirmed it, and turned up the wider hole: `echo
$(rm -rf /)` and backticks were auto-allowed too, and had been for the
whole four months the log covers.

Nothing went wrong in those four months, and the log shows why: the ~300
substitution commands actually allowed were all benign — `$(date ...)`
timestamps and heredoc commit messages — because the model writing them
was well-meaning. Which is exactly the threat model, and exactly why the
hole stayed invisible: an accident-catching net with a gap in it looks
identical to one without, right up until the accident.

So the fix went in before the post did. Substitutions now classify their
inner command recursively, and replaying the four months of allowed
commands through the new code prices the change at roughly one extra
prompt every three days. The draft claimed the hook fails towards the
human; reviewing the draft is what made that claim true.
