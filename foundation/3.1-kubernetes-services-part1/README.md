## 3.1. Kubernetes Services - Part 1

This is the 1st lab in a series of labs about k8s services. This lab explores the concepts related to k8s services and the drills down into the iptables chains that enable Kube Proxy to deliver the service.
In this lab, you will: \
3.1.1. Examine kubernetes service \
3.1.2. Explore ClusterIP iptables rules \
3.1.3. Explore NodePort iptables rules

### 3.1.0. Before you begin

This lab builds on the setup of previous labs. If you haven't completed them, please go back and ensure that your cluster is setup with k8s CNI and that Yaobank application is deployed.

### 3.1.1. Examine kubernetes services

We will run this lab from `worker1` so we can explore the iptables rules that kube-proxy has set up.

```
ssh worker1
```

Let's take a look at the services in the `yaobank` kubernetes namespace.

#### 3.1.1.1. List services

```
kubectl get svc -n yaobank
```
```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer   NodePort    10.49.50.18     <none>        80:30180/TCP   142m
database   ClusterIP   10.49.238.135   <none>        2379/TCP       142m
summary    ClusterIP   10.49.213.58    <none>        80/TCP         142m
```

We should have three services deployed. One `NodePort` service and two `ClusterIP` services.

#### 3.1.1.2. List service endpoints

Find the endpoints for each of the services.

```
kubectl get endpoints -n yaobank
```
```
NAME       ENDPOINTS                           AGE
customer   10.48.0.134:80                      122m
database   10.48.128.0:2379                    122m
summary    10.48.128.128:80,10.48.128.129:80   122m
```

You can see that the IP addresses listed as the service endpoints in the previous step map to the backing pods:

```
kubectl get pods -n yaobank -o wide
```
```
NAME                        READY   STATUS    RESTARTS   AGE    IP              NODE           NOMINATED NODE   READINESS GATES
customer-787758576-sz9w6    1/1     Running   0          122m   10.48.0.134     ip-10-0-1-31   <none>           <none>
database-67d9fdffc8-q4m6n   1/1     Running   0          122m   10.48.128.0     ip-10-0-1-30   <none>           <none>
summary-dc858dd7b-c7jxl     1/1     Running   0          122m   10.48.128.129   ip-10-0-1-31   <none>           <none>
summary-dc858dd7b-tjt99     1/1     Running   0          122m   10.48.128.128   ip-10-0-1-31   <none>           <none>
```

Each service is backed by one or more pods spread across the worker nodes in our cluster.

### 3.1.2. Explore ClusterIP iptables rules

Let's explore the iptables rules that implement the `summary` service.

#### 3.1.2.1. Get service endpoints

From the endpoints output in 3.1.1.2  we saw two different endpoints for the `summary` `ClusterIP` service.

```
kubectl get endpoints -n yaobank summary
NAME      ENDPOINTS                           AGE
summary   10.48.128.128:80,10.48.128.129:80   127m
```

Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to these endpoint IP addresses.

#### 3.1.2.2. Examine the KUBE-SERVICE chain

```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES
```
```
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.213.58         /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-OIQIZJVJK6E34BR4  tcp  --  *      *       0.0.0.0/0            10.49.213.58         /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  udp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.198.194        /* calico-system/calico-typha:calico-typha cluster IP */ tcp dpt:5473
    0     0 KUBE-SVC-RK657RLKDNVNU64O  tcp  --  *      *       0.0.0.0/0            10.49.198.194        /* calico-system/calico-typha:calico-typha cluster IP */ ...
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.238.135        /* yaobank/database:http cluster IP */ tcp dpt:2379
    0     0 KUBE-SVC-AE2X4VPDA5SRYCA6  tcp  --  *      *       0.0.0.0/0            10.49.238.135        /* yaobank/database:http cluster IP */ tcp dpt:2379
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.50.18          /* yaobank/customer:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-PX5FENG4GZJTCELT  tcp  --  *      *       0.0.0.0/0            10.49.50.18          /* yaobank/customer:http cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            10.49.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
  654 41144 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

Each iptables chain consists of a list of rules that are executed in order until a rule matches. The key columns/elements to note in this output are:
* `target` - which chain iptables will jump to if the rule matches
* `prot` - the protocol match criteria
* `source`, and `destination` - the source and destination IP address match criteria
* the comments that kube-proxy inculdes
* the additional match criteria at the end of each rule - e.g `dpt:80` that specifies the destination port match

#### 3.1.2.3. KUBE-SERVICES -> KUBE-SVC-XXXXXXXXXXXXXXXX

Now let's look more closely at the rules for the `summary` service.

```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep -E summary
```
```
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.213.58         /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-OIQIZJVJK6E34BR4  tcp  --  *      *       0.0.0.0/0            10.49.213.58         /* yaobank/summary:http cluster IP */ tcp dpt:80
```

The second rule directs traffic destined for the summary service clusterIP (`10.49.213.58` in the example output) to the chain that load balances the service (KUBE-SVC-XXXXXXXXXXXXXXXX).

#### 3.1.2.4. KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX

`kube-proxy` in `iptables` mode uses a randomized equal cost selection algorithm to load balance traffic between pods.  We currently have two `summary` pods.

Let's examine how this loadbalancing works. (Remember your chain name may be different than this example.)

```
sudo iptables -v --numeric --table nat --list KUBE-SVC-OIQIZJVJK6E34BR4
```
```
Chain KUBE-SVC-OIQIZJVJK6E34BR4 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-3XJTYYRQCZK5MWPC  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/summary:http */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-6ZQPIL5J2ULNLO5G  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/summary:http */
```

Notice that `kube-proxy` is using the `iptables` `statistic` module to set the probability for a packet to be randomly matched.  Make sure you scroll all the way to the right to see the full output.

The first rule directs traffic destined for the `summary` service to the chain that delivers packets to the first service endpoint (KUBE-SEP-3XJTYYRQCZK5MWPC) with a probability of 0.50000000000. The second rule unconditionally directs to the second service endpoint chain (KUBE-SEP-6ZQPIL5J2ULNLO5G). The result is that traffic is load balanced across the service endpoints equally (on average).

If there were 3 service endpoints then the first chain matches would be probability 0.33333333, the second probability 0.5, and the last unconditional. The result is each service endpoint receives a third of the traffic (on average).

#### 3.1.2.5. KUBE-SEP-XXXXXXXXXXXXXXXX -> `summary` pod

Let's look at one of the service endpoint chains. (Remember your chain names may be different than this example.)

```
sudo iptables -v --numeric --table nat --list KUBE-SEP-6ZQPIL5J2ULNLO5G
```
```
Chain KUBE-SEP-6ZQPIL5J2ULNLO5G (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.48.128.129        0.0.0.0/0            /* yaobank/summary:http */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/summary:http */ tcp to:10.48.128.129:80
```

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (`10.48.128.129` in this example). After this, standard Linux routing can handle forwarding the packet like it would for any other packet.

### 3.1.2.6. Recap

You've just traced the kube-proxy iptables rules used to load balance traffic to `summary` pods exposed as a service of type `ClusterIP`.

In summary, for a packet being sent to a clusterIP:
* The KUBE-SERVICES chain matches on the clusterIP and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

### 3.1.3. Explore NodePort iptables rules

Let's explore the iptables rules that implement the `customer` service.

#### 3.1.3.1. Get service endpoints
Find the service endpoints for `customer` `NodePort` service.

```
kubectl get endpoints -n yaobank customer
```
```
NAME       ENDPOINTS        AGE
customer   10.48.0.134:80   143m
```

The `customer` service has one endpoint (`10.48.0.134` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to this endpoint IP address.

### 3.1.3.2. KUBE-SERVICES -> KUBE-NODEPORTS

The `KUBE-SERVICE` chain handles the matching for service types `ClusterIP` and `LoadBalancer`. At the end of `KUBE-SERVICE` chain, another custom chain `KUBE-NODEPORTS` will handle traffic for service type `NodePort`.

```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep KUBE-NODEPORTS
```
```
 2028  128K KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

`match dst-type LOCAL` matches any packet with a local host IP as the destination. I.e. any address that is assigned to one of the host's interfaces.

### 3.1.3.3. KUBE-NODEPORTS -> KUBE-SVC-XXXXXXXXXXXXXXXX

```
sudo iptables -v --numeric --table nat --list KUBE-NODEPORTS
```
```
Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */ tcp dpt:30180
    0     0 KUBE-SVC-PX5FENG4GZJTCELT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */ tcp dpt:30180
```

The second rule directs traffic destined for the `customer` service to the chain that load balances the service (KUBE-SVC-PX5FENG4GZJTCELT in the example above). `tcp dpt:30180` matches any packet with the destination port of tcp 30180 (the node port of the `customer` service).

#### 3.1.3.4. KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX
(Remember your chain name may be different than this example.)

```
sudo iptables -v --numeric --table nat --list KUBE-SVC-PX5FENG4GZJTCELT
```
```
Chain KUBE-SVC-PX5FENG4GZJTCELT (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-VX6ED4DV5PMQKJAE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */
```

As we only have a single backing pod for the `customer` service, there is no loadbalancing to do, so there is a single rule that directs all traffic to the chain that delivers the packet to the service endpoint (KUBE-SEP-XXXXXXXXXXXXXXXX).

### 3.1.3.5. KUBE-SEP-XXXXXXXXXXXXXXXX -> `customer` endpoint
(Remember your chain name may be different than this example.)

```
sudo iptables -v --numeric --table nat --list KUBE-SEP-VX6ED4DV5PMQKJAE
```
```
Chain KUBE-SEP-VX6ED4DV5PMQKJAE (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.48.0.134          0.0.0.0/0            /* yaobank/customer:http */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */ tcp to:10.48.0.134:80
```

This rule delivers the packet to the `customer` service endpoint.

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (`10.48.0.134` in this example). After this, standard Linux routing can handle forwarding the packet like it would for any other packet.

Remember to exit from the worker node, and return to the bastion host:

```
exit
```

### 3.1.3.6. Recap

You've just traced the kube-proxy iptables rules used to load balance traffic to `customer` pods exposed as a service of type `NodePort`.

In summary, for a packet being sent to a NodePort:
* The end of the KUBE-SERVICES chain jumps to the KUBE-NODEPORTS chain
* The KUBE-NODEPORTS chaing matches on the NodePort and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).
