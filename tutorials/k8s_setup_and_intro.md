# Kubernetes Setup and Introduction

Like the Linux OS, Kubernetes (or shortly, k8s) is shipped by [many different distributions](https://nubenetes.com/matrix-table/#), each aimed for a specific purpose.
Throughout this course we will be working [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine), which is a Kubernetes cluster managed by GCP.  

## Provision a cluster


> [!NOTE]
> At any step, you can refer to the [official documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster) for more information.

To create a zonal cluster with the Google Cloud console, perform the following tasks:

1. Go to the [Google Kubernetes Engine page](https://console.cloud.google.com/kubernetes/list) in the Google Cloud console.
2. Click **+ Create**.
3. In the top right menu, choose **Switch to Standard Cluster** to open the standard cluster wizard. 
4. In the **Cluster basics** section, complete the following:
   - Enter the **Name** for your cluster, e.g. `john-k8s`.
   - For the **Location** type, select **Zonal**, and then select the `us-central1-c` zone.
   - Select a [release channel](https://cloud.google.com/kubernetes-engine/docs/concepts/release-channels) for GKE to pick versions for your cluster. Choose the **Regular** channel. 
     Versions in the Regular channel have been qualified over a longer period. They offer a balance of feature availability and release stability.
   - Keep the default version for the control plane.
5. From the navigation pane, under **Node Pools**, click **default-pool**.
6. In the **Node pool details** section, complete the following:
   - Enter a **Name** for the default [Node pool](https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools).
   - Optional: Choose the Node version.
   - Set the **Number of nodes** to **1**.
   - Since we used a release channel, automatic upgrade for nodes is enabled. 
7. From the navigation pane, under **Node Pools**, click **Nodes**.
   - In the **Image type** drop-down list, use the **Container-Optimized OS with containerd**. This is a VM image managed by Google and optimized for k8s cluster nodes. The container runtime would be [containerd](https://containerd.io/), not Docker.
   - For **Machine type**, choose the `e2-standard-4` type (4vCPU, 16GB RAM).
   - For **Boot disk size**, enter **50GB** of disk per node.
8. Click **Create**.


After you create a cluster, you need to [configure `kubectl`](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#generate_kubeconfig_entry) before you can interact with the cluster from the command line.

1. Initialize the `gcloud` CLI (sets up authentication, project, and configuration):

```bash
gcloud init
```

2. Authenticate with Google Cloud (log in to your Google account):

```bash
gcloud auth login
```

3. Retrieve and configure cluster credentials for `kubectl` to access the cluster:

```bash
gcloud components install kubectl
gcloud components install gke-gcloud-auth-plugin
gcloud container clusters get-credentials CLUSTER_NAME  --project=qodoai --region REGION_CODE
```

4. To make sure you can communicate with the cluster: 

```console
$ kubectl get nodes
NAME                                         STATUS   ROLES    AGE    VERSION
gke-john-test-default-pool-4c95604e-6g57     Ready    <none>   118s   v1.31.4-gke.1372000
```

## Deploy application in the cluster

Let's see Kubernetes cluster in all his glory! 

**Online Boutique** is a microservices demo application, consists of an 11-tier microservices.
The application is a web-based e-commerce app where users can browse items, add them to the cart, and purchase them.

Here is the app architecture and description of each microservice:

![k8s_online-boutique-arch][k8s_online-boutique-arch]


| Service                                              | Language      | Description                                                                                                                       |
| ---------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| frontend                           | Go            | Exposes an HTTP server to serve the website. Does not require signup/login and generates session IDs for all users automatically. |
| cartservice                     | C#            | Stores the items in the user's shopping cart in Redis and retrieves it.                                                           |
| productcatalogservice | Go            | Provides the list of products from a JSON file and ability to search products and get individual products.                        |
| currencyservice             | Node.js       | Converts one money amount to another currency. Uses real values fetched from European Central Bank. It's the highest QPS service. |
| paymentservice               | Node.js       | Charges the given credit card info (mock) with the given amount and returns a transaction ID.                                     |
| shippingservice             | Go            | Gives shipping cost estimates based on the shopping cart. Ships items to the given address (mock)                                 |
| emailservice                   | Python        | Sends users an order confirmation email (mock).                                                                                   |
| checkoutservice             | Go            | Retrieves user cart, prepares order and orchestrates the payment, shipping and the email notification.                            |
| recommendationservice | Python        | Recommends other products based on what's given in the cart.                                                                      |
| adservice                         | Java          | Provides text ads based on given context words.                                                                                   |
| loadgenerator                 | Python/Locust | Continuously sends requests imitating realistic user shopping flows to the frontend.                                              |


To deploy the app in you cluster, perform the below command from the root directory of our course repo (make sure the YAML file exists): 

```bash 
kubectl apply -f k8s/release-0.8.0.yaml
```

By default, **applications running within the cluster are not accessible from outside the cluster.**
There are various techniques available to enable external access, we will cover some of them later on.

Using **port forwarding** allows developers to establish a temporary tunnel for debugging purposes and access applications running inside the cluster from their local machines.

```bash
kubectl port-forward svc/frontend 8080:80
```

Visit the service in http://localhost:8080

## Pods and namespaces


Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.
A Pod is a group of **one or more containers**, with **shared storage** and **network resources**, and a specification for how to run the containers.

You can list the Online Boutique pods by:

```bash
kubectl get pods
```

Make sure all pods are `Running`. 

Pods and other resources are aggrandized in **namespaces**.
Namespace isolates resources within a cluster, usually for better organization and access control.

If you don't specify namespace, the `default` namespace is used. To list pods from all namespaces:

```bash
kubectl get pods -A
```

# Exercises 

> [!TIP]
> ### `kubectl` quick reference
> 
> | Description                                | Examples                                                |
> |--------------------------------------------|---------------------------------------------------------|
> | List cluster objects - basic information   | `kubectl get pods`, `kubectl get nodes`                 |
> | List cluster objects - from all namespaces | `kubectl get pods -A`.                                  |
> | List cluster objects - certain namespace   | `kubectl get pods -n kube-system`.                      |
> | List cluster objects - wider information   | `kubectl get pods -o wide`, `kubectl get nodes -o wide` |
> | Get full description of an object          | `kubectl describe pod POD_NAME`                         |
> | Apply a YAML manifest                      | `kubectl apply -f my-manifest.yaml`.                    |
> | Apply all manifests in a given dir         | `kubectl apply -f dir/`.                                |
> | Delete an applied manifest                 | `kubectl delete -f my-manifest.yaml`                    |
> | Watch pod logs                             | `kubectl logs my-pod`                                   |


### :pencil2: Using `kubectl`

Use `kubectl` to answer the following questions: 

1. How many pod replicas does the **frontend** microservice have? 
2. How many **containers** does a **frontend** pod have? 
3. Use the `kubectl describe` command to get the IP address of the **frontend** pod. 
4. For the single **frontend** running pod, how many environment variables does a container named `server` have? 
5. For the single **frontend** running pod, what is the Docker image the container named `server` based on? 
6. What is the node name that the **emailservice** pod was scheduled on (by the k8s scheduler)?
8. What is the port do the **checkoutservice** pods listend on? 

### :pencil2: Pod troubleshoot I

Apply the `k8s/customers-db.yaml` to deploy a MySQL pod in the cluster.

1. When a pod is in `Pending` status, one of the first places for debugging is the pod's events. Use the `kubectl describe` to investigate the root cause for the `Pending` status of the `customers` pod.
2. Use the `kubectl describe node YOUR_NODE_NAME` to get information about the current available capacity in the node (you can also get the same information from your node's **Node details** page in the GCP console), use your common sense to modify the YAML manifest so the customers-db pod can run in the cluster. 
3. When a pod is in `CrashLoopBackOff` status, the pod's containers were started successfully, but crashed due to internal error. 
   One of the first places for debugging in the pod's logs. Print the pod's log to address the issue. 
4. Inspired by the `k8s/release-0.8.0.yaml` YAML manifest, try to add the required env var to the `k8s/customers-db.yaml` manifest to make the MySQL pod running. 


Use the `kubectl delete` command to cleanup your cluster. 


[k8s_components]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_components.png
[k8s_online-boutique-arch]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_online-boutique-arch.png
[k8s_gke_architecture]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_gke_architecture.png

