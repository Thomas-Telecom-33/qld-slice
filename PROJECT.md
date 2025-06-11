# Projet – Blueprint for Slices BI Backend for Openstack

## Titre : **SLICES-RI compliant interface for a QKD BluePrint**

SLICES-RI (https://www.slices-ri.eu) compliant interface for a QKD BluePrint.

We would like to add a software interface compliant with the SLICES-RI (Scientific Large Scale Infrastructure for Computing/Communication Experimental Studies) to a QKD BluePrint experiment we have.  Again this involves creating code that interfaces with users through a webpage, and interact with SLICES and with the experiment.

---

## 🎯 Objectif général

- Développer un backend conforme aux standards SLICES.
- Permettre la création et la gestion de ressources expérimentales via l’API.
- Connecter l’API à une infrastructure OpenStack puis Kubernetes.
- Intégrer les outils SLICES (CLI, Auth, Gateway).
- Tester et déployer la solution dans un environnement simulé, puis réel.

---

## 📘 PHASE 0 — Documentation & Compréhension

### 🎯 Objectifs
- Comprendre le fonctionnement de SLICES (CLI, Auth, API, JWT).
- Étudier l’architecture de `slices-bi-blueprint`.
- Explorer la documentation officielle SLICES.

### ✅ Avancement
- Documentation parcourue.
- Authentification et structure des JWT comprises.
- Structure backend analysée (modèles, routes, tâches).

### 📦 Livrables
- Cahier de charges interne.
- Mapping d’architecture.
- Première visualisation Swagger/OpenAPI.

---

## 🗃️ PHASE 1 — Base de Données & Backend Local

### 🎯 Objectifs
- Mettre en place la base PostgreSQL locale.
- Vérifier les interactions entre API et base.
- Adapter la base à l'infrastructure Openstack.
- Utiliser OpenStack SDK pour récupérer images/flavors.
- Poster ces infos dans l’API `disk_images` et `flavors`.

### ✅ Avancement
- Base opérationnelle avec Alembic.
- Création de ressources via POST vérifiée.
- Persistance des tâches validée.
- Base de données Openstack intégrée.
- Script d’import des images/flavors.
- Base alimentée avec les vraies valeurs OpenStack.

### 📦 Livrables
- Base PostgreSQL + schéma Alembic.
- Modèles Python validés.
- Base testée.

---

## 🧪 PHASE 2 — Tests, API et Script d’Exemple

### 🎯 Objectifs
- Tester tous les endpoints (GET/POST images, flavors, resources, tasks...).
- Comprendre le flux complet (JWT, création, POST...).
- Automatiser via script `request-resources.sh`.

### ✅ Avancement
- Endpoints testés avec Swagger.
- Script corrigé, automatisé et fonctionnel.
- Génération automatique des IDs, JWT, etc.
- Validation JSON complète.

### 📦 Livrables
- Script de test 100% fonctionnel.
- Flux d’authentification et d’utilisation validé.
- JWT et ressources manipulés dynamiquement.

---

## ☁️ PHASE 3 — Intégration OpenStack

### 🎯 Objectifs
- Implémenter la logique de création de VM dans `create_compute_resource`.
- Implémenter la logique de suppresion de VM dans `delete_compute_resource`.
- Test via Swagger UI.
- Test via request-resources.sh

### 📦 Livrables
- Création réelle de VMs.
- Suppresion réelle de Vms.
- Infrastructure Openstack connectée et conforme aux attentes du projet.

A cette étape là, on doit être capable de créer/supprimer des Vms Openstack compatible avec SLICES (comme le demande le projet).

---

## 🐳 PHASE 4 — Déploiement Kubernetes

## 🌐 PHASE 5 — Interface Frontend (UI)

## 🔒 PHASE 6 — Sécurité, Permissions et Logs

## 🚀 PHASE 7 — Déploiement & Documentation finale

**Ce document retrace les grandes étapes de conception, tests et déploiement d’un backend SLICES-ready, avec intégration progressive dans une infra cloud.**
