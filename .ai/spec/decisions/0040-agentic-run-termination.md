# 0040: Unified Hard-Stop Contract for Agentic Runs

**Status:** Accepted [PLANNED: OLS-3298, OLS-4018]; outcome timing revised by OLS-4131
**Applies to:** lightspeed-agentic-operator, lightspeed-agentic-console

## Context

The cluster-wide `AgenticOLSConfig.spec.suspended` kill switch marks non-terminal runs `EmergencyStopped`, but its cleanup contract is best-effort and depends on sandbox references recorded in run status. A deletion error, a status-write race, or an orphaned workload can therefore leave a sandbox able to contact an LLM or the cluster after the system reports suspension complete. Clearing the kill switch before that teardown completes would also admit new work alongside the still-active sandbox.

There is also no way to stop one run without deleting its AgenticRun CR and its audit history. Approval denial prevents a stage from starting but cannot stop a stage already in progress. The console needs a per-run control, and both stop paths need the same reliable sandbox and access teardown behavior.

The design must preserve the condition-derived phase model, immutable sandbox-owned Result CRs, Kubernetes RBAC authorization, and durable audit evidence.

The original form of this decision wrote the terminal outcome immediately and performed cleanup asynchronously. That let a run report `Failed` or `EmergencyStopped` while its sandbox was still active, and left no signal between the stop request and the final outcome. OLS-4131 revises the timing: blocking stays immediate, the outcome is recorded only after teardown is confirmed, and a run-level `Stopping` condition exposes the interim.

## Decision

### One-way cancellation intent on AgenticRun

Add an optional `AgenticRun.spec.cancelled` boolean with effective default `false`. CRD validation permits only absent/`false` to true. Any caller with effective `patch agenticruns` permission may set it; no cancellation-specific role or group check is added.

Cancellation applies to every non-terminal phase, although the console exposes `Stop run` only while a sandbox is active. The reconciliation guard runs before approval, revision, terminal routing, and normal phase dispatch, so a persisted request prevents subsequent stages.

### Failed outcome with a specific cause

Per-run cancellation uses the existing `Failed` phase. Once teardown is confirmed (see the stopping contract below), the operator sets the phase-relevant workflow condition to `False`, reason `CancelledByUser`, and message `Run stopped by user`. The console keeps the Failed badge and displays `Stopped by user` as the cause.

There is no `Cancelled` phase or synthetic Result CR. Existing Result CRs remain immutable; late results cannot advance a stopped run or overwrite its stopped outcome. Cancellation cannot report which actions completed, so execution-time confirmation warns about possible partial cluster changes.

### Global suspension has precedence

For a non-terminal run where global suspension and per-run cancellation are both pending, global suspension wins and produces `EmergencyStopped`. If the run was already terminal before a later stop signal was processed, its terminal history is not rewritten. Global sweeping still removes active managed sandboxes associated with terminal runs.

### Safe global resumption

`AgenticOLSConfig.spec.suspended` remains `true` while the `Suspended` condition reason is `Stopping`. Admission rejects an attempt to clear the field until the operator verifies that teardown is complete — no active managed sandbox or access remains and no AgenticRun is still non-terminal — and records `AdminActivated`; it allows the change only when that reason is actually observed, because the condition write can lag the spec flip and an absent or stale condition is not permission to resume. Therefore the existing create-admission policy and reconciliation guard remain authoritative while any stopped sandbox workload or access can still exist or any stopped run still awaits its outcome. The suspension-activated Event is emitted when the field first becomes true, rather than being delayed until stopping finishes.

### Confirmed-teardown outcome with a shared `Stopping` condition [PLANNED: OLS-4131]

A stop request blocks workflow progression immediately, but the terminal outcome is not written until teardown is confirmed. Both stop paths set a run-level `Stopping` condition to `True` in the same reconcile that first blocks progression, and the run remains non-terminal while any teardown operation for it is pending or retrying. When the operator confirms that the run's sandbox workloads and sandbox access are removed, it records the final outcome (`Failed / CancelledByUser` or `EmergencyStopped`) and completes the `Stopping` condition (status `False`) in the same status update. If a teardown operation fails or an active workload remains, the run stays visibly stopping and cleanup is retried; the outcome is never recorded while an active workload or sandbox access remains. A run with no sandbox workload or access (for example `Pending` or `Proposed`) has nothing to tear down, so its outcome follows without delay.

Once a stopped outcome is recorded it is final: late workflow completions, Result CRs, approvals, or revision feedback cannot overwrite it or restart progression. Status writes should use resourceVersion-checked patches and re-evaluate on conflict so a stale completion write cannot undo stop progress or a recorded outcome; a separate controller queue is not required.

Both paths use one idempotent hard-stop contract:

1. revoke sandbox ServiceAccount reader access and execution RBAC;
2. delete bare Pods with zero grace;
3. delete SandboxClaims and hard-delete active backing Pods when claim cleanup is asynchronous;
4. discover resources from both run status and managed labels/owner identity;
5. retry from stop acceptance — while `Stopping=True` and after the outcome is recorded — until workloads and access are confirmed absent.

Per-run release never deletes a derived SandboxTemplate because compatible live SandboxClaims can reuse one. Template garbage collection is a separate operation that first verifies no live claim or workload references the template.

Sandbox status references remain while teardown is incomplete and are cleared only after workload and access removal. Together with the `Stopping` condition this gives the console a progress signal without adding a phase.

Global suspension directly lists active managed sandbox resources, including leaked or orphaned workloads and workloads attached to terminal runs. `AgenticOLSConfig` remains `Suspended=True / Stopping` until no active managed sandbox or associated access remains and no AgenticRun remains non-terminal; it reports `AdminActivated` only after both are confirmed. A run with no sandbox workload or access — for example `Pending` — is trivially torn down, so the resource check alone could complete before that run's reconciler records its outcome; requiring the run outcome as well closes that resumption window.

## Alternatives Considered

- **Dedicated AgenticRunCancellation CRD** — rejected because current requirements do not need stop-only RBAC delegation, requester fields, or an independent lifecycle. Kubernetes audit records the caller.
- **Cancellation through AgenticRunApproval** — rejected because cancellation is not an approval stage and coupling it to approval permissions would make authorization checks inaccurate.
- **New Cancelled or Cancelling phase** — rejected because a stop must not change the condition-derived phase model. The distinguishable cause is the `CancelledByUser` reason, and interim progress is exposed through the run-level `Stopping` condition rather than a `Cancelling` phase.
- **Synthetic Result CR with partial progress** — rejected because the operator cannot know which sandbox actions completed and Result CRs are sandbox-owned.
- **Cooperative shutdown before deletion** — rejected because an emergency control cannot trust the agent to respond or wait through a termination grace period.
- **Network isolation while retaining Pods** — rejected because it is more complex, may not terminate existing connections promptly, and keeps the untrusted process alive.
- **Stop only status-referenced non-terminal sandboxes** — rejected because status races, terminal cleanup failures, and orphaned workloads would preserve the safety gap in OLS-4018.
- **Use approval permission as the console stop gate** — rejected because `patch agenticrunapprovals` does not authorize the actual `patch agenticruns` operation.

## Consequences

- Per-run cancellation adds a mutable v1alpha1 field and CEL transition rule. Revision detection must not interpret that generation change as revision feedback.
- Users can distinguish ordinary failure from user cancellation through the normative `CancelledByUser` reason while retaining one Failed phase.
- Any user who can patch AgenticRuns can also set other mutable AgenticRun fields; field-only stop delegation remains unsupported.
- Global suspension completion requires both confirmed sandbox and RBAC teardown and a recorded outcome for every run it stops — a sandbox-less Pending run also holds the condition at `Stopping` until its outcome is written; it cannot be cleared while the `Suspended` condition reason is `Stopping`.
- The operator must expose the run `Stopping` condition, must continue reconciling cleanup from stop acceptance through the recorded outcome, and must discover resources independently of status references.
- Per-run cleanup cannot remove a reusable derived SandboxTemplate that is referenced by another live claim or workload.
- Hard deletion can lose in-process logs and unpublished results. Durable Result CRs, Kubernetes audit, and product audit storage are the supported forensic sources.
- The agentic console needs a separate access review for `patch agenticruns`, active-phase Stop controls, partial-execution warning, prominent `Stopping` presentation while the condition is true, failure-cause rendering, and shutdown-progress rendering.
- Operator tests must cover condition mapping, precedence, phase races, zero-grace deletion, both sandbox modes, orphan discovery, cleanup retries while stopping and after the outcome, stale-completion overwrite protection, late resources/results, suspension Stopping completion including the sandbox-less Pending-run race, rejection of premature deactivation (including before the `Suspended` condition is first recorded), and preservation of reusable templates with live references.
