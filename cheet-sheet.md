# Cheatsheet Docker, Kubernetes & Helm

## Docker

### Build

| Action | Commande |
|--------|----------|
| Build | `docker build -t <image>:<tag> <path>` |
| Build sans cache | `docker build --no-cache -t <image>:<tag> <path>` |

### Run

| Action | Commande |
|--------|----------|
| Run simple | `docker run <image>` |
| Run + supprimer après | `docker run --rm <image>` |
| Run en arrière-plan | `docker run -d <image>` |
| Run interactif (shell) | `docker run --rm -it <image> /bin/sh` |
| Run avec variable env | `docker run -e VAR=value <image>` |
| Run avec port exposé | `docker run -p 8080:8000 <image>` |

### Voir

| Action | Commande |
|--------|----------|
| Voir images | `docker images` |
| Voir conteneurs running | `docker ps` |
| Voir tous conteneurs | `docker ps -a` |
| Logs conteneur | `docker logs <container_id>` |
| Logs en live | `docker logs -f <container_id>` |

### Inspecter

| Action | Commande |
|--------|----------|
| Inspecter image | `docker inspect <image>` |
| Voir le CMD | `docker inspect <image> --format='{{.Config.Cmd}}'` |
| Voir l'entrypoint | `docker inspect <image> --format='{{.Config.Entrypoint}}'` |

### Stop / Supprimer

| Action | Commande |
|--------|----------|
| Stop un conteneur | `docker stop <container_id>` |
| Stop tous les conteneurs | `docker stop $(docker ps -q)` |
| Supprimer conteneur | `docker rm <container_id>` |
| Supprimer tous conteneurs | `docker rm $(docker ps -aq)` |
| Supprimer image | `docker rmi <image>` |
| Supprimer toutes images | `docker rmi $(docker images -q)` |
| Forcer suppression | `docker rmi -f <image>` |

### Registry (Docker Hub)

| Action | Commande |
|--------|----------|
| Login | `docker login` |
| Push | `docker push <image>:<tag>` |
| Pull | `docker pull <image>:<tag>` |

---

## Kubernetes (kubectl)

### Voir ressources

| Action | Commande |
|--------|----------|
| Voir tout | `kubectl get all -n <namespace>` |
| Voir pods | `kubectl get pods -n <namespace>` |
| Voir pods en live | `kubectl get pods -n <namespace> -w` |
| Voir avec détails | `kubectl get pods -o wide -n <namespace>` |
| Voir services | `kubectl get svc -n <namespace>` |
| Voir deployments | `kubectl get deploy -n <namespace>` |
| Voir namespaces | `kubectl get ns` |
| Voir secrets | `kubectl get secrets -n <namespace>` |

### Décrire / Debug

| Action | Commande |
|--------|----------|
| Décrire pod | `kubectl describe pod <nom> -n <namespace>` |
| Décrire deployment | `kubectl describe deploy <nom> -n <namespace>` |
| Logs d'un pod | `kubectl logs <pod> -n <namespace>` |
| Logs du crash précédent | `kubectl logs <pod> -n <namespace> --previous` |
| Logs en live | `kubectl logs -f <pod> -n <namespace>` |
| Entrer dans un pod | `kubectl exec -it <pod> -n <namespace> -- /bin/sh` |

### Modifier

| Action | Commande |
|--------|----------|
| Éditer ressource | `kubectl edit <ressource> <nom> -n <namespace>` |
| Appliquer fichier | `kubectl apply -f <fichier.yaml> -n <namespace>` |
| Redémarrer deployment | `kubectl rollout restart deployment <nom> -n <namespace>` |

### Supprimer

| Action | Commande |
|--------|----------|
| Supprimer pod | `kubectl delete pod <nom> -n <namespace>` |
| Supprimer par label | `kubectl delete pods -l <label>=<value> -n <namespace>` |
| Supprimer deployment | `kubectl delete deploy <nom> -n <namespace>` |

### Créer

| Action | Commande |
|--------|----------|
| Créer namespace | `kubectl create namespace <nom>` |
| Créer secret Docker | `kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<user> --docker-password=<pass> --docker-email=<email> -n <namespace>` |

---

## Helm

### Voir

| Action | Commande |
|--------|----------|
| Voir releases | `helm list -n <namespace>` |
| Voir valeurs d'une release | `helm get values <release> -n <namespace>` |
| Voir historique | `helm history <release> -n <namespace>` |
| Voir ce qui sera créé (dry-run) | `helm template <release> <chart> -n <namespace>` |

### Installer / Upgrade

| Action | Commande |
|--------|----------|
| Installer | `helm install <release> <chart> -n <namespace>` |
| Installer avec values | `helm install <release> <chart> -f values.yaml -n <namespace>` |
| Installer avec plusieurs values | `helm install <release> <chart> -f values.yaml -f secrets.yaml -n <namespace>` |
| Upgrade | `helm upgrade <release> <chart> -n <namespace>` |
| Install ou upgrade | `helm upgrade --install <release> <chart> -n <namespace>` |

### Rollback / Supprimer

| Action | Commande |
|--------|----------|
| Rollback | `helm rollback <release> <revision> -n <namespace>` |
| Désinstaller | `helm uninstall <release> -n <namespace>` |

---

## Astuces

### Alias utiles (à ajouter dans ~/.bashrc)

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kga='kubectl get all'
alias kd='kubectl describe'
alias kl='kubectl logs'
```

### Debug workflow

1. `kubectl get pods -n <ns>` → voir l'état
2. `kubectl describe pod <pod> -n <ns>` → voir les events
3. `kubectl logs <pod> -n <ns>` → voir les logs app
4. `kubectl exec -it <pod> -n <ns> -- /bin/sh` → entrer dedans