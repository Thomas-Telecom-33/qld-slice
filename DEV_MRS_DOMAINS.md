# SLICES BI Blueprint integration of MRS Domains

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 0  — Architecture

#### 2 VMs :
- VM MRS : 10.98.1.217
- VM BI Slices : 10.98.1.239
Toutes les 2 sont dans le même réseaux.

##### Vérification de la possibilité :
- Ping entre les IPs (10.98.1.239 vers 10.98.1.217)
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
```

---

### 1.3.1 Sur VM Slices :
```
10.98.1.217 keycloak.mrs.local
10.98.1.217 backend.mrs.local
10.98.1.217 portal.mrs.local
10.98.1.217 pgadmin.mrs.local
```

---

### 1.3.1 Sur Windows :
Excécuter Bloc-note en tant qu'administrateur.

Ouvrir C:\Windows\System32\drivers\etc\hosts et placer :

```bash
10.98.1.217 keycloak.mrs.local
10.98.1.217 backend.mrs.local
10.98.1.217 portal.mrs.local
10.98.1.217 pgadmin.mrs.local
```

---

### 1.4 Modifications du fichier compose.yml pour le container idp :
```python
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
```

Et modification du port de Databse :
```python
 ports:
      - "5433:5432" 
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

Faire :
```bash
docker compose down -v
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

1. 
Création du scope mrs-backend-aud
Mapper type  Audience
Included Custom Audience   mrs-backend
Add to access token ON
Add to token introspection ON

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

