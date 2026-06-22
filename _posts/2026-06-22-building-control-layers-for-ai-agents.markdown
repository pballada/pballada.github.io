---
layout: post
title: "Building Control Layers for AI Agents: A Practical Architecture for Safer Tool Use"
date: 2026-06-22 09:00:00 +0000
categories: [AI, DeveloperTools, SoftwareEngineering]
header_image: /assets/images/ai-autonomous-teammate.jpg
tags: [ai, agents, safety, developer-tools, architecture, product-engineering, claude]
---

The next wave of AI products is not just about generating better text.

It is about letting models **take actions**.

That means:

- calling tools
- reading and writing files
- triggering workflows
- modifying state
- operating across multiple systems
- running with partial autonomy

And the moment an AI system can act, one technical question becomes much more important than prompt quality.

> **What control layer sits between the model and the real world?**

That question moved closer to the center again this week after Google DeepMind outlined its new **AI Control Roadmap** for more capable agents.

The interesting part is not just the safety framing.

It is the architecture implication.

If advanced agent systems need monitoring, access control, async alerts, and shutdown mechanisms, then those concerns cannot live as vague future work.

They need to exist as concrete software.

This is where a lot of current AI products still feel immature.
They have strong model capability but weak operational boundaries.

So this post is not about AI safety as an abstract debate.
It is about a more practical engineering question:

> **How should you design a control layer around AI agents that use tools?**

Let’s make it concrete.

---

## The Core Shift: From Answers to Actions

A chatbot that returns a wrong answer is often recoverable.

An agent that can:

- send a message
- edit code
- create a ticket
- call an internal API
- delete a file
- trigger production automation

can fail in a much more expensive way.

That changes the system design.

The old interaction model looked roughly like this:

```text
user -> prompt -> model -> response
```

The new one looks more like this:

```text
user -> orchestrator -> model -> tool selection -> execution -> state change -> follow-up actions
```

Once that is true, the model is no longer just producing language.
It is participating in an execution loop.

And execution loops need controls.

---

## What a Control Layer Actually Is

When I say **control layer**, I mean the software boundary that governs how an AI agent interacts with tools, data, and side effects.

A useful control layer usually handles some combination of:

- permission checks
- tool allowlists
- scope limits
- execution approval rules
- budget limits
- audit logs
- anomaly detection
- human intervention
- rollback or shutdown paths

In other words, it answers questions like:

- is this tool call allowed?
- is it allowed for this user?
- is it allowed in this environment?
- is this action too risky to run automatically?
- should someone be alerted before or after it runs?
- how do we stop the system if behavior starts drifting?

That is not “extra” infrastructure.

For serious agent products, it is part of the product.

---

## A Practical Reference Architecture

A simple mental model is to stop thinking of the model as the component that owns execution.

Instead, treat the model as a **planner** operating inside a constrained runtime.

A practical architecture often looks like this:

```text
User
  -> Agent Runtime
    -> Policy Engine
      -> Tool Router
        -> Execution Sandbox
          -> Real Tools / APIs / Files / Services
    -> Audit Log
    -> Alerting System
    -> Kill Switch / Pause Controls
```

Each layer has a different job.

### 1. Agent Runtime
This coordinates the loop:

- gather context
- call the model
- interpret tool requests
- pass requests through policy
- collect results
- decide whether to continue

The runtime should not blindly execute whatever the model asks for.
That is the first big mistake.

### 2. Policy Engine
This is where the real rules live.

It should evaluate dimensions like:

- user identity
- workspace or tenant
- environment (`dev`, `staging`, `prod`)
- tool category
- data sensitivity
- action type (`read`, `write`, `delete`, `send`, `deploy`)
- risk score

This can start simple and still be very useful.

### 3. Tool Router
The model should never call arbitrary code directly.

Instead, it should request a structured tool action, and the router should map that request into a strongly defined tool interface.

That keeps execution legible.

### 4. Execution Sandbox
Even approved actions should run inside bounded environments whenever possible.

Examples:

- limited file system roots
- restricted network access
- timeouts
- resource caps
- separate credentials per tool domain
- dry-run mode for high-risk actions

### 5. Audit and Alerting
Every meaningful action should leave traces.

At minimum, log:

- who initiated the run
- which tool was called
- which arguments were used
- what policy allowed it
- what result came back
- whether any override or escalation occurred

### 6. Kill Switches
You need a fast way to:

- pause a run
- revoke a tool
- disable a tenant
- require human approval
- shut down the whole agent class if needed

This is not paranoia.
It is basic operational hygiene.

---

## Design Principle #1: Tool Access Should Be Explicit

One of the cleanest patterns in agent systems is to force tools into explicit contracts.

Bad pattern:

- let the model produce arbitrary shell commands
- let it infer hidden capabilities
- let it improvise side-effectful behavior from vague instructions

Better pattern:

- define tools with names, schemas, permissions, and expected side effects

For example:

```json
{
  "name": "create_github_issue",
  "risk": "medium",
  "requiresApproval": false,
  "inputSchema": {
    "repo": "string",
    "title": "string",
    "body": "string"
  }
}
```

And for a riskier tool:

```json
{
  "name": "deploy_production",
  "risk": "critical",
  "requiresApproval": true,
  "allowedEnvironments": ["staging", "prod"],
  "inputSchema": {
    "service": "string",
    "version": "string"
  }
}
```

This sounds obvious.

But a surprising amount of agent experimentation still collapses tool boundaries into “the model can basically do anything if prompted carefully enough.”

That does not scale.

---

## Design Principle #2: Separate Read Actions From Write Actions

A lot of systems lump tool usage into one flat capability set.
That is a mistake.

In practice, `read` and `write` actions should be treated very differently.

Examples of mostly-read actions:

- list files
- fetch issue metadata
- inspect calendar availability
- read docs
- query analytics

Examples of write or side-effectful actions:

- edit files
- send messages
- create PRs
- trigger CI
- modify tickets
- deploy code

This distinction matters because it lets you build better policies.

A useful agent often needs broad read access and narrow write access.

That leads to safer defaults like:

- **read widely**
- **write narrowly**
- **delete rarely**
- **send externally only with clear policy**

That single framing improves a lot of agent designs.

---

## Design Principle #3: Add Human Checkpoints at the Right Boundaries

Not every workflow needs human approval.
But some absolutely do.

The trick is not to ask for approval everywhere.
The trick is to ask at the **right transition points**.

Good approval boundaries often include:

- first external send
- destructive mutations
- production changes
- high-cost operations
- actions involving sensitive data
- cross-system workflow jumps

That gives you a cleaner pattern:

```text
model proposes action
-> policy scores action
-> runtime decides: allow / require approval / deny
-> execution continues only if allowed
```

This keeps the system useful without pretending every action has the same risk.

---

## Design Principle #4: Build for Interruption

A surprising number of agent demos are built as if the happy path is the only path.

Real systems need interruption.

That means your runtime should support:

- pausing after the next step
- stopping immediately
- cancelling long-running tool calls
- quarantining suspicious runs
- revoking credentials mid-session
- downgrading the run to read-only mode

Think of it this way.

If the agent can continue, the operator should be able to stop it.

That sounds simple, but it has real architectural consequences.
For example:

- tool calls need timeouts
- workflows need state checkpoints
- retries must be bounded
- queued actions must be inspectable
- session state cannot be hidden in one opaque model loop

Once agents become more autonomous, **interruption becomes a feature**.

---

## Design Principle #5: Observability Beats Vague Trust

One of the most common AI product mistakes is to rely on “the model is usually good enough” as an operational strategy.

That is not a strategy.
That is optimism.

If an agent is doing meaningful work, you need observability.

At minimum, I would want:

- run-level logs
- tool-call timelines
- approval events
- policy decisions
- cost and latency metrics
- failure reasons
- retry counts
- user-visible action summaries

A basic event model might look like this:

```json
{
  "runId": "run_4821",
  "step": 7,
  "event": "tool_call_approved",
  "tool": "write_file",
  "risk": "medium",
  "policy": "workspace_write_allowed",
  "timestamp": "2026-06-22T09:12:31Z"
}
```

This kind of event stream matters for more than debugging.
It becomes essential for:

- compliance
- incident review
- customer trust
- product tuning
- safety evaluation

The more autonomous the system, the less acceptable black-box execution becomes.

---

## A Minimal Policy Model That Goes a Long Way

You do not need a giant governance platform on day one.

A surprisingly effective first version can combine just a few concepts:

### Tool classification
Each tool gets a risk class:

- `low`
- `medium`
- `high`
- `critical`

### Action type
Each tool action declares:

- `read`
- `write`
- `delete`
- `send`
- `execute`

### Environment scope
Each tool is limited by environment:

- local
- dev
- staging
- prod
- external

### Approval mode
Each tool or action path defines one of:

- auto-allow
- require-human-approval
- deny-by-default

### Budget controls
Add basic limits like:

- max tool calls per run
- max external messages per session
- max file writes per task
- max spend or compute budget

That is already enough to prevent a lot of bad failure modes.

---

## If You Build Coding Agents, This Matters Even More

Coding agents are one of the clearest examples of why control layers matter.

They often have access to:

- the file system
- git state
- package managers
- test runners
- secrets by accident
- CI surfaces
- deployment workflows

That makes the tool boundary more important, not less.

A practical coding-agent setup should strongly consider:

- allowed directories
- safe vs unsafe command classes
- approval for networked package installs
- approval for destructive git operations
- read-only mode for exploration
- separate permissions for edit, test, commit, and push

This is also where products like Claude-style coding workflows become interesting.

The better the agent gets at doing real software work, the more important it becomes to separate these capabilities:

- inspect
- edit
- execute
- publish

Those are not all the same privilege.

And they should not be treated as one.

---

## The Most Useful Mindset: The Model Is Not the Root of Trust

This may be the single most important design shift.

Do not treat the model as the root of trust.

Treat the model as a high-variance planning component operating inside a runtime that owns trust boundaries.

That means:

- the runtime validates inputs
- the runtime enforces permissions
- the runtime decides whether execution can proceed
- the runtime records what happened
- the runtime exposes stop controls

The model can recommend.
It can plan.
It can rank options.
It can request tools.

But it should not unilaterally define what is allowed.

That responsibility belongs elsewhere.

---

## Final Thought

A lot of the AI industry still talks about agents as if the hard part were making them more capable.

That is definitely one hard part.

But if the current direction holds, the next generation of strong AI products will be defined just as much by their control architecture as by their model quality.

The durable systems will not be the ones that merely do impressive things in a demo.

They will be the ones that can:

- act usefully
- stay within bounds
- surface what they are doing
- accept interruption
- recover from mistakes
- earn trust over repeated use

That is an engineering problem.
And increasingly, it is one of the most important ones in AI product development.
