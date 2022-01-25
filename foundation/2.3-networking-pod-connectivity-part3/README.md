## 2.3. Networking:  Pod Connectivity - Part 2

This is the 3rd lab in a series of labs exploring k8s networking. This lab focused on understanding relevant address ranges and BGP advertisement.
In this lab, you will: \
2.3.1. Examine IP address ranges used by the cluster. \
2.3.2. Create additional Calico IP Pools. \
2.3.3. Configure Calico BGP Peering to connect with a network outside of the cluster. \
2.3.4. Configure a namespace to use externally routable IP addresses.

### 2.3.0. Before you begin

This lab builds in the learning in previous labs. You should have by now your k8s cluster setup with Calico and the Yaobank sample application deployed. 

### 2.3.1. Examine IP address ranges used by the cluster

#### 2.3.1.1. Kubernetes configuration

There are two address ranges that Kubernetes is normally configured with that are important to understand:
* The cluster pod CIDR is the range of IP addresses Kubernetes is expecting to be assigned to pods in the cluster.
* The services CIDR is the range of IP addresses that are used for the Cluster IPs of Kubernetes Sevices (the virtual IP that corresponds to each Kubernetes Service).

These are configured at cluster creation time (e.g. as initial kubeadm configuration).

You can find these values using the following command on the master node:
```
kubectl cluster-info dump | grep -m 2 -E "service-cluster-ip-range|cluster-cidr"
```
```
                            "--service-cluster-ip-range=10.49.0.0/16",
                            "--cluster-cidr=10.48.0.0/16",
```

#### 2.3.1.2. Calico configuration

It is also important to understand the IP Pools that Calico has been configured with, which offers fine grained control over IP address ranges to be used by pods in the cluster.

```
calicoctl get ippools
```
```
NAME                  CIDR             SELECTOR   
default-ipv4-ippool   10.48.0.0/24     all()      
pool2-ipv4-ippool     10.48.128.0/24   all()  
```

Overall we have the following address ranges:

| CIDR           |  Purpose                                                  |
|----------------|-----------------------------------------------------------|
| 10.48.0.0/16   | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24   | Calico - Initial default IP Pool                          |
| 10.48.128.0/24 | Calico pool we created in the previous lab                |
| 10.49.0.0/16   | Kubernetes Service Network (via kubeadm `--service-cidr`) |

### 2.3.2. Create additional Calico IP Pools

One use of Calico IP Pools is to distinguish between different ranges of addresses that different routablity scopes. If you are operating at very large scales then IP addresses are precious. You might want to have a range of IPs that is only routable within the cluster, and another range of IPs that is routable across your enterprise. In that case, you can choose which pods get IPs from which range depending on whether workloads from outside of the cluster need direct access to the pods or not.

We'll simulate this use case in this lab by creating another IP Pool to represent the externally routable pool.  (And we've already configured the underlying network to no allow routing of the existing IP Pool outside of the cluster.)

#### 2.3.2.1. Create externally routable IP Pool

We're going to create a new pool for `10.48.2.0/24` that is externally routable.
```
calicoctl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: external-pool
spec:
  cidr: 10.48.2.0/24
  blockSize: 29
  ipipMode: Never
  natOutgoing: true
EOF
```
```
calicoctl get ippools
```
```
NAME                  CIDR             SELECTOR   
default-ipv4-ippool   10.48.0.0/24     all()      
external-pool         10.48.2.0/24     all()      
pool2-ipv4-ippool     10.48.128.0/24   all()     
```

We now have:

| CIDR           |  Purpose                                                  |
|----------------|-----------------------------------------------------------|
| 10.48.0.0/16   | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24   | Calico - Initial default IP Pool                          |
| 10.48.128.0/24 | Calico pool we created in the previous lab                |
| 10.48.2.0/24   | Calico - External IP Pool (externally routable)           |
| 10.49.0.0/16   | Kubernetes Service Network (via kubeadm `--service-cidr`) |

### 2.3.3. Configure Calico BGP peering

#### 2.3.3.1. Examine BGP peering status

Switch to worker1:

```
ssh worker1
```

Make sure calicoctl binary installed on your worker1 node.

```
curl -o calicoctl -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.21.0/calicoctl"
```
```
chmod +x calicoctl
sudo mv calicoctl /usr/local/bin
```

Check the status of Calico on the node:

```
sudo calicoctl node status
```
```
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.1.20    | node-to-node mesh | up    | 05:40:21 | Established |
| 10.0.1.31    | node-to-node mesh | up    | 05:40:34 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

This shows that currently Calico is only peering with the other nodes in the cluster, and is not peering with any external router.

Exit back to the bastion host:

```
exit
```

#### 2.3.3.2. Add a BGP Peer

In this lab we will simulate peering to a router outside of the cluster by peering to host1. We've already set-up host1 to act as a router, and it is ready to accept new BGP peering.

Add the new BGP Peer:

```
calicoctl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
EOF
```

#### 2.3.3.3. Examine the new BGP peering status

Switch to worker1:

```
ssh worker1
```

Check the status of Calico on the node:

```
sudo calicoctl node status
```
```
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.1.20    | node-to-node mesh | up    | 05:40:21 | Established |
| 10.0.1.31    | node-to-node mesh | up    | 05:40:34 | Established |
| 10.0.1.10    | global            | up    | 08:03:51 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

The output shows that Calico is now peered with host1 (`10.0.1.10`), which is our bastion host. This means Calico can share routes to and learn routes from it.

In a real-world on-prem deployment you would typically configure Calico nodes within a rack to peer with the ToRs (Top of Rack) routers, and the ToRs are then connected to the rest of the enterprise or data center network. In this way pods can be reached from anywhere in the network if desired. You could even go as far as giving some pods public IP address and have them addressable from the internet if you wanted to.

We're done with adding the peers, so exit from worker1 to return back to the bastion host:

```
exit
```

### 2.3.4. Configure a namespace to use externally routable IP addresses

In the previous lab, we saw Calico supports annotations on both namespaces and pods that can be used to control which IP Pool (or even which IP address) a pod will receive when it is created.  Previously, we defined the annotations under the deployments themselves, but in this example we're going to create a namespace to host externally routable Pods.

#### 2.3.4.1. Create the namespace

Examine the namespaces we're about to create:

```
more 2.3-namespace.yaml
```

Notice the annotation that will determine which IP Pool pods in the namespace will use.

Apply the namespace:

```
kubectl apply -f 2.3-namespace.yaml
```

#### 2.3.4.2. Deploy an nginx pod

Now deploy a NGINX example pod in the `external-ns` namespace, along with a simple network policy that allows ingress on port 80.

```
kubectl apply -f 2.3-nginx.yaml
```

#### 2.3.4.3. Access the nginx pod from outside the cluster

Let's see what IP address was assigned:

```
kubectl get pods -n external-ns -o wide
```
```
NAME                     READY   STATUS    RESTARTS   AGE    IP            NODE           NOMINATED NODE   READINESS GATES
nginx-5ff4dff99d-6wjc5   1/1     Running   0          2m5s   10.48.2.232   ip-10-0-1-30   <none>           <none>
```

The output shows that the nginx pod has an IP address from the externally routable IP Pool.

Try to connect to it from host1:

```
curl 10.48.2.232
```

This should have succeeded, showing that the nginx pod is directly routable on the broader network, as that route is being learned by BGP in our bastion host:

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.232/29 via 10.0.1.30 dev ens5 proto bird
```

If you would like to see IP allocation stats from Calico-IPAM, run the following command.

```
calicoctl ipam show
```
