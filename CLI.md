# Mise en place de l'infrastructure

---
## Création du projet
Sur slice directement, dans "Request a new project".
Le prpject actuel se nomme "qkd".

`slices project list-projects`

`slices project use qkd`

---
## Création d'une slice

`slices experiment create qkd-slice`

`slices experiment show qkd-slice`

## Création d'une VM

`export SLICES_BI_SITE_ID=be-gent1-bi-vm1`

`slices bi diskimage list`

`slices bi create interface-vm --image "Ubuntu 22.04.5" --flavor m1.medium --duration 1d --experiment qkd-slice`

`slices bi list-resources`

---
## Connexion

`slices bi ssh --experiment qkd-slice interface-vm`




















