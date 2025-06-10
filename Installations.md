# SLICES BI Blueprint adaptation with Openstack 

---

## ÉTAPE 1 — Installations nécessaires

### 1.1 Docker & dépendances système

```bash
sudo apt update
```
```bash
git clone git@gitlab.ilabt.imec.be:slices-public/slices-bi-blueprint.git
```

---

### 1.2 Ouverture du projet dans VSCode

- Ouvrir le dépôt dans VSCode
- Lancer avec : `Reopen in Container` (Dev Container)
- Attendre le chargement complet

---

### 1.3 Installation des dépendances Python

Dans le terminal du container VSCode :

```bash
pip install --upgrade pip
```
```bash
pip install -e .[dev,test]
```

---

## ÉTAPE 2 — Vérification du backend FastAPI

### 2.1 Lancement du serveur FastAPI

Initialisé d'abord une première fois avec :
```bash
alembic upgrade head
```

Lancé automatiquement dans le container avec :
```bash
python -m uvicorn slices_bi_blueprint_backend.app:app --host 0.0.0.0 --port 8000 --reload
```

Puis il est possible d'accéder à :

- http://localhost:8000/docs
- http://localhost:8000/redoc

### 2.2 Vérification logs VSCode :

```bash
INFO:     127.0.0.1:34672 - "GET /redoc HTTP/1.1" 200 OK
INFO:     127.0.0.1:53914 - "GET /docs HTTP/1.1" 200 OK
```

Le backend tourne correctement.
---

## ÉTAPE 3 — Authentification via Slices CLI

Dans un nouveau terminal dans VSCode (container) :
Installation
```bash
pip install slices-cli --extra-index-url=https://doc.slices-ri.eu/pypi/
```
Authentification
```bash
slices auth login
```
Pour aides :
```bash
slices --help
```
Lister les projets
```bash
slices project list
```
Créer un projet nommé par exemple "qkd"
```bash
slices project use qkd
```
Si on veut savoir dans quel projet nous sommes :
```bash
slices project show
```
On peut aussi voir les experiences
```bash
slices experiment list
```
En créer une
```bash
slices experiment create qkd-experiment
```
---

## ÉTAPE 4 — Obtention des tokens

Il existe 2 tokens :

- Token user :
```bash
slices auth get-for-audience
```
Permet de faire des requêtes GET

- Token d'experience :
```bash
slices experiment jwt qkd-experiment
```
Permet de faire des GET & POST

On va donc rajouter son compte dans les ADMIN_USER_IDS du fichier .env du container, permettant ainsi d'avoir des droits nécessaires pour faire l'ensemble des requêtes.

Ce userID est récupérable en faisant :
```bash
slices project show
```
Sous .env, on rajoute alors notre UserID : 
```bash
ADMIN_USER_IDS=[ user_account.ilabt.imec.be_ ...]
```
---
## ÉTAPE 5 — Test des endpoints via SWAGUER

### 5.1 GET IMAGES/FLAVOR
Dans le swaguer http://localhost:8000/docs#, on peut alors tester les endpoint utilisés.

Dans "Authorize", on colle alors le token généré (soit le token user soit le token d'experience), par exemple:
"ey.........fbW"

Puis on clique sur Authorize et close.

On peut alors lancer notre première reqûete GET :
Images > Try it out > Excecute
La sortie attendue est alors 200.

### 5.2 POST IMAGES/FLAVOR
Maintenant, on va ajouter des images/flavor, avec le token d'experience (même procédure qu'en haut), le format du request body est déjà prérempli, on adapte si on veut puis Excecute.

Par exemple :
```bash
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
On doit aller obtenir un code 200. L'image est bien crée.


## 5.3 GET/POST resources
On peut récuperer toutes les infos demandés liés au projet et experience avec :
```bash
slices experiment show qkd-experiment --format json
```
On complète ainsi les champs project et experiment_id puis EXECUTE.

On doit aller obtenir un code 200.

On remplit les mêmes champs dans la requête POST, le body est déjà donné.
Concernant les JWT :
> Dans Authorize, toujours le token d'experience (Attention, il expire rapidement, environ 15 min)
```bash
slices experiment jwt qkd-experiment
```
> Dans la requete body, le token utilisateur
```bash
slices auth id-token https://slices-bi-blueprint.ilabt.imec.be
```
On doit aller obtenir un code 200. La resource est bien crée.

---
## ÉTAPE 6 — Adaptation de la base de donnée

Maintenant que l'on sait manipuler les ENDPOINTS, on va alors pouvoir adapter notre base de donnée à notre infrastructure.
Pour cela, il est nécessaire de récuperer l'ensemble des images et flavors de notre infra.

On peut alors faire un script de récupération comme celui-ci :
```bash
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

On a alors la liste des images et flavors que l'on va rajouter dans notre base de donnée via POST images et POST resources comme expliqué précédemment.

Exemples :

> POST Images :
```bash
{
  "friendly_name": "Ubuntu Server 20.04",
  "cluster_id": "default",
  "location": "ubuntu-server-20.04",
  "default_username": "ubuntu",
  "min_disk_mb": 8000,
  "min_ram_mib": 512,
  "short_description": "OpenStack",
  "long_description": "OpenStack image ID: 707d9055-76d7-4f3f-a2ee-9d7983f3baac",
  "tags": [],
  "hidden": false
}
```

> POST FLAVORS :
```bash

```


On peut vérifier tous ces ajouts via GET images et GET flavors.
On peut aussi vérifier cela à partir de notre VM :
```bash
docker exec -it slices-bi-blueprint_devcontainer-db-1 psql -U slices_bi -d slices_bi
\dt
SELECT * FROM disk_images;
SELECT * FROM flavors;
```
---
## ÉTAPE 7  — Test de création de ressource

Dans /examples, deux fichiers d'exemples sont présents :

On peut lancer le script de cette manière :
```bash
./request-resources.sh exp1
```

Résultat :
- Token récupéré
- Expériment créé
- Appels GET/POST sur /resources/ renvoient 404

Cause : les endpoints REST de gestion des ressources ne sont pas encore implémentés.

---

## ÉTAPE 8  — Fonction create_compute_resource :
La première fonction à compléter dans la template est la fonction de création d'une resource.
Nous devons ainsi compléter la partie "# TODO: implement your create resource logic here" pour être en accord avec Openstack.




