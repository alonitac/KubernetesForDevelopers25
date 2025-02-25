# Kubernetes config objects

## Secrets 

Kubernetes **Secrets** are a way to securely store and manage sensitive information such as passwords, API keys, or certificates in the cluster.
Secrets can be created independently of the Pods that use them, so there is less risk of the Secret (and its data) being exposed during the workflow of creating, viewing, and editing Pods.

Secrets can be **mounted as files** or exposed **as environment variables** within Pods.

### Provide secrets an environment variables

Suppose you have to provide some basic authentication information to your `nginx` service:

- Username `nginx-username`
- Password `39528$vdg7Jb`.

Create the Secret object using the `kubectl create secret` command:

```bash
kubectl create secret generic nginx-creds --from-literal='username=admin' --from-literal='password=39528$vdg7Jb'
```

The created Secret is an `Opaque` type secret.
It is used to store an arbitrary user-defined data.
Kubernetes support [other types](https://kubernetes.io/docs/concepts/configuration/secret/#secret-types) of secrets for different usages.

```console 
$ kubectl get secret nginx-creds
NAME                     TYPE     DATA   AGE
nginx-creds              Opaque   2      2m6s
```

The below example provides the secret data as environment variables to the running container:

```yaml
# k8s/deployment-demo-secret-env-var.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
      - name: server
        image: nginx:1.26.0
        env:
          - name: NGINX_WORKER_PROCESSES
            value: "2"
            
          - name: NG_USERNAME
            valueFrom:
              secretKeyRef:
                name: nginx-creds
                key: username
          - name: NG_PASSWORD
            valueFrom:
              secretKeyRef:
                name: nginx-creds
                key: password
```

In your shell, display the content of `NG_USERNAME` and `NG_PASSWORD` container environment variable:

```console
$ kubectl get pods -l app=nginx
NAME                            READY   STATUS    RESTARTS      AGE
nginx-55df5dcf48-6xbgx          1/1     Running   1 (91m ago)   19h

$ kubectl exec -it nginx-55df5dcf48-6xbgx -- /bin/sh 
root@nginx-55df5dcf48-6xbgx:/# echo $NG_USERNAME
admin

root@nginx-55df5dcf48-6xbgx:/# echo $NG_PASSWORD
39528$vdg7Jb
```


> [!NOTE]
> 
> You can also create secret from YAML manifests similarly to other k8s objects. 
> 
> To do so, **the data has to be encoded in Base64**:
> 
> ```console
> $ echo -n 'admin' | base64
> YWRtaW4=
> 
> $ echo -n '39528$vdg7Jb' | base64
> Mzk1MjgkdmRnN0pi
> ```
> 
> Apply the below manifest by: `kubectl apply -f k8s/secret-demo.yaml`:
> 
> ```yaml
> # k8s/secret-demo.yaml
> 
> apiVersion: v1
> kind: Secret
> metadata:
>   name: nginx-creds
> type: Opaque
> data:
>   username: YWRtaW4=
>   password: Mzk1MjgkdmRnN0pi
> ```
> 

[Read here](https://kubernetes.io/docs/concepts/configuration/secret/) for more information on how to provide secrets to containers. 

## ConfigMap

ConfigMap is a mechanism for storing non-sensitive configuration data in key-value pairs. 
ConfigMaps provide a convenient way to manage and **inject configuration settings into applications**, allowing for easy configuration changes without modifying the application's container image or restarting the Pod.

We continue with our `nginx` service as an example.
Let's say you want to change the default nginx configuration server (located under `/etc/nginx/conf.d/default.conf`). 

Two approaches can be taken here:

1. Build a new Docker image with your own config file, while using the `nginx:1.26.0` image as a base image in a Dockerfile.  
2. Use the original `nginx:1.26.0` image, and mount a `/etc/nginx/conf.d/` directory into the container file system, with your own config. 

The first approach introduced the overhead of build and maintaining a Docker image, only because a single file has to be changed in the pre-built image.
The seconds approach can be easily achieved using a ConfigMap.  

```yaml 
# k8s/configmap-demo.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  # this known as "file-like" keys. In YAML, the "|" coming after the key allows to have multi-line values
  default.conf: |
    server {
     listen 80;
     server_name  localhost;
     location / {
       proxy_pass http://netflix-movie-catalog-service:8080;
      }
    } 
```

After applying the ConfigMap, let's update our Deployment accordingly:

```yaml 
# k8s/deployment-demo-configmap-mount.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
      - name: server
        image: nginx:1.26.0
        env:
          - name: NGINX_WORKER_PROCESSES
            value: "2"
          - name: NG_USERNAME
            valueFrom:
              secretKeyRef:
                name: nginx-creds
                key: username
          - name: NG_PASSWORD
            valueFrom:
              secretKeyRef:
                name: nginx-creds
                key: password
        volumeMounts:
          - name: nginx-configurations
            mountPath: /etc/nginx/conf.d/
      volumes:
        - name: nginx-configurations
          configMap:
            name: nginx-conf
```

Connect to your nginx container and make sure the files has been mounted properly. 

Similarly to Secrets, ConfigMap can also be consumed as environment variables.

## Kubernetes core objects summary

![][k8s_core_objects]

# Exercises 

**Note**: To be done in the order of appearance.

### :pencil2: Add Nginx to the Netflix service

Add an Nginx Deployment to the Netflix service stack from previous tutorial.

![][k8s_netflix_simple_nginx]

Use `ConfigMap` to configure the Nginx to route traffic to the Netflix Frontend, 
by mounting the below code, stored as a `ConfigMap`, into the `/etc/nginx/conf.d/default.conf` path **within** the container:

```text
server {
     listen 80;
     server_name  localhost;
     location / {
       proxy_pass http://YOUR_NETFLIX_FRONTEND_SERVICE_HOSTNAME;
      }
    } 
```

Use `port-forward` to visit the Nginx and make sure the Netflix app is being served. 


### :pencil2: Grafana and Redis integration

Modify the Grafana deployment as follows: 

- `GF_SECURITY_ADMIN_USER` and `GF_SECURITY_ADMIN_PASSWORD` variables **should be read from dedicated Secret object** that you'll create with corresponding username and password.
- Instead of configuring the Redis datasource manually, configure it "as code" using a `ConfigMap`:
    - Create a `ConfigMap` as follows:
      ```yaml
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: grafana-datasources
      data:
        datasources.yaml: |-
          {
              "apiVersion": 1,
              "datasources": [
                {
                  "version": 2,
                  "name": "Redis",
                  "type": "redis-datasource",
                  "url": "<my-redis-service-url>",
                  "isDefault": true
                }
              ]
          }
      ```
      Change `<my-redis-service-url>` to your redis service URL.       

    - Mount the configmap into `/etc/grafana/provisioning/datasources` directory within the container. The Grafana server read all `.yaml` files in this dir and applies the data sources configurations. 
    - Make sure the datasource is configured on a clean Grafana deployment. 


### :pencil2: Configure credentials for InfluxDB

This short exercise demonstrates the idea the a secret can be used by multiple resources in the cluster.

1. Configure DB username and password `Secret` for the InfluxDB.
2. Provide the secret data to **both** the InfluxDB and the CronJob.
   - The InfluxDB expects the `INFLUXDB_ADMIN_USER` and `INFLUXDB_ADMIN_PASSWORD` env vars. 
   - In your CronJob object, add the following argument to every `curl` command that communicates with your DB: `-u $DB_USERNAME:$DB_PASSWORD`, while `DB_USERNAME` and `DB_PASSWORD` are env var provided to the Job. 



[NetflixMovieCatalog]: https://github.com/exit-zero-academy/NetflixMovieCatalog.git
[NetflixFrontend]: https://github.com/exit-zero-academy/NetflixFrontend.git
[k8s_core_objects]:  https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_core_objects.png
[k8s_netflix_simple_nginx]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/k8s_netflix_simple_nginx.png
