# Secrets hygiene for agent-driven ops

These skills have an agent operate software that can delete your media library.
The agent needs the *capability*. It never needs the *credential*. Those are
different things, and most setups hand over both without noticing.

An agent transcript is a log file. Anything the agent reads or prints lands in
it, gets summarized into future context, and may sync to whatever service hosts
the session. An API key that appears in a transcript once is not a secret
anymore.

## The exposure ladder

From worst to best. Each rung removes a place the key can leak.

1. **Key pasted into the chat.** Never. It is now in the transcript forever,
   and in every summary of that transcript.
2. **Key in a file the agent opens.** The common default, and barely better:
   `cat ~/.config/arr/env` puts the key in context just as surely as pasting
   it. So does `config.xml`, which is why the recipes here never tell the
   agent to read it.
3. **Key in the environment, injected by the shell.** The recipes in these
   skills are written as `-H "X-Api-Key: $ARR_KEY"` on purpose: the shell
   expands the variable, the agent composes the command without ever seeing
   the value. This is the floor these skills assume, and it costs nothing.
4. **Key held by a broker the agent cannot read.** The credential lives with a
   separate user or host; the agent calls a wrapper, the wrapper injects, the
   agent gets output only. Even a compromised or confused agent has nothing to
   exfiltrate. [HomelabHero](https://github.com/serversathome/homelabhero)
   productized this shape for SSH-driven homelab ops and deserves the credit.

Rung 3 is the sane default for a home stack. Reach for rung 4 when the agent
runs unattended, or against an instance you answer for but do not own.

## Rules that hold at any rung

- **Env file outside any repo, `chmod 600`, never committed.** No repo
  history, no accidental `git add`.
- **One key per app is what the arrs give you; treat each as the app's root
  password.** There is no scoped or read-only arr key, which makes the recycle
  bin (Settings > Media Management) and the read-first doctrine in these
  skills your actual guardrails.
- **Never echo the variable.** Not to debug, not to "check it is set". Use
  `test -n "$ARR_KEY" && echo set` if you must confirm existence.
- **Watch your own logs.** Shell history, `curl -v`, and pasted debug output
  can all reproduce a header that carries the key. The arrs also embed the
  key in some generated URLs (RSS, calendar feeds); treat those URLs as
  secrets too.

## The test

Ask the agent what the API key is. The right answer is that it cannot know.
If it can answer, you are on rung 2, and one bad prompt away from rung 1.
