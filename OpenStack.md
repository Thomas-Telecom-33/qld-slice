# OpenStack configuration

---

## 1. Fondamentals to create an OpenStack VM
- Name
- Flavor
- Network
- SSH key
- Cloud-init

---

## 2. Test avec un script de base

### 2.1 Openstack dashboard
Dans le dashboard Openstack > Accès API > Télécharger le fichier RC D'openstack
On a alors maintenant clouds.yaml qui sert à .... 
Exemple de fichier clouds.yaml :



On a alors juste à rajouter la section mot de passe et son mot de passe.

## 2.2 Ajout du fichier clouds.yaml dans la VM 
mkdir -p ~/.config/openstack
nano ~/.config/openstack/clouds.yaml
Puis coller le contenu

On installe également la bilbliotheque nécessaire
pip install openstacksdk

## 2.3 Script test test_create_vm.py 
On créer alors un script test avec les informations que nous connaissons concernant l'infrastructure que nous possédons.

import openstack
import base64

conn = openstack.connect(cloud='openstack')

IMAGE_NAME = ""
FLAVOR_NAME = ""
NETWORK_NAME = ""
KEY_NAME = ""
VM_NAME = ""

USERDATA = """#cloud-config
...
package_update: true
package_upgrade: true
...
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


