![RX-M, llc.](http://rx-m.io/rxm-cnc.svg)


# Kubernetes Networking 101

Welcome to the Kubernetes Networking 101 lab!!

This lab will walk you through six hands on exploratory steps, each step will give you a look at a different aspect of
Kubernetes networking. The steps are designed to take about 10 minutes but make a good jumping off point for hours of
further exploration.

The lab systems provided for people joining in-person are 2CPU/4GB/30GB Ubuntu 20.04 cloud instances. If you are virtual
or running the labs after the tutorial session, the lab should  work fine on any properly configured Kubernetes cluster,
though you will need admin access to install all of the tools used. If you would like to run a local Ubuntu 20.04 VM on
your laptop to complete the lab, you can find instructions for doing so here:
https://github.com/RX-M/classfiles/blob/master/lab-setup.md

Let the networking begin!!!


## 1. Pod Networking

To begin our Kubernetes networking journey we'll need to setup a Kubernetes cluster to work with. We can quickly stand
up a Kubernetes cluster on a plain vanilla Ubuntu server using the rx-m `k8s-no-cni.sh` shell script.

Let's do it!


### SSH to your lab system

When you arrived you received a "login creds" sheet for your private lab machine with the following info:

- Lab machine IP
- Lab key URL

To ssh to your machine you will need the IP, key and a username, which is "ubuntu".

Download the key file and make it private:

```
$ wget <YOUR KEY URL HERE>  net.pem

$ chmod 400 net.pem
```

Now log in to your assigned cloud instance with ssh:

> N.B. ssh instructions for mac/windows/linux are here if you need them:
>     https://github.com/RX-M/classfiles/blob/master/ssh-setup.md):

```
$ ssh -i net.pem ubuntu@<YOUR LAB MACHINE IP HERE>

The authenticity of host 'x.x.x.x (x.x.x.x)' can't be established.
ECDSA key fingerprint is SHA256:avCAN9BTeFbPGaZl2Ao+j7NBE89oGNaSYU1fL5FBHbY.

Are you sure you want to continue connecting (yes/no)? yes

Warning: Permanently added 'x.x.x.x' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.13.0-1022-aws x86_64)

...

Last login: Tue May 17 09:46:05 2022 from 172.58.27.10

ubuntu@ip-172-31-24-84:~$
```


### Install Kubernetes

You're in! You can poke around if you like (who, id, free, ps, whathaveyou) but we are on the clock, so the next step is
to stand up a Kubernetes cluster. Run the RX-M K8s install script as follows:

> N.B. This will take a minute or two.

```
ubuntu@ip-172-31-24-84:~$ curl https://raw.githubusercontent.com/RX-M/classfiles/master/k8s-no-cni.sh | sh


  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3446  100  3446    0     0  19691      0 --:--:-- --:--:-- --:--:-- 19691
# Executing docker install script, commit: 614d05e0e669a0577500d055677bb6f71e822356

...

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.24.84:6443 --token e29j76.vgcjt3mkfpe6s50h \
        --discovery-token-ca-cert-hash sha256:4b71b7abad35eedf9846be61f513d329a1783cd840f4ebddaf536b411b3ce91e
node/ip-172-31-24-84 untainted
node/ip-172-31-24-84 untainted

ubuntu@ip-172-31-24-84:~$
```

> N.B. Do not run any of the commands suggested in the output above.

Boom. K8s up. So what just happened? Well you can take a look at the script if you like, but this command just installed
the latest version of Kubernetes (1.24 at the time of this writing), using docker with the dockerd-shim as the CRI. This
is a one node cluster, so the node is configured to perform the control plane tasks of a control plane node, but
untainted so that we can also use it as a worker node to run pods.

Take a look at your cluster by getting the nodes:

```
ubuntu@ip-172-31-24-84:~$ kubectl get nodes

NAME              STATUS     ROLES           AGE   VERSION
ip-172-31-24-84   NotReady   control-plane   10s   v1.24.0

ubuntu@ip-172-31-24-84:~$
```

Hey, why is our node "NotReady"?!

During the Kubernetes install you may have seen the following statement 7 or 8 lines from the end of the output:

"You should now deploy a pod network to the cluster."

Kubernetes is not opinionated, it let's you choose your own CNI solution. Until a CNI plugin is installed our cluster
will be inoperable. Time to install Cilium!


### Install Cilium

We're going to use Cilium as our CNI networking solution. Cilium is an incubating CNCF project that implements a wide
range of networking, security and observability features, largely through the Linux kernel eBPF facility. This makes
Cilium fast and resource efficient.

Cilium offers a command line tool that we can use to install the CNI components. Download, extract and test the Cilium
CLI:

```
ubuntu@ip-172-31-24-84:~$ wget https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz

...

cilium-linux-amd64.tar.gz     100%[==============================================>]  21.33M  4.55MB/s    in 4.8s

2022-05-17 11:15:04 (4.47 MB/s) - ‘cilium-linux-amd64.tar.gz’ saved [22369103/22369103]

ubuntu@ip-172-31-24-84:~$ sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

cilium

ubuntu@ip-172-31-24-84:~$ cilium version

cilium-cli: v0.11.6 compiled with go1.18.2 on linux/amd64
cilium image (default): v1.11.4
cilium image (stable): v1.11.5
cilium image (running): unknown. Unable to obtain cilium version, no cilium pods found in namespace "kube-system"

ubuntu@ip-172-31-24-84:~$
```

Looks good! Now install the cilium CNI:

```
ubuntu@ip-172-31-24-84:~$ cilium install

ℹ️  using Cilium version "v1.11.4"
🔮 Auto-detected cluster name: kubernetes
🔮 Auto-detected IPAM mode: cluster-pool
ℹ️  helm template --namespace kube-system cilium cilium/cilium --version 1.11.4 --set cluster.id=0,cluster.name=kubernetes,encryption.nodeEncryption=false,ipam.mode=cluster-pool,kubeProxyReplacement=disabled,operator.replicas=1,serviceAccounts.cilium.name=cilium,serviceAccounts.operator.name=cilium-operator
ℹ️  Storing helm values file in kube-system/cilium-cli-helm-values Secret
🔑 Created CA in secret cilium-ca
🔑 Generating certificates for Hubble...
🚀 Creating Service accounts...
🚀 Creating Cluster roles...
🚀 Creating ConfigMap for Cilium version 1.11.4...
🚀 Creating Agent DaemonSet...
level=warning msg="spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[1].matchExpressions[0].key: beta.kubernetes.io/os is deprecated since v1.14; use \"kubernetes.io/os\" instead" subsys=klog
🚀 Creating Operator Deployment...
level=warning msg="spec.template.metadata.annotations[scheduler.alpha.kubernetes.io/critical-pod]: non-functional in v1.16+; use the \"priorityClassName\" field instead" subsys=klog
⌛ Waiting for Cilium to be installed and ready...
♻️  Restarting unmanaged pods...
♻️  Restarted unmanaged pod kube-system/coredns-6d4b75cb6d-2qmpv
♻️  Restarted unmanaged pod kube-system/coredns-6d4b75cb6d-xxjv7
✅ Cilium was successfully installed! Run 'cilium status' to view installation health

ubuntu@ip-172-31-24-84:~$
```

Crazy progress characters aside... Looks good! Check the cilium status:

```
ubuntu@ip-172-31-24-84:~$ cilium status

    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         disabled
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

Deployment        cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet         cilium             Desired: 1, Ready: 1/1, Available: 1/1
Containers:       cilium-operator    Running: 1
                  cilium             Running: 1
Cluster Pods:     2/2 managed by Cilium
Image versions    cilium             quay.io/cilium/cilium:v1.11.4@sha256:d9d4c7759175db31aa32eaa68274bb9355d468fbc87e23123c80052e3ed63116: 1
                  cilium-operator    quay.io/cilium/operator-generic:v1.11.4@sha256:bf75ad0dc47691a3a519b8ab148ed3a792ffa2f1e309e6efa955f30a40e95adc: 1

ubuntu@ip-172-31-24-84:~$
```

Sweet. Cilium is happy so we're happy.

In the installation we just used runs two different types of pods:

- Cilium Operator
- Cilium [the CNI plugin]

The Cilium "operator" (like a human operator but in code) manages the Cilium CNI components and supports various CLI and
control plane functions. Kubernetes "operators" run in pods typically managed by a Deployment (runs the pod[s] somewhere
in the cluster). The CNI plugin networking agents that configure pod network interfaces, generally run under a
DaemonSet, which ensures that one copy of the CNI plugin pod runs on each node. This way, when an administrator adds a
new node, the CNI agent is automatically started on the new node by the DaemonSet.

Now let take a look at the cluster:

```
ubuntu@ip-172-31-24-84:~$ kubectl get nodes

NAME              STATUS   ROLES           AGE   VERSION
ip-172-31-24-84   Ready    control-plane   91m   v1.24.0

ubuntu@ip-172-31-24-84:~$
```

Yes! Our node is now "Ready". We have a working, network enabled Kubernetes cluster.

Take a look at the pods running in the cluster:

```
ubuntu@ip-172-31-24-84:~$ kubectl get pod -A

NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   cilium-5gp4t                              1/1     Running   0          14m
kube-system   cilium-operator-6d86df4fc8-g2z66          1/1     Running   0          14m
kube-system   coredns-6d4b75cb6d-fzsbw                  1/1     Running   0          14m
kube-system   coredns-6d4b75cb6d-wvnvv                  1/1     Running   0          14m
kube-system   etcd-ip-172-31-24-84                      1/1     Running   0          105m
kube-system   kube-apiserver-ip-172-31-24-84            1/1     Running   0          105m
kube-system   kube-controller-manager-ip-172-31-24-84   1/1     Running   0          105m
kube-system   kube-proxy-929xs                          1/1     Running   0          105m
kube-system   kube-scheduler-ip-172-31-24-84            1/1     Running   0          105m

ubuntu@ip-172-31-24-84:~$
```

Note that all of the pods we have so far are part of the Kubernetes system itself, so they run in a namespace called
`kube-system`. We'll run our test pods in the `default` namespace. The `-A` switch shows pods in all namespaces.

These are the pods we have so far:

- cilium-5gp4t - the Cilium CNI plugin on our one and only node
- cilium-operator-6d86df4fc8-g2z66 - the cilium controller providing control plane functions for cilium
- coredns-6d4b75cb6d-fzsbw - the Kubernetes DNS server
- coredns-6d4b75cb6d-wvnvv - a DNS replica to ensure DNS never goes down
- etcd-ip-172-31-24-84 - the Kubernetes database used by the API server to store, well, everything
- kube-apiserver-ip-172-31-24-84 - the Kubernetes control plane API
- kube-controller-manager-ip-172-31-24-84 - manager for all of the built in controllers (Deployments, DaemonSets, etc.)
- kube-proxy-929xs - the Kubernetes service proxy, more on this guy in a bit
- kube-scheduler-ip-172-31-24-84 - the Pod scheduler, which assigns Pods to nodes in the cluster

Alright, let's look into all of this stuff!


### Explore the network

Let's think over the network environment that we have setup. We have three IP spaces:

- The Public internet: the virtual IP you sshed to is Internet routable over the public internet
- The Cloud network: the host IPs of the machines you are using in the cloud provider environment make up this network
- The Pod network: the Pod IPs used by the containers in your Kubernetes cluster make up the Pod network

Let's look at each of these networks and think about how they operate.


#### The Internet - public IP

In our case, the public IP address we use to ssh into our computer reaches a cloud gateway which is configured to
translate the public destination address to you Host IP address. This allows us to have a large number of hosts in the
cloud while using a small number of scarce public IPs to map to the few hosts that need to be exposed to the internet.
Once you have sshed into a cloud instance using a public IP, you can use that system as a "jump box" to ssh into the
hosts without public IPs.

In most clouds, you can discover a host's public IP by querying the cloud's metadata servers. Try it:

```
ubuntu@ip-172-31-24-84:~$ curl http://169.254.169.254/latest/meta-data/public-ipv4

18.185.35.194

ubuntu@ip-172-31-24-84:~$
```

In our case this public IP is 1:1 NATed (network address translated) with our Host (private) IP. In some cases, a host may receive a different outbound address (SNAT, source network address translation) when connecting out. This allows
even hosts that do not have an inbound public IP to reach out to the internet. You can check your outbound public IP
address like this:

```
ubuntu@ip-172-31-24-84:~$ curl -s checkip.dyndns.org | sed -e 's/.*Current IP Address: //' -e 's/<.*$//'

18.185.35.194

ubuntu@ip-172-31-24-84:~$
```

In our case they are the same.


#### The Cloud Network - private host IP

The host network, known as a virtual private cloud (VPC) in many cloud provider environments, uses IP addresses in
the one of the standard IANA reserved address ranges designed for local communications within a private network:

- 10.0.0.0/8
- 172.16.0.0/12
- 192.168.0.0/16

Identify your `host IP` address:

```
ubuntu@ip-172-31-24-84:~$ ip address | head

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 06:26:53:92:f4:7c brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet 172.31.24.84/20 brd 172.31.31.255 scope global dynamic ens5

ubuntu@ip-172-31-24-84:~$
```

The ens5 interface is the host's external interface. As you can see our host IP is within the 172.16.0.0/12 address
space which is not routable over the internet. Because all of the lab machines were created within the same VPC they can
reach each other within the cloud. Ask you neighbor for their private (host) IP address and try to ping it:

```
ubuntu@ip-172-31-24-84:~$ ping -c 3 172.31.24.122

PING 172.31.24.122 (172.31.24.122) 56(84) bytes of data.
64 bytes from 172.31.24.122: icmp_seq=1 ttl=63 time=0.388 ms
64 bytes from 172.31.24.122: icmp_seq=2 ttl=63 time=0.230 ms
64 bytes from 172.31.24.122: icmp_seq=3 ttl=63 time=0.232 ms

--- 172.31.24.122 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2044ms
rtt min/avg/max/mdev = 0.230/0.283/0.388/0.074 ms

ubuntu@ip-172-31-24-84:~$
```


#### The Pod Network - Pod IP

If the Internet is the outermost network in our milieu, the Pod network is the innermost. Let's create a pod an examine
it's network features.

Run an Apache Webserver container (`httpd`) in a pod on your Kubernetes cluster:

```
ubuntu@ip-172-31-24-84:~$ kubectl run web --image=httpd

pod/web created

ubuntu@ip-172-31-24-84:~$
```

Great! It may take the pod a few seconds to start. List your pods until the web pod is running:

```
ubuntu@ip-172-31-24-84:~$ kubectl get pod -o wide

NAME   READY   STATUS              RESTARTS   AGE   IP       NODE              NOMINATED NODE   READINESS GATES
web    0/1     ContainerCreating   0          8s    <none>   ip-172-31-24-84   <none>           <none>

ubuntu@ip-172-31-24-84:~$ kubectl get pod -o wide

NAME   READY   STATUS    RESTARTS   AGE   IP           NODE              NOMINATED NODE   READINESS GATES
web    1/1     Running   0          18s   10.0.0.128   ip-172-31-24-84   <none>           <none>

ubuntu@ip-172-31-24-84:~$
```

Note that our pod has a 10.x.x.x network address. This is the default range of addresses provided to pods by the cilium
CNI plugin. Check the Cilium IP Address Manager (IPAM) configuration:

```
ubuntu@ip-172-31-24-84:~$ cilium config view | grep cluster-pool

cluster-pool-ipv4-cidr                 10.0.0.0/8
cluster-pool-ipv4-mask-size            24
ipam                                   cluster-pool

ubuntu@ip-172-31-24-84:~$
```

As configured, all of our pods will be in a network with a 10.x.x.x prefix (10.0.0.0/8). Each host will have a 24 bit
subnet however (24), making it easy to determine where to route pod traffic amongst the hosts. You can also see the node
that Kubernetes scheduled the pod to, in the example above: `ip-172-31-24-84`. In a typical cluster there would be many
nodes and the Cilium network would allow all of the pods to communicate with each other, regardless of the node they run
on. This is in fact a Kubernetes requirement, all pods must be able to communicate with all other pods, though it is
possible to block undesired pod communications with network policies as we will see later.

In this default configuration, traffic between pods on the same node is propagated by the Linux kernel and traffic
between pods on different nodes uses an overlay network. This overlay network encapsulates traffic between nodes which
communicate using UDP tunnels between the hosts. Cilium supports both VXLAN and Geneve encapsulation.

Check your tunnel type:

```
ubuntu@ip-172-31-24-84:~$ cilium config view | grep tunnel

tunnel                                 vxlan

ubuntu@ip-172-31-24-84:~$
```

We can also disable tunneling in cilium, in which case the pod packets will be routed to the host network. This makes
things a little faster and more efficient but it means that you Pod network must integrate with your host network. Using
a tunneled (aka. overlay) network hides the Pod traffic within host to host communications tunnels making the Pod
network more independent, avoiding entanglement with the configuration of the cloud or bare metal network used by the
hosts.


### Test the Pod Network

To make sure that our Pod network is operating correctly we can run a test client Pod with an interactive shell that we
can run network diagnostics in. Start a Pod running the busybox container image:

```
ubuntu@ip-172-31-24-84:~$ kubectl run client -it --image=busybox

If you don't see a command prompt, try pressing enter.
/ #
```

This prompt is the prompt of a new shell running inside the `busybox` container in the `client` pod. Check the ip
address of the new Pod:

```
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9001 qdisc noqueue qlen 1000
    link/ether 42:a5:ba:f7:76:97 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.231/32 scope global eth0
       valid_lft forever preferred_lft forever
/ #
```

Note that you IP addresses will likely be different that the examples here. Try pinging the web pod from the client pod:

```
/ # ping -c 3 10.0.0.128
PING 10.0.0.128 (10.0.0.128): 56 data bytes
64 bytes from 10.0.0.128: seq=0 ttl=63 time=0.502 ms
64 bytes from 10.0.0.128: seq=1 ttl=63 time=0.202 ms
64 bytes from 10.0.0.128: seq=2 ttl=63 time=0.079 ms

--- 10.0.0.128 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.079/0.261/0.502 ms
/ #
```

Try using wget to reach the web server:

```
/ # wget -qO -  10.0.0.128

<html><body><h1>It works!</h1></body></html>

/ #
```

Success! We have a functioning Pod network, congratulations!!


### Clean up

To start our service exploration with a clean slate let's delete the web and client pods we created above. Exit out of
the client pod:

```
/ # exit
Session ended, resume using 'kubectl attach client -c client -i -t' command when the pod is running

ubuntu@ip-172-31-24-84:~$
```

> N.B. The exit shell command terminates the shell, which Kubernetes will immediately restart.

Now delete the `client` and `web` pods.

```
ubuntu@ip-172-31-24-84:~$ kubectl delete pod client web

pod "client" deleted
pod "web" deleted

ubuntu@ip-172-31-24-84:~$
```


## 2. Services

A typical pattern in microservice systems is that of replicating a stateless service many times to scale out and to
provide resilience. In Kubernetes a Deployment can be used to create a ReplicaSet which will in turn create several
copies of the same pod.  

Clients of such a replicated pod have challenges. Which pod to connect to? All of them? One of them? Which one?

Kubernetes Services provide an abstraction designed to make it easy for clients to connect to a dynamic, replicated
set of pods. The default Service type provides a virtual IP address, known as a Cluster IP, which clients can connect
to. Behind the scenes the Linux kernel forward the connection to one of the pods in the set.


### Create some Pods

Let's explore the operation of a basic service. to begin create a set of three pods running the httpd web server:


```
ubuntu@ip-172-31-24-84:~$ kubectl create deployment website --replicas=3 --image=httpd

deployment.apps/website created

ubuntu@ip-172-31-24-84:~$
```

Display the deployment and the pods it created:

```
ubuntu@ip-172-31-24-84:~$ kubectl get deploy

NAME      READY   UP-TO-DATE   AVAILABLE   AGE
website   3/3     3            3           52s

ubuntu@ip-172-31-24-84:~$ kubectl get pod --show-labels

NAME                      READY   STATUS    RESTARTS   AGE   LABELS
website-5746f499f-f967r   1/1     Running   0          61s   app=website,pod-template-hash=5746f499f
website-5746f499f-qzjjq   1/1     Running   0          61s   app=website,pod-template-hash=5746f499f
website-5746f499f-v9shz   1/1     Running   0          61s   app=website,pod-template-hash=5746f499f

ubuntu@ip-172-31-24-84:~$
```

Typically deployments are created by a Continuous Deployment component of a software pipeline. In the example above we
used `kubectl create deployment` to quickly establish a set of three httpd pods. As you can see, the pods all have the
`app=website` label. In Kubernetes, labels are arbitrary key/value pairs associated with resources. Deployments use
labels to identify the pods they own, however we can also use labels to tell a Service which pods to direct traffic to.


### Create a ClusterIP Service

Let's create a simple service to provide a stable interface to our set of pods. Create a Service manifest in yaml and
apply it to your cluster:

```
ubuntu@ip-172-31-24-84:~$ vim service.yaml

ubuntu@ip-172-31-24-84:~$ cat service.yaml

```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: website
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: website
```
```

ubuntu@ip-172-31-24-84:~$ kubectl apply -f service.yaml

service/website created

ubuntu@ip-172-31-24-84:~$
```

List your service:

```
ubuntu@ip-172-31-24-84:~$ kubectl get service

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   35h
website      ClusterIP   10.111.148.30   <none>        80/TCP    4m29s

ubuntu@ip-172-31-24-84:~$
```

> N. B. The `kubernetes` service is built in and front ends the cluster's API Server.

Our service, `website`, is of type ClusterIP and has an IP address of 10.111.148.30. Much like pod IP addresses, the
range for ClusterIPs can be defined at the time the cluster is setup. Unlike Pod IPs, which are typically assigned by
the CNI plugin, Cluster IPs are assigned by the Kubernetes control plane when the service is created. The Cluster IP
range must not overlap with any IP ranges assigned to nodes or pods.

Examine the default ClusterIP CIDR (Classless Inter-Domain Routing) address range:

```
ubuntu@ip-172-31-24-84:~$ kubectl cluster-info dump | grep -m 1 service-cluster-ip-range

                            "--service-cluster-ip-range=10.96.0.0/12",

ubuntu@ip-172-31-24-84:~$
```   

Given IPv4 addresses are 4 octets (bytes), 10.96.0.0 in binary is:

- 0000 1010
- 0110 0000
- 0000 0000
- 0000 0000

Given this is a stroke 12 (/12), the range of addresses available include anything with the first 12 bits:  

- 0000 1010
- 0110

The address received by our service is selected randomly from this range. In the example above, 10.111.148.30, in binary
is:

- 0000 1010
- 0110 1111
- 1001 0100
- 0001 1110

Note also, that our service specifies a specific port, `80`. Services can list as many port as a user may require. Port
can be listed in ranges and port can be translated by a service as well. When the target port is different from the
connecting port, the `targetPort` field can be specified. Our service simply forwards connections on port 80 to port 80
on one of the pods that match the selector.

Kubernetes creates `endpoint` resources to represent the IP addresses of Pods that have labels that match the service
selector. Endpoints are a sub-resource of the service that owns them. List the endpoints for the website service:

```
ubuntu@ip-172-31-24-84:~$ kubectl get endpoints website

NAME      ENDPOINTS                                 AGE
website   10.0.0.1:80,10.0.0.177:80,10.0.0.202:80   28m

ubuntu@ip-172-31-24-84:~$
```

The service has identified all three of the pods we created as targets. Test the service function by curling the service
ClusterIP (be sure to use the IP of the service on your machine):

```
ubuntu@ip-172-31-24-84:~$ curl 10.111.148.30

<html><body><h1>It works!</h1></body></html>

ubuntu@ip-172-31-24-84:~$
```

It works! Indeed.

Which pod did you hit? Who cares? They are replicas, it doesn't matter, that's the point!!


### Service Routing

So how does the connection get redirected? Like all good tech questions, the answer is, it depends. The default
Kubernetes implementation is to let the kube-proxy (which usually runs on every node in the cluster) modify the
iptables with DNAT rules.

Look for your service in the NAT table (again, be sure to use the IP address of your ClusterIP):

```
ubuntu@ip-172-31-24-84:~$ sudo iptables -L -vn -t nat | grep '10.111.148.30'

    1    60 KUBE-SVC-RYQJBQ5TR32XWAUN  tcp  --  *      *       0.0.0.0/0            10.111.148.30        
    /* default/website:http cluster IP */ tcp dpt:80

ubuntu@ip-172-31-24-84:~$
```

This rule says, jump to chain KUBE-SVC-RYQJBQ5TR32XWAUN when processing tcp connections heading to 10.111.148.30 on
port 80. Display the rule chain reported by your system:

```
ubuntu@ip-172-31-24-84:~$ sudo iptables -L -vn -t nat | grep -A4 'Chain KUBE-SVC-RYQJBQ5TR32XWAUN'

Chain KUBE-SVC-RYQJBQ5TR32XWAUN (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SEP-SUZPWGKL5FHWPETE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/website:http -> 10.0.0.177:80 */ statistic mode random probability 0.33333333349
    1    60 KUBE-SEP-FFKEUBR5SKHPYVCQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/website:http -> 10.0.0.1:80 */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-OPDTLPWI6RF27F2I  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/website:http -> 10.0.0.202:80 */

ubuntu@ip-172-31-24-84:~$
```

Note that the packets column (the first column titled "pkts") show one packet processed by the second rule in the
example above. That tells us which pod processed our curl request.

In the example above we see our three target pods selected at random. Since rules are evaluated sequentially and the
first matching rule is applied, the first rule is applied 33% of the time (1/3) via the condition:

`statistic mode random probability 0.33333333349`

If the random generated value is over 0.33... then there are two pods left, so the next rule hits 50% of the time and if
that rule misses then the final pod is always selected.

Examine one of the pod chains (again select a chain name for your machine's output):

```
ubuntu@ip-172-31-24-84:~$ sudo iptables -L -vn -t nat | grep -A3 'Chain KUBE-SEP-FFKEUBR5SKHPYVCQ'

Chain KUBE-SEP-FFKEUBR5SKHPYVCQ (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.0.0.1             0.0.0.0/0            /* default/website:http */
    1    60 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/website:http */ tcp to:10.0.0.1:80

ubuntu@ip-172-31-24-84:~$
```

The first rule marks the packet to avoid processing loops and the second rule DNATs (Destination network address
translates) the packet to the pod ip address and port, 10.0.0.1:80 in the above case.

Now that we know which pod was hit by our curl command, let's verify it by looking at the pod log. First look up the
name of the pod with the IP from the tables dump and then display the pod logs:

```
ubuntu@ip-172-31-24-84:~$ kubectl get pod -o wide | grep '10.0.0.1 '

website-5746f499f-qzjjq   1/1     Running   0          74m   10.0.0.1     ip-172-31-24-84   <none>           <none>

ubuntu@ip-172-31-24-84:~$ kubectl logs website-5746f499f-qzjjq

AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.0.1. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.0.1. Set the 'ServerName' directive globally to suppress this message
[Wed May 18 20:52:18.134335 2022] [mpm_event:notice] [pid 1:tid 140535473859904] AH00489: Apache/2.4.53 (Unix) configured -- resuming normal operations
[Wed May 18 20:52:18.134633 2022] [core:notice] [pid 1:tid 140535473859904] AH00094: Command line: 'httpd -D FOREGROUND'
172.31.24.84 - - [18/May/2022:21:45:00 +0000] "GET / HTTP/1.1" 200 45

ubuntu@ip-172-31-24-84:~$
```

The last entry shows our curl request: "GET / ...".

You can dump the logs of all the pods with the `app=website` label to verify that the other pods have not hits:

```
ubuntu@ip-172-31-24-84:~$ kubectl logs -l app=website

AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.0.177. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.0.177. Set the 'ServerName' directive globally to suppress this message
[Wed May 18 20:52:16.872163 2022] [mpm_event:notice] [pid 1:tid 139726115282240] AH00489: Apache/2.4.53 (Unix) configured -- resuming normal operations
[Wed May 18 20:52:16.872488 2022] [core:notice] [pid 1:tid 139726115282240] AH00094: Command line: 'httpd -D FOREGROUND'
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.0.1. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.0.1. Set the 'ServerName' directive globally to suppress this message
[Wed May 18 20:52:18.134335 2022] [mpm_event:notice] [pid 1:tid 140535473859904] AH00489: Apache/2.4.53 (Unix) configured -- resuming normal operations
[Wed May 18 20:52:18.134633 2022] [core:notice] [pid 1:tid 140535473859904] AH00094: Command line: 'httpd -D FOREGROUND'
172.31.24.84 - - [18/May/2022:21:45:00 +0000] "GET / HTTP/1.1" 200 45
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.0.202. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.0.0.202. Set the 'ServerName' directive globally to suppress this message
[Wed May 18 20:52:15.789629 2022] [mpm_event:notice] [pid 1:tid 140454359244096] AH00489: Apache/2.4.53 (Unix) configured -- resuming normal operations
[Wed May 18 20:52:15.789803 2022] [core:notice] [pid 1:tid 140454359244096] AH00094: Command line: 'httpd -D FOREGROUND'

ubuntu@ip-172-31-24-84:~$
```

-- or --

```
ubuntu@ip-172-31-24-84:~$ kubectl logs -l app=website | grep GET

172.31.24.84 - - [18/May/2022:21:45:00 +0000] "GET / HTTP/1.1" 200 45

ubuntu@ip-172-31-24-84:~$
```

There are other ways to load balance ClusterIPs, including:

- kube-proxy user mode
- kube-proxy iptables (the default)
- kube-proxy IPVS
- CNI plugins replacing kube-proxy

For example, you may have noticed in our Cilium config, the following line:

`kube-proxy-replacement                 disabled`

When kube-proxy-replacement is enabled, Cilium implements ClusterIPs by updating eBPF map entries on each node.


### Service Resilience

You might think a random load balancer might not produce the best results, however given stateless pods in a Kubernetes
environment are fairly dynamic, this works reasonably well in practice under many circumstances. When a pod is deleted
client connections are closed or broken, causing them to reconnect to one of the remaining pods, redistributing load
regularly. Many things cause pods to be deleted:

- Administrators terminating troublesome pods
- Autoscalers reducing the number of replicas
- Nodes (Kubelets) evicting pods when the node experiences resource contention
- Pods evicted by the scheduler when preemption is configured
- Node crash
- Node brought down for maintenance
- and so on.

Let's see how our service responds to changing pod replicas. Scale away one of your pods and then curl your ClusterIP:

```
ubuntu@ip-172-31-24-84:~$ kubectl scale deployment website --replicas=2

deployment.apps/website scaled

ubuntu@ip-172-31-24-84:~$ kubectl get pod

NAME                      READY   STATUS    RESTARTS   AGE
website-5746f499f-qzjjq   1/1     Running   0          85m
website-5746f499f-v9shz   1/1     Running   0          85m

ubuntu@ip-172-31-24-84:~$ curl 10.111.148.30

<html><body><h1>It works!</h1></body></html>

ubuntu@ip-172-31-24-84:~$
```

You can try to curl as many times as you like but the request will not fail because the deleted pod has been removed
from the service routing mesh. Display the endpoints:

```
ubuntu@ip-172-31-24-84:~$ kubectl get endpoints website

NAME      ENDPOINTS                   AGE
website   10.0.0.1:80,10.0.0.202:80   69m

ubuntu@ip-172-31-24-84:~$
```

Try deleting a pod and recheck your service endpoints (be sure to use the name of one of your pods):

```
ubuntu@ip-172-31-24-84:~$ kubectl get pod

NAME                      READY   STATUS    RESTARTS   AGE
website-5746f499f-qzjjq   1/1     Running   0          90m
website-5746f499f-v9shz   1/1     Running   0          90m

ubuntu@ip-172-31-24-84:~$ kubectl delete pod website-5746f499f-qzjjq

pod "website-5746f499f-qzjjq" deleted

ubuntu@ip-172-31-24-84:~$ kubectl get endpoints website

NAME      ENDPOINTS                    AGE
website   10.0.0.12:80,10.0.0.202:80   71m

ubuntu@ip-172-31-24-84:~$
```

What happened?!?

In the example above, the pod with IP `10.0.0.1` was deleted but the deployment's replica set is scaled to 2, so it
quickly created a replacement pod (`10.0.0.12` in the example). Not that pods managed by deployments are ephemeral and
when deleted, they stay deleted. Brand new pods are created to take their place.

### Clean up

Delete the resources you created in this step so that you can start the next step with a clean slate:

```
ubuntu@ip-172-31-24-84:~$ kubectl delete service/website deploy/website

service "website" deleted
deployment.apps "website" deleted

ubuntu@ip-172-31-24-84:~$
```

Next up is DNS!!


## 3. DNS

Most Kubernetes distributions use CoreDNS as the "built-in" DNS service for Kubernetes. The Kubernetes DNS can be used
to automatically resolve a service name to it's ClusterIP. Let's try it with our website service!

Create a website deployment and service:

```
ubuntu@ip-172-31-24-84:~$ kubectl create deployment website --replicas=3 --image=httpd

deployment.apps/website created

ubuntu@ip-172-31-24-84:~$ kubectl expose deployment website --port=80

service/website exposed

ubuntu@ip-172-31-24-84:~$
```


### Service DNS

Run a client Pod interactively:

```
ubuntu@ip-172-31-24-84:~$ kubectl run -it client --image busybox

If you don't see a command prompt, try pressing enter.
/ #
```

Now try hitting the website service by name:

```
/ # wget -qO - website

<html><body><h1>It works!</h1></body></html>

/ #
```

Wow! Free DNS!! How does this work?

It works like normal DNS pretty much. Step one when faced with a name and not an IP address is to look up the name in
DNS. On Linux the /etc/resolv.conf file is where we find the address of the name server to use for name resolution:

```
/ # cat /etc/resolv.conf

nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local eu-central-1.compute.internal
options ndots:5

/ #
```

As it turns out this file is injected into our container at the request of the Kubelet based on the Kubernetes and Pod
configuration settings. the address `10.96.0.10` is, wait for it ..., the ClusterIP of the CoreDNS Service. We'll verify
this in a few minutes.

Note the second line in the resolv.conf. The name `website` is not a fully qualified DNS name. Remember that Kubernetes
supports namespaces and that we are in the `default` namespace. The fully qualified DNS name of a Kubernetes service is
of the form:  <Service Name>.<Namespace>.svc.<Cluster DNS Suffix>

The default cluster suffix is `cluster.local`, but we could have configured it to be `k8s32.rx-m.com`, or whatever. So
when the resolver looks up `website` it applies the search suffix `default.svc.cluster.local` creating
`website.default.svc.cluster.local`, the fully qualified domain name of our service. The other search suffixes are
applied in order until resolution occurs or the lookup fails.

For example, qualify the service with the namespace:

```
/ # wget -qO - website.default

<html><body><h1>It works!</h1></body></html>

/ #
```

This works because our service is in the default namespace and the search suffix `svc.cluster.local` completes the
lookup. Try looking up the website service in the `kube-system` namespace:

```
/ # wget -qO - website.kube-system

wget: bad address 'website.kube-system'

/ #
```

No such service! Name spacing allows independent teams to use service names without worrying about collisions.

Exit the client shell and list the services in the `kube-system` namespace:

```
ubuntu@ip-172-31-24-84:~$ kubectl get service -n kube-system

NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   37h

ubuntu@ip-172-31-24-84:~$
```

Recognize that IP address? It's our cluster DNS service. Services give dynamic sets of pods a stable identity.


### Headless Services

As we have seen, pods under a deployment can come and go. Services can create a stable identity for as dynamic set of
pods in the form of a service DNS name. What if our individual pods have identity?

Pods that have a unique identity within a set are identified by their state and are therefore not technically
microservices (which are stateless and ephemeral). Examples include Cassandra Pods, Kafka Pods and Nats Pods. Each of
these examples may involve a cluster of pods, each running the same container, however they all have unique data and/or
responsibilities. If you connected to a Kafka broker pod that does not have the topic you would like to read, you will
be redirected to the correct pod. So how does this work with Kubernetes services?

It doesn't. Well, not with the services we have seen so far. However, a headless service, that is a service with no
ClusterIP, can be used for just such a purpose. To demonstrate, create a headless service for your website:

```
ubuntu@ip-172-31-24-84:~$ vim headless.yaml

ubuntu@ip-172-31-24-84:~$ cat headless.yaml

apiVersion: v1
kind: Service
metadata:
  name: headlesswebsite
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: website
  clusterIP: None

ubuntu@ip-172-31-24-84:~$ kubectl apply -f headless.yaml

service/headlesswebsite created

ubuntu@ip-172-31-24-84:~$
```

List your service:

```
ubuntu@ip-172-31-24-84:~$ kubectl get services

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
headlesswebsite   ClusterIP   None            <none>        80/TCP         41s
kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP        37h
website           NodePort    10.111.148.30   <none>        80:31100/TCP   133m

ubuntu@ip-172-31-24-84:~$
```

Notice that this service has no ClusterIP. Without a `head`, clients can not use this service to randomly load balance
over the backend pods (which would not work if we were trying to reach a specific pod!).

So how does the headless service help us? To demonstrate, let's reattach to our client pod:

```
/ # nslookup headlesswebsite

Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   headlesswebsite.default.svc.cluster.local
Address: 10.0.0.202
Name:   headlesswebsite.default.svc.cluster.local
Address: 10.0.0.12

*** Can't find headlesswebsite.svc.cluster.local: No answer
*** Can't find headlesswebsite.cluster.local: No answer
*** Can't find headlesswebsite.eu-central-1.compute.internal: No answer
*** Can't find headlesswebsite.default.svc.cluster.local: No answer
*** Can't find headlesswebsite.svc.cluster.local: No answer
*** Can't find headlesswebsite.cluster.local: No answer
*** Can't find headlesswebsite.eu-central-1.compute.internal: No answer

/ #
```

Ignoring the failed lookups, you can see that looking up a headless service by name actually returns the IP addresses of
the endpoints.

You can also retrieve the SRV record for the service:

```
/ # nslookup -q=SRV headlesswebsite

Server:         10.96.0.10
Address:        10.96.0.10:53

headlesswebsite.default.svc.cluster.local       service = 0 50 80 10-0-0-12.headlesswebsite.default.svc.cluster.local
headlesswebsite.default.svc.cluster.local       service = 0 50 80 10-0-0-202.headlesswebsite.default.svc.cluster.local

*** Can't find headlesswebsite.svc.cluster.local: No answer
*** Can't find headlesswebsite.cluster.local: No answer
*** Can't find headlesswebsite.eu-central-1.compute.internal: No answer

/ #
```

If you want to lookup a pod by name you need a headless service and a StatefulSet. A StatefulSet is much like a
Deployment in that it manages a set of pods, however a StatefulSet creates pods with stable identities, in other words,
stable DNS names. As we saw earlier, deleting a Deployment pod causes a brand new pod with a brand new name to be
created. Deleting a StatefulSet pod causes the exact same pod name to be recreated, and, if specified, attached to the
exact same set of storage volumes.

Exit the Client pod and create a StatefulSet with a headless Service to see how this works:

```
/ # exit
Session ended, resume using 'kubectl attach client -c client -i -t' command when the pod is running

ubuntu@ip-172-31-24-84:~$ vim stateful.yaml

ubuntu@ip-172-31-24-84:~$ cat stateful.yaml

apiVersion: v1
kind: Service
metadata:
  name: webstate
spec:
  ports:
  - port: 80
    name: webstate
  clusterIP: None
  selector:
    app: webstate
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: webstate
spec:
  selector:
    matchLabels:
      app: webstate
  serviceName: webstate
  template:
    metadata:
      labels:
        app: webstate
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8

ubuntu@ip-172-31-24-84:~$ kubectl apply -f stateful.yaml

service/webstate created
statefulset.apps/webstate created

ubuntu@ip-172-31-24-84:~$
```

Take a look at the StatefulSet:

```
ubuntu@ip-172-31-24-84:~$ kubectl get sts,po

NAME                        READY   AGE
statefulset.apps/webstate   1/1     79s

NAME                          READY   STATUS    RESTARTS        AGE
pod/client                    1/1     Running   2 (5m22s ago)   46m
pod/website-5746f499f-nnpxc   1/1     Running   0               79m
pod/website-5746f499f-v9shz   1/1     Running   0               170m
pod/webstate-0                1/1     Running   0               79s

ubuntu@ip-172-31-24-84:~$
```

The stateful pod has a predictable name, the name of the sts followed by a dash and an ordinal representing the order in
which the pod was created. Scale the StatefulSet:

```
ubuntu@ip-172-31-24-84:~$ kubectl scale sts webstate --replicas 3

statefulset.apps/webstate scaled

ubuntu@ip-172-31-24-84:~$ kubectl get pod -l app=webstate

NAME         READY   STATUS    RESTARTS   AGE
webstate-0   1/1     Running   0          3m43s
webstate-1   1/1     Running   0          14s
webstate-2   1/1     Running   0          11s

ubuntu@ip-172-31-24-84:~$
```

Now lets try DNS on our sts. Attach to the client pod and do some lookups:

```
ubuntu@ip-172-31-24-84:~$ kubectl attach -it client

If you don't see a command prompt, try pressing enter.

/ # nslookup webstate.default.svc.cluster.local

Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   webstate.default.svc.cluster.local
Address: 10.0.0.58
Name:   webstate.default.svc.cluster.local
Address: 10.0.0.38
Name:   webstate.default.svc.cluster.local
Address: 10.0.0.52

*** Can't find webstate.default.svc.cluster.local: No answer

/ # nslookup -q=SRV webstate.default.svc.cluster.local

Server:         10.96.0.10
Address:        10.96.0.10:53

webstate.default.svc.cluster.local      service = 0 33 80 webstate-0.webstate.default.svc.cluster.local
webstate.default.svc.cluster.local      service = 0 33 80 webstate-1.webstate.default.svc.cluster.local
webstate.default.svc.cluster.local      service = 0 33 80 webstate-2.webstate.default.svc.cluster.local

/ # wget -qO - webstate-1.webstate.default.svc.cluster.local

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

/ #
```

These pods have DNS identity!


### Cleanup

Delete all of the resources from this step to prepare for the next step:

```
ubuntu@ip-172-31-24-84:~$ kubectl delete pod/client service/headlesswebsite service/web service/webstate

pod "client" deleted
service "headlesswebsite" deleted
service "web" deleted
service "webstate" deleted

ubuntu@ip-172-31-24-84:~$ kubectl delete statefulset.apps/webstate

statefulset.apps "webstate" deleted

ubuntu@ip-172-31-24-84:~$ kubectl delete service/website deployment.apps/website

service "website" deleted
deployment.apps "website" deleted

ubuntu@ip-172-31-24-84:~$
```


## 4. External Access

In this step we'll take a look at ways to reach our cluster based services from outside the cluster.


### NodePort Services

As demonstrated earlier, ClusterIPs are virtual, they only exist as rules in IPTables or within some other forwarding
component of the kernel on nodes in the cluster. So how do we reach a service from outside of the cluster when the
kube-proxy and CNI plugins creating these forwarding instructions only run on cluster nodes.

Well, there are various ways to get into a cluster but one important way is through a NodePort Service. A NodePort
service uses a specific port on every host computer in the Kubernetes cluster to forward traffic to pods on the pod
network. In this way, if you can reach one of the host machines (nodes) you can reach all of the NodePort services in
the cluster.

Create a service using the NodePort type and a node port of 31100:

```
ubuntu@ip-172-31-24-84:~$ vim service.yaml

ubuntu@ip-172-31-24-84:~$ cat service.yaml

apiVersion: v1
kind: Service
metadata:
  name: website
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 31100
    name: http
  selector:
    app: website

ubuntu@ip-172-31-24-84:~$ kubectl apply -f service.yaml

service/website configured

ubuntu@ip-172-31-24-84:~$
```

Next create some pods for the service to forward to:

```
ubuntu@ip-172-31-24-84:~$ kubectl create deployment website --replicas=3 --image=httpd

deployment.apps/website created

ubuntu@ip-172-31-24-84:~$
```

Now from your laptop (not the cloud instance lab system), try to curl the public IP of your lab machine on port 31100:

```
$ curl -s 18.185.35.194:31100

<html><body><h1>It works!</h1></body></html>

$
```

How does this work?

While this can be implemented in many ways, the default behavior is again iptables based:

```
ubuntu@ip-172-31-24-84:~$ sudo iptables -nvL -t nat | grep 31100

    3   156 KUBE-EXT-RYQJBQ5TR32XWAUN  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/website:http */ tcp dpt:31100

ubuntu@ip-172-31-24-84:~$
```

Traffic targeting any IP on port 31100 jumps to chain `KUBE-EXT-RYQJBQ5TR32XWAUN`, which sends us back to the normal
service chain:

```
ubuntu@ip-172-31-24-84:~$ sudo iptables -nvL -t nat | grep -A3 'Chain KUBE-EXT-RYQJBQ5TR32XWAUN'

Chain KUBE-EXT-RYQJBQ5TR32XWAUN (1 references)
 pkts bytes target     prot opt in     out     source               destination
    3   156 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* masquerade traffic for default/website:http external destinations */
    3   156 KUBE-SVC-RYQJBQ5TR32XWAUN  all  --  *      *       0.0.0.0/0            0.0.0.0/0

ubuntu@ip-172-31-24-84:~$
```

NodePort enabled!


### Setup an Ingress Controller / Gateway

The downside of externally exposed services, like NodePorts, is that each set of pods needs its own service. Your
clients (in house developers, third party developers, etc.) probably do not want to keep track of a bunch of service
names, ports and IPs.

Kubernetes introduced an ingress framework to allow a single externally facing gateway (called an Ingress Controller) to
route HTTP/HTTPS traffic to multiple backend services.

Like many other networking functions, upstream Kubernetes does not come with an Ingress Controller, however there are
several good, free, open source options. In this lab we will use Emissary, a CNCF project built on top of the Envoy
proxy, another CNCF project. Emissary implements not only the features required by Kubernetes Ingress but also defines
many custom Resources we can use to access functionality well beyond the generic (but portable) basic Kubernetes
Ingress. Tools like Emissary are often called Gateways because they provide many advanced features used to control
inbound application traffic.

Installing Emissary is easy. Like many Kubernetes addons, Emissary prefers to run in its own namespace.

Create the Emissary namespace:

```
ubuntu@ip-172-31-24-84:~$ kubectl create namespace emissary

namespace/emissary created

ubuntu@ip-172-31-24-84:~$
```

Great, now we can create the Custom Resource Definitions (CRDs) Emissary depends on:

```
ubuntu@ip-172-31-24-84:~$ kubectl apply -f https://app.getambassador.io/yaml/emissary/2.2.2/emissary-crds.yaml

customresourcedefinition.apiextensions.k8s.io/authservices.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/consulresolvers.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/devportals.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/hosts.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/kubernetesendpointresolvers.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/kubernetesserviceresolvers.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/listeners.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/logservices.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/mappings.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/modules.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/ratelimitservices.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/tcpmappings.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/tlscontexts.getambassador.io created
customresourcedefinition.apiextensions.k8s.io/tracingservices.getambassador.io created
namespace/emissary-system created
serviceaccount/emissary-apiext created
clusterrole.rbac.authorization.k8s.io/emissary-apiext created
clusterrolebinding.rbac.authorization.k8s.io/emissary-apiext created
role.rbac.authorization.k8s.io/emissary-apiext created
rolebinding.rbac.authorization.k8s.io/emissary-apiext created
service/emissary-apiext created
deployment.apps/emissary-apiext created

ubuntu@ip-172-31-24-84:~$
```

The manifest applied above also creates the emissary-system namespace, an extension service and some security
primitives. Note that "Ambassador" was the original name of the "Emissary" project so there are sill many bits of the
tool that still use the old name.

Now we can setup the Emissary controller:

```
ubuntu@ip-172-31-24-84:~$ kubectl apply -f https://app.getambassador.io/yaml/emissary/2.2.2/emissary-emissaryns.yaml

service/emissary-ingress-admin created
service/emissary-ingress created
service/emissary-ingress-agent created
clusterrole.rbac.authorization.k8s.io/emissary-ingress created
serviceaccount/emissary-ingress created
clusterrolebinding.rbac.authorization.k8s.io/emissary-ingress created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-crd created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-watch created
deployment.apps/emissary-ingress created
serviceaccount/emissary-ingress-agent created
clusterrolebinding.rbac.authorization.k8s.io/emissary-ingress-agent created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-agent created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-agent-pods created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-agent-rollouts created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-agent-applications created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-agent-deployments created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-agent-endpoints created
clusterrole.rbac.authorization.k8s.io/emissary-ingress-agent-configmaps created
role.rbac.authorization.k8s.io/emissary-ingress-agent-config created
rolebinding.rbac.authorization.k8s.io/emissary-ingress-agent-config created
deployment.apps/emissary-ingress-agent created

ubuntu@ip-172-31-24-84:~$
```

It can take a bit for all of the container images to download and start so let's wait until the Emissary deployment is available:

```
ubuntu@ip-172-31-24-84:~$ kubectl -n emissary wait --for condition=available --timeout=90s deploy emissary-ingress

deployment.apps/emissary-ingress condition met

ubuntu@ip-172-31-24-84:~$
```

Take a look at our new resources:

```
ubuntu@ip-172-31-24-84:~$ kubectl get all -n emissary

NAME                                          READY   STATUS    RESTARTS   AGE
pod/emissary-ingress-845489689d-dz245         1/1     Running   0          23m
pod/emissary-ingress-845489689d-mdppn         1/1     Running   0          23m
pod/emissary-ingress-845489689d-s27gm         1/1     Running   0          23m
pod/emissary-ingress-agent-75ccf64bc8-6l6h6   1/1     Running   0          23m

NAME                             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
service/emissary-ingress         LoadBalancer   10.107.25.56     <pending>     80:31211/TCP,443:31361/TCP      23m
service/emissary-ingress-admin   NodePort       10.105.156.190   <none>        8877:30768/TCP,8005:31779/TCP   23m
service/emissary-ingress-agent   ClusterIP      10.96.6.49       <none>        80/TCP                          23m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/emissary-ingress         3/3     3            3           23m
deployment.apps/emissary-ingress-agent   1/1     1            1           23m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/emissary-ingress-845489689d         3         3         3       23m
replicaset.apps/emissary-ingress-agent-75ccf64bc8   1         1         1       23m

ubuntu@ip-172-31-24-84:~$
```

The emissary-ingress service is of type LoadBalancer, which includes NodePorts of 31211 mapped to port 80 and 31361
mapped to port 443.

The emissary-ingress deployment is scaled to 3. Given we are on a single node cluster, lets drop that down to 1:

```
ubuntu@ip-172-31-24-84:~$ kubectl scale deployment.apps/emissary-ingress --replicas=1 -n emissary

deployment.apps/emissary-ingress scaled

ubuntu@ip-172-31-24-84:~$
```

Alright, Emissary is all set. Now let's expose a couple of internal services through our new Gateway.


### Gateway Ingress

When using Emissary's more advanced Gateway functionality, we will need to create two custom resources to send traffic
to our website service.

- Listener - tells Emissary what ports and protocols to listen on
- Mapping - tells Emissary which traffic to forward to which service

List the CRDs created by Emissary:

```
ubuntu@ip-172-31-24-84:~$ kubectl api-resources | grep getambassador

authservices                            getambassador.io/v2                    true         AuthService
consulresolvers                         getambassador.io/v2                    true         ConsulResolver
devportals                              getambassador.io/v2                    true         DevPortal
hosts                                   getambassador.io/v2                    true         Host
kubernetesendpointresolvers             getambassador.io/v2                    true         KubernetesEndpointResolver
kubernetesserviceresolvers              getambassador.io/v2                    true         KubernetesServiceResolver
listeners                               getambassador.io/v3alpha1              true         Listener
logservices                             getambassador.io/v2                    true         LogService
mappings                                getambassador.io/v2                    true         Mapping
modules                                 getambassador.io/v2                    true         Module
ratelimitservices                       getambassador.io/v2                    true         RateLimitService
tcpmappings                             getambassador.io/v2                    true         TCPMapping
tlscontexts                             getambassador.io/v2                    true         TLSContext
tracingservices                         getambassador.io/v2                    true         TracingService

ubuntu@ip-172-31-24-84:~$
```

Let's create an HTTP Listener for use with our website service:

```
ubuntu@ip-172-31-24-84:~$ vim listen.yaml

ubuntu@ip-172-31-24-84:~$ cat listen.yaml

apiVersion: getambassador.io/v3alpha1
kind: Listener
metadata:
  name: emissary-ingress-listener
  namespace: emissary
spec:
  port: 80
  protocol: HTTP
  securityModel: XFP
  hostBinding:
    namespace:
      from: ALL

ubuntu@ip-172-31-24-84:~$ kubectl apply -f listen.yaml

listener.getambassador.io/emissary-ingress-listener created

ubuntu@ip-172-31-24-84:~$
```

Great now let's add a Mapping that forwards traffic using the `/web` route to our `website` service:

```
ubuntu@ip-172-31-24-84:~$ vim map.yaml

ubuntu@ip-172-31-24-84:~$ cat map.yaml

apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: website
spec:
  hostname: "*"
  prefix: /web/
  service: website

ubuntu@ip-172-31-24-84:~$ kubectl apply -f map.yaml

mapping.getambassador.io/website created

ubuntu@ip-172-31-24-84:~$
```

Ok, let's give it a try! From your laptop (outside of the cluster), curl the public IP of you lab system using the
nodeport for the Emissary Gateway and the `web` route:

```
$ curl -s http://18.185.35.194:31211/web/

<html><body><h1>It works!</h1></body></html>

$
```

It works! Try something that won't work:

```
$ curl -i http://18.185.35.194:31211/engine/

HTTP/1.1 404 Not Found
date: Thu, 19 May 2022 21:38:04 GMT
server: envoy
content-length: 0

$
```

As you can see, the Emissary control plane deploys an Envoy proxy to actually manage the data path. You can also see
that the route `engine` is not supported. Let fix that!


### Create Ingress Resources

The Kubernetes Ingress framework allows us to create ingress rules using the ingress resource type. Ingress resources
are not as powerful or feature filled as the Emissary CRDs. They are, however, portable and they do provide basic
functionality. If you can live with the smaller feature set of Ingress resources, perhaps you should, they are more
widely understood and will work with any decent Kubernetes gateway.

Emissary can of course support normal Ingress resources as well as its advanced CRDs. Let's wrap up this step by creating an ingress rule that routes traffic destined for `/engine` to an nginx deployment.

First create an nginx deployment and a service for it:

```
ubuntu@ip-172-31-24-84:~$ kubectl create deploy engine --image=nginx

deployment.apps/engine created

ubuntu@ip-172-31-24-84:~$ kubectl expose deploy engine --port=80

service/engine exposed

ubuntu@ip-172-31-24-84:~$
```

Now we can tell Emissary to route to the new service with an Ingress resource. An Ingress resource is a set of one or
more rules for processing inbound traffic received by the Ingress controller.

Create a standard Kubernetes Ingress for the nginx service:

```
ubuntu@ip-172-31-24-84:~$ vim ing.yaml

ubuntu@ip-172-31-24-84:~$ cat ing.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: ambassador
  name: web-ingress
spec:
  ingressClassName: ambassador
  rules:
  - http:
      paths:
      - path: /engine
        pathType: Prefix
        backend:
          service:
            name: engine
            port:
              number: 80

ubuntu@ip-172-31-24-84:~$ kubectl apply -f ing.yaml

ingress.networking.k8s.io/web-ingress configured

ubuntu@ip-172-31-24-84:~$
```

Alright now try hitting the `/engine` route again from your laptop:

```
$ curl -i http://18.185.35.194:31211/engine/
HTTP/1.1 200 OK
server: envoy
date: Thu, 19 May 2022 21:44:06 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 25 Jan 2022 15:03:52 GMT
etag: "61f01158-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 0

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

$
```

Boom! We now have two services running in our cluster that are exposed through the Emissary Ingress Gateway!!


## 5. Load Balancing

## 6. Service Mesh


<br>

Congratulations you have completed the lab! Amazing!!

<br>

_Copyright (c) 2022 RX-M LLC, Cloud Native Consulting, all rights reserved_
