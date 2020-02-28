# kubernetes-knowledge

# cheatsheet

## kubeadm
* kubeadm init executes all the steps needed to bring up a cluster
  ```
  preflight                    Run pre-flight checks
  kubelet-start                Write kubelet settings and (re)start the kubelet
  certs                        Certificate generation
    /ca                          Generate the self-signed Kubernetes CA to provision identities for other Kubernetes components
    /apiserver                   Generate the certificate for serving the Kubernetes API
    /apiserver-kubelet-client    Generate the certificate for the API server to connect to kubelet
    /front-proxy-ca              Generate the self-signed CA to provision identities for front proxy
    /front-proxy-client          Generate the certificate for the front proxy client
    /etcd-ca                     Generate the self-signed CA to provision identities for etcd
    /etcd-server                 Generate the certificate for serving etcd
    /etcd-peer                   Generate the certificate for etcd nodes to communicate with each other
    /etcd-healthcheck-client     Generate the certificate for liveness probes to healthcheck etcd
    /apiserver-etcd-client       Generate the certificate the apiserver uses to access etcd
    /sa                          Generate a private key for signing service account tokens along with its public key
  kubeconfig                   Generate all kubeconfig files necessary to establish the control plane and the admin kubeconfig file
    /admin                       Generate a kubeconfig file for the admin to use and for kubeadm itself
    /kubelet                     Generate a kubeconfig file for the kubelet to use *only* for cluster bootstrapping purposes
    /controller-manager          Generate a kubeconfig file for the controller manager to use
    /scheduler                   Generate a kubeconfig file for the scheduler to use
  control-plane                Generate all static Pod manifest files necessary to establish the control plane
    /apiserver                   Generates the kube-apiserver static Pod manifest
    /controller-manager          Generates the kube-controller-manager static Pod manifest
    /scheduler                   Generates the kube-scheduler static Pod manifest
  etcd                         Generate static Pod manifest file for local etcd
    /local                       Generate the static Pod manifest file for a local, single-node local etcd instance
  upload-config                Upload the kubeadm and kubelet configuration to a ConfigMap
    /kubeadm                     Upload the kubeadm ClusterConfiguration to a ConfigMap
    /kubelet                     Upload the kubelet component config to a ConfigMap
  upload-certs                 Upload certificates to kubeadm-certs
  mark-control-plane           Mark a node as a control-plane
  bootstrap-token              Generates bootstrap tokens used to join a node to a cluster
  kubelet-finalize             Updates settings relevant to the kubelet after TLS bootstrap
    /experimental-cert-rotation  Enable kubelet client certificate rotation
  addon                        Install required addons for passing Conformance tests
    /coredns                     Install the CoreDNS addon to a Kubernetes cluster
    /kube-proxy                  Install the kube-proxy addon to a Kubernetes cluster
  ```
* kubeadm init phase allows to specify phases to be executed
   ```bash
   # run all sub-steps from control-plane step
   kubeadm init phase control-plane controller-manager 

   # run controller-manager sub-step from control-plane step
   kubeadm init phase control-plane controller-manager 
   ```
 * reset a cluster
   ```bash
   # kill all the resources
   kubeadm reset
   
   # clean ip table
   iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
   ```

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
  * shared volume for multiple containers in a pod (elastic-search)
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: app
      namespace: elastic-stack
      labels:
        name: app
    spec:
      containers:
      - name: app
        image: kodekloud/event-simulator
        volumeMounts:
        - mountPath: /log
          name: log-volume

      - name: sidecar
        image: kodekloud/filebeat-configured
        volumeMounts:
        - mountPath: /var/log/event-simulator/
          name: log-volume

      volumes:
      - name: log-volume
        hostPath:
          # directory location on host
          path: /var/log/webapp
          # this field is optional
          type: DirectoryOrCreate
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
### env variables
* types of env variables
  * key-value
  * from configMaps 
    * central place to store env variables for multiple pods
    * creation options:
      * imperative
        ```
        kubectl create configmap <config-name> --from-literal=<key>=<value> (of --from-file=app_config.properties)
        ```
      * declarative
        ```
        kubectl create -f app-config.yaml
        ```
        **app-config.yaml**
        ```
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: app-config-map
        data:
          APP_COLOR: blue
          APP_ENV: prod
        ```
        **use env variable value from configMap**
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
             envFrom:
             - configMapRef:
                 name: webapp-config-map
        ```        
  * from secrets
    * creation options:
      * imperative
        ```
        kubectl create secret generic <secret-name> --from-literal=<key>=<value> (of --from-file=app_secrets.properties)
        ```
      * declarative
        ```
        kubectl create -f app-secret.yaml
        echo -n 'password' | base64 => to encode the data
        echo -n 'SefE=ew' | base64 decode => to decode the data
        ```
        **app-secret.yaml**
        ```
        apiVersion: v1
        kind: Secret
        metadata:
          name: app-config-map
        data:
          DB_HOST: sSDsd=
          DB_PASSWORD: SefE=ew
        ```
        **use env variable value from secret**
        ```
         apiVersion: v1
         kind: Pod
         metadata:
           name: ubuntu-sleeper-2
         spec:
           containers:
           - name: ubuntu
             image: ubuntu
             envFrom:
             - secretRef:
                 name: webapp-secrets
        ```
        or 
        ```
        ...
        env: 
          - name: DB_PASSWORD
            valueFrom: 
              secretKeyRef: 
                name: webapp-secrets
                key: DB_PASSWORD
        ```
        or 
        ```
        ...
        volumes:
        - name: app-secret-volume:
            secret:
              secretName: webapp-secret
              
        ...
        cat /opt/app-secret-volumes/DB_PASSWORD => to view the secret in container
        ```
        * best pracitces is to encrypt secret data at rest => [encrypt-data](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
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
      env:
      - name: HOST_IP
        value: 172.19.234.73
  ```
## maintainance
#### drain/cordon
* empty the node node01 of all applications and mark it unschedulable (must be forced if node contains pods that are not part of replicaset)
  ```
  kubectl drain node01 --ignore-daemonsets
  ```
* set node to be unschedulable without killing the running apps
  ```
  kubectl cordon node01
  ```
* configure node to be schedulable again
  ```
  kubectl uncordon node01
  ```
#### upgrade kubernetes 
* what is the latest stable version available for upgrade?
  ```
  kubeadm upgrade plan
  ```
* recommended way to upgrade
  ```
  ugrade one minor version at a time until the latest stable version
  ```
* upgrade procedure
  * uprade kubeadm tool
    ```
    apt install kubeadm=1.17.0-00
    ```
  * upgrade components
    ```
    kubeadm upgrade apply v1.17.0
    ```
  * upgrade the kubelet
    ```
    apt install kubelet=1.17.0-00
    ```
#### backup & restore
* three things to consider for backup (velero can help to backup a cluster deployment)
  * resources
    * store on github    
  * etcd db
  * persistance
* etcd
  * for all the etcd commands
    * specify 
      * --endpoints
      * --cacert
      * --cert
      * --key
  * possible to create a snapshot of etcd db
    ```bash
    etcdctl snapshot save snapshot.db
    etcdctl snapshot status snapshot.db #get the status
    ```
    **example**
      ```bash
      ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
     snapshot save /tmp/snapshot-pre-boot.db
      ```
  * to restore from db file
    ```bash
    service kube-apiserver stop #depends on etcd
    etcdctl snapshot restore snapshot.db
    system daemon-reload
    service etcd start
    service kube-apiserver start
    ```
    **example**
      ```
      ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --name=master \
     --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
     --data-dir /var/lib/etcd-from-backup \
     --initial-cluster=master=https://127.0.0.1:2380 \
     --initial-cluster-token etcd-cluster-1 \
     --initial-advertise-peer-urls=https://127.0.0.1:2380 \
     snapshot restore /tmp/snapshot-pre-boot.db
      ```
## security
### authentication
* option 0 (not for production): plain text file
  * create a file containing user credentials
    ```
    password123,user1,u0001
    password123,user2,u0002
    password123,user3,u0003
    password123,user4,u0004
    password123,user5,u0005
    ```
  * modify /etc/kubernetes/manifests/kube-apiserver.yaml to include the file into pod
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: kube-apiserver
      namespace: kube-system
    spec:
      containers:
      - command:
        - kube-apiserver
          <content-hidden>
        image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
        name: kube-apiserver
        volumeMounts:
        - mountPath: /tmp/users
          name: usr-details
          readOnly: true
      volumes:
      - hostPath:
          path: /tmp/users
          type: DirectoryOrCreate
        name: usr-details
    ```
  * modify the kube-apiserver startup options to include the basic-auth file
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      name: kube-apiserver
      namespace: kube-system
    spec:
      containers:
      - command:
        - kube-apiserver
        - --authorization-mode=Node,RBAC
          <content-hidden>
        - --basic-auth-file=/tmp/users/user-details.csv
    ```
  * create the necessary roles and role bindings for these users
    ```
    ---
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      namespace: default
      name: pod-reader
    rules:
    - apiGroups: [""] # "" indicates the core API group
      resources: ["pods"]
      verbs: ["get", "watch", "list"]

    ---
    # This role binding allows "jane" to read pods in the "default" namespace.
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: read-pods
      namespace: default
    subjects:
    - kind: User
      name: user1 # Name is case sensitive
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role #this must be Role or ClusterRole
      name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
      apiGroup: rbac.authorization.k8s.io
    ```
  * afterwards the users can login like so
    ```
    curl -v -k https://localhost:6443/api/v1/pods -u "user1:password123"
    ```
