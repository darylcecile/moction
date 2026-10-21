# Moction runtime specification

**Status:** Proposed; no runtime is implemented by this document

**Baseline:** GitHub.com Actions documentation reviewed 2026-10-15 to 2026-10-16

**Audience:** Implementers of the runtime, CLI, API, execution backends, and conformance suite

**Related:** [Research and sources](moction-runtime-research.md) | [Open questions](moction-runtime-open-questions.md)

## Purpose

Moction runs GitHub Actions locally for integration testing and exposes the
observable behavior needed to test clients of GitHub Actions. Existing workflows
must not require conversion into a Moction-specific workflow language. A developer
must be able to start an instance, register a repository, execute a workflow,
inspect it with the CLI or `gh`, and destroy the instance without changing their
normal GitHub authentication or repository configuration.

The long-term target is **full, versioned Actions compatibility**, including
orchestration and APIs, not merely successful execution of some `run:` steps.
The first useful release is deliberately narrower than that target. It must
publish its missing capabilities and must not be described as full parity.

An unconditional claim of full offline parity is impossible: a Linux container
cannot supply a macOS kernel, sign an OIDC token as GitHub, or reproduce a paid
cloud service without that service or an explicit substitute. This specification
keeps those requirements visible instead of silently excluding them. The
[product questions](moction-runtime-open-questions.md#q1-what-does-the-full-parity-release-claim-mean)
identify the decision needed before making a full-parity release claim.

All commands, package names, schemas, module paths and interfaces below are
**proposed implementation contracts**, except where explicitly identified as
executed research or referenced upstream interfaces. MUST, MUST NOT and SHOULD
are normative.

## Navigation

- [Compatibility contract](#compatibility-contract)
- [Architecture and language decision](#architecture-and-language-decision)
- [Configuration and first-run experience](#configuration-and-first-run-experience)
- [Workflow semantics](#workflow-semantics)
- [Execution and runner integration](#execution-and-runner-integration)
- [Public APIs and GitHub clients](#public-apis-and-github-clients)
- [Artifacts, cache and output streams](#artifacts-cache-and-output-streams)
- [Mocks, deterministic testing and SDK](#mocks-deterministic-testing-and-sdk)
- [Persistence and recovery](#persistence-and-recovery)
- [Security model](#security-model)
- [Performance specification](#performance-specification)
- [Conformance and delivery](#conformance-and-delivery)

## Compatibility contract

### Independent dimensions

Each compatibility bundle MUST declare and test the following dimensions:

| Dimension | Required observable behavior |
| --- | --- |
| Language | Workflow/action YAML, schemas, validation errors, expressions, contexts, anchors, reusable workflows and dependency lockfiles |
| Orchestration | Event matching, refs, DAGs, matrices, permissions, concurrency, environments, timeouts, cancellation, retries and re-runs |
| Execution | Shells, JavaScript/composite/container actions, pre/post hooks, services, command files, outputs, logs and cleanup |
| Public services | Actions REST, required GraphQL/Git/checks/deployment surfaces, tokens, pagination, errors, redirects and URLs |
| Runner services | Provisioning, messages, leases, action resolution, results, timelines, log streaming and blob protocols |
| Environment | OS, architecture, image/tool versions, filesystem, network, daemon access, hardware and machine snapshots |
| External trust | Issuers, attestations, cloud resources, organization policy, billing and external services |

A feature being parsed, mocked, or accepted by an API is not proof that it works.
For example, accepting `environment:` without withholding secrets until approval
is a compatibility failure, not partial environment support.

### Bundles and claims

A compatibility bundle is an immutable, content-addressed manifest containing:

- GitHub product target (`dotcom` or a specific GHES release), documentation date,
  supported REST versions, schema/OpenAPI revisions and patches.
- Runner release, source correspondence, binary digests, private-protocol fixture
  digests, parser/evaluator versions and patches.
- Qualified action SHAs and their bundled toolkit/Node requirements.
- Backend/image digests, OS/architecture, tool manifests and label mappings.
- Limits, capability matrix, dependent endpoint inventory and conformance report.

Use exact versions/digests in a release bundle; never `latest` or a mutable branch.
The research snapshots are inputs, not a pre-qualified bundle.

Qualified action SHAs form the regression corpus, not a hardcoded execution
allowlist. Arbitrary actions allowed by repository policy must run through the
supported standard action/runtime interfaces. Do not implement compatibility by
substituting special behavior for a short list of popular action names.

Each feature/operation has a ledger entry:

| Field | Meaning |
| --- | --- |
| `id`, `source` | Stable feature ID or OpenAPI operation ID and normative source |
| `product`, `versions` | Applicable product/API/runner versions |
| `status` | `equivalent`, `conditional`, `simulated`, `unimplemented`, or `not-applicable` |
| `requirements` | Necessary backend, image, policy and external capabilities |
| `tests` | Executable fixtures establishing the claim |
| `differences` | Explicit observed differences and their effects |
| `owner` | Implementation module responsible for closing the gap |

`equivalent` requires passing observable-behavior tests; `conditional` requires
those tests on the stated capability combination. `simulated` is never counted
as real execution coverage. `not-applicable` requires a product/version reason,
not lack of implementation. Unknown entries remain `unimplemented`.

`moction capabilities --json` returns the bundle and resolved host capabilities.
`moction doctor` shows missing prerequisites and actionable remediation.
Execution MUST fail explicitly when a required capability is unavailable; no
substitution of Ubuntu for Windows, successful empty artifacts, automatic mock
fallback, or ignored policy is allowed.

The complete target includes all 200 `actions/` operations in the
[frozen GitHub.com OpenAPI inventory](moction-runtime-research.md#public-api-surface),
plus Actions-related dependencies outside that category. Product-specific
enterprise descriptions require their own inventories. Supporting GitHub.com's
behavior through an enterprise-shaped local URL does not imply GHES parity.

### Modes and platform profiles

Execution modes and machine profiles are separate:

| Mode | Contract |
| --- | --- |
| `execute` | Real workflow planning and real action/step execution |
| `simulate` | Explicit, deterministic API/state-machine fixture; executes no workflow code |
| `replay` | Explicit replay of a sanitized recorded service interaction; no implicit upstream requests |

Mixed real and fabricated job outcomes are not an initial feature. Real jobs can
use explicitly configured mock **external services**, but a failed real job must
never be replaced by a simulated success.

| Profile | Intended use and limitations |
| --- | --- |
| `linux-container` | Default first-release fast path for trusted local development. Native x64/arm64 Linux containers; does not claim hosted-VM equivalence. |
| `linux-machine` | Fresh Linux VM or dedicated machine, with qualified hosted-image/tool inventory and job-local Docker capabilities. |
| `windows-machine` | Windows worker/VM with Windows shells, paths, services and tool inventory. Not a Linux container emulation. |
| `macos-machine` | Provisioned macOS worker/VM on eligible hardware, with qualified images and cleanup. |
| `isolated-linux` | Stronger per-job VM/microVM isolation for hostile code on capable hosts. |

Initial host support: Linux and macOS with a compatible Linux container engine;
WSL2 can use the Linux path. Native Windows CLI networking is a separate gate.
Architecture emulation is opt-in, visible in run metadata, and never described
as native execution.

`--require-equivalence` refuses a run unless every required ledger entry and
machine capability is equivalent for that bundle. Ordinary local execution
still rejects unsupported semantics, while prominently reporting known machine
differences. Required capabilities cannot always be inferred from shell text:
the selected runner image/profile and optional scenario requirements are the
contract; dynamic attempts to use unavailable facilities fail visibly.

## Architecture and language decision

### Decision

Use TypeScript/ESM for the CLI, daemon, scheduler, API adapters, and SDK. Use a
supported, pinned Node.js LTS distribution and GitHub's workflow-parser/expression
libraries. Use the pinned official Actions runner for step execution.

This keeps the user's preferred language on the application surface and reuses
GitHub's public TypeScript language implementation without rewriting the C#
runner. The scheduler must still be implemented: the runner accepts a prepared
job, not YAML. The architecture does not depend on `act`.

Recommended initial components are Fastify for HTTP, generated runtime validators
from pinned schemas, SQLite through a packaged `better-sqlite3` binding on a
dedicated worker, and a filesystem blob store. Pin dependencies and ship native
binding artifacts for each supported platform; installation must not silently
fall back to requiring a compiler. These are implementation choices, not measured
performance claims.

CPU-heavy hashing/compression and blocking database work stay off the main event
loop. Blob traffic uses streams with backpressure. Do not introduce distributed
queues, Kubernetes, Redis, or a network database for a single-machine product.
The process/backend interfaces permit later native helpers where measurements
justify them.

If an unmodified runner cannot be integrated reliably, milestone M0 must produce
an explicit ADR choosing a minimal C# protocol bridge, a maintained runner patch,
or an all-.NET control plane. Do not maintain two schedulers or quietly switch
to a less compatible executor. See the [alternatives](moction-runtime-research.md#images-isolation-and-alternatives).

### Components and trust boundaries

```text
 moction CLI / SDK / gh
          |
  isolated local client transport
          |
 +------------------------- daemon --------------------------+
 | Public GitHub API | Moction management API | Git service  |
 | Identity/policy -> event intake -> planner -> scheduler   |
 |                    |                 |                   |
 |             durable state       runner adapter           |
 |                    |                 |                   |
 |          logs / results / artifacts / cache / blobs      |
 +--------------------|-----------------|-------------------+
                      |                 |
                SQLite + files     scoped execution broker
                                        |
                         fresh runner + workspace per job
                         services / action containers / VMs
                                        |
                         explicit external-service mocks
```

The execution broker alone controls the host container/VM engine. A runner
session can request only operations within its assigned job capability. The
public API cannot submit raw host mounts or engine commands.

| Proposed module | Owns | Must not own |
| --- | --- | --- |
| `packages/contracts` | Versioned schemas, IDs, errors, generated API types | Runtime singleton state |
| `packages/workflow` | Parsing, validation, scoped evaluation, typed plans | Executing shell commands |
| `packages/core` | Events, authorization, scheduling, transitions, leases | HTTP-specific response formatting |
| `packages/github-api` | REST/GraphQL projections and version adapters | A second scheduler |
| `packages/runner-protocol` | Runner wire formats, authentication, messages/results | Independent policy decisions |
| `packages/execution` | Engine adapters, hooks, workspace/network lifecycle | GitHub user credentials |
| `packages/storage` | Transactions, logs/blob metadata, encryption, GC | Arbitrary job code |
| `packages/cli` | UX, daemon lifecycle, isolated `gh` execution | Direct database writes |
| `packages/sdk` | Typed management client and test lifecycle | In-process alternate engine |

Only create these packages when implemented; this spec does not require empty
scaffolding. A single daemon process with bounded workers is the deployment unit.

### Backend interface

The internal contract must distinguish provisioning from execution and must
support cancellation and cleanup even when a runner is unavailable:

```ts
type Capability = string;

interface ExecutionBackend {
  capabilities(): Promise<ReadonlySet<Capability>>;
  prepare(job: PreparedJob, signal: AbortSignal): Promise<JobAllocation>;
  start(allocation: JobAllocation, signal: AbortSignal): Promise<RunnerSession>;
  interrupt(session: RunnerSession): Promise<void>;
  terminate(allocation: JobAllocation): Promise<void>;
  inspect(allocation: JobAllocation): Promise<AllocationStatus>;
  destroy(allocation: JobAllocation): Promise<void>;
}
```

`PreparedJob` contains immutable source/image IDs, resource limits, execution
identity, endpoint projection, and scoped token references, not plaintext
controller credentials. `JobAllocation` identifies owned resources and a lease
generation. Typed failures distinguish invalid configuration, missing capability,
transient infrastructure failure and job failure.

## Configuration and first-run experience

### Distribution and lifecycle

Distribute a signed CLI/daemon package with a pinned runtime; an npm entry point
may select the platform artifact. Workflow authors must not need .NET installed
on their host. The runner and tool images are fetched separately by verified
digest, with size/progress shown before large downloads.

`moction up` must:

1. Resolve an instance name and state directory; acquire a single-instance lock.
2. Check engine connectivity, architecture, memory/disk, filesystem and required
   networking. Never install or reconfigure Docker silently.
3. Create local credentials and restrictive directories, migrate metadata, bind
   local interfaces, and verify readiness with an authenticated request.
4. Print usable CLI/API addresses, active bundle, profile, and unresolved
   capabilities. Return only after ready or with an actionable failure.

Default state lives under the platform's per-user application-state directory,
not inside the source repository. The exact resolved directory is printed by
`moction status --json`. Portable mode accepts an explicit directory.
Instance names are not trusted filesystem paths.

The daemon listens on a user-only Unix socket on POSIX. Optional browser/API TCP
listening is loopback-only by default. Runner-accessible listeners use a private
network and separately scoped authentication. Remote/public listening requires
an explicit address, TLS and authentication; it is never an automatic fallback.

`moction down` drains by default with a configurable deadline, then cancels and
cleans up according to the cancellation contract. It preserves run history.
`moction destroy INSTANCE` requires an explicit target and confirmation unless
`--yes`; it deletes only that instance's owned state/resources.
`moction gc` operates only on ledger-owned resources and supports `--dry-run`.

### Example local configuration

Configuration belongs outside workflow YAML. An optional `moction.yaml` may
contain instance defaults; it must never contain plaintext secrets.

```yaml
apiVersion: moction.dev/v1alpha1
instance: development
compatibility:
  bundle: actions-dotcom-2026-10-16
execution:
  mode: execute
  profile: linux-container
  concurrency: auto
  trust: local
network:
  sourceFetch: online
  jobEgress: online
storage:
  runRetentionDays: 30
  artifactRetentionDays: 30
  cacheLimitGiB: 10
```

The bundle name is an illustrative future release ID, not an existing artifact.
Resolve aliases, images and packages into an immutable runtime lock/manifest.
This runtime manifest is distinct from native `.github/workflows/actions.lock`.

Unknown configuration keys are errors. Precedence is CLI flags, environment
overrides, instance configuration, then documented defaults. Configuration
reloads apply to new runs; existing attempts retain their resolved configuration.
Changing credentials or revoking access applies immediately as an explicit
security event.

### CLI contract

```sh
moction up
moction repo add . --name example/project
moction doctor

moction run ci.yml --event workflow_dispatch --ref main --source worktree --wait
moction runs list
moction runs view 42
moction runs logs 42 --follow
moction runs cancel 42
moction runs rerun 42 --failed

moction exec -- gh api repos/example/project/actions/runs
moction exec -- gh workflow run ci.yml --ref main
moction exec -- gh run watch 42 --exit-status

moction secrets set API_KEY --repo example/project --stdin
moction fixtures apply tests/fixtures/actions.yaml
moction test tests/integration
moction down
```

Run IDs are illustrative. `run` accepts a workflow filename/path or numeric ID,
typed inputs, a JSON event file, explicit repository/namespace, backend/profile
requirements and `--json`. `run --wait` returns a terminal result, not just an
accepted dispatch. Default event is `workflow_dispatch`; a workflow without that
trigger is rejected unless an appropriate explicit event is supplied. `plan`
validates and prints a resolved plan without executing it.

`repo add` records a mapping and imports Git objects into Moction-owned storage.
It must not rewrite remotes, hooks, user branches, Git configuration, or ignore
files. It rejects ambiguous repository inference rather than selecting a random
registered repository.

Noninteractive calls never prompt. Human progress goes to stderr; `--json`
produces one documented result or error object on stdout. Secrets are accepted
through stdin or a secret-provider reference, not command-line argument values.

| Exit code | Meaning for Moction commands |
| --- | --- |
| `0` | Requested operation succeeded; for `run --wait`, workflow succeeded |
| `1` | Workflow completed with a failure conclusion |
| `2` | Invalid arguments, workflow, configuration or fixture |
| `3` | Infrastructure/authentication/storage failure or missing capability |
| `4` | Workflow cancelled |
| `5` | Wait/execution deadline exceeded |
| `130` | CLI interrupted by the user |

`moction exec -- COMMAND` instead preserves the child's exit status and signals;
wrapper setup failure returns `125`. A CLI disconnect does not cancel a run
unless `--cancel-on-disconnect` was explicitly selected. `--wait-timeout` stops
waiting, whereas a workflow timeout stops execution.

### Source selection and reproducibility

Provide two explicit source modes:

| Mode | Behavior |
| --- | --- |
| `commit` | Resolve the requested registered ref to an immutable Git commit. Default for GitHub-compatible dispatch APIs. |
| `worktree` | Snapshot tracked modifications and untracked non-ignored files into a synthetic commit in Moction's private object store. Default for local `moction run`. |

The worktree snapshot records its parent commit and a complete inclusion manifest;
it never changes the user's index or refs. Support explicit exclusion patterns
and preview before including large/untracked files. Reject files that change
during capture or retry capture coherently; never silently assemble a mixed-time
snapshot. Secrets should be supplied by the secret service, not copied into
snapshots. Local mounts and symlinks must not escape approved source roots.

The synthetic SHA becomes the actual `GITHUB_SHA` and local Git ref target for
that attempt. A worktree run is marked as a local extension in Moction metadata.
`--require-equivalence` requires committed/event-accurate sources.

A workspace starts empty as on a fresh runner. The real checkout action must
fetch from the local Git service; Moction must not pre-copy the repository and
claim checkout compatibility. Local actions using `./` therefore require their
files to exist in the workspace when reached.

Registering external action sources fetches only authorized repositories and
resolves refs to commit IDs. Store source provenance and repository identity.
Offline mode uses a prepared dependency/image bundle and reports all missing
objects before execution when discoverable. Dynamic fetches not inferable during
planning fail explicitly at runtime.

## Workflow semantics

See [language research](moction-runtime-research.md#workflow-language) for sources.
The parser, planner and scheduler must share one versioned semantic model.
Validation must preserve source locations and produce actionable diagnostics.

### Parse, validate, resolve and plan

The planning sequence is:

1. Authenticate the event sender and load the repository/policy snapshot.
2. Select workflow source/ref according to the event, not merely current worktree.
3. Parse YAML without executing tags or unbounded alias expansion; validate the
   selected Actions schema, file/size limits and supported current features.
4. Resolve workflow/action references, native lockfiles, policies and required
   capabilities. Retain runtime expression tokens rather than evaluating early.
5. Validate `needs`, cycles, names, permissions, context availability and input
   types. Construct a logical DAG, including reusable-workflow call boundaries.
6. Persist the event, workflow version, run, first attempt and planning result
   atomically; enqueue only after the transaction commits.
7. Evaluate deferred expressions and expand jobs when their required contexts
   become available. Persist every expansion before scheduling children.

A validation error is not a successful empty workflow. For public API operations,
map validation and "accepted event but no matching workflow" behavior according
to the operation's documented/recorded contract; do not invent a universal `422`.

Use GitHub's parser and evaluator as libraries. If they lack newly documented
syntax, add a narrowly scoped, tested schema/evaluator patch or report a missing
feature; silently dropping unknown keys is prohibited. YAML anchors/aliases
require bounded expansion and source diagnostics. Merge keys and other YAML
features require their own qualification rather than inheriting permissive
behavior from a generic YAML parser.

### Expressions and contexts

Provide an explicit evaluation environment per YAML location and phase. Maintain
the upstream context/function availability table in the bundle. Examples:
job `if` is evaluated before `matrix`; step conditions can see prior step
outcomes; `hashFiles` runs against a confined runner workspace, not daemon files.

Required semantics include:

- Actions booleans/null/strings/numbers/arrays/objects and loose numeric coercion;
  case-insensitive string comparison and the documented equality behavior for
  structured values. Do not use JavaScript truthiness, `==`, or `eval`.
- Property/index access, object filters, missing-value behavior, escaping and
  the different rules for expression interpolation versus expression-only fields.
- Implicit `success()` in conditions unless an applicable status-check function
  is present. `always()`, `cancelled()`, `failure()` and dependencies are evaluated
  against the correct job/step scope.
- Documented functions including `case()`, `fromJSON`, `toJSON`, formatting,
  containment and string functions.
- `hashFiles`: supported patterns, SHA-256 composition, workspace confinement,
  Windows case behavior, and empty string when there are no matches.

Keep expression values typed until their destination's conversion rule applies.
`inputs` preserves dispatch booleans, while `github.event.inputs` exposes their
string representation. Secret-tainted values must carry masking metadata across
evaluation and protocol serialization.

Context construction must cover `github`, `inputs`, `vars`, `secrets`, `env`,
`needs`, `matrix`, `strategy`, `job`, `runner` and `steps` where allowed.
Record source repository/ref separately from the caller's context in reusable
workflows. Values unavailable in the current phase must not leak from another
phase or from the daemon's environment.

### Events

Implement a versioned event catalog containing trigger names, activity types,
required payload fields, workflow-source selection, SHA/ref rules, permissions
and default-branch requirements. Event injection and real webhook delivery use
the same normalized event path.

The initial integration-testing release (M2) covers `workflow_dispatch`, `push`,
`pull_request`, `pull_request_target`, `repository_dispatch`, `schedule`,
`workflow_call` and `workflow_run`; full parity requires all documented triggers.
Unsupported triggers are reported, not treated as dispatch aliases.

Event rules include:

- Branch/tag/path filters, ordered include/exclude patterns, activity types and
  correct combination of branch and path predicates.
- Two-dot push and three-dot pull-request diffs, the documented new-branch case,
  diff limits and fallback rules. For the researched baseline: 3,000 diff files;
  pushes over 1,000 commits or diff-generation timeouts cause matching workflows
  to run rather than silently skip.
- Default-branch workflow presence where required; workflow/ref selection for
  fork PRs and privileged `pull_request_target`; accurate merge/base/head SHAs.
- `workflow_run` and check/deployment events generated by committed state
  transitions. Apply documented recursion/chain limits and branch semantics.
- Events caused by the local equivalent of `GITHUB_TOKEN` obey GitHub's
  recursion suppression, including documented exceptions for dispatch events.
- Webhook delivery IDs are deduplicated transactionally. Synthetic test events
  use explicit delivery IDs so tests can exercise both deduplication and new events.

Schedules use a persisted next-fire cursor and time-zone database version.
Honor the five-minute minimum, default UTC, optional IANA timezone and documented
DST behavior. An absent daemon cannot reproduce GitHub's continuously available
scheduler: default restart policy skips missed wall-clock occurrences and records
that fact; optional catch-up is a labeled local extension. Deterministic scenarios
use a virtual scheduler clock. Workflow processes still see real OS time unless
the test explicitly provisions a time-control capability.

### Matrices and dependencies

After prerequisites are available, evaluate the logical job's `if`; only then
evaluate and expand its matrix. Persist immutable axis definitions and generated
variant IDs. Do not use completion order to assign IDs.

Matrix implementation must:

1. Expand the original Cartesian product using stable source-key/value ordering.
2. Apply partial-match `exclude` entries.
3. Apply `include` in declaration order: augment compatible original combinations
   without overwriting original axis values; later includes may replace fields
   previously added by includes. Entries that cannot merge create standalone
   combinations; those are not treated as original combinations for later merges.
4. Apply documented expansion limits before allocating resources.
5. Track `max-parallel`, `fail-fast`, per-variant `continue-on-error`, outputs,
   cancellation and dependency aggregation independently.

Empty/dynamic matrices and invalid JSON must have qualified error behavior, not
an assumed "skip everything" fallback. The 256-job matrix boundary and interaction
between multiple matrix jobs need explicit boundary fixtures.

A non-tolerated failure with `fail-fast` cancels eligible queued/running siblings;
a tolerated failure does not. Matrix `max-parallel` is in addition to global
resource capacity. Jobs requiring unavailable outputs wait for dependency
completion, not a polling loop. A skipped or failed dependency affects the
dependent's condition according to status-function semantics.

Store actual step `outcome` separately from post-`continue-on-error` `conclusion`.
Likewise, preserve a job's recorded execution result separately from the result
used in dependency/workflow aggregation where upstream behavior differs.
Matrix job outputs may race/overwrite according to documented behavior; tests
must not invent deterministic execution ordering as a parity guarantee.

### Reusable workflows and action resolution

Resolve `owner/repo/path@ref`, local `./...` and supported `$/...` references
according to their distinct rules. A local action resolves from the workspace
at execution; `$/` resolves from the action source repository at the executing
commit and cannot be treated as `./` or accept an `@ref` suffix. The latter is
GitHub.com-specific in the researched baseline.

Reusable workflows require:

- Correct typed inputs/defaults/validation, declared outputs and secrets.
- Ten-level nesting and fifty unique reusable workflows for the researched
  baseline; explicit cycle detection and identity-aware reference resolution.
- Caller context where specified, no automatic workflow `env` propagation, and
  permission restrictions that can only stay equal or become more restrictive.
- Correct `secrets: inherit` eligibility and explicit secret forwarding on each
  nested call; environment secrets are applied at their documented scope.
- Matrix invocation/output behavior, including last-successful-completion and
  empty-output rules, established through fixtures.
- Re-run resolution behavior: a full re-run may re-resolve mutable called refs;
  failed/specific-job re-runs use their original resolved called commits.

For `.github/workflows/actions.lock`, use the native versioned schema and exact
repository identities. Validate onboarded workflow dependency completeness,
locked commit reachability, owner/repository IDs, transitive actions, and allowed
action policy against the locked commit. A called reusable workflow uses its
defining repository's lockfile. Moction's runtime/image lock must not replace it.

### Concurrency and scheduling

Maintain separate mechanisms for DAG readiness, matrix concurrency, repository
concurrency groups, environment gates, runner availability, and host resource
capacity. A worker slot alone is not a GitHub concurrency group.

Group key = namespace + repository identity + case-normalized evaluated group.
Persist the original display value separately. Workflow and job groups can
conflict and can be owned through reusable-workflow ancestry.

Within one serializable scheduler transaction:

1. Evaluate group/queue/cancel settings in their allowed context.
2. Reject `queue: max` with `cancel-in-progress: true`.
3. Under `single`, cancel/replace the prior pending member; under `max`, enqueue
   up to 100 pending members and cancel an excess arrival.
4. Request cancellation of the running member if the applicable condition allows.
5. Grant the group only after the prior owner's lease is relinquished or fenced;
   maintain at most one active owner.

Order eligible waiters by their persisted entry into the group. Do not promise
dispatch-order FIFO. Cancellation of a waiter and promotion of the next member
must be atomic. Hold and release nested workflow/job leases according to the
qualified upstream ownership rules; avoid deadlock by representing ancestor
ownership rather than reacquiring the same lease blindly.

The local resource scheduler defaults to fair round-robin across ready runs with
FIFO jobs within each run, subject to the constraints above. This is a local
policy, not a guarantee about GitHub's global fleet scheduler. A running job is
never preempted merely because a newer local run is shorter.

### Permissions, secrets and environments

Calculate effective permissions from organization/repository defaults, workflow,
job and called-workflow restrictions, then apply event/fork policy. When a
`permissions` block specifies individual scopes, unspecified scopes follow
GitHub's documented `none` behavior, not inheritance from the host.
Treat metadata access, `read-all`, `write-all`, new scopes and explicit empty maps
according to the selected schema. The default local repository policy is
read-only contents with required metadata access, configurable through policy APIs.

Issue a unique, short-lived local `GITHUB_TOKEN` per job. Enforce it on Git,
contents, checks, artifacts and all other supported endpoints. Fork-origin
low-trust events do not receive privileged tokens or secrets by default.
`pull_request_target` is not automatically safe: workflow source and untrusted
checkout inputs require the correct privileged-trigger protections.

Resolve organization, repository and environment variables/secrets at their
documented time and precedence. Environment gates are durable states, not sleeps
inside an occupied runner:

- Required reviewers: up to six users/teams; one eligible approval is sufficient.
- Optional prevention of self-approval and configurable administrator bypass.
- Wait timer of 1 to 43,200 minutes and appropriate overall expiration.
- Separate branch/tag deployment rules with their own glob semantics, not the
  workflow path matcher. `*` does not cross `/` under the documented rules.
- Up to six enabled protection rules and custom protection-rule callbacks.
- Environment secrets become accessible only after approval/protection succeeds.
- `environment.deployment: false` suppresses deployment records without bypassing
  applicable environment policy; its conflict with custom deployment protection
  rules is validated.

Local reviewer identities are real principals in Moction's fixture/identity
model, not arbitrary strings that grant themselves access. Approval APIs authorize
the acting principal and record an immutable audit event.

### Steps, outputs and action lifecycles

Delegate shell generation, JS/composite/Docker action execution, pre/post hooks,
command-file handling, masking and runner job cleanup to the official runner.
Qualify behavior for shell defaults and error modes, working directories,
environment precedence, container default shells and platform paths.

The adapter must correctly serialize metadata for `node20`/`node24`, container
entrypoints/args, composite steps, local action paths and inputs. It must not add
automatic `INPUT_*` variables to composites or enforce `required: true` action
inputs differently from the runner's actual behavior.

Support `GITHUB_ENV`, `GITHUB_OUTPUT`, `GITHUB_PATH`, `GITHUB_STATE`, step summaries,
multiline delimiters, workflow commands, annotations, problem matchers, groups
and command-injection protections through the pinned runner. Do not parse
untrusted log text twice to recreate an inconsistent step-command implementation.

Job outputs have the researched 1 MB/job and 50 MB/run limits, approximated in
UTF-16. Secret-bearing outputs are suppressed with a warning as upstream does.
Secret values must not escape through API output fields, summaries or mock exports.
Post steps run in the runner's defined order and condition context; workspace
cleanup happens after them, including cancellation paths.

### Timeouts, cancellation and re-runs

Use monotonic time for process/lease deadlines and persisted wall-clock deadlines
for durable workflow policy. Enforce workflow, job, step, queue, approval and
lease timeouts separately. Limits depend on the advertised hosted/self-hosted
profile; do not impose the six-hour hosted limit on a self-hosted compatibility
profile that documents five days.

Ordinary cancellation:

1. Authorize, persist `cancel_requested`, emit an event and return the public
   API's accepted status without claiming that the process is already dead.
2. Re-evaluate job conditions; `always()` can preserve eligible work.
3. Send cancellation to the runner, which re-evaluates step conditions and
   performs its interrupt/7.5-second/terminate/2.5-second/kill sequence.
4. Bound server-side cancellation by the documented five-minute timeout, then
   fence and terminate remaining owned execution resources.
5. Publish final status only after result reconciliation; record lost results
   as infrastructure failures rather than inventing a successful completion.

Force-cancel follows its distinct contract and does not remain blocked by
`always()`. Daemon shutdown uses the same machinery, not a separate cleanup path.

Re-runs retain the run ID, increment `run_attempt`, allocate new attempt/job/lease
identities, and preserve original ref/SHA and original actor privileges. Record
the triggering actor separately. Full, failed-only and job-with-dependents
re-runs have distinct selection rules; unaffected dependency outputs are reused
only where the upstream behavior permits. Prior attempts remain inspectable.

Enforce the thirty-day re-run window and fifty-re-run limit of the researched
baseline. A deterministic "replay every source at its recorded SHA" feature is
a separate local replay mode, not a redefinition of GitHub re-run semantics.

### Snapshots and other advanced capabilities

Support parsing and planning of `snapshot` string/mapping forms, conditional
publication, image name/version selection and successful-job-only publication.
Model one produced image per requesting job, initial `1.0.0`, automatic minor
increments, explicit supported major selection and inherited base expiration.
Publish an image version only after durable completion of backend creation;
failed provisioning must not advertise a usable image.

An OCI filesystem snapshot is not equivalent to a full hosted-machine image.
Only a qualified backend may satisfy that capability. Larger-runner groups,
custom images, GPU, private networking, organization policies and new API
resources stay in the ledger even before an execution backend supports them.

## Execution and runner integration

### Mandatory feasibility spike

Before building the general scheduler, prove one pinned, unmodified runner can
complete this local sequence:

1. Provision/authenticate an ephemeral runner and establish a session.
2. Receive and acquire a prepared job; renew its lease.
3. Execute a shell step, a JavaScript action, a composite action and real checkout.
4. Publish live logs, annotations, outputs and a terminal result.
5. Upload/download a modern artifact and save/restore a v2 cache.
6. Cancel a blocked step and run its eligible post cleanup.
7. Recover a control-plane restart without accepting stale or duplicate completion.
8. Do all of the above without contacting GitHub private orchestration endpoints.

Capture sanitized request/response fixtures, protocol versions and required
feature flags. Distinguish development source from the released binary under
test. If the spike fails, the M0 ADR must name the exact missing protocol and
chosen alternative; proceeding with assumptions is not an acceptable gate pass.

### Protocol inventory

The private adapter owns at least these independently authenticated surfaces:

| Surface | Required behavior |
| --- | --- |
| Runner registration/bootstrap | Short-lived registration/removal tokens, labels/groups, JIT/ephemeral configuration, session identity |
| Broker | Create/delete session, message acquisition, acknowledgement, cancellation delivery and required connectivity probes |
| Run service | `acquirejob`, `renewjob`, `completejob`, typed 404/409/422 acquisition errors and completion reconciliation |
| Job preparation | Plan/timeline/job UUIDs, typed contexts/template tokens, steps, variables, resources, mask hints, defaults, outputs, environment and newer message fields |
| Action resolution | Authorized source download metadata, immutable source/version mapping and appropriate scoped credentials |
| Timeline/results | Ordered step transitions, annotations, outputs, telemetry and environment URL |
| Logs | Live append protocol, sequence/cursor handling, finalization and public log projection |
| Artifacts/cache | Runtime JWT scopes, Twirp methods, signed blob transfer and finalization |
| OIDC | Permission-gated local token endpoint with accurate claim construction |

Implement the wire serializer from pinned contract fixtures, not unchecked
TypeScript objects. Context/template token tags and field omission/null behavior
matter. Unknown message versions must fail during qualification or handshake,
not be blindly accepted.

Disable uncontrolled runner self-updates. A runner upgrade is a compatibility
bundle update with verified binary provenance and regression results. Offline
mode must also suppress unapproved discovery/update traffic; it cannot depend
on GitHub being reachable to start a locally qualified runner.

Renewal retries must be bounded and jittered. A lease is identified by allocation
ID plus monotonically increasing generation; stale workers cannot publish after
revocation/reassignment. Acknowledgement is not completion. A timeout cannot
cause two valid owners of the same job.

### Container topology

For the Linux container backend:

- Create a fresh runner container, workspace volume and private job network.
  The control-plane state directory and user home are never mounted.
- Mount a minimal hook/broker client read-only. The official runner uses the
  documented `ACTIONS_RUNNER_CONTAINER_HOOKS` integration to create
  job/services/action containers through the scoped broker.
- The broker validates all mounts, images, ports, resource options, command
  arguments and response paths against the job allocation. It accepts no
  arbitrary Docker-socket access or raw host paths from a job.
- Job-container steps and services share the expected network and service aliases.
  Service health checks must complete before steps start.
- For a job without a job container, preserve localhost service-port behavior.
  Use a per-job network namespace anchor and scoped port forwarding where needed;
  do not publish service ports indiscriminately on the developer's host.
- All action/job/service containers share only the workspace/temp volumes that
  GitHub would make available. Correct UID/GID, permissions, executable bits,
  symlinks and host/container path translation are required.

The hooks cover runner-managed containers, not arbitrary `docker` calls inside
workflow code. A workflow needing a Docker daemon requires a declared job-local
daemon capability: a qualified rootless nested daemon or an isolated VM daemon.
Never attach the unrestricted host engine socket merely to make such a workflow
pass. An explicitly consented host-Docker development override is a separate
unsafe capability, not the default or a hostile-code boundary.

Containers remain owned by one allocation even if a step changes labels or names.
Resource handles are stored outside job-writable files. Cleanup must delete
only positively identified Moction-owned resources and tolerate already-deleted
objects. After timeout/crash, reclaim orphaned networks, ports, volumes and
sidecars using the durable ownership ledger.

### Images and prewarming

`runs-on` arrays require all labels; groups restrict eligible workers in addition
to labels. Match OS and architecture exactly unless emulation was explicitly
enabled. The compatibility bundle maps aliases such as `ubuntu-latest` to
versioned, visible images; updates must not change existing run attempts.

Do not claim that an `ubuntu:24.04` base image is a GitHub-hosted runner image.
Publish software inventory, shells, tool cache, filesystem/machine differences,
architecture and provenance for every supported image.

Warm caches may retain verified image layers, Git objects, action archives,
tool downloads and pristine prebooted allocations. They must never reuse a
workspace, credentials, writable layers or process namespace after running job
code. Prewarmed runners receive job credentials only on allocation and are
destroyed after use. New run freshness is a correctness constraint, not a
performance knob.

Native/VM backends implement the same ownership and lease interfaces. Windows
process-tree cleanup and macOS runner prerequisites need their own tests; Linux
signals and Docker behavior must not be assumed portable.

## Public APIs and GitHub clients

### Separate API surfaces

| Surface | Purpose |
| --- | --- |
| GitHub-compatible REST | Unmodified external Actions clients; enterprise-shaped `/api/v3` plus qualified GitHub-shaped projection |
| GitHub-compatible GraphQL | Real query execution for the declared Actions/client dependency subset |
| Git service and ancillary APIs | Checkout, repository metadata, refs, contents, checks/deployments and client dependencies |
| Moction management `/_moction/v1` | Local instance, fixtures, namespaces, capabilities, clock, plans and diagnostics |
| Private runner/results/blob services | Versioned runner/action transport; not a public management API |

Management responses must not leak into GitHub responses as undocumented required
fields. Local execution mode/differences are available through management
resources and explicit diagnostic headers.

### Management API contract

Publish an OpenAPI description for `/_moction/v1`, separate from the upstream
schemas. All routes require a management capability scoped to the instance or
namespace. The CLI and SDK use these routes rather than filesystem/database
shortcuts.

| Method/path below `/_moction/v1` | Request and response |
| --- | --- |
| `GET /status` | `200` with instance ID, daemon version, readiness, bundle, profile, state path and owned-resource counts |
| `GET /capabilities` | `200` with bundle digest and ledger entries including status, requirements and diagnostics |
| `POST /namespaces` | Name, mode, seed, optional clock start/TTL; `201` with ID and a namespace-scoped credential delivered once |
| `DELETE /namespaces/{ns}` | `202` with cleanup operation ID; revoke new work/tokens, cancel owned runs, remove owned resources, then publish completion |
| `POST /namespaces/{ns}/repositories` | Registered name plus approved local path or uploaded Git bundle; `201` with repository ID, default ref and source digest |
| `POST /namespaces/{ns}/events` | Event type, delivery ID and payload; `202` with event ID and matched run IDs, possibly an empty set |
| `POST /namespaces/{ns}/plans` | Repository, workflow, ref, source mode, event and inputs; `200` with plan, warnings and required capabilities, without execution |
| `POST /namespaces/{ns}/runs` | Same selectors plus execution options; `201` with durable run ID, attempt number, mode and usable management/GitHub URLs |
| `GET /namespaces/{ns}/runs/{id}` | `200` with internal phase, public status/conclusion, source/image provenance, capability differences and queue reason |
| `POST /namespaces/{ns}/runs/{id}/cancel` | Optional force flag; `202` after recording the authorized cancellation request |
| `GET /namespaces/{ns}/runs/{id}/events` | Server-sent event stream with durable numeric IDs and `Last-Event-ID` reconnection |
| `POST /namespaces/{ns}/fixtures` | Parsed, versioned fixture; `201` after atomic import, with created resource IDs and content digest |
| `POST /namespaces/{ns}/clock/advance` | Nonnegative integer milliseconds; `200` with new time and processed event cursor; disallowed for real execution clocks |
| `GET /operations/{id}` | `200` with pending/running/completed/failed state and explicit cleanup/import errors |

Management lists use opaque cursors and a bounded `limit` (default 50, maximum
100); their pagination contract is independent of GitHub's. Uploads are separate
streaming requests, not base64 source archives in unbounded JSON. Server-side
source paths are accepted only from an authorized local registration client and
must resolve within explicitly approved roots.

A run request has this minimum JSON form:

```json
{
  "repository": "example/project",
  "workflow": ".github/workflows/ci.yml",
  "ref": "main",
  "source": { "mode": "commit" },
  "event": { "type": "workflow_dispatch" },
  "inputs": {},
  "mode": "execute"
}
```

An accepted response includes `id`, `runAttempt`, `mode`, `managementUrl`,
`githubApiUrl` and `htmlUrl`. Inputs/options must match the selected namespace
mode; a request cannot implicitly change a simulated namespace into real
execution. For generic event injection, a valid nonmatching event is not an
infrastructure error.

Management failures use a stable `error` object with `code`, `message`,
`requestId` and optional structured `details`. Use `400` for malformed envelopes,
`401`/`403` for authentication/authorization, `404` for missing resources, `409`
for state conflicts, `422` for semantic validation, `429` for enforced capacity,
and `503` for infrastructure unavailability. Missing execution capability is
`422` with `code: capability_unavailable` and the missing requirements. GitHub
facades retain their distinct upstream or explicitly non-equivalent behavior.

Event streams send heartbeats, enforce bounded buffers and disconnect slow
consumers with a resumable cursor. If a requested cursor has expired, return
`410` with the earliest available cursor instead of silently skipping events.
Namespace TTL expiry executes the same observable cleanup operation as deletion.
Minimal unauthenticated liveness reveals no repositories, paths or credentials.

### Contract generation and HTTP rules

Generate route/type/validation inventory from the pinned OpenAPI operation IDs.
Supplement with verified headers, aliases, previews and dependent services.
Every operation receives a ledger entry and an explicit authorization rule.
The frozen OpenAPI's complete method/path/request/response definitions are
incorporated as the normative schema contract; the tables below highlight
behavioral requirements rather than replacing that full inventory.

Preserve:

- Exact methods, paths, identifier/filename aliases, query parameters, success
  statuses, required/nullable/omitted fields, enum values and error shapes.
- `Accept` negotiation, `X-GitHub-Api-Version`, applicable conditional requests
  and ETags, pagination `Link` headers, redirect `Location`, request IDs, rate-limit
  headers and content types.
- Stable numeric public IDs, opaque node IDs, run numbers, attempts, step numbers,
  statuses versus conclusions, ordering and timestamp semantics.
- Coherent URLs for the current client projection. No response may accidentally
  send a client to the real GitHub repository or an unreachable container address.
- Authorization and repository visibility on every read as well as every write.

Public numeric IDs must stay within safe integer range for supported JavaScript
clients; allocate them durably and reject exhaustion instead of losing precision.
Protocol UUIDs and public IDs are separate columns. Do not fabricate upstream
GitHub node IDs for local objects.

Validation/auth/not-found/conflict/limit errors follow the selected operation,
including the standard `message`, optional structured `errors`, documentation
reference and status fields where applicable. Unknown routes are `404`.
A known but unavailable capability returns an explicit error, normally `501`
with `X-Moction-Error-Code: capability_unavailable`, and marks that operation
non-equivalent. This is preferable to success-shaped lies but is not a parity
claim. Authentication errors must not expose private resource existence.

The control plane may have higher local limits than GitHub, but `--require-equivalence`
and the conformance profile enforce documented limits. Optional relaxed limits
are a labeled local extension, not an invisible difference.

### REST versions and principal routes

Requests without a version header use `2022-11-28`. Also implement
`2026-03-10` as a distinct version adapter; unsupported version errors must match
the qualified upstream behavior. Never let a newest schema silently redefine
older responses.

The table lists a high-usage subset, not the entire 200-operation target.
Paths are relative to `/repos/{owner}/{repo}` unless stated otherwise.

| Method/path | Required behavior |
| --- | --- |
| `GET /actions/workflows` and `GET /actions/workflows/{id-or-file}` | Workflow identity, path, state, list filters and pagination |
| `PUT /actions/workflows/{id}/enable` or `/disable` | Persist enablement; event matching respects it |
| `POST /actions/workflows/{id}/dispatches` | Validate ref/typed inputs; old version returns `204` unless `return_run_details: true`, then `200`; new version always `200` with `workflow_run_id`, `run_url`, `html_url` and does not accept the removed option as a supported parameter |
| `GET /actions/runs`, workflow-specific `/runs`, `GET /actions/runs/{id}` | Filters, statuses, attempt-aware fields and coherent dependent URLs |
| `POST /actions/runs/{id}/cancel` and `/force-cancel` | `202` accepted; qualified `409` conflicts; distinct cancellation semantics |
| `POST /actions/runs/{id}/rerun`, `/rerun-failed-jobs`, `/actions/jobs/{id}/rerun` | `201` on accepted re-run; debug logging and correct job/dependency selection |
| `GET /actions/runs/{id}/attempts/{attempt}` and nested `/jobs` or `/logs` | Historical attempts remain stable and inspectable |
| `GET /actions/runs/{id}/jobs`, `GET /actions/jobs/{id}` | `latest` versus `all` filters, steps, runner/labels, timestamps, annotations/check URLs |
| `GET /actions/jobs/{id}/logs` and `/steps/{step}/logs` | Qualified redirect/content contract; job logs use `302` with a usable `Location` |
| `GET /actions/runs/{id}/logs` | Downloadable ZIP with CLI-compatible layout; attempt variants and deletion semantics |
| `GET /actions/runs/{id}/artifacts` and repository `/actions/artifacts` | Access-controlled metadata, digest/size/expiry, correct filters and pagination |
| `GET /actions/artifacts/{id}/{format}` | Download redirect and actual finalized bytes, not metadata masquerading as a download |
| Run approval, pending-deployment and protection-rule routes | Authorized durable decisions that unblock or reject actual gates |
| `/actions/caches`, usage and cache-policy routes | Real scope/entry data, quotas, eviction and mutation semantics |
| Secret/variable, permission/policy, runner/group and hosted-runner routes | Scoped durable management resources with their documented effects |

For dispatch on the old version, return details only when requested; for the new
version, return the exact documented fields. The response is emitted after the
run and event are durable. This fixes a common client integration gap without
relying on "list the latest run" races.

### Concurrency-group APIs

Implement these projections from the actual lease/queue tables:

- Repository `GET /actions/concurrency_groups`: active groups, `per_page` and
  `after`, `total_count`, `group_name`, `group_url`, `last_acquired_at`.
- `GET /actions/concurrency_groups/{name}`: inactive/missing is `404`;
  `ahead_of_run` and `ahead_of_job` are mutually exclusive, with `422` for a
  target outside the group. Include run/job identities and pending/running state.
- `GET /actions/runs/{id}/concurrency_groups`: configured groups remain visible
  even when empty; name ordering and `before`/`after` cursors follow the contract.

Ancestor/reusable-workflow membership affects "ahead of" queries. A run query
does not include every unrelated job lease, while a job query includes its
ancestor path. Preserve cursor navigation rules; a forward `after` request must
not get an invented `prev` link. Do not implement all list APIs with a common
page-number assumption.

### Secrets, variables and runner administration

Secret APIs expose public keys and metadata, never secret plaintext. Accept
LibSodium sealed-box ciphertext and `key_id`, decrypt only in the secret service,
then re-encrypt at rest. Qualify key rotation and stale-key errors against
fixtures; do not invent an upstream validity window. Reject unknown keys
explicitly. Create is `201`, update `204`.
Organization visibility (`all`, `private`, `selected`) and selected repository
IDs actually govern availability.

Variables use their separate schemas and timing/precedence rules. Repository,
organization and environment scopes are not interchangeable.

Runner APIs must reflect live/known workers, labels, busy/offline status, group
authorization and ephemeral/JIT lifecycle. Registration/removal tokens expire
and cannot access repository data. A hosted-runner resource is not "ready" until
its backend is provisioned. Unsupported machine/hardware requests fail
explicitly; simulated resources are available only in explicit simulation mode.

### Git and ancillary service contract

Provide authenticated smart HTTP Git using a bounded Git subprocess or equivalent
qualified library, with object/ref access confined to registered repositories.
Required checkout capabilities include commits/branches/tags, shallow fetch,
requested depth/ref, merge refs, archives/contents fallback, submodules and LFS
where advertised. Annotate missing LFS/submodule capabilities before promising
checkout parity.

The checkout action must receive a local `GITHUB_SERVER_URL` and actual token
permissions. Do not rewrite a user's global Git config or silently fall through
to GitHub. Explicit external dependencies use an allowlisted source-fetch
identity distinct from job credentials.

Inventory at least repository/user/organization metadata, refs/contents/commits,
check suites/runs/annotations, deployments/environments/statuses, pull-request
metadata and required GraphQL fields. `gh workflow run` reads workflow contents;
`gh run view` can request jobs, workflow details, annotations and log fallback
routes. An API facade limited to `/actions/runs` is insufficient.

GraphQL must parse and validate real queries, variables, fragments, pagination
and field-level authorization for its declared schema. Do not match query strings
or return a canned object regardless of selection. Unsupported fields return
normal GraphQL errors and remain a ledger gap. Full Actions parity is not a
claim to implement every unrelated GitHub product API.

### `gh` transport and endpoint projection

The research verified a POSIX transport path with unmodified `gh 2.102.0`.
Implement `moction exec -- gh ...` using a private per-invocation configuration:

```yaml
# Generated inside a temporary, user-only GH_CONFIG_DIR.
http_unix_socket: /absolute/path/to/instance/api.sock
```

The wrapper sets `GH_HOST=moction.localhost`,
`GH_REPO=moction.localhost/OWNER/REPO`, the local enterprise token and isolated
`GH_CONFIG_DIR`. It clears conflicting GitHub token variables, disables
interactive authentication/update checks, and does not modify normal `gh`
credentials. Never inherit a real GitHub token as the local token.

Explicit `--hostname moction.localhost` works. Do not recommend
`--hostname localhost:PORT`: the tested CLI rejects the colon. `GH_HOST` accepting
a port in some paths is not a stable substitute. An absolute `gh api` URL is a
useful narrow alternative, not a complete solution for `gh run` commands.

The server needs client-specific endpoint projections:

| Audience | Addresses |
| --- | --- |
| POSIX `gh` | Logical HTTPS `moction.localhost` URLs routed through the Unix socket, REST under `/api/v3`, GraphQL `/api/graphql` |
| Ordinary host clients/browser | Actual loopback TCP origin if enabled, or SDK Unix-socket transport |
| Runner/action containers | Reachable private-network origins with job-scoped authentication and configured DNS/trust |
| Native/VM workers | Reachable, authenticated, TLS-protected origins for that worker network |

Construct `api_url`, `server_url`, Git URLs, results URLs and blob redirects from
the authenticated client's projection. Every returned absolute URL must be
usable by that client. Host browser `html_url` values must point to an enabled
local run/workflow page, not a nonexistent GitHub page; a minimal read-only HTML
view suffices initially.

Use `.localhost` hostnames where required by qualified artifact clients. Within
containers, explicitly map these names to the private service, not the
container's own loopback. Toolkit versions may differ: the suffix exception is
a tested bundle property, not a universal standard.

The artifact client can discard an `ACTIONS_RESULTS_URL` path prefix, so route
its methods at the origin's expected paths. Use token/host tenancy, not a mandatory
path prefix. Include blob redirects in endpoint tests: the initial `gh api` call
working does not establish that downloads work.

Unix-socket transport is not a native Windows solution. Native Windows support
requires a separately proven trusted loopback/TLS hostname strategy or an
upstream-supported transport. It must not silently change system DNS, occupy
privileged ports, disable TLS verification or require administrator access.
Until that gate passes, advertise WSL2 rather than native Windows parity.

## Artifacts, cache and output streams

### Results and blob services

Implement the qualified Twirp JSON/protobuf representations of
`github.actions.results.api.v1.ArtifactService` and the corresponding cache
service. Artifact methods are `CreateArtifact`, `FinalizeArtifact`,
`ListArtifacts`, `GetSignedArtifactURL`, `DeleteArtifact`; cache methods include
`CreateCacheEntry`, `FinalizeCacheEntryUpload`, `GetCacheEntryDownloadURL`.
The actual bundle may need additional/legacy contracts; unsupported generations
must be visible.

Runner-issued runtime tokens contain the expected
`Actions.Results:<run-backend-id>:<job-backend-id>` scope and the correct
expiry/issuer/audience. Separate permissions govern artifact writes, cache reads,
cache writes and repository access.

Signed upload/download URLs bind object ID, method, expiry, size bounds and
namespace. They do not expose filesystem paths or permit listing sibling data.
Implement the blob provider's actual client operations, including block/chunk
upload, commit, range reads and headers used by the qualified SDK. A generic
single `PUT` server is insufficient if the bundled client uses a different
transfer protocol.

Uploads are staging objects until finalization verifies expected size/digest and
durably publishes metadata. Expired/incomplete uploads are GC candidates; they
must not appear as finalized artifacts or cache hits. Finalization and retries
are idempotent for the same upload identity, with conflicts for incompatible
repeated payloads.

### Artifact semantics

Artifact metadata includes repository, run/attempt/job backend IDs, name, public
ID, size, digest, content/archive mode, creation/expiry timestamps and finalization
state. Enforce access controls on metadata and content equally.

Support immutable uploads, no unintended append semantics, overwrite as
delete-and-new-ID, multiple named artifacts, hidden-file selection behavior,
compression options, 500 artifacts/job and retention policy. File selection and
archive production should remain in the official action; preserve its uploaded
bytes. Current single-file `archive: false` behavior requires distinct qualified
download handling rather than blindly wrapping everything in ZIP.

Deletion/expiry revokes new downloads and updates list/get behavior. Define
in-flight download behavior through contract fixtures; do not delete bytes still
referenced by another finalized object. Preserve executable bits and permissions
only where the actual archive/action protocol does; do not claim ZIP restores
Unix metadata that the client did not retain.

### Cache semantics

Store cache metadata under repository + logical ref scope + key + version.
Version reflects the qualified client format/compression inputs, not a Moction-only
hash of the key. Cache entries are immutable after successful finalization.

Lookup follows the documented ordering: exact key/version, applicable partial
matches, ordered restore keys, and permitted default/base-branch fallback.
Choose most recent eligible matches where upstream specifies that. A PR merge-ref
cache is not visible to arbitrary other branches. Do not grant a job permissions
merely because its guessed key exists.

Enforce `cache-mode` independently from `GITHUB_TOKEN`:

| Mode | Read | Write |
| --- | --- | --- |
| `read` | Yes | No |
| `write` | Yes | Yes |
| `write-only` | No | Yes |
| `none` | No | No |

Apply trigger-specific default-branch write rules and reusable-workflow
restrictions. Preserve the real action's warning/info/success behavior when an
operation is disallowed; the server must expose the right token/error contract
rather than deciding that every denied save fails the job.

Reservations and concurrent saves need transactional conflict handling. Do not
overwrite an already finalized entry. Quota and eviction are repository-aware,
last-access tracked, and observable through cache APIs. Cache deletion cannot
delete an artifact sharing deduplicated blob bytes.

### Logs, summaries and annotations

Store append-only, ordered log chunks with attempt/job/step IDs, sequence numbers,
timestamps, byte lengths and content digests. Duplicate appends with identical
sequence/payload are safe; mismatches are protocol errors. Support incremental
streaming through Moction's API and public finalized log downloads.

Redact secrets before logs are durably stored or exported. Test masking across
chunk boundaries, multi-line secrets, runner-added masks and common transformed
representations. Masking is best-effort and does not make malicious code safe.
Do not expose raw protocol request bodies containing tokens in diagnostic logs.

Build run-log ZIPs with the job/step layout expected by `gh run view --log`,
including duplicate/unsafe-name handling and attempt selection. Stream ZIP
generation or cache a content-addressed finalized ZIP; never buffer an entire
large run in process memory.

Annotations, problem matchers, step summaries, check results and environment URLs
are durable typed data and must be available through their appropriate APIs/UI.
Enforce payload limits with visible warnings/errors matching the contract.
On storage failure, mark infrastructure failure and stop safely; never report
that lost logs or artifacts were saved.

## Mocks, deterministic testing and SDK

### Namespace isolation

Each test creates an isolated namespace with its own repositories, IDs, identities,
policies, clock, scenarios, runs and token capabilities. Every storage lookup and
resource allocation is namespace-scoped. Shared immutable image/source/blob
caches may deduplicate bytes, but cannot bypass per-namespace authorization.

Fixture import is schema-validated and atomic: an invalid fixture leaves no
partially created policy or successful-looking repository. Import errors include
file/path/line where available. Unknown fields and references are errors.
Fixtures contain secret-provider references, not real credentials.

### Scenario contract

Scenarios are Moction resources outside workflow YAML. They can seed repositories,
events, principals, permissions, approvals, cache/artifacts and external-service
responses. Explicit `simulate` mode additionally scripts scheduler/job outcomes.

```yaml
apiVersion: moction.dev/v1alpha1
kind: Scenario
name: approval-then-failure
mode: simulate
clock:
  start: "2026-01-01T00:00:00Z"
seed: 42
repository:
  name: example/project
  source: ./fixture-repo
jobs:
  - workflow: .github/workflows/deploy.yml
    job: deploy
    durationMs: 1500
    result: failure
    logs:
      - "fixture deployment failed"
faults:
  - route: "GET /repos/example/project/actions/runs"
    onRequest: 1
    response:
      status: 503
      headers:
        Retry-After: "1"
```

The parser resolves the workflow/job selectors before starting and rejects
ambiguous/missing matches. Matrix selectors must be explicit typed axis maps.
Simulation respects real API and state-transition rules: it cannot mark a job
successful before its required gate solely because a duration elapsed.

Fault matching includes method, canonical path, selected query/body predicates,
request ordinal and namespace. Effects include delay, status/error, disconnect,
rate limit and bounded partial transfer. Match precedence is declaration order,
with ambiguity diagnostics where multiple rules are otherwise indistinguishable.
Request ordinals are assigned transactionally at ingress.

A seeded simulation clock controls scheduler/lease/event progression and recorded
timestamps. Determinism means identical ordered inputs produce identical outputs;
concurrent real network arrival order is itself an input. It does not promise
determinism for arbitrary real actions, operating-system scheduling or external
networks. `clock advance` is authorized and unavailable to a workflow token.

### Mock external services and replay

For `execute` mode, attach explicit HTTP/TCP mock services to the job network.
Expose service addresses through fixture-provided configuration rather than
silently intercepting all TLS traffic. TLS mocks use an explicit local CA in the
test trust configuration; never set global "skip verification" flags.

`--offline` blocks external source fetching and job egress, except named local
services. An ordinary cache miss is not permission to go online. Online mode is
visible and may fetch public dependencies, but must not borrow cloud credentials.
Policy can separate control-plane source fetch from job egress.

Replay cassettes are explicitly imported/exported, schema-validated and sanitized.
Default recording is local-only; recording an upstream GitHub API needs explicit
authorization, a scoped credential and a write policy. Never replay remote writes
against GitHub automatically or export tokens, secret values, signed URLs, or
private source unintentionally. Store errors and timing as first-class replay
data rather than changing every recorded operation into success.

### SDK

Provide a typed `@moction/sdk` client over the management API. It must not embed
an alternate scheduler or require mutation of global process environment:

```ts
await using instance = await createMoction({
  namespace: "checkout-integration",
  mode: "execute",
  profile: "linux-container",
});

await instance.repositories.add({
  name: "example/project",
  path: fixtureRepository,
});
const run = await instance.runs.dispatch({
  repository: "example/project",
  workflow: "ci.yml",
  ref: "main",
});
const result = await instance.runs.wait(run.id, { signal });
assert.equal(result.conclusion, "success");
```

`createMoction` can attach to an existing daemon or start an owned one; ownership
determines whether `dispose` removes a namespace or stops a daemon. The returned
handle implements `AsyncDisposable` through the same idempotent `dispose` method;
`await using` preserves both body and cleanup failures via `SuppressedError`.
The client supplies isolated `gh` execution options without setting `process.env`.
Support `AbortSignal`, bounded waits, typed errors, event subscriptions with
reconnect cursors, cleanup timeouts and a TTL sweeper for crashed test clients.
A failed cleanup is surfaced, not swallowed; preserve the original test failure
alongside cleanup diagnostics.

## Persistence and recovery

### Storage layout and invariants

Use SQLite WAL on a local filesystem, one bounded writer queue, foreign keys,
versioned migrations and explicit busy/deadline handling. Network filesystems are
unsupported unless separately qualified. Database work runs off the event loop.
Durability settings must be documented; fast tests may opt into ephemeral state,
but persistent default mode must survive a daemon crash without acknowledged
metadata disappearing.

```text
instance/
  state.db
  credentials/
  repositories/
  blobs/sha256/
  uploads/
  logs/
  runtime/
```

All directories have restrictive permissions. Credentials use the platform key
store when available or an explicit user-protected key file; plaintext secrets
must not be stored in SQLite. The lock excludes simultaneous daemon ownership.
SQLite WAL sidecars are part of backup/recovery procedures.

Core tables and relationships:

| Entity | Required keys/data |
| --- | --- |
| Namespace/principal/token | Namespace ID; principal identity; hashed bearer token reference, scopes, expiry, revocation |
| Repository/workflow/version | Stable public IDs; owner/name; source/ref/SHA; workflow path/content digest; enablement |
| Event | Delivery ID, event type, trust classification, source SHA/ref, payload digest, deduplication key |
| Run | Repository/workflow ID, public ID, run number, event ID, original actor, latest attempt |
| Attempt | Run ID + attempt number, triggering actor, resolved config/dependencies, state, deadlines, timestamps |
| Logical job/variant | Attempt, workflow-call path, `needs`, matrix values, capabilities, state and outputs |
| Step/result | Variant + step number, outcome/conclusion, timeline IDs, annotations and summaries |
| Allocation/session/lease | Resource handles, runner identity, generation, expiry, heartbeat and terminal fencing |
| Concurrency/gate | Normalized group, owner/queue sequence; environment approvals, protection states and deadlines |
| Artifact/cache/upload | Scoped metadata, immutable blob reference, staging/finalized state, retention and version |
| Log chunk | Attempt/job/step + sequence, digest, file offset, length and redaction state |
| Snapshot | Base image, requested version, backend task, readiness, expiry and provenance |
| Outbox/audit | Monotonic event sequence, aggregate ID/version, payload, delivery state |

Use database constraints for uniqueness, foreign keys, legal generation ownership
and single concurrency-group ownership. All joins include namespace identity;
opaque IDs alone are not authorization. Secret-bearing output storage is separate
from exportable run metadata.

### State model

Use internal states richer than public GitHub statuses:

```text
accepted -> validating -> waiting_dependencies -> waiting_policy
         -> waiting_capacity -> provisioning -> running -> finalizing -> terminal
```

Branches can skip states when already satisfied. Cancellation is a durable
request alongside the current state until execution stops; it is not an
instantaneous destructive transition. Gate decisions, lease generations and
result versions are independent records.

Public adapters map these states to the qualified `queued`, `pending`, `waiting`,
`requested`, `in_progress`, `completed` and other allowed values for the resource.
Do not expose internal enum strings accidentally. Conclusions remain null until
the relevant terminal contract permits them and distinguish success, failure,
cancelled, timed-out, skipped, neutral, action-required and stale cases.

An aggregate transition, its outputs and its outbox event commit in the same
transaction. HTTP handlers and worker callbacks submit commands to this model;
they do not independently update status columns. Optimistic aggregate versions
prevent late callbacks from undoing a cancellation or newer attempt.

### Idempotency and delivery

Internal commands, event delivery, lease operations and blob finalization have
stable request IDs and idempotency rules. Management writes may use an
`Idempotency-Key`; GitHub routes retain their upstream behavior rather than
inventing universal deduplication for intentional repeated dispatches.

Use a transactional outbox for scheduler wakeups, notifications and derived
workflow events. Consumers acknowledge monotonically and can safely replay.
No claim of exactly-once process execution is made across arbitrary crashes.
The achievable invariant is a single valid lease owner and idempotent acceptance
of one terminal result, with ambiguous infrastructure outcomes exposed.

Write large blobs to staging, verify them, durably place them in content-addressed
storage, then transactionally publish metadata. Never publish a reference before
bytes are available. Orphaned bytes from a crash can be collected after a grace
period; bytes with live references cannot.

### Recovery rules

| Failure | Required response |
| --- | --- |
| Daemon restart with live runner | Reconcile session/lease and engine ownership; accept reconnect only for the current generation |
| Runner disappears | Wait bounded lease grace, inspect/fence resources, record infrastructure failure; never guess success |
| Completion arrives twice | Same result is idempotent; conflicting result is rejected and audited |
| Cancellation races completion | Serialize through aggregate version/lease ownership and preserve the accepted terminal decision |
| Container starts before callback persists | Recover from allocation token/resource ledger; do not create an untracked duplicate |
| Disk full during log/upload | Fail the write explicitly, stop or fail execution safely, retain diagnostics; no finalized false metadata |
| Crash after blob write before DB commit | Leave unreferenced blob for delayed GC |
| Engine unavailable | Reject new allocations, retain queued state with actionable diagnostics, bound waiting by policy |
| Upgrade/migration failure | Keep original data recoverable; do not start a partially migrated scheduler |

Automatic retry is safe for idempotent provisioning before workflow code starts.
After code may have executed, do not retry arbitrary side effects automatically.
Mark infrastructure failure and require the documented re-run/explicit retry
policy. User-directed job re-runs are distinct attempts.

Default local retention is thirty days for runs/artifacts and a 10 GiB cache
budget; compatibility settings can change these to the declared repository
policy. GC respects active uploads, readers, attempts and deduplicated references.
Backup/export is a consistent metadata-plus-blob snapshot with secret values
excluded by default.

## Security model

### Trust modes

The default `trust: local` profile is for the developer's trusted workflows,
not hostile multi-tenant execution. Containers share a kernel and cannot be
marketed as a complete hostile-code boundary. Untrusted fork execution requires
an isolated backend and the fork permission policy; missing isolation is a
capability error.

Source-fetch credentials, management credentials, workflow tokens, runner
session tokens, runtime artifact/cache tokens, mock-service tokens and signing
keys are distinct authorities. A workflow token cannot create namespaces, grant
itself secrets, advance the clock, inspect another run's private data, or access
the engine broker outside its allocation.

All local listeners authenticate except explicitly public health/liveness data.
POSIX socket permissions, constant-time credential checks, origin/host validation
and CSRF protections for any browser write surface are required. Do not treat
"localhost" as sufficient authorization.

### Required controls

- No raw host Docker socket, developer home, SSH agent, cloud config, browser
  profile or real GitHub credentials mounted into a job by default.
- Per-job network/filesystem/process/resource isolation, explicit egress policy,
  controlled DNS and no inadvertent reachability to cloud metadata services.
- Path traversal, symlink escape, archive bomb, malicious tar extraction,
  decompression-size and unsafe mount protections at source/blob boundaries.
- Image/action digest validation, policy-based source allowlists, repository
  identity checking, and qualified dependency lockfile enforcement.
- Secret encryption at rest, least-privilege delivery after gates, expiry and
  revocation, redaction before export, and no plaintext secret read endpoint.
- Authentication on private runner/results endpoints, short-lived signed blob
  capabilities and replay/stale-lease prevention.
- Resource limits for processes, memory, CPU, PIDs, disk, log rate and uploads;
  fail visibly rather than freezing the developer machine.
- Minimal structured audit events for policy changes, secret metadata changes,
  approvals, unsafe overrides and external-network/credential grants.

A malicious workflow can deliberately encode or transmit any secret it is
authorized to receive. Masking and log filters do not remove that risk.

### Local OIDC and external services

Expose a local OIDC issuer with discovery/JWKS, short-lived signed tokens,
rotation, audience validation, subject customization and qualified workflow/
repository/ref/actor/environment/reusable-workflow claims. Issuance requires
`id-token: write` in the correct job scope.

The issuer is never `https://token.actions.githubusercontent.com`. A mock
relying party may trust it; real cloud access needs an explicit cloud trust
configuration. Do not issue a success-shaped token pretending to be GitHub.

Attestations, package registries, Pages, deployment targets, private networking
and billing similarly need explicit local substitutes or real provisioned
services. Report the mode and provenance. Organization/enterprise policies and
hosted-runner APIs must not become no-op success endpoints merely because the
runtime is local.

## Performance specification

### Measurement status and reference environments

**All numbers in this section are proposed acceptance budgets. None were measured
by this research.** Adjust them only through recorded benchmark evidence and an
explicit spec change, not by silently changing test workloads.

Reference A: native Linux x64, eight logical CPUs, 16 GiB RAM, local NVMe storage,
supported engine, no CPU emulation, four-job cap. Record actual CPU/storage/OS/
kernel/engine/runtime/image versions with every result.
Reference B: macOS with a Linux VM-backed container engine, the same assigned
CPU/memory budget; publish separate results rather than averaging them with A.
ARM64 has an independent native result set.

Define temperature precisely:

- **Process-cold:** daemon stopped, durable state exists; no running workers.
- **Dependency-cold:** required images/actions/tool archives are absent.
- **Warm:** daemon ready, dependencies locally verified, clean allocations may be
  prewarmed, but no prior job's writable state is reused.
- **Cache-hit:** workflow explicitly restores a previously finalized dependency
  cache; distinct from warm image/source state.

Network pull/download time is reported separately with bytes and network rate,
not hidden from end-to-end first-run measurements.

### Budgets

| Workload on reference A | Proposed gate |
| --- | --- |
| Process-cold daemon to authenticated readiness, no migration | p95 <= 2 seconds |
| Idle daemon plus its storage workers, no runner processes | Combined RSS <= 256 MiB; average CPU <= 1% of one logical CPU over five minutes |
| Indexed list/get of twenty runs from 100,000 stored runs, twenty concurrent clients | p95 <= 50 ms; p99 <= 150 ms; no full-table scan |
| Valid dispatch acknowledgment with dependencies already resolved | p95 <= 150 ms; durable acknowledgment, not in-memory acceptance |
| Plan a 100-job, no-matrix-expansion-dependent shell DAG | p95 <= 250 ms; schema/cache state reported |
| Warm single no-op Linux job, free capacity | Accepted-to-first-step-log p95 <= 2 seconds; accepted-to-terminal p95 <= 5 seconds |
| Four independent warm no-op jobs, four slots | All terminal p95 <= 8 seconds; no accidental global serialization |
| Accepted log chunk to observable tail event | p95 <= 200 ms under four active jobs |
| 1 GiB finalized precompressed blob, local streaming | >= 100 MiB/s with <= 64 MiB additional server RSS per active transfer |
| Ordinary cancellation request delivery to responsive runner | p95 <= 1 second, separate from documented process shutdown grace |
| Namespace teardown of four finished jobs | p95 <= 5 seconds; no leaked owned containers, ports, credentials or volumes |

For reference B, report a provisional <= 2x execution-start/completion budget,
but do not multiply semantic timeouts or excuse API/storage failures. A budget
failure is evidence to optimize or revise a target, not permission to weaken
isolation or return results early.

No universal dependency-cold startup limit is defensible without image size and
bandwidth. Set a distribution goal of <= 1 GiB compressed for the basic
shell/Node development image, and publish the actual size and missing tools.
Full hosted-image/tool parity uses a separate, potentially much larger profile.
Preflight shows transfer size and offline missing dependencies.

### Benchmark suite and methodology

Use fixed source/image/action digests and retain machine-readable results:

| Fixture | Exercises |
| --- | --- |
| Shell no-op, twenty sequential steps | Planner/runner/step startup overhead |
| Checkout plus TypeScript test project | Git service, real JS action, tool cache, dependency-cold and cache-hit cases |
| Four- and sixteen-variant matrices | Scheduler fairness, max-parallel, fail-fast, CPU/memory limits |
| PostgreSQL service plus integration test | Container hooks, health checks, localhost/container networking and cleanup |
| Nested reusable workflow | Context/permissions/source resolution overhead |
| 1 GiB artifact, compressible and precompressed data | Compression CPU versus transfer/storage throughput |
| Exact/prefix/miss dependency cache | Lookup, permissions, upload conflicts and decompression cost |
| 100,000 historical runs and concurrent watchers | Indexes, pagination, outbox tails, memory growth |
| Cancellation, crash and disk-full injection | Recovery latency and resource leaks |

For each temperature/platform, perform at least five warmups and thirty measured
iterations for job workloads. Use at least 10,000 requests for API latency
percentiles. Report p50/p95/p99 where sample size supports them, CPU time, peak and
steady RSS, I/O, transferred bytes, image/storage footprint, failures and queue
wait. Do not report p99 of thirty jobs as a reliable population tail estimate.

Separate queue delay, provisioning, runner startup, user execution, post cleanup
and finalization with monotonic spans. Compare against direct local execution
and optionally `act` on the same hardware to identify overhead; these comparisons
do not establish semantic compatibility. Run a multi-hour soak with repeated
namespace creation/deletion and bounded memory/disk growth.

### Resource and hot-path design

Auto concurrency reserves at least 2 GiB for the host/control plane and uses a
default 2 GiB/job reservation plus declared service memory. Cap active jobs by
available memory, engine/VM limits, CPU and configured maximum. If capacity is
below one job, return an explicit preflight error; do not clamp an impossible
calculation to one. Estimates are visible and user-adjustable.

Apply actual cgroup/VM limits and account for service containers, not only runner
RSS. Swap pressure/OOM is a job infrastructure diagnostic. Large uploads have a
separate concurrency budget so they cannot consume all scheduler resources.

Required optimizations, subject to correctness:

- Cache parsed/validated workflows by content digest + schema/evaluator version,
  never by mutable filename alone.
- Resolve/download immutable source and image content once with bounded
  concurrent fetches and per-object single-flight coordination.
- Use indexed scheduler-ready queries and event-driven wakeups instead of scanning
  every run on every tick.
- Batch log writes and metadata updates within a bounded latency window, preserving
  redaction, order and backpressure.
- Stream blobs/ZIPs, use bounded transfer buffers, and isolate CPU-heavy work.
- Prewarm only pristine allocations; zero job-state reuse.

Metrics must include event-loop delay, writer-queue depth, lease-renewal lateness,
job queue reason, runner startup, container operations, blob throughput, disk
pressure, cache hits and cleanup failures. No external telemetry is enabled by
default.

## Conformance and delivery

### Test strategy

Maintain a corpus of workflow YAML, action metadata, Git fixtures, API exchanges,
protocol messages, expected state transitions and qualified image probes.
Normalize only inherently variable values such as local IDs/URLs/timestamps;
never normalize away different conclusions, authorization, missing fields or
ordering that is part of the contract.

Use four independent layers:

1. **Language/unit:** shared parser/expression fixtures plus new feature cases;
   source diagnostics, coercion, context availability and matrix algorithms.
2. **Protocol/contract:** versioned REST/GraphQL/Git and runner/results/blob fixtures;
   real unmodified clients, not only tests of generated schemas.
3. **Local end-to-end:** actual pinned runner and action SHAs on each advertised
   backend with crash, cancellation and permission fault injection.
4. **Differential:** run controlled workflows on GitHub and locally; compare
   normalized events/results/outputs/API traces. Real GitHub runs are opt-in,
   use an authorized disposable repository and scoped credentials, and respect
   cost/rate limits. Never upload user secrets or source as a test convenience.

Mock mode gets its own conformance suite; passing it cannot satisfy execution
coverage. Hosted differential evidence is needed before claiming behavior that
the public docs or libraries leave ambiguous.

### Minimum acceptance cases

| ID | Required assertion |
| --- | --- |
| C01 | Unchanged workflow dispatches through both REST versions and default-header behavior; exact status/body verified |
| C02 | `gh workflow list/run` and `gh run list/view/watch/cancel/rerun/download` work unmodified, including logs, artifacts and redirects |
| C03 | User `gh` credentials/config/remotes remain byte-for-byte unchanged; local commands cannot send local credentials to GitHub |
| C04 | Checkout obtains the selected real/synthetic SHA through smart Git; no pre-populated workspace shortcut |
| C05 | Expressions, missing values, coercion, `case()`, status checks and location-specific contexts match fixtures |
| C06 | Job `if` precedes matrix expansion; include/exclude, fail-fast, max-parallel and tolerated failures match |
| C07 | Reusable nesting/identity/permissions/secrets/outputs and full-versus-partial re-run ref resolution match |
| C08 | Both concurrency queue modes, case-insensitive groups, overflow, cancellation and all group APIs share consistent state |
| C09 | Environment review/wait/custom gates, self-review restrictions, secret timing and `deployment: false` behave correctly |
| C10 | Fork/privileged-trigger permissions, recursion suppression and source selection prevent unauthorized reads/writes |
| C11 | JS/composite/Docker actions, service networking, pre/post steps, summaries, annotations and command files work |
| C12 | `always()` cancellation, force-cancel, timeout escalation and cleanup have correct outcomes and no surviving processes |
| C13 | Real artifact action upload/download handles digest, overwrite/new ID, retention, archive mode and hidden files |
| C14 | Cache exact/prefix/default/base/PR scopes, versioning, reservations and every `cache-mode` enforce access correctly |
| C15 | Runner reconnection/stale leases, duplicate completion and daemon crash cannot publish two terminal results |
| C16 | Secrets use encrypted write contracts; read APIs/exports never return plaintext |
| C17 | Native lockfiles and newer syntax are enforced, not merely parsed; invalid identities/missing dependencies fail |
| C18 | Time zones/DST, event diff boundaries, dispatch-input limits and documented duration/matrix/output boundaries are tested |
| C19 | All 200 frozen Actions operations plus dependency inventory have schema, authorization, behavior and negative tests before full API parity |
| C20 | Each advertised OS/architecture/image/snapshot capability passes platform probes; unsupported hardware/identity is explicit |
| C21 | Deterministic simulation, replay/fault ordering, namespace isolation and SDK disposal are repeatable |
| C22 | Performance budgets, disk-full behavior, resource accounting and soak-test cleanup pass on published reference machines |
| C23 | Offline mode performs no unapproved DNS/HTTP/source fetch and reports missing dependencies without fallback |
| C24 | Local OIDC is correctly scoped and cannot impersonate GitHub's issuer; mock/cloud trust paths are explicit |

For CLI commands outside the initially probed list, acceptance is future work.
Include `gh cache`, `gh secret` and API pagination/error scripts as their
underlying surfaces are implemented. Current CLI source can perform ancillary
calls; capture them rather than maintaining a guessed route allowlist forever.

### Milestones and dependencies

| Milestone | Deliverable | Exit gate |
| --- | --- | --- |
| M0: feasibility and contract freeze | Runner/protocol spike, versioned source inventory, transport fixtures, backend topology proof and language ADR | Mandatory runner sequence passes; real checkout/artifact/cache/cancellation work; unresolved blockers have an explicit architecture decision |
| M1: narrow vertical slice | Persistent daemon, CLI/repo/source lifecycle, one real Linux workflow, public dispatch/list/get/jobs/logs, isolated `gh` | C01-C04 basic path, C11 shell/JS subset, C12 basic cancellation, C15 crash cases; no full-parity claim |
| M2: integration-testing release | Matrices/DAGs/reuse, services, artifacts/cache, event subset, namespaces, mocks, SDK and performance instrumentation | Applicable C05-C18/C21-C23 pass; all remaining features visibly unsupported; real integration example works end to end |
| M3: Actions language/API parity candidate | Complete documented language/events/policies/environments/runner APIs, all frozen Actions operations and dependent services | C01-C19 pass for the declared product/version and qualified backend; missing external/platform requirements remain explicit |
| M4: platform and external-capability parity | Qualified Windows/macOS/machine backends, snapshots, isolated mode and declared service integrations | C20/C24 plus complete per-profile suite; approved release claim and published compatibility ledger |
| Continuous | Compatibility drift monitoring, security updates, performance regression gates and migration maintenance | Each bundle update passes affected fixtures and documents changes |

M0 precedes implementation commitment. M1 precedes M2; M3 depends on the scheduler,
identity and services built in M2. Backend prototyping can proceed independently
after M0, but a platform cannot be advertised before its full suite passes.
No schedule estimate is asserted without results from the protocol spike.

### Definition of done

A release is done only when:

- A clean machine with documented prerequisites can install, start, execute the
  advertised unchanged workflow, inspect it through `gh`, and cleanly shut down.
- The bundle, capability ledger and conformance results are published and exactly
  identify missing or conditional behavior.
- Unsupported requests fail explicitly, no real execution falls back to simulation,
  and user credentials/configuration remain untouched.
- Persistence, cancellation, secrets, failure recovery and resource cleanup are
  verified, not just the happy path.
- The release's performance goals are measured with retained workloads/results.
- Documentation includes offline preparation, image/capability differences,
  local trust boundaries, API versions, CLI contracts and upgrade/rollback rules.

"It should just work" means predictable setup, correct execution, compatible
clients, useful diagnostics and no hidden changes to the developer's environment.
It does not mean silently claiming unsupported GitHub behavior.
