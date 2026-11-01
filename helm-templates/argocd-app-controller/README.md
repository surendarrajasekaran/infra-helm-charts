# Argo CD application controller chart

This chart renders Argo CD `Application` resources from an `applications` map. Its structure follows the supplied app-controller reference; implementation and defaults are generic, without organization-specific repositories or cluster identities.

Each enabled application selects its project, destination namespace and source. Supported primary source types are `helm`, `directory` and `kustomize`. Helm sources accept `releaseName`, `helmValuesFiles` (or one `helmValuesFile`), `helmParameters` and `helmValues`. `additionalSources` adds related sources such as namespace controls, inheriting the primary repository/revision unless overridden. Empty directory/Kustomize maps are omitted to avoid normalization drift.

Global values provide repository/revision, project, destination server and sync policy defaults. Per-application sync policies override those defaults, including explicit `false` values. Optional `namePrefix`, cluster/environment labels, deletion finalizers and ignore rules are configurable. No replica-ignore rules are added automatically: replica plans remain reconciled.

For Idea Board, `helm-overides/argocd-app-controller/custom-values.yaml` enables exactly two Applications. This chart is installed as a small Helm bootstrap release in `argocd`; it creates no controller Pods and no parent Argo Application. Argo CD continues to manage the workloads. Application deletion leaves workloads running by default; `cascadeDeletion` explicitly opts into cascading deletion.

```sh
helm template idea-board-test-apps helm-templates/argocd-app-controller \
  --namespace argocd -f helm-overides/argocd-app-controller/custom-values.yaml
```

The scoped AppProject and namespace/credential bootstrap are administered separately. Changing application definitions requires upgrading this Helm bootstrap release. Image/resource changes remain in `helm-overides/idea-fe|idea-be/custom-values.yaml` and reconcile automatically.
