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
   - Make sure you use the low-cost **e2-medium** instance edition (2vCPU, 4GB RAM).
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

```bash
kubectl get nodes
```

## Kubernetes main components

When you deploy Kubernetes (using minikube, or any other distro), you get a **cluster**.

A Kubernetes cluster consists of a set of worker machines, called **nodes**, that run containerized applications (known as **Pods**).
Every cluster has at least one worker node.

The **control plane** manages the worker nodes and the Pods in the cluster.
In production environments, the control plane usually runs across multiple computers and a cluster usually runs multiple nodes, providing fault-tolerance and high availability.

![][k8s_components]

#### Control Plane main components

The control plane's components make global decisions about the cluster (for example, scheduling), as well as detecting and responding to cluster events (for example, starting up a new pod when a deployment's replicas field is unsatisfied).

- **kube-apiserver**: The API server is the front end for the Kubernetes control plane.
- **etcd**: Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data.
- **kube-scheduler**: Watches for newly created Pods with no assigned node, and selects a node for them to run on.
- **kube-controller-manager**: Runs [controllers](https://kubernetes.io/docs/concepts/overview/components/#kube-controller-manager). There are many different types of controllers. Some examples of them are:
  - Responsible for noticing and responding when nodes go down.
  - Responsible for noticing and responding when a Deployment is not in its desired state.

#### Node components

Node components run on every node, maintaining running pods and providing the Kubernetes runtime environment.

- **kubelet**: An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod.
- **kube-proxy**: kube-proxy is a network proxy that runs on each node in your cluster. It allows network communication to your Pods from network sessions inside or outside your cluster.
- **Container runtime**: It is responsible for managing the execution and lifecycle of containers ([containerd](https://containerd.io/) or [CRI-O](https://cri-o.io/)).


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

Using port forwarding allows developers to establish a temporary tunnel for debugging purposes and access applications running inside the cluster from their local machines.

```bash
kubectl port-forward svc/frontend 8080:80
```

Finally, delete the Online Boutique Service resources by: 

```bash 
kubectl delete -f k8s/release-0.8.0.yaml
```


[k8s_components]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_components.png
[k8s_online-boutique-arch]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_online-boutique-arch.png
[k8s_gke_architecture]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_gke_architecture.png

