# Idea Board GitOps deployment state

Argo CD reads desired application manifests from this repository. Application source, Terraform infrastructure and build/publish automation live in [om-idea-board](https://github.com/surendarrajasekaran/om-idea-board).

| Path | Purpose |
| --- | --- |
| `charts/nucleus` | Supplied nucleus 1.1.0 chart, vendored unchanged |
| `charts/config` | Frontend/backend chart input values |
| `charts/nucleus-source.json` | Chart provenance and content hashes |
| `bootstrap/idea-board-test.yaml` | Scoped Argo AppProject and Application |
| `environments/<target>/manifests` | Current rendered desired state consumed by Argo |
| `environments/<target>/release.json` | Image digests, source commit, resource profile and verified public URL |
| `environments/<target>/status.json` | Verified healthy release history and pending recovery guard |
| `releases/<target>/<id>` | Immutable release snapshots with checksums |
| `automation` | Reviewed copies of the source repository health/AI helpers |
| `.github/workflows/health.yml` | New-release and scheduled HTTP assessment and guarded Git recovery |

The application pipeline tests, builds and scans immutable ECR image digests before publishing desired state here through a dedicated write deploy key. Argo reconciles Git from inside private EKS. The health workflow has no AWS or Kubernetes credentials. Its GitHub token may update this repository, and normal fast-forward pushes reject concurrent stale observations.

Configure `AI_PROVIDER=nvidia`, `AI_MODEL=nvidia/llama-3.3-nemotron-super-49b-v1.5` and optional secret `AI_API_KEY` in the `aws-test` GitHub environment. Without a key or usable provider response, results explicitly use deterministic policy fallback. No live model result is implied by configuration. NVIDIA trial access is for prototyping and remains subject to provider quotas and terms.

The monitor requires a public URL and matching frontend/backend release identities. New releases must pass readiness and a real create/read smoke test. On failure it commits previously verified application images/resources while preserving current Secret references and security controls. The first failed release has no automatic recovery target. A recovery commit must converge and pass health checks before being called successful. Database migrations/data and infrastructure are never rolled back automatically.

Secrets, Terraform state, credentials and certificates must never be committed here. Argo's project denies Secret and cluster-resource management. Namespace creation, Restricted Pod Security labels and database credential/CA Secrets are bootstrapped through an authenticated private API session.

For setup, private administration, provider choices and limitations see the [GitOps guide](https://github.com/surendarrajasekaran/om-idea-board/blob/main/docs/gitops.md). Changes to the health implementation should first be reviewed and tested in the application repository, then synchronized into `automation/`. Changing chart defaults requires a source pipeline release to regenerate manifests. Editing an archived snapshot invalidates its checksum and prevents recovery.

Initial target: `idea-board-aws-test`, namespace `idea-board-test`, AWS `mit-qa` / `us-west-2`. The test board is anonymous and uses HTTP; use synthetic data. Argo's UI remains private.
