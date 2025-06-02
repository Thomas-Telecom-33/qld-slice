# PROJECT
Automatisation via WEB
---
Interface web qui :
- Génère dynamiquement des fichiers .yaml (Cloud-init) pour configurer la VM au boot
- Génère dynamiquement des fichiers .json pour décrire les ressources SLICES
- Lance des commandes SLICES-CLI automatiquement

>Fichier yaml : sert à installer automatiquement les packages nécessaires au boot
Peut-etre faire un sudo apt update au boot aussi
=> Voir quels outils sont nécessaires à installer pour la config initiale

>Fichier json : sera dynamique
Champs à compléter pour une configuration manuelle et lancement auto
d'instances :
Un formulaire simple côté frontend permettrait de choisir :
Nom de la VM : my-vm-test
Flavor : m1.medium/small/large/Xtralarge
Image : Ubuntu 22.04.5
Durée : 1d...
=> Voir si d'autres params sont nécessaires

Et ensuite, via un bouton par exmeple "Launch instance" on lancerait alors la commande CLI :
slices bi create-from-file myconfig.json --experiment qkd-slice
=> Voir autres fonctionnalités : 
Status
INFOS/Details
Destroy
ect

---
