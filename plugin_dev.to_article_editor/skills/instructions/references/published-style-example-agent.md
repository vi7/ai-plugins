---

title: "I let an AI agent make an infrastructure change"
published: false
description: "An AI agent took an infrastructure change from ticket to deployment. It also found a bug I would have missed, broke a consumer, and confidently got one important thing wrong."
tags: ai, devops, automation, sre
---------------------------------

# I let an AI agent make an infrastructure change

An AI agent recently took an infrastructure ticket from "please create these queues" to deployed and tested.

It also:

* found a bug I would probably have shipped;
* briefly broke a consumer;
* confidently gave me the wrong explanation for some infrastructure versus code drift.

All three were useful.

The first one showed me what agents can do.

The other two showed me what they need before you let them touch infrastructure.

## The ticket

Typical BAU ticket:

> Create a few message queues, configure dead-lettering and permissions, and make them available to the application.

Normally this is a small DevOps task.

But infrastructure changes are rarely isolated. The broker was shared, there were several environments, existing applications depended on it, and some configuration was generated from code.

So I decided not to do the change myself.

I gave it to an AI coding agent running in a terminal.

Its permissions were deliberately asymmetric:

**It could read almost everything. It could change almost nothing without my approval.**

My rules were simple:

* production is read-only;
* non-production changes require approval;
* changes must be reversible;
* don't guess when something is ambiguous.

The agent followed a fixed workflow:

```text
explore
  ↓
plan + pre-mortem
  ↓
independent review
  ↓
questions to human
  ↓
implement
  ↓
non-production rollout
  ↓
test
  ↓
review
  ↓
handoff
```

This turned out to matter more than the model itself.

## The first useful thing: it didn't start coding

The agent first explored the environment.

That immediately found several things I would have been tempted to skip.

### The ticket was ambiguous

The non-production broker was shared by several environments.

The queue names in the ticket didn't contain an environment identifier, so blindly creating them would have made different test environments share the same queues.

The agent stopped and asked which environments actually needed the change.

Good catch.

### There was already a pattern

Myself and other team members had implemented something very similar before.

Instead of inventing a new configuration, the agent found the existing implementation and used it as a reference.

This is one of the least glamorous but most valuable things an agent can do:

**look around before inventing things.**

### It found a potential time bomb

The planned queue configuration had a dangerous default. Under the expected failure scenario, messages could accumulate in memory while the consumer was unavailable. Enough messages could eventually trigger the broker's memory protection and affect unrelated applications. The agent suggested adding a size limit and changing the storage policy.

That was useful.

But _I didn't trust the suggestion_ just because it sounded reasonable.

I asked for an independent review.

## The second agent found the real problem

A separate agent reviewed the plan. Its first finding was marked **CRITICAL**. There was a regular expression containing an escaped dot.

Something like:

```text
amq\.default
```

Perfectly valid regex. Except this value eventually ended up inside JSON. And `\.` isn't a valid JSON escape.

The interesting part was when this would fail. Not during deployment. Not during the usual validation.

The broker only read that particular configuration during a restart.

So the change could have been deployed successfully, tested successfully, and then caused a failure weeks later when somebody restarted a node.

That's exactly the kind of infrastructure bug I want automation to catch.

The fix was tiny.

The important change was in the process:

**don't validate the template. Validate the actual artefact the system will consume.**

Render it. Parse it. Do that for every environment.

The reviewer also found a few other problems in the planned tests and configuration.

This was a good reminder that an agent doesn't have to produce the final answer to be useful.

Sometimes _its best job_ is simply:

> "Here are five things you haven't thought about."

## Then we changed the infrastructure

After I answered its questions, the agent implemented the change. It didn't import an entire configuration file.

It made targeted changes:

```text
create these objects
change these permissions
leave everything else alone
```

That sounds obvious, but it's an important property for infrastructure automation.

A command that says "make the whole configuration look like this" can have a very different blast radius from:

```text
create X
create Y
modify Z
```

Then it tested the change using the real application identity.

It tested the happy path:

```text
publish
→ consume
→ fail
→ retry
→ retry
→ retry
→ dead-letter
```

It also tested the negative cases:

- Could the application access things it shouldn't?
- Could it create things it shouldn't?

The answer needed to be no.

And then came one of my favourite tests. The agent deliberately filled the queue until the configured limit was reached.

If you configure a limit but never actually hit it, you don't really know whether the limit works. A limit you haven't triggered is still partly a theory.

## Then I let it restart the nodes

The infrastructure change also needed a rolling restart.

Before doing that, the agent checked for configuration differences between the repository and the live system. It also checked whether there were messages that could be affected by the restart. There was one queue containing messages that would not survive the restart.

So it asked me andI approved it.

The restart began, and everything looked great: all nodes were healthy, replicas were synchronized, no broker alarms, the dashboard was green.

Then the agent found a problem.

## The dashboard was green. The application wasn't.

One test consumer had stopped consuming, while it's Pod was healthy.

The cluster was healthy, the monitoring dashboard was mostly green, BUT the application was still broken.

The restart had caused the application to reconnect and redeclare its queue. The queue already existed with slightly different settings from what the application expected.

The broker rejected the declaration and the application stopped consuming, while its health check didn't notice.

This lasted for about ten minutes. No customer impact, because it was a test environment.

But it exposed a very important problem with the original verification:

**I was checking server health, not client health.**

The agent had done what I asked, the cluster was healthy, but that wasn't enough. :(

So we changed the procedure - before every disruptive operation, we now capture the important client-side state:

```text
consumers
connections
restart counts
```

After the operation, we compare it. Cluster health tells you that the cluster is healthy, but it doesn't tell you that your applications are happy.

Those are two different questions.

## One more miss: the agent got drift wrong

Earlier, the agent had found three broker policies present in the live system but missing from Git. It confidently called them manual drift. I believed it and even warned the team.

Later, I asked it to check Git history. Turns out two policies *had* been in the repo, but were removed months earlier. The broker's config import added settings without deleting old ones, so the policies stayed live. Worse, those two policies were still protecting dead-letter queues. Removing them as “drift” would have made things less reliable.

Only one policy was actually created manually. The agent had checked the current repo, skipped its history, and filled the gap with a plausible explanation.

**Plausible isn't verified.** Now any drift claim needs a history check and evidence before anyone acts on it.

## So we changed the rules

Each failure became a new rule.

| Problem                                            | New rule                                                            |
| -------------------------------------------------- | ------------------------------------------------------------------- |
| A restart broke a consumer                         | Notify the human immediately when user-facing behaviour changes     |
| Cluster was healthy but client was broken          | Compare client-side state before and after disruptive operations    |
| "Drift" was based on current Git state only        | Check Git history before calling something drift                    |
| Review repeated facts from the plan                | Reviewers must independently verify important facts                 |
| A plausible statement had no evidence              | Every important fact must have a command, query or source behind it |
| A configuration file could fail only after restart | Validate the exact artefact consumed by the system                  |

This is probably the most important part of the whole experiment.

**The agent is allowed to be wrong.**

The system around the agent shouldn't make the same mistake twice.

The workflow itself now contains these checks, so I don't have to remember them next time.

## What this means for a business

If you're thinking about using AI to automate DevOps or infrastructure work, I wouldn't start by giving an agent production access. I'd start much more boringly.

### 1. Give it read access first

Let it investigate.

Let it correlate logs and metrics.

Let it find configuration.

Let it prepare a change.

You can learn a lot about the agent without giving it the ability to break anything.

### 2. Gate writes with credentials

This is important.

Don't rely on:

> "Don't touch production."

That's a prompt.

Instead:

> The production credential literally cannot modify production.

That's a control.

### 3. Keep approval where the blast radius begins

The agent can prepare the change.

The human approves it.

You can gradually automate more once you understand the failure modes.

### 4. Make the agent ask questions efficiently

One message with five numbered questions is useful.

Five separate interruptions are annoying.

A good agent should come back with something like:

```text
1. These four environments need the queues. Agree?
2. Existing pattern A looks closest. Use it?
3. This policy may affect memory usage. Apply it?
4. One queue contains messages that won't survive restart. Proceed?
```

That turns the human into _a decision maker_ instead of a human API.

### 5. Turn mistakes into guardrails

This is where I think agentic automation becomes really interesting.

A mistake shouldn't just produce:

> "The AI made a mistake."

It should produce:

> "What rule can we add so neither the AI nor the next engineer makes this mistake again?"

That's how the system gets better.

## What actually changed for me

The agent didn't replace the DevOps engineer.

It changed what the DevOps engineer was doing.

Instead of spending the morning:

```text
read ticket
find config
search Git
check cluster
write YAML
run commands
test
write documentation
```

I spent much more of the time asking:

```text
What do we actually need to change?

What could go wrong?

How will we know it worked?

What must never be allowed?
```

Those are much more valuable questions to automate around.

And there is an interesting business implication here.

You don't necessarily need an AI that can autonomously run your entire infrastructure. You need an AI that can take a well-defined piece of infrastructure work and move it through a **safe, repeatable process**, while escalating the decisions that actually require human judgement.

That's a much smaller problem.

And, in my experience, a much more useful one.

## If you're starting with AI in DevOps

Three habits are worth stealing even if you never use an AI agent:

* **Prove your limits.** If you configure a timeout, retry count or size limit, trigger it deliberately and see what happens.
* **Check clients, not only servers.** "The cluster is healthy" does not mean "the application works."

The agent didn't replace judgement. It made the places where judgement was missing much easier to see.

And that's probably the part of this experiment I found most valuable.

---

I'm experimenting with AI-assisted infrastructure automation, particularly around DevOps and on-prem environments.

**Would you trust an AI agent to make a production infrastructure change if the permissions, approval gates and rollback path were designed correctly?**

I'm curious where other teams draw that line.
