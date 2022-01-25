
# Install Calico in IPVS mode


Calico has support for kube-proxy’s ipvs proxy mode. Calico ipvs support is activated automatically if Calico detects that kube-proxy is running in that mode.

ipvs mode provides greater scale and performance vs iptables mode. However, it comes with some limitations. In IPVS mode
– Its a good kick-start activity to understand Calico Enterprise capabilities.

## Requirements

1. A cluster running Kubernetes v1.11+
2. Load the below required kernel modules and install `ipvsadm` and `ipset` on all nodes.

## IMPORTANT: This step must not be done in the bastion host as usual, but in all the nodes in the kubernetes cluster (control1, worker1, and worker2):

```
sudo apt install -y ipvsadm ipset
```
Load the modules
```
sudo modprobe ip_vs 
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr 
sudo modprobe ip_vs_sh
sudo modprobe nf_conntrack
sudo sysctl --system
sudo sysctl -p
```
Check loaded modules:
```
lsmod | grep -e ip_vs -e nf_conntrack
```
```
cut -f1 -d " " /proc/modules | grep -e ip_vs -e nf_conntrack
```

## Steps to enable IPVS mode 

1. Change the configMap of kube-proxy, modify "mode" from "" to "ipvs"

```
kubectl -n kube-system edit cm kube-proxy
```

2. Delete all the active proxy pods

```
for i in $(kubectl get pods -n kube-system -o name | grep kube-proxy) ; do kubectl delete $i -n kube-system ; done
```

3. Check the logs of new kube-proxy pods

```
for i in $(kubectl get pods -n kube-system -o name | grep kube-proxy) ; do kubectl logs $i -n kube-system | grep "Using ipvs Proxier" ; done
```

If you are able to find the mentioned String in the logs, IPVS mode is being used by the cluster. You can always see detailed logs for more depth about the IPVS mode.

## Verify and Debug IPVS

Users can use ipvsadm tool to check whether kube-proxy are maintaining IPVS rules correctly. This must be done in any of the kubernetes nodes, and not the bastion host. We will use the kubernetes API server.

```
ssh worker1
```
```
kubectl get svc
```
```
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.49.0.1      <none>        443/TCP   46h
```
```
kubectl get ep
```
```
NAME            ENDPOINTS                                  AGE
kubernetes      10.0.1.20:6443                             46h
```

As the API server has a single endpoint as our cluster has just 1 master, we can grep a single line below the Cluster IP to check the IPVS proxy rules for it:

```
sudo ipvsadm -ln | grep -A1 10.49.0.1:443
```
```
TCP  10.49.0.1:443 rr
  -> 10.0.1.20:6443               Masq    1      0          0         
```

## Why kube-proxy can't start IPVS mode

Use the following check list to help you solve the problems:

1. Specify `mode=ipvs` 

Check whether the kube-proxy mode has been set to ipvs in the `kube-proxy` configmap.

3. Install required kernel modules and packages

Check whether the IPVS required kernel modules have been compiled into the kernel and packages installed. (see Requirements)

## Demo

Considering that you have the cluster running in `ipvs` mode and Calico is now configured, lets us create a `nginx-deployment` and a `service` and observe how `ipvs` loadbalancing works.

```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
EOF
```

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: service-nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
EOF

```
Examine the ClusterIP of `service-nginx` service

```
kubectl get svc
```
```
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.49.0.1      <none>        443/TCP   46h
service-nginx   ClusterIP   10.49.155.85   <none>        80/TCP    7s
```

Now let us list the ipvs table and check how are service maps to the pods created using deployment (this command **must** be run from one of the workers):

```
sudo ipvsadm -l | grep -A3 $(kubectl get svc service-nginx --no-headers | awk {'print $3'})
```
```
TCP  ip-10-49-155-85.ca-central-1 rr
  -> ip-10-48-0-7.ca-central-1.co Masq    1      0          0         
  -> ip-10-48-0-8.ca-central-1.co Masq    1      0          0         
  -> ip-10-48-0-136.ca-central-1. Masq    1      0          0   
```

Here `p-10-49-155-85.ca-central-1` is the service, and the below you can see the list of pods/endpoints, while `rr` means the `loadbalancing` used is round-robin.

The addresses shown below correspond to the endpoints of that service:
```
kubectl get ep
```
```
NAME            ENDPOINTS                                  AGE
kubernetes      10.0.1.20:6443                             46h
service-nginx   10.48.0.136:80,10.48.0.7:80,10.48.0.8:80   108s
```

Check you can access the service in it's Cluster IP address (10.49.155.85 in our example):

```
curl 10.49.155.85
```
```
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
```
