# Prepare
For this part, you'll need a sandbox or VM running Linux where you can install k3s.

A VMware Workstation or VirtualBox image will work fine.  If you have access to a Cloud-hosted VM on AWS, Azure, or Google Cloud, that should work too.

K3s has some [minimum system requirements](https://docs.k3s.io/installation/requirements#hardware).  For this demo with only one small container the minimums are plenty.
- at least 512 MB RAM, although a Gigabyte or more is preferred
- at least 1 CPU core

Ensure that you can SSH to the Linux machine, or that you have access to the terminal.

This assumes you already have [Docker Desktop](https://www.docker.com/) installed on your local development machine.

# Install k3s single-node kubernetes cluster
Run the following command to download and install k3s:

```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--no-deploy traefik" sh -s - --write-kubeconfig-mode 644
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```
Notes:
- We're opting-out of the traefik ingress controller, which is the default in k3s.
- k3s default configuration is to install the kubernetes master and worker on the same node.  This helps for simple configurations, but in larger-scale deployments you'd typically have multiple worker nodes on different machines which form a cluster.

Create a `values/ingress-nginx.yaml` configuration file with the following content:
```
---
controller:
  ingressClassResource:
    default: true
...
```

Run the following command to install the nginx incress controller:
```
helm upgrade --install \
    --namespace ingress-nginx --create-namespace \
    --repo https://kubernetes.github.io/ingress-nginx \
    --version 4.2.5 \
    --values values/ingress-nginx.yaml \
    ingress-nginx ingress-nginx
```

# Upload your docker container to a repository
Create a DockerHub account: `https://hub.docker.com/`

Log in to dockerhub via the command-line:

```
docker login
```

You'll be prompted for your username and password:

```
Username: <enter username>
Password: <enter password>
Login Succeeded
```

Tag your docker container:
```
docker tag <container-name>:<version> <dockerhub-username>/<container-name>:<version>
```
ex:
```
docker tag hello-world:latest msvaterlaus/hello-world:latest
```

Push your docker container to dockerhub:
```
docker push <dockerup-username>/<container-name>:<version>
```
ex:
```
docker push msvaterlaus/hello-world:latest
```


# Deploy a container to k3s
Test deploying a `hello-world` docker container hosted on dockerhub to the k3s cluster:

Create a `hello-world-deployment.yaml` manifest file with the following content (and replace `<dockerhub-username>` in the file):
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: <dockerhub-username>/hello-world:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

Deploy the container via kubectl:
```
kubectl apply -f hello-world-deployment.yaml
```

Verify that the container (inside a pod) is running:
```
kubectl get pods
```

If the pod did not start correctly (CrashLoopBackoff), you can get more details by running:
```
kubectl describe pod <pod-name>
```

or

```
kubectl logs <pod-name>
```

# Deploy an ingress configuration
In order to route network traffic to your container, you need to tell the ingress controller what URLs should be routed to that container.

Create a `hello-world-ingress.yaml` manifest file with the following content:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
```

Deploy the container via kubectl:
```
kubectl apply -f hello-world-ingress.yaml
```

You should now be able to access the container's API or UI via `<IP or Hostname>/testpath` in the web browser.

# Helm Charts
Deploying an application to kubernetes may involve multiple pieces:
- The service
- The ingress config to route network traffic to your app
- Additional values in a configmap
- Persistent volume claims
- Auto-scaling information
- ...

To nicely wrap all of the pieces together, we use a [Helm](https://helm.sh/) chart.  This isn't required, but could simplify more complicated deployments.

SystemLink provides a [starter helm chart](https://dev.azure.com/ni/DevCentral/_git/Skyline?path=/HelmStarterScaffold) that you can use to create one of your own.

# Troubleshooting

- Switching to the nginx ingress controller does not adopt any existing ingress configurations.  If you deployed an ingress config before switching from traefik to nginx, you'll need to delete and redeploy the ingress configurations.

- My container doesn't start after deploying to k3s.  What could be wrong?
  
  Run `kubectl describe pod <pod-name>` and check these common issues:
  - ImagePullBackoff - The kubernetes cluster can't find your docker container image on dockerhub, or doesn't have access to it.
  - CrashLoopBackoff - The container started, but exited.  Containers are expected to run forever unless there's an issue that causes it to exit.  Many times, you haven't specified a command that keeps the container from exiting.  Check the command in the [hello-world-deployment.yaml](hello-world/k3s-manifests/hello-world-deployment.yaml)