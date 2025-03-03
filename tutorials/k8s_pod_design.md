# Pods and containers design

In this tutorial we will take a closer look on Pods and containers characteristics, and how they are designed and scheduled on nodes properly.

Let's apply the following `netflix-frontend` Deployment (delete any previous Deployments if exist):

```yaml
# k8s/resources-demo.yaml 

apiVersion: apps/v1
kind: Deployment
metadata:
  name: netflix-frontend
  labels:
    app: netflix-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: netflix-frontend
  template:
    metadata:
        labels:
          app: netflix-frontend
    spec:
      containers:
      - name: server
        image: alonithuji/netflix-frontend:0.0.1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 200Mi
---
apiVersion: v1
kind: Service
metadata:
  name: netflix-frontend-service
spec:
  selector:
    app: netflix-frontend
  ports:
  - name: http
    port: 3000
    targetPort: 3000
```

## Resource management for Pods and containers

As can be seen from the above YAML manifest, when you specify a Pod, you can optionally specify how much of each resource (CPU and memory) a container needs: 

```text 
Limits:
  cpu:     200m
  memory:  200Mi
Requests:
  cpu:     100m
  memory:  100Mi
```

What do the **Limits** and **Requests** mean? 

### Resource limits

When you specify a resource **Limits** for a container, the **kubelet** enforces those limits, as follows:

- If the container tries to consume more than the allowed amount of memory, the system kernel terminates the process that attempted the allocation, with an out of memory (OOM) error.
- If the container tries to consume more than the allowed amount of CPU, it is just not allowed to do it.

Let's connect to the Pod and create an artificial CPU load, using he `stress` command:


```console 
$ kubectl exec -it <netflix-frontend-pod-name> -- /bin/bash
root@netflix-frontend-698dcc74b9-7d56w:/ # apt update
...

root@netflix-frontend-698dcc74b9-7d56w:/ # apt install stress-ng
...

root@netflix-frontend-698dcc74b9-7d56w:/ # stress-ng --cpu 2 --vm 1 --vm-bytes 50M -v
stress-ng: info:  [723] defaulting to a 86400 second (1 day, 0.00 secs) run per stressor
stress-ng: info:  [723] dispatching hogs: 2 cpu, 1 vm
```

Watch the Pod's CPU and memory metrics in the Kubernetes UI Dashboard. Although the stress test tries to use 2 cpus, it's limited to 200m (200 mili-cpu, which is equivalent to 0.2 cpu), as specified in the `.resources.limits.cpu` entry.
In addition, the Pod **is** able to use a `50M` of memory as it's below the specified limit (200 megabytes). 

Let's try to use more than the memory limit:

```console
/src # stress-ng --cpu 2 --vm 1 --vm-bytes 500M -v
stress-ng: info:  [723] defaulting to a 86400 second (1 day, 0.00 secs) run per stressor
stress-ng: info:  [723] dispatching hogs: 2 cpu, 1 vm
```

Watch how the processed is killed by the kernel using a `SIGKILL` signal. 

### Resources requests 

While resources limits is quite straight forward, resources request has a completely different purpose. 

When you specify the resource **Requests** for containers in a Pod, the **kube-scheduler** uses this information to decide which node to place the Pod on. How?   

When a Pod is created, the kube-scheduler should select a Node for the Pod to run on. Each node has a maximum capacity for CPU and RAM. 
The scheduler ensures that the resource CPU and memory requests of the scheduled containers is less than the capacity of the node.

```console
$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   15d   v1.27.4

$ kubectl describe nodes minikube
Name:               minikube
[ ... lines removed for clarity ...]
Capacity:
  cpu:                2
  ephemeral-storage:  30297152Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3956276Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  30297152Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3956276Ki
  pods:               110
[ ... lines removed for clarity ...]
Non-terminated Pods:          (23 in total)
  Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
  default                     adservice-746b758986-vh7zt                    100m (5%)     300m (15%)  180Mi (4%)       300Mi (7%)     15d
  default                     cartservice-5d844fc8b7-rl7fp                  200m (10%)    300m (15%)  64Mi (1%)        128Mi (3%)     15d
  default                     checkoutservice-5b8645f5f4-gbpwn              50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     currencyservice-79b446569d-dqq7l              50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     emailservice-55df5dcf48-5ksdz                 50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     frontend-66b6775756-bxhbq                     50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     frontend-pod                                  50m (2%)      100m (5%)   5Mi (0%)         128Mi (3%)     50m
  default                     loadgenerator-78964b9495-rphvs                50m (2%)      500m (25%)  256Mi (6%)       512Mi (13%)    15d
  default                     paymentservice-8f98685c6-cjcmx                50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     productcatalogservice-5b9df8d49b-ng6pz        100m (5%)     200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     recommendationservice-5b4bbc7cd4-rkdft        50m (2%)      200m (10%)  220Mi (5%)       450Mi (11%)    15d
  default                     redis-cart-76b9545755-nr785                   70m (3%)      125m (6%)   200Mi (5%)       256Mi (6%)     15d
  default                     shippingservice-648c56798-m8vjh               100m (5%)     200m (10%)  64Mi (1%)        128Mi (3%)     15d
  kube-system                 coredns-5d78c9869d-jkx8m                      100m (5%)     0 (0%)      70Mi (1%)        170Mi (4%)     15d
  kube-system                 etcd-minikube                                 100m (5%)     0 (0%)      100Mi (2%)       0 (0%)         15d
  kube-system                 kube-apiserver-minikube                       250m (12%)    0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 kube-controller-manager-minikube              200m (10%)    0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 kube-proxy-kv6m6                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 kube-scheduler-minikube                       100m (5%)     0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 metrics-server-7746886d4f-t5btd               100m (5%)     0 (0%)      200Mi (5%)       0 (0%)         15d
  kube-system                 storage-provisioner                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kubernetes-dashboard        dashboard-metrics-scraper-5dd9cbfd69-lm4s2    0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kubernetes-dashboard        kubernetes-dashboard-5c5cfc8747-86kzb         0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                1820m (91%)   2925m (146%)
  memory             1743Mi (45%)  2840Mi (73%)
```

By looking at the “Pods” section, you can see which Pods are taking up space on the node.

Note that although actual memory resource usage on nodes is low (`45%`), the scheduler still refuses to place a Pod on a node if the memory request is more than the available capacity of the node. 
This protects against a resource shortage on a node when resource usage later increases, for example, during a daily peak in request rate.

Test it yourself! Try to change the `.resources.request.cpu` to a larger value (e.g. `3000m` instead of `100m`), and re-apply the Pod. What happened? 

Since the total CPU requests on that node is `91%` (`1820m` out of `2000m` CPU), you can see that if a new Pod requests more than `180m` or more than `1.2Gi` of memory, that Pod will not fit on that node.

> [!NOTE]
> In Production, always specify resource request and limit. It allows an efficient resource utilization and ensures that containers have the necessary resources to run effectively and scale as needed within the cluster.

## Configure quality of service for Pods

What happened if a container exceeds its memory request and the node that it runs on becomes short of memory overall? it is likely that the Pod the container belongs to will be **evicted**.

When you create a Pod, Kubernetes assigns a **Quality of Service (QoS) class** to each Pod as a consequence of the resource constraints that you specify for the containers in that Pod.
QoS classes are used by Kubernetes to decide which Pods to evict from a Node experiencing [Node Pressure](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/). 

1. [Guaranteed](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#guaranteed) - The Pod specifies resources request and limit. CPU and Memory requests and limit are equal.
2. [Burstable](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#burstable) - The Pod specifies resources request and limit. But CPU and Memory requests and limit are not equal.
3. [BestEffort](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#besteffort) - The Pod does not specify resources request or limit for some of its containers.


When a Node runs out of resources, Kubernetes will first evict `BestEffort` Pods running on that Node, followed by `Burstable` and finally `Guaranteed` Pods.
When this eviction is due to resource pressure, **only Pods exceeding resource requests are candidates** for eviction.

Try to simulate Node pressure by overloading the node from a `Guaranteed` Pod, `Burstable` Pod and `BestEffort` Pod, and see the behaviour. 

## Container probes - Liveness and Readiness probes

A **probe** is a diagnostic performed periodically by the **kubelet** on a container, usually by an HTTP request, to check the container status. 

- **Liveness probes** are used to determine the health of a container (a.k.a. health check), by a periodic checks. The container is restarting if the probe fails.
- **Readiness probes** are used to determine whether a Pod is ready to receive traffic (when it is prepared to accept requests, preventing traffic from being sent to Pods that are not fully operational, mostly because the pod is still initializing, or is about to terminate as part of a rolling update).

> [!NOTE]
> In Production, always specify Liveness and Readiness probes, they are essential to ensure the availability and proper functioning of applications within Pods.

### Define a Liveness probe

The `netflix-frontend` service, as you may know, exposes an default endpoint: `/`. Upon an HTTP request, if the endpoint returns a `200` status code, it means "the server is alive".

In the below example, the `livenessProbe` entry defines a liveness probe for our Pod: 

```yaml
# k8s/liveness-demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: netflix-frontend
  labels:
    app: netflix-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: netflix-frontend
  template:
    metadata:
        labels:
          app: netflix-frontend
    spec:
      containers:
      - name: server
        image: alonithiji/netflix-frontend:0.0.1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 200Mi
        livenessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/"
            port: 3000
```

If the prob HTTP requests to `/` will fail, the **kubelet** will restart the container.

## Horizontal Pod Autoscaling

A **HorizontalPodAutoscaler** (HPA for short) automatically updates the number of Pods to match demand, usually in a Deployment or StatefulSet.

The HorizontalPodAutoscaler controller periodically (by default every 15 seconds) adjusts the desired scale of its target (for example, a Deployment) to match observed metrics such as average CPU utilization, average memory utilization, or any other custom metric you specify.
The common use for HPA is to configure it to fetch metrics from a [Metrics Server](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#metrics-server) API (should be installed as an add-on). 
Every 15 seconds, the HPA controller queries the Metric Server and fetches resource utilization metrics (e.g. CPU, Memory) for each Pod that are part of the HPA. 
The controller then calculates the desired replicas, and scale in/out based on the current replicas.

Let's create an autoscaler object for the `netflix-frontend` Deployment:

```yaml
# k8s/hpa-autoscaler-demo.yaml

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: netflix-frontend-hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: netflix-frontend
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

When defining the pod specification, the resource **Requests**, like CPU and memory must be specified.
This is used to determine the resource utilization and used by the HPA controller to scale the target up or down.
In the above example, the HPA will scale up once the Pod is reaching 50% of the `.resources.requests.cpu`. 

Next, let's see how the autoscaler reacts to increased load. 
To do this, you'll start a different Pod to act as a client. 
The container within the client Pod runs in an infinite loop, sending queries to the `netflix-frontend-service` service.

```bash 
kubectl run -it load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.001; do (wget -q -O- http://netflix-frontend-service &); done"
```

# Exercises 

### :pencil2: Zero downtime during scale

Your goal in this exercise is to achieve zero downtime during scale up/down events of the NetflixFrontend.

1. If haven't done yet, Deploy the NetflixFrontend app as a simple `Deployment` (with the corresponding `Service`). 
2. Generate some incoming traffic (20 requests per second) from a dedicated pod: 
   ```bash
   kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.05; do (wget -q -O- http://SERVICE_URL &); done"
   ```
   Chane `SERVICE_URL` accordingly.    

3. Watch the `load-generator` pod logs. Did you lose requests for a short moment? Why?
4. Stop the load generator pod and define a `liveness` and `readiness` probes in your Deployment.
5. Regenerate the load and make sure the app is scaled up properly. 
6. **Bonus**: does your app scaled down properly? 
7. **Bonus**: Try to perform rolling update during load (e.g. from version `0.0.1` to `0.0.2`). Did you lose requests during the rolling update?

[k8s_multicontainer_pod]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_multicontainer_pod.png
