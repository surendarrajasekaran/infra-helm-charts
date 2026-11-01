# Idea Board GitOps

Platform provisioning, private access and the Argo Helm bootstrap live in [terraform-infrastructure](https://github.com/surendarrajasekaran/terraform-infrastructure). Argo CD owns application deployment. Application CI builds and scans images, then updates the frontend/backend Helm values here.

```text
argocd-manifests/       Parent Application, scoped projects and network policies
helm-templates/        nucleus and the generic argocd-app-controller chart
helm-overides/         Frontend, backend and app-controller custom-values.yaml
releases/              Current release/status and immutable rollback snapshots
```

## Argo ownership

`argocd-manifests/idea-board-controller.yaml` is the one-time bootstrap Application. It points to `helm-templates/argocd-app-controller`, using `helm-overides/argocd-app-controller/custom-values.yaml`. Argo then creates and maintains two child Applications:

- `idea-board-backend`: nucleus with backend values, plus namespace policies from `argocd-manifests/idea-board-aws-test/controls.yaml`.
- `idea-board-frontend`: nucleus with frontend values.

There are three Applications in Argo: one parent and two services. The controller is installed as Helm release `argocd`; there is no separate Helm installation for the app-controller chart or either service. Commit application-definition changes to Git and the parent reconciles them. The two services reconcile independently; the health workflow checks both release identities together.

The parent's project permits only Application resources in `argocd`. The workload project permits approved namespaced resources in `idea-board-test`, excluding Secrets and cluster resources. Namespace creation, credentials and database CA stay in authenticated private bootstrap. Application deletion has no cascading finalizer by default. Replicas are reconciled.

## Release and AI health

`releases/idea-board-aws-test/release.json` records the desired image digests, resource plan and URL. `status.json` records verified healthy releases and pending rollback. Subdirectories contain immutable checksummed snapshots. These records are needed for guarded AI recovery; they are not Argo deployment manifests. Current rendered frontend/backend duplicates are removed.

`.github/workflows/health.yml` checks out a pinned, reviewed revision of health/AI/recovery scripts from [om-idea-board](https://github.com/surendarrajasekaran/om-idea-board). There is no copied `automation` directory. Updating those scripts requires reviewing and advancing that workflow's source revision. The runner has no AWS or Kubernetes credentials.

The workflow checks exact frontend/backend identity, readiness and new-release create/read, then prints the Gemini decision and evidence in Actions. Hard failures force Git recovery when a verified prior release exists. Recovery changes the same Helm values while preserving current security and Secret references; it never rolls back database data, schema or infrastructure. Unchanged healthy releases skip AI calls. It observes HTTP only.

Gemini configuration remains `AI_ENVIRONMENT=GEMINI_API_KEY`, `AI_PROVIDER=gemini`, `AI_MODEL=gemini-3.5-flash-lite`, with the key stored in Actions secrets in the selected environment. Billing depends on the project's actual tier. See the [delivery guide](https://github.com/surendarrajasekaran/om-idea-board#deploy-a-change) for bootstrap, AI setup and demo steps, and [verification](https://github.com/surendarrajasekaran/om-idea-board#current-deployment) for observed results. The anonymous AWS test board uses HTTP and synthetic data; the Argo UI stays private.
