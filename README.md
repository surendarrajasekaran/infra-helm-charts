# Idea Board GitOps

Helm charts, environment values and Argo CD application definitions for Idea Board. [App CI](https://github.com/surendarrajasekaran/om-idea-board#readme) builds/scans images, then publishes image digests and resource settings here. Argo reads Git and deploys the services.

[terraform-infrastructure](https://github.com/surendarrajasekaran/terraform-infrastructure#readme), a private repository, provisions the platform and installs the Argo controller. This repository owns application definitions and deployment values.

## Repository layout

| Path | Purpose |
| --- | --- |
| `helm-templates/nucleus/` | Shared workload chart, retained with provenance checksums |
| `helm-templates/argocd-app-controller/` | Generic chart rendering child Applications |
| `helm-overides/argocd-app-controller/custom-values.yaml` | Frontend/backend Application definitions and sources |
| `helm-overides/idea-fe/custom-values.yaml` | Frontend image, resources, Service and runtime settings |
| `helm-overides/idea-be/custom-values.yaml` | Backend image, resources, migration and Secret references |
| `argocd-manifests/projects.yaml` | Separate parent and workload permissions |
| `argocd-manifests/idea-board-controller.yaml` | Parent Application for one-time bootstrap |
| `argocd-manifests/idea-board-aws-test/controls.yaml` | Namespace quota and workload network policies |
| `releases/` | Desired release, verified-health history and immutable recovery snapshots |
| `.github/workflows/` | Helm validation and HTTP health/recovery |

The directory name `helm-overides` is intentional in the current repository; use that exact spelling.

## Argo ownership

```text
Terraform-managed Argo CD controller
└── idea-board-controller                 parent Application
    ├── idea-board-backend                nucleus + backend values + policies
    └── idea-board-frontend               nucleus + frontend values
```

The parent points to `helm-templates/argocd-app-controller` and loads `helm-overides/argocd-app-controller/custom-values.yaml`. Argo creates and maintains both children: three Application entries in total. Bootstrap the parent once; later definition/value changes reconcile from Git.

The parent project manages Application resources in `argocd`. The workload project permits approved resources in the app namespace, excluding Secrets and cluster resources. Operators prepare the namespace and credentials separately. Each child permits HPA to manage only its own Deployment `spec.replicas`; all other workload fields and HPA settings remain reconciled. The workload AppProject explicitly permits `autoscaling/HorizontalPodAutoscaler`. Application deletion does not cascade by default because deletion finalizers are not enabled.

## First-time setup

Use an authorized operator machine with Git, Terraform 1.11.4, AWS CLI v2, Helm 3.19 and `kubectl`. Clone this repository beside `terraform-infrastructure`. Complete its AWS apply, including the Terraform-managed Argo release, and select the intended Kubernetes context.

The supplied manifests describe namespace `idea-board-test` and delivery target `aws-test`. For another target or forks, update repository URLs, controller values, namespace/project names, workload settings and workflow source/target references together. Review database egress and registry access. Existing release history belongs to its original target; a new target needs separate history without copied endpoints or recovery candidates.

### 1. Prepare namespace and credentials

From this repository root:

```sh
kubectl create namespace idea-board-test --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace idea-board-test \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite
```

The backend references three existing Secrets. Create them through an authenticated operator session. Run this block in one shell; it reads initialized Terraform state into private temporary files without printing credentials. AWS uses its official RDS trust bundle; GCP instead exposes its database CA through Terraform.

```sh
(
  set -eu
  umask 077
  TF_ROOT="../terraform-infrastructure/apps/onboard/aws"
  SECRET_DIR="$(mktemp -d)"
  trap 'rm -rf "$SECRET_DIR"' EXIT

  terraform -chdir="$TF_ROOT" output -raw database_url > "$SECRET_DIR/runtime-url"
  terraform -chdir="$TF_ROOT" output -raw migration_database_url > "$SECRET_DIR/migration-url"
  terraform -chdir="$TF_ROOT" output -raw application_password > "$SECRET_DIR/app-password"
  printf '%s' ideas_app > "$SECRET_DIR/app-user"
  curl --fail --silent --show-error \
    https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem > "$SECRET_DIR/ca.crt"

  kubectl -n idea-board-test create secret generic idea-board-database \
    --from-file=DATABASE_URL="$SECRET_DIR/runtime-url" --dry-run=client -o yaml | kubectl apply -f -
  kubectl -n idea-board-test create secret generic idea-board-migration \
    --from-file=DATABASE_URL="$SECRET_DIR/migration-url" \
    --from-file=APP_DATABASE_USER="$SECRET_DIR/app-user" \
    --from-file=APP_DATABASE_PASSWORD="$SECRET_DIR/app-password" \
    --dry-run=client -o yaml | kubectl apply -f -
  kubectl -n idea-board-test create secret generic idea-board-database-ca \
    --from-file=ca.crt="$SECRET_DIR/ca.crt" --dry-run=client -o yaml | kubectl apply -f -
)
```

The migration init container creates the schema and restricted application role using its separate credentials. The running backend uses `ideas_app`. Secret rotation is an operator task; CI preserves the references.

### 2. Publish first images and values

Configure the app repository's `aws-test` environment, OIDC role and GitOps deploy key using the [Actions guide](https://github.com/surendarrajasekaran/om-idea-board#configure-github-actions). Configure Gemini in both repositories. If this repository is private, add read credentials for it to Argo before syncing the parent.

For a new target, initialize separate release history without a previous deployment URL. The first app release can publish images/values with `PUBLIC_URL` unset; health discovery skips records without a URL. Existing targets retain their verified history and endpoint. Run **AWS ECR and Argo CD GitOps release** in the app repository and confirm it publishes digests accessible to this cluster before creating the parent.

### 3. Bootstrap projects and parent

```sh
kubectl apply -f argocd-manifests/projects.yaml
kubectl apply -f argocd-manifests/idea-board-controller.yaml
kubectl -n argocd get applications
```

Argo installs both services through the parent. Do not separately Helm-install the app-controller or service charts. The backend Application also reconciles quota/network policies before its workload; the frontend uses the backend's internal Service.

### 4. Verify and enable health monitoring

```sh
kubectl -n argocd get applications
kubectl -n idea-board-test rollout status deployment/idea-board-backend --timeout=300s
kubectl -n idea-board-test rollout status deployment/idea-board-frontend --timeout=300s
kubectl -n idea-board-test get services
```

Confirm the parent and both children are **Synced / Healthy**. Obtain the frontend LoadBalancer address from its Service and test `/health/ready` and `/health/release`. Set the app environment's `PUBLIC_URL`, then run another release to record it in metadata and enable health discovery. Open the board, submit an idea and read it back.

The supplied frontend exposes HTTP; use synthetic demo data. Argo's UI is a ClusterIP Service accessed through approved operator connectivity. TLS, authentication and an organization-specific ingress setup require additional configuration before broader use.

## Normal releases and configuration changes

Merge app changes in `om-idea-board`. Its release workflow tests, asks Gemini for a valid plan, builds/scans immutable images and commits values/release metadata here. Argo reconciles services independently; health checks their identities together.

For deployment configuration changes, edit the relevant `helm-overides/` or app-controller values and open a PR. Images, release identities and AI resource requests are publisher-managed. Security, routing, Secret references and other settings remain in reviewed GitOps configuration.

## Health and recovery

[GitOps health and guarded recovery](.github/workflows/health.yml) runs on release metadata changes, a five-minute schedule and manual dispatch. Scheduled timing is best-effort. It loads health tools from a reviewed app-repository revision; advance the source pin when adopting changes to health/recovery helpers.

Configure `GEMINI_API_KEY`, `AI_PROVIDER`, `AI_MODEL` and optional `AI_ENVIRONMENT` using the app guide. The selected environment must exist. Allow the workflow's `GITHUB_TOKEN` to write contents under repository branch rules so it can record health/recovery. The runner has no cloud or Kubernetes credentials.

The monitor waits for matching frontend/backend identities, samples 20 readiness requests and checks create/read for new releases. Gemini receives numeric HTTP evidence. Local gates reject failed convergence, functional checks, excessive failures or latency even if AI says `keep`. Healthy unchanged releases skip AI calls. Summaries identify `live-ai`, `policy-fallback` or `steady-state-policy` and show the decision/evidence.

`release.json` records desired images, profile and URL; `status.json` records verified healthy releases. Checksummed snapshots support recovery. Recovery commits verified images/resources to the same values while preserving current security and Secret references. It does not restore database data, schema or infrastructure. Without a verified prior release, operator action is required.

This is HTTP release verification, not continuous pod/infrastructure monitoring. A successful publish alone does not prove runtime health. If another commit wins a race while health is saving results, inspect the latest release's health run before rerunning; do not force-push over newer state.

## Preview and validate

These commands lint/render locally without installing applications:

```sh
helm lint helm-templates/nucleus -f helm-overides/idea-fe/custom-values.yaml
helm lint helm-templates/nucleus -f helm-overides/idea-be/custom-values.yaml
helm lint helm-templates/argocd-app-controller \
  -f helm-overides/argocd-app-controller/custom-values.yaml
helm template idea-board-test-apps helm-templates/argocd-app-controller \
  --namespace argocd -f helm-overides/argocd-app-controller/custom-values.yaml
```

[Validate Helm configuration](.github/workflows/validate.yml) checks chart lint, parent/child definitions, nucleus provenance and rendered workload security. The upstream nucleus README documents chart capabilities; this guide describes its use through Argo.

## Availability and autoscaling

Both `helm-overides/idea-fe/custom-values.yaml` and `helm-overides/idea-be/custom-values.yaml` enable `autoscaling/v2` HPA through the unchanged nucleus chart:

| Setting | Frontend and backend |
| --- | --- |
| Minimum / maximum | 2 / 4 pods per service |
| CPU target | 70% of requested CPU, averaged across service pods |
| Scale up | 30-second stabilization, at most one additional pod per 60 seconds |
| Scale down | 300-second stabilization, at most one fewer pod per 60 seconds |
| Voluntary disruption protection | PDB keeps at least one pod available |

Memory is not a scaling metric here. At a 100m CPU request, the CPU target corresponds to approximately 70m average usage per pod. A lower AI CPU request can therefore cause earlier scaling; these settings need load testing. The AI release publisher can set the minimum to two or three, while preserving the maximum, target and placement rules.

Argo renders each Deployment without `spec.replicas`. Both child Applications have a narrowly scoped ignore rule for that field and `RespectIgnoreDifferences=true`, preventing synchronization from resetting HPA scale. Images, resources, HPA configuration and security settings remain managed by Git. When disabling HPA, also remove those ignore rules and set a fixed `workload.replicaCount` of at least two in the same change.

Pod anti-affinity prefers separate hosts (weight 100) and zones (weight 50). A hostname topology rule uses `maxSkew: 1` and `DoNotSchedule`; the zone rule prefers balance with `ScheduleAnyway`. The selectors match the actual Deployment labels. These rules spread new pods across available nodes; they do not guarantee nodes exist in separate zones or move existing pods automatically when another node joins. Hard anti-affinity would cap replicas at the number of nodes, so it is deliberately avoided. Two workers are the Terraform default; a single-node override removes node-failure protection. Node autoscaling is not installed, and HPA can report pending pods when current capacity is exhausted.

Terraform owns the EKS `metrics-server` add-on and the restricted control-plane-to-Metrics-Server security-group rule. GKE Autopilot provides resource metrics through its managed platform. The namespace quota accommodates both services at their four-pod cap plus one rolling-update surge pod per service. Rollback preserves current HPA bounds and scheduling rules even when recovering an older single-replica snapshot.

After provisioning and Argo synchronization, verify with an authorized cluster context:

```sh
kubectl get nodes -L topology.kubernetes.io/zone
kubectl -n kube-system rollout status deployment/metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl -n idea-board-test top pods
kubectl -n idea-board-test get hpa
kubectl -n idea-board-test describe hpa idea-board-frontend
kubectl -n idea-board-test describe hpa idea-board-backend
kubectl -n idea-board-test get pods -o wide
kubectl -n idea-board-test get pdb
```

Look for Metrics API `Available=True`, HPA `AbleToScale=True` / `ScalingActive=True`, numeric CPU targets instead of `<unknown>`, and service replicas on different hosts. Use a controlled load test in the test environment to observe scale-up and the five-minute scale-down window; frontend static traffic may need substantial concurrency to raise CPU. Check scheduling events if pods remain Pending. Verify a node drain only after confirming capacity and replacement readiness. These runtime checks require a running cluster and have not been executed against the deleted test stack.
