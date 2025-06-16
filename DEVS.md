
# SLICES BI Blueprint adaptation with Openstack

═══════════════════════════════════════════════════════════════════════════════════

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

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 2 — Vérification du backend FastAPI

### 2.1 Initialisation de la base de données

Lancer la migration initiale :
```bash
alembic upgrade head
```
---

### 2.2 Démarrage du serveur FastAPI

```bash
python -m uvicorn slices_bi_blueprint_backend.app:app --host 0.0.0.0 --port 8000 --reload
```

Accès via navigateur :
- http://localhost:8000/docs
- http://localhost:8000/redoc

---

### 2.3 Vérification des logs

```bash
INFO:     127.0.0.1:34672 - "GET /redoc HTTP/1.1" 200 OK
INFO:     127.0.0.1:53914 - "GET /docs HTTP/1.1" 200 OK
```

═══════════════════════════════════════════════════════════════════════════════════

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

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 4 — Obtention des tokens

- **Token utilisateur (GET uniquement)** :
```bash
slices auth id-token https://slices-bi-blueprint.ilabt.imec.be
```

- **Token d’expérience (GET & POST)** :
```bash
slices experiment jwt qkd-experiment
```
---

### Définir les administrateurs

Récupérer l’ID utilisateur :

```bash
slices project show
```

Modifier `.env` :
```env
ADMIN_USER_IDS=["user_account.ilabt.imec.be_..."]
```

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 5 — Test des endpoints via Swagger UI

### 5.1 GET /disk-images et /flavors

1. Aller sur http://localhost:8000/docs
2. Cliquer sur "Authorize", coller un token (utilisateur ou expérience).
3. Lancer les endpoints de type GET.
4. Attendre un retour `200 OK`.

---

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
---

### 5.3 GET/POST /resources

Informations à récupérer :
```bash
slices experiment show qkd-experiment
```

**Attention à la date `expires_at`** :

Choisir une date *avant* celle d’expiration de l’expérience, format :
```json
"expires_at": "2025-06-11T12:30:00Z"
```

Générer un `request_id` :
```bash
python3 -c "import uuid; print(uuid.uuid4())"
```

Token JWT utilisateur dans le body, JWT d’expérience dans le header `Authorization`.

---

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

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 6 — Adaptation de la base de données

### Installer la bibliothèque Python nécessaire :
```bash
pip install openstacksdk
```

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

---

### Vérification via PostgreSQL

```bash
docker exec -it slices-bi-blueprint_devcontainer-db-1 psql -U slices_bi -d slices_bi
```
```sql
\dt
SELECT * FROM disk_images;
SELECT * FROM flavors;
```

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 7 — Test de création de ressource via script

**Fichier :** `/examples/request-resources.sh`

Ce script permet de tester automatiquement la création d'une ressource via l’API.

---

### 7.1 Modification de l’image et du flavor

Avant exécution, adapter les champs suivants dans le corps JSON du script :

```json
"disk_image_id": "image_tld-city-bp1_{UUID_IMAGE}",
"flavor_id": "flavor_tld-city-bp1_{UUID_FLAVOR}"
```

---

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

---

### 7.3 Lancement du script

Le backend doit tourner :
```bash
python -m uvicorn slices_bi_blueprint_backend.app:app --host 0.0.0.0 --port 8000 --reload
```

Et un worker doit être lancer :
```bash
python -m taskiq worker slices_bi_blueprint_backend.worker:broker --reload --log-level=DEBUG
```

Et taskIQ peut être lancer :
```bash
python -m taskiq worker slices_bi_blueprint_backend.worker:scheduler --log-level=DEBUG
```

Puis on lance le script de test :
```bash
./request-resources.sh <experience_name>
```

> Remplacer `experience_name` par le nom de l’expérience à créer ou à utiliser.

---

### 7.4 Le script automatise les étapes suivantes :

- Authentification SLICES
- Sélection du projet
- Création d’expérience
- Récupération des tokens (expérience + utilisateur)
- Création de la ressource via `POST /resources`

---

### 7.5 Résultat attendu

- Code HTTP `200`
- La ressource est visible via un GET sur `/resources`
- La tâche est traçable via un GET sur `/tasks/{task_id}`

> Pour vérifier la ressource via un appel `GET`, veillez à bien utiliser :
>
> - Le JWT d’expérience valide (attention : expire rapidement)
> - Le bon ID de l’expérience (`experiment_id`)

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 8 — Tester la création d'une VM via script python

Pour permettre de se familiariser avec la bibiliothèque openstacksdk et de tester la création/suppresion de resources, on peut procéder ainsi :

---

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
      password: your_password
      project_name: your_project
      user_domain_name: Default
      project_domain_name: Default
    region_name: RegionOne
    interface: public
```

---

### 8.2 Ajout du fichier `clouds.yaml` dans la VM

Créer le dossier de configuration et y placer le fichier :
```bash
mkdir -p ~/.config/openstack
nano ~/.config/openstack/clouds.yaml
```

Puis coller le contenu du fichier `clouds.yaml`.

---

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

---

### 8.4 Résultats attendus

- **Terminal** : Affichage de l’état de la VM et de son adresse IP. Si tout se passe bien, un accès SSH est possible.
- **OpenStack Dashboard** : La VM apparaît dans la liste des instances.

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 9 — Implémentation des fonctions OpenStack

L’objectif de cette étape est d’implémenter les fonctions permettant la **création** et la **suppression** de machines virtuelles (VMs) via OpenStack.

---

### 9.1 Création d'une ressource (VM)

La logique est répartie de manière claire dans deux fichiers distincts :


#### `infrastructure/openstack_backend.py`
Ce fichier contient **les appels directs à l’API OpenStack** via la librairie `openstacksdk`.

- Il gère la connexion à OpenStack, la création d’une instance, l’assignation de la clé SSH, du flavor, de l’image, et l’injection du `cloud-init` via `userdata`.
- Ce fichier encapsule toutes les dépendances avec OpenStack, ce qui facilite la maintenance et les tests.


#### `tasks/compute_resource.py`
Ce fichier expose la fonction **`create_compute_resource()`**, qui est appelée en tâche asynchrone via **TaskIQ**.

Elle orchestre l’ensemble du processus :
- récupération des paramètres depuis la base (flavor, image, userdata),
- mise à jour de l’état dans la BDD,
- appel à `create_vm()`.

---

###  **Problème identifié 1** : `openstacksdk` est **synchrone**  
L’appeler directement dans une fonction async **bloque la boucle d’événement** (risque de ralentir tout le backend).

#### **Solution implémentée** :  
Utilisation de `run_in_executor()` pour déporter l’appel dans un **thread dédié**, permettant au backend de rester **réactif et non bloquant** :

```python
loop = asyncio.get_running_loop()
server_id = await loop.run_in_executor(
    None,
    create_vm,
    name,
    image_name,
    flavor_name,
    network_name,
    userdata,
)
```
---

### Problème identifié 2 : cloud.yaml en dur dans openstack_backend.py

Le fichier `cloud.yaml` était attendu manuellement pour établir une connexion à OpenStack, ce qui rendait la configuration dépendante du poste ou du conteneur utilisé.

#### Solution implémentée :
Utilisation directe du fichier `clouds.yaml` via :
```python
conn = connect(cloud="openstack")
```
où `"openstack"` est le nom de la configuration dans le fichier `~/.config/openstack/clouds.yaml`.

#### Mise en place dans le conteneur :
```bash
sudo mkdir -p /home/vscode/.config/openstack
sudo chown -R vscode:vscode /home/vscode/.config
nano /home/vscode/.config/openstack/clouds.yaml
```
→ Coller le contenu du fichier téléchargé via Horizon : *Accès API > Télécharger clouds.yaml*

---

### Problème identifié 3 : Network codé en dur dans tasks/compute_resource.py

Le nom du réseau était défini statiquement (`network = "private"`) dans la fonction de création de VM, empêchant toute flexibilité ou configuration dynamique.

#### Solution implémentée :

- Ajout du champ `network_name` dans le modèle `ComputeResourceCreate` (`schemas/compute_resource.py`) :
```python
network_name: str = Field(
    description="The name of the network this resource should be connected to.",
    examples=["provider-lab", "private-net"],
)
```

- Transmission dynamique dans `api/compute_resource.py` :
```python
request_config = {
    **shared_request_config,
    "userdata": new_resource.userdata,
    "network_name": new_resource.network_name,
    "network_interfaces": [rni.model_dump_json() for rni in req_network_interfaces],
}
```

- Récupération du paramètre dans `tasks/compute_resource.py` :
```python
network_name = request_cfg.get("network_name")
if not network_name:
    raise ValueError("Missing 'network_name' in request_config for ComputeResource.")
```

- Passage du paramètre à `create_vm()` dans `openstack_backend.py`.

#### Validation :
- Ajout de `"network_name": "xxxxx"` dans le script `request-bi-resources.sh`
- Résultat : VM créée dans le bon réseau, visible sur Horizon

#### Logs de succès :
```
[OpenStack] network 'xxxxx' found
[OpenStack] VM 'xxxxx' created successfully
```

---

#### Tester la création d’une VM

Utiliser le script fourni request-resources.sh :  
```bash
./example/request-resources.sh <exp_name>
```

Les paramètres attendus à l'intérieur :
- `VM_NAME` : Nom de la machine à créer
- `DISK_IMAGE_ID` : ID d’image OpenStack
- `FLAVOR_ID` : ID du flavor OpenStack
- `NETWORK_NAME` : Nom du réseau
- `EXPIRATION_DATE` : Date d'experiation de l'experience
- `SSH_PUBLIC_KEY` : Clé SSH
- `USERDATA_SCRIPT` : Fichier de configuration cloud-init

---

### 9.2 Suppression d'une ressource (VM)

Comme précédemment, la logique est répartie de manière claire dans deux fichiers distincts :


#### `infrastructure/openstack_backend.py`

La suppression est assurée par la fonction delete_vm.

#### `tasks/compute_resource.py`

La fonction **`delete_compute_resource()`**, qui est appelée en tâche asynchrone via **TaskIQ**.
Comme pour la création, on utilise `run_in_executor()` pour déporter l’appel dans un **thread dédié**, permettant au backend de rester **réactif et non bloquant** :

#### Tester la suppression d’une VM
Via le SWAGUER UI dans la section DELETE RESOURCES.
Résultats :
- Code HTTP `200`
- La ressource n'est plus visible via un GET sur `/resources`
- La tâche est traçable via un GET sur `/tasks/{task_id}`





═══════════════════════════════════════════════════════════════════════════════════
