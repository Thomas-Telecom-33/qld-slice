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
sudo install minikube-linux-amd64 /usr/local/bin/minikube
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

> Crée un Pod nommé nginx-test.
> 
> Contient un seul container basé sur l'image officielle nginx.
>
> Expose le port 80 à l’intérieur du Pod.
>
> C’est la brique de base de Kubernetes, non répliquée, non auto-réparée.

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
k describe pod nginx-test
```

Voir les logs du conteneur dans le Pod :
```bash
k logs nginx-test
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

> Crée un Service de type NodePort nommé nginx-service.
>
> Relie les requêtes entrantes sur le port 30080 du node au port 80 du container NGINX.
>
> Utilise le label app=nginx pour cibler le bon Pod.
>
> Permet d’accéder à l’application depuis l’extérieur du cluster.

Label :
```bash
k label pod nginx-test app=nginx
```

On lance le service :
```bash
k apply -f nginx-service.yaml
```
> service/nginx-service created

On check :
```bash
k get svc
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

> Définit un Deployment nommé nginx-deployment.
>
> Lance 2 réplicas du Pod (pour la redondance).
>
> Gère automatiquement les mises à jour et remplacements de Pods.
>
> Repose sur les mêmes labels app=nginx pour la sélection.
>
> Utilise l’image nginx:latest (modifiable pour rolling update)

---

### 4.2. Appliquer le fichier de déploiement
```bash
k apply -f nginx-deployment.yaml
```

Affiche les déploiements actifs :
```bash
k get deployment
```

Supprimer un pod manuellement
Simule une panne pour tester l’auto-réparation :
```bash
k delete pod <pod_name>
```

Kubernetes recrée automatiquement un pod si nécessaire.

Lister les pods du déploiement
```bash
k get pods
```

---

### 4.3 Mise à jour avec un rolling update
Mettre à jour une application (changement d’image, de version, de config…) sans temps d’arrêt pour les utilisateurs.

Met à jour l'image de Nginx :
```bash
k set image deployment/nginx-deployment nginx=nginx:1.25
```

Suivre l'état de la mise à jour :
```bash
k rollout status deployment/nginx-deployment
```

Revenir à la version précédente si besoin :
```bash
k rollout undo deployment/nginx-deployment
```

Obtenir des détails
```bash
k describe deployment nginx-deployment
```

Supprimer le déploiement (optionnel)
```bash
k delete deployment nginx-deployment
```

═══════════════════════════════════════════════════════════════════════════════════

## 5. Namespace 

### Objectif :
Organiser les ressources dans un cluster Kubernetes en isolant les environnements (dev, test, prod...)

---

Lister les namespaces existants
```bash
k get namespaces
```

Créer un namespace spécifique
```bash
k create namespace demo
```
> namespace/demo created

Déployer un Pod dans ce namespace

Créer un fichier :
```bash
nano nginx-ns.yaml
```

Contenu :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
  namespace: demo
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

> Crée un Pod dans un namespace personnalisé (demo).
>
> Montre comment isoler les ressources dans différents environnements.
>
> Nécessite que le namespace demo soit créé au préalable.
>
> Fonctionne comme un Pod classique mais placé dans un espace logique dédié.

Appliquer la configuration dans le namespace
```bash
k apply -f nginx-ns.yaml
```
> pod/nginx-demo created

Vérifier que le pod est bien dans le namespace demo
```bash
k get pods -n demo
```
> nginx-demo   1/1     Running   0          23s

═══════════════════════════════════════════════════════════════════════════════════

## 6. ConfigMaps & Secrets

### Objectif :
Injecter des variables de configuration et des données sensibles dans les Pods Kubernetes de manière sécurisée.

---

### 6.1 ConfigMap

Une **ConfigMap** permet d'injecter des variables de configuration non sensibles dans des pods.  
Elle est utile pour externaliser les paramètres de configuration d'une application.

Créer une ConfigMap à partir de valeurs en ligne :
```bash
k create configmap demo-config \
  --from-literal=APP_ENV=development \
  --from-literal=APP_VERSION=1.0
```

Afficher le contenu YAML de la ConfigMap :
```bash
k get configmap demo-config -o yaml
```

---

### 6.2 Secret

Un **Secret** permet de stocker des données sensibles (mots de passe, clés API, etc.) de manière encodée.

Créer un Secret avec des valeurs sensibles :
```bash
k create secret generic demo-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cr3t
```

Afficher le contenu YAML du Secret :
```bash
kubectl get secret demo-secret -o yaml
```

---

### 6.3 Utiliser ConfigMap et Secret dans un Pod

Créer un fichier de configuration :
```bash
nano nginx-env.yaml
```

Contenu du fichier :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-config-demo
spec:
  containers:
    - name: nginx
      image: nginx
      env:
        - name: ENV
          valueFrom:
            configMapKeyRef:
              name: demo-config
              key: APP_ENV
        - name: VERSION
          valueFrom:
            configMapKeyRef:
              name: demo-config
              key: APP_VERSION
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: demo-secret
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: demo-secret
              key: DB_PASSWORD
```

> Crée un Pod nommé nginx-config-demo avec des variables d’environnement injectées.
>
> Récupère :
>
> - ENV et VERSION depuis une ConfigMap (demo-config).
>
> - DB_USERNAME et DB_PASSWORD depuis un Secret (demo-secret).
>
> Ce Pod montre comment découpler configuration et code, de manière sécurisée.

Déployer le Pod avec ConfigMap et Secret :
```bash
k apply -f nginx-env.yaml
```

---

### 6.4 Vérifier les variables injectées

Entrer dans le conteneur pour consulter les variables d'environnement :
```bash
k exec -it nginx-config-demo -- /bin/bash
```

Puis :
```bash
env | grep -E 'ENV|VERSION|DB_'
```

Exemple de sortie :
```bash
DB_PASSWORD=s3cr3t
ENV=development
DB_USERNAME=admin
VERSION=1.0
```

═══════════════════════════════════════════════════════════════════════════════════
