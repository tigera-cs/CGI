## 4.3. Network Policy - Part 3

This is the last lab in a series of labs exploring network policies.

In this lab you will: \
4.3.1. Create Egress Lockdown policy as a Security Admin for the cluster \
4.3.2. Grant selective Internet access \
4.3.3. Protect the Host \
4.3.4. Add Policy for Kubernetes NodePorts

### 4.3.0. Before you begin

This lab builds on the previous lab and cannot be run independently. So if you haven't already done so, please go back and complete lab 4.2.
In the K8s Network Policy deployed in the firts Network Polciy lab , the developer's policy allows all pod egress, even to the Internet.

### 4.3.1. Create Egress Lockdown policy as a Security Admin for the cluster

This lab guides you through the process of deploying a egress lockdown policy in our cluster.
In Lab 4.2, we have applied a policy that allows all pod access. Here we will implement a more restrictive policy that allows minimal access and denies everything else.
Let's first start with verifying connectivity with the configuration of Lab 4.2 already applied.

#### 4.3.1.1. Confirm that pods are able to initiate connections to the Internet

Access the customer pod.

```
CUSTOMER_POD=$(kubectl get pod -l app=customer -n yaobank -o jsonpath='{.items[0].metadata.name}')
```
```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer -- bash
```
```
$ ping -c 3 8.8.8.8
$ curl -I www.google.com
$ exit
```

This succeeds since the policy in place allows full internet access.

Let's now create a Calico GlobalNetworkPolicy to restrict Egress to the Internet to only pods that have the ServiceAccount that is labeled  "internet-egress = allowed".

#### 4.3.1.2. Examine and apply the network policy

Examine the policy before applying it. While Kubernetes network policies only have Allow rules, Calico network policies also support Deny rules. As this policy has Deny rules in it, it is important that we set its precedence higher than the K8s policy Allow rules. To do this we specify `order`  value of 600 in this policy, which gives it higher precedence than the k8s policy (which does not have the concept of policy precedence, and is assigned a fixed order value of 1000 by Calico). 

```
more 4.3-egress-lockdown.yaml
```
```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: egress-lockdown
spec:
  order: 600
  selector: ''
  types:
  - Egress
  egress:
    - action: Allow
      source:
        serviceAccounts:
          selector: internet-egress == "allowed"
      destination: {}
    - action: Deny
      source: {}
      destination:
        notNets:
          - 10.48.0.0/16
          - 10.49.0.0/16
          - 10.50.0.0/24
          - 10.0.1.0/24
```

Notice the notNets destination parameter that excludes known cluster networks from the deny rule. This is another feature specific to calico that allows matching based on ip subnets.
Now let's apply the policy.

```
calicoctl apply -f 4.3-egress-lockdown.yaml
```

#### 4.3.1.3. Verify access the Internet

```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer -- bash
```
```
$ ping -c 3 8.8.8.8
$ curl -I www.google.com
```

These commands should fail - pods are now restricted to only accessing other pods and nodes within the cluster. You may need to terminate the command with CTRL+C and exit back to your node.

```
$ exit
```

### 4.3.2. Grant selective Internet access

Now let's take the case where there is a legitimate reason to allow connections from the Customer pod to the internet. As we used a Service Account label selector in our egress policy rules, we can enable this by adding the appropriate label to the pod's Service Account.

#### 4.3.2.1. Add "internet-egress=allowed" to the Service Account

```
kubectl label serviceaccount -n yaobank customer internet-egress=allowed
```

#### 4.3.2.2. Verify the pod can now access the internet

```
kubectl exec -ti $CUSTOMER_POD -n yaobank -c customer -- bash
```
```
$ ping -c 3 8.8.8.8
$ curl -I www.google.com
$ exit
```

Now you should find that the customer pod is allowed Internet Egress, but other pods (like Summary and Database) are not.

#### 4.3.2.3. Managing trust across teams

There are many ways of dividing responsibilities across teams using Kubernetes RBAC.

Let's take the following case:
* The secops team is responsible for creating Namespaces and Services accounts for dev teams. Kubernetes RBAC is setup so that only they can do this.
* Dev teams are given Kubernetes RBAC permissions to create pods in their Namespaces, and they can use, but not modify any Service Account in their Namespaces.

In this scenario, the secops team can control which teams should be allowed to have pods that access the internet.  If a dev team is allowed to have pods that access the internet then the dev team can choose which pods access the internet by using the appropriate Service Account. 

This is just one way of dividing responsibilities across teams.  Pods, Namespaces, and Service Accounts all have separate Kubernetes RBAC controls and they can all be used to select workloads in Calico network policies.

### 4.3.3. Protect the Host

Thus far, we've created policies that protect pods in Kubernetes. However, Calico Policy can also be used to protect the host interfaces in any standalone Linux node (such as a baremetal node, cloud instance or virtual machine) outside the cluster. Furthermore, it can also be used to protect the Kubernetes nodes themselves.

The protection of Kubernetes nodes themselves highlights some of the unique capabilities of Calico - since this needs to account for various control plane services (such as the apiserver, kubelet, controller-manager, etcd, and others. In addition, one needs to also account for certain pods that might be running with host networking (i.e., using the host IP address for the pod) or using hostports. To add an additional layer of challenge, there are also various services (such as Kubernetes NodePorts) that can take traffic coming to reserved port ranges in the host (such as 30000-32767) and NAT it prior to forwarding to a local destination (and perhaps even SNAT traffic prior to redirecting it to a different worker node). 

Lets explore how Calico policy can handle these scenarios.

#### 4.3.3.1. Observe that Kubernetes Control Plane has been left exposed by Kubeadm

Lets start by seeing how the default cluster deployment have left some control plane services exposed to the world.

Identifiy where etcd is running:

```
kubectl get pod -n kube-system -l component=etcd -o wide
```
```
NAME                READY   STATUS    RESTARTS   AGE   IP          NODE           NOMINATED NODE   READINESS GATES
etcd-ip-10-0-1-20   1/1     Running   0          3d    10.0.1.20   ip-10-0-1-20   <none>           <none>
```

Now, let's try to access it from our bastion host:

```
curl -v 10.0.1.20:2379
```
```
*   Trying 10.0.1.20:2379...
* TCP_NODELAY set
* Connected to 10.0.1.20 (10.0.1.20) port 2379 (#0)
> GET / HTTP/1.1
> Host: 10.0.1.20:2379
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Empty reply from server
* Connection #0 to host 10.0.1.20 left intact
curl: (52) Empty reply from server
```

This should succeed - i.e., the Kubernetes cluster's etcd store is left exposed for attacks, along with the rest of the control plane. This could lead to a compromise if not handled properly.

#### 4.3.3.2. Create Network Policy for Hosts

Lets create a host endpoint policies for the kubernetes master node to provide an example on how calico can enforce the security on those. But first, we must know Calico implements a failsafe polciy to avoid locking down an environment by mistake, this is done through the Felix Configuration. As an example, we will remove the etcd, and ssh ports from the failsafe list, and then apply a polciy to allow only the SSH access, as we have a single master in our cluster, so there is no need to expose etcd port 2379. Let's check our current felix configuration:

```
calicoctl get felixconfig default -o yaml --export 
```

And let's compare it with the yaml manifest we will apply:

```
more 4.3-felix.yaml 
```
```
apiVersion: projectcalico.org/v3
kind: FelixConfiguration
metadata:
  creationTimestamp: null
  name: default
spec:
  bpfLogLevel: ""
  failsafeInboundHostPorts:
  - net: ""
    port: 68
    protocol: udp
  - net: ""
    port: 179
    protocol: tcp
  - net: ""
    port: 2380
    protocol: tcp
  - net: ""
    port: 5473
    protocol: tcp
  - net: ""
    port: 6443
    protocol: tcp
  - net: ""
    port: 6666
    protocol: tcp
  - net: ""
    port: 6667
    protocol: tcp
  logSeverityScreen: Info
  reportingInterval: 0s
```

Implicitly, the failsafe port list includes ports: "tcp:22, udp:68, tcp:179, tcp:2379, tcp:2380, tcp:5473, tcp:6443, tcp:6666, tcp:6667". We intentionally have removed ports tcp:22, and tcp:2379, so all nodes we include in our Host Endpoint (hep) definition will be affected if traffic towards them is not explicitly allowed in a Calico security policy. Let's apply this felix configuration:

```
calicoctl apply -f 4.3-felix.yaml
```

#### 4.3.3.3. Create Host Endpoints

Now, let's proceed to add our master node to the Host Endpoints we want to enforce.

```
more 4.3-host-endpoint-master.yaml 
```
```
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: master1
  labels:
    node-role.kubernetes.io/master: ""
spec:
  interfaceName: ens5
  node: ip-10-0-1-20
  expectedIPs: ["10.0.1.20"]
```
```
calicoctl apply -f 4.3-host-endpoint-master.yaml
```

One thing important to note is we are defining enforcement on a specific interface. We should see now this node in our Host Endpoint list using the default profile as we have not defined any:

```
calicoctl get hep -o wide
```
```
NAME      NODE           INTERFACE   IPS         PROFILES   
master1   ip-10-0-1-20   ens5        10.0.1.20              
```


#### 4.3.3.4. Now lets attempt to attack etcd from the standalone host. Run the curl again from the bastion host:

```
curl -v 10.0.1.20:2379
```
```
*   Trying 10.0.1.20:2379...
* TCP_NODELAY set
^C
```

We have successfully locked down etcd in the Kubernetes control plane, so it is not accessible from unrelevant nodes.

Similarly, traffic to the SSH port should get through only towards the worker nodes (as they are not defined as Host Endpoints, so they are not being enforced), however, SSH traffic must be blocked towards our master:

```
nc -ztv 10.0.1.30 22
```
```
Connection to 10.0.1.30 22 port [tcp/ssh] succeeded!
```
```
nc -ztv 10.0.1.31 22
```
```
Connection to 10.0.1.31 22 port [tcp/ssh] succeeded!
```
```
nc -ztv 10.0.1.20 22
```
```
^C
```

Now, let's allow again SSH traffic towards our master node by means of a Calico Network Polciy rule:

```
more 4.3-global-host-policy-master.yaml 
```
```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: k8s-outside2master-allowed-ports
spec:
  selector: node-role.kubernetes.io/master == ""
  order: 300
  ingress:
  - action: Allow
    protocol: TCP
    destination:
      ports: ["22", 4001, "10250:10256", 9099, 6443 ]
  egress:
  - action: Allow
```

```
calicoctl apply -f 4.3-global-host-policy-master.yaml
```

You should be able to ssh now:

```
nc -ztv 10.0.1.20 22
```
```
Connection to 10.0.1.20 22 port [tcp/ssh] succeeded!
```
```
ssh control1
exit
```
