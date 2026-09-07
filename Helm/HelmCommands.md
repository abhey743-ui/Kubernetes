# Helm Commands Used in the Course

| Helm Command | Description |
|---|---|
| `helm create [NAME]` | Create a default Helm chart with the given name. |
| `helm dependencies build` | Recompile/build the dependencies for the given Helm chart. |
| `helm install [NAME] [CHART]` | Install the given Helm chart into a Kubernetes cluster. |
| `helm upgrade [NAME] [CHART]` | Upgrade a specified release to a new version of a chart. |
| `helm history [NAME]` | Display historical revisions for a given Helm release. |
| `helm rollback [NAME] [REVISION]` | Roll back a release to a previous revision. |
| `helm uninstall [NAME]` | Uninstall all resources associated with a given Helm release. |
| `helm template [NAME] [CHART]` | Render chart templates locally using the supplied values without installing them. |
| `helm list` | List all Helm releases inside a Kubernetes cluster. |
