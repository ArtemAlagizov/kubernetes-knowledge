## networking
* [linux routing basics](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#linux-routing-basics)
* [linux dns basics](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#linux-dns-basics)
* [network namespaces](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#network-namespaces)
* [networking in docker](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking-in-docker)
* [kubernetes cluster networking](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#kubernetes-cluster-networking)
* [pod networking](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#pod-networking)
* [cni in kubernetes](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#cni-in-kubernetes)
* [cni weave](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#cni-weave)
* [service networking](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#service-networking)
* [dns in kubernetes](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#dns-in-kubernetes)
* [core dns](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#core-dns)

### linux routing basics
  * to see current routes use **route**
  * to list and modify interfaces on the host
    ```
    ip link
    ```
  * to see ip addresses assigned to those interfaces
    ```
    ip addr
    ```
  * set ip address on an interface
    ```
    ip addr add 192.168.1.10/24 dev eth0
    ```
  * to add a route
    ```
    ip route add 192.168.2.0/24 via 192.168.1.1
  * for systems to communicate with each other routing through gateways needs to be configured on all systems
  * to set a default router through which all the outgoing requests will go
    ```
    ip route add default via 192.168.2.1
    ```
  * to enable packet forwarding
    ```
    vi /proc/sys/net/ipv4/ip_forward
    # 0 is default => no forwarding
    ```
  * to persist packet forwarding
    ```
    vi /etc/sysctl.conf
    # net.ipv4.ip_forward = 1
    ```
  
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### linux dns basics
* point host to dns server
  ```
  vi /etc/resolv.conf
  # nameserver 192.168.1.100
  # nameserver 8.8.8.8
  # search mysite.com prod.mysite.com => will search for host names appending the search entries
  ```
* resolver first checks /etc/hosts and if doesn't find an entry there goes to nameserver
  * order can be changed in **/etc/nsswitch.conf** **hosts:**
    ```
    hosts:          files mdns4_minimal [NOTFOUND=return] dns
    ```
* record types on dns server
  * A => ip to host names
  * AAAA => ip6 to host names 
  * CNAME => mapping one name to another name
* tools to test dns
  * ping
  * nslookup => queries dns server
  * dig
  
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### network namespaces
* create new network namespace 
  ```
  ip netns add <namespace-name>
  ```
* list network namespaces
  ```
  ip netns
  ```
* execute a command inside a namespace
  ```
  ip netns exec <namespace-name> <command, ie ip link>
  # or
  ip -n <namespace-name> link
  ```
* connect 2 namespaces (red and blue) by virtual cable
  ```bash
  # create virtual cable
  ip link add veth-red type veth peer name veth-blue
  
  # attach cable ends to namespaces
  ip link set veth-red netns red
  ip link set veth-blue netns blue
  
  # assign ip addresses in these namespaces
  ip -n red addr add 192.168.15.1 dev veth-red
  ip -n blue addr add 192.168.15.2 dev veth-blue
  
  # bring up the interfaces within the 2 namespaces
  ip -n red link set veth-red up
  ip -n blue link set veth-blue up
  
  # check the connection
  ip -n red ping 192.168.15.2
  
  # delete a virtual cable
  ip -n red link del veth-red # second end gets deleted automatically
  ```
* check arp table of a namespace
  ```
  ip -n red arp
  ```
* create internal network on a host
  * one option is to use **linux bridge** (virtual switch)
    ```bash
    # create a bridge
    ip link add v-net-0 type bridge
    
    # bring the bridge up
    ip link set dev v-net-0 up
    
    # connect namespaces to the bridge
    ip link set veth-red netns red
    ip link set veth-red-br master v-net-0
    ip link set veth-red netns blue
    ip link set veth-blue-br master v-net-0
    ip -n red addr add 192.168.15.1 dev veth-red
    ip -n blue addr add 192.168.15.2 dev veth-blue
    ip -n red link set veth-red up
    ip -n blue link set veth-blue up
    ```
  * connect the bridge to the host
    ```
    ip addr add 192.168.15.5/24 dev v-net-0
    ```
  * connect namespace to another location on LAN (192.168.1.3)
    ```
    ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5
    ```
  * enable host to send messages to another location on LAN
    ```bash
    # add NAT on the host, this way when receiving packets from namespace it would appear coming from the host
    iptables -t -nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
    
    # check if works
    ip netns exec blue ping 192.168.1.3
    ```
  * enable connectivity to the outer world
    ```bash
    ip netns exec blue ip route add default via 192.168.15.5
    
    # check if works
    ip netns exec blue ping 8.8.8.8
    ```
  * enable outer world to reach blue namespace
    ```bash
    # route incoming traffic to the blue namespace
    iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.15.2:80 -j DNAT
    ```

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### networking in docker
* types of network
  * none => no connectivity to/from the container
    ```
    docker run --network none nginx
    ```
  * host => share host networking, no port mapping needed, it will be available on the port exposed in the container
    ```bash
    docker run --network host nginx
    
    # nginx will be available at http://<host-ip>:80
    ```
  * bridge => internal virtual network
    ```bash
    docker run --network bridge nginx
    ```
* docker creates virtual cables from bridge network to containers as described above
* docker allows to map ports for accessing a container through docker host
  ```bash
  docker run -p 8080:80 nginx
  
  # after http://<host-ip>:8080 will lead to port 80 of the nginx container  
  
  # docker runs the command below once the command above is executed  
  iptables -t nat -A DOCKER -j DNAT --dport 8080 --to-destination http://<internal-container-ip>:80
  ```
  * to see the rules docker creates
    ```
    iptables -nvL -t nat
    ```
 * cni (container network interface)
   * the set of step described above is done by all container-runtimes like docker, rkt, mesos, kubernetes
   * bridge program was created to handle this for container-runtimes
   * container network interface needed to be specified
     * cni is implemented by rkt, mesos, kubernetes but not docker (it has container network model => cnm)
       * it is still possible to use cni plugins with docker, it needs to be done in 2 steps (kubernetes does it)
         * create conteainer without network
           ```
           docker run --network=none nginx
           ```
         * invoke the bridge plugin 
           ```
           bridge add 2e34dcf34 /var/run/netns/2e34dcf34
           ```
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### kubernetes cluster networking 
* requerements for nodes
  * each node must have uniq
    * ip address
    * hostname
    * mac
  * master node    
    * ports should be available
      * 6443 for kube-api
      * 10250 for kubelet
      * 10251 for kube-scheduler
      * 10252 for kube-controller-manager
      * 2379 for etcd
      * 2380 for etcd in case of multimaster
  * worker node
    * ports should be available
       * 30000-32767 for services
       * 10250 for kubelet
       * 10251 for kube-scheduler
       * 10252 for kube-controller-manager
* get ip address of a node
  ```
  kubectl get node -o wide
  ```
* get mac of the master node
  ```
  ip link show ens3
* get mac of a node
  ```
  arp <node-name>
  ```
* get state of an interface
  ```
  ip link show <interface-name, ie docker0>
  ```
* check which route is used to ping google from the master node
  ```
  ip route show default
  ```
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### pod networking
* no current solution for pod networking => has to be done by us
* requirements
  * every pod should have an ip address
  * every pod should be able to communicate with every other pod in the same node
  * every pod should be able to communicate with every other pod on other nodes without NAT
* some solutions for this are => weavworks, flannel
* custom solution (based on the above)
  * given a set of nodes on a network
    * to enable connectivity within a pod
      * create bridge network on every node
      * choose ip range and give each node a subnet, 10.44.2.1/24, 10.44.3.1/24
      * set the ip range for the bridge interfaces
      * create a script that will assign ip address to containers
        ```bash
        # create virtual cable
        ip link add veth-red type veth peer name veth-blue

        # attach cable ends to namespaces
        ip link set veth-red netns red
        ip link set veth-blue netns blue

        # assign ip addresses in these namespaces
        ip -n red addr add 192.168.15.1 dev veth-red
        ip -n blue addr add 192.168.15.2 dev veth-blue

        # bring up the interfaces within the 2 namespaces
        ip -n red link set veth-red up
        ip -n blue link set veth-blue up
        ```
    * to enable pod-pod connectivty
      * add entry into routing table of the node 1 to route traffic to internal ip of pod on node 2 through external ip of node 2
        ```bash
        # on node1:
        ip route add <internal-ip-of-pod-on-node2> via <external-ip-of-node-2>
        ```
      * better solution
        * define all these routes on a router
        * point all host to use it as a default gateway
          | network     |gateway     |
          |:------------|:-----------|
          |10.244.1.0/24|192.168.1.11|
          |10.244.2.0/24|192.168.1.12|
          |10.244.3.0/24|192.168.1.13|        
          * this way it becomes a big bridge network across nodes (10.244.0.0/16)
        * cni standards have specific requirements for the script for assigning ip addresses to containers
          * there should be ADD section
          * there should be DEL section
        * kubelet on each node needs to be configured to use the script
          * --cni-conf-dir=/etc/cni/net.d (to get script name)
          * --cni-bin-dir=/etc/cni/bin (to get script itself)
          * it runs 
            ```
            ./net-script.sh add <container> <namespace>
            ```
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### cni in kubernetes
* cni service is configured in kubelet service of each node
  ```bash
  # kubelet.service file
  --network-plugin=cni
  
  # also by running the command
  ps -aux | grep kubelet
  ```
* cni bin directory contains all the supported cni plugin executables
  ```bash
  ls /opt/cni/bin
  ```
* cni config directory has list of config files 
  ```bash
  # this is where kubelet looks to see which plugin needs to be used
  # in case of multiple files it will choose in alphabetical order
  ls /etc/cni/net.d
  ```
  * example of config file
    ```json
    {
      "cniVersion": "0.2.0",
      "name": "mynet",
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16",
        "routes": [
          { 
            "dst": "0.0.0.0/0"
          }
        ]
      }
    }
    ```
    ```bash
    # isGateway => indicates if bridge network should get an ip address assigned to it so that it can act as a gateway
    # ipMasq => indicates if ip address should be masquaraded
    # ipam => range of ip addresses and routes that will be assigned to pods
    #   type => host-local indicates that ip addresses will be managed locally, not remotely
    ```
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### cni weave
* weave agent on every node
  * agents have information on network topology, pod ip addresses, so on
  * agents share this information with each other
  * packets are sent to agents that handle them further
* deployment
  * deploy weave agents in pods as daemonset on each node
    ```
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
    ```
  * check logs
    ```
    kubectl logs weave-net-5wfwf weave -n kube-system
    ```
* check ip range used by weave
  ```
  ip addr show weave
  ```
* get default gateway configured on the pods scheduled on particular node
  ```
  kubectl exec -it <pod-name> sh
  # ip route 
  ```
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### service networking
* services are virtual objects with an ip address
* services are cluster wide concepts
* service available within a cluster (type clusterIp)
* outside world can access pod via service of NodePort type
* once a service is created it gets handled by kube-proxy
  * kube-proxy creates forwarding rules from service ip:port to pod ip
* get the range of IP addresses configured for PODs on this cluster
  ```bash
  # ipalloc-range here
  kubectl logs <weave-pod-name> weave -n kube-system  
  ```
* get IP Range configured for the services within the cluster
  ```bash
   ps -aux | grep kube-api
   
   --service-cluster-ip-range=10.96.0.0/12
  ```
* get type of proxy is the kube-proxy configured to use
  ```
  kubectl logs kube-proxy-ft6n7 -n kube-system
  ```
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### dns in kubernetes
* services are resolved in the following way
  ```bash
  http://<hostname><namespace><type><root>
  
  # example: http://web-service.apps.svc.cluster.local
  ```
  
* for pods kubernetes creates hostnames by replacing dots with dashes in the ip address of the pod
  ```
  10.244.1.4 => 10-244-1-4
  ```
  
[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### core dns
* core dns service is deployed as 2 pods in a cluster (replicaset in a deployment)
  * the pod runs coredns executable
  * the pod watches new pods/services being created 
    * once pod/service created coredns adds a record about it to its database
* regular pods point to coredns service in **/etc/resolv.conf** as a nameserver
  * this is done automatically by kubernetes when pods are created (kubelet does it)
    ```
    nameserver 10.96.0.11

    # search allows to react coredns service by any of the below names
    search default.svc.cluster.local svc.cluster.local cluster.local
    ```
* core dns requires **Corefile**
  * contains configured plugins for reporting health, managing metrics, cache, ...
    ```bash
    # cat /etc/coredns/Corefile

    .:53 {
          errors
          health
          ready
          kubernetes cluster.local in-addr.arpa ip6.arpa {
             pods insecure
             fallthrough in-addr.arpa ip6.arpa
             ttl 30
          }
          prometheus :9153
          forward . /etc/resolv.conf
          cache 30
          loop
          reload
          loadbalance
      }
    ```
  * plugin that makes coredns work with kubernetes is named kubernetes
    * that is where top-level domain name is set (cluster.local)
    * pods option is responsible for creating pods in the cluster  
  * any record that dns server cannot solve is forwarded to proxy plugin
  * **Corefile** is passed in to the pod as configmap object
    * to modify the config => modify the configmap object
 * identify the DNS solution implemented in this cluster
   ```
   kubectl get pods -n kube-system
   ```
 * get ip of the coredns server that should be configured on pods to resolve services
   ```bash
   # cluster ip value in the output of the following command
   kubectl get service -n kube-system 
   ```
 * find is the configuration file located for configuring the coredns service
   ```
   kubectl exec <core-dns-pod-name> -n kube-system ps
   ```
 * redirect the output from pod-0 mysql service nslookup to a file /root/nslookup.out
   ```
   kubectl exec -it hr nslookup mysql.payroll > /root/nslookup.out
   ```
 * list dns records of coredns
   ```
   not known yet TODO
   ```

[back to the top](https://github.com/ArtemAlagizov/kubernetes-knowledge/blob/master/topics/networking.md#networking)
### ingress
* layer serving load-balancer built in a cluster that can be configured using native kubernetes primitives
  * ssl included
  * still needs to be exposed to the outer world
* **ingress controller**
  * deployed solution (nginx, haproxy, traefik)
    * nginx is deployed as **Deployment** kind
      ```bash
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: ingress-controller
        namespace: ingress-space
      spec:
        replicas: 1
        selector:
          matchLabels:
            name: nginx-ingress
        template:
          metadata:
            labels:
              name: nginx-ingress
          spec:
            serviceAccountName: ingress-serviceaccount
            containers:
              - name: nginx-ingress-controller
                image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
                args:
                  - /nginx-ingress-controller
                  - --configmap=$(POD_NAMESPACE)/nginx-configuration
                  - --default-backend-service=app-space/default-http-backend
                env:
                  - name: POD_NAME
                    valueFrom:
                      fieldRef:
                        fieldPath: metadata.name
                  - name: POD_NAMESPACE
                    valueFrom:
                      fieldRef:
                        fieldPath: metadata.namespace
                ports:
                  - name: http
                    containerPort: 80
                  - name: https
                    containerPort: 443
      ```
      * monitors new resources to update nginx configuration accordingly
        * needs **ServiceAccount** for permission to do so
          ```bash
          
          ```
          * create a ServiceAccount
            ```bash
            kubectl create serviceaccount ingress-serviceaccount --namespace ingress-space
            ```
    * regular nginx properties (log-path,keep-alive,ssl-protocols) need to be passed in as **ConfigMap** object
      ```bash
      
      ```     
      * create a ConfigMap object
        ```
        kubectl create configmap nginx-configuration --namespace ingress-space
        ```      
  * doesn't come by default in a cluster   
  * **Service** is needed to expose ingress controller to the outer space
    ```bash
    apiVersion: v1
    kind: Service
    metadata:
      name: ingress
      namespace: ingress-space
    spec:
      type: NodePort
      ports:
      - port: 80
        targetPort: 80
        protocol: TCP
        nodePort: 30080
        name: http
      - port: 443
        targetPort: 443
        protocol: TCP
        name: https
      selector:
        name: nginx-ingress
    ```
    * create Service object
      ```
      kubectl expose deployment -n ingress-space ingress-controller --type=NodePort --port=80 --name=ingress --dry-run -o yaml >ingress.yaml
      ```
* **ingress resources**
  * set of rules for ingress to route traffic for different domain names of a website such as wear.online-store.com, watch.online-store.com, ...
    * each rule can have a set of paths for the domain
      * if rules don't cover path traffic goes to default backend
  * 1 rule 2 paths  
    ```
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
        nginx.ingress.kubernetes.io/ssl-redirect: "false"      
      name: ingress-wear-watch
      namespace: app-space
      resourceVersion: "1693"
    spec:
      rules:
      - http:
          paths:
          - backend:
              serviceName: wear-service
              servicePort: 8080
            path: /wear
          - backend:
              serviceName: video-service
              servicePort: 8080
            path: /stream
    status:
      loadBalancer:
        ingress:
        - {}
    ```    
    or 2 rules 1 path
    ```
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
        nginx.ingress.kubernetes.io/ssl-redirect: "false"      
      name: ingress-wear-watch
      namespace: app-space
      resourceVersion: "1693"
    spec:
      rules:
      - http:
          host: watch.online-store.com
          paths:
          - backend:
              serviceName: video-service
              servicePort: 8080
      - http:
          host: wear.online-store.com
          paths:
          - backend:
              serviceName: wear-service
              servicePort: 8080
    status:
      loadBalancer:
        ingress:
        - {}
    ```
    * [rewrite-target annotation](https://kubernetes.github.io/ingress-nginx/examples/rewrite/)
* edit ingress resource
  ```
  kubectl edit ingress --namespace app-space
  ```
