## TLS

### procedure
* actors
  * CertificateAuthority
  * server
  * admin
* steps
  * admin generates a key pair to establishe ssh connection to the server
  * server generates a key pair for securing the website
  * to encrypt http traffic server sends verification request with certificate to the CA
  * CA signs the certificate using their own private key and sends it back to the server
  * server configures the web app on the server to use the certificate
  * when request comes from the browser server sends the certificate
  * browser fetches public key from CA and verifies the quiality of the certificate
  * browser generates a symmetric key that it wishes to use for the further communication
  * browser fetches server public key and encrypts the symmetric key and sends it to the server
  * server decrypts the message and stores the symmetric key for further communication
  * optionally client can generate a key-pair, verifty it with CA and send it to the server
### types of certificates
* root 
  * configured at CA servers
* client
  * configured on clients
    * admin (admin.crt, admin.key)
    * kube-proxy (proxy.crt, proxy.key)
    * kube-scheduler (scheduler.crt, scheduler.key)
    * kube-controller-manager (controller-manager.crt, controller-manager.key)
* server
  * configured on servers
    * kube-apiserver (apiserver.crt, apiserver.key)
      * is a client of etcd as this is the only thing talking to etcd server
    * etcd server (etcd.crt, etcd.key)
    * kubelet server (kubelet.crt, kubelet.key)
    
 ### certificate creation
 * create certificate and key pair for CA using openssl
   * generate keys 
     ```
     openssl genrsa -out ca.key 2048
     ```
   * certificate signing request
     ```
     openssl req -new -key ca.key  -subj "/CN=KUBERNETES-CA" -out ca.csr
     ```
   * sign certificates
     ```
     openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
     openssl x509 -req -in /etc/kubernetes/pki/apiserver-etcd-client.csr -CA /etc/kubernetes/pki/etcd/ca.crt -CAkey /etc/kubernetes/pki/etcd/ca.key -CAcreateserial -out /etc/kubernetes/pki/apiserver-etcd-client.crt
     ```
* create certificate and key pair for admin and CA certificate using openssl
  * generate keys 
    ```
    openssl genrsa -out admin.key 2048
    ```
  * certificate signing request
    ```
    openssl req -new -key ca.key  -subj "/CN=kube-admin/O=system:masters" -out admin.csr
    ```
  * sign certificates with CA, which makes it a vlid certifiate within the cluster
    ```
    openssl x509 -req -in -CA admin.csr -CAKey ca.key ca.key -out admin.crt
     ```
* in the same way create certificate and key pair for contol plane components
  * /CN=system-kube-scheduler
  * /CN=system-kube-control-manager
  * /CN=system-kube-proxy
* create certificate for server in the same way
  * for kube-api server extra params to pass is openssl.cnf, containing aliases and ip
  * for kubelet each node must generate its' own certificate (kubelet.yaml)
    * also generate certificate for communicating with kube-api server
      * use naming of system:node:node01
      * also add nodes to GROUP system:group

### view certificates
* get kube-api server certificates details
  ```
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
  ```
  * CN
  * alternative names
  * issuer
  * expiration date
### check logs
* natively
  ```
  journalctl -u etcd.service -l
  ```
* in a pod
  ```
  kubectl logs etcd-master
  ```
### certificate api
* CertificateSigningRequest example file
  ```
  apiVersion: certificates.k8s.io/v1beta1
  kind: CertificateSigningRequest
  metadata:
    name: akshay
  spec:
    groups:
    - system:authenticated
    request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZV3R6YUdGNU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQXkyYTQ3blZJbFZuVi8vcStLOFVVMVlmeFhkOTB3TmJ0dzRSTmtWQlFkSVVOCkRQNHVMa2hmMVU2Vk1RbTljZ1F6MUtOTmdOU3BFc1E4SHJ0VkxZWDFrWWh6cnNnUWZ0dm5uV1BoVTdDRWZyeWEKZVpSQ1Bvb210bEl2WFZtSFZXcVJXNmFCaTNoajcybmQwTmEycElWbmF2OWY2KzcvT1RuMFRtSTJoeUthd2IxUwpYOW4rNm8vMlN1c2hyN3VZeDA1ZUVXVyt2QnBWK0t2ZzhMNWhMY0Z0bWx1TVJDRVFCeTFTQ05TNlZtRTdRVWNPCmlIYWgzUCsrZktQdlRreUcwMnBCZUlZdG15aTg5RkRZZFZCbGJ3TWdpa002QWZXTHRsbmpxVElCaEFyOU9LYngKV1RYeUFDdHhTa25CTkRoM1l3RWEwTGxBSTBBYWU4Y0lKeXFKYnlqR3Z3SURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBQUZFODNZQnlTdXB5MUNuelVsNzRPbFB5VTlwR2t3ZHppczBNQjF4cnBkR1pKcndIVjNrCnd2Unphclc5WWlRZE9icFlMWEJuRUJlU2RMYityQnBSa1N2R3BYSzlhOVFZZm0rb21uWDlSbjVsNWp0eG5Gc2UKdUNabHpJQVIzYXRBYjNtUmVRVXBWRlFuYlRqcVhnSDhYVEhJSVM0QXduK3A4VEtCa1dFUzhPRkhzNWRRNnRUOApFYUVtWlYyZnlxZzlsQ0FpeXBwZlRDKy9rUkVYeDBlSVluMHhaUFNuR3grbm9BUFh2T3UxN3diNWp3ZlNMNGI3CnlQN1lUQndReUEwa0VjYTlNWFBleSs2WW5QUEFlMEJiM0VGQnE2NjRIa2lZMW04ZHpXUnNSNVpVODQ0QlJPaE4KUHBBTDVvdFFFUE5PRWxtaUNyNkQ4amUwSG5haWNIZUc4WkE9Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
    usages:
    - digital signature
    - key encipherment
    - server auth
  ```
* get base64 version of a certificate request (crt)
  ```
  base64 -w 0 akshay.csr
  ```
* get cert requests
  ```
  kubectl get csr
  ```
* approve cert request
  ```
  kubectl certificate approve akshay
  ```
* deny cert request
  ```
  kubectl certificate deny agent-smith
  ```
### kube config
* needed for convenience and to not type every time --server=... --client-key=..., ... to commands like **kubectl get pods**
* 3 main sections
  * users
  * clusters
  * context => connects the above two, **current-context** is the default (or %HOME/.kube/config)
* view config file
  ```
  kubectl config view
  ```
* switch context
  ```
  kubect config --kubeconfig=/root/my-kube-config use-context user@cluster
  ```
* example file
  ```
  apiVersion: v1
  clusters:
  - cluster:
      certificate-authority-data: DATA+OMITTED
      server: https://172.17.0.16:6443
    name: kubernetes
  contexts:
  - context:
      cluster: kubernetes
      user: kubernetes-admin
      namespace: default
    name: kubernetes-admin@kubernetes
  current-context: kubernetes-admin@kubernetes
  kind: Config
  preferences: {}
  users:
  - name: agent-x
    user:
      client-certificate: /etc/kubernetes/pki/users/agent-x/agent-x.crt
      client-key: /etc/kubernetes/pki/users/agent-x/agent-x.key
  - name: kubernetes-admin
    user:
      client-certificate-data: REDACTED
      client-key-data: REDACTED
  ```
