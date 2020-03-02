## networking
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
  
