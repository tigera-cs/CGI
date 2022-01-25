## 2.2. Networking: Calico-IPAM Lab

This lab will explore k8s ip adress management via Calico IPAM.

In this lab, you will: \
2.2.1. Check existing IP Pools, and create new IP Pools. \
2.2.2. Update the yaobank deployments with the new IP Pools. \
2.2.3. Verify host routing.

### 2.2.0. Before you begin

This lab builds in the learning in previous labs. You should have by now your k8s cluster setup with Calico and the Yaobank sample application deployed. 

### 2.2.1. Check existing IP Pools  and create new IP Pools

As we checked in the previous lab range is 10.48.0.0/24, which is actually Calico's Initial default IP Pool range of our k8s cluster.

```
calicoctl get ippool
```
```
NAME                  CIDR           SELECTOR   
default-ipv4-ippool   10.48.0.0/24   all()      
```
Examine the content of the manifest 2.2-ippools.yaml

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pool2-ipv4-ippool
spec:
  blockSize: 26
  cidr: 10.48.128.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
```

You can see that we have effectively broken the default ippool into to equal subnets. 
Apply the manifest and verify the output.

```
calicoctl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pool2-ipv4-ippool
spec:
  blockSize: 26
  cidr: 10.48.128.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
EOF
```

```
calicoctl get ippools
```
```
NAME                  CIDR             SELECTOR   
default-ipv4-ippool   10.48.0.0/24     all()      
pool2-ipv4-ippool     10.48.128.0/24   all()    
```

### 2.2.2. Update the yaobank deployments with the new IP Pools

Calico supports annotations on both namespaces and pods that can be used to control which IP Pool (or even which IP address) a pod will receive when it is created. 

There is a new version of the yaobank manifest file (`2.2-yaobank-ipam.yaml`), so we added the necessary annotation to use the created new pool when deploying. You can modify the existing deployment if you wish, but for convenience, we can just delete the old deployment, and use that new manifest. The change is shown below:

```
    metadata:
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"pool2-ipv4-ippool\"]"

```

The annotation explicitly assigns pool2 to summary and database deployments. Before applying the manifest, let's check the current assigned IPs to compare later:

```
kubectl get pod -n yaobank -o wide
```
```
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
customer-787758576-f9mxf    1/1     Running   0          89m   10.48.0.133   ip-10-0-1-31   <none>           <none>
database-64bfcc464d-4vjx6   1/1     Running   0          89m   10.48.0.3     ip-10-0-1-30   <none>           <none>
summary-748b977d44-9qcm6    1/1     Running   0          89m   10.48.0.131   ip-10-0-1-31   <none>           <none>
summary-748b977d44-fv42t    1/1     Running   0          89m   10.48.0.132   ip-10-0-1-31   <none>           <none>
```

Now let's delete the old deployment (this can take a bit of time, so please be patient and wait until the prompt returns).

```
kubectl delete ns yaobank
```
```
namespace "yaobank" deleted
```

Now implement the manifest with the changes in the annotations, and check the pods. Note the customer pod retained the old pool IP, while summary and database deployments show an IP address from the new pool we created. 

```
kubectl apply -f -<<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: yaobank
  labels:
    istio-injection: disabled

---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: yaobank
  labels:
    app: database
spec:
  ports:
  - port: 2379
    name: http
  selector:
    app: database

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database
  namespace: yaobank
  labels:
    app: yaobank

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: yaobank
spec:
  selector:
    matchLabels:
      app: database
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: database
        version: v1
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"pool2-ipv4-ippool\"]"
    spec:
      serviceAccountName: database
      containers:
      - name: database
        image: calico/yaobank-database:certification
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 2379
        command: ["etcd"]
        args:
          - "-advertise-client-urls"
          - "http://database:2379"
          - "-listen-client-urls"
          - "http://0.0.0.0:2379"
      nodeSelector:
        kubernetes.io/hostname: "ip-10-0-1-30.ca-central-1.compute.internal"

---
apiVersion: v1
kind: Service
metadata:
  name: summary
  namespace: yaobank
  labels:
    app: summary
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: summary
    
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: summary
  namespace: yaobank
  labels:
    app: yaobank
    database: reader
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: summary
  namespace: yaobank
spec:
  replicas: 2
  selector:
    matchLabels:
      app: summary
      version: v1
  template:
    metadata:
      labels:
        app: summary
        version: v1
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"pool2-ipv4-ippool\"]"
    spec:
      serviceAccountName: summary
      containers:
      - name: summary
        image: calico/yaobank-summary:certification
        imagePullPolicy: Always
        ports:
        - containerPort: 80
 
---
apiVersion: v1
kind: Service
metadata:
  name: customer
  namespace: yaobank
  labels:
    app: customer
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30180
    name: http
  selector:
    app: customer
    
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: customer
  namespace: yaobank
  labels:
    app: yaobank
    summary: reader
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customer
  namespace: yaobank
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customer
      version: v1
  template:
    metadata:
      labels:
        app: customer
        version: v1
    spec:
      serviceAccountName: customer
      containers:
      - name: customer
        image: calico/yaobank-customer:certification
        imagePullPolicy: Always
        ports:
        - containerPort: 80
      nodeSelector:
        kubernetes.io/hostname: "ip-10-0-1-31.ca-central-1.compute.internal"
---
EOF
```
```
kubectl get pod -n yaobank -o wide
```
```
NAME                        READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
customer-787758576-sz9w6    1/1     Running   0          3m55s   10.48.0.134     ip-10-0-1-31   <none>           <none>
database-67d9fdffc8-q4m6n   1/1     Running   0          3m56s   10.48.128.0     ip-10-0-1-30   <none>           <none>
summary-dc858dd7b-c7jxl     1/1     Running   0          3m55s   10.48.128.129   ip-10-0-1-31   <none>           <none>
summary-dc858dd7b-tjt99     1/1     Running   0          3m55s   10.48.128.128   ip-10-0-1-31   <none>           <none>
```

Calico IPAM provides the flexibility as well of assigning ippools to namespaces or even in alignment with your topology to specific nodes or racks.

### 2.2.3. Verify host routing

These changes will be reflected automatically in our routing, as the nodes will advertise /26 blocks for this new pool. Below we can see the routing table from the master node perspective:

```
ssh control1
```
```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.20 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.20 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.20 metric 100 
10.48.0.0/26 via 10.0.1.30 dev ens5 proto bird 
10.48.0.128/26 via 10.0.1.31 dev ens5 proto bird 
blackhole 10.48.0.192/26 proto bird 
10.48.0.193 dev cali0ac1d0a8afc scope link 
10.48.128.0/26 via 10.0.1.30 dev ens5 proto bird 
10.48.128.128/26 via 10.0.1.31 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```

Examine the output of the routing table that is relevant to our deployment:
* specific routes point to the veth interfaces connecting to local pods
* blackhole route is created for the ip block local to the host (saying do route traffic for the local/26 block to any other node)
* routes to the pod on workers are learned via BPG and a /26 block advertisement from the nodes

When finished, exit from the master node so you return to the bastion host:

```
exit
```
