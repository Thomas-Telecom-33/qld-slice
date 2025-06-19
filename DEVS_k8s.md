
# SLICES BI Blueprint adaptation with k8s

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 1 — Installations nécessaires
sudo apt update && sudo apt upgrade -y

### 1.1 kubectl 
Outil en ligne de commande pour interagir avec un cluster Kubernetes.

Télécharge, rend exécutable et déplace `kubectl` dans le chemin système :
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Vérifie la version cliente installée de `kubectl` :
```bash
kubectl version --client
```
> Client Version: v1.33.1  
> Kustomize Version: v5.6.0

Raccourcis pratiques :  
Crée un alias `k` pour `kubectl` et active l'autocomplétion :
```bash
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

### 1.2 Minikube
Outil pour exécuter un cluster Kubernetes local à des fins de test.

Télécharge puis installe `minikube` :
```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```
> minikube version: v1.36.0  

### 1.3 Cluster Kubernets

Créer un container docker pour Kubernets :
```bash
minikube start --driver=docker
```

Check :
```bash
minikube status
```
> minikube
> type: Control Plane
> host: Running
> kubelet: Running
> apiserver: Running
> kubeconfig: Configured

Puis :
```bash
k get nodes
```
> minikube   Ready    control-plane   55s   v1.33.1
> On a alors un noeud minikube avec le statut Ready.

Puis :
```bash
mkdir -p ~/kube-share
cp ~/.kube/config ~/kube-share/config
```

Côté Vscode :

- docker-compose.yml à adapter :
Les volumes :
```python
volumes:
      - ../..:/workspaces:cached
      - ~/.kube:/home/ubuntu/.kube:ro
      - ~/.minikube:/home/ubuntu/.minikube:ro
```

- dockerfile à adapter :
  Avec les commandes vus précédemment.




