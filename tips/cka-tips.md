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
