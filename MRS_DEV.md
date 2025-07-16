# SLICES BI Blueprint integration of MRS Domains

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 0  — Architecture

#### 2 VMs :
- VM MRS : 10.98.1.238
- VM BI Slices : 10.98.1.239
Toutes les 2 sont dans le même réseaux.

##### Vérification de la possibilité :
- Ping entre les IPs (10.98.1.239 vers 10.98.1.217) OK
- Ajout des entrées dans /etc/hosts
- ping backend.mrs.local OK
- curl -k https://backend.mrs.local/reporting/charts/objects-by-type
Endpoints du swagguer répondent.


## ÉTAPE 1 — Installations et configurations nécessaires

### 1.1 Cloner le dépôt
```bash
git clone https://gitlab.inria.fr/slices-ri/mrs.git
```
Puis :
```bash
cd mrs/infra/domains
```

---

### 1.2 Créer le fichier .env :
```env
# Images tag
MRS_C_LOCAL_IMAGES_TAG=latest

# Traefik config
MRS_C_TRAEFIK_ENTRYPOINT_UNSECURE=web
MRS_C_TRAEFIK_ENTRYPOINT_SECURE=websecure
MRS_C_TRAEFIK_NETWORK=traefik_services

# Domain names
MRS_C_BACKEND_HOST=backend.mrs.local
MRS_C_PORTAL_HOST=portal.mrs.local
MRS_C_IDP_HOST=keycloak.mrs.local
MRS_C_PGADMIN_HOST=pgadmin.mrs.local

# Admin credentials
MRS_C_POSTGRES_PASSWORD=
MRS_C_KEYCLOAK_ADMIN_PASSWORD=
MRS_C_PGADMIN_DEFAULT_PASSWORD=

# System user credentials
MCR_C_DB_UP_BACKEND=
MCR_C_DB_UP_KEYCLOAK=
```

---

### 1.3 Ajouter les noms de domaines :

### 1.3.1 Sur VM MRS :
Se rendre dans :
```bash
sudo nano /etc/hosts
```

Placer en fin de fichier :
```bash
127.0.0.1 keycloak.mrs.local
127.0.0.1 backend.mrs.local
127.0.0.1 portal.mrs.local
127.0.0.1 pgadmin.mrs.local
127.0.0.1 traefik.mrs.test
```

---

### 1.3.1 Sur VM Slices :
```
10.98.1.238 keycloak.mrs.local
10.98.1.238 backend.mrs.local
10.98.1.238 portal.mrs.local
10.98.1.238 pgadmin.mrs.local
```

---

### 1.3.1 Sur Windows :
Excécuter Bloc-note en tant qu'administrateur.

Ouvrir C:\Windows\System32\drivers\etc\hosts et placer :

```bash
10.98.1.238 keycloak.mrs.local
10.98.1.238 backend.mrs.local
10.98.1.238 portal.mrs.local
10.98.1.238 pgadmin.mrs.local
```

---

### 1.4 Modifications du fichier compose.yml :
```python
# Be compatible with docker swarm deployments (not tested) 
version: '3'

# Use a better name prefix for containers/volumes/networks than 'domains'
name: slices-mrs-domains

services:
  
  # The PostgreSQL database used by both the MRS backend and the Keycloak IdP
  database:
    image: postgres
    restart: unless-stopped
    environment:
      # The password for the superadmin 'postgres' user
      POSTGRES_PASSWORD: ${MRS_C_POSTGRES_PASSWORD}
      
      # The password for an internal user used by the MRS backend to access the database.
      # Used by the `init-mrs.sh` script.
      MCR_C_DB_UP_BACKEND: ${MCR_C_DB_UP_BACKEND}
      
      # The password for an internal user used by the Keycloak IdP to access the database.
      # Used by the `init-keycloak.sh` script.
      MCR_C_DB_UP_KEYCLOAK: ${MCR_C_DB_UP_KEYCLOAK}

    ports:
      - "5433:5432" 
    
    volumes:
      # The persistent volume so that the stored data can survive container recreations
      - postgres_data:/var/lib/postgresql/data
      
      # The initialization scripts
      - source: '../containers/postgres/init/'
        target: '/docker-entrypoint-initdb.d/'
        read_only: true
        type: 'bind'
    
    # Used to delay the start of backend and keycloak until the database is ready
    healthcheck:
      test: pg_isready
      interval: 10s
      timeout: 3s
      retries: 10
      start_period: 10s

  backend:
    # Delay until database is healthy, else the initial migration fails and the container crashes
    depends_on:
      database:
        condition: service_healthy
    build:
      context: ../../SlicesMrs.Backend
    image: slices-ri/mrs-backend:${MRS_C_LOCAL_IMAGES_TAG:-latest}
    environment:
      # The database connection string
      ASPNETCORE_ConnectionStrings__DefaultConnection: "Host=database;Database=mrs_backend;Username=mrs_backend;Password=${MCR_C_DB_UP_BACKEND};"
      
      # The OpenID Connect Discovery location. Will be fetched in order to validate the JWTs.
      # Note that we are using the internal DNS to simplify the connection, but this also means that we need to
      # use non-S http, since the cert is configured on traefik (which we are bypassing).
      ASPNETCORE_App__Auth__OidcMetadataAddress: "https://keycloak.mrs.local/realms/slices/.well-known/openid-configuration"
      ASPNETCORE_App__Auth__OidcMetadataRequireHttps: 'false'
      
      # The audience to check for when validating the JWTs. See OIDC spec for more information. 
      ASPNETCORE_App__Auth__OidcAudience: mrs-backend
      
      # The OIDC (OAuth2 to be specific) config used by the Swagger UI
      ASPNETCORE_App__Auth__SwaggerAuthorizationUrl: "https://${MRS_C_IDP_HOST}/realms/slices/protocol/openid-connect/auth"
      ASPNETCORE_App__Auth__SwaggerTokenUrl: "https://${MRS_C_IDP_HOST}/realms/slices/protocol/openid-connect/token"
      ASPNETCORE_App__Auth__SwaggerUiClientId: slices-mrs-backend-swagger
      ASPNETCORE_App__Auth__SwaggerUiClientRealm: slices
      
      # Apply database migrations (i.e. schema management) on startup.
      # Will cause the container to crash if not successfully.
      ASPNETCORE_App__AutoMigrateDb: 'true'
    restart: unless-stopped
    extra_hosts:
      - "keycloak.mrs.local:host-gateway"
    networks:
      # Needed to connect to the database
      - default
      
      # Needed for traefik to connect to us 
      - traefik
    labels:
      # Instructs traefik docker provider to process this container
      traefik.enable: true

      # Traefik HTTPS redirect middleware 
      # Single shared for all MRS services
      traefik.http.middlewares.slices-mrs-redirect-websecure.redirectscheme.scheme: https
      
      # Traefik config for HTTP requests.
      # Do the redirect in Traefik as aspnetcore's HttpsRedirectionMiddleware cannot figure out the destination port
      # when running behind a reverse proxy.
      traefik.http.routers.slices-mrs-backend-unsecure.middlewares: slices-mrs-redirect-websecure
      traefik.http.routers.slices-mrs-backend-unsecure.rule: 'Host(`${MRS_C_BACKEND_HOST}`)'
      traefik.http.routers.slices-mrs-backend-unsecure.entrypoints: ${MRS_C_TRAEFIK_ENTRYPOINT_UNSECURE}

      # Traefik config for HTTPS requests
      traefik.http.routers.slices-mrs-backend-secure.rule: 'Host(`${MRS_C_BACKEND_HOST}`)'
      traefik.http.routers.slices-mrs-backend-secure.entrypoints: ${MRS_C_TRAEFIK_ENTRYPOINT_SECURE}
      traefik.http.routers.slices-mrs-backend-secure.tls: true

      # Make sure traefik reaches us on the correct docker network
      traefik.docker.network: ${MRS_C_TRAEFIK_NETWORK}

      # Make sure traefik reaches us on the correct port
      traefik.http.services.backend-slicesmrs.loadbalancer.server.port: '80'

  portal:
    build:
      context: ../../SlicesMrs.Spa
    image: slices-ri/mrs-portal-domains:${MRS_C_LOCAL_IMAGES_TAG:-latest}
    restart: unless-stopped
    # Important: for NG_APP_ variables escape the following characters if used: ` ' "
    environment:
      # OIDC options
      NG_APP_OIDC_AUTHORITY: "https://${MRS_C_IDP_HOST}/realms/slices"
      NG_APP_OIDC_CLIENT_ID: slices-mrs-spa

      # The href for "my account" link
      NG_APP_MY_ACCOUNT_HREF: "https://${MRS_C_IDP_HOST}/realms/slices/account"

      # Where to point the AJAX calls and swagger links
      NG_APP_BACKEND_API_BASE: "https://${MRS_C_BACKEND_HOST}"
    networks:
      # Needed for traefik to connect to us.
      # Note no default network - we don't need to talk to the database
      - traefik
    labels:
      # Instructs traefik docker provider to process this container
      traefik.enable: true
      
      # Traefik config for HTTP requests
      traefik.http.routers.slices-mrs-portal-unsecure.middlewares: slices-mrs-redirect-websecure
      traefik.http.routers.slices-mrs-portal-unsecure.rule: 'Host(`${MRS_C_PORTAL_HOST}`)'
      traefik.http.routers.slices-mrs-portal-unsecure.entrypoints: ${MRS_C_TRAEFIK_ENTRYPOINT_UNSECURE}

      # Traefik config for HTTPS requests
      traefik.http.routers.slices-mrs-portal-secure.rule: 'Host(`${MRS_C_PORTAL_HOST}`)'
      traefik.http.routers.slices-mrs-portal-secure.entrypoints: ${MRS_C_TRAEFIK_ENTRYPOINT_SECURE}
      traefik.http.routers.slices-mrs-portal-secure.tls: true

      # Make sure traefik reaches us on the correct docker network
      traefik.docker.network: ${MRS_C_TRAEFIK_NETWORK}

  idp:
    # Delay until database is healthy, so that the initial bootstrap can complete successfully
    depends_on:
      database:
        condition: service_healthy
    # '-domains' due to ENV KC_PROXY=edge
    image: slices-ri/mrs-idp-domains:${MRS_C_LOCAL_IMAGES_TAG:-latest}
    build:
      context: ../containers/keycloak
    # --optimized makes sure that keycloak doesn't attempt to re-build itself at the start (causes longer start times)
    # --import-realm instructs keycloak to import the slices.realm.json
    command: start --optimized --import-realm
    restart: unless-stopped
    environment:
      # HTTP access options
      #KC_PROXY: edge - commented out as not needed due to custom image & --optimized
      #KC_HOSTNAME: ${MRS_C_IDP_HOST}
      KC_HTTP_ENABLED: "true"        
      KC_HTTP_PORT: "8080"    
      KC_PROXY: "edge" 
      KC_HOSTNAME: "https://keycloak.mrs.local"
      #KC_HOSTNAME: ${MRS_C_IDP_HOST}
      KC_HOSTNAME_STRICT_HTTPS: "true"

      # Database access options
      #KC_DB: postgres - commented out as not needed due to custom image & --optimized
      KC_DB_URL_HOST: database
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${MCR_C_DB_UP_KEYCLOAK}
      KC_DB_URL_DATABASE: keycloak

      # The initial admin user options. Ignored after the initial startup
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: ${MRS_C_KEYCLOAK_ADMIN_PASSWORD}

      # Options for the slices.realm.json file
      # KCRI: KeyCloak Realm Import
      KCRI_BACKEND_ORIGIN: "https://${MRS_C_BACKEND_HOST}"
      KCRI_PORTAL_ORIGIN: "https://${MRS_C_PORTAL_HOST}"
    networks:
      # Needed to connect to the database
      - default
      
      # Needed for traefik to connect to us 
      - traefik
    volumes:
      # The definition of the slices realm.
      # Used only during the initial startup, ignored afterwards
      - source: '../containers/keycloak/slices.realm.json'
        target: '/opt/keycloak/data/import/slices.realm.json'
        read_only: true
        type: 'bind'
    labels:
      # Instructs traefik docker provider to process this container
      traefik.enable: true

      # Traefik config for HTTP requests
      traefik.http.routers.slices-mrs-idp-unsecure.middlewares: slices-mrs-redirect-websecure
      traefik.http.routers.slices-mrs-idp-unsecure.rule: 'Host(`${MRS_C_IDP_HOST}`)'
      traefik.http.routers.slices-mrs-idp-unsecure.entrypoints: ${MRS_C_TRAEFIK_ENTRYPOINT_UNSECURE}

      # Traefik config for HTTPS requests
      traefik.http.routers.slices-mrs-idp-secure.rule: 'Host(`${MRS_C_IDP_HOST}`)'
      traefik.http.routers.slices-mrs-idp-secure.entrypoints: ${MRS_C_TRAEFIK_ENTRYPOINT_SECURE}
      traefik.http.routers.slices-mrs-idp-secure.tls: true

      # Make sure traefik reaches us on the correct docker network
      traefik.docker.network: ${MRS_C_TRAEFIK_NETWORK}

  # Utility to access the database without exposing the db server itself to the world
  pgadmin:
    image: dpage/pgadmin4
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin1@example.com
      PGADMIN_DEFAULT_PASSWORD: ${MRS_C_PGADMIN_DEFAULT_PASSWORD}
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      - default
      - traefik
    labels:
      traefik.enable: true

      traefik.http.routers.slices-mrs-pgadmin-unsecure.middlewares: slices-mrs-redirect-websecure
      traefik.http.routers.slices-mrs-pgadmin-unsecure.rule: 'Host(`${MRS_C_PGADMIN_HOST}`)'
      traefik.http.routers.slices-mrs-pgadmin-unsecure.entrypoints: ${MRS_C_TRAEFIK_ENTRYPOINT_UNSECURE}

      traefik.http.routers.slices-mrs-pgadmin-secure.rule: 'Host(`${MRS_C_PGADMIN_HOST}`)'
      traefik.http.routers.slices-mrs-pgadmin-secure.entrypoints: ${MRS_C_TRAEFIK_ENTRYPOINT_SECURE}
      traefik.http.routers.slices-mrs-pgadmin-secure.tls: true

      traefik.docker.network: ${MRS_C_TRAEFIK_NETWORK}

volumes:
  # The persistent volume for PostgreSQL
  postgres_data:

  # The persistent volume for pgAdmin
  pgadmin_data:

networks:
  # Network for communication between traefik and MRS containers.
  # external: true as it is expected to already exist, even before MRS is deployed
  traefik:
    external: true
    name: ${MRS_C_TRAEFIK_NETWORK}
```


---

### 1.5 Modification du dockerfile :

Se rendre dans :
```bash
cd /mrs/infra/containers/keycloak
```

Ici se trouve le dockerfile, il faut changer la version de Keycloack pour la version 26.2.5 :

On a alors :
```python
FROM quay.io/keycloak/keycloak:26.2.5 as builder
FROM quay.io/keycloak/keycloak:26.2.5
```

---

### 1.6 Lancement des containers

Lancer d'abord traefik :
```bash
cd /mrs/infra/traefik
```
Puis :
```bash
docker compose up -d 
```
Traefik est alors lancé.

Puis lancer les autres containers :
```bash
cd /mrs/infra/domains
```
Puis :
```bash
docker compose up -d 
```

Pour vérifier que tout est build :
```bash
docker ps
```
> mrs-idp-domains
>
> mrs-backend
> 
> pgadmin4
> 
> postgres
>
> mrs-portal-domains
>
> traefik
>

---

### 1.7 Accès à Keycloack

Sur le navigateur web, accéder à `https://keycloak.mrs.local/`

Se connecter avec le compte `admin` et le mot de passe définie dans le `.env`

Changer de realm et choisir el realm `slices`.

---

#### Créer un user :

Sous users > Add user et lui donner un mot de passe dans credentials.

---

#### Modifier le realm settings :

Frontend URL : `https://keycloak.mrs.local`

---

### 1.8 Accéder au portail

#### Modifier le client mrs-spa dans Keycloack :

Sous clients > mrs-spa modifier :

- Client ID : `slices-mrs-spa`
- Root URL : `https://portal.mrs.local`
- Valid redirect URIs : `https://portal.mrs.local/*`
- Valid post logout redirect URIs : `https://portal.mrs.local/*`
- Web origins : `https://portal.mrs.local`
  
Sur la navigateur web, accéder à `https://portal.mrs.local/`

Puis cliquer sur login.

On rentre les identifiants du user crée sous Keycloack.

Puis on remplit les informations demandées.

On est alors connecté sur le portail.

---

### 1.9 Accéder au swagger

#### Créer le client slices-mrs-backend-swagger dans Keycloack :
- Client ID : `slices-mrs-backend-swagger`
- Root URL : `https://backend.mrs.local/swagger`
- Valid redirect URIs : `https://backend.mrs.local/swagger/oauth2-redirect.html`
- Web origins : `+`
  
Sur la navigateur web, accéder à `https://backend.mrs.local/swagger/`

On a alors accès au swagger.

Cliquer sur le bouton `Authorize`, et se connecter avec le client créé slices-mrs-backend-swagger.

On est alors redirigé vers le portail `Keycloack` où l'on se conecte avec le user crée.

On est alors authorisé à utiliser le swagger.

Problèmes rencontrés et solution apportées :

1. Création du scope mrs-backend-aud :

- Mapper type : Audience
- Included Custom Audience : mrs-backend
- Add to access token : ON
- Add to token introspection : ON

L'ajouter au client slices-mrs-backend-swagger

2. Modification de appsettings.json :
```python
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "App": {
    "Auth": {
      "OidcMetadataAddress": "https://keycloak.mrs.local/realms/slices/.well-known/openid-configuration",
      "OidcMetadataRequireHttps": false,
      "OidcAudience": "mrs-backend",
      "SwaggerAuthorizationUrl": "https://keycloak.mrs.local/realms/slices/protocol/openid-connect/auth",
      "SwaggerTokenUrl": "https://keycloak.mrs.local/realms/slices/protocol/openid-connect/token",
      "SwaggerUiClientId": "slices-mrs-backend-swagger",
      "SwaggerUiClientRealm": "slices"
    }
  }
}
```

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 2 — Tests du SWAGGER

Des exemples de payloads sont à disposition sous `/mrs/payloads`. 

#### POST :

Exemple de payload pour la création d'une VM Openstack :
```python
{
  "$type": "Base",
  "identifier": "vm2",
  "name": "VM2",
  "description": "Virtual Machine",
  "resourceType": "Base",
  "createdAt": "2025-07-09T00:00:00Z",
  "version": "1.0",
  "creators": [
    {
      "firstName": "John",
      "lastName": "Doe",
      "email": "j.doe@example.com",
      "organization": "TestOrg",
      "website": "http://localhost"
    }
  ],
  "contributors": [],
  "accessType": "Remote",
  "accessMode": "Free",
  "accessRights": "Open access",
  "rights": "Standard usage rights apply.",
  "license": "",
  "licenseUri": "",
  "primaryLanguages": ["en"],
  "otherLanguages": [],
  "keywords": ["vm", "virtual machine", "test"],
  "subjects": ["computing"],
  "requiredObjects": [],
  "relatedObjects": [],
  "contact": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "j.doe@example.com",
    "organization": "TestOrg",
    "website": "http://localhost"
  },
  "publicContact": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "j.doe@example.com",
    "organization": "TestOrg",
    "website": "http://localhost"
  }
}

```

L'`external identifier` est alors généré automatiquement.

On peut alors utiliser les autres routes pour `DELETE`, `PUT` et `GET`.

═══════════════════════════════════════════════════════════════════════════════════

