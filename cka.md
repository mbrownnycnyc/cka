* https://app.pluralsight.com/course-player?clipId=18a74fb8-708b-4c8f-964b-41758e7245cb

# Kubernetes installation and configuration fundamentals

## Course Overview

## Exploring the Kubernetes Architecture

### What Is Kubernetes? Kubernetes Benefits and Operating Principles

* what is kubernetes?
  * container orchestrator
  * workload placement: deploy container based app to hardware
  * provides an infrastructure abstraction
    * LBs, hardware allocation, etc
  * maintains desired state
* benefits of kubernetes
  * speed of deployment
    * absorb change quickly
  * ability to recover quickly
    * achieves desired state repeatedly
  * hide infrastructure complexity in the cluster
    * storage, network place, etc

#### kubernetes principles
* desired state/declarative config
  * write code to describe deployment, kubernetes brings it online
* controllers/control loop
  * monitors and maintains desired state
  * if failure, then the controller will bring the service up.
* kubernetes API/API server
  * collection of objects that we can use to build and define systems
  * API server is central comm hub for cluster
  * controller uses the API server to acheive desired state

### Introducing the Kubernetes API - Objects and API Server

* API objects
  * collection of primitives to represent your system's state
    * ex: pod, nodes
  * enables config of state
    * declaratively: describe the state that should be acheived
    * imperatively: (manually) executed a bunch of commands to achieve a state
* API server
  * REST API over HTTP using JSON
  * only way that we interact with our cluster
  * only way kubernetes interacts with your cluster
  * desired state is serialized and persisted into the cluster data store

### Understanding API Objects - Pods
* single or collection of containers that we deploy as a single unit.  The 
* unit of scheduling work.
* defined in the manifest.
* ephemeral: no pod is ever redeployed.
* atomicity: the pod is there or it's not.
  * multicontainer pods: if one container dies, then the entire pod becomes unavailable.

* k8s Controllers are responsible (via the API server) to maintain communication with a Pod to maintain the desired state by:
  * observing state of pod: up/down
  * health of pod: application is good/bad via Probes
    * Probes: within manifest, you define Probe checks.

### Understanding API Objects - Controllers
* keeps the system in a desired state.
* Controllers are exposed via Workload resource API objects via the API Server.
  * create and manage Pods for you
* Controllers monitor and respond to the state of health.

#### Controller type: ReplicaSet controller
* define a number of replicas that should be running at all times.
* you generally don't create `ReplicaSets` directly, but create `Deployments`.

### controller type: Deployment controller
* admin defines a deployment.
* Deployment manages the rollout of the ReplicaSet.
  * commonly used when deploying container versions
  * can roll back

#### Understanding API Objects - Services
* provide a persistent access point to applications that we deploy in Pods
  * as things change, and controllers deploy new Pods, the Services provide a single point of access.
  * persists IP and DNS name for the service
  * networking abstraction for Pod access
* Services are dynamically updated based on Pod lifecycle
  * updates routing information, firewall config, etc.
* Services can be scaled by adding/removing pods
* provides load balancing

#### Understanding API Objects - Storage
* store data persistently
* Persistent Volume: pod independant storage that's defined at the Cluster level
  * when a Pod wants access to a storage, defines a persistent volume claim
  * decouples pod from storage

### Kubernetes Cluster Components Overview and Control Plane
* control plane node: implements major control functions of a cluster (used to be called "Master Node")
  * coordinates:
    * cluster operations
    * monitoring
    * pod scheduling
    * access point for cluster amdin
* Node/worker node
  * responsible for starting pods and containers
  * implement networking
  * contributes to compute capacity
  * Cluster is made up of many nodes (depending on configuration)
  * can be virtual or physical machines

#### control plane node
![](2022-08-30-06-44-16.png)

* `API Server`: primary access point for cluster admin, comm hub, it's stateless.
  * `kubectl` interacts with the API Server to configure.
  * central to the control of cluster, config changes are communicated via this component.
  * Simple interface.
  * REST API: GET, PUSH, POST
  * persists state of cluster to `etcd`
* Cluster store `etcd`: persists the state of the cluster objects
  * persists states
  * API Object maintainance
  * Stores state as key-value pairs
* `Scheduler`: tells k8s which nodes to start pods on based on the pod's desired state
  * watches API server for unscheduled pods
  * schedules pods on nodes
  * evaluates resources needed for pods
  * respects any contraints described for pods
* `Controller Manager`: implementing lifecycle functions of the Controllers, that control and monitor the state of the objects such as pods.  This is what commands Controllers to maintain the desired state.
  * controller loop execution
  * implement lifecycle functions and desired states
  * watch and update the API server of the state of the cluster
  * Made up of specific Controllers with specific roles, such as `ReplicaSet`

### Nodes
* a node is where the app pods run
* starts a pod and ensures the containers in pods are up and running
* implement networking
* can have many nodes in cluster based on scalability requirements
* nodes are either physical machines or VMs

#### node components
* each of these components run on all the nodes (including control plane node)

* `kubelet`: responsible for starting/stopping pods on nodes.
  * communicates directly with `API Server` to monitor for changes in the environment... sending info and rcving commands relevant to the `kubelet`'s role (to start pods).
  * monitors API server for changes
  * responsible for pod lifecycle (starting and stopping pods [and containers])
  * reports to API server on Node and Pod state
  * Pod health probe execution
* `kube-proxy`: responsible for pod networking and implementing out services abstraction ont he node itself.
  * communicates directly with `API Server` to monitor for changes in the environment... sending info and rcving commands relevant to the `kube-proxy`'s role (to change network topology).
  * `iptables`
  * implements services abstraction
  * routing traffic to Pods
  * load balancing
* `container runtime`: the actual runtime env, responsible for pulling container image from container registry and providing execution environment for container image and pod abstraction
  * downloads images and runs containers.
  * wrapped within Container Runtime Interface (CRI)
    * can swap out container runtime.
  * `containerd`: default runtime used by kubernetes, it is CRI compliant.
    * in `v1.20`, Docker was depreciated as the default container runtime.  It will be removed in `v1.22`, but you can still use containers built for Docker.


### Cluster Add-on Pods
* pods that provide special services to the cluster itself.
  * DNS:
    * provide DNS services within cluster via coreDNS server
    * IPs for the services and the search suffix is placed into the network config for any pods within the cluster via the cluster API server
    * commonly used for service discovery
  * Ingress controllers (optional)
    * advanced HTTP and layer 7 load balancers that handle content routing requests
  * Dashboard (optional)
    * for web based admin of k8s cluster
  * network overlays


### Pod Operations
* a cluster is established
  * this means it has a control plane node and two worker nodes
  * using `kubectl` we submit commands to configure a cluster, to the control plane node, that define
    * we want 3 replicas of the pod (submitted to API server, and stored to `etcd`)
    * controller manager will spin up the three requested replicas in replica set, which is submitted to the scheduler.
    * the scheduler then tells API server that the pods need to be spun up on nodes it selects, and the config decisions are written to `etcd`
      * which nodes did the pod replicas get scheduled on?
      * depedant on resources requested in the pods, and the resources available in the nodes in the cluster.
      * the `kubelet`s on the nodes will check in with the API server to see if there is any work queued for execution.
        * once the queued work is received, the pod will be spun up.
  * The controller manager receives information from the nodes regarding the state of pods running on each node.
    * if a node drops, the node is no longer reporting state.
      * controller manager will signal to the scheduler to locate a node to schedule the replica of the pod on.
        * by default the control plane node is NOT to be used for pod replica targets, only on worker node.

### Service Operations
* network or access point to applicaitons running within cluster
* review at cluster level
* pods are created within cluster for web app
* expose access with a Service, running HTTP on tcp 80
  * users can access on port 80
  * fixed and persistent service endpoint, which will be a DNS name or ip address
    * requests will be load balanced across replica set of pods
* services abstract access to pods' services
  * "self-healing" of underlying pods within a replica set occurs by the replica set controller

### Kubernetes Networking Fundamentals
* every pod deployed gets assigned a unique IP addr.
* pods on a node can communicate with all pods on all nodes in a cluster without NAT.
* agents on a node (kubelet, etc) can communicate with all pods on that node.

#### k8s network design

![](2023-01-13-16-47-22.png)

* multi container pod within cluster (within a node)
  * the containers within this pod communicate via namespaces within localhost
* additional pods deployed to the cluster (within a node)
  * these pods will inter-communicate via a software defined network bridge, using the real IPs of the pods themslves.
* pod on one node need to reach out to a pod on a second node
  * occurs between the real IPs of the pods themselves, so layer 2 and/or 3 connectivity must exist between the nodes
  * overlay network
    * if you don't control the underlying network infrastructure (inter-node), then you can deploy an overlay network to provide the overlay of layer 2/3 connectivity inter-node.
* external services
  * `kube-proxy` exposes the service within a cluster to external clients
    * the `service` then interacts with the pods.


## Installing and Configuring Kubernetes

### Installation Considerations
* where are you going to install?
  * cloud
    * two major use cases
      * IaaS: VMs as nodes
        * OS and k8s cluster must be managed.
      * PaaS: managed service
        * lose flexibility in versioning.
  * on prem
    * base metal or VMs
  * which one to choose?
    * skill set?c
    * cloud footprint already?
* cluster networking
  * overlay network versus metal r+s
* scalability
  * nodes, etc
* HA and DR
  * single control plane node?

### Installation Methods
* desktop installation
  * dev environments
    * docker-desktop
    * lens
* `kubeadm`
  * bootstraps cluster quickly
* cloud IaaS/PaaS

### Installation Requirements
* system requirements
  * need linux (ubuntu/RHEL)
  * 2 CPUs, 2GB RAM
  * swap is disabled on *nix
* container runtime
  * CRI compatbiel
    * containerd <-- we'll use this
    * docker
    * CRI-O
* networking
  * connectivity between all nodes
  * nodes need unique hostnames
  * nodes need unique MAC addresses

### Understanding Cluster Networking Ports
* setting up security perimters
  * control plane node provides services to the cluster
  * working nodes need access to the API server
* API server: tcp 6443, used by all cluster items (and admin via `kubectl`)
* `etcd`: tcp 2379-2380, used by API server and any etcd replicas
* Scheduler: tcp 10251, used by itself only (localhost)
* Controller Manager: tcp 10252, used by itself only (localhost)
* `kubelet`: tcp 10250, control plane services
  * worker nodes also run kubelets, tcp 10250, control plane needs access to worker node's kubelets
  * NodePort service: tcp 30000-32767, used by components that need access to the services published on the NodePorts
    * NodePort service: exposes `services` via ports on each node in cluster, and port ranges are allocated from the tcp port range.


### Getting Kubernetes
* github.com/kubernetes/kubernetes
* *nix repos

### Building Your Own Cluster
* steps
  * install and configure packages
  * create the cluster
  * configure pod networking
  * join nodes to cluster
* required packages on all nodes (worker or control plane)
  * container runtime: `containerd`
  * `kubelet`
  * `kubeadm`: create cluster, joins nodes
  * `kubectl`: configure pod network, etc.
* 

### Installing Kubernetes on VMs
* refer to wsl2.md in this repo for some work I did in that area, however I need to move on now... so I'm going to use vmware player and build four VMs.
* you should grab vmware workstation pro download and then extract `vmnetcfg.exe` from the installer and place it in `C:\Program Files (x86)\VMware\VMware Player`
  * configure the vswitch to be on network `172.16.94.0/24`
* "ubuntucontrol" = 172.16.94.10,"ubuntuworkernode1" = 172.16.94.11,"ubuntuworkernode2" = 172.16.94.12,"ubuntuworkernode3" = 172.16.94.13
  * Onboard as NAT networking type
  * update local Windows host at `C:\windows\system32\drivers\etc\hosts` for all nodes' IPs
* after ubuntu 22.02 install, configure as follows:
  * set each to a static IP within the vmware player established vswitch subnet by creating a netplan config at `/etc/netplan/99_config.yaml`, `sudo netplan try --state /etc/netplan/`, then `sudo netplan apply`
    ```
    #it's a yaml, so indents matter
    network:
      version: 2
      renderer: networkd
      ethernets:
        ens33:
          addresses:
            - 172.16.94.10/24
          routes:
            - to: default
              via: 172.16.94.2
          nameservers:
              search: [localdomain]
              addresses: [172.16.94.2]
    ```

### Lab Environment Overview

![](2023-03-30-08-16-57.png)

* one control plane node, and three worker nodes
  * `kubectl` on control plane node
  * ubuntu version 22.04
  * remember to disable swap
  * add `/etc/hosts` entries for each node



### Demo: Installing and Configuring containerd

* goals:
  * install the following
    * containerd
    * kubelet
    * kubeadm
    * kubectl
  * review how `systemd` manages these
   
* interface with the control plane VM, decrease swappiness, then install containerd
```
#disable swap
sudo sed -i '/ swap / s/^/#/' /etc/fstab
sudo swapoff -a

#containerd prereqs
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
# affect at runtime
sudo modprobe overlay
sudo modprobe br_netfilter

#k8s prereqs
cat <<EOF | sudo tee /etc/sysctl.d/00-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
# affect at runtime
sudo sysctl --system


sudo apt update -y

#install and configure containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
#set the cgroup driver to systemd in /etc/containerd/config.toml
$below `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]` find and change the following value:
SystemdCgroup = true
sudo systemctl restart containerd
```

### Demo: Installing and Configuring Kubernetes Packages

1. install kubernetes packages

```
# install kubernetes
# https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo bash -c 'cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io kubernetes-xenial main
EOF'

sudo apt update
#list versions
apt-cache policy kubelet | head -n 20

#pin to a specific version during install
VERSION=1.20.1-00
sudo apt install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION

#disable upgrading of these packages by apt so that you can control what versions you're using:
sudo apt-mark hold kubelet kubeadm kubectl containerd
```

2. review kubelet systemd unit status, note that it will fail to start because there's no cluster config (see next section)
```
sudo systemctl status kubelet.service
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Thu 2023-03-30 10:10:22 UTC; 1s ago
       Docs: https://kubernetes.io/docs/home/
    Process: 4049 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255/EXCEPTION)
   Main PID: 4049 (code=exited, status=255/EXCEPTION)
        CPU: 100ms

Mar 30 10:10:22 ubuntucontrol systemd[1]: kubelet.service: Main process exited, code=exited, status=255/EXCEPTION
Mar 30 10:10:22 ubuntucontrol systemd[1]: kubelet.service: Failed with result 'exit-code'.
```

3. review containerd systemd unit status
```
sudo systemctl status containerd.service
```

4. set both `kubelet` and `containerd` to start upon system boot
```
sudo systemctl enable kubelet.service containerd.service
```

### Bootstrapping a Cluster with kubeadm (on the control plane node)
* create a cluster by invoking `kubeadm init`.  This performs the following:
1. validation occurs
  * RAM
  * compatible container runtime check and that it's running
2. creates a CA (certs are used for encryption and authentication)
3. generates `kubeconfig` files
4. generates static pob manifests
5. wait for the control plane pods to start
6. taints the control plane node
  * this will cause the control plane node to never schedule user pods on the control plane node
7. generates a bootstrap taken
8. starts add-on components: DNS and kube-proxy

### Understanding the Certificate Authority's Role in Your Cluster
* kubeadm init creates a self signed CA
* you can tell kubeadm to integrate into an external PKI
* CA is used: /etc/kubernetes/pki
  * to secure cluster comms throughout cluster
    * used for API Server comms
  * to authenticate users and cluster components (nodes, etc)
* The CA certs are distributed to each node

### kubeadm Created kubeconfig Files and Static Pod Manifests

#### kubeconfig files
* a `kubeconfig` file defines how to connect to the cluster
  * includes:
    * client certs
    * CA certs
    * cluster API server network location
* `kubeadm` creates various `kubeconfig` files that are used by the control plane node and worker nodes within `/etc/kubernetes`
  * `admin.conf` (kubernetes-admin): is the admin account/superuser
  * `kubelet.conf`: used to help kubelet to locate the API server and provide auth cert
  * `controller-manager.conf`: used to help controller manager to locate the API server and provide auth cert
  * `scheduler.conf`: : used to help scheduler to locate the API server and provide auth cert

#### static pod manifests
* manifest describes a config of a pod
* generated by `kubeadm init`
  * produces files in `/etc/kubernetes/manifests`
* core control plan components:
  * etcd
  * api server
  * controller manager
  * scheduler
* `kubelet` watches the directory `/etc/kubernetes/manifests` for changes to the config

### Pod Networking Fundamentals

![](2023-03-30-06-29-30.png)

* overlay network options
  * Flannel: layer 3 virtual network
  * Calico: layer 3 and policy based traffic management
  * Weave Net: multi host network

### Creating a Cluster Control Plane Node and Adding a Node

#### create a cluster control plane node, admin user, and overlay network
* download yaml manifest that describes the pod overlay network
```
wget https://docs.projectcalico.org/manifests/calico.yaml
```
* create a cluster config file
```
kubeadm config print init-defaults | tee ClusterConfigurations.yaml
```
* init the cluster
```
sudo kubeadm init --config=Clusterconfiguration.yaml --cri-socket /run/containerd/containerd.sock
```
  * once this command has exited, all control plane pods will be up and running
  * this command will also output:
    * commands to have workernodes join the cluster
    * how to execute kubeadm to create an admin user

##### creating a cluster admin user
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -d) $HOME/.kube/config
```
* inside of this kubeconfig file will contain network info on API server as well as certs used for authentication

##### deploy the pod network
```
kubectl apply -f calico.yaml
# this will push the overlay network related pods into the cluster
```

#### adding a node to a cluster
1. install packages
2. `kubeadm join`
  * takes additional parameters: bootstrap token, CA cert hash, and the location of the API server
3. downloads cluster info
4. node submits a CSR to the API server (used for kubelet to auth to API server)
5. CA signs the CSR automatically
  * `kubeadm join` downloads the cert and stores in `/var/lib/kubelet/pki`
6. creates `/etc/kubernetes/kubelet.conf`
  * contains client auth cert
  * contains API server info 
* invocation is: (note that `kubeadm init` you ran earlier produces this command)
```
kubeadm join APISERVER:6443 --token [token] --discovery-token-ca-cert-hash sha256:[hash]
```

### Demo: Creating a Cluster Control Plane Node
1. create the cluster

```
# https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-50-nodes-or-less
wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml

#if you need to do this later: https://stackoverflow.com/a/60185268/843000
#if needed, findCALICO_IPV4POOL_CIDR and modify this range to something outside of the ranges used by any infrastructure (in our case there is an overlap)
vim calico.yaml

#create a kubeconfig file
kubeadm config print init-defaults | tee ClusterConfiguration.yaml
vim ClusterConfiguration.yaml
#modify localAPIEndpoint/advertiseAddress to the control plane node's address (172.16.94.10)
#modify nodeRegistration/criSocket to the containerd socket (/run/containerd/containerd.sock)
#modify kubernetesVersion to match the current version  (v1.20.1)
#set the cgroupDriver to systemd (matching containerd)
cat <<EOF | cat >> ClusterConfiguration.yaml
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfigutation
cgroupDriver: systemd
EOF

#build the cluster!
sudo kubeadm init --config=ClusterConfiguration.yaml --cri-socket /run/containerd/containerd.sock

[init] Using Kubernetes version: v1.20.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local ubuntucontrol] and IPs [10.96.0.1 172.16.94.10]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost ubuntucontrol] and IPs [172.16.94.10 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost ubuntucontrol] and IPs [172.16.94.10 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 14.502704 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node ubuntucontrol as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
[mark-control-plane] Marking the node ubuntucontrol as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: abcdef.0123456789abcdef
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.94.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:3822f8596a80bbde636e3c60a2e96bb70c2314163d7d2c5180a314399ebe4769
```

2. create admin credentials and add completion
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null #not working right now, it's okay
```

3. create a pod network
```
kubectl apply -f calico.yaml
#note that this **WILL** fail, due to the fact that the k8s apiVersion supported in the latest calico release doesn't support k8s v1.20, but v1.21, or v1.25 (I'm reading both/either may be the case)
  # https://github.com/projectcalico/calico/issues/6132#issuecomment-1134776125
  # https://github.com/kubernetes-sigs/metrics-server/issues/1104
#regardless, we will TRY to get this moving by adjusting the value from `apiVersion: policy/v1` to `apiVersion: policy/v1beta1` in ./calico.yaml... annddd... success:
att@ubuntucontrol:~$ kubectl apply -f calico.yaml
poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers unchanged
serviceaccount/calico-node unchanged
configmap/calico-config unchanged
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org configured
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrole.rbac.authorization.k8s.io/calico-node unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-node unchanged
daemonset.apps/calico-node configured
deployment.apps/calico-kube-controllers unchanged
```

4. review all pods and nodes that were created to support the deployed services (SDN/overlay network, control plane services)
```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5bb7768754-92rf9   1/1     Running   0          12m #this is calico related
kube-system   calico-node-qw5pf                          0/1     Running   0          12m #this is calico related
kube-system   coredns-74ff55c5b-qbg9s                    1/1     Running   0          54m #this is control plane add on (dns)
kube-system   coredns-74ff55c5b-xbtsh                    1/1     Running   0          54m #this is control plane add on (dns)
kube-system   etcd-ubuntucontrol                         1/1     Running   0          54m #this is control plane (etcd)
kube-system   kube-apiserver-ubuntucontrol               1/1     Running   0          54m #this is control plane (api server)
kube-system   kube-controller-manager-ubuntucontrol      1/1     Running   0          54m #this is control plane (controller manager)
kube-system   kube-proxy-sqbm5                           1/1     Running   0          54m #this is related to local node functionality (kubeproxy, remember this implements service networking ON THIS NODE)
kube-system   kube-scheduler-ubuntucontrol               1/1     Running   0          54m #this is control plane (scheduler)

$ kubectl get nodes
NAME            STATUS   ROLES                  AGE   VERSION
ubuntucontrol   Ready    control-plane,master   64m   v1.20.1
```

5. review systemd units
* remember kubelet.service systemd unit was erroring earlier because there wasn't a cluster... it should be fine now
```
$ sudo systemctl status kubelet.service
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Thu 2023-03-30 13:55:56 UTC; 1h 6min ago
       Docs: https://kubernetes.io/docs/home/
   Main PID: 4093 (kubelet)
      Tasks: 14 (limit: 4531)
     Memory: 40.8M
        CPU: 1min 50.619s
     CGroup: /system.slice/kubelet.service
             └─4093 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock
```

6. review the static pod manifests
* static pod manifests describe what's needed to have the pod start up on the node
```
$ ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
$ sudo more /etc/kubernetes/manifests/etcd.yaml
$ sudo more /etc/kubernetes/manifests/kube-apiserver.yaml
$ sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | egrep cert
```

7. review kubeconfig files for all pods on this node
* remember kubeconfig files contain cluster info (api server connection info, cluster authentication information) for the node
```
$ ls /etc/kubernetes
```

### Demo: Adding a Node to Your Cluster
1. build out the baseline config for the workernode1 VM
```
#disable swap
sudo sed -i '/ swap / s/^/#/' /etc/fstab
sudo swapoff -a

#containerd prereqs
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
# affect at runtime
sudo modprobe overlay
sudo modprobe br_netfilter

#k8s prereqs
cat <<EOF | sudo tee /etc/sysctl.d/00-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
# affect at runtime
sudo sysctl --system

sudo apt update -y

#install and configure containerd
echo 'Acquire::Retries "10";' > /etc/apt/apt.conf.d/80-retries
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
#set the cgroup driver to systemd in /etc/containerd/config.toml
$below `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]` find and change the following value:
SystemdCgroup = true
sudo systemctl restart containerd
```

2. install kubernetes packages

```
# install kubernetes
# https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo bash -c 'cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io kubernetes-xenial main
EOF'

sudo apt update

#pin to a specific version during install
VERSION=1.20.1-00
sudo apt install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION

#disable upgrading of these packages by apt so that you can control what versions you're using:
sudo apt-mark hold kubelet kubeadm kubectl containerd

#validate containerd is running
sudo systemctl status containerd.service
#enable at boot
sudo systemctl enable kubelet.service containerd.service
```

3. Get the cluster auth token and cert hash from the control plane node so that the worker node can join the cluster
* you need the bootstrap token and the CA cert hash
```
#go back to control node
#review bootstrap tokens (if one exists, it's fine... one the node joins the cluster, it will automatically rcv a new token when the old token expires)
kubeadm token list
#create a new token for fun
kubeadm token create
lqn8hs.yrgmidj9vrzozkxl

#get the cert hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.*\s//'
3822f8596a80bbde636e3c60a2e96bb70c2314163d7d2c5180a314399ebe4769

#or just freggin do one of these little guys:
$ kubeadm token create --print-join-command
kubeadm join 172.16.94.10:6443 --token lk42bq.p2599stsnx88zdi8 --discovery-token-ca-cert-hash sha256:3822f8596a80bbde636e3c60a2e96bb70c2314163d7d2c5180a314399ebe4769
```

4. join the worker node to the cluster:
* join, which will immediately trigger 
```
#on the worker node
sudo kubeadm join 172.16.94.10:6443 --token lk42bq.p2599stsnx88zdi8 --discovery-token-ca-cert-hash sha256:3822f8596a80bbde636e3c60a2e96bb70c2314163d7d2c5180a314399ebe4769
```
* on control plane node, take a look at the nodes and note the status of workernode.  The pods that are necessary for worker node functionality and communication will immediately be deployed the new node (kubeproxy, calico, etc)
```
ubuntucontrol:~$ kubectl get nodes
NAME                STATUS     ROLES                  AGE    VERSION
ubuntucontrol       Ready      control-plane,master   154m   v1.20.1
ubuntuworkernode1   NotReady   <none>                 22s    v1.20.1

#observe the deployment progress
ubuntucontrol:~$ kubectl get pods --all-namespaces --watch

# then check to verify worker node is "Ready"
ubuntucontrol:~$ kubectl get nodes
NAME                STATUS   ROLES                  AGE     VERSION
ubuntucontrol       Ready    control-plane,master   158m    v1.20.1
ubuntuworkernode1   Ready    <none>                 4m34s   v1.20.**1**
```

5. repeat for worker nodes 2 and 3.
* i had a bad time due to probably my ZTNA client messing around with my internet connectivity, so I was seeing 
```
ubuntucontrol:~$ kubectl get pods --all-namespaces --watch
NAMESPACE     NAME                                       READY   STATUS                  RESTARTS   AGE
kube-system   calico-kube-controllers-5bb7768754-92rf9   1/1     Running                 0          140m
kube-system   calico-node-5x8wq                          0/1     Running                 0          29m
kube-system   calico-node-mbdvm                          0/1     Running                 0          16m
kube-system   calico-node-qw5pf                          0/1     Running                 0          140m
kube-system   calico-node-zn6w6                          0/1     Init:ImagePullBackOff   0          12m
kube-system   coredns-74ff55c5b-qbg9s                    1/1     Running                 0          3h2m
kube-system   coredns-74ff55c5b-xbtsh                    1/1     Running                 0          3h2m
kube-system   etcd-ubuntucontrol                         1/1     Running                 0          3h2m
kube-system   kube-apiserver-ubuntucontrol               1/1     Running                 0          3h2m
kube-system   kube-controller-manager-ubuntucontrol      1/1     Running                 0          3h2m
kube-system   kube-proxy-2fkn2                           1/1     Running                 0          16m
kube-system   kube-proxy-gnhpm                           1/1     Running                 0          12m
kube-system   kube-proxy-sqbm5                           1/1     Running                 0          3h2m
kube-system   kube-proxy-tkjg8                           1/1     Running                 0          29m
kube-system   kube-scheduler-ubuntucontrol               1/1     Running                 0          3h2m
```
* this error is discussed well here: https://www.tutorialworks.com/kubernetes-imagepullbackoff/
  * again, I'm convinced it's the ZTNA client causing the VM on my machine to fail
* here's some tshooting
```
kubectl get pod --all-namespaces
kubectl get daemonset --all-namespaces
kubectl describe daemonset calico-node -n kube-system
#interesting things here:
containers/*/image: docker.io/calico/node:v3.25.0

#determine the failing node
kubectl get nodes

#on the target node take a look at the kubelet service unit's output
journalctl -b -f -u kubelet.service

Mar 30 17:11:23 ubuntuworkernode3 kubelet[24358]: E0330 17:11:23.358875   24358 pod_workers.go:191] Error syncing pod ad4a80eb-f93d-4f24-af67-bc46dd1d183c ("calico-node-zn6w6_kube-system(ad4a80eb-f93d-4f24-af67-bc46dd1d183c)"), skipping: failed to "StartContainer" for "upgrade-ipam" with ImagePullBackOff: "Back-off pulling image \"docker.io/calico/cni:v3.25.0\""
Mar 30 17:11:24 ubuntuworkernode3 kubelet[24358]: E0330 17:11:24.738627   24358 kubelet.go:2160] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized

# let's smack kubelet around on the failing node
sudo systemctl restart kubelet.service && journalctl -b -f -u kubelet.service
#this worked...
# note to self of probable causes: DNS, network/IP connectivity, "things not coming up in the correct order"
```

### Managed Cloud Deployment Scenarios: AKS, EKS, and GKE
* AWS: Elastic Kubernetes Service (EKS)
* Google Kubernetes Engine (GKE)
* Azure Kubernetes Service (AKS)

### Demo: Creating a Cluster in the Cloud with Azure Kubernetes Service

1. install azure cli tools and auth
```
Invoke-WebRequest -Uri https://aka.ms/installazurecliwindows -OutFile .\AzureCLI.msi
Start-Process msiexec.exe -Wait -ArgumentList '/I AzureCLI.msi /quiet'
rm .\AzureCLI.msi
az login --use-device-code
#you might need to move $env:userprofile/.azure to $env:userprofile/_orig.azure
```

2. build resource group for AKS resources
```
az group create --name "kubernetes-cloud" --location centralus
```

3. list aks versions
```
az aks get-versions --location centralus -o table
```

4. create a cluster
```
az aks create --resource-group "kubernetes-cloud" --generate-ssh-keys --name cscluster --node-count 3
```
* if you need kubectl, you can run `az aks install-cli`
  
5. get creds from AKS cluster and add them to `~/.kube/config`
```
az aks get-credentials --resource-group "kubernetes-cloud" --name cscluster
```

6. list contexts from `~/.kube/config` and change context
```
kubectl config get-contexts
kubectl config use-context cscluster
```

7. delete the AKS cluster
```
az aks delete --resource-group "kubernetes-cloud" --name cscluster --yes --no-wait
```

## Working with Your Kubernetes Cluster

* introducing and using kubernetes
* a closer look at `kubectl`
* demo: using `kubectl`: nodes, pods, api reosurces, bash auto completion
* app and pod deployment (and working with yaml manifests)
* demo: imperative deployments (and workign with resources in your cluster)
* demo: exposing and accessing services in your cluster
* demo: declaritive deploymentsa and acessing and mofiying existing resources in your cluster

### using `kubectl`
* primary CLI tool for controlling workloads (how to communicate with the API server)
  * used for operations (CRUD, etc) (perform an "action", aka verbs)
  * affect resources (affected object, aka nouns)
  * returns output (gets metadata)

#### operations:
* `apply`/`create`: create resources and sending deployments
* `run`: start a pod from an image (a pod not managed by a controller)
* `explain`: documentation for resources (shows api objects)
* `delete`: delete resources
* `get`: list resources
* `describe`: detailed resource info (good for tshooting)
* `exec`: execute a command on a container inside a pod (similar to `docker exec`)
* `logs`: view logs to stdout from a container running inside a pod

#### resources:
* `nodes` (`no`)
* `pods` (`po`)
* `services` (`svc`)
* and more

#### output:
* modidying output
* specify formats:
  * `wide`: output additional info
  * `yaml`: outputs YAML
  * `json`: outputs JSON
  * `dry-run`: prints an object without sending it to the API Server.  Good for creating resources.

### a closer look at `kubectl`
* https://kubernetes.io/docs/reference/kubectl/cheatsheet/
* `kubectl get pods [pod1] --output=yaml`
* `kubectl create deployment nginx --image=nginx`

### demo: using kubectl with nodes, pods, and api resources, configure bash auto-completion
1. get clsuter info
```
kubectl cluster-info
```

2. get the nodes
```
kubectl get nodes
kubectl get nodes -o wide
kubectl get nodes -o yaml
```

3. get the pods
```
kubectl get pods
kubectl get pods -A

$ kubectl get pods -A -o wide
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE     IP               NODE                NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-5bb7768754-92rf9   1/1     Running   0          4h59m   192.168.246.65   ubuntucontrol       <none>           <none>
kube-system   calico-node-5x8wq                          0/1     Running   0          3h7m    172.16.94.11     ubuntuworkernode1   <none>           <none>
kube-system   calico-node-mbdvm                          0/1     Running   0          175m    172.16.94.12     ubuntuworkernode2   <none>           <none>
kube-system   calico-node-qw5pf                          0/1     Running   0          4h59m   172.16.94.10     ubuntucontrol       <none>           <none>
kube-system   calico-node-zn6w6                          0/1     Running   0          171m    172.16.94.13     ubuntuworkernode3   <none>           <none>
kube-system   coredns-74ff55c5b-qbg9s                    1/1     Running   0          5h41m   192.168.246.67   ubuntucontrol       <none>           <none>
kube-system   coredns-74ff55c5b-xbtsh                    1/1     Running   0          5h41m   192.168.246.66   ubuntucontrol       <none>           <none>
kube-system   etcd-ubuntucontrol                         1/1     Running   0          5h41m   172.16.94.10     ubuntucontrol       <none>           <none>
kube-system   kube-apiserver-ubuntucontrol               1/1     Running   0          5h41m   172.16.94.10     ubuntucontrol       <none>           <none>
kube-system   kube-controller-manager-ubuntucontrol      1/1     Running   0          5h41m   172.16.94.10     ubuntucontrol       <none>           <none>
kube-system   kube-proxy-2fkn2                           1/1     Running   0          175m    172.16.94.12     ubuntuworkernode2   <none>           <none>
kube-system   kube-proxy-gnhpm                           1/1     Running   0          171m    172.16.94.13     ubuntuworkernode3   <none>           <none>
kube-system   kube-proxy-sqbm5                           1/1     Running   0          5h41m   172.16.94.10     ubuntucontrol       <none>           <none>
kube-system   kube-proxy-tkjg8                           1/1     Running   0          3h7m    172.16.94.11     ubuntuworkernode1   <none>           <none>
kube-system   kube-scheduler-ubuntucontrol               1/1     Running   0          5h41m   172.16.94.10     ubuntucontrol       <none>           <none>
```
* note that each node has kube-proxy and for comms with the "real IP" versus the calico SDN IPs.

4. get all info
```
$ kubectl get all -A
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   pod/calico-kube-controllers-5bb7768754-92rf9   1/1     Running   0          5h8m
kube-system   pod/calico-node-5x8wq                          0/1     Running   0          3h16m
kube-system   pod/calico-node-mbdvm                          0/1     Running   0          3h4m
kube-system   pod/calico-node-qw5pf                          0/1     Running   0          5h8m
kube-system   pod/calico-node-zn6w6                          0/1     Running   0          179m
kube-system   pod/coredns-74ff55c5b-qbg9s                    1/1     Running   0          5h50m
kube-system   pod/coredns-74ff55c5b-xbtsh                    1/1     Running   0          5h50m
kube-system   pod/etcd-ubuntucontrol                         1/1     Running   0          5h50m
kube-system   pod/kube-apiserver-ubuntucontrol               1/1     Running   0          5h50m
kube-system   pod/kube-controller-manager-ubuntucontrol      1/1     Running   0          5h50m
kube-system   pod/kube-proxy-2fkn2                           1/1     Running   0          3h4m
kube-system   pod/kube-proxy-gnhpm                           1/1     Running   0          179m
kube-system   pod/kube-proxy-sqbm5                           1/1     Running   0          5h50m
kube-system   pod/kube-proxy-tkjg8                           1/1     Running   0          3h16m
kube-system   pod/kube-scheduler-ubuntucontrol               1/1     Running   0          5h50m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  5h50m
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   5h50m

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/calico-node   4         4         0       4            0           kubernetes.io/os=linux   5h8m
kube-system   daemonset.apps/kube-proxy    4         4         4       4            4           kubernetes.io/os=linux   5h50m

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           5h8m
kube-system   deployment.apps/coredns                   2/2     2            2           5h50m

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/calico-kube-controllers-5bb7768754   1         1         1       5h8m
kube-system   replicaset.apps/coredns-74ff55c5b                    2         2         2       5h50m
```

5. get all api objects that are available in k8s
```
$ kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
podtemplates                                   v1                                     true         PodTemplate
replicationcontrollers            rc           v1                                     true         ReplicationController
resourcequotas                    quota        v1                                     true         ResourceQuota
secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount
services                          svc          v1                                     true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io/v1              false        APIService
controllerrevisions                            apps/v1                                true         ControllerRevision
daemonsets                        ds           apps/v1                                true         DaemonSet
deployments                       deploy       apps/v1                                true         Deployment
replicasets                       rs           apps/v1                                true         ReplicaSet
statefulsets                      sts          apps/v1                                true         StatefulSet
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io/v1                true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling/v1                         true         HorizontalPodAutoscaler
cronjobs                          cj           batch/v1beta1                          true         CronJob
jobs                                           batch/v1                               true         Job
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
leases                                         coordination.k8s.io/v1                 true         Lease
bgpconfigurations                              crd.projectcalico.org/v1               false        BGPConfiguration
bgppeers                                       crd.projectcalico.org/v1               false        BGPPeer
blockaffinities                                crd.projectcalico.org/v1               false        BlockAffinity
caliconodestatuses                             crd.projectcalico.org/v1               false        CalicoNodeStatus
clusterinformations                            crd.projectcalico.org/v1               false        ClusterInformation
felixconfigurations                            crd.projectcalico.org/v1               false        FelixConfiguration
globalnetworkpolicies                          crd.projectcalico.org/v1               false        GlobalNetworkPolicy
globalnetworksets                              crd.projectcalico.org/v1               false        GlobalNetworkSet
hostendpoints                                  crd.projectcalico.org/v1               false        HostEndpoint
ipamblocks                                     crd.projectcalico.org/v1               false        IPAMBlock
ipamconfigs                                    crd.projectcalico.org/v1               false        IPAMConfig
ipamhandles                                    crd.projectcalico.org/v1               false        IPAMHandle
ippools                                        crd.projectcalico.org/v1               false        IPPool
ipreservations                                 crd.projectcalico.org/v1               false        IPReservation
kubecontrollersconfigurations                  crd.projectcalico.org/v1               false        KubeControllersConfiguration
networkpolicies                                crd.projectcalico.org/v1               true         NetworkPolicy
networksets                                    crd.projectcalico.org/v1               true         NetworkSet
endpointslices                                 discovery.k8s.io/v1beta1               true         EndpointSlice
events                            ev           events.k8s.io/v1                       true         Event
ingresses                         ing          extensions/v1beta1                     true         Ingress
flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta1   false        FlowSchema
prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta1   false        PriorityLevelConfiguration
ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
ingresses                         ing          networking.k8s.io/v1                   true         Ingress
networkpolicies                   netpol       networking.k8s.io/v1                   true         NetworkPolicy
runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
poddisruptionbudgets              pdb          policy/v1beta1                         true         PodDisruptionBudget
podsecuritypolicies               psp          policy/v1beta1                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io/v1           true         RoleBinding
roles                                          rbac.authorization.k8s.io/v1           true         Role
priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
```
* "NAMESPACED" is if the resource can be within a namespace

6. learn more about resources (useful while building yamls)
```
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod --recursive
```

7. describe will give you a lot of info on resources
```
$ kubectl describe nodes ubuntucontrol
Name:               ubuntucontrol
Roles:              control-plane,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ubuntucontrol
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node-role.kubernetes.io/master=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 172.16.94.10/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 192.168.246.64
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 30 Mar 2023 13:55:53 +0000
Taints:             node-role.kubernetes.io/master:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  ubuntucontrol
  AcquireTime:     <unset>
  RenewTime:       Thu, 30 Mar 2023 19:52:19 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 30 Mar 2023 14:38:38 +0000   Thu, 30 Mar 2023 14:38:38 +0000   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Thu, 30 Mar 2023 19:50:18 +0000   Thu, 30 Mar 2023 13:55:49 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 30 Mar 2023 19:50:18 +0000   Thu, 30 Mar 2023 13:55:49 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 30 Mar 2023 19:50:18 +0000   Thu, 30 Mar 2023 13:55:49 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 30 Mar 2023 19:50:18 +0000   Thu, 30 Mar 2023 14:38:25 +0000   KubeletReady                 kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:  172.16.94.10
  Hostname:    ubuntucontrol
Capacity:
  cpu:                2
  ephemeral-storage:  19430032Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3983176Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  17906717462
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3880776Ki
  pods:               110
System Info:
  Machine ID:                 f3ac3a329eba47d4886ec66d88244590
  System UUID:                26cb4d56-9c4a-3cd7-42e5-4f513cc7df13
  Boot ID:                    8cf1045f-a9fe-48e2-8b5e-5190450d339d
  Kernel Version:             5.15.0-69-generic
  OS Image:                   Ubuntu 22.04.2 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.12
  Kubelet Version:            v1.20.1
  Kube-Proxy Version:         v1.20.1
Non-terminated Pods:          (9 in total)
  Namespace                   Name                                        CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                                        ------------  ----------  ---------------  -------------  ---
  kube-system                 calico-kube-controllers-5bb7768754-92rf9    0 (0%)        0 (0%)      0 (0%)           0 (0%)         5h14m
  kube-system                 calico-node-qw5pf                           250m (12%)    0 (0%)      0 (0%)           0 (0%)         5h14m
  kube-system                 coredns-74ff55c5b-qbg9s                     100m (5%)     0 (0%)      70Mi (1%)        170Mi (4%)     5h56m
  kube-system                 coredns-74ff55c5b-xbtsh                     100m (5%)     0 (0%)      70Mi (1%)        170Mi (4%)     5h56m
  kube-system                 etcd-ubuntucontrol                          100m (5%)     0 (0%)      100Mi (2%)       0 (0%)         5h56m
  kube-system                 kube-apiserver-ubuntucontrol                250m (12%)    0 (0%)      0 (0%)           0 (0%)         5h56m
  kube-system                 kube-controller-manager-ubuntucontrol       200m (10%)    0 (0%)      0 (0%)           0 (0%)         5h56m
  kube-system                 kube-proxy-sqbm5                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         5h56m
  kube-system                 kube-scheduler-ubuntucontrol                100m (5%)     0 (0%)      0 (0%)           0 (0%)         5h56m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                1100m (55%)  0 (0%)
  memory             240Mi (6%)   340Mi (8%)
  ephemeral-storage  100Mi (0%)   0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)
Events:              <none>
```

8. help, which contains examples
```
kubectl -h
kubectl get -h
kubectl create -h
```

9. bash completion

```
sudo apt -y install bash-completion
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

### application deployment in k8s
* imperative configuration: generally considered single-stream configuration method
```
# creates a pod nginx with image nginx
kubectl create deployment nginx --image=nginx

#creates a pod for nginx
kubectl run nginz --image=nginx
```
* declaratively: define desired state in code
  * manifest: these contain descriptions of nodes
    * can be yaml or json
```
#deployment.yaml will contain yaml that defines the resources that should be spun up on the cluster
kubectl apply -f deployment.yaml
```

#### basic manifest: deployment
1. create a minimum/basic manifest yaml
* remember to use the `kubectl explain` and `dry-run` to work on building manifests
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
template:
  metadata:
    labels:
      app: hello-world
  spec:
    containers:
    - image: gcr.io/google-samples/hello-app:1.0
      name: hello-app
```
  * api resource versions allow for stability any time the manifest is executed, given that API versions are maintained historically.
  * `apps` is used for deployments
  * `kind`: defines
  * `spec/selector` labels match template label metadata

2. deploy this example to the cluster
```
kubectl apply -f deployment.yaml
```

#### generate manifests with `dry-run`
* you can use `--dry-run=client -o yaml` to produce a yaml manifest for an action at the command line
  * this will allow you to easily create well-formed yaml from specific invocations
```
$ kubectl create deployment hello-world --image=gcr.io/google-samples/hello-app:1.0 --dry-run=client -o yaml > deployment_dryrun.yaml

matt@ubuntucontrol:~$ diff deployment.yaml deployment_dryrun.yaml  -y
apiVersion: apps/v1                                             apiVersion: apps/v1
kind: Deployment                                                kind: Deployment
metadata:                                                       metadata:
                                                              >   creationTimestamp: null
                                                              >   labels:
                                                              >     app: hello-world
  name: hello-world                                               name: hello-world
spec:                                                           spec:
  replicas: 1                                                     replicas: 1
  selector:                                                       selector:
    matchLabels:                                                    matchLabels:
      app: hello-world                                                app: hello-world
template:                                                     |   strategy: {}
  metadata:                                                   |   template:
    labels:                                                   |     metadata:
      app: hello-world                                        |       creationTimestamp: null
  spec:                                                       |       labels:
    containers:                                               |         app: hello-world
    - image: gcr.io/google-samples/hello-app:1.0              |     spec:
      name: hello-app                                         |       containers:
                                                              >       - image: gcr.io/google-samples/hello-app:1.0
                                                              >         name: hello-app
                                                              >         resources: {}
                                                              > status: {}
```

#### application deployment process

![](2023-03-30-16-56-53.png)

* what the API server is doing when `kubectl create deployment` is issued
1. admin issues `kubectl apply -f deployment.yaml`
2. deployment will creates a `replicaset`
3. `replicaset` will create `pods`, based on the `deployment.yaml`
4. `API server` will parse manifest yaml and store those objects in `etcd`
5. the `controller manager` is watching the `etcd` for any new interesting objects
6. if recognized as interesting (like a deployment), the `controller manager` will create a `controller` that will create a `replicaset`
7. the `replicaset` is going to create the required number of `pods` and write `pod` info to `etcd`
8. `scheduler` is watching `etcd` for `pod` info and validates that all pods that need/should be scheduled, the `scheduler` will schedule the `pods` to run on the `nodes`.
9. `scheduler` then updates `pod `info in `etcd` specifying the `node` where it is running
10. note that at this point in time `pods` haven't yet started on the `nodes`, images aren't yet pulled from the repo, etc.  `nodes` aren't aware yet that there are  published `pods` to be executed.... until...
11. on each `node`, `kubelet` is continuously watching `etcd` contents (via calls to the `api server`), and the `api server` reply will contain "yes, you have a pod that's to be scheduled"
12. the `node` resident `kubelet` signals to the `node` resident `container runtime` to pull down the image as per manifest `spec/container/image`, and start the `pod`.
13. if the `pod` is a member of a `service`, then once the `pod` is running, `container runtime` updates the `node` resident `kube-proxy`.