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

# Log

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
