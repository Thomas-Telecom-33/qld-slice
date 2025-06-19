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

═══════════════════════════════════════════════════════════════════════════════════
## 🐳 PHASE 4 — Déploiement Kubernetes

⚙️ Étape 1 : Installation & configuration du cluster Kubernetes dans la VM

 1.1 Choisir l’outil adapté (recommandé : Minikube ou K3s)
 
 1.2 Installer Kubernetes dans la VM
 
 1.3 Vérifier accès avec kubectl get nodes
 
 1.4 S’assurer que le fichier ~/.kube/config est utilisable par le conteneur VSCode
 
 1.5 Tester dans VSCode : kubectl get pods fonctionne dans le terminal du conteneur

---

🧠 Étape 2 : Créer le backend kubernetes_backend.py

 2.1 Créer un fichier src/slices_bi_blueprint_backend/infrastructure/kubernetes_backend.py
 
 2.2 Y définir :
connect_k8s() : initialise le client Kubernetes Python (config.load_kube_config())
create_pod(name, image, command, env) → retourne le nom du pod
delete_pod(name)

 2.3 Implémenter wait_for_pod_ready(name) si nécessaire

---

🧩 Étape 3 : Étendre tasks/compute_resource.py

 3.1 Dans create_compute_resource, ajouter :
Dispatch : if flavor.flavor_type == "k8s": ...

 3.2 Appeler create_pod() à la place de create_vm()

 3.3 Mettre model_vm.provider_resource_id = pod_name

 3.4 Adapter la partie delete_compute_resource pour appeler delete_pod() si flavor_type == "k8s"

---

🧱 Étape 4 : Vérifications côté modèles & schémas

 4.1 Assurer que :

Les flavors pour Kubernetes ont "flavor_type": "k8s"
request_config contient "image" et autres infos utiles

 4.2 Aucun changement requis dans les modèles SQL ni endpoints REST — parfait

---

🧪 Étape 5 : Tests et validation de bout en bout

 5.1 Créer un flavor K8s et une image de test

 5.2 Utiliser le script de test modifié avec flavor_type "k8s"

 5.3 Vérifier :
Création de pod visible dans kubectl
Retour 200 de l’API
Logs du backend corrects

 5.4 Supprimer la ressource → pod détruit ?

---

🧹 Étape 6 : Robustesse et finalisation

 6.1 Gérer les erreurs dans create_pod() (CrashLoop, Pending…)

 6.2 Ajouter des logs explicites dans le backend

 6.3 Rendre le code générique pour d'autres types de containers

 6.4 (Optionnel) Permettre de passer des volumes, env, ou resources via request_config

---

🗂️ Étape 7 : Documentation

 7.1 Mettre à jour le fichier PROJECT.md

 7.2 Mettre à jour DEVS_k8s.md









═══════════════════════════════════════════════════════════════════════════════════
## 🌐 PHASE 5 — Interface Frontend (UI)
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
