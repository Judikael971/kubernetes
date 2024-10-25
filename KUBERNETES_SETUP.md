# Installation Kubernetes Cluster On-Premise via kubeadm

### Prérequis: 

Il existe **deux types de serveurs** utilisés dans le déploiement de clusters Kubernetes:

• **Master**: Un Master **(control plane)** est l'endroit où les appels d'API de contrôle pour les pods, les contrôleurs de réplication, les services, les nœuds et autres composants d'un cluster Kubernetes sont exécutés.

• **Node**: Un Node (nœuds) est un système qui fournit les environnements d'exécution pour les conteneurs. Un ensemble de pods de conteneur peut s'étendre sur plusieurs nodes.


#### Les exigences minimales en configuration machine sont:

• **Memory**: 2 GiB ou plus de RAM par machine

• **CPUs**: Au moins 2 processeurs sur la machine control plane **Master**

• **Connexion Internet** 

• **Connectivité réseau complète entre les machines du cluster** – Qu'elle soit privée ou public


### Ports et Protocoles

**Master (control plane)**
| Protocol	| Direction	| Port Range	| Purpose	| Used By | 
| ----------|-----------|---------------|-----------|---------|
| TCP	| Inbound	| 6443	| Kubernetes API server	| All | 
| TCP	| Inbound	| 2379-2380	| etcd server client API	| kube-apiserver, etcd | 
| TCP	| Inbound	| 10250	| Kubelet API	| Self, Control plane | 
| TCP	| Inbound	| 10259	| kube-scheduler	| Self | 
| TCP	| Inbound	| 10257	| kube-controller-manager	| Self | 

**Worker node(s)**
| Protocol	| Direction	| Port Range	| Purpose	| Used By | 
| ----------|-----------|---------------|-----------|---------|
| TCP	| Inbound	| 10250	| Kubelet API	| Self, Control plane | 
| TCP	| Inbound	| 10256	| kube-proxy	| Self, Load balancers | 
| TCP	| Inbound	| 30000-32767	| NodePort Services	| All | 

## ---------- Steps ----------

### Step 1. Configurer le serveur Debian 12

Mise à jour du serveur

```
sudo apt update
sudo apt -y full-upgrade
sudo reboot -f
```

Renommage du serveur (optionnel) 
```
cat /etc/hostname
sudo hostnamectl set-hostname NOM_SERVEUR
sudo systemctl restart systemd-hostnamed
```

Définition d'hosts pour les IP du cluster (optionnel) 
```
cat /etc/hosts
sudo vim /etc/hosts
```

Ajouter les hosts
```
192.168.0.1 HOSTNAME_MASTER
192.168.0.2 HOSTNAME_WORKER
...
```
Mettre l'IP des serveurs


### Step 2. Désactivation du Swap

Eteindre le swap.

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Désactiver définitivement l'espace d'échange Linux dans `/etc/fstab`. Rechercher la ligne de swap et commentez la grace au signe # (hashtag)

```
$ sudo vim /etc/fstab
#/swap.img none swap sw 0 0
```

Vérifier si le réglage est correct

```
sudo swapoff -a
sudo mount -a
free -h
```

Activer les modules du kernel et configurer `sysctl`

```
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system
```


### Step 3. Installation d'un Container Runtime

Pour exécuter des conteneurs dans des pods, Kubernetes utilise un environnement d'exécution de conteneur (container runtime). Les container runtimes supportés sont:

• Docker (version 18.09 or later)

• containerd (version 1.2.2 or later)

• CRI-O (version 1.11 or later)

**NOTE: Vous devez choisir un runtime à la fois.**


#### - Installation de CRI-O 

Installer les dépendances pour ajouter des repositories
```
sudo apt update
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

Définition de la variable d'environnement "CRI-O VERSION"
```
vim .bashrc
```

Ajout de la variable `export CRIO_VERSION="v1.30"`.

Prise en charge de l'export

```
source .bashrc
```

Ajout du repository de CRI-O
```
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list
```

Installation du package cri-o
```
apt-get update
apt-get install -y cri-o
```

Lancement du service CRI-O
```
systemctl start crio.service
```

source [github CRI-O](https://github.com/cri-o/packaging/blob/main/README.md#distributions-using-deb-packages)


### Step 4. Installation de kubelet, kubeadm and kubectl

Définition de la variable d'environnement "KUBERNETES VERSION"
```
vim .bashrc
```

Ajout de la variable `export KUBERNETES_VERSION="v1.31"`.

Prise en charge de l'export

```
source .bashrc
```

Ajoutez le repository Kubernetes sur tous les serveurs du futur cluster.

```
sudo apt install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
```

Installer les packages requis.

```
sudo apt update
sudo apt install -y vim git curl wget kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

Vérifier l'installation en controlant la version de kubectl.

```
$ kubectl version --client && kubeadm version

### OUTPUT
Client Version: v1.31.2
Kustomize Version: v5.4.2
kubeadm version: &version.Info{Major:"1", Minor:"31", GitVersion:"v1.31.2", GitCommit:"5864a4677267e6adeae276ad85882a8714d69d9d", GitTreeState:"clean", BuildDate:"2024-10-22T20:33:59Z", GoVersion:"go1.22.8", Compiler:"gc", Platform:"linux/amd64"}
```

### Step 5. Initialisation du Master node

Se connecter au serveur à utiliser comme `master` et s'assurer que le module `br_netfilter` est chargé :

```
lsmod | grep br_netfilter
```

Activer le service kubelet.

```
sudo systemctl enable kubelet
```

Initialiser la machine qui exécutera les composants du `control plane` qui incluent `etcd` (la base de données du cluster) et le serveur API.

Pull container images

```
sudo kubeadm config images pull
```

**NOTE**: Si vous avez de plusieurs sockets CRI installé, veuillez utiliser **--cri-socket** pour en sélectionner une :

```
# CRI-O
sudo kubeadm config images pull --cri-socket unix:///var/run/crio/crio.sock 
```

Options de base utilisées pour amorcer le cluster (**kubeadm init**).

```
--control-plane-endpoint : endpoint partagé pour tous les nœuds du control plane. (Peut être DNS ou IP)

--pod-network-cidr : Used to set a Pod network add-on CIDR

--cri-socket : À utiliser si vous avez de plusieurs sockets CRI installé, pour définir le chemin du socket d'exécution

--apiserver-advertise-address : Set advertise address for this particular control-plane node's API server
```

Il existe 2 façons d'amorcer un cluster.

**Amorçage du cluster dans endpoint partagé**

```
### With CRI-O ###
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///var/run/crio/crio.sock  \
  --upload-certs \
  --control-plane-endpoint=IP_MASTER_NODE
```

(**NOTE**: If you restart your system, then swap would be enabled again. So, you have to disable it again.)

Le résultat de la commande d'initialisation.

```
....
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join IP_MASTER_NODE:6443 --token erfn0x.XXXXXXXXXXXX \
 --discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX \
 --control-plane --certificate-key XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join IP_MASTER_NODE --token erfn0x.XXXXXXXXXXXX \
 --discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Configurer `kubectl` à l'aide des commandes dans le résultat :

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Controler les status du cluster:

```
kubectl cluster-info

## Output
Kubernetes control plane is running at https://IP_MASTER_NODE:6443

CoreDNS is running at https://IP_MASTER_NODE:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

**NOTE (Not necessary)**: Open the control plane at  <https://IP_MASTER_NODE:6443>, if you get a **Forbidden** error, run the following

```
kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous

###
Ref: https://www.edureka.co/community/34714/code-error-403-when-trying-to-access-kubernetes-cluster
```

Additional Master nodes(Control Plane) can be added using the command in installation output:

```
kubeadm join IP_MASTER_NODE:6443 --token erfn0x.zjvydfubby17yxum \
 --discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX \
 --control-plane --certificate-key XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Step 6. Install network plugin on Master

In this guide we’ll use Calico. You can choose any other supported network plugins.

```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml 

kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

You should see the following output.

```
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
.....
installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created
```

Confirm that all of the pods are running:

```
watch kubectl get pods --all-namespaces
```

Confirm master node is ready:

```
# Docker
$ kubectl get nodes -o wide

### Ouput
NAME       STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION         CONTAINER-RUNTIME
HOSTNAME   Ready    control-plane   5m10s   v1.31.2   IP_MASTER_NODE  <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-21-cloud-amd64   cri-o://1.30.6
```

### Step 7. Add Worker nodes

With the control plane ready you can add worker nodes to the cluster for running scheduled workloads.

**NOTE:** Run this only if your worker was used as master before:

```
sudo kubeadm reset -f --cri-socket unix:///var/run/crio/crio.sock
```

Enable root user

```
sudo su
```

To join, on the worker node, run the following: -

```
kubeadm join IP_MASTER_NODE:6443 --token erfn0x.zjvydfubby17yxum \
--discovery-token-ca-cert-hash sha256:6ffbebf035b8817b62a76ded4eaba2bf52cf20ba1c62afbe0bae2f98818bf8f6 \ 
--cri-socket unix:///var/run/crio/crio.sock \ 
--v=2
```

Confirm worker nodes are ready:

```
kubectl get nodes -o wide

### Output
NAME         STATUS   ROLES           AGE   VERSION
etalddqm01   Ready    <none>          15m   v1.25.4
etalddqm02   Ready    control-plane   25m   v1.25.4
```

### Step 8: Deploy application on cluster

Now that the worker has joined the control plane. We don\t need to interact directly with it anymore.

To validate that our cluster is working by deploying an application. Run this on the master node.

```
kubectl apply -f https://k8s.io/examples/pods/commands.yaml
```

Check to see if pod started

```
kubectl get pods

### Output
NAME           READY   STATUS      RESTARTS   AGE
command-demo   0/1     Completed   0          7m29s
```

### Step 9: Install Kubernetes Dashboard

Kubernetes dashboard can be used to deploy containerized applications to a Kubernetes cluster, troubleshoot your containerized application, and manage the cluster resources.

The deployment of Deployments, StatefulSets, DaemonSets, Jobs, Services and Ingress can be done from the dashboard or from the terminal with kubectl. if you want to scale a Deployment, initiate a rolling update, restart a pod, create a persistent volume and persistent volume claim, you can do all from the Kubernetes dashboard.

- **Step 9A**: Configure kubectl

Reference: <https://computingforgeeks.com/how-to-install-kubernetes-dashboard-with-nodeport/>

- **Step 9B**: Deploy Kubernetes Dashboard

The default Dashboard deployment contains a minimal set of RBAC privileges needed to run. You can deploy Kubernetes dashboard with the command below.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

#### Set Service to use NodePort

This will use the default values for the deployment. The services are available on ClusterIPs only as can be seen from the output below:

```
$ kubectl get svc -n kubernetes-dashboard

### Output
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.110.230.178   <none>        8000/TCP   32s
kubernetes-dashboard        ClusterIP   10.101.229.22    <none>        443/TCP    32s
```

Patch the service to have it listen on NodePort:

```
kubectl --namespace kubernetes-dashboard patch svc kubernetes-dashboard -p '{"spec": {"type": "NodePort"}}'

### Output
service/kubernetes-dashboard patched
```

Confirm the new setting:

```
kubectl get svc -n kubernetes-dashboard kubernetes-dashboard -o yaml

### Output
...
...
spec:
  clusterIP: 10.101.229.22
  clusterIPs:
  - 10.101.229.22
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 32502
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

- **NodePort** exposes the Service on each Node’s IP at a static port (the NodePort). A ClusterIP Service, to which the NodePort Service routes, is automatically created.

Let us change the nodePort from 32502 to 32000 and apply the patch. Create a new file called `nodeport_dashboard_patch.yaml`

```
nano nodeport_dashboard_patch.yaml
```

Add the following in the file.

```
spec:
  ports:
  - nodePort: 32000
    port: 443
    protocol: TCP
    targetPort: 8443
```

Apply the patch

```
kubectl -n kubernetes-dashboard patch svc kubernetes-dashboard --patch "$(cat nodeport_dashboard_patch.yaml)"

### Output
service/kubernetes-dashboard patched
```

Check deployment status:

```
kubectl get deployments -n kubernetes-dashboard

### Output
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
dashboard-metrics-scraper   1/1     1            1           11m
kubernetes-dashboard        1/1     1            1           11m
```

Two pods should be created – One for dashboard and another for metrics.

```
kubectl get pods -n kubernetes-dashboard

### Output
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-64bcc67c9c-hgfhj   1/1     Running   0          12m
kubernetes-dashboard-5c8bd6b59-j665z         1/1     Running   0          12m
```

Since we changed service type to NodePort, let’s confirm if the service was actually created.

```
kubectl get service -n kubernetes-dashboard

### Output
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.110.230.178   <none>        8000/TCP        12m
kubernetes-dashboard        NodePort    10.101.229.22    <none>        443:32000/TCP   12m
```

- **Step 9C**: Accessing Kubernetes Dashboard

#### Port Forward Method

We can access the dashboard many ways (Refer: <https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md>)

We will be using the `kubectl-port-forward` method here

Run the following command

```
kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard 8080:443
```

To access Kubernetes Dashboard go to:

```
https://localhost:8080
```

#### NodePort method

Since we changed the config from ClusterIP to NodePort. In case you are trying to expose Dashboard using NodePort on a multi-node cluster, then you have to find out IP of the node on which Dashboard is running to access it. Instead of accessing https://<master-ip>:<nodePort> you should access https://<node-ip>:<nodePort>.

- To get Node IP, run he following

```
kubectl get nodes -o wide

### Output
NAME         STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
etalddqm01   Ready    <none>          4h7m    v1.25.4   10.3.15.121   <none>        Ubuntu 20.04.5 LTS   5.4.0-132-generic   docker://20.10.21
etalddqm02   Ready    control-plane   4h17m   v1.25.4   IP_MASTER_NODE   <none>        Ubuntu 18.04.6 LTS   5.4.0-132-generic   docker://20.10.21
```

Our control plane NodeIP is `10.3.15.121`

**NOTE**: The dashboard might be running on worker node too, so try opening the dashboard on its NodeIP.

To view dashboard, open: `https://10.3.15.121:32000`

#### Create admin-user to generate token & access the dashboard

(1) Create a `dashboard-admin.yaml`

`nano dashboard-admin.yaml`

(2) Add the following inside the yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

(3) Apply the config

```
kubectl apply -f dashboard-admin.yaml

### Output
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

(4) Create a token

```
kubectl -n kubernetes-dashboard create token admin-user
```

Paste the token in the dashboard URL.

**Read More**: <https://komodor.com/learn/kubernetes-dashboard/>

### Step 10: Setup Prometheus and Grafana on Kubernetes using prometheus-operator

Monitoring Production Kubernetes Cluster(s) is an important and progressive operation for any Cluster Administrator.

Prometheus is a full fledged solution that enables Developers and SysAdmins to access advanced metrics capabilities in Kubernetes. The metrics are collected in a time interval of 30 seconds, this is a default settings. The information collected include resources such as Memory, CPU, Disk Performance and Network IO as well as R/W rates. By default the metrics are exposed on your cluster for up to a period of 14 days, but the settings can be adjusted to suit your environment.

Grafana is used for analytics and interactive visualization of metrics that’s collected and stored in Prometheus database. You can create custom charts, graphs, and alerts for Kubernetes cluster, with Prometheus being data source. In this guide we will perform installation of both Prometheus and Grafana on a Kubernetes Cluster. For this setup kubectl configuration is required, with Cluster Admin role binding.

To get a complete an entire monitoring stack we will use kube-prometheus project which includes Prometheus Operator among its components. The kube-prometheus stack is meant for cluster monitoring and is pre-configured to collect metrics from all Kubernetes components, with a default set of dashboards and alerting rules.

You should have kubectl configured and confirmed to be working:

```
kubectl cluster-info

### Output
Kubernetes control plane is running at https://10.80.55.156:6443
CoreDNS is running at https://10.80.55.156:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

#### Step 1: Clone kube-prometheus project

```
git clone https://github.com/prometheus-operator/kube-prometheus.git
```

Navigate to the kube-prometheus directory:

```
cd kube-prometheus
```

#### Step 2: Create monitoring namespace, CustomResourceDefinitions & operator pod

- Create a namespace and required CustomResourceDefinitions:

```
kubectl create -f manifests/setup

### Output
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com created
namespace/monitoring created
```

- The namespace created with CustomResourceDefinitions is named monitoring

```
kubectl get ns monitoring

### Output
NAME          STATUS   AGE
monitoring    Active   2m
```

#### Step 3: Deploy Prometheus Monitoring Stack on Kubernetes

- Once you confirm the Prometheus operator is running you can go ahead and deploy Prometheus monitoring stack.

```
kubectl create -f manifests/

### Output
...
...
...
networkpolicy.networking.k8s.io/prometheus-operator created
prometheusrule.monitoring.coreos.com/prometheus-operator-rules created
service/prometheus-operator created
serviceaccount/prometheus-operator created
servicemonitor.monitoring.coreos.com/prometheus-operator created
```

Give it few seconds and the pods should start coming online. This can be checked with the commands below:

```
kubectl get pods -n monitoring

### Output
NAME                                   READY   STATUS    RESTARTS        AGE
alertmanager-main-0                    1/2     Running   1 (77s ago)     3m53s
alertmanager-main-1                    2/2     Running   1 (3m34s ago)   3m53s
alertmanager-main-2                    1/2     Running   1 (77s ago)     3m53s
blackbox-exporter-59cccb5797-b2nlg     3/3     Running   0               5m39s
grafana-5c6cc77844-nvc2m               1/1     Running   0               5m38s
kube-state-metrics-84db6cc79c-wg97l    3/3     Running   0               5m38s
node-exporter-4nx8n                    2/2     Running   0               5m38s
node-exporter-xzhhf                    2/2     Running   0               5m38s
prometheus-adapter-757f9b4cf9-5jkhd    1/1     Running   0               5m38s
prometheus-adapter-757f9b4cf9-ndzsv    1/1     Running   0               5m38s
prometheus-k8s-0                       2/2     Running   0               3m51s
prometheus-k8s-1                       2/2     Running   0               3m51s
prometheus-operator-7cf95bc44c-bvmfz   2/2     Running   0               5m38s
```

- To list all the services created you’ll run the command:

```
kubectl get svc -n monitoring

### Output
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.107.33.62     <none>        9093/TCP,8080/TCP            7m59s
alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   6m13s
blackbox-exporter       ClusterIP   10.103.16.46     <none>        9115/TCP,19115/TCP           7m59s
grafana                 ClusterIP   10.106.147.243   <none>        3000/TCP                     7m58s
kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            7m58s
node-exporter           ClusterIP   None             <none>        9100/TCP                     7m58s
prometheus-adapter      ClusterIP   10.98.64.212     <none>        443/TCP                      7m58s
prometheus-k8s          ClusterIP   10.107.153.143   <none>        9090/TCP,8080/TCP            7m58s
prometheus-operated     ClusterIP   None             <none>        9090/TCP                     6m11s
prometheus-operator     ClusterIP   None             <none>        8443/TCP                     7m58s
```

#### Step 4: Access Prometheus, Grafana, and Alertmanager dashboards

We can access the dashboards using 2 methods:

- Using `kubectl proxy` (Not secure in production)

- Using `NodePort / LoadBalancer` (Secure in production)

##### Let us use the NodePort method to access the dashboards

To access Prometheus, Grafana, and Alertmanager dashboards using one of the worker nodes IP address and a port you’ve to edit the services and **set the type to NodePort**.

You need a **Load Balancer** implementation in your cluster to use service type LoadBalancer. Refer: <https://computingforgeeks.com/deploy-metallb-load-balancer-on-kubernetes/>

- Patch **Prometheus** config from ClusterIP to NodePort

```
kubectl --namespace monitoring patch svc prometheus-k8s -p '{"spec": {"type": "NodePort"}}'

### Output
kubectl --namespace monitoring patch svc prometheus-k8s -p '{"spec": {"type": "NodePort"}}'
```

- Patch **Alertmanager** config from ClusterIP to NodePort

```
kubectl --namespace monitoring patch svc alertmanager-main -p '{"spec": {"type": "NodePort"}}'

### Output
kubectl --namespace monitoring patch svc alertmanager-main -p '{"spec": {"type": "NodePort"}}'
```

- Patch **Grafana** config from ClusterIP to NodePort

```
kubectl --namespace monitoring patch svc grafana -p '{"spec": {"type": "NodePort"}}'

### Output
kubectl --namespace monitoring patch svc grafana -p '{"spec": {"type": "NodePort"}}
```

Confirm that the each of the services have a Node Port assigned / Load Balancer IP addresses:

```
kubectl -n monitoring get svc  | grep NodePort

### Output
alertmanager-main       NodePort    10.107.33.62     <none>        9093:31377/TCP,8080:32704/TCP   17m
grafana                 NodePort    10.106.147.243   <none>        3000:30705/TCP                  17m
prometheus-k8s          NodePort    10.107.153.143   <none>        9090:30303/TCP,8080:31012/TCP   17m
```

We can access the services as below:

```
# Grafana
NodePort: http://node_ip:30705
For us: https://10.80.55.156:30705

# Prometheus
NodePort: http://node_ip:30303
For us: http://10.80.55.156:30303

# Alert Manager
NodePort: http://node_ip:31377
For us: http://10.80.55.156:31377
```

#### Destroying / Tearing down Prometheus monitoring stack

If at some point you feel like tearing down Prometheus Monitoring stack in your Kubernetes Cluster, you can run kubectl delete command and pass the path to the manifest files we used during deployment.

```
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```

Reference: <https://computingforgeeks.com/setup-prometheus-and-grafana-on-kubernetes/>

### --------- Additional Commands ---------

### - Remove a worker node from the Kubernetes cluster

(1) List the nodes and get the <node-name> you want to from the cluster

```

kubectl get nodes -o wide

### Output

NAME                STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
dxbdrvtkubernetes   Ready    control-plane   20h   v1.25.4   10.80.55.156   <none>        Ubuntu 20.04.2 LTS   5.4.0-131-generic   docker://20.10.21
etalddqm02          Ready    <none>          20h   v1.25.4   IP_MASTER_NODE    <none>        Ubuntu 18.04.6 LTS   5.4.0-132-generic   docker://20.10.21

```

(2)  Drain or (remove from the cluster) the node from the **control plane**

```

kubectl drain etalddqm02 --delete-local-data --force --ignore-daemonsets

```

(3) Now, from the **worker node** to be removed, un-configure kubernetes

```

kubeadm reset -f --cri-socket unix:///var/run/crio/crio.sock

```

(4) Go back to the **control plane node** and delete the worker node

```

kubectl delete node etalddqm02

```

### --------- TROUBLESHOOT ---------

- To stop/reset a Kubernetes Cluster, run the following command on the master:

```

sudo kubeadm reset -f --cri-socket unix:///var/run/crio/crio.sock

```

- To get **Node IP**: `kubectl get nodes -o wide`

- To get **Cluster IP**: `kubectl cluster-info`

**Reference**: <https://computingforgeeks.com/deploy-kubernetes-cluster-on-ubuntu-with-kubeadm/>
