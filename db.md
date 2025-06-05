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

On nettoie :
DELETE FROM compute_resources;
DELETE FROM disk_images;
DELETE FROM flavors;

On ajoute les images :
INSERT INTO disk_images (
    id, friendly_name, created_at, cluster_id, location,
    default_username, min_disk_mb, min_ram_mib, tags,
    short_description, long_description, hidden
)
VALUES
-- Debian 11
('da20995c-642f-4f6b-abce-64b80716b5f4', 'debian-11', '2023-07-11T10:49:11Z', 'default',
 'da20995c-642f-4f6b-abce-64b80716b5f4', 'debian', 1000, 512, ARRAY[]::varchar[],
 'Debian 11 image', NULL, false),

-- Ubuntu Server 20.04
('707d9055-76d7-4f3f-a2ee-9d7983f3baac', 'ubuntu-server-20.04', '2022-05-25T08:26:39Z', 'default',
 '707d9055-76d7-4f3f-a2ee-9d7983f3baac', 'ubuntu', 1000, 512, ARRAY[]::varchar[],
 'Ubuntu Server 20.04 LTS', NULL, false),

-- GRAL_IkerH
('acfd2bb8-8722-4949-943f-1ad2003988fb', 'GRAL_IkerH', '2022-02-08T11:20:09Z', 'default',
 'acfd2bb8-8722-4949-943f-1ad2003988fb', 'ubuntu', 1000, 512, ARRAY[]::varchar[],
 'GRAL IkerH image', NULL, false),

-- Cirros
('98192d7e-4b38-4564-9d74-fd9b4a80175c', 'cirros-0.5.2', '2021-05-25T13:14:43Z', 'default',
 '98192d7e-4b38-4564-9d74-fd9b4a80175c', 'cirros', 100, 128, ARRAY[]::varchar[],
 'Cirros test image', NULL, false);


On ajoute les flavors :
INSERT INTO flavors (id, friendly_name, description, flavor, created_at)
VALUES
-- flavor-5
('129c0d54-4857-4670-8914-99a99731a0d6', 'flavor-5', '2 vCPU, 4 GB RAM, 16 GB Disk',
 '{"vcpu": 2, "ram_mib": 4096, "disk_gib": 16, "flavor_type": "vm"}', NOW()),

-- flavor-2
('1b095739-f135-49a3-9e20-9069eb3c723c', 'flavor-2', '1 vCPU, 512 MB RAM, 2 GB Disk',
 '{"vcpu": 1, "ram_mib": 512, "disk_gib": 2, "flavor_type": "vm"}', NOW()),

-- flavor-6
('1eab3df4-cd30-4738-9908-e2a43e3e4e33', 'flavor-6', '4 vCPU, 8 GB RAM, 32 GB Disk',
 '{"vcpu": 4, "ram_mib": 8192, "disk_gib": 32, "flavor_type": "vm"}', NOW()),

-- flavor-4
('352a6a88-4441-475b-a351-fe1decae8a5d', 'flavor-4', '2 vCPU, 2 GB RAM, 8 GB Disk',
 '{"vcpu": 2, "ram_mib": 2048, "disk_gib": 8, "flavor_type": "vm"}', NOW()),

-- flavor-1
('74616279-b953-4f48-9728-d56ae8ec7fdb', 'flavor-1', '1 vCPU, 256 MB RAM, 1 GB Disk',
 '{"vcpu": 1, "ram_mib": 256, "disk_gib": 1, "flavor_type": "vm"}', NOW()),

-- flavor-11
('814e9ae2-590d-40a9-b5d2-2b17347299fe', 'flavor-11', '4 vCPU, 8193 MB RAM, 200 GB Disk',
 '{"vcpu": 4, "ram_mib": 8193, "disk_gib": 200, "flavor_type": "vm"}', NOW()),

-- flavor-12
('8fa2a8b2-f1d6-4cd6-8b7b-21c4d5369c36', 'flavor-12', '4 vCPU, 8 GB RAM, 80 GB Disk',
 '{"vcpu": 4, "ram_mib": 8192, "disk_gib": 80, "flavor_type": "vm"}', NOW()),

-- flavor-8
('a20eb9f1-559d-49cd-b55b-d26d76fbb93c', 'flavor-8', '8 vCPU, 16 GB RAM, 64 GB Disk',
 '{"vcpu": 8, "ram_mib": 16384, "disk_gib": 64, "flavor_type": "vm"}', NOW()),

-- flavor-3
('a5b72cd9-2c75-44db-8406-7f4d0826bbf8', 'flavor-3', '1 vCPU, 1 GB RAM, 4 GB Disk',
 '{"vcpu": 1, "ram_mib": 1024, "disk_gib": 4, "flavor_type": "vm"}', NOW()),

-- flavor-7
('b88eb0cc-68c6-43e6-9add-fceb80bc859f', 'flavor-7', '8 vCPU, 12 GB RAM, 48 GB Disk',
 '{"vcpu": 8, "ram_mib": 12288, "disk_gib": 48, "flavor_type": "vm"}', NOW()),

-- flavor-9
('bfa4f841-907a-42b0-b5a6-2b5e219d1d0b', 'flavor-9', '16 vCPU, 16 GB RAM, 64 GB Disk',
 '{"vcpu": 16, "ram_mib": 16386, "disk_gib": 64, "flavor_type": "vm"}', NOW()),

-- flavor-10
('c5e2b5e4-672f-4b1f-a3c0-6e20ef5d1ff6', 'flavor-10', '16 vCPU, 16 GB RAM, 512 GB Disk',
 '{"vcpu": 16, "ram_mib": 16386, "disk_gib": 512, "flavor_type": "vm"}', NOW());



Côté utilisateur :

slices_bi=# INSERT INTO users (
    user_id, preferred_username, first_name, last_name, affiliation, organization, email, updated_at
)
VALUES (
    'user_default_01jxyzexampleuser123456789', -- TypeID (type="user", suffix aléatoire)
    'thomas',
    'Thomas',
    'Thomas',
    'UPV',
    'I2T',
    'thomas@example.com',
    NOW()
);
INSERT 0 1
slices_bi=# INSERT INTO projects (
    id, project_id, name, expires_at, updated_at
)
VALUES (
    DEFAULT,
    'proj_default_01jxyzexampleproj123456789', -- TypeID (type="proj")
    'Test Project',
    NOW() + INTERVAL '30 days',
    NOW()
);
INSERT 0 1
slices_bi=# INSERT INTO experiments (
    id, experiment_id, project_id, name, updated_at
)
VALUES (
    DEFAULT,
    'exp_default_01jxyzexampleexp123456789', -- TypeID (type="exp")
    'proj_default_01jxyzexampleproj123456789',
    'Test Experiment',
    NOW()
);
INSERT 0 1


Obtenir un experience_id :
python3 -c "from uuid_utils import uuid7; from slices_bi_blueprint_backend.utils.typeid import TypeID; print(TypeID.from_uuid(uuid7(), 'exp', 'default'))"

Obtenir un token JWT slice :
slices auth login
slices project use qkd
slices auth get-for-audience https://slices-bi-blueprint.ilabt.imec.be

lices experiment create my-first-exp
slices experiment jwt <exp_id>



