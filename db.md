# Template de base :

docker ps

docker exec -it slices-bi-blueprint_devcontainer-db-1 psql -U slices_bi -d slices_bi

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

# Adaptation à notre infrastructure 

## Images :



## Flavors :




