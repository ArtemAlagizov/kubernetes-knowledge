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
  
  # update the config file
  ...
  clusters:
  - cluster:
      server: https://172.17.0.14:6443
  ...
  
  # verify
  kubectl cluster-info --kubeconfig=/root/super.kubeconfig
  # should show correct info now
  ```
### troubleshoot deployment not scaling after scale command
* check control plane pods 
  ```
  kubectl get pods -n kube-system
  ```
* fix any issues with static control pods
  ```
  cd /etc/kubernetes/manifests
  ```
### upgrade the current version of kubernetes from 1.16 to 1.17.0 exactly using the kubeadm utility. make sure that the upgrade is carried out one node at a time starting with the master node. to minimize downtime, the deployment gold-nginx should be rescheduled on an alternate node before upgrading each node
* master node
  ```bash
  kubectl drain node master --ignore-daemonsets
  apt-get install kubeadm=1.17.0-00
  kubeadm  upgrade plan v1.17.0
  kubeadm  upgrade apply v1.17.0
  apt-get install kubelet=1.17.0-00
  
  # make master node schedulable
  kubectl uncordon master
  kubectl drain node01 --ignore-daemonsets
  ```
* node01
  ```
  apt-get install kubeadm=1.17.0-00
  kubeadm upgrade node --kubelet-version v1.17.0
  apt-get install kubelet=1.17.0-00
  ```
* master
  ```bash
  kubectl uncordon node01
  
  # make sure this is scheduled on master node
  kubectl get pods -o wide | grep gold
  ```
### print the names of all deployments in the admin2406 namespace in the following format: DEPLOYMENT CONTAINER_IMAGE READY_REPLICAS NAMESPACE \n \<deployment name\> \<container image used\> \<ready replica count\> \<Namespace\>. the data should be sorted by the increasing order of the deployment name
```
kubectl -n admin2406 get deployment -o custom-columns=DEPLOYMENT:.metadata.name,CONTAINER_IMAGE:.spec.template.spec.containers[].image,
READY_REPLICAS:.status.readyReplicas,NAMESPACE:.metadata.namespace --sort-by=.metadata.name > /opt/admin2406_data
```
