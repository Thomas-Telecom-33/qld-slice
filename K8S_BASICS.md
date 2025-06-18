# K8S_BASICS

═══════════════════════════════════════════════════════════════════════════════════

## 1. Installations nécessaires

Créer un dossier de travail pour le projet Kubernetes
```bash
mkdir k8s_lab
```
---

### 1.1 DOCKER :
Outil permettant d'exécuter des conteneurs en local.

Mettre à jour les paquets, installe Docker, puis l’active au démarrage du système :
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

Ajoute l'utilisateur au groupe docker, recharge le groupe et teste l’installation avec une image de démonstration :
```bash
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

---

### 1.2 kubectl 
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

---

### 1.3 Minikube 
Outil pour exécuter un cluster Kubernetes local à des fins de test.

Télécharge puis installe `minikube` :
```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
minikube version
```
> minikube version: v1.36.0  

═══════════════════════════════════════════════════════════════════════════════════

## 2. Initialisation

### 2.1 Lancer le cluster :
Démarre un cluster Kubernetes local avec `minikube` :
```bash
minikube start
```

Affiche les nœuds disponibles dans le cluster :
```bash
kubectl get nodes
```
ou
```bash
k get nodes
```

---

### 2.2 Créer un premier Pod
Un Pod est l’unité de déploiement la plus petite dans Kubernetes. Il encapsule un ou plusieurs conteneurs.
On créé un pod NGINX avec un fichier YAML :

Créer un fichier de configuration pour un Pod NGINX :
```bash
nano nginx-pod.yaml
```

Contenu du fichier :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

Déployer le Pod dans le cluster :
```bash
k apply -f nginx-pod.yaml
```

Vérifie les pods déployés :
```bash
k get pods
```

Afficher les détails du Pod :
```bash
kubectl describe pod nginx-test
```

Voir les logs du conteneur dans le Pod :
```bash
kubectl logs nginx-test
```

On a notre premier pod :
> nginx-test   1/1     Running   0   16m

═══════════════════════════════════════════════════════════════════════════════════

## 3. Services
Les pods sont éphémères et non accessibles de l’extérieur par défaut.
Un Service expose un ou plusieurs pods sur un port stable.

Fichier de configuration :
```bash
nano nginx-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

Label :
```bash
kubectl label pod nginx-test app=nginx
```

On lance le service :
```bash
k apply -f nginx-service.yaml
```
> service/nginx-service created

On check :
```bash
kubectl get svc
```
> nginx-service   NodePort  ...

Obtenir l’adresse IP de Minikube :
```bash
minikube ip
```
> 192.168.49.2

Créer un tunnel vers le service :
```bash
minikube service nginx-service
```
On peut accéder à NGINX via le navigateur à une URL comme :
> http://127.0.0.1:36615/

═══════════════════════════════════════════════════════════════════════════════════

## 4. Deployment

Un **Deployment** est un objet Kubernetes qui :
- Gère la création et la mise à jour de Pods.
- Permet de déployer une réplique de ton application (1, 2, 3 pods… ou plus).
- Gère le remplacement automatique d’un pod si un conteneur tombe (self-healing).
- Permet des mises à jour sans interruption (rolling updates).

---

### 4.1. Créer un fichier de configuration `nginx-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

---

### 4.2. Appliquer le fichier de déploiement
```bash
kubectl apply -f nginx-deployment.yaml
```

Affiche les déploiements actifs :
```bash
k get deployment
```

Supprimer un pod manuellement
Simule une panne pour tester l’auto-réparation :
```bash
kubectl delete pod <pod_name>
```

Kubernetes recrée automatiquement un pod si nécessaire.

Lister les pods du déploiement
```bash
k get pods
```

---

### Mise à jour avec un rolling update
Met à jour l'image de Nginx :
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
```

Suivre l'état de la mise à jour :
```bash
kubectl rollout status deployment/nginx-deployment
```

Revenir à la version précédente si besoin :
```bash
kubectl rollout undo deployment/nginx-deployment
```

4.8. Obtenir des détails
```bash
kubectl describe deployment nginx-deployment
```

Supprimer le déploiement (optionnel)
```bash
kubectl delete deployment nginx-deployment
```
