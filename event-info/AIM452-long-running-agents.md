# AIM452 — Build long-running agents that persist for days on AgentCore

**Level:** 400 – Expert
**Event:** AWS re:Invent 2026, November 30 – December 4, Las Vegas

> Most agents handle a single request and forget everything. Production workloads are different — an incident investigation spans hours, a document review takes days, a customer onboarding stretches across weeks. These agents need to persist state, survive restarts, and pick up exactly where they left off. This session covers how to build long-running agents on Amazon Bedrock AgentCore using Runtime for durable execution and Memory for cross-session context. Learn checkpointing patterns, human-in-the-loop pause and resume, and cost strategies that keep idle agents from burning through your budget.

The talk is not published yet. These notes are the public AgentCore model that abstract describes, so the material is learnable without a seat.

## Two products, two jobs

**Runtime** keeps the working process and its files alive, and can stop and start that process without losing the workspace.

**Memory** keeps facts after the process, and after the session, are gone.

Files on disk do not become "the user prefers weekly summaries." A memory record does not restore `node_modules`. Use both.

## Two compute types

| | microVM | Runtime Instances |
| --- | --- | --- |
| Shape | Serverless, one agent per session, fast start | EC2 managed instance in your account, many agents per session |
| Longest one run | 8 hours | 14 days |
| Files that survive a stop | Managed session storage, up to 1 GB, 14 days idle | EBS volume on the capacity provider. Survives stop. Deleted when the session is deleted |
| GPU | No | Yes (`g4dn`, `g5`, `g6`, `g6e`, `gr6`, `g6f`, `gr6f`, `g7e`, `inf2`) |
| Network | Public or VPC | VPC only |
| Bill | AgentCore consumption while the microVM exists | EC2 price in your account (Savings Plans and reservations apply) plus an AgentCore management fee |

Use a microVM for an orchestrator that routes work and returns quickly. Use Instances when the job itself needs days, a shared filesystem, the OS, or a GPU. A lightweight microVM agent can dispatch to Instance workers through the same `InvokeAgentRuntime` API.

Compute type cannot change after the runtime is created. On Instances, the capacity provider (OS, instance types, VPC, volumes, IAM) is fixed after creation except for its description. Duplicate it to change it.

Official comparison: [Instances](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-instances-how-it-works.html). Announcement with a two-agent shared directory: [Runtime instances](https://aws.amazon.com/blogs/aws/runtime-instances-persistent-compute-for-production-ai-agents-on-amazon-bedrock-agentcore/).

## Three clocks

### 1. This boot of the compute

`maxLifetime` starts when the environment is created and does not reset when you call again. Default is 8 hours. The ceiling is 8 hours on microVMs and 14 days on Instances.

When it hits, AgentCore stops the compute. The session id remains. The next invoke with that same `runtimeSessionId` starts a new environment and attaches the saved disk. Each resume gets its own lifetime. A multi-week onboarding is a chain of runs, not one process that stays up for a month.

### 2. Idleness

`idleRuntimeSessionTimeout` defaults to 15 minutes. It resets on every invoke. It does not reset just because the process is sitting there.

The agent reports status on `/ping`:

- `Healthy` means idle. After the idle timeout, the compute is torn down.
- `HealthyBusy` means background work is still running. The idle timeout does not fire.

The SDK brackets real work:

```python
task_id = app.add_async_task("investigate")
# work on another thread; do not block the entrypoint
app.complete_async_task(task_id)
```

`time_of_last_update` on the ping body must change only when status changes. Stamping "now" on every ping looks like continuous activity, so the idle timeout never fires and the session lives until `maxLifetime`. That is the expensive failure mode.

Blocking work inside `@app.entrypoint` also blocks `/ping`. The platform then treats the agent as dead and kills it around the 15-minute mark. Background work goes on another thread or an async task.

Docs: [Long-running and async agents](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-long-run.html), [Lifecycle settings](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-lifecycle-settings.html).

### 3. Durable data, which has its own expiry

- In-memory Python objects and anything outside the mount die with the compute.
- Managed session storage (microVM) restores the mount path on resume. Cap is 1 GB. Retention is 14 days of idle time. A runtime version update clears it. The mount is present during invocation, not during container init.
- Instance EBS volumes reattach on resume. EBS is tied to an Availability Zone, so the replacement instance has to come up where the volume lives. Deleting the session deletes the volume.
- AgentCore Memory is independent of both. Short-term memory is the event log for one actor plus session: messages and tool calls, reloaded on the next invoke with that session id. Long-term memory is extracted asynchronously into strategies (semantic, summary, user preference, episodic, or custom) and retrieved later by `actorId`, including in a new session.

Call `StopRuntimeSession` and wait until it finishes before you invoke again. The stop flushes files. A resume that races the stop mounts an incomplete disk.

Docs: [Persist files across stop/resume](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-persistent-filesystems.html), [Sessions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html), [Long-term memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/long-term-memory-long-term.html).

## Checkpoint, then pause for a person

The durable pattern is a small state file on the persistent mount. A LangGraph checkpointer written to the container's local disk disappears with the compute.

```text
/mnt/workspace/cases/INC-1042/state.json
```

```json
{
  "phase": "needs_human",
  "question": "Isolate the payments ASG, or keep serving while we trace?",
  "hypotheses": ["bad deploy 14:10", "upstream timeout"],
  "evidence": ["notes/timeline.md", "queries/alb.txt"],
  "next_action": "wait"
}
```

The loop:

1. The agent writes that file, plus any raw evidence, before it asks the person.
2. It calls `complete_async_task` so `/ping` returns `Healthy`.
3. The entrypoint returns `{ "status": "needs_human", "question": "..." }` immediately.
4. The app calls `stop_runtime_session`. Compute billing stops. The volume or session storage remains.
5. Hours or days later, the human answers. Call `InvokeAgentRuntime` with the same `runtimeSessionId` and the answer in the payload.
6. The entrypoint reads `state.json` first and continues from `phase`, instead of asking the model to reconstruct the case from chat.

Also write a short case summary into long-term memory under the case or actor id. Session storage expires after 14 idle days, and a runtime update drops the microVM mount. Memory is what is still there if the disk is gone. On resume, the harness loads short-term events for that session and retrieves long-term records before the model runs.

The same session id is how collaborating agents share a machine. Two runtimes on one capacity provider, invoked with one `runtimeSessionId`, land on one instance and see the same files. The writer drops `code.py`. The reviewer reads it. They do not need to call each other.

## Cost control

Idle-but-alive is the leak. `HealthyBusy` is correct while a scan is running. It is wrong while you wait for a person or for the next business day.

- Waiting on a human: checkpoint, return `Healthy`, `StopRuntimeSession`.
- Overnight with no work: stop. Resume with the same session id. Disk comes back. The first invoke after a stop is slower because it provisions compute.
- Dev: idle 2–5 minutes, max lifetime 15–30 minutes.
- Interactive: idle 10–15 minutes, max a few hours.
- A real multi-hour job on a microVM: idle 30 minutes, max 8 hours, and stay `HealthyBusy` only during the job.
- A multi-day Instance job: raise both limits up to 14 days, and still stop when the agent is blocked. `idleRuntimeSessionTimeout` must be less than or equal to `maxLifetime`, and an Instance runtime cannot exceed the capacity provider's own max lifetime.
- When the case is finished, delete the session so the EBS volume goes away. Stopping is not deleting.
- Keep the cheap orchestrator on microVMs. Put only the long, GPU, or multi-agent work on Instances.

## The three examples in the abstract

**Incident, a few hours.** A microVM is enough if one boot stays under 8 hours. Checkpoint the timeline and hypotheses to `/mnt/workspace`. Stay `HealthyBusy` only while queries run. If you page a human, stop the session.

**Document review, several days.** An Instance, or a microVM that stops each evening and resumes the next morning on the same session id. The corpus and notes live on the mount. A running summary lives in long-term memory in case you pass 14 idle days or ship a new runtime version.

**Onboarding, weeks.** Do not hold one process open. Each wait (legal, access, the customer) is a stop. The session id and volume carry the workspace. Memory carries decisions and preferences into later sessions. Fourteen days is the cap on one boot, not on the case.

## Read next

1. [Long-running and async agents](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-long-run.html)
2. [Lifecycle settings](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-lifecycle-settings.html)
3. [Persist files across stop/resume](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-persistent-filesystems.html)
4. [Instances](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-instances-how-it-works.html) and the [announcement post](https://aws.amazon.com/blogs/aws/runtime-instances-persistent-compute-for-production-ai-agents-on-amazon-bedrock-agentcore/)
5. [Long-term memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/long-term-memory-long-term.html)
