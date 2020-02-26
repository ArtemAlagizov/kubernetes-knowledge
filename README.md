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
    description: filter out which pods can be scheduled on a node, 
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
    description: choose which pods can be scheduled on a node, 
    usecases: different teams work on different nodes, and pods need to be separated  respectively, 
    BUT: does not guarantee that extra pods are not scheduled on a desired node, 
    solution => combine it with taints/toleration  
* set affinity of deployment to a label on a node:
  ```
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: red
  spec:
    replicas: 3
    selector:
      matchLabels:
        run: nginx
    template:
      metadata:
        labels:
          run: nginx
      spec:
        containers:
        - image: nginx
          imagePullPolicy: Always
          name: nginx
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: node-role.kubernetes.io/master
                  operator: Exists
  ```


### daemonset 
    description: run a pod accross all nodes, 
    usecases: log collection, monitoring
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
### static pods
    used to create and maintain kubernetes administration plane components
* get all the static pods:
  ```
  kubectl get pods --all-namespaces
  # all with -master appended in the name are static pods, for instance etcd-master
  ```
* create a static pod named static-busybox that uses the busybox image and the command sleep 1000
  ```
  kubectl run --restart=Never --image=busybox static-busybox --dry-run -o yaml --command -- sleep 1000 > /etc/kubernetes/manifests/static-busybox.yaml
  ```
* edit static pod
  ```
  edit a definition at /etc/kubernetes/manifests/.*.yaml
  ```
### multiple schedulers
    it is possible to use multiple schedulers to schedule pods
* get image used to deploy the kubernetes scheduler
  ```
  kubectl describe pod kube-scheduler-master --namespace=kube-system
  ```
* example of scheduler with one master (leader-elect=false)
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    annotations:
      scheduler.alpha.kubernetes.io/critical-pod: ""
    creationTimestamp: null
    labels:
      component: my-scheduler
      tier: control-plane
    name: my-scheduler
    namespace: kube-system
  spec:
    containers:
    - command:
      - kube-scheduler
      - --address=127.0.0.1
      - --kubeconfig=/etc/kubernetes/scheduler.conf
      - --leader-elect=false
      - --port=10282
      - --scheduler-name=my-scheduler
      - --secure-port=0
      image: k8s.gcr.io/kube-scheduler-amd64:v1.16.0
      imagePullPolicy: IfNotPresent
      livenessProbe:
        failureThreshold: 8
        httpGet:
          host: 127.0.0.1
          path: /healthz
          port: 10282
          scheme: HTTP
        initialDelaySeconds: 15
        timeoutSeconds: 15
      name: kube-scheduler
      resources:
        requests:
          cpu: 100m
      volumeMounts:
      - mountPath: /etc/kubernetes/scheduler.conf
        name: kubeconfig
        readOnly: true
    hostNetwork: true
    priorityClassName: system-cluster-critical
    volumes:
    - hostPath:
        path: /etc/kubernetes/scheduler.conf
        type: FileOrCreate
      name: kubeconfig
  status: {}  
  ```
* use custom scheduler
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
  spec:
    containers:
    -  image: nginx
       name: nginx
    schedulerName: my-scheduler
  ```

## monitoring and logging
* solutions
  * prometheus 
  * metrics-server
    * to get memory consumtion per node
      ```
      kubectl top node
      ```
* logging
  * to see logs from pod live with one container
    ```
    kubectl logs -f <pod-name>
    ```
  * to see logs from pod live with more than one container
    ```
    kubectl logs -f <pod-name> <container-name>
    ```
## lifecycle management
* two types of updates (StrategyType in deployment description)
  * recreate
    * kill all the deployments and recreate them, results in app downtime
  * rolling updates
    * kill and deploy new version gradually
    * RollingUpdateStrategy tells how many pods can be down at a time (RollingUpdateStrategy: 25% max unavailable, 25% max surge)
* edit deployment on the fly
  ```
  kubectl edit deployment <deployment-name>
  ```
* example deployment file
  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    annotations:
      deployment.kubernetes.io/revision: "4"
    creationTimestamp: "2020-02-26T07:15:11Z"
    generation: 4
    name: frontend
    namespace: default
    resourceVersion: "2513"
    selfLink: /apis/apps/v1/namespaces/default/deployments/frontend
    uid: 9e4a7177-9edc-4595-9161-3dd5a244cb02
  spec:
    minReadySeconds: 20
    progressDeadlineSeconds: 600
    replicas: 4
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        name: webapp
    strategy:
      rollingUpdate:
        maxSurge: 25%
        maxUnavailable: 25%
      type: RollingUpdate
    template:
      metadata:
        creationTimestamp: null
        labels:
          name: webapp
      spec:
        containers:
        - image: kodekloud/webapp-color:v2
          imagePullPolicy: IfNotPresent
          name: simple-webapp
          ports:
          - containerPort: 8080
            protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
  status:
    availableReplicas: 4
    conditions:
    - lastTransitionTime: "2020-02-26T07:15:59Z"
      lastUpdateTime: "2020-02-26T07:15:59Z"
      message: Deployment has minimum availability.
      reason: MinimumReplicasAvailable
      status: "True"
      type: Available
    - lastTransitionTime: "2020-02-26T07:15:11Z"
      lastUpdateTime: "2020-02-26T07:34:39Z"
      message: ReplicaSet "frontend-84bb97f469" has successfully progressed.
      reason: NewReplicaSetAvailable
      status: "True"
      type: Progressing
    observedGeneration: 4
    readyReplicas: 4
    replicas: 4
    updatedReplicas: 4
  ```
## commands and agruments
* in pod definition file
  * **command** corresponds to **entrypoint** in Dockerfile (run at startup of container)
  * **args** corresponds to **cmd** in Dockerfile default param passed to the entrypoint)
* example file
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: ubuntu-sleeper-2
  spec:
    containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep"]
      args: ["5000"]
  
  ```
