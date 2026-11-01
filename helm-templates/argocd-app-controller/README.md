# Argo CD application controller chart

This chart renders Argo CD `Application` resources from an `applications` map. Its structure follows the supplied app-controller reference; implementation and defaults are generic, without organization-specific repositories or cluster identities.

Each enabled application selects its project, destination namespace and source. Supported primary source types are `helm`, `directory` and `kustomize`. Helm sources accept `releaseName`, `helmValuesFiles` (or one `helmValuesFile`), `helmParameters` and `helmValues`. `additionalSources` adds related sources such as namespace controls, inheriting the primary repository/revision unless overridden. Empty directory/Kustomize maps are omitted to avoid normalization drift.

Global values provide repository/revision, project, destination server and sync policy defaults. Per-application sync policies override those defaults, including explicit `false` values. Optional `namePrefix`, cluster/environment labels, deletion finalizers and ignore rules are configurable. No replica-ignore rules are added automatically: replica plans remain reconciled.

For Idea Board, the parent Application in `argocd-manifests/idea-board-controller.yaml` points to this chart and loads `helm-overides/argocd-app-controller/custom-values.yaml`. Argo creates and maintains the two service Applications. Do not install this chart separately with Helm.

```sh
helm template idea-board-test-apps helm-templates/argocd-app-controller \
  --namespace argocd -f helm-overides/argocd-app-controller/custom-values.yaml
```

This command only previews the Application definitions. Follow the [repository setup guide](../../README.md#first-time-setup) to prepare credentials and bootstrap the scoped projects and parent through an authorized Kubernetes API connection. Later definition/value changes are committed to Git and reconciled by Argo. The parent and children have no cascading-deletion finalizers by default; explicit deletion/retirement requires operator care.
