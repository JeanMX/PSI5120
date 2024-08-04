# Execution Script - Intermediate Assessment PSI5120

This work aims to implement a web server in a Kubernetes cluster with automatic horizontal auto-scaling. Firstly, we will implement using minikube and after that, we will 

## Summary

### 1. Executing Kubernetes Cluster using Minikube

1.1 [Installing the metrics-server](#1.1)

1.2 [Starting Minikube and enabling metrics-server](#1.2)

1.3 [Running and exposing php-apache server](#1.3)

1.4 [Creating the HorizontalPodAutoscaler](#1.4)

1.5 [Increasing the load](#1.5)

1.6 [Stopping the load](#1.6)

### 2. AWS EKS Cluster

2.1 [Creating EKS Cluster](#2.1)

2.2 [Running HorizontalPodAutoscaler](#2.2)

## 1. Executing Kubernetes Cluster using Minikube <a name="1."></a>

## Prerequisites

- [Docker](https://docs.docker.com/engine/install/) 

- [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download)

- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) (version 1.23 or higher)

### 1.1 Installing the metrics-server <a name="1.1"></a>

Firstly, delete the existing Metrics Server if there are any.

```console
$ kubectl delete -n kube-system deployments.apps metrics-server
```

Install from GitHub with the follow command.

```console
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
```

For secure communication, it is necessary to edit the components.yaml file. Select the "Deployment" part in the file. Add the line below to the "containers.args" field under the deployment object.


```console
- --kubelet-insecure-tls
```

After the modification, the file will look like as following:

```console
spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
```

Adding "metrics-server" to your Kubernetes instance.

```console
kubectl apply -f components.yaml
```

After 1-2 minutes, verify the existence of metrics-server, running the command:

```console
$ kubectl -n kube-system get pods
```

To verify that the "metrics-server" can access the resources of the pods and nodes, you can run the following commands:

```console
$ kubectl top pods
$ kubectl top nodes
```

If the value metrics, under the current, is a number and not unknown, so it is correct.

### 1.2 Starting minikube and enable metrics-server <a name="1.2"></a>

Now, it is necessary to start minikube. It is necessary at least two nodes. So, you need to use the following command:

```console
$ minikube start --nodes 2 -p multinode-demo
```

To get the list of nodes, you can use the command:

```console
$ kubectl get nodes
```

To enable metrics-server, you should use the following command:

```console
$ minikube addons enable metrics-server
```
### 1.3 Running and expose php-apache server <a name="1.3"></a>

To demonstrate a HorizontalPodAutoscaler, you will first start a Deployment that runs a container using the hpa-example image, and expose it as a Service using the following manifest:

Run the following command to runs a container using the hpa-example image, and expose it as a Service:

```console
$ kubectl apply -f https://k8s.io/examples/application/php-apache.yaml
```

You can see the php-apache.yaml in this repository.

### 1.4 Creating the HorizontalPodAutoscaler <a name="1.4"></a>

After the server is running, it is necessary to create the autoscaler using kubectl. We will create the HorizontalPodAutoscaler running the command below:

```console
$ kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

You can check the current status of the HorizontalPodAutoscaler, by the command:

```console
$ kubectl get hpa
```
The output will be something like below:


```console
NAME         REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   cpu: 0%/50%   1         10        1          159m
```
### 1.5 Increase the load <a name="1.5"></a>

To see the autoscaler reacts, you will open a different terminal and you will run the command below:

```console
$ kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

After a minute you can run the command:

```console
$ kubectl get hpa php-apache --watch
```
You will see below the "TARGET" the highest percentage and the "REPLICAS" number will increase as well.

### 1.6 Stopping the load <a name="1.6"></a>

In the terminal that you executed the load generator, you need to press \<ctrl> + c to stop.

Then you can run the command:

```console
$ kubectl get hpa php-apache --watch
```
Then you will see below the "TARGET" percentage decreasing to zero.

## 2. AWK EKS Cluster <a name="2."></a>

## Prerequisites


- [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html#eksctl-install-update)

- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) (version 1.23 or higher)

- [Installing AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

- [Set up AWS CLI](https://docs.aws.amazon.com/eks/latest/userguide/install-awscli.html)

- [Create EKS Cluster](#2.1)

- [Metrics Server](#1.2)

### 2.1 Creating EKS Cluster <a name="2.1"></a>

Firstly, you need to create a role. On your AWS account search for **Elastic Kubernetes Service**. Click on "Add cluster" and set it as below:

- Cluster name: eks_cluster
- Network VPC: Default
- Subnets: east-us-1a, east-us-1b, east-us-1c
- Security group: Default

Now, we will create a group of nodes. On the cluster that you created right now, goes to "Compute", click on "Add node group" and set it as below:

- Name: eks_node_group_1
- Node IAM role: Creates one with the name eks_workers_role
- Instance type: t3.medium

After that, we will set the cluster. Use the sequence of commands below.

```console
$ aws sts get-caller-identity
$ aws eks update-kubeconfig --region us-east-1 --name eks_cluster
```

It is necessary to set the nginx. Inside this repository, there is nginx-deployment.yaml file. To deploy, use the command:

```console
$ kubectl apply -f nginx-deployment.yaml
```

Run the below command to expose the Nginx deployment as a LoadBalancer service.

```console
$ kubectl expose deployment nginx-deployment --name=nginx-service --port=80 --target-port=80 --type=LoadBalancer
```

To see information about the Nginx service with LoadBalancer, execute the command below:

```console
$ kubectl get service nginx-service
```
### 2.2 Running HorizontalPodAutoscaler <a name="2.2"></a>

After set up the Cluster EKS on your local device, you can do the same procedure from topic [1.3 Running and expose php-apache server](#1.3)


### REFERENCE

Kubernetes Documentation. (n.d.). Horizontal Pod Autoscale Walkthrough. Retrieved August 4, 2024, from https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

Amazon Web Services. (n.d.). Horizontal Pod Autoscaler. Retrieved August 4, 2024, from https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html

Makkaya, C. (2020, July 22). Kubernetes: Creating and testing a Horizontal Pod Autoscaling (HPA) in Kubernetes cluster. Medium. Retrieved August 4, 2024, from https://cmakkaya.medium.com/kubernetes-creating-and-testing-a-horizontal-pod-autoscaling-hpa-in-kubernetes-cluster-548f2378f0c3
