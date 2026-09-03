Kubernetes Commands used in the course
Kubernetes Command	Description
"kubectl apply -f filename"	To create a deployment/service/configmap based on a given YAML file
"kubectl get all"	To get all the components inside your cluster
"kubectl get pods"	To get all the pods details inside your cluster
"kubectl get pod pod-id"	To get the details of a given pod id
"kubectl describe pod pod-id"	To get more details of a given pod id
"kubectl delete pod pod-id"	To delete a given pod from cluster
"kubectl get services"	To get all the services details inside your cluster
"kubectl get service service-id"	To get the details of a given service id
"kubectl describe service service-id"	To get more details of a given service id
"kubectl get nodes"	To get all the node details inside your cluster
"kubectl get node node-id"	To get the details of a given node
"kubectl get replicasets"	To get all the replica sets details inside your cluster
"kubectl get replicaset replicaset-id"	To get the details of a given replicaset
"kubectl get deployments"	To get all the deployments details inside your cluster
"kubectl get deployment deployment-id"	To get the details of a given deployment
"kubectl get configmaps"	To get all the configmap details inside your cluster
"kubectl get configmap configmap-id"	To get the details of a given configmap
"kubectl get events --sort-by=.metadata.creationTimestamp"	To get all the events occured inside your cluster
"kubectl scale deployment accounts-deployment --replicas=1"	To set the number of replicas for a deployment inside your cluster
"kubectl set image deployment gatewayserver-deployment gatewayserver=eazybytes/gatewayserver:s11 --record"	To set a new image for a deployment inside your cluster
"kubectl rollout history deployment gatewayserver-deployment"	To know the rollout history for a deployment inside your cluster
"kubectl rollout undo deployment gatewayserver-deployment --to-revision=1"	To rollback to a given revision for a deployment inside your cluster
"kubectl get pvc"	To list the pvcs inside your cluster
"kubectl delete pvc data-happy-panda-mariadb-0"	To delete a pvc inside your cluster
