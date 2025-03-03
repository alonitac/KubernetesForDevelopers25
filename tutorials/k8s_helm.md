# Helm - The Kubernetes Package Manager

## Motivation

**Helm** is a "package manager" for Kubernetes. Here are some of the main features of the tool:

- **Helm Charts**: Instead of dealing with numerous YAML manifests, which can be a complex task, Helm introduces the concept of a "Package" (known as **Chart**) – a cohesive group of related YAML manifests that collectively define a single application within the cluster.
  For example, an application might consist of a Deployment, Service, HorizontalPodAutoscaler, ConfigMap, and Secret. 
  These manifests are interdependent and essential for the seamless functioning of the application. 
  Helm encapsulates this collection, making it easier to manage, version, and deploy as a unit.

- **Sharing Charts**: Helm allows you to share your charts, or use other's charts. Want to deploy MongoDB server on your cluster? Someone already done it before, you can use her Helm Chart to with your own configuration values to deploy the MongoDB.  

- **Dynamic manifests**: Helm allows you to create reusable templates with placeholders for configuration values. For example:
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: {{ .Values.serviceName }}   # the service value will be dynamically placed when applying the manifest
  spec:
    selector:
      app: {{ .Values.Name }}   # the selector value will be dynamically placed when applying the manifest
  ...
  ```
  
  This becomes very useful when dealing with multiple clusters that share similar configurations (Dev, Prod, Test clusters). 
  Instead of duplicating YAML files for each cluster, Helm enables the creation of parameterized templates.

## Install Helm

https://helm.sh/docs/intro/install/

## Helm Charts 

Helm uses a packaging format called charts. A chart is a collection of files that describe a related set of Kubernetes resources. 
A single chart might be used to deploy some single application in the cluster.

### Deploy Grafana using Helm

> [!IMPORTANT]
> Before starting, remove any Grafana deployment from your cluster, including all related resources. 


Like DockerHub, there is an open-source community Hub for Charts of famous applications.
It's called [Artifact Hub](https://artifacthub.io/packages/search?kind=0), check it out. 

Let's review and install the [official Helm Chart for Grafana](https://artifacthub.io/packages/helm/grafana/grafana).

To deploy the Grafana Chart, you first have to add the **Repository** in which this Chart exists. 
A Helm Repository is the place where Charts can be collected and shared.

```bash
helm repo add grafana-charts https://grafana.github.io/helm-charts
helm repo update
```

The `grafana-charts` Helm Repository contains many different Charts maintained by the Grafana organization. 
[Among the different Charts](https://artifacthub.io/packages/search?repo=grafana&sort=relevance&page=1) of this repo, there is a Chart used to deploy the Grafana server in Kubernetes. 

Deploy the `grafana` Chart by: 

```bash
helm install grafana grafana-charts/grafana 
```

The command syntax is as follows: `helm install <release-name> <helm-repo-name>/<chart-name>`.

Whenever you install a Chart, a new **Release** is created.   
In the above command, the Grafana server has been released under the name `grafana`, using the `grafana` Chart from the `grafana-charts` repo.

During installation, the Helm client will print useful information about which resources were created, what the state of the Release is, and also whether there are additional configuration steps you can or should take.

#### Try it yourself

Review the release's output. Then use `port-forward` to visit the Grafana server.

### Customize the Grafana release

When installed the Grafana Chart, the server has been release with default configurations that the Chart author decided for you. 

A typical Chart contains hundreds of different configurations, e.g. container's environment variables, custom secrets, etc..

Obviously, you want to customize the Grafana release according to your configurations.
Good Helm Chart should allow you to configure the Release according to your configurations. 

To see what options are configurable on a Chart, go to the [Chart's documentation page](https://artifacthub.io/packages/helm/grafana/grafana), or use the `helm show values grafana-charts/grafana` command. 

Let's override some of the default configurations by specifying them in a YAML file, and then pass that file during the Release upgrade:

```yaml
# k8s/grafana-values.yaml

persistence:
  enabled: true
  size: 2Gi

env:
  GF_DASHBOARDS_VERSIONS_TO_KEEP: 10

```

The above values configure the Grafana server data to be persistent, and define some Grafana related environment variable. 

To apply the new Chart values, `upgrade` the Release: 

```bash
helm upgrade -f grafana-values.yaml grafana grafana-charts/grafana
```

An `upgrade` takes an existing Release and upgrades it according to the information you provide. 
Because Kubernetes Charts can be large and complex, Helm tries to perform the least invasive upgrade. 
It will only update things that have changed since the last release.

#### Try it yourself

Review the [Official Grafana Helm values](https://artifacthub.io/packages/helm/grafana/grafana), and add more Chart values overrides in `grafana-values.yaml` to achieve following configurations: 

1. The deployed Grafana [image tag version](https://hub.docker.com/r/grafana/grafana/tags) is higher than `8.0.0`. 
2. The [`redis-datasource`](https://grafana.com/grafana/plugins/redis-datasource/?tab=overview) plugin is installed.
3. A redis datasource is configured to collect metrics from your `redis` service.


If something does not go as planned during a release, it is easy to roll back to a previous release using `helm rollback [RELEASE] [REVISION]`:

```shell
helm rollback grafana 1
```

To uninstall the Chart release:

```shell
helm uninstall grafana
```


# Exercises 

### :pencil2: Redis cluster using Helm

Provision a Redis cluster with **1 master** and **2 replicas** using [Bitnami Helm Chart](https://artifacthub.io/packages/helm/bitnami/redis). 

Configure your **NetflixFrontend** to work with your Redis cluster instead of the existed `redis` Deployment as done in a previous exercise.  

### :pencil2: Create your own Helm chart for the NetflixFrontend service

In this exercise we will create a Chart for the [NetflixFrontend][NetflixFrontend].

We'll leverage Helm Charts to deploy the NetflixFrontend in two different environments, while maintaining only a single set of YAML manifests. 

1. Create a new Helm Chart skeleton in the `k8s/` dir in the **NetflixInfra** repo:

```bash
helm create netflix-frontend
```

Inside of this directory, Helm will expect the following structure:

```text
netflix-frontend/
  Chart.yaml          # A YAML file containing information about the chart
  values.yaml         # The default configuration values for this chart
  charts/             # A directory containing any charts upon which this chart depends.
  templates/          # A directory of templates that, when combined with values, will generate valid Kubernetes manifest files.
```

For more information about Chart files structure, go to [Helm docs](https://helm.sh/docs/topics/charts/). 

2. Change the default values in `values.yaml` to match the original `netflix-frontend` service.
3. Create value files for specific environment:
   - `values.yaml` with default values applied for all environments. 
   - `netflix-frontend/values-dev.yaml` with overrides for the dev env (e.g., assume different image tag is used per env).
   - `netflix-frontend/values-prod.yaml` with overrides for the prod env. 
4. Perform a dry run installation for the `dev` release (deployed in the `dev` namespace) by:

```bash
kubectl create ns dev
helm install netflix-frontend-dev ./netflix-frontend -f values.yaml -f values-dev.yaml --namespace dev --dry-run
```

5. If everything was set properly, remove the `--dry-run` to install a new release. 
6. Repeat the process for the `prod` release.

> [!TIP]
> As you edit your chart, you can validate that it is well-formed by running `helm lint`.

### :pencil2: Deploy your NetflixFrontend chart as an ArgoCD app

Based on the below ArgiCD Application manifest, create apps for both dev and prod envs. 
Make sure ArgoCD releases the charts successfully. 

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: netflix-frontend-dev
  namespace: dev
spec:
  project: default
  source:
    repoURL: ... # Your NetflixInfra repo URL
    targetRevision: main
    path: k8s/netflix-frontend  # Path to the Helm chart inside the NetflixInfra repo
    helm:
      releaseName: netflix-frontend-dev
      valueFiles:
        - values.yaml
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```


[NetflixFrontend]: https://github.com/exit-zero-academy/NetflixFrontend.git
