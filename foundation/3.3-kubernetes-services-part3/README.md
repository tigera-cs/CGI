## 3.3. Kubernetes Service - Ingress

This is the 3rd of a series of labs about k8s services. This lab exposes the Yaobank Customer service to the outside via ingress controller. In this lab, you will: 

3.3.1. Remove previous Yaobank deployment. \
3.3.2. Deploy an ingress controller that listens to all namespaces. \
3.3.3. Deploy an updated Yaobank manifest including ingress.

### 3.3.0. Before you begin

This lab leverages the lab deployment we have developed so far, except for the yaobank application, so you must complete previous labs before continuing.

### 3.3.1. Remove previous Yaobank deployment

In this lab, we will be exposing the Yaobank Customer service using a ingress controller. Let's start with removing the previous Yaobank deployment and proceed to deploying the new configuration. For simplicity, let's just remove the namespace which deletes all included objects (this could take a bit of time, please be patient, and wait for the prompt to return. If it does not come back in a couple of minutes, press "Enter" to make sure it has not finished already).

```
kubectl delete ns yaobank
```

### 3.3.2. Deploy an ingress controller that listens to all namespaces

Ingress is the built-in kubernetes framework for load-balancing http traffic. Cloud providers offer a similar functionality out of the box via cloud load-balancers. Ingress allows the manipulation of incoming http requests, natting/routing traffic to back-end services based on provided host/path or even passing-through traffic. It can effectively provide L7-based policies and typical load-balancing features such as stickiness, health probes or weight-based load-balancing.

Let's start with examining the already predeployed ingress controller:

```
kubectl get all -n ingress-nginx
```
```
NAME                                       READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-n4fc4   0/1     Completed   0          44m
pod/ingress-nginx-admission-patch-mlkkp    0/1     Completed   0          44m
pod/ingress-nginx-controller-9l9kr         1/1     Running     0          40m
pod/ingress-nginx-controller-f8qdh         1/1     Running     0          40m

NAME                                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/ingress-nginx-controller-admission   ClusterIP   10.49.247.30   <none>        443/TCP   44m

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/ingress-nginx-controller   2         2         2       2            2           kubernetes.io/os=linux   44m

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           4m27s      44m
job.batch/ingress-nginx-admission-patch    1/1           4m27s      44m
```

Key things to look in the output are:
* The ingress has been deployed as a DaemonSet to be run on each of the worker nodes, so you must have two of them running. Note thera are two other containers with state of completed, as those are init containers which were used to configure and patch the ingress pods at the time of deployment.
* The ingress hooks in port 443 of our nodes, so if we reach the workers through a web browser, they will redirect traffic to our applications based on the rules we configure for our ingress services. You can check its config with the command below:

```
kubectl describe service/ingress-nginx-controller-admission -n ingress-nginx
```
```
Name:              ingress-nginx-controller-admission
Namespace:         ingress-nginx
Labels:            app.kubernetes.io/component=controller
                   app.kubernetes.io/instance=ingress-nginx
                   app.kubernetes.io/name=ingress-nginx
                   app.kubernetes.io/version=0.46.0
                   helm.sh/chart=ingress-nginx-3.30.0
Annotations:       <none>
Selector:          app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
Type:              ClusterIP
IP Families:       <none>
IP:                10.49.247.30
IPs:               10.49.247.30
Port:              https-webhook  443/TCP
TargetPort:        webhook/TCP
Endpoints:         10.0.1.30:8443,10.0.1.31:8443
Session Affinity:  None
Events:            <none>
```

By default the ingress controller will listen to all namespaces, and service ingress objects created on them so it will programmed with the needed rules to forward our traffic. Currently we have not created any, so if we try to access our lab in port 443 using our browser (`https:labname.lynx.tigera.ca`), we will get a 404 error from our ingress controller.

### 3.3.3. Deploy an updated Yaobank manifest including ingress

Let's check our modified yaobank application which includes an ingress resource:

```
cat 3.3-yaobank.yaml | tail -20
```
```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: ingress-yaobank-customer
  namespace: yaobank
spec:
  rules:
  - host: '*.lynx.tigera.ca'
    http:
      paths:
      - backend:
          service:
            name: customer
            port:
              number: 80
        path: /
        pathType: Prefix
```    

The rule we are using has a wildcard that will match any host in our domain for a conenction attempted to port 443 in any of our worker nodes, and it will terminate the HTTPS connection, and redirect the traffic cleartext to port 80 in our 'customer' application. Let's apply our manifest:

```
kubectl apply -f 3.3-yaobank.yaml
```

...and verify our ingress has been successfully deployed:

```
kubectl get ingress -n yaobank
```
```
NAME                       CLASS    HOSTS              ADDRESS   PORTS   AGE
ingress-yaobank-customer   <none>   *.lynx.tigera.ca             80      8s
```

Note we do not use a NodePort service anymore for our customer application in this last manifest, as now we will use the ingress to access it:

```
kubectl get svc -n yaobank
```
```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
customer   ClusterIP   10.49.112.223   <none>        80/TCP     11m
database   ClusterIP   10.49.177.243   <none>        2379/TCP   11m
summary    ClusterIP   10.49.180.235   <none>        80/TCP     11m
```

### 3.3.3. Verify connectivity

Now if we access our lab in port 443 using our browser (`https:labname.lynx.tigera.ca`), we must get a Welcome page from our application.
