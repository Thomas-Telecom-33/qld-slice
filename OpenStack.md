# OpenStack VM Configuration Guide

---

## 1. Éléments essentiels pour créer une VM OpenStack

Pour créer une instance OpenStack, les paramètres suivants sont nécessaires :

- Nom de la machine
- Type de machine (Flavor)
- Réseau
- Clé SSH
- Fichier cloud-init (personnalisation au démarrage)

---

## 2. Création d'une VM avec un script Python

### 2.1 Configuration de l'accès via le dashboard

Dans le tableau de bord OpenStack :  
**Projet > Accès API > Télécharger le fichier RC d'OpenStack**

Cela permet d'obtenir un fichier `clouds.yaml` contenant la configuration nécessaire pour se connecter à OpenStack avec la bibliothèque `openstacksdk`.

Il faut ensuite y ajouter manuellement la section `password`.

**Exemple simplifié de fichier `clouds.yaml`** :
```yaml
clouds:
  openstack:
    auth:
      auth_url: 
      username: your_username
      project_name: your_project
      user_domain_name: Default
      project_domain_name: Default
    region_name: RegionOne
    interface: public
    password: your_password
```

### 2.2 Ajout du fichier `clouds.yaml` dans la VM

Créer le dossier de configuration et y placer le fichier :
```bash
mkdir -p ~/.config/openstack
nano ~/.config/openstack/clouds.yaml
```

Puis coller le contenu du fichier `clouds.yaml`.

Installer la bibliothèque Python nécessaire :
```bash
pip install openstacksdk
```

---

### 2.3 Script Python de test : `test_create_vm.py`

Ce script permet de créer une VM OpenStack en utilisant les paramètres définis, il s'agit d'un script de test permettant de se familiariser avec la bilbliothèque openstacksdk :
```bash
import openstack
import base64

conn = openstack.connect(cloud='openstack')

IMAGE_NAME = ""
FLAVOR_NAME = ""
NETWORK_NAME = ""
KEY_NAME = ""
VM_NAME = "VM_test"

USERDATA = """#cloud-config
...
package_update: true
package_upgrade: true
"""

print("[INFO] Looking for components")

image = conn.compute.find_image(IMAGE_NAME)
flavor = conn.compute.find_flavor(FLAVOR_NAME)
network = conn.network.find_network(NETWORK_NAME)
keypair = conn.compute.find_keypair(KEY_NAME)

if not all([image, flavor, network, keypair]):
    raise Exception("[ERROR] Components are not found")

print("[INFO] Components found")

print("[INFO] VM is launching")

server = conn.compute.create_server(name=VM_NAME,
    image_id=image.id,
    flavor_id=flavor.id,
    networks=[{"uuid": network.id}],
    key_name=keypair.name,
    user_data=base64.b64encode(USERDATA.encode("utf-8")).decode("utf-8"),
)

print("[INFO] Waiting...")
server = conn.compute.wait_for_server(server)

print(f"[INFO] VM {server.name} is ready (status={server.status})")

for net_name, addresses in server.addresses.items():
    for addr in addresses:
        print(f"{net_name} → {addr['addr']} ({addr['OS-EXT-IPS:type']})")
```
---

### 2.4 Résultats attendus

- **Terminal** : Affichage de l’état de la VM et de son adresse IP. Si tout se passe bien, un accès SSH est possible.
- **OpenStack Dashboard** : La VM apparaît dans la liste des instances.

---


