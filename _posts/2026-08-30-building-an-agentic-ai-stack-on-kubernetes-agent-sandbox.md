---
layout: post
title: "Building an agentic AI stack on Kubernetes Agent Sandbox: what it's for and what it's like to use"
date: 2026-08-30
summary: >-
  Prompt-based restrictions are a request, not a control. What it takes to move
  agent permissions into Kubernetes, and what running it actually teaches you.
tags: [kubernetes, ai, agents, security]
---

Sooner or later someone is going to ask your platform team to run AI agents in
production, if they haven't done already. Not a chatbot — agents: things that hold credentials, query databases,
call internal APIs, and occasionally write and execute code. And when they do, the
question that lands on infrastructure is not "which model" but a much older one:
**what stops it doing that?**

The uncomfortable answer, in most agent frameworks today, is a paragraph of English
in a system prompt. *You must not access customer records. Do not run code unless
asked.* That is a request, not a control. It sits in the same channel as the
attacker's input, it has no audit trail, and there is nothing to point a security
reviewer at. You can try adding guardrails before and after and during the calls etc, but with the complexity and cost of these extra layers, it's not always worth it or simply not manageable at scale.

[Agentic Meetings](https://github.com/erdincka/meetings) is a working demonstration
of the alternative, built on [Kubernetes Agent
Sandbox](https://agent-sandbox.sigs.k8s.io/) — a Kubernetes SIG Apps project that
adds `Sandbox`, `SandboxClaim`, `SandboxTemplate` and `SandboxWarmPool` as
first-class API objects. The application on top is a multi-agent meeting simulator:
a supervisor picks who speaks next, and each participant argues from their
organisational role over the company's documents. That part is an ordinary LLM demo.
The part worth an infrastructure architect's time is underneath — every agent runs
in its own kernel-isolated pod, and **what each one may do is a Kubernetes
authorisation decision**.

This piece is about how it is put together and what it is like to operate. There is
almost no machine learning in it, which is the point.

---

## The shape of the thing

![A meeting running: agents take turns while the events log records each supervisor decision](/assets/img/agent-sandbox-demo.gif)

Three trust boundaries, crossed once per turn:

![The architecture: a browser reaches a FastAPI backend through Gateway API; the backend dispatches turns to Tier A persona sandboxes, which claim Tier B execution sandboxes through the Kubernetes API server.](/assets/img/agent-sandbox-architecture.png)

**The backend** holds the orchestration graph, the database credential, and the
conversation state. **Tier A** is one gVisor-isolated pod per meeting attendee,
drawn from a warm pool, running that persona's reasoning loop. **Tier B** is a
second, stricter sandbox tier — no network at all — where any Python the model
writes actually executes.

The interesting edge is the one from Tier A to Tier B. When an agent decides it
needs to run code, its own pod creates a `SandboxClaim` against the API server
**using its own ServiceAccount token**. The backend is not in that path and cannot
broker around the decision. If the persona's profile carries the right role, the
claim succeeds. If it does not, the API server returns 403 — and the agent is told
so, in-band, as a tool result.

That last detail is a design choice worth copying. The denial is *reported, not
raised*: the tool returns `DENIED_BY_CLUSTER: ...`, the agent absorbs it and carries
on contributing to the meeting, and the refusal lands in the transcript and the
audit matrix as a policy decision rather than a stack trace. A control that crashes
the workload gets switched off within a week. A control that degrades it gracefully
survives contact with users.

## Five layers, and the ones that matter

The project describes its enforcement as five layers, and is honest about which are
load-bearing:

| Layer | Control | What it actually stops |
|---|---|---|
| Prompt | Only granted tools are registered | Honest mistakes |
| Runtime | Grants intersected with a mounted capability list | A compromised backend over-granting |
| Secret | Credentials mounted only into templates that need them | There is nothing to steal |
| NetworkPolicy | Default-deny egress per profile | A stolen credential being *used* |
| RBAC | Only some ServiceAccounts may claim an exec sandbox | The agent itself |

Read that bottom-up and it is a familiar defence-in-depth story. The top layer is
the one every agent framework ships with, and on its own it is worth roughly
nothing. The bottom two are ordinary Kubernetes objects that your cluster already
knows how to enforce and your security team already knows how to audit.

That is the real argument for doing this on Kubernetes rather than inside an agent
framework: **you are not inventing a permission system.** You are reusing the one
that already governs everything else you run, with the tooling, the audit trail and
the review process that comes with it.

## What it's like to use

You configure it, define who is in the room, and watch a meeting run.

![The agent registry](/assets/img/agent-sandbox-roles.png)

Personas are data, edited in the UI: a role, a tone, a risk level, and a set of
granted tools. The grant is the part with teeth — a persona's tool list resolves to
a **capability profile**, and the profile decides which ServiceAccount its sandbox
runs under. Editing a persona is therefore a privilege change, and the app treats it
as one: the operator role can do it, the viewer role cannot. That split is not the
usual read/write one, and getting it right required noticing that "edit this text
field" and "grant this agent the ability to execute code" were the same action.

Start a meeting and the supervisor takes over: it picks a speaker, that persona's
sandbox runs a turn, the utterance streams back over a WebSocket, and the events log
records every routing decision beside the transcript. Ask the General Counsel
persona to run a calculation and you get the demonstrable moment — a 403 from the
API server, surfaced in the audit matrix.

And you do not have to take the UI's word for it, which is the part that matters
if you are the one signing off on this. The same decision is one command away:

![Verifying enforcement against a live cluster](/assets/img/agent-sandbox-enforcement.png)

Three things there are worth separating. `/proc/version` returning
`4.19.0-gvisor` proves the kernel boundary is real rather than a silent fallback
to `runc`. The pod listing shows one ServiceAccount per persona, all under the
`gvisor` runtime class. And `kubectl auth can-i` answers the actual question —
**no** for the General Counsel, **yes** for the Finance Director — without
involving the model, the prompt or the application at all.

That last property is what makes this auditable. Your security reviewer does not
have to trust the app's audit matrix, or reason about whether a model could be
talked out of its instructions. They can ask the API server directly, the same
way they would for any other workload on the cluster.

![A concluded meeting](/assets/img/agent-sandbox-conclusion.png)

Meetings end with a decision record — notes, agreed actions, identified gaps —
rather than just a transcript.

## Six decisions an infrastructure architect will care about

### 1. "Installed" and "working" are different questions

This is the transferable lesson, and the project hit it at four separate layers.

A misconfigured `RuntimeClass` handler **does not fail loudly** — on several
container runtimes, containerd silently falls back to `runc`. The pod is green,
`Ready`, and completely unisolated. A CNI that ignores NetworkPolicy looks identical
to one that enforces it: the policies apply cleanly, `kubectl get networkpolicy` is
reassuring, and nothing is blocked. On kind's default CNI, a supposedly-restricted
persona reached Postgres on five consecutive attempts while the security model
claimed the opposite. Gateway API CRDs can be present with no controller watching
them. And an internal API router can be written, reviewed, unit-tested and never
mounted.

Each of those presents as a healthy deployment. The habit that falls out is
**provoke the behaviour and observe what happens** — not "is the object present",
not "is the pod ready". The repo ships a `make preflight` that reports, for the
requirements that can be present and inert, whether the thing actually functions.
Isolation is verified by reading `/proc/version` inside a running sandbox, never by
a readiness check.

If you take one thing from this project into your own environment, take that.

### 2. Keep the orchestration graph in one place

It is tempting to distribute the agent graph across the sandboxes. Don't. Sandboxes
here are **turn executors, not graph participants**: the backend owns the graph and
the state, and hands a sandbox one self-contained turn.

The alternative was tried. Running the reasoning loop remotely while RPC-ing
individual tool calls back cost a network round trip *per tool call* and turned a
demo into a distributed-systems project. Distributed checkpointing across untrusted
pods is a research problem; it is not a prerequisite for isolating agents.

### 3. Size the warm pool by concurrent speakers, not attendees

gVisor pods are not fast to start, so both tiers are warm-pooled. Two operational
details:

- **Acquisition is lazy.** A five-person meeting where two attendees never speak
  should not hold five pods. The claim happens on first selection.
- **A sandbox is held per persona, not per turn.** Claiming per turn drains a pool
  of two by the third turn of a four-person meeting, and everyone after that pays a
  cold start or fails outright.

Budget roughly **8 CPU and 16 GiB allocatable** for a full demo, most of it warm
pool sitting idle. It runs smaller by reducing warm replicas, at the cost of a
several-second pause the first time each persona speaks. That is the trade to have
in hand before someone asks why idle pods are burning quota.

### 4. Decide what a sandbox never holds

The rule here is absolute: **sandboxes never hold the application database
credential.** Everything a persona needs arrives through a scoped internal API on
the backend, authenticated with the sandbox's own projected ServiceAccount token —
validated by the API server, with identity read from pod labels rather than from a
request body the model composed.

Model calls go the same way. Sandboxes never call the inference provider; they reach
it through the backend's `/internal/v1/llm` proxy. So a persona sandbox needs no
route off the cluster and holds no provider key — which also means the provider can
sit behind a CDN with no stable address without any of it reaching NetworkPolicy.

### 5. Make the boundary legible from outside

Telemetry is optional here and off by default, on the grounds that observability
must never be why a system fails to start. When it is on, a single `traceparent` is
propagated over the sandbox RPC and again when a persona claims an exec sandbox, so
**one meeting turn renders as one trace spanning all three trust boundaries.**

That is what makes *slow*, *denied* and *broken* three distinguishable outcomes
rather than one indistinguishable pause. A system where those three look identical
from the outside is not one you should grant more autonomy to, however good its
isolation is.

Two supporting choices: metric labels are low-cardinality by construction — a
persona's *profile*, never its agent id — and **denials are counted separately from
errors**, because collapsing them would hide the one signal the project exists to
surface.

### 6. Deploys that report success and change nothing

Images are tagged by content digest, not `:latest`. A mutable tag leaves the
Deployment spec unchanged when the image is rebuilt, so `helm upgrade` finds nothing
to roll and the pod keeps serving stale code — a deploy that reports success and
changed nothing. Images are also cosign-signed with SLSA provenance, and `make
verify-images` answers "did this come from this repository's CI", which is a
different question from "did it change".

Schema changes run as an Alembic Helm pre-upgrade hook rather than as a startup side
effect, so a migration is explicit and reviewable. Configuration is environment-only
— credentials from Secrets, operator-tunable values in a database table — and
attempting to set a credential through the settings API returns a 422.

## What it does not claim

Worth stating plainly, because demos in this space usually overclaim.

gVisor is a strong boundary, not a perfect one. The five layers stop specific,
enumerated things and the repo documents what each one measures. NetworkPolicy is
only a control if your CNI enforces it — on a cluster where it does not, two of the
five layers are decorative, and the preflight check says so rather than pretending
otherwise. And none of this makes the *model* trustworthy; it makes the model's
capabilities bounded and its refusals observable, which is a different and more
achievable goal.

## Trying it

You need a cluster that can actually support the controls:

| Requirement | Why |
|---|---|
| Kubernetes 1.31+ | `ImageVolume`, which is how pgvector reaches Postgres |
| A RuntimeClass with kernel-level isolation | The boundary the design rests on |
| Agent Sandbox v0.5.6+ | The `Sandbox` / `SandboxClaim` / `SandboxTemplate` API |
| CloudNativePG 1.30+ | Postgres 18 with pgvector as a declarative extension |
| Gateway API + a live GatewayClass | The WebSocket upgrade the transcript needs |
| A CNI that **enforces** NetworkPolicy | Two of the five enforcement layers |

Then:

```bash
brew install kubectl helm kubeconform cosign uv
cp deploy/cluster/cluster.env.example deploy/cluster/cluster.env   # then edit
make preflight
```

`preflight` tells you what is missing and, for the requirements that can be present
and inert, whether they actually work. Install what it flags, then:

```bash
deploy/cluster/install-prerequisites.sh all
make verify-images && make deploy && make seed
make operator-token
```

Inference is any OpenAI-compatible endpoint — Ollama on your own machine works for
development. There is deliberately no in-cluster model server: serving a model well
is a different problem from orchestrating agents, and running both on one cluster
makes the demo compete for CPU with the sandboxes it exists to serve.

The repo's [architecture](https://github.com/erdincka/meetings/blob/main/docs/architecture.md),
[sandbox security model](https://github.com/erdincka/meetings/blob/main/docs/sandbox-security-model.md)
and [lessons learned](https://github.com/erdincka/meetings/blob/main/docs/lessons-learned.md)
docs go deeper than this piece does — the last of those is unusually candid about
controls that looked right and did nothing.

---

## The takeaway

The agent-isolation problem is not new, and it is not really an AI problem. It is
the multi-tenancy problem, arriving through a new door: untrusted code, generated at
runtime, that wants credentials and network access.

Kubernetes has spent a decade building answers to that — namespaces, ServiceAccounts,
RBAC, NetworkPolicy, runtime classes, admission control — and Agent Sandbox is the
work of wiring agent workloads into them rather than inventing a parallel permission
system inside a Python framework.

For an infrastructure architect, that reframing is the useful part. You do not need
to become an ML engineer to have an opinion about how agents should be run. You need
to insist that the enforcement live where you can see it, test it, and show it to an
auditor — and then treat "the prompt says not to" as what it is, which is a comment.
