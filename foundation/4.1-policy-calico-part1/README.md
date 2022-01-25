## 4.1. Policy: Part 1

This is the first lab of a series of labs focusing on Calico k8s network policy. Throughout this lab we will deploy and test our first Calico k8s network policy. 
In this lab we will: \
4.1.1. Verify connectivity from customer pod. \
4.1.2. Apply a simple Calico Policy.

### 4.1.0. Before you begin

Throughout this lab, we will be using the yaobank app we deployed in the previous lab.

### 4.1.1. Verify connectivity from Customer pod

Let's start getting the address of the summary service:

```
kubectl get pod -n yaobank -l app=summary -o wide
```
```
NAME                       READY   STATUS    RESTARTS   AGE    IP            NODE           NOMINATED NODE   READINESS GATES
summary-748b977d44-8vt45   1/1     Running   0          14m   10.48.0.150    ip-10-0-1-31   <none>           <none>
summary-748b977d44-h9gx6   1/1     Running   0          14m   10.48.0.149    ip-10-0-1-31   <none>           <none>
```

And then accessing our customer pod, and checking the connectivity towards any of the summary pods (please note your IP address will be different):

```
kubectl exec -it -n yaobank $(kubectl get pod -n yaobank -l app=customer -o name) -- bash
```
```
	ping -c3 10.48.0.150 >>>>>>>>>>>>>>>>>>>>>> replace the IP of one of your summary pods here
	curl -v telnet://10.48.0.150:80 >>>>>>>>>>> replace the IP of one of your summary pods here
	ping summary
	curl -v telnet://summary:80
	exit
```

You should have the following behaviour:
* ping to the pod ip is successful
* curl to the pod ip is successful
* ping to summary fails
* curl to summary is successful

As we have learned in Lab 3.1, services rely on kube-proxy which load-balances the service request to backing pods. The service is listening to TCP port 80 so the ping failure is expected.

Exit the customer pod, and verify you can reach the customer service through the ingress controller using your browser (`https://<labname>,lynx.tigera.ca`).

### 4.1.2. Apply a simple Calico Policy

Let's limit connectivity on the Nginx pods to only allow inbound traffic to port 80 from the customer pod.
Examine the manifest before applying it:

``` 
cat 4.1-customer2summary.yaml 
```
```
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: customer2summary
  namespace: yaobank
spec:
  order: 500
  selector: app == "summary"
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == "customer"
    destination:
      ports:
      - 80
  types:
    - Ingress
```

The policy allows matches the label app=summary which is assigned to summary pods, and allows TCP port 80 traffic from source matching the label app=customer which is assigned to customer pods.
Let's apply our first Calico network policy and examine the effect.

```
calicoctl apply -f 4.1-customer2summary.yaml 
```

Now, let's repeat the tests we have done in section 4.1.1.
You should have the following behaviour:

* ping to the pod now fails. This is expected since icmp was not allowed in the policy we have applied.
* curl to the pod ip or summary is successful

Let's cleanup the network policy for now.

```
calicoctl delete -f 4.1-customer2summary.yaml 
```
