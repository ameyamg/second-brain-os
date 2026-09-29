# The Day-One Plan

Six modules is a lot of theory to hold at once, so here is the whole course compressed into one working day. Five layers, and on each one the single cheapest move that pays for itself immediately. None of them needs new infrastructure. All of them can be done before you next let an agent run unattended.

## Context: make the stable part byte-stable

Take everything that never changes between runs — role, rules, output format, tool guidance — and move it into one system prompt that is byte-for-byte identical every time. [Module 1](../course-1-context/how-models-read.md) covered why: models read position-sensitively, and a stable prefix is the part they treat as ground rules rather than conversation. The operational bonus is that prompt caching only works on identical prefixes, so the same edit that makes behaviour more consistent also makes every run cheaper. Dynamic material — the task, retrieved notes, tool results — goes after the stable block, never inside it. The deeper craft of what goes into context and in what order is the [context engineering page of the harness handbook](../track-harness/context-engineering.md).

## Loop: set a turn limit and a dollar limit

Before an agent runs without you watching, it needs two ceilings: a maximum number of turns and a maximum spend per run. [Module 2](../course-2-loop/loops-vs-workflows.md) made the case that a loop without a stop condition is not autonomous, it is unbounded — and the failure mode is not dramatic, it is an agent politely retrying a broken tool forty times at your expense. Both limits are a few lines of code, and hitting either one should end the run loudly, with the trace saved, so you learn something from every truncated run. Stop conditions, critics, and overnight runs are the subject of the [loop engineering handbook](../track-loop/README.md).

## Gate: label 200 examples of your most common decision

Find the one decision your system makes most often — keep or discard, route to A or B, worth flagging or not — and hand-label 200 real examples of it. [Module 3](../course-3-gate/cheap-decisions.md) argued that most of an agent's calls are not generation at all, they are classification, and a labelled set turns the fuzziest part of your pipeline into a measured one: you now know the accuracy of whatever makes that decision, and you can swap in something cheaper and prove nothing broke. An afternoon of labelling is the whole cost. The [Jev engineering handbook](../track-jev/README.md) is the deep dive on typed decisions with confidence scores.

## Harness: run the agent as a user that cannot delete anything

Create a separate account or role for the agent with read access, write access to its own workspace, and no delete, no payment, no admin — then run it there, today. [Module 4](../course-4-harness/the-office.md) framed the harness as the office the agent works in; this is the cheapest version of that office, and it converts a whole class of catastrophic failures into permission errors you read about in a log. It also makes the audit question trivial: anything done by that account was done by the agent. Guardrails, tool policies, and graduation to wider permissions are in the [harness handbook](../track-harness/README.md).

## Evals: write the last five failures down as test cases

Take the five most recent times the agent got something wrong — you remember them — and turn each into a test case: the input, and what a correct output must contain. [Module 5](../course-5-evals/two-kinds-of-checks.md) showed the two kinds of check; five regression cases is the humblest possible suite, and it is already enough to gate your next prompt change. From here the flywheel runs itself: every new failure becomes a case, and the suite grows exactly where your system is weakest. The full discipline is the [eval engineering handbook](../track-evals/README.md).

## The hard part moved

The honest closing note: the model is no longer the hard part. It arrives capable, improves without your help, and costs less every year. What separates a demo from a system you trust is everything around it — what it reads, when it stops, how it decides, what it is allowed to touch, and how you know it still works. That is the five layers. They are not glamorous, they are mostly plumbing, and they are where the work is now.
