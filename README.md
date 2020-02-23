# kubernetes-knowledge

# cheatsheet

## creation of resources:
* create an NGINX Pod: 
  ```
  kubectl run --generator=run-pod/v1 nginx --image=nginx
  ```
* generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)
  ```
  kubectl run --generator=run-pod/v1 nginx --image=nginx --dry-run -o yaml
  ```
* create a deployment
  ```
  kubectl create deployment --image=nginx nginx
  ```
* scale deployment
  ```
  kubectl scale --replicas=3 deployment webapp
  ```
* create a deployment file
  ```
  kubectl create deployment --image=nginx nginx --dry-run -o yaml
  ```
  * _NOTE_: kubectl create deployment does not have a --replicas option. You could first create it and then scale it using the kubectl scale command
* save it to a file - (If you need to modify or add some other details)
  ```
  kubectl create deployment --image=nginx nginx --dry-run -o yaml > nginx-deployment.yaml
  ```
* create a Service named redis-service of type ClusterIP to expose pod redis on port 6379
  ```
  kubectl expose pod redis --port=6379 --name redis-service --dry-run -o yaml
  ```
  * _NOTE_: This will automatically use the pod's labels as selectors
* or
  ```
  kubectl create service clusterip redis --tcp=6379:6379 --dry-run -o yaml
  ```
  * _NOTE_: This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service
* create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes
  ```
  kubectl expose pod nginx --port=80 --name nginx-service --dry-run -o yaml
  ```
  * _NOTE_: This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.
* or
  ```
  kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run -o yaml
  ```
  * _NOTE_: This will not use the pods labels as selectors
