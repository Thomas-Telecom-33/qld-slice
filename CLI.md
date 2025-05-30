# 💻 Mise en place de l'infrastructure SLICES-RI pour le projet QKD

---

## 1. Gestion des clés SSH

### Vérification de la clé existante

```bash
ls -l ~/.ssh/id_rsa
ls -l ~/.ssh/id_rsa.pub
```

### Enregistrement de la clé publique (si pas encore fait)

```bash
slices pubkey register ~/.ssh/id_rsa.pub
```
---

## 2. Création et sélection du projet

Le projet est créé via le portail web SLICES :  
https://portal.slices-ri.eu → *"Request a new project"*

```bash
slices project list-projects
slices project use qkd
```

---

## 3. Création de la slice

```bash
slices experiment create qkd-slice
slices experiment show qkd-slice
```

---

## 4. Déploiement d'une VM simple

### Définir le site cible

```bash
export SLICES_BI_SITE_ID=be-gent1-bi-vm1
```

### Lister les images disponibles

```bash
slices bi diskimage list
```

### Créer une VM `vm-test`

```bash
slices bi create vm-test --image "Ubuntu 22.04.5" --flavor m1.medium --duration 1d --experiment qkd-slice
```

### Vérifier les ressources

```bash
slices bi list-resources
```

---

## 5. Connexion SSH à la VM

```bash
slices bi ssh --experiment qkd-slice vm-test
```

---

## 6. Déploiement avec Cloud-Init

### Fichier `cloudinit.yaml`

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - git
  - python3
  - python3-pip
  - ...
  - ...
```
---

## 7. Fichier de configuration JSON

### `vm-cloudinit.json`

```json
{
  "resources": [
    {
      "friendly_name": "vm-cloudinit-test",
      "site_id": "be-gent1-bi-vm1",
      "flavor": "m1.small",
      "disk_image": "Ubuntu 22.04.5",
      "cloudinit_file": "cloudinit.yaml"
    }
  ]
}
```

---

## 8. Création de la VM avec cloud-init

```bash
slices bi create-from-file vm-cloudinit.json --experiment qkd-slice
```

---

## 9. Suppression d'une VM

```bash
slices bi destroy interface-vm --experiment qkd-slice
slices bi destroy vm-cloudinit-test --experiment qkd-slice
```

---

## Résultat

La VM est automatiquement provisionnée avec les paquets définis dans `cloud-init.yaml`, et connectable via SSH.

---
