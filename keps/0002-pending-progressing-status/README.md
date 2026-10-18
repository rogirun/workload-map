<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright (c) 2026 NVIDIA Corporation -->

# KEP-0002: Pending, Progressing, and minimal viable Running

- Status: provisional
- Authors: @rogirun
- Issue: https://github.com/dsx-ai-factory/workload-map/issues/347
- Created: 2026-10-18
- Last updated: 2026-10-18

## Summary

Add two normalized statuses, `Pending` and `Progressing`, that refine
`Initializing`, and define `Running` as the workload's minimal viable state.

- `Pending`: no pod of the workload is bound to a node. The workload is not
  scheduled yet and waits for capacity, quota, or its turn in a queue.
- `Progressing`: at least one pod is bound, so the workload holds node
  resources, and the minimal viable state is not reached yet.
- `Running`: every component the workload cannot function without has its
  minimum number of ready pods. One ready pod is not enough for a
  multi-component workload. For example, a LeaderWorkerSet is `Running` once
  one complete group (the leader and all its workers) is ready.

`Initializing` stays. It is reported alongside `Pending` or `Progressing`, and
alone when a definition cannot tell which, so existing consumers see no
change.

The facts that separate these statuses are pod binding and per-component
readiness. Most workload CRs carry neither. The only way to answer for every
workload type is to let a definition read the workload's pods and, for some
stacks, the scheduler's own objects. That is the resource references feature.
This KEP proposes, as the holistic solution, status mappings that read
references, written in CEL. It lists the mandatory changes that path needs,
in dependency order. Two cheaper options are described as well, with the
reason each one is only partial, per workload.

## Motivation

A workload that shows `Initializing` for two hours is either stuck (no quota,
unschedulable, waiting in a queue) or starting normally (pulling images,
running init containers). The user needs to act in the first case and wait in
the second. Karta cannot tell them which.

The two cases also differ in cost. A pod is bound when the scheduler sets
`spec.nodeName`. From that moment the node's allocatable resources, GPUs
included, are charged to the pod, even while the image is still pulling. The
kubelet only pulls images for pods already bound to its node. To an external
viewer, `Pending` must therefore mean "not scheduled, holds nothing", the same
meaning the Kubernetes pod phase and every queueing system give it.

`Running` has the opposite problem. Several catalog definitions report
`Running` as soon as one pod is ready, or as soon as the workload controller
counts one replica ready. The LeaderWorkerSet controller counts a group ready
off its leader, so a group whose workers are unschedulable reads `Running`
while it cannot serve (#228). `Running` must mean the workload can do its job.

| Status | Pods | Node resources held |
|---|---|---|
| `Pending` | None bound | None |
| `Progressing` | At least one bound, minimal viable state not reached | Yes, not yet productive |
| `Running` | Minimal viable state reached | Yes, productive |

A partially bound gang holds resources, so it is `Progressing`.

### Requirements

The issue, read together with the related LWS issue #228, sets these
requirements. Each option below is judged against them.

| Id | Requirement |
|---|---|
| R1 | Report `Pending` while no pod of the workload is bound, and `Progressing` once any pod is bound and the workload is not `Running`. |
| R2 | Work for every catalog definition, not only for some. |
| R3 | Report `Running` only at the minimal viable state: every required component has its minimum ready pods (#228). |
| R4 | Cover the causes the issue names: unschedulable, quota, queue. |
| R5 | Prove every mapped state with a recorded e2e flow, as `AGENTS.md` requires. |

### Findings

Research covered the 23 catalog definitions and all 67 recorded e2e flows.

- Most workload CRs do not carry the bound/unbound fact. Their controllers
  count pods (`active`, `readyReplicas`) without reporting scheduling. An
  unschedulable pod and a pulling pod look the same on the CR.
- Only Pod, Agent Sandbox, and the head pod of a Ray cluster expose binding on
  the CR.
- Some CRs count ready pods per component (JobSet per replicated job, Dynamo
  per service, Grove per replica against clique `minAvailable`). Others
  aggregate in a way that hides a missing component (LWS counts a group ready
  off its leader).
- No recording today captures a `Pending` state. The pod fixture stops before
  the scheduler writes `PodScheduled=False`. Every other stalled fixture uses
  a missing image, which is a bound pod.
- The pod is the universal source of both facts: `spec.nodeName` and the
  `PodScheduled` and `Ready` conditions. Scheduler-side objects (a Kueue
  `Workload`, a gang `PodGroup`, a Grove `PodClique`) carry the cause.

### Goals

- Add `Pending` and `Progressing` to the status vocabulary and to
  `statusMappings`, without removing `Initializing`.
- Define `Running` as the minimal viable state, per catalog definition.
- Define the holistic solution and the mandatory changes it needs.
- Keep every existing definition and consumer working unchanged.
- Prove each mapped state with a recorded e2e flow.

### Non-goals

- Removing or deprecating `Initializing`. That is a later decision, made once
  consumers have adopted the new values.
- Changing `Degraded`. A workload above its minimum but below its desired
  count is `Running`. `Degraded` keeps its current per-definition meaning.
- Detecting a workload that is bound but stuck. This covers
  `ImagePullBackOff`, a crash before ready, and a partially bound gang whose
  remaining pods can never be scheduled. All of them read `Progressing` under
  this KEP. Detecting them is a follow-up.
- Karta fetching anything from the cluster. Consumers keep doing all reads.
- New features for definitions written in jq. References are bound only in
  CEL definitions.

## Proposal

### Status semantics

| Status | Meaning |
|---|---|
| `Initializing` | The workload is starting. Reported whenever `Pending` or `Progressing` is, and alone when the definition cannot tell which. |
| `Pending` | No pod of the workload is bound. Causes: unschedulable, scheduling gates, quota, queue, schedule not reached, pods not created yet. |
| `Progressing` | At least one pod is bound and the minimal viable state is not reached. Covers image pulls, init containers, and gangs still being bound. |
| `Running` | The minimal viable state is reached: every required component has its minimum ready pods. Holds from the minimum up to the desired count. |

`Initializing` becomes an umbrella. A consumer that knows only the old values
keeps seeing `Initializing` where it does today. A consumer that knows the new
values reads the refinement from the same `MatchedStatuses` list, which is
already multi-valued. `Pending`, `Progressing` and `Running` must not match
together.

### Minimal viable state

Each catalog definition states its minimal viable state inside its `running`
matcher. There is no new CRD field for it. The minimum is a property of the
workload type, and often of the workload's own spec, so it is written as a
CEL expression over the root object and the referenced pods.

Where the workload API already declares a minimum, the definition uses it:
Grove clique `minAvailable`, Ray worker group `minReplicas`, a gang
`PodGroup` `minMember`. Where it does not, the catalog sets one, following
three rules:

- Independent replicas (Deployment, StatefulSet, serving types): one ready
  replica.
- Sharded units (an LWS group, a multi-node Dynamo or LLMInferenceService
  replica): one complete unit, every pod of it ready.
- Training gangs (PyTorchJob, MPIJob, JobSet): every pod ready, because a
  partial world size cannot train.

The workload's `progressing` matcher is "some pod bound and not `Running`", so
it repeats the negation of the `running` expression.

### Holistic solution: CEL status mappings over references

A definition declares the workload's pods, and optionally a scheduler object,
as references. Its `pending` matcher checks that no pod is bound. Its
`running` matcher checks the minimal viable state. Its `progressing` matcher
checks that some pod is bound and the workload is not `Running`. The
definition owns the selector and the minimum, because both are
workload-specific. Nothing in the rule is hidden in Go code.

#### Mandatory changes

The changes are listed in dependency order. M1 and M2 are existing work items.
This KEP adds requirements to them but does not redesign them.

M1. CEL as a definition language (#330).

- CRD: `spec.expressionLanguage` (`jq` or `cel`) on `Karta`. Every expression
  field accepts CEL when the definition selects it. The workload is bound as
  `object`.
- The CEL environment must include the Kubernetes CEL list library, so that a
  definition can sum replica counts (`[...].sum()`), and the optional-field
  syntax (`object.?status.?x.orValue(d)`).
- Evaluation has a cost budget. A status expression iterates over every pod
  of a workload, and the LWS minimum iterates over pods per leader, so an
  unbounded expression must fail fast instead of stalling a reconcile.

M2. Resource references (#80, `docs/design/references/high-level-design.md`),
expressed in CEL.

- CRD: `structureDefinition.references[]`, each with `name`, `gvk`, and one of
  `lookup.nameExpression` or `list.matchLabels` / `list.matchExpressions`.
  `LabelValue.expression` is a CEL expression over `object`.
- Binding: each reference is exposed to CEL as `references.<name>`. A `list`
  is a list, possibly empty. A `lookup` that finds nothing leaves the key
  absent, so `has(references.<name>)` is the existence test.
- Bindings are lazy. A reference is resolved only when an expression that
  reads it is evaluated. A workload whose root CR already decides its status
  (for example `Completed`) costs no pod read.
- A validator gate rejects definitions that declare references unless the
  consumer opts in (`ErrReferencesNotSupported`), as the references design
  already proposes.

M3. Selector sourced from the workload. This extends M2.

Deployment, StatefulSet and similar types select their pods with their own
`spec.selector`. A label value per key cannot express that. The references
design lists whole-selector sourcing as out of scope. This KEP needs it.

```go
type ListReference struct {
    MatchLabels      map[string]LabelValue
    MatchExpressions []LabelSelectorRequirement

    // SelectorExpression is a CEL expression over the root object that returns
    // a metav1.LabelSelector. Mutually exclusive with MatchLabels and
    // MatchExpressions.
    // +kubebuilder:validation:Optional
    SelectorExpression *string `json:"selectorExpression,omitempty"`
}
```

M4. Status vocabulary.

```go
// +kubebuilder:validation:Enum=Initializing;Pending;Progressing;Running;Completed;Failed;Degraded;Undefined;Suspended;Suspending;Resuming
type ResourceStatus string

const (
    // PendingStatus indicates no pod of the component is bound to a node:
    // it is not scheduled yet and holds no node resources.
    PendingStatus ResourceStatus = "Pending"

    // ProgressingStatus indicates at least one pod of the component is bound
    // and its minimal viable state is not reached yet.
    ProgressingStatus ResourceStatus = "Progressing"

    // RunningStatus indicates the component reached its minimal viable state:
    // every required sub-component has its minimum ready pods.
    RunningStatus ResourceStatus = "Running"
)

type StatusMappings struct {
    // Initializing defines matchers for a component that is starting when the
    // definition cannot tell Pending from Progressing.
    Initializing []StatusMatcher `json:"initializing,omitempty"`

    // Pending defines matchers for the Pending status.
    // +kubebuilder:validation:Optional
    // +listType=atomic
    Pending []StatusMatcher `json:"pending,omitempty"`

    // Progressing defines matchers for the Progressing status.
    // +kubebuilder:validation:Optional
    // +listType=atomic
    Progressing []StatusMatcher `json:"progressing,omitempty"`

    // ... Running, Completed, Failed, Degraded, Suspended, Suspending, Resuming ...
}
```

- CRD: two optional properties under `statusMappings`. The enum marker is not
  part of the CRD schema, because no CRD field is typed `ResourceStatus`. The
  new `Running` meaning changes documentation only.
- `Entries()` adds the two entries. `matchStatus` appends `Initializing` when
  `Pending` or `Progressing` matched, so authors do not repeat matchers.

M5. SDK.

- `ResourceReader`, `references.Resolve`, `WithReferenceReader` and
  `WithReferences`, as in the references design.
- `Component.GetStatus` evaluates status matchers with the references bound.
- CEL compile-time checks: an expression that reads `references.<name>` must
  name a declared reference.

M6. Consumers and tooling.

- `kli get` and `kli describe` pass a reader. `kli get` is pod-free today. It
  stays pod-free for workloads whose root CR decides the status, thanks to
  lazy binding. For starting workloads it lists pods, one namespaced list
  shared by all workloads of a namespace.
- The `--phase` flag gains `Pending` and `Progressing`.
- The e2e recorder captures every referenced object next to the root object
  in each snapshot. That is a recording schema bump. Replay feeds the captured
  objects back through `WithReferences`, so the suite stays offline.

M7. Catalog.

- Each definition moves to CEL and declares a `pods` reference, its minimal
  viable state in `running`, and `pending` and `progressing` matchers. The
  per-workload table below lists what each needs.
- New e2e fixtures per definition: unschedulable (a `nodeSelector` to a
  missing node, or an impossible GPU request), quota-rejected, Kueue-queued
  where the type supports Kueue, and for multi-component types one with a
  required component unschedulable (must be `Progressing`, not `Running`). The
  recorder must keep recording until the scheduler's verdict appears.

#### Example: LeaderWorkerSet, minimal viable state

`Running` needs one complete group: a ready leader, and as many ready pods
with the leader's group index as the group size.

```yaml
spec:
  expressionLanguage: cel
  structureDefinition:
    references:
      - name: pods
        gvk: {group: "", version: v1, kind: Pod}
        list:
          matchLabels:
            leaderworkerset.sigs.k8s.io/name: {expression: object.metadata.name}
    rootComponent:
      statusDefinition:
        statusMappings:
          pending:
            - byExpression:
                expression: >-
                  !references.pods.exists(p, !has(p.metadata.deletionTimestamp) &&
                    has(p.spec.nodeName))
                expectedResult: "true"
          running:
            # some leader whose whole group is ready
            - byExpression:
                expression: >-
                  references.pods.exists(l,
                    l.metadata.labels['leaderworkerset.sigs.k8s.io/worker-index'] == '0' &&
                    l.status.?conditions.orValue([]).exists(c, c.type == 'Ready' && c.status == 'True') &&
                    references.pods.filter(p,
                      p.metadata.labels['leaderworkerset.sigs.k8s.io/group-index'] ==
                        l.metadata.labels['leaderworkerset.sigs.k8s.io/group-index'] &&
                      p.status.?conditions.orValue([]).exists(c, c.type == 'Ready' && c.status == 'True')
                    ).size() >= object.spec.leaderWorkerTemplate.?size.orValue(1))
                expectedResult: "true"
          progressing:
            # some pod bound, and no complete group ready
            - byExpression:
                expression: >-
                  references.pods.exists(p, !has(p.metadata.deletionTimestamp) &&
                    has(p.spec.nodeName)) &&
                  !references.pods.exists(l, ...same as running...)
                expectedResult: "true"
```

The leader pod carries the same `group-index` label as its workers, so the
filter counts the leader once. A group of size 4 with the leader and one
worker ready is `Progressing`, not `Running`.

#### Example: JobSet, queue and training gang

```yaml
references:
  - name: pods
    gvk: {group: "", version: v1, kind: Pod}
    list:
      matchLabels:
        jobset.sigs.k8s.io/jobset-name: {expression: object.metadata.name}
  - name: queue
    gvk: {group: kueue.x-k8s.io, version: v1beta1, kind: Workload}
    list:
      matchLabels:
        kueue.x-k8s.io/job-uid: {expression: object.metadata.uid}
statusMappings:
  pending:
    # queued by Kueue and not admitted yet
    - byExpression:
        expression: >-
          references.queue.exists(w, !w.status.?conditions.orValue([])
            .exists(c, c.type == 'Admitted' && c.status == 'True'))
        expectedResult: "true"
    # admitted or not queued, and no pod bound yet
    - byExpression:
        expression: >-
          !object.status.?conditions.orValue([]).exists(c,
            c.type in ['Completed', 'Failed', 'Suspended'] && c.status == 'True') &&
          !references.pods.exists(p, !has(p.metadata.deletionTimestamp) &&
            p.status.phase in ['Pending', 'Running'] && has(p.spec.nodeName))
        expectedResult: "true"
  running:
    # training gang: every replicated job has all its pods ready
    - byExpression:
        expression: >-
          object.spec.replicatedJobs.all(r,
            object.status.?replicatedJobsStatus.orValue([]).exists(s,
              s.name == r.name && s.ready + s.succeeded >= r.replicas))
        expectedResult: "true"
```

JobSet counts a child Job ready when all its pods are ready, so its minimum
is readable from the CR. The pods are needed only for `pending` and
`progressing`.

#### Example: Deployment, selector from the workload (M3)

```yaml
references:
  - name: pods
    gvk: {group: "", version: v1, kind: Pod}
    list:
      selectorExpression: object.spec.selector
running:
  - byExpression:
      expression: object.status.?readyReplicas.orValue(0) >= 1
      expectedResult: "true"
```

A rollout selects pods of both the old and the new ReplicaSet. During a
rollout the old pods are ready, so the workload stays `Running`. The
refinement applies to the first rollout.

#### Per-workload references and minimal viable state

The workload-level pod labels come from upstream controllers. Entries marked
"verify" must be checked against the operator version recorded under
`test/e2e/recorded_data` before the catalog PR. "CR" means the minimum is
readable from the root object. "Pods" means it needs the `pods` reference.

| Definition | `pods` selector | Minimal viable `Running` | Source | Extra reference |
|---|---|---|---|---|
| Pod | None. The object is the pod. | The pod is ready | CR | None |
| Agent Sandbox | None. The CR mirrors `PodScheduled`. | `Ready=True` | CR | None |
| CronJob | None | A run was scheduled (unchanged) | CR | None |
| Deployment | `selectorExpression: object.spec.selector` | One ready replica | CR | None |
| StatefulSet | `selectorExpression: object.spec.selector` | One ready replica | CR | None |
| Job | `batch.kubernetes.io/job-name` | One ready pod | CR | Kueue `Workload` |
| JobSet | `jobset.sigs.k8s.io/jobset-name` | Every replicated job fully ready | CR | Kueue `Workload` |
| PyTorchJob | `training.kubeflow.org/job-name` | Every replica of every type ready | Pods | Kueue `Workload` |
| MPIJob | `training.kubeflow.org/job-name` | Launcher running and every worker ready | Pods, or the `Running` condition (verify) | Kueue `Workload` |
| LeaderWorkerSet | `leaderworkerset.sigs.k8s.io/name` | One complete group ready | Pods | None |
| RayCluster | `ray.io/cluster` | Head ready and every worker group at `minReplicas` ready | Pods | Kueue `Workload` |
| RayJob | `ray.io/cluster` from `status.rayClusterName`, with `ray.io/node-type` present (excludes the submitter pod) | Job `RUNNING` and the cluster at its minimum | CR and pods | Kueue `Workload` |
| RayService | `ray.io/cluster` from the pending, else the active, cluster name | `Ready=True` (serving) | CR | None |
| Knative Service | `serving.knative.dev/service` | `Ready=True` (one ready revision pod serving) | CR | None |
| KServe InferenceService | `serving.kserve.io/inferenceservice` | Predictor ready, transformer ready if declared | CR | None |
| LLMInferenceService (both versions) | Workload label (verify) | Router ready if declared, and one complete workload replica | Pods | None |
| Milvus | `app.kubernetes.io/instance` (verify) | Standalone pod ready, or one ready pod per cluster component | CR (`componentsDeployStatus`) or pods | None |
| NIMService | Workload label (verify) | One ready replica | CR | None |
| NIMCache | Caching job's pod label (verify) | Caching job pod running (unchanged) | CR | None |
| Dynamo (both versions) | `nvidia.com/dynamo-graph-deployment-name` (verify) | Frontend and one complete replica of each worker service ready | CR (`services`) for single-node, pods for multi-node | None |
| Grove PodCliqueSet | Workload label (verify) | One replica with every clique at `minAvailable` | CR (`availableReplicas >= 1`) | `PodClique` list as an alternative |

#### Why this meets every requirement

- R1: `pending` and `progressing` read binding directly from the pods.
- R2: every definition gets a selector.
- R3: `running` encodes each workload's minimal viable state, per component.
- R4: unschedulable and gated pods are unbound. Quota-rejected pods are never
  created. Either way no pod is bound until the first one is. Queued workloads
  are matched through the Kueue `Workload`, before any pod exists.
- R5: the recorder captures the referenced objects, so replay proves the
  states offline.

### Partial option A: root-CR matchers only

The catalog definitions gain `pending` and `progressing` matchers, and
tighter `running` matchers, where the root CR carries the signal. Definitions
without a signal keep `initializing`. Needs only M4. Works with jq
definitions.

```yaml
# core-pod-v1
pending:
  - byPhase: Pending
    byExpression:
      expression: '[(.status.conditions // [])[] | select(.type == "PodScheduled" and .status == "True")] | length == 0'
      expectedResult: "true"
progressing:
  - byPhase: Pending
    byConditions: [{type: PodScheduled, status: "True"}]
```

CRD changes: M4 only.

Why it is partial. It meets R1 for 5 of 23 definitions and partially for 2
more. It meets R3 only where the CR counts readiness per component.

| Definition | What option A gives | Requirement missed and why |
|---|---|---|
| Pod | Full split from `PodScheduled` | None |
| Agent Sandbox | Full split from the mirrored `PodScheduled` | None |
| CronJob | `Pending` before the first tick | None. No pods exist in that state. |
| RayJob, RayCluster | Split from `HeadPodReady`. Reason `Unknown` means the head pod has no `PodReady` condition, which the kubelet adds only after binding. | R1: workers bound before the head read `Pending` for a moment. R3: worker group minimums are not on the CR. R4: Kueue-queued reads `Suspended`. |
| RayService | Nothing for R1. `Running` on `Ready=True` is already minimal viable. | R1: `Ready=False` with reason `Initializing` hides both cases. |
| Knative Service | `Progressing` when `ConfigurationsReady=True` and only networking is left | R1: `RevisionMissing` is ambiguous. An unschedulable revision is expected to surface as `ConfigurationsReady=False`, which maps to `Failed`. |
| Deployment | `Pending` on `ReplicaFailure=True` with no pod created (quota). `Running` on one ready replica. | R1: unschedulable pods are invisible. `Progressing=True` holds until `ProgressDeadlineExceeded` turns it `Failed` after 600s by default. |
| StatefulSet | `Running` on one ready replica | R1: only replica counts. |
| Job | `Running` on one ready pod (unchanged) | R1: `active` counts unbound pods too. R4: Kueue-queued reads `Suspended`. |
| JobSet | `Running` when every replicated job is fully ready | R1: `active` counts unbound pods too. R4: Kueue. |
| PyTorchJob | Nothing | R1: `Created=True` only means pods were submitted. R3: `replicaStatuses.active` counts running, not ready, pods per type. R4: Kueue. |
| MPIJob | `Running` from the `Running` condition, if it requires every worker (verify) | R1: `Created=True` only means pods were submitted. R4: Kueue. |
| LeaderWorkerSet | Nothing | R1: `Progressing=True` with reason `GroupsProgressing` throughout. R3: `readyReplicas` and `Available=True` count a group ready off its leader (#228). |
| LLMInferenceService | Nothing | R1: `Ready=False` reasons copy the Deployment's (`MinimumReplicasUnavailable`). R3: multi-node groups have the LWS gap. |
| KServe InferenceService | Nothing for R1 | R1: no starting mapping exists. `Ready=Unknown` covers both cases. |
| Dynamo | `Running` from per-service `readyReplicas` for single-node services | R1: per-service replica counts only. R3: multi-node replicas have the LWS gap. |
| Grove PodCliqueSet | `Running` on `availableReplicas >= 1` | R1: `availableReplicas` only. Scheduling counts live on child `PodClique` objects. |
| NIMService, NIMCache | Nothing for R1 | R1: state `NotReady` with a rollout message only. |
| Milvus | `Running` from `componentsDeployStatus` (verify) | R1: `status: Pending` covers everything. The rest is in condition messages. |

R5 is reachable for the definitions it covers, with new fixtures.

### Partial option B: a pod rule in the SDK

The consumer passes the workload's pods to the SDK. When the root matched
`Initializing` and no `pending` or `progressing` matcher decided, the SDK
applies one rule in Go:

| Pods passed (non-terminal, not deleting) | Result |
|---|---|
| None bound, or none at all | `Pending` |
| At least one bound | `Progressing` |

```go
// WithPods supplies the live pods of the workload. When set, GetStatus refines
// Initializing into Pending or Progressing from their scheduling state.
func WithPods(pods []corev1.Pod) StatusOption
```

CRD changes: M4, plus possibly an opt-out on `StatusDefinition`
(`podRefinement: Enabled | Disabled`) for definitions whose pods do not reflect
progress. No CEL and no references.

Because `Progressing` starts at the first bound pod, the rule splits the two
statuses without knowing the minimum. It meets R1 for every definition whose
root reports `Initializing` while starting. Why it is still partial:

- It cannot define `Running`. A generic rule does not know which components a
  workload needs or their minimums, so `running` stays as option A leaves it.
  This breaks R3 for LWS, PyTorchJob, Ray, multi-node Dynamo and
  LLMInferenceService.
- It cannot see causes outside pods. A Kueue-queued workload has no pods and
  its root matches `Suspended`, so the rule never runs. This breaks R4 for
  every Kueue-capable type.
- It refines `Initializing` only. Definitions whose root reports `Undefined`
  while starting never reach the rule. This breaks R1 and R2 for those types.
- It does not know which pods belong to the workload. The definition has no
  workload-level selector, so every consumer must attribute pods itself.
- The rule is fixed in Go. A definition cannot adapt it, and the Go API is
  superseded once references land.

| Definition | What option B gives | Requirement missed and why |
|---|---|---|
| Pod, Agent Sandbox, CronJob | Same as option A | None |
| Deployment, StatefulSet | Split for the first rollout | None, given correct attribution by the consumer |
| Job | Split | R4: Kueue-queued stays `Suspended`. |
| JobSet | Split | R4: Kueue. |
| PyTorchJob | Split | R3: no per-type readiness for `running`. R4: Kueue. |
| MPIJob | Split | R4: Kueue. |
| LeaderWorkerSet | Split | R3: `Running` still counts a group ready off its leader (#228). |
| RayCluster | Nothing | R1, R2: the root reports `Undefined` while starting, so the rule never runs. R3: worker group minimums. R4: Kueue. |
| RayJob | Split while `jobDeploymentStatus` is `Initializing` | R1: the root reports `Undefined` while `jobDeploymentStatus` is `Running` without `jobStatus`. R3, R4: as RayCluster. |
| RayService | Split | None for R1. Pods of the active and the pending cluster are mixed during an upgrade. |
| Knative Service | Split | R1: an unschedulable revision maps to `Failed` on the root before the rule runs. |
| KServe InferenceService | Nothing | R1, R2: the root reports `Undefined` while starting. |
| LLMInferenceService | Split | R3: multi-node groups have the LWS gap. |
| Dynamo | Split | R3: multi-node replicas have the LWS gap. |
| Grove PodCliqueSet | Split. Gated pods read `Pending`. | None, given correct attribution. |
| NIMService | Split once the state is `NotReady` | R1: during model caching (`PVC-Created`) the root reports `Undefined`. |
| NIMCache, Milvus | Split | None for R1, given correct attribution. Milvus dependency pods (etcd, MinIO) are part of the workload and count as bound. |

R5 needs the recorder to capture pods, the same schema bump as M6.

### Comparison

| Requirement | Option A | Option B | Holistic (CEL references) |
|---|---|---|---|
| R1 not bound vs bound | Pod, Agent Sandbox, CronJob, RayJob, RayCluster. Knative and Deployment partially. | Every type whose root reports `Initializing` while starting | Yes |
| R2 every definition | No (7 of 23) | No (types starting in `Undefined` are missed) | Yes |
| R3 minimal viable `Running` | Only where the CR counts readiness per component | Same as option A | Yes |
| R4 unschedulable, quota, queue | Unschedulable for 5 types. Quota for Deployment. No queue. | Unschedulable and quota. No queue. | All three |
| R5 recorded proof | New fixtures | New fixtures, recorder captures pods | New fixtures, recorder captures references |
| CRD changes | M4 | M4, maybe an opt-out | M1, M2, M3, M4 |
| Dependencies | None | None | CEL (#330), references (#80) |

### Recommendation and rollout

1. M4 and option A ship first. They are additive, need no consumer work, and
   fix Pod, Agent Sandbox, Ray and CronJob for `Pending` and `Progressing`,
   and the CR-readable types for `Running`.
2. The holistic path follows M1, then M2 and M3, then M5, M6 and M7. Each
   catalog definition moves to CEL with its `pods` reference and its minimal
   viable `running` in its own PR, with its fixtures. LeaderWorkerSet goes
   first, since #228 is open.
3. Option B is an interim choice, not a step on the holistic path. It covers
   R1 for most types early, at the cost of a Go API that references supersede.
   See Open questions.

## Compatibility and migration

- CRD: every change is additive. M1 adds `expressionLanguage`, M2 and M3 add
  `references`, and M4 adds two `statusMappings` fields. No stored Karta
  becomes invalid. Rolling back the CRD drops the new fields from Kartas that
  use them.
- Definitions: a jq definition behaves exactly as today. A CEL definition
  without references behaves like its jq equivalent.
- Consumers: `Initializing` is still reported wherever it is today. Consumers
  that switch exhaustively over `ResourceStatus` must accept two new values.
  Consumers that pick one status from `MatchedStatuses` should prefer the
  refinement. A consumer that evaluates definitions with references must
  supply a reader or pre-resolved values, or reject them through the
  validator gate.
- Running: the minimal viable `running` matchers move some workloads from
  `Running` to `Progressing`: an LWS group with unready workers, a training
  gang with unready ranks, a Ray cluster below its worker minimums. That is
  the #228 fix, and it is called out in the release notes. The reverse also
  happens: a StatefulSet or Deployment with one of three replicas ready moves
  from `Initializing` to `Running`.
- Kueue: the holistic path reports a queued workload as `Pending`, not
  `Suspended`. That changes an existing state and is called out in the
  release notes. See Open questions.

## Test plan

- API: `Entries()` covers the new statuses. `matchStatus` adds the
  `Initializing` umbrella.
- Catalog unit tests: for every definition, at most one of `Pending`,
  `Progressing` and `Running` matches on any recorded snapshot.
- Recorder: waits for a predicate, not only for the first snapshot in the
  target state. Captures referenced objects with each snapshot (schema bump).
- Fixtures per definition: unschedulable, quota-rejected, Kueue-queued where
  supported, a required component unschedulable for multi-component types,
  and the existing happy paths, which must now show `Progressing` before
  `Running`. For LeaderWorkerSet: a group of size 4 with one worker
  unschedulable must stay `Progressing`.
- SDK: reference resolution, lazy binding, the validator gate, and CEL compile
  checks for undeclared references.

## Risks and alternatives

- Cost. Listing pods for every starting workload adds API load. Lazy binding
  and one shared namespaced list bound it. Consumers choose their reader, for
  example an informer cache. The LWS minimum is quadratic in the pod count;
  the CEL cost budget bounds it.
- Stuck while bound. `ImagePullBackOff`, and a partial gang whose remaining
  pods never fit, read `Progressing`, which suggests waiting. With pods
  referenced, a follow-up can map long stalls to `Degraded` or surface the pod
  reason.
- Repetition. `progressing` repeats the negation of `running`. A wrong edit to
  one and not the other makes them overlap or leave a gap. Tests over the
  recordings catch it for the shipped catalog.
- Minimums are workload-specific. A wrong minimum reports `Running` too early
  or too late. Each minimum is proven by fixtures.
- Alternatives considered:
  - A dedicated component field for the minimum (`minReadyExpression` on
    `ComponentDefinition`), evaluated by the SDK with the existing pod
    selectors. Rejected for now: it adds a CRD field and Go logic, while the
    same minimum fits in the `running` matcher.
  - `Running` only at the full desired count. Rejected: a serving workload
    that already serves traffic would not read `Running`.
  - Reporting `Degraded` between the minimum and the desired count. Rejected:
    every normal scale-up and rollout would show `Degraded`.
  - Treating a partial gang as `Pending`. Rejected: it holds node resources,
    and to an external viewer `Pending` means not scheduled.
  - Replacing `Initializing` outright, as the issue first proposed. Rejected
    for now: it breaks consumers, and only the holistic path can fill the new
    buckets for every definition.
  - Events (`FailedScheduling`). Rejected: they expire and are rate-limited.

## Open questions

- Should Kueue-queued workloads move from `Suspended` to `Pending` (holistic
  path), or report both?
- Should option B ship as an interim, given that references supersede it?
- Should CEL definitions support named variables (`spec.variables`), so the
  minimal viable expression is written once and reused by `running` and
  `progressing`?
- Can a platform override a catalog minimum (for example 1 leader and 1 worker
  for an LWS whose workers are independent), or does it copy the definition?
- Does option A ship for jq definitions only, or wait for M1 so the catalog is
  migrated once?

## Appendix: catalog gaps found during research

These are independent of this KEP and can go straight to PRs.

- PyTorchJob and MPIJob keep matching `Initializing` after they start or finish,
  because `Created=True` never clears. Recordings show
  `[Initializing, Running]` and `[Initializing, Completed]`.
- RayJob reports `Undefined` while `jobDeploymentStatus` is `Running` and
  `jobStatus` is not yet set.
- RayCluster and KServe InferenceService have no starting mapping and report
  `Undefined` until ready.
- NIMService reports `Undefined` in state `PVC-Created`.

## History

- 2026-10-18: Proposal opened.
