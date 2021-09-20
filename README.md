# Spark History Server on Kubernetes

Easy setup to run a Spark history server on Kubernetes and display logs of Spark jobs.
Steps to do
1. create new Docker image
2. Deploy Kubernetes resources manual or via helm chart


## Architecture

The official Spark binaries already include the spark history server available. In terms of Kubernetes one just has to start the Docker image it with a different `entrypoint` command and mount a folder which shares the Spark logs. 

The basic Idea of the setup is, that the spark application writes the logs in a directory that is persistant and from there the Spark history server can read the data afterwards


![drawio_spark-history-server-architecture](https://user-images.githubusercontent.com/16557412/134069388-e4b7f4a0-f4ea-4a11-8b6d-dacd73707431.png)


## Docker Image

We need to build a new docker image and modify the standard Spark Dockerfile a bit to start the history server instead of the driver or executor. Please follow the instruction on the latest Spark documentation to build the initial Spark Docker images first and than add an additional layer with a new entrypoint by creating a new Dockerfile with the following content.

```dockerfile
FROM <repo>/spark-py:3.1.2
WORKDIR /
# Reset to root to run installation tasks
USER root
RUN groupadd -g 185 spark && \
    useradd -u 185 -g 185 spark
USER 185
# add new entrypoint
ENTRYPOINT bash /opt/spark/sbin/spark-daemon.sh start org.apache.spark.deploy.history.HistoryServer 1
```
Then build and push so that Kubernetes can pull it.

```shell
# build from dockerfile
docker build -t <repo>/spark-history-server:3.1.2 -f Dockerfile.history

# push into repo Kubernetes can pull from
docker push <repo>/spark-history-server:3.1.2
```


## Kubernetes Resources
Basicaly we need only two resources 
* Persistent Volume Claim (to persist the Spark Logs)
* Deployment (with the Pod running the history server)

It make sense to add an Ingress to have a fixed address to call the history server
* Service
* Ingress

#### Persistent Volume Claim
Prerequsit is that there is a persistent storage class setup. Or any other persistent volume.

A  `persistentVolumeClaim`  is created via a yaml file `pvc_spark-history-server.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spark-history-server
  namespace: spark
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: longhorn
  volumeMode: Filesystem
```

This `pvc` has to be mounted into the Spark job yaml and the history server yaml

```yaml
spec:
	containers:
    - name: job
      image: xx
        volumeMounts:
          - name: log-data
            mountPath: /tmp/spark-events
            readOnly: true
  volumes:
  - name: log-data
    persistentVolumeClaim:
    claimName: spark-history-server
```

#### Deployment

The history server itself is setup as deployment `deploy-spark-history-server.yaml`.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spark-history-server
  namespace: spark
  labels:
    app: spark-history-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spark-history-server
  template:
    metadata:
      name: spark-history-server
      labels:
        app: spark-history-server
    spec:
      containers:
        - name: spark-history-server
          image: <repo>/spark-history-server:3.1.2
          env:
            - name: SPARK_NO_DAEMONIZE
              value: "false"
          resources:
            requests:
              memory: "1Gi"
              cpu: "100m"
            limits:
              memory: "10Gi"
              cpu: "2"
          ports:
            - name: http
              protocol: TCP
              containerPort: 18080
          volumeMounts:
            - name: data
              mountPath: /tmp/spark-events
              readOnly: true
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: spark-history-server
```

now the Spark history server can be accessed on the http://localhost:18080 via port forwarding

```
kubectl port-forward spark-history-server 18080:18080
```

#### Service 

It is easier to set up a service and ingress in order to access the history permanetely on the same host address `svc-spark-history-server.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
 name: spark-history-server
 namespace: spark-poc
spec:
 selector:
  app: spark-history-server
 ports:
  - port: 18080
  protocol: TCP
  targetPort: 18080
 type: ClusterIP
```

#### Ingress

The ingress resource maps the service to an outside hostpath. Typically the IP or Path of the Kubernetes Cluster itself `ingress-spark-history-server.yaml`.

```yaml
apiVersion: [networking.k8s.io/v1](http://networking.k8s.io/v1)
kind: Ingress
metadata:
 name: spark-history-server
 namespace: spark-poc
spec:
 rules:
  - host: spark-history.k8s-prod1.local.parcit
  http:
   paths:
    - backend:
     service:
      name: spark-history-server
      port:
       number: 18080
    pathType: ImplementationSpecific
```

Now the history server can be accessed directly via the path https://spark-history.k8s-prod1.local.parcit

## Helm Chart

Instead of deploying the four yamls one can pack it in a helm chart. 
The easiest way is to create an empty chart via

```shell
helm create spark-history-server
```

which creates a folder structure like

```shell
spark-history-server
|-- Chart.yaml
|-- charts
|-- templates
|   |-- NOTES.txt
|   |-- _helpers.tpl
|   |-- deployment.yaml
|   |-- ingress.yaml
|   |-- service.yaml
|-- values.yaml
```

and place all our yamls in the template folder

```shell
spark-history-server
|-- Chart.yaml
|-- templates
|   |-- _helpers.tpl
|   |-- deploy.yaml
|   |-- ingress.yaml
|   |-- service.yaml
|		|-- pvc.yaml
|-- values.yaml
```

However the template can be parameterised a bit via the values in the files `Chart.yaml` and `values.yaml` .

The following configurations create a full working helm chart

#### Chart.yaml

Helm metadata for tracking of the installed version

```yaml
apiVersion: v2
name: spark-history-server
description: Spark History Server for ParcIT Kubernetes Spark Jobs
type: application
version: 1.0.0
appVersion: 1.0.0
```

#### values.yaml

parameters used in the resource yaml files

```yaml
namespace: "spark-poc"
replicaCount: 1
nameOverride: "spark-history-server"
fullnameOverride: "spark-history-server"
deployment:
  enable: true
  name: "deploy-spark-history-server"
  image: "svr-nexus01.local.parcit:8124/repository/dm-docker/spark-poc/spark-history-server:3.1.2"
  pullPolicy: Always
pvc:
  enable: false
  name: "pvc-spark-history-server"
  storageClassName: "longorn"
service:
  enabled: true
  name: "svn-spark-history-server"
ingress:
  enabled: true
  name: "ingress-spark-history-server"
  host: "spark-history.k8s-prod1.local.parcit"
resources:
  limits:
    cpu: 2
    memory: 10Gi
  requests:
    cpu: 100m
    memory: 1Gi
```

#### pvc.yaml

only create on initial installation, later the set the parameter `enabled: false` to avoid that all data are flushed on updates

```yaml
{{- if .Values.pvc.enabled -}}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ .Values.pvc.name }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "spark-history-server.labels" . | nindent 4 }}
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: {{ .Values.storageClassName }}
  volumeMode: Filesystem
{{- end }}
```

#### deploy.yaml

main deployment defining the pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.deployment.name }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "spark-history-server.labels" . | nindent 4 }}
    app: spark-history-server
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: spark-history-server
  template:
    metadata:
      name: spark-history-server
      labels:
        app: spark-history-server
    spec:
      containers:
        - name: spark-history-server
          image: {{ .Values.deployment.image }}
          imagePullPolicy: {{ .Values.deployment.pullPolicy }}
          env:
            - name: SPARK_NO_DAEMONIZE
              value: "false"
          resources:
            requests:
              memory: {{ .Values.resources.requests.memory }}
              cpu: {{ .Values.resources.requests.cpu }}
            limits:
              memory: {{ .Values.resources.limits.memory }}
              cpu: {{ .Values.resources.limits.cpu }}
          ports:
            - name: http
              protocol: TCP
              containerPort: 18080
          volumeMounts:
            - name: data
              # default path the spark history server looks for logs
              mountPath: /tmp/spark-events
              readOnly: true
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: {{ .Values.pvc.name }}
```

#### service.yaml

```yaml
{{- if .Values.service.enabled -}}
# create service to expose history server for ingress
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.service.name }}
  labels:
    {{- include "spark-history-server.labels" . | nindent 4 }}
  namespace: {{ .Values.namespace }}
spec:
  selector:
    {{- include "spark-history-server.selectorLabels" . | nindent 4 }}
  ports:
  - port: 18080
    protocol: TCP
    targetPort: 18080
  type: ClusterIP
{{- end }}
```

#### ingress.yaml

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.ingress.name }}
  labels:
    {{- include "spark-history-server.labels" . | nindent 4 }}
  namespace: {{ .Values.namespace }}
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - backend:
          service:
            name: {{ .Values.service.name }}
            port:
              number: 18080
        pathType: ImplementationSpecific
{{- end }}
```

### Installation

Test the installation of the Helm Chart via

```shell
helm install spark-history-server --dry-run --debug ./spark-history-server
```

and install or update via

```shell
helm install spark-history-server ./spark-history-server
# or install else update
helm upgrade spark-history-server ./spark-history-server --install
```

