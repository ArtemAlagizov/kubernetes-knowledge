## spin up 5 node cluster locally [based on [this](https://github.com/mmumshad/kubernetes-the-hard-way)]
### setup
2 masters, 2 workers, 1 loadbalancer in front of the masters

### prerequisites
  * virtual box installed
  * vagrant installed

### steps
#### provision VMs
  ```bash
  cd spin-up-5-node-cluster/vagrant
  vargant up
  ```
#### install tools to perform administrative tasks, such as creating certificates, kubeconfig files and distributing them to the different VMs
* as an example it is done on master-1
* generate ssh key pair 
  ```bash
  ssh-keygen
  ```
* distribute it to other nodes to easily ssh/scp to other nodes
  ```
  vargant@master-1:~$ ssh master-2
  cat >> ~/.ssh/authorized_keys <<EOF
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3DTnFRfC8D7jWwg95ppVYKeqQgYRinjLPDfp5egx3/GinXe7CJtJz3vpNEnta3xPlc27aJiNWtkrjfj6CUUL4NcZYRe6erTHKeZ+nKBNLTa80AmYXtUkwpTAcnaTeh7zX9vemTK/UxpfqcLsILmIyJ6D72J38ZUjeMkhxpxLjQBD4yDkzGl0sbZDiHAnudHrp5Sk gU2y6VT+vfuZBS9aHyzmPN1SsptK2rbdob+hwqe2UqLzKBQvuLjXWCaO4mknEzt3CF5GA2YTO6DogZSL3+rLEZw3OV428DhE4kOwiTHPDG5b2ccF4rE7q4TcLPvkxkJU2/n03+Q3zRbLdIB6b vagrant@master-1
  EOF
  ```
* install kubectl on master-1
  ```bash
  wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubectl
  chmod +x kubectl
  sudo mv kubectl /usr/local/bin/
  
  # verify installation
  kubectl version --client
  ```
#### provision a Certificate Authority that can be used to sign additional tls certificates
* on master-1
  * to be able to generate a Certificate Signing Request remove the following line at /etc/ssl/openssl.conf
    ```
    RANDFILE                = $ENV::HOME/.rnd
    ```
  * create a CA certificate, then generate a Certificate Signing Request and use it to create a private key
    ```
    openssl genrsa -out ca.key 2048
    openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
    openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
    ```
  * generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes admin user
    * generate the admin client certificate and private key
      ```bash
      # generate private key for admin user
      openssl genrsa -out admin.key 2048

      # generate CSR for admin user
      openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr

      # sign certificate for admin user using CA servers private key
      openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out admin.crt -days 1000
      ```
    * generate the kube-controller-manager client certificate and private key  
      ```
      openssl genrsa -out kube-controller-manager.key 2048
      openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
      openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000
      ```
    * generate the kube-proxy client certificate and private key
      ```
      openssl genrsa -out kube-proxy.key 2048
      openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy" -out kube-proxy.csr
      openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-proxy.crt -days 1000
      ```
    * generate the kube-scheduler client certificate and private key
      ```
      openssl genrsa -out kube-scheduler.key 2048
      openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
      openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-scheduler.crt -days 1000
      ```
    * create a conf file for kube-apiserver certificate (file is needed to specify dns names/ips for resolution of kubernetes components)
      ```
      cat > openssl.cnf <<EOF
      [req]
      req_extensions = v3_req
      distinguished_name = req_distinguished_name
      [req_distinguished_name]
      [ v3_req ]
      basicConstraints = CA:FALSE
      keyUsage = nonRepudiation, digitalSignature, keyEncipherment
      subjectAltName = @alt_names
      [alt_names]
      DNS.1 = kubernetes
      DNS.2 = kubernetes.default
      DNS.3 = kubernetes.default.svc
      DNS.4 = kubernetes.default.svc.cluster.local
      IP.1 = 10.96.0.1
      IP.2 = 192.168.5.11
      IP.3 = 192.168.5.12
      IP.4 = 192.168.5.30
      IP.5 = 127.0.0.1
      EOF
      ```
    * generates certs for kube-apiserver
      ```
      openssl genrsa -out kube-apiserver.key 2048
      openssl req -new -key kube-apiserver.key -subj "/CN=kube-apiserver" -out kube-apiserver.csr -config openssl.cnf
      openssl x509 -req -in kube-apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 1000
      ```
    * create a conf file for etcd certificate (file is needed to specify dns names/ips for resolution of kubernetes components)
      ```
      cat > openssl-etcd.cnf <<EOF
      [req]
      req_extensions = v3_req
      distinguished_name = req_distinguished_name
      [req_distinguished_name]
      [ v3_req ]
      basicConstraints = CA:FALSE
      keyUsage = nonRepudiation, digitalSignature, keyEncipherment
      subjectAltName = @alt_names
      [alt_names]
      IP.1 = 192.168.5.11
      IP.2 = 192.168.5.12
      IP.3 = 127.0.0.1
      EOF
      ```
    * generates certs for etcd
      ```
      openssl genrsa -out etcd-server.key 2048
      openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -out etcd-server.csr -config openssl-etcd.cnf
      openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 1000
      ```
    * generate the service-account certificate and private key
      ```
      openssl genrsa -out service-account.key 2048
      openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
      openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 1000
      ```
    * copy the appropriate certificates and private keys to each controller instance (master-2 in this case)
      ```
      for instance in master-2; do
        scp ca.crt ca.key kube-apiserver.key kube-apiserver.crt \
          service-account.key service-account.crt \
          etcd-server.key etcd-server.crt \
          ${instance}:~/
      done
      ```
#### generate kubeconfigs, which enable kubernetes clients to locate and authenticate to the kubernetes api servers
* each kubeconfig will use ip address of the load balancer as kubernetes api server 
  ```
  LOADBALANCER_ADDRESS=192.168.5.30
  ```
* generate a kubeconfig file for the kube-proxy service
  ```
  {
    kubectl config set-cluster kubernetes-the-hard-way \
      --certificate-authority=ca.crt \
      --embed-certs=true \
      --server=https://${LOADBALANCER_ADDRESS}:6443 \
      --kubeconfig=kube-proxy.kubeconfig

    kubectl config set-credentials system:kube-proxy \
      --client-certificate=kube-proxy.crt \
      --client-key=kube-proxy.key \
      --embed-certs=true \
      --kubeconfig=kube-proxy.kubeconfig

    kubectl config set-context default \
      --cluster=kubernetes-the-hard-way \
      --user=system:kube-proxy \
      --kubeconfig=kube-proxy.kubeconfig

    kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
  }
  ```
* generate a kubeconfig file for the kube-controller-manager service
  ```
  {
    kubectl config set-cluster kubernetes-the-hard-way \
      --certificate-authority=ca.crt \
      --embed-certs=true \
      --server=https://${LOADBALANCER_ADDRESS}:6443 \
      --kubeconfig=kube-controller-manager.kubeconfig

    kubectl config set-credentials system:kube-controller-manager \
      --client-certificate=kube-controller-manager.crt \
      --client-key=kube-controller-manager.key \
      --embed-certs=true \
      --kubeconfig=kube-controller-manager.kubeconfig

    kubectl config set-context default \
      --cluster=kubernetes-the-hard-way \
      --user=system:kube-controller-manager \
      --kubeconfig=kube-controller-manager.kubeconfig

    kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
  }
  ```
* generate a kubeconfig file for the kube-scheduler service
  ```
  {
    kubectl config set-cluster kubernetes-the-hard-way \
      --certificate-authority=ca.crt \
      --embed-certs=true \
      --server=https://${LOADBALANCER_ADDRESS}:6443 \
      --kubeconfig=kube-scheduler.kubeconfig

    kubectl config set-credentials system:kube-scheduler \
      --client-certificate=kube-scheduler.crt \
      --client-key=kube-scheduler.key \
      --embed-certs=true \
      --kubeconfig=kube-scheduler.kubeconfig

    kubectl config set-context default \
      --cluster=kubernetes-the-hard-way \
      --user=system:kube-scheduler \
      --kubeconfig=kube-scheduler.kubeconfig

    kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
  }
  ```
* generate a kubeconfig file for the admin user
  ```
  {
    kubectl config set-cluster kubernetes-the-hard-way \
      --certificate-authority=ca.crt \
      --embed-certs=true \
      --server=https://${LOADBALANCER_ADDRESS}:6443 \
      --kubeconfig=admin.kubeconfig

    kubectl config set-credentials admin \
      --client-certificate=admin.crt \
      --client-key=admin.key \
      --embed-certs=true \
      --kubeconfig=admin.kubeconfig

    kubectl config set-context default \
      --cluster=kubernetes-the-hard-way \
      --user=admin \
      --kubeconfig=admin.kubeconfig

    kubectl config use-context default --kubeconfig=admin.kubeconfig
  }
  ```
* copy the appropriate kube-proxy kubeconfig files to each worker instance
  ```
  for instance in worker-1 worker-2; do
    scp kube-proxy.kubeconfig ${instance}:~/
  done
  ```
* copy the appropriate kube-controller-manager and kube-scheduler kubeconfig files to each controller instance
  ```
  for instance in master-1 master-2; do
    scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
  done
  ```
### data encryption
* to store data in etcd in an encrypted way:
  * generate an encryption key
    ```
    ENCRYPTION_KEY= $(head -c 32 /dev/urandom | base64)
    ```
  * create encryption-config.yaml
    ```
    kind: EncryptionConfig
    apiVersion: v1
    resources:
      - resources:
          - secrets
        providers:
          - aescbc:
              keys:
                - name: key1
                  secret: ${ENCRYPTION_KEY}
          - identity: {}
    ```
  * distribute encryption-config.yaml file to master nodes
    ```
    for instance in master-1 master-2; do
      scp encryption-config.yaml ${instance}:~/
    done  
    ```
### bootstrap etcd cluster
* download binaries 
  ```
  wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
  ```
* install etcd-server and etcdctl
  ```
  {
    tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
    sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
  }
  ```
* configure the etcd server
  * copy certificates
    ```
    {
      sudo mkdir -p /etc/etcd /var/lib/etcd
      sudo cp ca.crt etcd-server.key etcd-server.crt /etc/etcd/
    }
    ```
  * the instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. retrieve the internal ip address of the master(etcd) nodes
    ```
    INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
    ```
  * each etcd member must have a unique name within an etcd cluster. set the etcd name to match the hostname of the current compute instance
    ```
    ETCD_NAME=$(hostname -s)
    ```
  * create the etcd.service systemd unit file
    ```
    cat <<EOF | sudo tee /etc/systemd/system/etcd.service
    [Unit]
    Description=etcd
    Documentation=https://github.com/coreos

    [Service]
    ExecStart=/usr/local/bin/etcd \\
      --name ${ETCD_NAME} \\
      --cert-file=/etc/etcd/etcd-server.crt \\
      --key-file=/etc/etcd/etcd-server.key \\
      --peer-cert-file=/etc/etcd/etcd-server.crt \\
      --peer-key-file=/etc/etcd/etcd-server.key \\
      --trusted-ca-file=/etc/etcd/ca.crt \\
      --peer-trusted-ca-file=/etc/etcd/ca.crt \\
      --peer-client-cert-auth \\
      --client-cert-auth \\
      --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
      --listen-peer-urls https://${INTERNAL_IP}:2380 \\
      --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
      --advertise-client-urls https://${INTERNAL_IP}:2379 \\
      --initial-cluster-token etcd-cluster-0 \\
      --initial-cluster master-1=https://192.168.5.11:2380,master-2=https://192.168.5.12:2380 \\
      --initial-cluster-state new \\
      --data-dir=/var/lib/etcd
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
* start etcd server on each master
  ```
  {
    sudo systemctl daemon-reload
    sudo systemctl enable etcd
    sudo systemctl start etcd
  }
  ```
* verify that it is running
  ```
  sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
  ```
### bootstrap control plane
* create folder for config files
  ```
  sudo mkdir -p /etc/kubernetes/config
  ```
* download binaries
  ```
  wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubectl"
  ```
* install the binaries
  ```
  {
    chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
    sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
  }
  ```
* configure kube api server
  * copy certificates
    ```
    {
      sudo mkdir -p /var/lib/kubernetes/

      sudo cp ca.crt ca.key kube-apiserver.crt kube-apiserver.key \
        service-account.key service-account.crt \
        etcd-server.key etcd-server.crt \
        encryption-config.yaml /var/lib/kubernetes/
    }
    ```
  * the instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. retrieve the internal ip address of the master nodes
    ```
    INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
    ```
  * create the kube-apiserver.service systemd unit file
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
    [Unit]
    Description=Kubernetes API Server
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    ExecStart=/usr/local/bin/kube-apiserver \\
      --advertise-address=${INTERNAL_IP} \\
      --allow-privileged=true \\
      --apiserver-count=3 \\
      --audit-log-maxage=30 \\
      --audit-log-maxbackup=3 \\
      --audit-log-maxsize=100 \\
      --audit-log-path=/var/log/audit.log \\
      --authorization-mode=Node,RBAC \\
      --bind-address=0.0.0.0 \\
      --client-ca-file=/var/lib/kubernetes/ca.crt \\
      --enable-admission-plugins=NodeRestriction,ServiceAccount \\
      --enable-swagger-ui=true \\
      --enable-bootstrap-token-auth=true \\
      --etcd-cafile=/var/lib/kubernetes/ca.crt \\
      --etcd-certfile=/var/lib/kubernetes/etcd-server.crt \\
      --etcd-keyfile=/var/lib/kubernetes/etcd-server.key \\
      --etcd-servers=https://192.168.5.11:2379,https://192.168.5.12:2379 \\
      --event-ttl=1h \\
      --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
      --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \\
      --kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver.crt \\
      --kubelet-client-key=/var/lib/kubernetes/kube-apiserver.key \\
      --kubelet-https=true \\
      --runtime-config=api/all \\
      --service-account-key-file=/var/lib/kubernetes/service-account.crt \\
      --service-cluster-ip-range=10.96.0.0/24 \\
      --service-node-port-range=30000-32767 \\
      --tls-cert-file=/var/lib/kubernetes/kube-apiserver.crt \\
      --tls-private-key-file=/var/lib/kubernetes/kube-apiserver.key \\
      --v=2
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
* configure kube controller manager 
  * move the kube-controller-manager kubeconfig into place
    ```
    sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
    ```
  * create the kube-controller-manager.service systemd unit file  
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
    [Unit]
    Description=Kubernetes Controller Manager
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    ExecStart=/usr/local/bin/kube-controller-manager \\
      --address=0.0.0.0 \\
      --cluster-cidr=192.168.5.0/24 \\
      --cluster-name=kubernetes \\
      --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
      --cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
      --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
      --leader-elect=true \\
      --root-ca-file=/var/lib/kubernetes/ca.crt \\
      --service-account-private-key-file=/var/lib/kubernetes/service-account.key \\
      --service-cluster-ip-range=10.96.0.0/24 \\
      --use-service-account-credentials=true \\
      --v=2
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
* configure kube scheduller 
  * move the kube-scheduler kubeconfig into place
    ```
    sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
    ```
  * create the kube-scheduler.service systemd unit file  
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
    [Unit]
    Description=Kubernetes Scheduler
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    ExecStart=/usr/local/bin/kube-scheduler \\
      --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
      --address=127.0.0.1 \\
      --leader-elect=true \\
      --v=2
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
* start controller services
  ```
  {
    sudo systemctl daemon-reload
    sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
    sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
  }
  ```
* verify
  ```
  kubectl get componentstatuses --kubeconfig admin.kubeconfig
  ```
### bootstrap control plane load balancer
* install haproxy
  ```
  sudo apt-get update && sudo apt-get install -y haproxy
  ```
* configure haproxy
  ```
  cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg 
  frontend kubernetes
      bind 192.168.5.30:6443
      option tcplog
      mode tcp
      default_backend kubernetes-master-nodes

  backend kubernetes-master-nodes
      mode tcp
      balance roundrobin
      option tcp-check
      server master-1 192.168.5.11:6443 check fall 3 rise 2
      server master-2 192.168.5.12:6443 check fall 3 rise 2
  EOF
  ```
* restart 
  ```
  sudo service haproxy restart
  ```
* verify
  ```
  curl  https://192.168.5.30:6443/version -k
  ```
### bootstrap worker nodes
* worker nodes need the following components
  * kubelet => primary “node agent” that runs on each node, that can register the node with the apiserver using one of: the hostname; a flag to override the hostname; or specific logic for a cloud provider
  * kube-proxy => network proxy runs on each node. this reflects services as defined in the Kubernetes API on each node and can do simple TCP, UDP, and SCTP stream forwarding or round robin TCP, UDP, and SCTP forwarding across a set of backends
* kubelet setup
  * provisioning kubelet client certificates
    * kubernetes uses a special-purpose authorization mode called Node Authorizer, that specifically authorizes API requests made by kubelets. in order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the system:nodes group, with a username of system:node:<nodeName>
    * generate a certificate and private key for one worker node
      ```
      master-1$ cat > openssl-worker-1.cnf <<EOF
      [req]
      req_extensions = v3_req
      distinguished_name = req_distinguished_name
      [req_distinguished_name]
      [ v3_req ]
      basicConstraints = CA:FALSE
      keyUsage = nonRepudiation, digitalSignature, keyEncipherment
      subjectAltName = @alt_names
      [alt_names]
      DNS.1 = worker-1
      IP.1 = 192.168.5.21
      EOF

      openssl genrsa -out worker-1.key 2048
      openssl req -new -key worker-1.key -subj "/CN=system:node:worker-1/O=system:nodes" -out worker-1.csr -config openssl-worker-1.cnf
      openssl x509 -req -in worker-1.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out worker-1.crt -extensions v3_req -extfile openssl-worker-1.cnf -days 1000
      ```
  * when generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes Node Authorizer.
    * generate a kubeconfig file
      ```
      LOADBALANCER_ADDRESS=192.168.5.30
      
      {
        kubectl config set-cluster kubernetes-the-hard-way \
          --certificate-authority=ca.crt \
          --embed-certs=true \
          --server=https://${LOADBALANCER_ADDRESS}:6443 \
          --kubeconfig=worker-1.kubeconfig

        kubectl config set-credentials system:node:worker-1 \
          --client-certificate=worker-1.crt \
          --client-key=worker-1.key \
          --embed-certs=true \
          --kubeconfig=worker-1.kubeconfig

        kubectl config set-context default \
          --cluster=kubernetes-the-hard-way \
          --user=system:node:worker-1 \
          --kubeconfig=worker-1.kubeconfig

        kubectl config use-context default --kubeconfig=worker-1.kubeconfig
      }
      ```
    * copy certificates to the worker
      ```
      master-1$ scp ca.crt worker-1.crt worker-1.key worker-1.kubeconfig worker-1:~/
      ```
    * download and install worker binaries
      ```
      worker-1$ wget -q --show-progress --https-only --timestamping \
      https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubectl \
      https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-proxy \
      https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubelet
      ```
      ```
      worker-1$ sudo mkdir -p \
      /etc/cni/net.d \
      /opt/cni/bin \
      /var/lib/kubelet \
      /var/lib/kube-proxy \
      /var/lib/kubernetes \
      /var/run/kubernetes
      ```
      ```
      {
        chmod +x kubectl kube-proxy kubelet
        sudo mv kubectl kube-proxy kubelet /usr/local/bin/
      }
      ```
    * configure the Kubelet
      ```
      {
        sudo mv ${HOSTNAME}.key ${HOSTNAME}.crt /var/lib/kubelet/
        sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
        sudo mv ca.crt /var/lib/kubernetes/
      }
      ```
      ```
      worker-1$ cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
      kind: KubeletConfiguration
      apiVersion: kubelet.config.k8s.io/v1beta1
      authentication:
        anonymous:
          enabled: false
        webhook:
          enabled: true
        x509:
          clientCAFile: "/var/lib/kubernetes/ca.crt"
      authorization:
        mode: Webhook
      clusterDomain: "cluster.local"
      clusterDNS:
        - "10.96.0.10"
      resolvConf: "/run/systemd/resolve/resolv.conf"
      runtimeRequestTimeout: "15m"
      EOF
      ```
    * create the kubelet.service systemd unit file
      ```
      worker-1$ cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
      [Unit]
      Description=Kubernetes Kubelet
      Documentation=https://github.com/kubernetes/kubernetes
      After=docker.service
      Requires=docker.service

      [Service]
      ExecStart=/usr/local/bin/kubelet \\
        --config=/var/lib/kubelet/kubelet-config.yaml \\
        --image-pull-progress-deadline=2m \\
        --kubeconfig=/var/lib/kubelet/kubeconfig \\
        --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.crt \\
        --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}.key \\
        --network-plugin=cni \\
        --register-node=true \\
        --v=2
      Restart=on-failure
      RestartSec=5

      [Install]
      WantedBy=multi-user.target
      EOF
      ```
  * configure the Kubernetes Proxy
    ```
    worker-1$ sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
    ```
    * create the kube-proxy-config.yaml configuration file
      ```
      worker-1$ cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
      kind: KubeProxyConfiguration
      apiVersion: kubeproxy.config.k8s.io/v1alpha1
      clientConnection:
        kubeconfig: "/var/lib/kube-proxy/kubeconfig"
      mode: "iptables"
      clusterCIDR: "192.168.5.0/24"
      EOF
      ```
    * create the kube-proxy.service systemd unit file
      ```
      worker-1$ cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
      [Unit]
      Description=Kubernetes Kube Proxy
      Documentation=https://github.com/kubernetes/kubernetes

      [Service]
      ExecStart=/usr/local/bin/kube-proxy \\
        --config=/var/lib/kube-proxy/kube-proxy-config.yaml
      Restart=on-failure
      RestartSec=5

      [Install]
      WantedBy=multi-user.target
      EOF
      ```
* start the Worker Services
  ```
  {
    sudo systemctl daemon-reload
    sudo systemctl enable kubelet kube-proxy
    sudo systemctl start kubelet kube-proxy
  }
  ```
* verify
  ```
  master-1$ kubectl get nodes --kubeconfig admin.kubeconfig
  ```
### tls bootstrap worker nodes
### configure kubectl for remote access
### deploy pod networking solution => weave
### kube api server to kubelet config
### deploy the dns cluster add-on
### smoke test
### e2e test
### to be continued
