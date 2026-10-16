# moction

Moction is a proposed local GitHub Actions runtime for mocks and integration
testing, with a GitHub-compatible API and a CLI.

**Status:** specification only; the runtime and proposed commands are not implemented.

- [Runtime specification](docs/specs/moction-runtime.md): architecture, workflow
  semantics, API/CLI contracts, execution, persistence, performance, and delivery gates.
- [Research and decisions](docs/specs/moction-runtime-research.md): primary sources,
  alternatives, compatibility findings, and the local GitHub CLI transport experiment.
- [Open questions](docs/specs/moction-runtime-open-questions.md): unresolved decisions,
  proposed defaults, and when each decision is needed.

The target is versioned, demonstrable Actions compatibility, not an unsupported
claim that a Linux container can reproduce every GitHub-hosted platform and service.