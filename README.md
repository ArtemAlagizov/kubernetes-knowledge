# kubernetes-knowledge

# cheatsheet

## scheduling

### creation of resources:
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
  kubectl run blue --image=nginx --replicas=6
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

### labels/selectors
* filter out pods based on labels
  ```
  kubectl get pods --selector env=dev
  ```
* filter out all resources based on labels
  ```
  kubectl get all --selector env=dev
  ```
* label a node 
  ```
  kubectl label node node01 color=blue
  ```
### annotations
* used for extra meta information like
  * integration info like version 
### taint/tolerations 
    filter out which pods can be scheduled on a node, 
    usecases: different teams work on different nodes, and pods need to be separated respectively, 
    BUT: does not guarantee that desired pod ends up on a desired node, 
    solution => combine it with node affinity

* add taint on master, NoSchedule
  ```
  kubectl taint nodes master node-role.kubernetes.io/master=all:NoSchedule
  kubectl taint nodes node01 spray=mortein:NoSchedule
  ```
* remove the taint on master, which currently has the taint effect of NoSchedule
  ```
  kubectl taint nodes master node-role.kubernetes.io/master:NoSchedule-
  ```
* **examples**
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: bee
  spec:
    containers:
    - image: nginx
      name: bee
    tolerations:
    - key: spray
      value: mortein
      effect: NoSchedule
      operator: Equal
  ```
### node affinity
    choose which pods can be scheduled on a node, 
    usecases: different teams work on different nodes, and pods need to be separated  respectively, 
    BUT: does not guarantee that extra pods are not scheduled on a desired node, 
    solution => combine it with taints/toleration  

### daemonset => run a pod accross all nodes, usecases: log collection, monitoring
* example of fluentD logger running on all nodes other than master:
  ```
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: fluentd-elasticsearch
    namespace: kube-system
    labels:
      k8s-app: fluentd-logging
  spec:
    selector:
      matchLabels:
        name: fluentd-elasticsearch
    template:
      metadata:
        labels:
          name: fluentd-elasticsearch
      spec:
        tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        containers:
        - name: fluentd-elasticsearch
          image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
          - name: varlog
            mountPath: /var/log
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
            readOnly: true
        terminationGracePeriodSeconds: 30
        volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
  ```
