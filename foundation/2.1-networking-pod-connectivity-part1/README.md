## 2.1. Networking: Pod Connectivity - Part 1

This lab is the first of a series of labs exploring k8s networking concepts. This lab focuses on basic k8s networking from the POD and Host perspectives.   
In this lab, you will: \
2.1. Examine what the network looks like from the perspecitve of a pod (the pod network namespace) \
2.2. Examine what the network looks like from the perspecitve of the host (the host network namespace)

### 2.1.0. Before you begin

The prerequisite to this lab is completing Lab1, which is installing Calico CNI and deploying the sample application Yaobank.

### 2.1.1. Examine pod network namespace

We'll start by examining what the network looks like from the pod's point of view. Each pod get's its own Linux network namespace, which you can think of as giving it an isolated copy of the Linux networking stack. 

#### 2.1.1.1. Find the name and location of the customer pod

From k8s master node, get the details of the customer pod using the following command.

```
kubectl get pods -n yaobank -l app=customer -o wide
```
```
NAME                       READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
customer-787758576-f2zgn   1/1     Running   0          3m56s   10.48.0.131   ip-10-0-1-31   <none>           <none>
```

Note the node on which the pod is running on (`ip-10-0-1-31` in the example above.)

#### 2.1.1.2. Exec into the customer pod

Use kubectl to exec into the pod so we can check the pod networking details. 

```
kubectl exec -it -n yaobank $(kubectl get pods -n yaobank -l app=customer -o name) -- bash
```

#### 2.1.1.3. Examine the pod's networking

First we will use `ip addr` to list the addresses and associated network interfaces that the pod sees.

```
ip addr
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether 62:fc:7f:a0:5f:de brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.48.0.133/32 brd 10.48.0.133 scope global eth0
       valid_lft forever preferred_lft forever
```

* There is an `eth0` interface which has the pods actual IP address, (`10.48.0.133` in the example). Notice this matches the IP address that `kubectl get pods` returned earlier. The above output shows a tunnel interface, as in the case above IPIP is configured in the pool at the time of deployment, however in your lab, such interface will not be there as we did not set this parameter in the manifest "1-custom-resources.yaml" which we inspected in the previous lab:

```
calicoctl get ippool -o yaml
```
```
apiVersion: projectcalico.org/v3
items:
- apiVersion: projectcalico.org/v3
  kind: IPPool
  metadata:
    creationTimestamp: "2021-05-08T09:34:21Z"
    name: default-ipv4-ippool
    resourceVersion: "1385"
    uid: bf6da8a8-a282-4fb9-9796-aecbb6d276f2
  spec:
    blockSize: 26
    cidr: 10.48.0.0/24
    ipipMode: Never
    natOutgoing: true
    nodeSelector: all()
    vxlanMode: Never
```

Note the fields:

* blockSize: used by Calico IPAM to efficiently assign ip addresses in blocks to be advertised by BGP.
* ipipMode/vxlanMode: to allow or disable ipip and vxlan overlay. Valid options are Never, Always and Crosssubnet.

Next let's look more closely at the interfaces using `ip link`. 

```
ip -c link
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 62:fc:7f:a0:5f:de brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

Look at the `eth0` part of the output. The are a couple of things to note:
* `eth0` is a link to the host network namespace (indicated by `link-netnsid 0` at the end of that line), and it is the fourth interface in our example above (the number in your lab will be different). That is one end of the virtual ethernet pair which calico will plumb to the host network namespace.
* The `@if11` after the interface name is the interface number of the other end of the veth pair within the host network namespace. Remember the number you see for later, as we will take look at the other end of the veth pair shortly.

Finally, let's look at the routes the pod sees.

```
ip route
```
```
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0  scope link 
```

This shows that the pod's default route is out over the `eth0` interface. Contrary to the pod interface we checked above, the interface calico creates for the pods in the host network namespace is unnumbered, that is the reason we use a "dummy" IP address in the APIPA range.

#### 2.1.1.4. Exit from the customer pod
We've finished checking our pod's view of the network, so we'll exit out of the exec to return to the bastion host.

```
exit
```

### 2.1.2. Examine the host's network namespace

#### 2.1.2.1. SSH into the customer pod's host node
We'll start by switching to the node where the customer pod is running. In our example earlier this was worker 1. (If you've forgotten which node it was for you then repeat step 2.1.1.1 above to find the node.)

```
ssh ubuntu@ip-10-0-1-31
```

#### 2.1.2.2. Examine interfaces
Now we're on the node hosting the customer pod we'll take a look to examine the other end of the veth pair. In our example output earlier, the `@if11` indicated it should be interface number 11 in the host network namespace. (Your interface numbers may be different)

```
ip -c link
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:77:82:a8:2f:60 brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:a0:ca:63:94 brd ff:ff:ff:ff:ff:ff
4: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 8981 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
9: cali8d74f92a1c1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
10: cali93ae965a37d@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
11: cali12a8cc48008@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
```

Looking at interface number 11 in this example, `cali12a8cc48008` links to `@if4` in network namespace ID 2 (the customer pod's network namespace).  You may recall that interface 4 in the pod's network namespace was `eth0`, so this looks exactly as expected for the veth pair that connects the customer pod to the host network namespace.  

You can also see the host end of the veth pairs to other pods running on this node, all beginning with `cali`. This interfaces are unnumbered as we pointed before:

```
ip addr | grep -A3 cali12a8cc48008
```
```
11: cali12a8cc48008@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
```

#### 2.1.2.3. Examine routes
First let's remind ourselves of the `customer` pod's IP address:

```
kubectl get pods -n yaobank -l app=customer -o wide
````

Now lets look at the routes on the host.
```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.31 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.31 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.31 metric 100 
10.48.0.0/26 via 10.0.1.30 dev ens5 proto bird 
blackhole 10.48.0.128/26 proto bird 
10.48.0.131 dev cali8d74f92a1c1 scope link 
10.48.0.132 dev cali93ae965a37d scope link 
10.48.0.133 dev cali12a8cc48008 scope link 
10.48.0.192/26 via 10.0.1.20 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```

In this example output, we can see the route to the customer pod's IP (`10.48.0.133`) is via the `cali12a8cc48008` interface, the host end of the veth pair for the customer pod. You can see similar routes for each of the IPs of the other pods hosted on this node. It's these routes that tell Linux where to send traffic that is destined to a local pod on the node.

We can also see several routes labelled `proto bird`. These are routes to pods on other nodes that Calico has learned over BGP. We will get deeper on BGP as the training progresses, but for now consider this route in the example output above `10.48.0.192/26 via 10.0.1.20 dev ens5 proto bird`.  It indicates pods with IP addresses falling within the `10.48.0.192/26` CIDR can be reached via `10.0.1.20` which is a different node.

Calico uses route aggregation to reduce the number of routes when possible. (e.g. `/26` in this example). The `/26` corresponds to the default block size that Calico IPAM (IP Address Management) allocates on demand as nodes need pod IP addresses, and which is specified as the parameter `blockSize` in the pool we checked in step 2.1.1.3.

You can also see the `blackhole 10.48.0.128/26 proto bird` route. The CIDR `10.48.0.128/26` corresponds to the block of IPs that Calico IPAM allocated on demand for this node. This is the block from which each of the local pods got their IP addresses. The blackhole route tells Linux that if it can't find a more specific route for an individual IP in that block then it should discard the packet (rather than sending it out the default route to the network). You will only see traffic that hits this rule if something is trying to send traffic to a pod IP that doesn't exist, for example sending traffic to a recently deleted pod.

If Calico IPAM runs out of blocks to allocate to nodes, then it will use unused IPs from other nodes' blocks. These will be announced over BGP as more specific routes, so traffic to pods will always find its way to the right host.

### 2.2.4. Exit from the node
We've finished our tour of the Customer pod's host's view of the network. Remember exit out of the exec to return to the bastion host.

```
exit
```
