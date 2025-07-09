# Projet – Blueprint for Slices BI Backend for Openstack

## Titre : **SLICES-RI compliant interface for a QKD BluePrint**

SLICES-RI (https://www.slices-ri.eu) compliant interface for a QKD BluePrint.

We would like to add a software interface compliant with the SLICES-RI (Scientific Large Scale Infrastructure for Computing/Communication Experimental Studies) to a QKD BluePrint experiment we have.  Again this involves creating code that interfaces with users through a webpage, and interact with SLICES and with the experiment.

═══════════════════════════════════════════════════════════════════════════════════

## 🎯 Objectif général

- Développer un backend conforme aux standards SLICES.
- Permettre la création et la gestion de ressources expérimentales via l’API.
- Connecter l’API à une infrastructure OpenStack puis Kubernetes.
- Fournir une interface utilisateur permettant de faire cela

═══════════════════════════════════════════════════════════════════════════════════

## 📘 PHASE 0 — Documentation & Compréhension

### 🎯 Objectifs
- Comprendre le fonctionnement de SLICES (CLI, Auth, API, JWT).
- Étudier l’architecture de `slices-bi-blueprint`.
- Explorer la documentation officielle SLICES.
---
### ✅ Avancement
- Documentation parcourue.
- Authentification et structure des JWT comprises.
- Structure backend analysée (modèles, routes, tâches).
---
### 📦 Livrables
- Cahier de charges interne.
- Mapping d’architecture.
- Première visualisation Swagger/OpenAPI.

═══════════════════════════════════════════════════════════════════════════════════

## 🗃️ PHASE 1 — Base de Données & Backend Local

### 🎯 Objectifs
- Mettre en place la base PostgreSQL locale.
- Vérifier les interactions entre API et base.
- Adapter la base à l'infrastructure Openstack.
- Utiliser OpenStack SDK pour récupérer images/flavors.
- Poster ces infos dans l’API `disk_images` et `flavors`.
---
### ✅ Avancement
- Base opérationnelle avec Alembic.
- Base de données Openstack intégrée.
- Script d’import des images/flavors.
---
### 📦 Livrables
- Base PostgreSQL + schéma Alembic.
- Modèles Python validés.
- Base testée.

═══════════════════════════════════════════════════════════════════════════════════

## 🧪 PHASE 2 — Tests, API et Script d’Exemple

### 🎯 Objectifs
- Tester tous les endpoints (GET/POST images, flavors, resources, tasks...).
- Comprendre le flux complet (JWT, création, POST...).
- Automatiser via script `request-resources.sh`.
---
### ✅ Avancement
- Endpoints testés avec Swagger.
- Script corrigé, automatisé et fonctionnel.
- Génération automatique des IDs, JWT, etc.
- Validation JSON complète.
---
### 📦 Livrables
- Script de test 100% fonctionnel.
- Flux d’authentification et d’utilisation validé.
- JWT et ressources manipulés dynamiquement.

═══════════════════════════════════════════════════════════════════════════════════
## ☁️ PHASE 3 — Intégration OpenStack

### 🎯 Objectifs
- Implémenter la logique de création de VM dans `create_compute_resource`.
- Implémenter la logique de suppresion de VM dans `delete_compute_resource`.
- Test via Swagger UI.
- Test via request-resources.sh
- Régler les problèmes rencontrés + améliorer des choses existantes.
---
### ✅ Avancement
- Création de VM openstack fonctionnelle : la VM apparait désormais sur Horizon
- Création non bloquante
- Création via fichier clouds.yaml
- Gestion de plusieurs network
- Ajout des installations automatiques de openstacksdk et slices via dockerfile
- Correction AssertionError: Please, `connect()` the broker first
- Suppression de VM openstack fonctionnelle : la VM est supprimée sur Horizon
---
### 📦 Livrables
- Création réelle de VMs.
- Suppresion réelle de Vms.
- Infrastructure Openstack connectée et conforme aux attentes du projet.
- DEVS_OPENSTACK.md

═══════════════════════════════════════════════════════════════════════════════════
## 🐳 PHASE 4 — Déploiement Kubernetes

### ✅ Avancement

Étape 1 : Installation & configuration du cluster Kubernetes

Étape 2 : Créer le backend kubernetes_backend.py

Étape 3 : Étendre tasks/compute_resource.py

Étape 4 : Vérifications côté modèles & schémas

Étape 5 : Tests et validation de bout en bout

### 📦 Livrables
- Création réelle de pods/containers.
- Suppresion réelle de pods/containers.
- DEVS_KUBERNETES.md

═══════════════════════════════════════════════════════════════════════════════════

## 📊 PHASE 5 — Intégration du MRS (Metadata Registry System)

### 🎯 Objectifs
- Comprendre le fonctionnement du MRS dans l'écosystème SLICES-RI.
- Ajouter au backend un module permettant de publier des métadonnées dans le MRS après chaque création de ressource (VM/pod).
- Automatiser l’enregistrement de ces métadonnées depuis les tâches existantes (compute_resource.py).
- Garantir la conformité au format d’objet attendu par le MRS (base/service/dataset...).
- Sécuriser les appels MRS avec un token (JWT).

### ✅ Avancement

Étape 1 — Analyse technique & installation et configurations locales : OK

Étape 2 — Installation et configurations sur la VM, cohabitation avec le backend existant : TODO

Étape 3 — Préparation du client MRS : TODO

Étape 4 — Génération du payload : TODO

Étape 5 — Intégration dans compute_resource.py : TODO

Étape 6 — Test et validation : TODO

### 📦 Livrables


═══════════════════════════════════════════════════════════════════════════════════

## 🌐 PHASE 6 — Interface Frontend (UI)
x²
Front-end avec spécifications à venir.

TODO :
> clouds.yaml :
  - Chaque utilisateur doit téléverser son clouds.yaml une seule fois via le frontend.
  - Ce fichier est stocké côté backend, lié à son compte (base de données ou fichier privé).
  - Lors d'une action (ex: création de VM), le backend utilise automatiquement ce fichier.
    
═══════════════════════════════════════════════════════════════════════════════════
## 🔒 PHASE 6 — Sécurité, Permissions et Logs
═══════════════════════════════════════════════════════════════════════════════════
## 🚀 PHASE 7 — Déploiement & Documentation finale

- Script de lancement global backend/frontend
- USER.md

  
═══════════════════════════════════════════════════════════════════════════════════
