# BI BACKEND

---

## ÉTAPE 1 — Installations nécessaires

### Docker & dépendances système

```bash
sudo apt update
```
```bash
git clone git@gitlab.ilabt.imec.be:slices-public/slices-bi-blueprint.git
```

---

## Ouverture du projet dans VSCode

- Ouvrir le dépôt dans VSCode
- Lancer avec : `Reopen in Container` (Dev Container)
- Attendre le chargement complet

---

## Installation des dépendances Python

Dans le terminal du container VSCode :

```bash
pip install --upgrade pip
```
```bash
pip install -e .[dev,test]
```

---

## ÉTAPE 2 — Vérification du backend FastAPI

### Lancement du serveur FastAPI

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

### Vérification logs VSCode :

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


## ÉTAPE 6 — 








## ÉTAPE  — Test de création de ressource

Toujours dans /examples :

```bash
./request-resources.sh exp1
```

Résultat :

- Token récupéré
- Expériment créé
- Appels GET/POST sur /resources/ renvoient 404

Cause : les endpoints REST de gestion des ressources ne sont pas encore implémentés.

---

