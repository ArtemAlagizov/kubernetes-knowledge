## spin up 5 node cluster locally [based on [this](https://github.com/mmumshad/kubernetes-the-hard-way)]
### setup
2 masters, 2 workers, 1 loadbalancer

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
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD......8+08b vagrant@master-1
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
#### provision a Certificate Authority that can be used to sign additional TLS certificates
* create a CA certificate, then generate a Certificate Signing Request and use it to create a private key
  ```
  # remove the following line at /etc/ssl/openssl.conf
  RANDFILE                = $ENV::HOME/.rnd
  
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
