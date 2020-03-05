
## security
### tls
  see [tls part](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/tls.md)
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
    
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### node-node communication
* communication between nodes must be encrypted

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### roles
* identify the authorization modes configured on the cluster
  ```bash
  kubectl describe pod kube-apiserver-master -n kube-system
  #check --authorization-mode
  ```

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### role binding 
* maps roles to users
* check current role permissions
  ```
  kubectl auth can-i create deployment
  ```
* check permissions for a particular user
  ```
  kubectl auth can-i delete node --as dev-user --namespace default
  ```
* example of a role and binding that enables a user to create, list and delete pods
  ```
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    namespace: default
    name: developer
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "create"]
  ```

  ```
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: dev-user-binding
  subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: developer
    apiGroup: rbac.authorization.k8s.io
  ```
  
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### namespaced and not namespaced resources
* get list of namespaced/not namespaced resources run
  ```
  kubectl api-resources --namespaced=true
  ```

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### cluster roles
* identify the authorization modes configured on the cluster wide scope

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### cluster role binding 
* maps cluster roles to users
* example of a clusterrole and clusterrolebinding for access to nodes
  ```
  kind: ClusterRole
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: storage-admin
  rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "watch", "list", "create", "delete"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "watch", "list", "create", "delete"]
  ```
  ```
  kind: ClusterRoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: michelle-storage-admin
  subjects:
  - kind: User
    name: michelle
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: storage-admin
    apiGroup: rbac.authorization.k8s.io
  ```
  
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### image security
* create a secret object with the credentials required to access the registry
  ```
  kubectl create secret docker-registry private-reg-cred --docker-username=dock_user --docker-password=dock_password --docker-server=myprivateregistry.com:5000 --docker-email=dock_user@myprivateregistry.com
  ```
* example deployment file
  ```
  kubectl edit deploy web
  # add imagePullSecrets
  ```
  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    annotations:
      deployment.kubernetes.io/revision: "3"
      kubernetes.io/change-cause: kubectl set image deployments/web web=myprivateregistry.com:5000/nginx:alpine
        --record=true
    creationTimestamp: null
    generation: 1
    labels:
      run: web
    name: web
    selfLink: /apis/apps/v1/namespaces/default/deployments/web
  spec:
    progressDeadlineSeconds: 600
    replicas: 2
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        run: web
    strategy:
      rollingUpdate:
        maxSurge: 25%
        maxUnavailable: 25%
      type: RollingUpdate
    template:
      metadata:
        creationTimestamp: null
        labels:
          run: web
      spec:
        containers:
        - image: myprivateregistry.com:5000/nginx:alpine
          imagePullPolicy: IfNotPresent
          name: web
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        imagePullSecrets:
        - name: private-reg-cred
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
  status: {}  
  ```
  
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### security contexts
* get what is the user used to execute the sleep process within the 'ubuntu-sleeper' pod?
  ```
  kubectl exec ubuntu-sleeper whoami
  ```
* change user used to execute sleep process
  ```bash
  ...
  spec:
  containers:
  - command:
    - sleep
    - "4800"
    image: ubuntu
    imagePullPolicy: Always
    name: ubuntu
    resources: {}
    securityContext:
      runAsUser: 1010  <= THIS
      capabilities:
        add: ["SYS_TIME"] <= to be able to set time within the container
  ```
* user ID defined in the securityContext of the container overrides the User ID in the POD

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
### network policy
* ingress => traffic into a pod, egress => traffic to ouside of a pod
* policy example 
  ```
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    creationTimestamp: null
    generation: 1
    name: payroll-policy
    selfLink: /apis/networking.k8s.io/v1/namespaces/default/networkpolicies/payroll-policy
  spec:
    ingress:
    - from:
      - podSelector:
          matchLabels:
            name: internal
      ports:
      - port: 8080
        protocol: TCP
    podSelector:
      matchLabels:
        name: payroll
    policyTypes:
    - Ingress
  ```
* policy to allow traffic from the 'Internal' application only to the 'payroll-service' and 'db-service'
  ```
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: internal-policy
    namespace: default
  spec:
    podSelector:
      matchLabels:
        name: internal
    policyTypes:
    - Egress
    - Ingress
    ingress:
      - {}
    egress:
    - to:
      - podSelector:
          matchLabels:
            name: mysql
      ports:
      - protocol: TCP
        port: 3306

    - to:
      - podSelector:
          matchLabels:
            name: payroll
      ports:
      - protocol: TCP
        port: 8080
  ```

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/security.md#security)
