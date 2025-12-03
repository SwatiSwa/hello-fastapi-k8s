# Kubernetes

This folder contains Kubernetes manifests and instructions to run the `hello-fastapi` app on a local cluster (Minikube) or a remote cluster.

Files in this folder:
- `deployment.yaml` — Deployment for `hello-fastapi` (default image: `hello-fastapi:latest`)
- `service.yaml` — Service exposing the pods (ClusterIP by default)

Prerequisites
- `kubectl` configured for your cluster
- For local testing: `minikube` (or `kind`) installed
- Docker or access to a container registry when deploying to remote clusters

Options to provide the image to the cluster

1) Build inside Minikube (recommended for local Minikube)

```bash
# point your shell to minikube's docker daemon
eval "$(minikube -p minikube docker-env)"

# from project root, build the image with the same tag used in the manifest
docker build -t hello-fastapi:latest .

# (optional) return your shell to the host Docker
eval "$(minikube -p minikube docker-env --unset)"

# apply manifests
kubectl apply -f k8s/

# restart rollout if needed
kubectl rollout restart deployment/hello-fastapi-deployment
```

2) Push to a registry (works for any cluster)

```bash
# build locally
docker build -t <registry-or-username>/hello-fastapi:latest .

docker login

docker push <registry-or-username>/hello-fastapi:latest

# update k8s/deployment.yaml to use the pushed image
# then apply
kubectl apply -f k8s/
```

3) Use `imagePullPolicy: IfNotPresent` when you built the image on the node or with Minikube

If your deployment uses `imagePullPolicy: Always` (default for `:latest`) Kubernetes will try to pull from a registry every time. To allow local images to be used, set `imagePullPolicy: IfNotPresent` in `k8s/deployment.yaml`.

Accessing the service

- Minikube service (quick):

```bash
minikube service hello-fastapi-service --url
# open returned url in browser, e.g. http://192.168.x.y:xxxxx
```

- Port-forward (no cluster changes):

```bash
kubectl port-forward svc/hello-fastapi-service 8080:80
# then open http://127.0.0.1:8080/ and /docs
```

- NodePort (exposes service on each node):

```bash
kubectl patch svc hello-fastapi-service -p '{"spec":{"type":"NodePort"}}'
kubectl get svc hello-fastapi-service -o wide
# then open http://<minikube-ip>:<nodePort>/
```

Troubleshooting

- Pods failing with `ImagePullBackOff`:
  - Build image inside Minikube (option 1) or push image to registry (option 2), or set `imagePullPolicy: IfNotPresent`.
  - Check pod events: `kubectl describe pod <pod-name>`
  - View logs: `kubectl logs <pod-name> -c hello-fastapi`

- Check resources:

```bash
kubectl get deployments,replicasets,pods,svc -l app=hello-fastapi
kubectl describe deployment hello-fastapi-deployment
```

If you want, I can add a `kustomization.yaml` to this folder to make overrides easier (e.g. swapping image names per environment). I can also add a CI workflow to build and push images automatically.