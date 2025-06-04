# BI BACKEND

---

## ÉTAPE 1 — Installations nécessaires

### Docker & dépendances système

```bash
sudo apt update
```
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
```bash
sudo usermod -aG docker $USER
```
```bash
newgrp docker
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

Dans un nouveau terminal dans VSCode (container) sans /examples :

```bash
pip install slices-cli --extra-index-url=https://doc.slices-ri.eu/pypi/
```
```bash

slices auth login
```
```bash

slices project list
```
```bash

slices project use qkd
```

---

## ÉTAPE 4 — Test de création de ressource

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

