
# SLICES BI Blueprint adaptation with Openstack

---

## ÉTAPE 1 — Installations nécessaires

### 1.1 Docker & dépendances système

```bash
sudo apt update
```

Cloner le dépôt principal :

```bash
git clone git@gitlab.ilabt.imec.be:slices-public/slices-bi-blueprint.git
```

---

### 1.2 Ouverture du projet dans VSCode

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

Fichier : `/examples/request-resources.sh`

Lancement :
```bash
./request-resources.sh exp1
```

Le script automatise :
- Authentification
- Création d’expérience
- Génération de tokens
- Création de ressource avec POST

### Résultat attendu :

Code `200`, ressource visible via GET, tâche visible via GET /tasks.

---

## ÉTAPE 8 — Implémentation de `create_compute_resource`

Dans le fichier du template (à compléter) :

Rechercher :
```python
# TODO: implement your create resource logic here
```

À compléter pour lancer une vraie VM sur OpenStack, à partir des données reçues (image, flavor, username...).
