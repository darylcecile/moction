# Moction runtime: research and decision record

**Research dates:** 2026-10-15 to 2026-10-16

**Status:** Evidence for a proposed implementation, not a compatibility certification

**Related:** [Runtime specification](moction-runtime.md) | [Open questions](moction-runtime-open-questions.md)

## Executive finding

A local GitHub Actions service for mocks and integration testing is feasible.
An offline, containers-only replacement with unconditional parity with every
GitHub Actions feature is not. Workflow syntax, job execution, orchestration,
public APIs, runner images, and externally anchored services are separate
compatibility problems.

The recommended architecture is a **TypeScript control plane and CLI, GitHub's
TypeScript language libraries, and a pinned official Actions runner for job
execution**. Docker-compatible Linux containers provide the fast development
path; native or VM workers provide capabilities that a container cannot reproduce.
The runner protocol adapter must pass a feasibility gate before substantial
implementation depends on it.

Compatibility maintenance and environment fidelity are larger risks than HTTP
server language performance. Replacing TypeScript with Go does not supply the
missing scheduler, artifact protocol, permissions, or macOS machine. There is
no benchmark evidence here that warrants a wholesale lower-level rewrite.

## Method and evidence boundaries

Primary documentation and public implementation sources were read directly.
Search summaries were discovery aids, not evidence. In particular, the claim
that all expression evaluation is proprietary is incorrect: GitHub publishes
expression implementations and workflow-parser libraries. The complete hosted
orchestration service is not supplied by those libraries.

The following snapshots were inspected. A development commit, published package,
released runner, and toolkit bundled into a released action are different
artifacts. The implementation must qualify a consistent bundle, not assume
that inspection of one proves the behavior of another.

| Source | Reviewed revision or release | Use |
| --- | --- | --- |
| `actions/runner` | `796cc18ffaaafd5cea525d6231f5a8c93c11c603` | Job model, broker/run-service clients, container hooks |
| Runner release metadata | `v2.338.0`, published 2026-10-06 | Candidate binary, not execution-tested here |
| `actions/languageservices` | `4043eda158e16579cc5fb1b0b07a4bce2a76f0b5` | TypeScript parser/evaluator |
| `actions/toolkit` | `7e25f67e8fd17369aa21e6e816ac99654106e0dc` | Artifacts, cache, runtime token, host detection |
| `cli/cli` | `3d7967df5a94052b0554898ffa02526f0f90b8c9`; API command also read at `v2.102.0` | Host and request construction |
| `cli/cli` follow-up search | `b758ce35643e64cfe4d61bc58b6009c4088e5e0c` | Workflow dispatch and run-log consumers |
| `cli/go-gh` | `7a33a08c5595f7c61662d1f54ad173de87ca2963` | Unix-socket transport and host/authentication constraints |
| `ChristopherHX/runner.server` | `992ccbbbf9afcde477c38c316e053b1af457ad40` | Runner-backed emulator prior art |
| `nektos/act-docs` | `35cab6107d2afb3269a61dc5de5c15c151a4574f` | Project-documented incompatibilities |
| `github/rest-api-description` | `ac212dce7637bdfe66f169d81b3a04820c59039a` | Complete extraction of `actions/` operations from the bundled description |
| Installed `gh` | `2.102.0` | Local fixture-based transport experiment |

Official documentation URLs below were retrieved on the research dates and are
mutable. The implementation must preserve relevant source snapshots and fixtures
in its compatibility bundle. This research did **not** execute an official
runner against a local orchestration service, benchmark containers, or run
differential workflows on GitHub. Performance numbers in the runtime spec are
acceptance targets, not measured achievements.

## Workflow language

### Reusable components

[`@actions/workflow-parser`](https://github.com/actions/languageservices/blob/4043eda158e16579cc5fb1b0b07a4bce2a76f0b5/workflow-parser/README.md)
parses workflow YAML into a schema-validated intermediate representation;
`convertWorkflowTemplate` produces a more convenient workflow model.
[`@actions/expressions`](https://github.com/actions/languageservices/blob/4043eda158e16579cc5fb1b0b07a4bce2a76f0b5/expressions/README.md)
provides parsing, evaluation, and Actions-specific value types. Both publish
TypeScript types and compiled ECMAScript modules under MIT.

Neither supplies a complete scheduler or every context-dependent runtime
function. Their documentation describes them as individual implementations of
shared behavior with shared test data. A particular package release may lag
current documented syntax.

The [maintenance notice](https://github.com/actions/languageservices/blob/4043eda158e16579cc5fb1b0b07a4bce2a76f0b5/README.md)
states that ordinary contributions are not currently being accepted, while
security updates and major breaking fixes remain in scope. Budget for a small,
tested compatibility patch set; do not assume immediate upstream acceptance.

### Features a historical emulator design would miss

The [workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax),
[expressions](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions),
[contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts),
and [reuse](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)
references establish these implementation requirements:

| Finding | Consequence |
| --- | --- |
| Job `if` precedes matrix expansion. | `matrix` must not become available in job-level `if`. |
| Context/function availability depends on the YAML location. | A globally populated context dictionary is incorrect. |
| Expressions have special coercions, case-insensitive string comparison, and implicit status checks. | JavaScript `eval`, truthiness, or equality cannot substitute. |
| `case()` is documented. | An old fixed list of expression functions can reject valid workflows. |
| YAML anchors and aliases are supported. | Blanket historical rejection of aliases is stale; this does not imply support for merge keys or arbitrary YAML tags. |
| Reusable workflows allow ten levels and fifty unique reusable workflows. | Do not preserve historical four-level/twenty-workflow limits. |
| Dispatch allows twenty-five top-level input properties and 65,535 characters. | The older ten-input assumption is stale. |
| Path-filter documentation describes a 3,000-file diff boundary. | Do not copy the historical 300-file limit. |
| Schedules accept IANA `timezone`, with documented DST behavior. | UTC-only cron parsing is insufficient. |
| `concurrency.queue` accepts `single` or `max`. | A single mutex is not the concurrency model. |
| Workflow/job `cache-mode` controls scoped cache access. | Cache capability is distinct from repository token permissions. |
| `jobs.<id>.snapshot` creates custom images. | It is a backend capability, not an ignorable key. |
| `environment.deployment: false` separates environment use from deployment creation. | Secret gates and deployment records need separate state. |
| `$/` action references resolve the action's source repository at its running commit. | Not every relative-looking reference is a workspace path. |

The [concurrency guide](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)
specifies case-insensitive repository-scoped groups, at most one running member,
replacement of the pending member under `single`, and up to 100 pending members
under `max`. `queue: max` and `cancel-in-progress: true` are incompatible. Ordering
is based on entry into the waiting group, not necessarily dispatch order.

The [limits reference](https://docs.github.com/en/actions/reference/limits)
includes a 256-job matrix limit, 500 KB workflow files, 35-day workflow duration,
30-day approval wait, 24-hour self-hosted queue duration, six-hour hosted jobs,
five-day self-hosted execution, and fifty re-runs. Store limits in the
compatibility bundle. Ambiguous aggregate-versus-per-matrix wording and exact
byte boundaries require boundary fixtures, not guesses.

The [action metadata reference](https://docs.github.com/en/actions/reference/workflows-and-actions/metadata-syntax)
distinguishes JavaScript, composite, and Docker actions, documents `node20` and
`node24`, pre/post conditions and composite-input differences, and notes that
`required: true` on an action input does not itself enforce a runtime failure.
These are reasons to reuse the runner instead of creating a new step executor.

### Events, re-runs, and cancellation

The [event reference](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
defines activity types, workflow-source selection, ref/SHA behavior, and
default-branch presence requirements. Not every webhook event triggers workflows.

The [re-run guide](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/re-run-workflows-and-jobs)
retains the original event SHA/ref and original actor's privileges, distinguishes
that actor from the actor requesting a re-run, and specifies a thirty-day re-run
window and fifty-re-run limit. The reuse reference additionally distinguishes
full re-runs, which can resolve a mutable reusable-workflow reference again,
from failed/specific-job re-runs, which reuse the original called commit.

The [cancellation reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-cancellation)
requires condition re-evaluation. The runner interrupts a step, waits 7.5 seconds,
terminates it, waits 2.5 seconds, then kills its process tree if needed; the server
has a five-minute cancellation timeout. Ordinary cancellation and force
cancellation are not the same operation.

### Dependency lockfiles

The [lockfile documentation](https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles)
describes a public-preview format at `.github/workflows/actions.lock`, generated
by `gh actions-lock`. The documented example is version `v0.0.2`.
`workflows` maps onboarded workflow paths to dependencies; `dependencies` records
the reference, `sha1-...` commit representation, and numeric owner/repository IDs.

For an onboarded workflow every dependency must be present. The locked commit
must exist within a branch of the source repository, and source identity is
checked. Transitive action dependencies are covered; a reusable workflow's own
dependencies are governed by the lockfile in its defining repository. Moction
must honor this rather than introduce a competing Actions lockfile format.
The exact generator output and rejection cases still need executable fixtures.

## Runner and service protocols

The runner is an executor, not a complete local Actions service.
[`AgentJobRequestMessage`](https://github.com/actions/runner/blob/796cc18ffaaafd5cea525d6231f5a8c93c11c603/src/Sdk/DTPipelines/Pipelines/AgentJobRequestMessage.cs)
receives a prepared plan, typed contexts, template tokens, steps, variables,
mask hints, resources, timeline, outputs, and environment information.

| Client | Observed protocol |
| --- | --- |
| [`BrokerHttpClient`](https://github.com/actions/runner/blob/796cc18ffaaafd5cea525d6231f5a8c93c11c603/src/Sdk/WebApi/WebApi/BrokerHttpClient.cs) | Session creation/deletion, `message` polling, `acknowledge`, typed broker errors, WebSocket connectivity-probe path |
| [`RunServiceHttpClient`](https://github.com/actions/runner/blob/796cc18ffaaafd5cea525d6231f5a8c93c11c603/src/Sdk/RSWebApi/RunServiceHttpClient.cs) | `acquirejob`, `renewjob`, `completejob`; acquisition distinguishes missing, already acquired, and unprocessable jobs |

These are not public `/repos/.../actions` endpoints. The adapter also needs
bootstrap/authentication, action resolution, logs, timeline/results publication,
completion, and cancellation. Sending YAML to the official runner does not
provide those services. No claim is made that the few inspected methods are a
complete registration or execution protocol.

GitHub documents Linux-only, public-preview
[container customization hooks](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/customize-containers):
`prepare_job`, `cleanup_job`, `run_container_step`, `run_script_step`.
They offer a scoped container-management boundary without modifying action code,
but their mounts, argument handling, response files, health checks and context
updates must be qualified against the selected runner.

## Artifact and cache protocols

An artifact-list REST endpoint does not make `actions/upload-artifact` work.
The toolkit uses separate Twirp services and signed blob URLs:

| Contract | Primary source |
| --- | --- |
| Artifact create/finalize/list/download-URL/delete | [Artifact Twirp client](https://github.com/actions/toolkit/blob/7e25f67e8fd17369aa21e6e816ac99654106e0dc/packages/artifact/src/generated/results/api/v1/artifact.twirp-client.ts) |
| Cache create/finalize/download-URL | [Cache Twirp client](https://github.com/actions/toolkit/blob/7e25f67e8fd17369aa21e6e816ac99654106e0dc/packages/cache/src/generated/results/api/v1/cache.twirp-client.ts) |
| Runtime-token backend IDs | [Artifact token utility](https://github.com/actions/toolkit/blob/7e25f67e8fd17369aa21e6e816ac99654106e0dc/packages/artifact/src/internal/shared/util.ts) |
| Origin extraction and host classification | [Artifact configuration](https://github.com/actions/toolkit/blob/7e25f67e8fd17369aa21e6e816ac99654106e0dc/packages/artifact/src/internal/shared/config.ts) |

The runtime token must contain the backend IDs expected in an
`Actions.Results:<run-backend-id>:<job-backend-id>` scope. It cannot be an
arbitrary opaque string. The inspected artifact client uses the **origin** of
`ACTIONS_RESULTS_URL`; path-only tenancy cannot be relied on.

The inspected toolkit treats hosts ending in `.localhost` as non-GHES, unlike an
arbitrary custom hostname. This exception must be verified in the toolkit actually
bundled with every supported action release. Bare `localhost` is not the same
suffix. The [upload action documentation](https://github.com/actions/upload-artifact/blob/main/README.md)
still states that v4+ is unsupported on GHES, so enterprise-shaped REST routing
must not inadvertently select an unsupported artifact backend.

Current upload documentation includes v7, immutable artifacts, overwrite as
replacement with a new ID, hidden-file exclusions, single-file `archive: false`,
compression choices, digests, and a 500-artifact-per-job limit.
The [cache action documentation](https://github.com/actions/cache/blob/main/README.md)
describes v6 on its default branch, the v2 service migration, runtime requirements,
and read-only-token behavior. A baseline of "Node 16 actions and artifact v3"
would be obsolete.

The [dependency caching reference](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching)
defines exact/prefix matching, ordered restore keys, cache versions, default/base
branch visibility, PR merge-ref isolation, and trigger-dependent default-branch
write restrictions. `cache-mode` adds independent read/write capabilities. Do not
reduce these rules to "all pull requests can never write a cache": merge-ref and
default-branch scopes differ.

Archive formats and client compression remain client-owned. The server must
preserve bytes and permissions rather than replace official actions with
special-case Moction implementations.

The [checkout action documentation](https://github.com/actions/checkout/blob/main/README.md)
also affects bundle selection: current v7 adds protections around unsafe fork
checkout in privileged workflows; v6 moved persisted credentials under runner
temporary storage, and authenticated Git from container actions requires runner
`2.329.0` or later. Node24-based actions require a sufficiently recent runner
(`2.327.1` for the documented runtime transition). The cache v2 migration has its
own older minimum (`2.231.0`). Select a mutually qualified bundle rather than
treating any one minimum as sufficient for all current actions.

## Public API surface

The [OpenAPI repository](https://github.com/github/rest-api-description/blob/main/README.md)
publishes stable bundled OpenAPI 3.0 descriptions under MIT. It notes incomplete
header coverage and alias/multi-segment-path limitations. Generate contracts from
the frozen description, then supplement them with behavioral fixtures.

The complete JSON at
[`descriptions/api.github.com/api.github.com.json`](https://github.com/github/rest-api-description/blob/ac212dce7637bdfe66f169d81b3a04820c59039a/descriptions/api.github.com/api.github.com.json)
was parsed. Counting operations whose `operationId` starts with `actions/`
produced **200 operations**:

| OpenAPI subcategory | Operations | Runtime responsibility |
| --- | ---: | --- |
| `artifacts` | 5 | Metadata, access, downloads and deletion |
| `cache` | 18 | Entries, quotas, usage, retention and administration |
| `concurrency-groups` | 3 | Observable workflow/job queues |
| `hosted-runners` | 16 | Local resource model and provisioned backend capabilities |
| `oidc` | 2 | Repository subject customization |
| `permissions` | 36 | Repository/organization Actions policy and access controls |
| `policies` | 10 | Repository/organization policy resources |
| `self-hosted-runner-groups` | 14 | Group membership and repository/workflow access |
| `self-hosted-runners` | 26 | Registration, lifecycle, labels and JIT configuration |
| `secrets` | 20 | Encrypted write APIs, metadata and visibility |
| `variables` | 20 | Scoped configuration and visibility |
| `workflow-jobs` | 5 | Job/step records and log downloads |
| `workflow-runs` | 19 | Lifecycle, attempts, approvals, gates and usage |
| `workflows` | 6 | Discovery, dispatch, enablement and usage |
| **Total** | **200** | **Target inventory, not implemented coverage** |

Category membership follows upstream metadata: a route containing `/jobs` may
belong to `workflow-runs`. This count is not all of GitHub, all enterprise
descriptions, or all Actions dependencies. Environment, deployment, checks,
Git/content, identity, organization OIDC, and custom-image operations can live
under other categories or descriptions and must be added to the dependency
inventory. Preview and future additions need explicit bundle updates.

The [workflow API](https://docs.github.com/en/rest/actions/workflows),
[run API](https://docs.github.com/en/rest/actions/workflow-runs),
[version guide](https://docs.github.com/en/rest/about-the-rest-api/api-versions),
and [breaking-change register](https://docs.github.com/en/rest/about-the-rest-api/breaking-changes)
identify two relevant versions:

| Version | Workflow dispatch success |
| --- | --- |
| `2022-11-28`, including requests without a version header | `204` without run details; `200` with `return_run_details: true`, as confirmed in the frozen OpenAPI |
| `2026-03-10` | Always `200` with `workflow_run_id`, `run_url`, `html_url`; removes `return_run_details` |

The older version is documented as supported through 2028-03-10. The rendered
workflow endpoint page showed newer behavior even when requested with an older
`apiVersion` query parameter; it is not sufficient evidence of the old contract.
Other verified contracts include `202` cancellation, `201` re-run, and `302` job-log
download. Global "all writes return 200" behavior would break compatibility.

[Concurrency-group endpoints](https://docs.github.com/en/rest/actions/concurrency-groups)
use cursor pagination and distinguish inactive groups from historical
run-associated groups. [Secret endpoints](https://docs.github.com/en/rest/actions/secrets)
use LibSodium-encrypted values and do not return secret contents. Both need real
semantics, not schema-shaped placeholders.

## GitHub CLI transport experiment

The [CLI environment guide](https://cli.github.com/manual/gh_help_environment)
documents `GH_HOST`, `GH_REPO`, `GH_CONFIG_DIR`, and enterprise token variables.
Setting only `GH_HOST` is insufficient when repository inference still selects
the checkout's real GitHub remote.

Source inspection established:

- [Host handling](https://github.com/cli/cli/blob/3d7967df5a94052b0554898ffa02526f0f90b8c9/internal/ghinstance/host.go)
  builds enterprise REST URLs under `/api/v3/` and GraphQL under `/api/graphql`.
  GitHub.com REST uses `https://api.github.com/`, not `/api/v3`.
- The host validator rejects colons and slashes;
  [`gh api`](https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/api/api.go)
  applies it to explicit `--hostname`.
- [API request construction](https://github.com/cli/cli/blob/3d7967df5a94052b0554898ffa02526f0f90b8c9/pkg/cmd/api/http.go)
  preserves absolute URLs. This works for a loopback API call but does not
  configure `gh run` and `gh workflow`.
- [`go-gh` options](https://github.com/cli/go-gh/blob/7a33a08c5595f7c61662d1f54ad173de87ca2963/pkg/api/client_options.go)
  constrain `api_host` to a hostname without a scheme or port.
- [`go-gh` transport](https://github.com/cli/go-gh/blob/7a33a08c5595f7c61662d1f54ad173de87ca2963/pkg/api/http_client.go)
  reads `http_unix_socket` and routes ordinary and TLS dials directly to that
  socket. This does not disable certificate checks on unrelated network clients.

### Executed experiment

On Linux, an ephemeral standard-library HTTP fixture listened on a Unix socket.
An isolated `GH_CONFIG_DIR/config.yml` contained `http_unix_socket: <socket>`.
The child process had a temporary home, local-only dummy enterprise token,
`GH_HOST=moction.localhost`, and
`GH_REPO=moction.localhost/fixture/repo`. Real user credentials were not inherited.
The fixture recorded method, host, path, API version, and whether local
authentication was received, never credential contents.

| Unmodified `gh 2.102.0` invocation | Observed result |
| --- | --- |
| `gh api repos/fixture/repo/actions/runs --jq .total_count` | `GET /api/v3/repos/fixture/repo/actions/runs`; fixture result `1` |
| `gh run list --json databaseId,status,conclusion` | Retrieved runs and workflow metadata; returned completed fixture run 42 |
| `gh workflow list --json id,name,state` | Retrieved `/api/v3/.../actions/workflows`; returned workflow 7 |
| `gh api graphql -f 'query=query { viewer { login } }'` | `POST /api/graphql`; returned fixture viewer |
| `gh api ... --hostname moction.localhost` | Accepted and routed locally |
| Same basic commands with `GH_HOST=localhost:4317` over the socket | Accepted in this environment-variable path |
| `gh api ... --hostname localhost:4317` | Rejected before sending a request: invalid hostname |

All successful requests carried the local token and `2022-11-28` API-version
header. Temporary fixture/configuration state was removed on exit.

This proves a **transport strategy and these basic consumers**, not a Moction
runtime, a full GraphQL server, dispatch, watches, artifacts, redirects, TLS over
TCP, or Windows support. The production implementation must promote the probe
into retained conformance tests. A bare hostname plus isolated Unix socket is the
recommended POSIX `moction exec -- gh ...` implementation.

Follow-up source inspection of
[`workflow run`](https://github.com/cli/cli/blob/b758ce35643e64cfe4d61bc58b6009c4088e5e0c/pkg/cmd/workflow/run/run.go)
and [`run view`](https://github.com/cli/cli/blob/b758ce35643e64cfe4d61bc58b6009c4088e5e0c/pkg/cmd/run/view/view.go)
shows why a broader suite is needed: dispatch can request run details, workflow
content is read, and log viewing fetches attempts/jobs/workflows and parses ZIP
layout, with per-job fallback requests.

## Images, isolation, and alternatives

[`actions/runner-images`](https://github.com/actions/runner-images/blob/main/README.md)
contains hosted VM image definitions and changing tool inventories. An Ubuntu
container is not that machine. OS release, x64/arm64, shells, tool cache, filesystem,
kernel, Docker access and hardware must be part of an environment claim.

[Docker security](https://docs.docker.com/engine/security/) documents the privileged
nature of daemon access. [Rootless mode](https://docs.docker.com/engine/security/rootless/)
reduces daemon privileges but needs host support and does not establish a hostile
multi-tenant boundary by itself.
[`Firecracker`](https://github.com/firecracker-microvm/firecracker/blob/main/README.md)
uses Linux KVM and is a plausible future isolation backend, not a universal laptop
dependency or a mechanism for running macOS in Linux containers. Its performance
claims are not Moction measurements.

The [custom-image guide](https://docs.github.com/en/actions/how-tos/manage-runners/larger-runners/use-custom-images)
defines successful-job snapshot creation, string/mapping forms, conditional
snapshots, automatic minor versions, and inherited expiration. Local image
creation must model these rules, not call `docker commit` and claim equivalence.

| Candidate | Useful reuse | Decision |
| --- | --- | --- |
| `nektos/act` | Docker-first execution, user experience, fixtures, comparison baseline | Not the default engine: its [documented gaps](https://github.com/nektos/act-docs/blob/35cab6107d2afb3269a61dc5de5c15c151a4574f/src/not_supported.md) include ignored/incomplete concurrency, permissions, environments, timeouts, cancellation, summaries and more. This is project documentation, not a fresh audit of each code path. |
| `ChristopherHX/runner.server` | Scheduler, matrices, reusable workflows, contexts, artifact/cache services and protocol prior art | Research/reference candidate, not an assumed drop-in. Its [README](https://github.com/ChristopherHX/runner.server/blob/992ccbbbf9afcde477c38c316e053b1af457ad40/README.md) describes a modified runner, optional registration protection, and missing protections in some fork configurations. Public API and security equivalence require separate proof. |
| Official runner alone | Step/action execution, shells, command files, cleanup | Reuse it, but implement the missing control plane. |
| New TypeScript step executor | One implementation language | Reject initially: unnecessary reimplementation of subtle runner behavior. |
| All-.NET control plane | Closer source/serialization reuse with runner | Viable fallback if the adapter spike disproves the TypeScript approach; do not retain two schedulers. |
| Go/Rust control plane | Potential distribution, memory or CPU advantages | No evidence yet that they outweigh TypeScript library reuse. Isolate only a measured bottleneck. |
| Firecracker-first | Stronger isolation on suitable Linux hosts | Optional later backend; provisioning/platform costs conflict with low-friction local startup. |

## Security and external identity

The [OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)
defines issuer, audience, subject, workflow, actor and reusable-workflow claims.
A local issuer can reproduce claims for a mock relying party, but cannot sign as
`https://token.actions.githubusercontent.com`. Existing cloud policies will not
automatically trust it.

The [secure-use reference](https://docs.github.com/en/actions/reference/security/secure-use)
explains privileged-trigger/fork risks, Docker-socket exposure, and the limits of
masking. The [environment reference](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
requires protection gates before environment secrets become accessible. It also
distinguishes reviewers, wait timers, branch/tag rules and bypass policy.

Local attestations, package registries, Pages, deployments, cloud credentials,
GPU hardware, private networking, Copilot requests and real billing require a
declared substitute or provisioned capability. Successful placeholders do not
establish parity.

## Decisions and remaining feasibility gates

The evidence supports TypeScript for the application, runner reuse for execution,
separate public/private protocols, a versioned compatibility bundle, explicit
simulation, and conformance-driven releases.

The following are required implementation investigations, not results established
by this research:

| Gate | Evidence required |
| --- | --- |
| Runner provisioning and complete wire inventory | Unmodified pinned runner registers, executes, cancels, reconnects and completes locally without GitHub private services. |
| API-version behavior | Version-specific fixtures cover statuses, dispatch details, errors, redirects, pagination and omitted headers. |
| Full CLI contract | Dispatch, watch, logs, artifacts, cache and secret commands work on every advertised platform, including download redirects. |
| Actual action bundles | Supported action SHAs accept the local origin and use the advertised Node/runtime/blob protocols. |
| New language keys and lockfiles | Parser, scheduler, runner and source identity behavior agree; permissive parsing alone is insufficient. |
| Full API dependency inventory | All 200 frozen Actions operations plus enterprise/dependent surfaces have semantics, authorization, owners and tests. |
| Execution fidelity | Backend/image manifests and probes establish exactly which machine capabilities are equivalent. |
| Performance | Reproducible cold/warm, latency, resource and streaming measurements satisfy the runtime spec. |
