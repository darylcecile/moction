# Moction runtime: open questions

**Status:** Decisions to resolve during implementation, not blockers to this specification

**Related:** [Runtime specification](moction-runtime.md) | [Research](moction-runtime-research.md)

The specification uses the proposed defaults below so implementation can begin.
Questions are separated from the normative design as requested. Defaults are
recommendations, not decisions already made by the repository owner. Questions
marked **gate** must be resolved before the corresponding capability or release
claim ships.

## Q1. What does the full-parity release claim mean?

**Question:** Is the intended claim full observable Actions behavior for a
declared GitHub product/version and provisioned platform, or unconditional
equivalence to all GitHub-hosted services on an offline laptop?

**Proposed default:** Keep the full documented Actions language/API/orchestration
target, including currently unsupported features, in a versioned ledger. Claim
equivalence only for tested capability combinations. Do not call the initial
Linux release full parity. Local OIDC/cloud substitutes are explicitly simulated
or separately trusted rather than advertised as GitHub's identity.

**Why it matters:** Unconditional offline equivalence is not technically
achievable for hardware, OS kernels, external issuers and paid cloud services.
Removing those requirements silently would also fail the original request.

**Decision needed:** Before release naming/marketing and M3 acceptance. **Gate.**

## Q2. Which host and runner platforms must ship first?

**Question:** Must the first useful release support native Windows hosts and
Windows/macOS jobs, or can those follow the Linux-container development path?

**Proposed default:** Linux and macOS hosts running native-architecture Linux
containers first; WSL2 uses the Linux CLI path. Windows/macOS jobs require
qualified machine backends and stay in the full target. Native Windows `gh`
transport is a separate feasibility gate.

**Why it matters:** Linux containers cannot reproduce Windows/macOS execution,
and the verified Unix-socket `gh` approach is not proof of native Windows support.

**Decision needed:** M0 distribution/backend plan. **Gate for any advertised platform.**

## Q3. Is hostile workflow code a first-release requirement?

**Question:** Is Moction initially for trusted local projects, or must it safely
execute arbitrary fork pull requests and multiple mutually untrusted users?

**Proposed default:** Trusted local workflows by default. Low-trust execution
requires a stronger isolated backend and explicit fork policy. No unrestricted
host Docker socket, developer home or cloud credentials in job containers.

**Why it matters:** Rootless containers reduce some risks but are not a complete
hostile multi-tenant boundary. Stronger isolation changes startup time, required
hardware, networking and implementation scope.

**Decision needed:** M0 threat model; before accepting untrusted code. **Gate.**

## Q4. How much hosted-machine fidelity is necessary by default?

**Question:** Should `ubuntu-latest` prioritize a small, fast local image or a
larger machine profile matching GitHub's tool inventory and machine behavior?

**Proposed default:** A clearly identified Linux-container development profile,
with pinned alias mappings, a published tool manifest and visible differences.
Offer `--require-equivalence` and a machine profile rather than pretending that
a minimal container is a hosted runner.

**Why it matters:** Small images improve first-run time, but arbitrary workflows
can assume preinstalled tools, kernel features, Docker, privileged operations or
full-machine snapshot support.

**Decision needed:** M0 image/topology spike and before the M2 public default.

## Q5. Which Docker behaviors must work inside workflow steps?

**Question:** Do initial workloads need arbitrary `docker`/Compose/Testcontainers
commands, privileged builds, nested daemons or only runner-managed services and
container actions?

**Proposed default:** Runner-managed containers use scoped hooks/broker access.
Arbitrary daemon use requires a qualified job-local rootless daemon or isolated
machine. Host-engine access is an explicit unsafe opt-in, never an invisible fix.

**Why it matters:** Container hooks alone do not provide a Docker daemon to shell
steps. Treating the host socket as an implementation detail creates a major
security and reproducibility difference.

**Decision needed:** M0 backend proof and representative integration fixtures.

## Q6. What is the fallback if the unmodified runner adapter fails?

**Question:** Is a small C# bridge or maintained runner patch acceptable if a
pure TypeScript service cannot reliably satisfy the pinned private protocols?

**Proposed default:** Attempt the unmodified official runner first. Stop at a
documented M0 failure and choose between a minimal typed bridge, a tightly scoped
runner patch or an all-.NET control plane. Keep the TypeScript CLI/SDK where
useful. Do not introduce a second scheduler or silently switch to `act`.

**Why it matters:** The official runner protocol is the largest technical risk;
its complete wire/bootstrap behavior was not executed by this research.

**Decision needed:** At the end of M0, before general implementation. **Gate.**

## Q7. Should normal runs be online or offline by default?

**Question:** May ordinary development runs fetch actions/images/dependencies and
reach public services, or should all external access require explicit preparation?

**Proposed default:** Visible online source fetching and job egress for normal
trusted development. Tests can select strict offline mode with a preloaded
dependency bundle and named local mock services. No cloud credentials are
inherited in either case.

**Why it matters:** Offline-first improves determinism but cannot "just work" for
an uncached workflow that downloads packages. An offline flag that quietly falls
back online is worse than either explicit policy.

**Decision needed:** M1 UX and M2 integration-testing defaults.

## Q8. Should the local CLI run the worktree or only committed code?

**Question:** Should `moction run` include uncommitted tracked changes and
untracked non-ignored files?

**Proposed default:** Yes, through a visible synthetic commit in Moction-owned
Git storage, with preview/exclusions and no modification of the user's index.
GitHub-compatible API dispatch uses committed registered refs by default.
Strict equivalence requires committed/event-accurate sources.

**Why it matters:** Worktree runs fit local development but must not pretend that
`GITHUB_SHA` names the original commit while executing different files. Large
untracked files and accidentally included credentials also need protection.

**Decision needed:** M1 source/checkout contract.

## Q9. Which GitHub clients are release gates?

**Question:** Beyond `gh api`, which `gh` commands, SDKs and integrations must
work without modification in the first release?

**Proposed default:** Progress from the verified list/API transport to workflow
dispatch; run view/watch/cancel/re-run/log/download; then cache, secret and
variable clients. Qualify a current and a previous supported `gh` release.
Add Octokit contract tests for implemented Actions routes.

**Why it matters:** Clients call ancillary APIs and consume absolute URLs,
redirects, archives and GraphQL. A successful `gh api` request alone does not
prove those behaviors.

**Decision needed:** M1 client matrix; expand at M2/M3.

## Q10. How should native Windows and non-`gh` clients reach the API?

**Question:** Are explicit loopback URLs acceptable for ordinary clients, and
what installation/trust changes are acceptable if native Windows `gh` cannot
use the POSIX transport?

**Proposed default:** Unix socket for POSIX `gh`, an optional authenticated
loopback TCP origin for generic clients, private TLS-protected runner origins,
and WSL2 until native Windows routing is proven. Never change system DNS/trust,
disable TLS verification or reserve privileged ports without consent.

**Why it matters:** `gh --hostname` rejects `host:port`; the successful
environment-variable/Unix-socket experiment does not resolve every platform.

**Decision needed:** M0 networking ADR and before native Windows distribution. **Gate.**

## Q11. Which external GitHub or cloud products need local substitutes?

**Question:** Must the first integration-testing release emulate package
registries, Pages, deployments, attestations, cloud OIDC, GPU/private networking
or only core Actions services?

**Proposed default:** Fully retain these features in the compatibility ledger.
Initially offer explicit mock relying parties/services or return a capability
error. Real integrations require separate, deliberately provisioned identities
and resources; local tokens never impersonate GitHub's issuer.

**Why it matters:** An arbitrary action can call unrelated GitHub or cloud APIs.
"Any workflow action" is not the same bounded task as "all Actions endpoints."
Success-shaped placeholders would conceal broken integration tests.

**Decision needed:** M2 fixture ecosystem and M3/M4 scope. **Gate for each claim.**

## Q12. What real-GitHub differential testing is authorized?

**Question:** Can the project maintain a disposable GitHub repository and scoped
credentials for contract comparisons, and what cost/rate limits apply?

**Proposed default:** Local fixture tests always run. Hosted differential tests
are explicit opt-in, use project-owned synthetic workflows/data, and have a
configured run/minute budget. No user repository, private source or credentials
are uploaded merely to test parity.

**Why it matters:** Some scheduling, error, matrix and private-protocol details
cannot be established by schemas alone. Full parity claims need independent
observations, but creating remote runs incurs side effects and potentially cost.

**Decision needed:** M0 test governance; required before M3 certification. **Gate.**

## Q13. Which performance budgets are product requirements?

**Question:** Are the proposed warm five-second no-op completion, 256 MiB idle
control plane and 1 GiB basic-image size the right tradeoffs for target machines?

**Proposed default:** Treat them as falsifiable acceptance targets, not promises
or measurements. Benchmark native Linux first and publish macOS/ARM64 results
separately. Never meet a latency target by reusing dirty job state, skipping
durability or weakening isolation.

**Why it matters:** The protocol/image spike may reveal that startup, daemon
nesting or the tool inventory dominates. A useful developer experience needs
both warm-loop performance and transparent first-download cost.

**Decision needed:** After M0 measurements, before M2 performance gating.

## Q14. What should be retained between local test runs?

**Question:** Should default instances preserve thirty days of run/artifact data
and a 10 GiB cache, or favor automatic disposal and a smaller disk budget?

**Proposed default:** Persistent named development instances use those visible
limits; SDK-created test namespaces are disposable with a TTL for crashes.
Immutable dependency/image caches can be shared without sharing credentials
or writable workspaces. Provide dry-run GC and explicit exports.

**Why it matters:** Caches accelerate development but consume disk and can
conceal nondeterministic tests. Retention must remain compatible with repository
settings when testing that behavior.

**Decision needed:** M1 storage UX and M2 SDK defaults.

## Q15. Is local approval interaction enough, or is a broader web UI required?

**Question:** Must users manage approvals and inspect runs through a graphical
interface immediately, or are CLI/API interactions sufficient?

**Proposed default:** CLI/API management plus minimal local read-only run/workflow
pages so `html_url` values are usable. Add approval controls only with proper
principal authorization and CSRF protection; do not make a dashboard a dependency
of the scheduler.

**Why it matters:** A rich UI is separate product work, but URLs returned to
clients must not point to nonexistent pages or the real GitHub repository.

**Decision needed:** M1 API projection and M2 UX scope.

## Q16. How should new GitHub features and legacy versions be maintained?

**Question:** How many runner/action/API bundles should remain supported, and
how quickly must new workflow syntax become available?

**Proposed default:** Pin each release to a declared bundle, support the researched
REST versions explicitly, and qualify a bounded current/previous client-action
matrix. Never silently upgrade running instances or reinterpret historic
attempts. Deprecations need migration notes and a published support window.

**Why it matters:** Private protocols, preview lockfiles, container hooks,
toolkits and schema keys can change independently. "Latest" is not a stable
implementation target.

**Decision needed:** M0 manifest format and M3 maintenance policy.

## Q17. What redistribution and provenance checks belong in releases?

**Question:** Will Moction distribute runner binaries/tool images, fetch them
on demand, or require users to supply some proprietary/platform components?

**Proposed default:** Ship signed Moction artifacts, fetch verified upstream
runner/image content where permitted, and retain source/license notices and
provenance. Review redistribution terms for every image/tool bundle and platform
before publishing it. Do not assume that a repository's open-source license
covers all bundled third-party tools or hosted images.

**Why it matters:** A reproducible "one command" installation needs a legitimate,
maintained supply chain as well as working code.

**Decision needed:** M0/M1 packaging; before distributing each image or binary. **Gate.**
