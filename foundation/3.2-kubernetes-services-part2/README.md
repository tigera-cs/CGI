## 3.2. Kubernetes Service - BGP advertisement

This is the 2nd of a series of labs about k8s services. This lab explores different scenarios for advertising services ip addresses via BGP. In this lab, you will: \
3.2.1. Advertise the service IP range \
3.2.2. Advertise individual service cluster IP \
3.2.3. Advertise individual service external IP

### 3.2.0. Before you begin

This lab builds on previous lab setup and requires Calico CNI to setup, Yaobank application deployed and BGP peering established with teh bastion host.

### 3.2.1. Advertise service cluster IP addresses

Advertising services over BGP allows you to directly access the service without using NodePorts or a cluster Ingress Controller.

#### 3.2.1.1. Examine routes

Before we begin take a look at the state of routes on bastion host which does not belong to the k8s cluster:

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.232/29 via 10.0.1.30 dev ens5 proto bird 
```

If you completed the previous lab correctly, you should see one route that was learned from Calico that provides access to the nginx customer pod that was created in the externally routable namespace (the route ending in `proto bird` in this example output). In this lab we will advertise Kubernetes services (rather than pod CIDRs) over BGP.

#### 3.2.1.2. Add Calico BGP configuration

Examine the default BGP configuration we are about to apply

```
more 3.2-default-bgp.yaml
```

The `serviceClusterIPs` clause tells Calico to advertise the cluster IP range.

Apply the configuration

```
calicoctl apply -f 3.2-default-bgp.yaml
```

Verify the BGPConfiguration contains the `serviceClusterIPs` key

```
calicoctl get bgpconfig default -o yaml
```
```
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
  resourceVersion: "44447"
  uid: 1df0e2fe-cded-42f6-a860-1e6c9e61644d
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
```

#### 3.2.1.3. Examine routes

Examine the routes again on the bastion host:

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.232/29 via 10.0.1.30 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
```

You should now see the cluster service cidr `10.49.0.0/16` advertised from each of the kubernetes cluster nodes. This means that traffic to any service's cluster IP address will get load-balanced across all nodes in the cluster by the network using ECMP (Equal Cost Multi Path). Kube-proxy then load balances the cluster IP across the service endpoints (backing pods) in exactly the same way as if a pod had accessed a service via a cluster IP.

#### 3.2.1.4. Verify we can access cluster IPs

Find the cluster IP for the `customer` service.

```
kubectl get svc -n yaobank customer
```
```
NAME       TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
customer   NodePort   10.49.50.18   <none>        80:30180/TCP   166m
```

In this example output it is `10.49.50.18`. Your IP may be different.

Confirm you can access it from host1.

```
curl $(kubectl get svc -n yaobank customer --no-headers | awk {'print $3'})
```

### 3.2.2. Advertise local service cluster IP addresses

You can set `externalTrafficPolicy: Local` on a Kubernetes service to request that external traffic to a service should only be routed via nodes which have a local service endpoint (backing pod). This preserves the client source IP and avoids the second hop associated  NodePort or ClusterIP services when kube-proxy loadbalances to a service endpoint (backing pod) on another node.

Traffic to the cluster IP for a service with `externalTrafficPolicy: Local` will be load-balanced across the nodes with endpoints for that service.

#### 3.2.2.1. Add external traffic policy

Update the `customer` service to add `externalTrafficPolicy: Local`.

```
kubectl patch svc -n yaobank customer -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

#### 3.2.2.2. Examine routes

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.232/29 via 10.0.1.30 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
10.49.50.18 via 10.0.1.31 dev ens5 proto bird 
```

You should now have a `/32` route for the yaobank customer service (`10.49.50.18` in the above example output) advertised from the node hosting the customer service pod (worker2, `10.0.1.31` in the example output above).

For each active service with `externalTrafficPolicy: Local`, Calico advertise the IP for that service as a `/32` route from the nodes that have endpoints for that service. This means that external traffic to the service will get load-balanced across all nodes in the cluster that have a service endpoint (backing pod) for the service by the network using ECMP (Equal Cost Multi Path). Kube-proxy then DNATs the traffic to the local backing pod on that node (or load-balances equally to the local backing pods if there is more than one on the node).

The two main advantages of using `externalTrafficPolicy: Local` in this way are:
* There is a network efficiency win avoiding potential second hop of kube-proxy load-balancing to another node.
* The client source IP addresses are preserved, which can be useful if you want to restrict access to a service to specific IP addresses using network policy applied to the backing pods.  (This is an alternative approach to that we will explore later in Lab4.2 where will use Calico host endpoint `preDNAT` policy to restict external traffic to the services.)

### 3.2.2.3. Note the impact on nodePorts

Earlier in these labs we accessed the YAO Bank app using NodePorts in the master node (the command we used for that was `curl 10.0.1.20:30180`). As we've now set externalTrafficPolicy: Local, this will no longer work since there are no customer pods hosted on it. Accessing via the nodePort on the worker the customer pod has been scheduled would work indeed.

### 3.2.3. Advertise service external IP addresses

If you want to advertise a service using an IP address outside of the service cluster IP range, you can configure the service to have one or more `externalIPs`.

#### 3.2.3.1. Examine the existing services

Before we begin, let's examine again the kubernetes services in the `yaobank` namespace.

```
kubectl get svc -n yaobank
```
```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer   NodePort    10.49.50.18     <none>        80:30180/TCP   172m
database   ClusterIP   10.49.238.135   <none>        2379/TCP       172m
summary    ClusterIP   10.49.213.58    <none>        80/TCP         172m
```

Note that none of them currently have an `EXTERNAL-IP`.

#### 3.2.3.2. Update BGP configuration

Update the Calico BGP configuration to advertise a service external IP CIDR range of `10.50.0.0/24`.

```
calicoctl patch BGPConfig default --patch \
   '{"spec": {"serviceExternalIPs": [{"cidr": "10.50.0.0/24"}]}}'
```

Note that `serviceExternalIPs` is a list of CIDRs, so you could for example add individual /32 IP addresses if there were just a small number of specific IPs you wanted to advertise.

#### 3.2.3.2. Examine routes

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.232/29 via 10.0.1.30 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
10.49.50.18 via 10.0.1.31 dev ens5 proto bird 
10.50.0.0/24 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
```

You should now have a route for the external ID CIDR (10.50.0.10/24) with next hops to each of our cluster nodes.

#### 3.2.3.4. Assign the service external IP

Assign the service external IP `10.50.0.10` to the `customer` service.

```
kubectl patch svc -n yaobank customer -p  '{"spec": {"externalIPs": ["10.50.0.10"]}}'
```

Examine the services again to validate everything is as expected:

```
kubectl get svc -n yaobank -l app=customer
```
```
NAME       TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
customer   NodePort   10.49.50.18   10.50.0.10    80:30180/TCP   175m
```

You should now see the external ip (`10.50.0.10`) assigned to the `customer` service.  We can now access the `customer` service from outside the cluster using the external ip address (10.50.0.10) we just assigned.

#### 3.2.3.5. Verify we can access the service's external IP

Connect to the `customer` service from the standalone node using the service external IP `10.50.0.10`.

```
curl 10.50.0.10
```

As you can see the service has been made available outside of the cluster via bgp routing and load balancing.

### 3.2.4. Recap

We've covered five different ways so far in our lab for connecting to your pods from outside the cluster.
* Via a standard NodePort on a specific node.
* Direct to the pod IP address by configuring a Calico IP Pool that is externally routable.
* Advertising the service cluster IP range. (And using ECMP to load balance across all nodes in the cluster)
* Advertising individual cluster IPs. (Services with `externalTrafficPolicy: Local`, using ECMP to load balance only to the nodes hosting the pods backing the service)
* Advertising service external-IPs. (So you can use service IP addresses outside of the cluster IP range)

There are more ways for cluster external connectivity, nonetheless we have covered the most common scenarios. This gives you an idea about Calico's versatility and ability to fit with a broad range of networking needs.
