
# SLICES BI Blueprint adaptation with k8s

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 1 — Installations nécessaires
```bash
sudo apt update && sudo apt upgrade -y
```

---

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

Raccourcis pratiques :  
Crée un alias `k` pour `kubectl` et active l'autocomplétion :
```bash
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

---

### 1.2 Minikube
Outil pour exécuter un cluster Kubernetes local à des fins de test.

Télécharge puis installe `minikube` :
```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

---

### 1.3 Cluster Kubernetes

Créer un container docker pour Kubernetes :
```bash
minikube start --driver=docker
```

Vérification :
```bash
minikube status
```

Puis :
```bash
kubectl get nodes
```

Puis copie du fichier de configuration :
```bash
mkdir -p /home/ubuntu/shared-kube
cp ~/.kube/config /home/ubuntu/shared-kube/config
```

Configuration dans VSCode :

- docker-compose.yml :
```yaml
volumes:
  - ../..:/workspaces:cached
  - /home/ubuntu/shared-kube:/home/vscode/.kube:ro
  - ~/.minikube:/home/vscode/.minikube:ro
network_mode: host
```

- .env :
```
REDIS_SERVER=localhost
POSTGRES_SERVER=localhost
```

Rebuild du conteneur puis :
```bash
kubectl get nodes
```

═══════════════════════════════════════════════════════════════════════════════════

## Étape 2 — Création de kubernetes_backend.py

Créer `kubernetes_backend.py` dans `infrastructure`.

Objectif : permettre la gestion d’un pod (Kubernetes) via les appels backend.

Installation du client Kubernetes Python :
```bash
pip install kubernetes
```

Le fichier inclura :
- Fonction de création d’un pod (avec image depuis disk_image.location)
- Fonction de suppression du pod
- Utilisation du nom comme identifiant unique
- Namespace : `default`

═══════════════════════════════════════════════════════════════════════════════════

## Étape 3 — Adaptations dans tasks/compute_resources.py

Dans la fonction `create_compute_resource` :
- détecter si flavor correspond à un pod k8s (par ex. via son friendly_name ou autre stratégie)
- appeler `kubernetes_backend.create_pod(...)`

Dans la fonction `delete_compute_resource` :
- appeler `kubernetes_backend.delete_pod(...)` si c’est un pod K8s

═══════════════════════════════════════════════════════════════════════════════════

## Étape 4 — Ajout des images et flavors

### 4.1 Images

POST d’une image utilisée uniquement comme "référence logique" pour Kubernetes :

```json
{
  "friendly_name": "nginx-k8s",
  "cluster_id": "default",
  "location": "nginx",
  "default_username": "n/a",
  "min_disk_mb": 0,
  "min_ram_mib": 0,
  "short_description": "Docker image for Kubernetes",
  "long_description": "Official Docker image for NGINX, used for Kubernetes pod deployment.",
  "tags": ["k8s", "nginx", "docker"],
  "hidden": false
}
```

### 4.2 Flavors

POST d’un flavor dédié aux pods Kubernetes :

```json
{
  "friendly_name": "k8s-nginx",
  "description": "Kubernetes pod running Nginx",
  "flavor_type": "vm",
  "ram_mib": 0,
  "vcpus": 0,
  "disk_mb": 0
}
```

Note : on conserve `flavor_type: vm` pour éviter des modifications trop profondes dans le backend.

═══════════════════════════════════════════════════════════════════════════════════

## Étape 5 — Tests de création et suppression

### 5.1 POST

```json
{
  "expires_at": "2025-06-24T08:20:00Z",
  "resources": [
    {
      "friendly_name": "nginx-pod-test",
      "disk_image_id": "image_tld-city-bp1_xxx",
      "flavor_id": "flavor_tld-city-bp1_xxx",
      "network_name": "default",
      "userdata": ""
    }
  ],
  "ssh_authorized_keys": [
    "..."
  ],
  "tokens": [
    {
      "jwt": "..."
    }
  ]
}
```

Explication : même si le pod ne démarre pas d’image OpenStack, il faut fournir un `disk_image_id` et `flavor_id` pour rester cohérent avec les modèles REST.

Validation :
```bash
kubectl get pods
```

### 5.2 DELETE

Récupérer l’ID de la ressource, puis appeler l’API DELETE.

Validation :
```bash
kubectl get pods
```
Résultat : le pod ne figure plus dans la liste.

═══════════════════════════════════════════════════════════════════════════════════

## Conclusion

Le backend SLICES est désormais compatible avec `Kubernetes`.

#### Grâce à l’intégration mise en place, le système est capable de :

- `Créer` dynamiquement des pods sur demande,

- `Supprimer` proprement des pods existants,

- `Suivre` en temps réel l’état de chaque tâche grâce à Redis et TaskIQ.


#### Cette architecture repose sur Kubectl.

Il est désormais facile d’ajouter de nouvelles fonctionnalités.
