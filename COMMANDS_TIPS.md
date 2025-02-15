# Events
Afficher l
```bash
kubectl events
```

# STATIC POD
Par défaut, le répertoire dans lequel sont stockés les manifests de création des pods static est `/etc/kubernetes/manifests/`.
Le chemin de ce répertoire est personnalisable. Pour être sûr du path Static, il faut parcourir le fichier de config de kublet.
```bash
grep staticPodPath /var/lib/kubelet/config.yaml
```

# API SERVER

Liste des plugins d'admission activés (en lien avec les schedulers) :
```bash
kubectl exec -it POD_NAME -n NAMESPACE -- kube-apiserver -h | grep 'enable-admission-plugins'
```
Liste des plugins d'admission désactivés (en lien avec les schedulers) :
```bash
kubectl exec -it POD_NAME -n NAMESPACE -- kube-apiserver -h | grep 'disable-admission-plugins'
```

# Logs et Metrics Server

## Metrics Server

Deployer **Metrics Server** sur le cluster Kubernetes
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Visualiser les metriques des nodes:
```bash
kubectl top node
```

Visualiser les metriques des pods:
```bash
kubectl top pod
```
## Logs
Manager les logs des applications :
```bash
kubectl logs -f POD_NAME
```
S'il y a plusieurs conteneurs dans le POD et que l'on souhaite cibler les logs d'un conteneur en particulier : 
```bash
kubectl logs -f POD_NAME CONTAINER_NAME
```

# Application Lifecycle
## Rolling Updates and Rollbacks
### Rollout
A chaque nouveau déploiement, un rollout crée une nouvelle révision.

Pour visualiser l'état du rollout :
```bash
kubectl rollout status deployment/DEPLOYMENT_NAME
```

Pour visualiser les révisions et l'historique du rollout :
```bash
kubectl rollout history deployment/DEPLOYMENT_NAME
```
### Deployment Strategy
Il existe deux types de stratégies de déploiement :
- **Recreate**, détruit toutes les instances simultanément et en recrée de nouvelles ensuite. L'inconvénient est que l'application sera injoignable le temps que les nouvelles instances soient Up.
- **Rolling Update** (statégie par défaut), détruit une instance et en recrée une nouvelle ainsi de suite jusqu'à ce que toutes les instances soient mises à jour. Bénéfice, maintien de la continuité d'accessibilité de l'application.
