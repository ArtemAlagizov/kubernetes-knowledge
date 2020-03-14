### create static pod on worker node
* check kubelet status
  ```
  sudo service kubelet status
  ```
* get config.yaml location
* check config.yaml for staticPodPath
* set it to /etc/kubernetes/manifests
### create a user  with permissions to update pods in a namespace
* create csr object
  * get base64 representaton of csr
    ```
    cat user.csr | base 64 | tr -d '\n' 
    ```
* create role
* create role-binding
### check dns resolution of pod and service
* create pod
  ```
  kubectl run --generator=run-pod/v1 nginx --image=nginx
  ```
* expose pod
  ```bash
  kubectl expose pod nginx --port=80 --name nginx-service --dry-run -o yaml > nginx.service.yaml
  
  # specify nodePort afterwards and create the service
  kubectl apply -f nginx.service.yaml
  ```
* run the following to check dns
  ```
  kubectl run --generator=run-pod/v1 nslookup --image:busybox:1.28 --rm -it -- nslookup <service-name> > nginx.svc
  kubectl run --generator=run-pod/v1 nslookup --image:busybox:1.28 --rm -it -- nslookup <dashed-pod-ip>.<namespace>.pod > nginx.pod
  ```
### create deployment, apply rolling update, record it
* create deployment
  ```
  kubectl create deployment --image=nginx
  ```
* apply roling update
  ```
  kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record
  ```
### create serviceaccount that has access to list all persistentvolumes
* create service account
  ```
  kubectl create serviceaacount pvviewer
  ```
* create cluster role to list persistent volumes
  ```
  kubectl create clusterrole pvviewer-role --resource=persistentvolumes --verb=list
  ```
* create cluster role binding
  ```
  kubectl create clusterrolebinding pvviewr-role-binding --clusterrole=pvviewer-role --serviceaccount=default:pvviwer
  ```
* specify service account in a pod
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
  spec:
    containers:
    - image: nginx
      name: nginx
      volumeMounts:
      - mountPath: /var/run/secrets/tokens
        name: vault-token
    serviceAccountName: build-robot
  ```
* verify
  ```bash
  kubectl describe pod nginx
  
  ...
  Volumes:
    pvviewer-token-6m8n9:
      Type:        Secret (a volume populated by a Secret)
      SecretName:  build-robot-token-6m8n9 # <= this indicates correct use of the service account
      Optional:    false
  ```
### list all internal ip addresses of nodes with node names
  ```
  kubectl get nodes -o jsonpath='{range.items[*]} {.status.addresses[?(@.type=="InternalIP")].address} of {.metadata.name}'
  ```
### allow incoming connections from all pods to a pod without modifying existing deployments
* create an ingress network policy
  ```
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: ingress-to-nptest
  spec:
    podSelector:
      matchLabels:
        run: np-test-1
    ingress:
    - from:
      - podSelector: {}
      ports:
      - protocol: TCP
        port: 80
  ```
* verify 
  ```bash
  kubectl run --generator=run-pod/v1 busybox --image=busybox --rm -it -- sh
  
  # inside the container
  wget --spider --timeout=1 np-test-service
  ```
### taint node with key:env_type, value:production and effect:NoSchedule, create pod woth tolerations to run on this node
* taint node
  ```
  kubectl taint node node01 env_type=production:NoSchedule
  ```
* create matching toleration for pod
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    creationTimestamp: null
    labels:
      run: prod-redis
    name: prod-redis
  spec:
    containers:
    - image: redis:alpine
      imagePullPolicy: IfNotPresent
      name: prod-redis
      resources: {}
    tolerations:
    - key: "env_type"
      operator: "Equal"
      value: "production"
      effect: "NoSchedule"
    dnsPolicy: ClusterFirst
    restartPolicy: Always
  status: {}
  ```
* verify
  ```
  kubectl get pod -o wide
  ```
### create pod with labels and in hr namespace
  ```bash
  # create namespace
  kubectl create namespace hr
  
  # create pod 
  kubectl run --generator=run-pod/v1 hr-pod --image=redis:alpine --labels=environment=production,tier=frontend --dry-run -o yaml > 7-hr-pod.yaml
  
  # add namespace to the pod metadata
  apiVersion: v1
  kind: Pod
  metadata:
    creationTimestamp: null
    labels:
      environment: production
      tier: frontend
    name: hr-pod
    namespace: hr
  spec:
    containers:
    - image: redis:alpine
      imagePullPolicy: IfNotPresent
      name: hr-pod
      resources: {}
    dnsPolicy: ClusterFirst
    restartPolicy: Always
  status: {}
  
  # deploy pod
  kubectl apply -f 7-hr-pod.yaml
  ```
### fix kubeconfig file
  ```bash
  # verify that host and port for kube-apiserver are correct
  kubectl cluster-info --kubeconfig=/root/super.kubeconfig
  ...
  clusters:
  - cluster:
      server: https://172.17.0.14:6443
  ...
  ```