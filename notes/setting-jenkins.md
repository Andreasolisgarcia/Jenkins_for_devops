# Jenkins Setup Guide
> One-time setup before running the pipeline

---

## 0. Verify tools are available on the VM

```bash
which docker    # → /usr/bin/docker
which helm      # → /usr/local/bin/helm
which kubectl   # → /usr/local/bin/kubectl
```

All three must return a path, otherwise the pipeline stages will fail.

---

## 1. Give Jenkins access to Docker

Jenkins runs as its own user and can't use Docker by default:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

> Without this, the Build & Push stage fails with `permission denied on /var/run/docker.sock`.

---

## 2. Create Kubernetes namespaces

```bash
kubectl create namespace dev
kubectl create namespace qa
kubectl create namespace staging
kubectl create namespace prod
```

---

## 3. Create Jenkins credentials

`Jenkins → Manage Jenkins → Credentials → Global → Add Credential`

| Credential ID | Type | Content | Used in |
|---|---|---|---|
| `dockerhub` | Username/Password | Docker Hub login | Build & Push stage |
| `kubeconfig` | Secret file | `/etc/rancher/k3s/k3s.yaml` | All Helm deploy stages |
| `values-movie-secret` | Secret file | `values-movie-secret.yaml` | Deploy stages (movie) |
| `values-cast-secret` | Secret file | `values-cast-secret.yaml` | Deploy stages (cast) |
| `GITHUB_CREDS` | Username/Password | GitHub login + token | Saving Tags stage |

### Get the kubeconfig file

```bash
cat /etc/rancher/k3s/k3s.yaml
```

Copy the content → upload as a Secret file in Jenkins.

### Secret values file structure

```yaml
# values-movie-secret.yaml  ← never commit this file
secret:
  POSTGRES_PASSWORD: yourpassword
```

```yaml
# values-cast-secret.yaml  ← never commit this file
secret:
  POSTGRES_PASSWORD: yourpassword
```

These files are listed in `.gitignore`.

---

## 4. Configure the Jenkins pipeline

`Jenkins → New Item → Pipeline → name: fastapi-pipeline`

| Field | Value |
|---|---|
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | your GitHub repo URL |
| Branches | `*/dev` and `*/master` |
| Script Path | `Jenkinsfile` |

Enable: **GitHub hook trigger for GITScm polling**

---

## 5. Configure the GitHub webhook

`GitHub → repo → Settings → Webhooks → Add webhook`

| Field | Value |
|---|---|
| Payload URL | `http://<EC2-PUBLIC-IP>:8080/github-webhook/` |
| Content type | `application/json` |
| Events | Just the push event |

> If the EC2 IP changes (instance restart), update it here **and** in Jenkins:  
> `sudo vim /var/lib/jenkins/jenkins.model.JenkinsLocationConfiguration.xml`

---

## Checklist

```
✅ Docker, Helm, kubectl accessible in PATH
✅ Jenkins added to docker group + restarted
✅ 4 namespaces created (dev, qa, staging, prod)
✅ 5 credentials created in Jenkins
✅ Pipeline configured (SCM → GitHub)
✅ GitHub webhook configured
```


