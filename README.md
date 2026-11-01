# Idea Board GitOps state

Argo CD reads desired application manifests from this repository. [om-idea-board](https://github.com/surendarrajasekaran/om-idea-board) owns application code, Terraform and build/publish automation.

| Path | Purpose |
| --- | --- |
| `bootstrap/idea-board-test.yaml` | Scoped Argo AppProject and Application |
| `helm-common/nucleus` | Canonical nucleus 1.1.0 chart and provenance |
| `helm-overides/idea-fe/custom-values.yaml` | Frontend image, resources and configuration |
| `helm-overides/idea-be/custom-values.yaml` | Backend image, resources and configuration |
| `environments/<target>/manifests` | Namespace controls plus rendered evidence/recovery inputs |
| `environments/<target>/release.json` | Source commit, immutable image digests, resource profile and public URL |
| `environments/<target>/status.json` | Verified healthy release history and pending recovery guard |
| `releases/<target>/<id>` | Immutable release snapshots and checksums used for recovery |
| `automation/` | Reviewed copies of source health, AI and recovery helpers |
| `.github/workflows/health.yml` | New-release/scheduled HTTP assessment and guarded Git recovery |

## Chart ownership

The chart and both component values files live here, using the exact folder layout above. The application pipeline updates each image tag/digest, resource profile and release identity in the current values, previews the exact Helm render, and commits the values plus checksummed recovery snapshots. The application repository has a pinned GitOps reference for CI checks and no duplicate chart.

Argo CD renders `helm-common/nucleus` twice: once with the backend values and Helm release name `idea-board-backend`, then with frontend values and release name `idea-board-frontend`. They produce separate Deployments and Services. A third source includes only `controls.yaml` for namespace security; rendered frontend/backend files are retained for validation/recovery and excluded from that directory source. The same Argo application coordinates controls → backend → frontend with sync waves.

Argo renders Helm and reconciles resources from inside private EKS. The health workflow needs no AWS or Kubernetes credentials. Its GitHub token can update this repository; fast-forward pushes reject stale concurrent observations.

## Gemini and recovery

The selected provider/model is `gemini` / `gemini-3.5-flash-lite`. This repository and the source repository set repository variable `AI_ENVIRONMENT=GEMINI_API_KEY`, selecting the dedicated environment containing secret `GEMINI_API_KEY` and provider/model variables. Without that selector, AI jobs use the target environment (`aws-test`). Environment names and secret names are distinct settings. AWS build/publish jobs use `aws-test` separately.

The health step prints its validated decision and evidence and adds an **AI health assessment** table to the Actions run summary. Unchanged healthy releases explicitly show **AI call skipped**. Gemini uses header authentication and JSON-schema output with strict local validation. Failed requests or rejected proposals explicitly use `policy-fallback`; required-live planning fails instead. Use the source repository's manual **Check live AI provider** workflow to test a replacement key/model. A Free Tier project in [Google AI Studio](https://aistudio.google.com/api-keys) is required for free quota; billing-enabled usage may be charged. See the [GitOps guide](https://github.com/surendarrajasekaran/om-idea-board/blob/main/docs/gitops.md) for credential placement and billing conditions.

The monitor requires matching frontend/backend release identities, readiness and a new-release create/read smoke test. It has HTTP-only telemetry and skips model calls for an unchanged healthy release. Failed hard gates force recovery even when AI recommends keeping the release. Recovery updates these same Helm values with previous verified images/resources, re-renders the current chart and commits a new snapshot while preserving current Secret references and security controls. Pre-migration manifest-only snapshots remain usable. The first failed installation has no recovery target; a pending recovery must converge and pass health checks before being called successful. Database migrations, data and infrastructure are never rolled back automatically.

Release snapshots and healthy history are recovery inputs, not disposable generated files. Editing an archived snapshot invalidates its checksum. Namespace creation, Restricted Pod Security and database credential/CA Secrets are bootstrapped through the private API; Argo's project denies Secret and cluster-resource management. Secrets, state, credentials and certificates do not belong in Git.

Monitor code is reviewed/tested in source `scripts/`, then synchronized into `automation/` so this workflow can run independently. Chart/value changes here run the Helm validation workflow. Application CI publishes value updates; Argo consumes the chart and values directly.

## Verified baseline

[Required-live Gemini release](https://github.com/surendarrajasekaran/om-idea-board/actions/runs/41282749169) and [independent deployed health](https://github.com/surendarrajasekaran/infra-helm-charts/actions/runs/41282887901) passed with `source=live-ai`: exact component convergence, create/read success, 20 requests, zero failures and p95 110.77 ms. Gemini recommended `keep`. A previous Git recovery drill used deterministic policy and preserved verification data. See [source verification](https://github.com/surendarrajasekaran/om-idea-board/blob/main/docs/verification.md) for evidence, image digests and later releases.

Target: `idea-board-aws-test`, namespace `idea-board-test`, AWS `mit-qa` / `us-west-2`. Current release/history are authoritative in [release.json](environments/idea-board-aws-test/release.json) and [status.json](environments/idea-board-aws-test/status.json). The public test board is anonymous and uses HTTP; use synthetic data. The Argo UI remains private.
