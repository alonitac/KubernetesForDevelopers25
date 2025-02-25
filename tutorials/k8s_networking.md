# Kubernetes Networking

The Kubernetes network model is built out of several pieces:

- Communication achieved by 2 layers of networking: **Pod network** and **Node network**.
- Each node has its own IP address. Pods also have their own IP address. 
- All cluster nodes and pods are part of the same VPC. 
  [Google Virtual Private Cloud (VPC)](https://cloud.google.com/vpc/docs/overview) is service that logically isolated network to enable secure communication between resources.
  a VPC is divided into a list of **regional subnetworks (subnets)** in data centers, all connected by a global wide area network.

- In GKE, your standard mode cluster are using **VPC-native** networking model. 
  In this model, Pods and nodes share the same VPC infrastructure. 
   
  Nodes receive IPs from the **primary subnet range**, while pods get IPs from a **secondary range** within the same subnet.

   ![](https://i.sstatic.net/Ll5Rm.png)
- All pods can communicate with all other pods, whether they are on the same node or on different nodes.

- Pod-to-Service communications: this is covered by **Services**.
   - Services act as **stable endpoints** that abstract the dynamic IPs of pods.
   - When a pod communicates with a service, traffic is routed to one of the service's backend pods.
     `kube-proxy` manages this routing by maintaining IP tables or rules to forward traffic to the correct pod.

## Expose applications outside the cluster using a Service of type `LoadBalancer`

Kubernetes allows you to create a Service of `type=LoadBalancer`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

This Service takes effect only on cloud providers which support external load balancers (like Google Cloud Load Balancing). 
Applying this Service will **provision a Load Balancer resource for your Service**.

![][k8s_networking_lb_service]

Traffic from the Load Balancer is directed to the backend Pods by the Service. The cloud provider decides how it is load balanced across different cluster's Nodes.

Note that the actual creation of the load balancer happens asynchronously, and information about the provisioned balancer is published in the Service's `.status.loadBalancer` field.


## Ingress and Ingress controller 

A Service of type `LoadBalancer` is the core mechanism that allows you to expose application to clients outside the cluster. 

What now? should we set up a separate Elastic Load Balancer for each Service we wish to make accessible from outside the cluster?
Doesn't this approach seem inefficient and overly complex in terms of resource utilization and cost?

It is. Let's introduce an **Ingress** and **Ingress Controller**.

Ingress Controller is an application that runs in the Kubernetes cluster that manage external access to services within the cluster. 
There are [many Ingress Controller implementations](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) for different usages and clusters. 

[Nginx ingress controller](https://github.com/kubernetes/ingress-nginx) is one of the popular used one. 
Essentially, it's the same old good Nginx webserver app, exposed to be available outside the cluster (using Service of type `LoadBalancer`), and configured to route incoming traffic to different Services in the cluster (a.k.a. reverse proxy). 

![][k8s_networking_nginx_ic]

**Ingress** is another Kubernetes object, that defines the **routing rules** for the Ingress Controller (so you don't need to edit Nginx `.conf` configuration files yourself).

Let's deploy an Ingress Controller and apply an Ingress with routing rules. 

## Deploy the Nginx Ingress Controller

Ingress controllers are not started automatically with a cluster, you have to deploy it manually. 
We'll deploy the [Nginx Ingress Controller behind a Google Network Load Balancer](https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml
```

The above manifest mainly creates:

- Deployment `ingress-nginx-controller` of the Nginx webserver.
- Service `ingress-nginx-controller` of type `LoadBalancer`. 
- IngressClass `nginx` to be used in the Ingress objects (see `ingressClassName` below).
- RBAC related resources. 

To route traffic to the NetflixFrontend service, first let's create a custom domain for your service: 

1. Navigate to **Network Services** > **Load Balancing**.
2. Select your Load Balancer.
3. Find the **Frontend Configuration** section, and note down the IP Address.
4. Under **Network Services**, go to **Cloud DNS**.
5. Click on the `qodok8s.xyz` zone name (which is a real public domain, already registered for you)
6. Inside the DNS Zone, click **Add Standard**.
7. Enter:
    - **DNS Name**: your custom subdomain, e.g. `john.qodok8s.xyz`.
    - **IPv4 Address**: Enter your Load Balancer's IP.

8. Click "Create".

Now, apply the below `Ingress` (change values according to your configurations): 

```yaml
# k8s/ingress-demo.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: netflix-frontend
spec:
  ingressClassName: nginx
  rules:
  - host: YOUR_LB_IP_or_DOMAIN_NAME 
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: YOUR_NETFLIX_FRONTEND_SERVICE_HERE
            port:
              number: YOUR_SERVICE_PORT_HERE
```

Nginx is configured to automatically discover all ingress where `ingressClassName: nginx` is present, like yours.

Visit the application using your domain name. 

> [!NOTE]
> #### The relation between **Ingress** and **Ingress Controller**:
> 
> **Ingress** only defines the *routing rules*, it is not responsible for the actual routing mechanism.  
> An Ingress controller is responsible for fulfilling the Ingress routing rules. 
> In order for the Ingress resource to work, the cluster must have an Ingress Controller running.

# Exercises 

### :pencil2: Blue/Green using Nginx ingress controller

1. Create two **NetflixFrontend** deployments (denoted "blue" and "green").
2. Define which one is active, and configure your Ingress to route traffic to the active deployment.
   You can also scale the inactive deployment to 0 replicas.
3. Deploy a new version of your app to the inactive deployment and scale it up until it is fully ready to accept end-user traffic.
4. Configure your Ingress to route traffic to the inactive deployment, making it the new active one.
5. You can scale down the old deployment or keep it running for a while after the release to ensure the new version is stable.

### :pencil2: Canary using Nginx 

You can add [nginx annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) to specific `Ingress` objects to customize their behavior.

In some cases, you may want to "canary" a new set of changes by sending a small number of requests to a different service than the production service. 
The [canary annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary) enables the Ingress spec to act as an alternative service for requests to route to depending on the rules applied.

In this exercise we'll deploy a canary for the NetflixFrontend service. 

- Deploy the NetflixFrontend service in a version which is not your most up-to-date (e.g. `0.8.0` instead of `0.9.0`). 
- Now you want to deploy the newer app version (e.g. `0.9.0`) but you don't confident with this deployment.
  Create other (separated) YAML manifests for the new version of the service, call then `netflix-frontend-canary`. 
- Create another `Ingress` pointing to your canary Deployment, as follows: 

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: netflix-frontend-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5"
spec:
  ingressClassName: nginx
  rules:
     # TODO ... Make sure the `host` entry is the same as the existed netflix-frontend Ingress. 
```

This Ingress routes 5% of the traffic to the canary deployment. 

Test your configurations by periodically access the application:

```bash
/bin/sh -c "while sleep 0.05; do (wget -q -O- http://YOUR_SERVICE_PUBLIC_DOMAIN &); done"
```

Change `YOUR_SERVICE_PUBLIC_DOMAIN` accordingly.

**Bonus**: Use [different annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) to perform a canary deployment which routes users based on a request header `FOO=bar`, instead of specific percentage.


[k8s_networking_lb_service]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_lb_service.png
[k8s_networking_nginx_ic]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_nginx_ic2.png
[k8s_networking_np_service]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_np_service.png
[k8s_networking_np_lb_service]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_networking_np_lb_service.png
