<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright (c) 2026 NVIDIA Corporation -->

# KEP-0002: Pending, Progressing, and minimal viable Running

- Status: provisional
- Authors: @rogirun
- Issue: https://github.com/dsx-ai-factory/workload-map/issues/347
- Created: 2026-10-18
- Last updated: 2026-10-20

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
viewer, `Pending` must therefore mean "not scheduled, holds nothing", the
meaning queueing systems give it. The pod phase `Pending` is broader: it also
covers a bound pod that is pulling its image.

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

### User stories

- A platform admin sees a workload starting for an hour. `Pending` tells them
  it waits for quota or a queue, so they act on capacity. `Progressing` tells
  them it already holds GPUs, so they look at the pods.
- A serving platform routes traffic only to `Running` inference workloads. An
  LWS whose workers are unschedulable must not read `Running`.
- A training user submits a 64-rank PyTorchJob. They need to know whether the
  gang is still queued, partly bound, or complete.
- A cost dashboard charges a team for workloads that hold node resources. It
  needs `Progressing` and `Running` apart from `Pending`.

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

Research covered the 23 catalog definitions and all 68 recorded e2e flows.

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
- A Kubernetes client in status evaluation. `pkg/resource` stays
  client-free. Reads go through the reader the consumer supplies to
  `pkg/references`, as in the references design.
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

The three statuses describe startup only. A change after the workload reached
`Running` (a rollout, a scale-up, a JobSet restart) keeps it `Running` as long
as the minimal viable state holds. If it drops below the minimum, it reads
`Progressing` again. Reporting "running, but changing" is out of scope. See
Open questions.

### Matcher guards

Every `pending`, `progressing` and `running` matcher in the catalog applies
these guards, so a stale or vanishing object does not report a startup state:

- `status.observedGeneration` equals `metadata.generation` where the workload
  reports it. A snapshot taken before the controller observed a spec change
  carries old counts and resolves to `Undefined`.
- The workload is not being deleted (`metadata.deletionTimestamp` unset).
- Pods being deleted do not count as bound or ready.
- A workload scaled to zero desired replicas, or a paused Deployment, matches
  none of the three.

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
- Bindings are lazy per reference. A reference is resolved only when an
  expression that reads it is evaluated. A workload whose root CR already
  decides its status (for example `Completed`) costs no pod read. This is
  stricter than the references design, which resolves every reference on the
  first accessor call that needs one.
- A validator gate rejects definitions that declare references unless the
  consumer opts in (`ErrReferencesNotSupported`), as the references design
  already proposes.

M3. Selector sourced from the workload. This extends M2.

In M2, a `list` reference fixes the label keys when the definition is
written. Only the values can come from the workload. That works when the
controller stamps every pod with a known label carrying the workload's name:

```yaml
# JobSet: the controller adds jobset.sigs.k8s.io/jobset-name to every pod.
# The key is known in advance; only the value comes from the workload.
list:
  matchLabels:
    jobset.sigs.k8s.io/jobset-name: {expression: object.metadata.name}
```

Deployment and StatefulSet do not stamp such a label. Their pods carry the
labels the user wrote in the pod template, and the workload finds them
through its own `spec.selector`. The keys differ from one workload to the
next:

```yaml
# Deployment A
kind: Deployment
metadata: {name: llama-serving}
spec:
  selector:
    matchLabels: {app: llama, tier: inference}
  template:
    metadata:
      labels: {app: llama, tier: inference}
---
# Deployment B
kind: Deployment
metadata: {name: embedder}
spec:
  selector:
    matchLabels: {team: search}
    matchExpressions:
      - {key: model, operator: In, values: [bge, e5]}
  template:
    metadata:
      labels: {team: search, model: bge}
```

A single `apps-deployment-v1` definition must find the pods of both. M2
cannot express that:

- `matchLabels: {app: {expression: ...}}` finds Deployment A but not B, which
  has no `app` label.
- No pod label carries the Deployment's name, so there is no fixed key to
  match on.
- Pods are owned by a ReplicaSet, not by the Deployment, so the owner
  reference does not lead to the workload either.

`selectorExpression` returns the whole selector from the workload. The
consumer converts it to a `metav1.LabelSelector` and lists pods with it:

```yaml
# apps-deployment-v1, apps-statefulset-v1
references:
  - name: pods
    gvk: {group: "", version: v1, kind: Pod}
    list:
      selectorExpression: object.spec.selector
```

| Workload | `selectorExpression` result | `List` the consumer issues |
|---|---|---|
| Deployment A | `{matchLabels: {app: llama, tier: inference}}` | `app=llama,tier=inference` |
| Deployment B | `{matchLabels: {team: search}, matchExpressions: [{key: model, operator: In, values: [bge, e5]}]}` | `team=search,model in (bge,e5)` |

The expression can also build a selector, when the controller stamps a
known label but the definition needs a narrower set:

```yaml
# RayJob: the cluster's pods, without the submitter pod
list:
  selectorExpression: >-
    {'matchLabels': {'ray.io/cluster': object.status.rayClusterName},
     'matchExpressions': [{'key': 'ray.io/node-type', 'operator': 'Exists'}]}
```

The same RayJob selector also fits M2's `matchLabels` and `matchExpressions`.
`selectorExpression` is required only when the keys come from the workload.

API change:

```go
type ListReference struct {
    MatchLabels      map[string]LabelValue
    MatchExpressions []LabelSelectorRequirement

    // SelectorExpression is a CEL expression over the root object that returns
    // a label selector in the shape of metav1.LabelSelector. Mutually exclusive
    // with MatchLabels and MatchExpressions.
    // +kubebuilder:validation:Optional
    SelectorExpression *string `json:"selectorExpression,omitempty"`
}
```

Rules:

- Exactly one of `selectorExpression`, or `matchLabels` and
  `matchExpressions`, is set. The validator enforces it.
- The result must convert to a `metav1.LabelSelector`. A result that does not
  convert fails resolution with an error, not an empty list.
- An empty selector is rejected. It would match every pod in the namespace.
- The selector applies in the workload's namespace only, as for every
  reference.

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

The examples omit the matcher guards for brevity.

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

- R1 met: Pod and Agent Sandbox (`PodScheduled`), CronJob (no pods before the
  first tick), RayJob and RayCluster from `HeadPodReady`. Reason `Unknown`
  means the head pod has no `PodReady` condition, which the kubelet adds only
  after binding. Workers bound before the head still read `Pending` for a
  moment.
- R1 partial: Deployment reads `Pending` on `ReplicaFailure=True` with no pod
  created (quota), but unschedulable pods are invisible. Knative reads
  `Progressing` once only networking is left, but `RevisionMissing` is
  ambiguous.
- R1 missed, counts only: StatefulSet, Job, JobSet, Dynamo, Grove, Milvus.
  Their counts (`active`, `readyReplicas`, `availableReplicas`) include
  unbound pods or cover only ready ones.
- R1 missed, conditions hide it: PyTorchJob and MPIJob (`Created=True` only
  means pods were submitted), LeaderWorkerSet (`Progressing=True` throughout),
  LLMInferenceService, RayService, KServe InferenceService, NIMService and
  NIMCache.
- R3 missed: LeaderWorkerSet counts a group ready off its leader (#228).
  PyTorchJob counts running, not ready, pods per type. RayCluster worker
  group minimums are not on the CR. Multi-node Dynamo and LLMInferenceService
  replicas have the LWS gap.
- R4 missed: every Kueue-capable type reads `Suspended` while queued.

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

Per workload:

- R1 met, given correct attribution by the consumer: Pod, Agent Sandbox,
  CronJob, Deployment and StatefulSet (first rollout), Job, JobSet,
  PyTorchJob, MPIJob, LeaderWorkerSet, RayService, LLMInferenceService,
  Dynamo, Grove (gated pods read `Pending`), NIMCache and Milvus. Milvus
  dependency pods (etcd, MinIO) count as bound.
- R1 and R2 missed, the root starts in `Undefined`: RayCluster, KServe
  InferenceService, RayJob while `jobDeploymentStatus` is `Running` without
  `jobStatus`, NIMService in state `PVC-Created`.
- R1 missed, another status wins first: Knative maps an unschedulable
  revision to `Failed` before the rule runs.
- R3 missed: as option A.
- R4 missed: as option A.

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

Exit criteria:

| Milestone | Done when |
|---|---|
| M4 and option A | `Pending` and `Progressing` are in the API, `kli get --phase` accepts them, and every option A definition has a recorded unschedulable flow. |
| Holistic path, per definition | The definition uses CEL and a `pods` reference, its fixtures prove `Pending`, `Progressing` and `Running`, and the co-match test passes on its recordings. |
| Holistic path, complete | Every catalog definition is migrated and no definition reports `Initializing` alone. |
| Deprecating `Initializing` | Out of scope. A later KEP may propose it once the holistic path is complete and consumers have moved to the refinements. |

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

### Enablement and rollback

Every part is opt-in per definition. A definition opts in with
`pending:`/`progressing:` (M4), with `expressionLanguage: cel` (M1), and with
`references` (M2). There is no global switch. To roll a workload type back,
apply the previous definition. To keep CEL but drop the pod reads, apply a
CEL definition without `references` and without the matchers that read them.
It then behaves as option A: the root CR decides alone. Removing only the
`references` block fails the compile check in M5.

### Upgrade and downgrade

- Upgrade order: CRD, then SDK consumers (operator, `kli`, platforms), then
  catalog definitions. A definition that uses the new fields must not be
  applied before the CRD that knows them.
- Downgrade in reverse order: catalog first, then consumers, then the CRD.
  Downgrading the catalog only (same CRD) restores today's statuses exactly,
  including Kueue-queued workloads reading `Suspended`.
- Downgrading the CRD while CEL definitions are stored prunes
  `expressionLanguage` and `references`. Their expressions are then read as
  jq and fail, so those workloads report `Undefined`. This is why the catalog
  goes first.
- For jq definitions with option A matchers, a CRD downgrade prunes
  `pending` and `progressing` only. They fall back to their `initializing`
  matchers. The catalog keeps `initializing` matchers next to the refinements
  for this reason. `matchStatus` reports `Initializing` once when both match.

### Version skew

| Skew | Behavior |
|---|---|
| New jq definition (option A), old SDK consumer | The consumer ignores the unknown `pending` and `progressing` fields and reports `Initializing` from the definition's `initializing` matchers. No error. |
| New CEL definition, consumer without M1 | The consumer ignores `expressionLanguage`, evaluates the CEL expressions as jq, and fails. The catalog moves a definition to CEL only after the supported consumers have M1. |
| Old consumer receives `Pending` or `Progressing` in `MatchedStatuses` (for example through a platform API built on the SDK) | It still finds `Initializing` in the same list. A consumer that rejects unknown values must be upgraded first. |
| New SDK, consumer without a reference reader | The validator gate rejects definitions with references (`ErrReferencesNotSupported`), or the consumer passes pre-resolved values. It never reports a refinement from missing data. |
| New `kli`, old catalog | `--phase Pending` and `--phase Progressing` return no workloads until the catalog is upgraded. |

### RBAC

The Karta operator reconciles Karta objects only and needs no new rule. Every
consumer that evaluates a definition with references reads the referenced
objects with its own identity:

- `list` on `pods` in the workload's namespace, for the `pods` reference.
- `list` on `workloads.kueue.x-k8s.io`, where a definition references the Kueue
  `Workload`.

A `kli` user who can list the workload but not its pods gets `Initializing`,
not an error. Platforms that run the SDK in a controller add these rules to
their own role. The references design owns the general rule; this KEP adds the
two resources above.

## Monitoring and troubleshooting

- A reference that fails to resolve (forbidden, timeout, invalid selector)
  leaves the refinement unmatched, so the workload reads `Initializing`, and
  the SDK returns the resolution error to the consumer. It never guesses
  `Pending`.
- A CEL expression over its cost budget fails the same way, with an error
  naming the definition and the matcher.
- `kli describe` shows which matcher produced each status, the number of pods
  bound and ready per component, and the pod reason for unbound pods
  (`PodScheduled=False` reason and message), so "why is this `Progressing`"
  has an answer.
- Overlapping `pending`, `progressing` and `running` matchers are caught by
  the co-match replay test in CI, not at runtime. Runtime metrics for
  resolution errors and evaluation cost belong to the metrics exporter KEP.

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
- Guards: replay snapshots with a stale `observedGeneration`, a deletion
  timestamp, and zero desired replicas match none of the three statuses.
- Skew: a definition with the new fields evaluated by the previous SDK release
  reports `Initializing`.
- Infrastructure: the recording cluster needs Kueue with a `ClusterQueue`
  smaller than one fixture, a `ResourceQuota` for the quota fixtures, and a
  node label no node carries for the unschedulable fixtures. GPU-free fixtures
  are enough. An impossible extended-resource request stands in for GPUs.

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
- Should a change after `Running` (a rollout, a scale-up, a restart) be
  visible, for example as `Running` plus a separate updating signal, without
  breaking the rule that `Pending`, `Progressing` and `Running` are exclusive?
  Native signals exist for several types: Deployment `Progressing`, RayService
  `UpgradeInProgress`, Grove and LWS `UpdateInProgress`, JobSet
  `RestartingJobSet`.

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
- 2026-10-18: Decided that any bound pod makes a workload `Progressing`, so a
  partially bound gang is `Progressing`, not `Pending`.
- 2026-10-18: Decided that `Running` is the minimal viable state, expressed in
  the `running` matcher with no new CRD field. Between the minimum and the
  desired count the workload is `Running`; `Degraded` is unchanged.
- 2026-10-18: Decided to keep `Initializing` as an umbrella reported with
  either refinement, instead of deprecating it.
- 2026-10-19: Added user stories, matcher guards, the startup-only scope of
  the three statuses, compatibility detail (enablement, upgrade, skew, RBAC),
  monitoring, and exit criteria. Collapsed the per-workload option tables.
