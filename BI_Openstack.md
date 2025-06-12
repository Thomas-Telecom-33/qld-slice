
# SLICES BI Blueprint adaptation with Openstack

---

## ÉTAPE 1 — Installations nécessaires

### 1.1 Git clone

Accès au gitlab :
```bash
https://gitlab.ilabt.imec.be/slices-public/slices-bi-blueprint
```

Cloner le dépôt principal :

```bash
git clone git@gitlab.ilabt.imec.be:slices-public/slices-bi-blueprint.git
```
---

### 1.2 Ouverture du projet dans VSCode

Comme expliqué dans le README.md :
- Ouvrir le dépôt cloné dans VSCode.
- Lancer : `Reopen in Container (Dev Container)`
- Attendre que l’environnement se charge complètement.

---

### 1.3 Installation des dépendances Python

Dans le terminal à l’intérieur du conteneur :

```bash
pip install --upgrade pip
pip install -e .[dev,test]
```

---

## ÉTAPE 2 — Vérification du backend FastAPI

### 2.1 Initialisation de la base de données

Lancer la migration initiale :
```bash
alembic upgrade head
```

### 2.2 Démarrage du serveur FastAPI

```bash
python -m uvicorn slices_bi_blueprint_backend.app:app --host 0.0.0.0 --port 8000 --reload
```

Accès via navigateur :
- http://localhost:8000/docs
- http://localhost:8000/redoc

### 2.3 Vérification des logs

```bash
INFO:     127.0.0.1:34672 - "GET /redoc HTTP/1.1" 200 OK
INFO:     127.0.0.1:53914 - "GET /docs HTTP/1.1" 200 OK
```

---

## ÉTAPE 3 — Authentification via Slices CLI

### Installation
```bash
pip install slices-cli --extra-index-url=https://doc.slices-ri.eu/pypi/
```

### Connexion

```bash
slices auth login
```

### Commandes utiles

```bash
slices --help
slices project list
slices project use qkd
slices project show
slices experiment list
slices experiment create qkd-experiment
```

---

## ÉTAPE 4 — Obtention des tokens

- **Token utilisateur (GET uniquement)** :
```bash
slices auth id-token https://slices-bi-blueprint.ilabt.imec.be
```

- **Token d’expérience (GET & POST)** :
```bash
slices experiment jwt qkd-experiment
```

### Définir les administrateurs

Récupérer l’ID utilisateur :

```bash
slices project show
```

Modifier `.env` :
```env
ADMIN_USER_IDS=["user_account.ilabt.imec.be_..."]
```

---

## ÉTAPE 5 — Test des endpoints via Swagger UI

### 5.1 GET /disk-images et /flavors

1. Aller sur http://localhost:8000/docs
2. Cliquer sur "Authorize", coller un token (utilisateur ou expérience).
3. Lancer les endpoints de type GET.
4. Attendre un retour `200 OK`.

### 5.2 POST /disk-images et /flavors

Utiliser un **token d’expérience** (durée de vie courte, 15 min env.).

Exemple :
```json
{
  "friendly_name": "TEST",
  "cluster_id": "default",
  "location": "TEST",
  "default_username": "TEST",
  "min_disk_mb": 8000,
  "min_ram_mib": 512,
  "short_description": "string",
  "long_description": "string",
  "tags": [],
  "hidden": false
}
```

### 5.3 GET/POST /resources

Informations à récupérer :
```bash
slices experiment show qkd-experiment --format json
```

**Attention à la date `expires_at`** :

```bash
slices experiment show qkd-experiment
```

Choisir une date *avant* celle d’expiration de l’expérience, format :
```json
"expires_at": "2025-06-11T12:30:00Z"
```

Générer un `request_id` :
```bash
python3 -c "import uuid; print(uuid.uuid4())"
```

Token JWT utilisateur dans le body, JWT d’expérience dans le header `Authorization`.

### 5.4 GET /tasks/{id}

Récupérer l’`id` depuis la ressource POSTée :
```json
{
  "id": "task_tld-city-bp1_00sqtwepw69hev0qxd08d2f3dk",
  "status": "finished",
  "completed": 100,
  ...
}
```

---

## ÉTAPE 6 — Adaptation de la base de données

### Script de récupération OpenStack :

```python
import openstack

def main():
    conn = openstack.connect()

    print("IMAGES")
    for image in conn.image.images():
        print(f"\n ID: {image.id}")
        print(f"Name: {image.name}")
        print(f"Created At: {image.created_at}")
        print(f"Status: {image.status}")
        print(f"Visibility: {image.visibility}")
        print(f"Tags: {image.tags}")

    print("\n FLAVORS")
    for flavor in conn.compute.flavors():
        print(f"\n ID: {flavor.id}")
        print(f"Name: {flavor.name}")
        print(f"vCPUs: {flavor.vcpus}")
        print(f"RAM (MB): {flavor.ram}")
        print(f"Disk (GB): {flavor.disk}")
        print(f"Ephemeral (GB): {flavor.ephemeral}")
        print(f"Swap (MB): {flavor.swap}")
        print(f"RXTX Factor: {flavor.rxtx_factor}")
        print(f"Is Public: {flavor.is_public}")

if __name__ == "__main__":
    main()
```

### Ajout des éléments via API

Exemple image :
```json
{
  "friendly_name": "Ubuntu Server 20.04",
  "cluster_id": "default",
  "location": "ubuntu-server-20.04",
  "default_username": "ubuntu",
  "min_disk_mb": 8000,
  "min_ram_mib": 512,
  "short_description": "OpenStack",
  "long_description": "OpenStack image ID: XXXXX",
  "tags": [],
  "hidden": false
}
```

Exemple flavor :
```json
{
  "friendly_name": "Flavor-4",
  "description": "Openstack Flavor4 : ID : XXXXX",
  "flavor_type": "vm",
  "ram_mib": 2048,
  "vcpus": 2,
  "disk_mb": 8000
}
```

### Vérification via PostgreSQL

```bash
docker exec -it slices-bi-blueprint_devcontainer-db-1 psql -U slices_bi -d slices_bi
```
```sql
\dt
SELECT * FROM disk_images;
SELECT * FROM flavors;
```

---


## ÉTAPE 7 — Test de création de ressource via script

**Fichier :** `/examples/request-resources.sh`

Ce script permet de tester automatiquement la création d'une ressource via l’API.

### 7.1 Modification de l’image et du flavor

Avant exécution, adapter les champs suivants dans le corps JSON du script :

```json
"disk_image_id": "image_tld-city-bp1_{UUID_IMAGE}",
"flavor_id": "flavor_tld-city-bp1_{UUID_FLAVOR}"
```

### 7.2 Pour lister les UUIDs disponibles dans la base de données :

Accéder au conteneur PostgreSQL :

```bash
docker exec -it slices-bi-blueprint_devcontainer-db-1 psql -U slices_bi -d slices_bi
```

Puis exécuter les requêtes suivantes :

```sql
SELECT id, friendly_name FROM disk_images;
SELECT id, friendly_name FROM flavors;
```

Cela vous retournera les UUIDs à utiliser dans les IDs `disk_image_id` et `flavor_id` avec les préfixes correspondants.


### 7.3 Lancement du script

Le backend doit tourner :
```bash
python -m uvicorn slices_bi_blueprint_backend.app:app --host 0.0.0.0 --port 8000 --reload
```

Et un worker doit être lancer :
```bash
python -m taskiq worker slices_bi_blueprint_backend.worker:broker --reload --log-level=DEBUG
```

Puis on lance le script :
```bash
./request-resources.sh <experience_name>
```

> Remplacer `Nom_experience` par le nom de l’expérience à créer ou à utiliser.

### 7.4 Le script automatise les étapes suivantes :

- Authentification SLICES
- Sélection ou création du projet
- Création d’expérience
- Récupération des tokens (expérience + utilisateur)
- Création de la ressource via `POST /resources`


### 7.5 Résultat attendu

- Code HTTP `200`
- La ressource est visible via un GET sur `/resources`
- La tâche est traçable via un GET sur `/tasks/{task_id}`

> Pour vérifier la ressource via un appel `GET`, veillez à bien utiliser :
>
> - Le JWT d’expérience valide (attention : expire rapidement)
> - Le bon ID de l’expérience (`experiment_id`)

---

## ÉTAPE 8 — Tester la création d'une VM via script python

Pour permettre de se familiariser avec la bibiliothèque openstacksdk et de tester la création/suppresion de resources, on peut procéder ainsi :

### 8.1 Configuration de l'accès via le dashboard

Dans le tableau de bord OpenStack :  
**Projet > Accès API > Télécharger le fichier RC d'OpenStack**

Cela permet d'obtenir un fichier `clouds.yaml` contenant la configuration nécessaire pour se connecter à OpenStack avec la bibliothèque `openstacksdk`.

Il faut ensuite y ajouter manuellement la section `password`.

**Exemple simplifié de fichier `clouds.yaml`** :
```yaml
clouds:
  openstack:
    auth:
      auth_url: 
      username: your_username
      project_name: your_project
      user_domain_name: Default
      project_domain_name: Default
    region_name: RegionOne
    interface: public
    password: your_password
```

### 8.2 Ajout du fichier `clouds.yaml` dans la VM

Créer le dossier de configuration et y placer le fichier :
```bash
mkdir -p ~/.config/openstack
nano ~/.config/openstack/clouds.yaml
```

Puis coller le contenu du fichier `clouds.yaml`.

Installer la bibliothèque Python nécessaire :
```bash
pip install openstacksdk
```

### 8.3 Script Python de test : `test_create_vm.py`

Ce script permet de créer une VM OpenStack en utilisant les paramètres définis, il s'agit d'un script de test permettant de se familiariser avec la bilbliothèque openstacksdk :
```bash
import openstack
import base64

conn = openstack.connect(cloud='openstack')

IMAGE_NAME = ""
FLAVOR_NAME = ""
NETWORK_NAME = ""
VM_NAME = "VM_test"

USERDATA = """#cloud-config
password: xxxxxxx
chpasswd: { expire: False }
ssh_pwauth: True
package_update: true
package_upgrade: true
"""

print("[INFO] Looking for components")

image = conn.compute.find_image(IMAGE_NAME)
flavor = conn.compute.find_flavor(FLAVOR_NAME)
network = conn.network.find_network(NETWORK_NAME)

if not all([image, flavor, network]):
    raise Exception("[ERROR] Components are not found")

print("[INFO] Components found")

print("[INFO] VM is launching")

server = conn.compute.create_server(
    name=VM_NAME,
    image_id=image.id,
    flavor_id=flavor.id,
    networks=[{"uuid": network.id}],
    user_data=base64.b64encode(USERDATA.encode("utf-8")).decode("utf-8")
)

print("[INFO] Waiting...")
server = conn.compute.wait_for_server(server)

print(f"[INFO] VM {server.name} is ready (status={server.status})")

for net_name, addresses in server.addresses.items():
    for addr in addresses:
        print(f"{net_name} → {addr['addr']} ({addr['OS-EXT-IPS:type']})")
```

### 8.4 Résultats attendus

- **Terminal** : Affichage de l’état de la VM et de son adresse IP. Si tout se passe bien, un accès SSH est possible.
- **OpenStack Dashboard** : La VM apparaît dans la liste des instances.

---

## ÉTAPE 9 — Implémentation des fonctions OpenStack

L’objectif de cette étape est d’implémenter les fonctions permettant la **création** et la **suppression** de machines virtuelles (VMs) via OpenStack.

### 9.1 Création d'une ressource (VM)

La logique est répartie de manière claire dans deux fichiers distincts :

#### `infrastructure/openstack_backend.py`
Ce fichier contient **les appels directs à l’API OpenStack** via la librairie `openstacksdk`.

- Il gère la création d’une instance, l’assignation de la clé SSH, du flavor, de l’image, et l’injection du `cloud-config`.
- Il isole les dépendances avec OpenStack, facilitant les tests et la maintenance.

#### `tasks/compute_resource.py`
Ce fichier expose la fonction **`create_compute_resource`**.
Elle orchestre la création d’une VM.

#### Tester la création d’une VM

Utiliser le script fourni request-resources.sh :  
```bash
./example/request-resources.sh <exp_name>
```

Les paramètres attendus à l'intérieur :
- `cloud-config` : fichier de configuration cloud-init
- `vm-name` : nom de la machine à créer
- `image-id` : ID d’image OpenStack
- `flavor-id` : ID du flavor OpenStack
- `ssh-key` : nom de la clé SSH


### 9.2 Suppression d'une ressource (VM)

À faire :  
Compléter la fonction `delete_compute_resource` dans `tasks/compute_resource.py`  
et une méthode dédiée dans `openstack_backend.py` qui appelle `conn.compute.delete_server(...)` comme réalisée précédemment avec la fonction create.
Puis tester via SWAGUER.

