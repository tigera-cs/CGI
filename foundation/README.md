Training labs demonstrate the working of Calico CNI, K8S networking, services and policy.

 – Its a good kickstart activity to understand Calico capabilities.

 Requirements

#### K8S cluster with sufficient CPU, storage and memory.

##### Environment specifications

Calico Lab Hosts

IP Address Node/Purpose

##### 10.0.1.10 Bastion Host (BGP Peer and Jump Server)

##### 10.0.1.20 Kubernetes Master (Control1)

##### 10.0.1.30 Kubernetes Worker (Worker1)

##### 10.0.1.31 Kubernetes Worker (Worker2)

#### Calico Lab Networks

##### CIDR Purpose

10.48.0.0/16 Kubernetes Pod Network (via kubeadm --pod-network-cidr)

10.49.0.0/16 Kubernetes Service CIDR (via kubeadm --service-cidr)

### Introduction to Labs

Lab1: Installing Calico CNI on K8S cluster.


Lab2: Understanding basics of Pod networking, IP-pool, IP address assignment to Pods.


Lab3: Understaning the working of Service in Kubernetes, exploration of Iptables chains & rules, advertising Service IP address and exposing the service using Ingress controller.


Lab4: This lab focuses on Network policy which enables/disables/partially-enable communication accross k8S endpoints.

- marker added -
