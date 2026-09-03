# Kubernetes Commands Cheat Sheet

| Kubernetes Command                                                                                         | Description                                                                   |
| ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `kubectl apply -f filename`                                                                                | To create a deployment, service, ConfigMap, etc. based on a given YAML file.  |
| `kubectl get all`                                                                                          | To get all the components inside your cluster.                                |
| `kubectl get pods`                                                                                         | To get details of all the pods inside your cluster.                           |
| `kubectl get pod pod-id`                                                                                   | To get the details of a given pod.                                            |
| `kubectl describe pod pod-id`                                                                              | To get more detailed information about a given pod.                           |
| `kubectl delete pod pod-id`                                                                                | To delete a given pod from the cluster.                                       |
| `kubectl get services`                                                                                     | To get details of all the services inside your cluster.                       |
| `kubectl get service service-id`                                                                           | To get the details of a given service.                                        |
| `kubectl describe service service-id`                                                                      | To get more detailed information about a given service.                       |
| `kubectl get nodes`                                                                                        | To get details of all the nodes inside your cluster.                          |
| `kubectl get node node-id`                                                                                 | To get the details of a given node.                                           |
| `kubectl get replicasets`                                                                                  | To get details of all ReplicaSets inside your cluster.                        |
| `kubectl get replicaset replicaset-id`                                                                     | To get the details of a given ReplicaSet.                                     |
| `kubectl get deployments`                                                                                  | To get details of all the deployments inside your cluster.                    |
| `kubectl get deployment deployment-id`                                                                     | To get the details of a given deployment.                                     |
| `kubectl get configmaps`                                                                                   | To get details of all the ConfigMaps inside your cluster.                     |
| `kubectl get configmap configmap-id`                                                                       | To get the details of a given ConfigMap.                                      |
| `kubectl get events --sort-by=.metadata.creationTimestamp`                                                 | To get all events that occurred inside your cluster, sorted by creation time. |
| `kubectl scale deployment accounts-deployment --replicas=1`                                                | To set the number of replicas for a deployment.                               |
| `kubectl set image deployment gatewayserver-deployment gatewayserver=eazybytes/gatewayserver:s11 --record` | To set a new image for a deployment.                                          |
| `kubectl rollout history deployment gatewayserver-deployment`                                              | To view the rollout history for a deployment.                                 |
| `kubectl rollout undo deployment gatewayserver-deployment --to-revision=1`                                 | To roll back a deployment to a given revision.                                |
| `kubectl get pvc`                                                                                          | To list the PersistentVolumeClaims (PVCs) inside your cluster.                |
| `kubectl delete pvc data-happy-panda-mariadb-0`                                                            | To delete a PVC inside your cluster.                                          |
