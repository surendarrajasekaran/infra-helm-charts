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

Configure `AI_PROVIDER=gemini`, `AI_MODEL=gemini-3.5-flash-lite` and secret `GEMINI_API_KEY` in the selected AI environment in **both this repository and om-idea-board** (repository secrets also work). This test uses repository variable `AI_ENVIRONMENT=GEMINI_API_KEY` to select the dedicated environment where the replacement key was saved; without that selector, AI jobs use the target environment (`aws-test`). Environment names and secret names are separate settings. AWS build/publish jobs use `aws-test`. Gemini's native API uses a header key and JSON-schema output, with strict local validation and bounded requests. A provider-specific key takes precedence over legacy `AI_API_KEY`; remove obsolete generic keys when changing providers. Without a usable response, reports explicitly use `policy-fallback`; configuration alone does not prove live AI. Use a Free Tier project in [Google AI Studio](https://aistudio.google.com/api-keys) for free quota: billing-enabled projects may incur charges, and this test pins the model instead of following the changing `latest` alias. See [current pricing](https://ai.google.dev/gemini-api/docs/pricing) and the source GitOps guide before enabling billing. Exposed keys must be revoked and replaced in both repositories.

The monitor requires a public URL and matching frontend/backend release identities. New releases must pass readiness and a real create/read smoke test. On failure it commits previously verified application images/resources while preserving current Secret references and security controls. The first failed release has no automatic recovery target. A recovery commit must converge and pass health checks before being called successful. Database migrations/data and infrastructure are never rolled back automatically.

Secrets, Terraform state, credentials and certificates must never be committed here. Argo's project denies Secret and cluster-resource management. Namespace creation, Restricted Pod Security labels and database credential/CA Secrets are bootstrapped through an authenticated private API session.

For setup, private administration, provider choices and limitations see the [GitOps guide](https://github.com/surendarrajasekaran/om-idea-board/blob/main/docs/gitops.md). Changes to the health implementation should first be reviewed and tested in the application repository, then synchronized into `automation/`. Changing chart defaults requires a source pipeline release to regenerate manifests. Editing an archived snapshot invalidates its checksum and prevents recovery.

Initial target: `idea-board-aws-test`, namespace `idea-board-test`, AWS `mit-qa` / `us-west-2`. The test board is anonymous and uses HTTP; use synthetic data. Argo's UI remains private.

## Verified AWS test delivery — November 1, 2026

[Source release pipeline](https://github.com/surendarrajasekaran/om-idea-board/actions/runs/41275785592) and [new-release health verification](https://github.com/surendarrajasekaran/infra-helm-charts/actions/runs/41276535583) passed. GitOps release commit `0015f83` was reconciled by Argo CD; health commit `6408d3e` records its exact frontend/backend version and create/read checks. The health window had 20 requests, zero failures and p95 153.21 ms. That verification used explicit policy fallback. Gemini is now configured; live AI verification requires a successful run with the new secret. A prior deliberately incompatible image was recovered through Git, and verification data persisted.

The app runs in AWS mit-qa / us-west-2 with private EKS/RDS and immutable private ECR images. The Argo UI is private, and the unused ApplicationSet controller has zero replicas. See the source repository's verification record for image digests, security checks and remaining GCP/live-AI work.
