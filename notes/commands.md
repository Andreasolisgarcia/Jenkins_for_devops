# Commands Cheat Sheet

---

## Docker

```bash
# Build images
docker build -t reasg/jenkins_for_devops_movie_service:latest ./movie-service/
docker build -t reasg/jenkins_for_devops_cast_service:latest ./cast-service/

# Build without cache (force full rebuild)
docker build --no-cache -t reasg/jenkins_for_devops_movie_service:latest ./movie-service/
docker build --no-cache -t reasg/jenkins_for_devops_cast_service:latest ./cast-service/

# Push to Docker Hub
docker push reasg/jenkins_for_devops_movie_service:latest
docker push reasg/jenkins_for_devops_cast_service:latest

# Test a container locally (without Kubernetes)
docker run --rm \
  -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@localhost/cast_db_dev \
  reasg/jenkins_for_devops_cast_service:latest
```

---

## Helm

```bash
# Dry-run: render templates without deploying (useful to debug values)
helm template movie-service ./helm/charts \
  -f ./helm/values-movie.yaml \
  -f ./helm/values-movie-secret.yaml

helm template cast-service ./helm/charts \
  -f ./helm/values-cast.yaml \
  -f ./helm/values-cast-secret.yaml

# Install / upgrade (idempotent — works for first install and updates)
helm upgrade --install movie-service ./helm/charts \
  -f ./helm/values-movie.yaml \
  -f ./helm/values-movie-secret.yaml \
  -n dev

helm upgrade --install cast-service ./helm/charts \
  -f ./helm/values-cast.yaml \
  -f ./helm/values-cast-secret.yaml \
  -n dev

# Uninstall a release completely
helm uninstall cast-service -n dev
helm uninstall movie-service -n dev

# Run Helm tests
helm test movie-service -n dev --logs
helm test cast-service -n dev --logs
```

---

## Kubectl — Namespaces & Ingress

```bash
# Create the 4 environments
kubectl create namespace dev
kubectl create namespace qa
kubectl create namespace staging
kubectl create namespace prod

# Apply Ingress manually (not managed by Helm)
kubectl apply -f k8s/ingress.yaml -n dev
kubectl apply -f k8s/ingress.yaml -n staging
```

---

## Kubectl — Debugging

```bash
# Check pod status
kubectl get pods -n dev
kubectl get pods -n dev -w   # watch mode

# Check services (find NodePort)
kubectl get svc -n dev

# Get NodePort URL
export NODE_PORT=$(kubectl get --namespace dev \
  -o jsonpath="{.spec.ports[0].nodePort}" \
  services cast-service-fastapiapp)
export NODE_IP=$(kubectl get nodes --namespace dev \
  -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT

# Check env vars injected into the DB pod
kubectl exec -it movie-service-db-0 -n dev -- env | grep POSTGRES

# Delete stuck Helm test pod (before re-running helm test)
kubectl delete pod movie-service-fastapiapp-test-connection -n dev --ignore-not-found
kubectl delete pod cast-service-fastapiapp-test-connection -n dev --ignore-not-found

# Force K3s to pull a fresh image (clears local cache)
sudo crictl rmi reasg/jenkins_for_devops_cast_service:latest
sudo crictl rmi reasg/jenkins_for_devops_movie_service:latest
```

---

## Git

```bash
# See which files changed between last two commits (used in pipeline)
git diff --name-only HEAD~1 HEAD
```

---

## Jenkins

```bash
# Update Jenkins URL when EC2 public IP changes (after instance restart)
# Find this line in the file and update the URL:
# <jenkinsUrl>http://OLD-IP:8080/</jenkinsUrl>
sudo vim /var/lib/jenkins/jenkins.model.JenkinsLocationConfiguration.xml

# Then restart Jenkins to apply
sudo systemctl restart jenkins
```

> The GitHub webhook Payload URL must also be updated in GitHub → repo → Settings → Webhooks.

---

## Utils

```bash
# Decode a base64 secret (e.g. from kubectl get secret)
echo "bW92aWVfZGJfcGFzc3dvcmQ=" | base64 --decode

# Get public IP of the EC2 instance
curl -s http://checkip.amazonaws.com
```