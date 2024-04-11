# Kubernetes application that requires a database to store its data and a caching service to enhance performance

The application consists of three containers:

1.	a web server that serves user request  
2.	a postgresql database container.
3.	a caching service that caches frequently accessed data.

## Considerations:

1. Web server container can connect to the database container and redis cache.
2. database data is persistent.
3. Inbound communication from the internet to the web application needs to be available via HTTPS only.
4. The web server and redis containers should be configured with autoscaling based on CPU and memory usage.
5. Make sure that the redis container only deploys on nodes that have the label of: cache, using affinities.
6. Ensure an example of liveliness, readiness, and startup probes are documented.

# Implementation:

To meet the requirements outlined for the Kubernetes application, I've created several Kubernetes resources, including deployments, services, persistent volume claims, and  an Ingress resource for HTTPS access. Below is an example configuration for each component:

## Web Server Deployment:

### web_server.yml

1. Image: Your web server Docker image.
2. Environment variables: Database connection details, Redis connection details
3. Autoscaling: Based on CPU and memory usage
4. Liveness, readiness, and startup probes: Example probes included in the deployment manifest

```bash
kubectl apply -f web_server.yml
```
## PostgreSQL Deployment:
### postgresql.yml

This Kubernetes manifest creates a separate container for PostgreSQL, ensures persistent storage, and exposes it as a ClusterIP service within the Kubernetes cluster. Let's break down the components:

#### ConfigMap: 
Defines a ConfigMap named ```postgres-secret```
 containing environment variables for PostgreSQL configuration, such as the database name, username, password, and data directory (PGDATA).

#### PersistentVolume: 
Defines a PersistentVolume named ```postgres-pv``` with a capacity of 1Gi and a hostPath volume source. This volume is used to store PostgreSQL data and is configured to retain data even if pods are deleted.

#### PersistentVolumeClaim: 
Defines a PersistentVolumeClaim named ```postgres-pvc``` to dynamically provision storage from the PersistentVolume. This claim is bound to the PersistentVolume defined earlier.

#### Deployment: 
Defines a Deployment named postgresql with two replicas. Each replica runs a PostgreSQL container based on the ```postgres:latest``` image. The containers mount the PersistentVolumeClaim to ```/var/lib/postgresql/data``` to store database data persistently. The environment variables are injected into the container from the ConfigMap.

#### Service: 
Defines a ClusterIP service named ```psql-service-np``` that exposes PostgreSQL internally within the Kubernetes cluster. This service type makes the PostgreSQL service accessible from other pods within the cluster.

With this setup, other containers or pods within the Kubernetes cluster can communicate with the PostgreSQL container using the service name (psql-service-np) as the hostname and the specified port (5432).

### Note: 
web application containers or pods are deployed in the same Kubernetes cluster and namespace as the PostgreSQL container to communicate with it via the ClusterIP service.

```bash
kubectl apply -f postgresql.yml
```

## Redis Deployment

This Kubernetes manifest creates a ```StatefulSet``` for Redis with three replicas and exposes it as a Service within the Kubernetes cluster. Let's break down the components:

#### StatefulSet:
Defines a StatefulSet named redis with three replicas. Each replica runs a Redis container based on the ```redis:latest``` image. The StatefulSet ensures stable, unique network identifiers for each pod and ordered deployment and scaling.

#### Affinity:
Specifies node affinity to schedule Redis pods on nodes labeled with ```type: cache```. This helps to co-locate Redis pods with cache-specific nodes available in your cluster.

#### Containers:
Configures the Redis container with resource requests and limits for memory and CPU usage. It also defines readiness, liveness, and startup probes to ensure the container's health and readiness for serving traffic.

With this setup, other containers or pods within the Kubernetes cluster can communicate with the Redis pods using the service name (redis-service) as the hostname and the specified port (6379).

## Configuring TLS/SSL certificates
I have tested this setup on AWS EKS and on local machine which is running docker with kubernetes. 

### Setting-up the Ingress controller
1. Add and Install NGINX Helm repository: First, add the NGINX Helm repository to your Helm configuration and then install it.
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx
```
I have used below commands to generate the certs for testing purpose only.
```bash
mkdir certs

openssl req -x509 -nodes -days 9999 -newkey rsa:2048 -keyout certs/ingress-tls.key -out certs/ingress-tls.crt

```

### Creating the Kubernetes Secret
```bash
kubectl create secret tls ingress-cert --namespace default --key=certs/ingress-tls.key --cert=certs/ingress-tls.crt -o yaml
```
Additionally, I have tried implementing this over the EKS cluster. 
```bash
helm install load balancer
helm install aws-load-balancer-controller eks/aws-load-balancer-controller
-n kube-system
--set clusterName=clusterName
--set serviceAccount.create=false
--set serviceAccount.name=aws-load-balancer-controller
```

## Contributing

Pull requests are welcome. For major changes, please open an issue first
to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License

for any questions, feel free to drop an email to bharatnegi0207@gmail.com
