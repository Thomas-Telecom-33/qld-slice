# Template de base :

docker ps

docker exec -it slices-bi-blueprint_devcontainer-db-1 psql -U slices_bi -d slices_bi

ubuntu@slices-qkd:~$ docker exec -it slices-bi-blueprint_devcontainer-db-1 psql -U slices_bi -d slices_bi
psql (17.5 (Debian 17.5-1.pgdg120+1))
Type "help" for help.

slices_bi=# \dt
               List of relations
 Schema |       Name        | Type  |   Owner
--------+-------------------+-------+-----------
 public | alembic_version   | table | slices_bi
 public | compute_resources | table | slices_bi
 public | disk_images       | table | slices_bi
 public | experiments       | table | slices_bi
 public | flavors           | table | slices_bi
 public | projects          | table | slices_bi
 public | users             | table | slices_bi
(7 rows)

slices_bi=# \du
                             List of roles
 Role name |                         Attributes
-----------+------------------------------------------------------------
 slices_bi | Superuser, Create role, Create DB, Replication, Bypass RLS

Pour obtenir plus de détails :

slices_bi=# \d alembic_version par exemple

# Adaptation à notre infrastructure 

Voici un script permettant de lister les images et flavors disponibles dans notre infrastructure Openstack :

get_openstack_resources.py

```bash
import openstack

def main():
    conn = openstack.connect()

    print("IMAGES")
    for image in conn.image.images():
        print(f"\n ID: {image.id}")
        print(f"Name: {image.name}")
        print(f"Created At: {image.created_at}")
        print(f"Status: {image.status}")
        print(f"Visibility: {image.visibility}")
        print(f"Tags: {image.tags}")

    print("\n FLAVORS")
    for flavor in conn.compute.flavors():
        print(f"\n ID: {flavor.id}")
        print(f"Name: {flavor.name}")
        print(f"vCPUs: {flavor.vcpus}")
        print(f"RAM (MB): {flavor.ram}")
        print(f"Disk (GB): {flavor.disk}")
        print(f"Ephemeral (GB): {flavor.ephemeral}")
        print(f"Swap (MB): {flavor.swap}")
        print(f"RXTX Factor: {flavor.rxtx_factor}")
        print(f"Is Public: {flavor.is_public}")

if __name__ == "__main__":
    main()
````
On alors la liste des images et flavors avec leurs détails (tous ne sont pas nécessaires).






