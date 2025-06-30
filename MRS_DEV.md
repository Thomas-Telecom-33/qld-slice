# SLICES BI Blueprint integration of MRS

═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 1 — Installations et configurations nécessaires

### 1.1 Cloner le dépôt
```bash
git clone https://gitlab.inria.fr/slices-ri/mrs.git
```
Puis :
```bash
cd mrs/infra/localhost-ports
```

---

### 1.2 Créer le fichier .env :
```env
MRS_C_BACKEND_PORT=9001
MRS_C_PORTAL_PORT=9002
MRS_C_IDP_PORT=9003
MRS_C_PGADMIN_PORT=9004

MRS_C_BACKEND_ORIGIN=http://localhost:9001
MRS_C_PORTAL_ORIGIN=http://localhost:9002
MRS_C_IDP_ORIGIN=http://localhost:9003

MRS_C_POSTGRES_PASSWORD=mrsdbpass
MRS_C_KEYCLOAK_ADMIN_PASSWORD=admin123
MRS_C_PGADMIN_DEFAULT_PASSWORD=pgadmin123

MCR_C_DB_UP_BACKEND=mrsdbpass
MCR_C_DB_UP_KEYCLOAK=keycloakpass
```

---

### 1.3 Nettoyer le fichier slices.realm.json
Certaines propriétés ne sont plus supportées dans la version actuelle de Keycloak. 
```bash
jq 'del(
  .maxTemporaryLockouts,
  .bruteForceStrategy,
  .localizationTexts,
  .webAuthnPolicyExtraOrigins,
  .verifiableCredentialsEnabled,
  .adminPermissionsEnabled
)' containers/keycloak/slices.realm.json > tmp && mv tmp containers/keycloak/slices.realm.json
```

---

### 1.4 Modifier compose.yml de cette façon
```python
version: '3'

name: slices-mrs-localhost-ports

services:
  database:
    image: postgres
    environment:
      POSTGRES_PASSWORD: ${MRS_C_POSTGRES_PASSWORD}
      MCR_C_DB_UP_BACKEND: ${MCR_C_DB_UP_BACKEND}
      MCR_C_DB_UP_KEYCLOAK: ${MCR_C_DB_UP_KEYCLOAK}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - source: '../containers/postgres/init/'
        target: '/docker-entrypoint-initdb.d/'
        read_only: true
        type: 'bind'
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
    #image: slices-ri/mrs-backend:latest
    environment:
      ASPNETCORE_ConnectionStrings__DefaultConnection: "Host=database;Database=mrs_backend;Username=mrs_backend;Password=${MCR_C_DB_UP_BACKEND};"
      ASPNETCORE_App__Auth__OidcMetadataAddress: "http://idp:8080/realms/slices/.well-known/openid-configuration"
      ASPNETCORE_App__Auth__OidcMetadataRequireHttps: 'false'
      ASPNETCORE_App__Auth__OidcAudience: slices-mrs-backend
      ASPNETCORE_App__Auth__SwaggerAuthorizationUrl: "${MRS_C_IDP_ORIGIN}/realms/slices/protocol/openid-connect/auth"
      ASPNETCORE_App__Auth__SwaggerTokenUrl: "${MRS_C_IDP_ORIGIN}/realms/slices/protocol/openid-connect/token"
      ASPNETCORE_App__Auth__SwaggerUiClientId: slices-mrs-backend-swagger
      ASPNETCORE_App__Auth__SwaggerUiClientRealm: slices
      ASPNETCORE_App__Auth__SwaggerUiUsePkce: 'false'
      ASPNETCORE_App__AutoMigrateDb: 'true'
    ports:
      - published: '${MRS_C_BACKEND_PORT}'
        target: 80

  portal:
    build:
      context: ../../SlicesMrs.Spa
    #image: slices-ri/mrs-portal:latest
    # Important: for NG_APP_ variables escape the following characters if used: ` ' "
    environment:
      NG_APP_OIDC_AUTHORITY: "${MRS_C_IDP_ORIGIN}/realms/slices"
      NG_APP_OIDC_CLIENT_ID: slices-mrs-spa
      NG_APP_MY_ACCOUNT_HREF: "${MRS_C_IDP_ORIGIN}/realms/slices/account"
      NG_APP_BACKEND_API_BASE: "${MRS_C_BACKEND_ORIGIN}"
    ports:
      - published: '${MRS_C_PORTAL_PORT}'
        target: 80

  idp:
    depends_on:
      database:
        condition: service_healthy
    image: quay.io/keycloak/keycloak:26.0.0
    command: start-dev --import-realm --hostname-strict=false --hostname=localhost
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: database
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${MCR_C_DB_UP_KEYCLOAK}
      KC_DB_URL_DATABASE: keycloak

      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: ${MRS_C_KEYCLOAK_ADMIN_PASSWORD}

      KC_PROXY: edge
      KC_HTTP_ENABLED: true

      KCRI_BACKEND_ORIGIN: "${MRS_C_BACKEND_ORIGIN}"
      KCRI_PORTAL_ORIGIN: "${MRS_C_PORTAL_ORIGIN}"
    volumes:
      - source: '../containers/keycloak/slices.realm.json'
        target: '/opt/keycloak/data/import/slices.realm.json'
        read_only: true
        type: 'bind'
    ports:
      - published: '${MRS_C_IDP_PORT}'
        target: 8080

  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin4_container
    ports:
      - published: ${MRS_C_PGADMIN_PORT}
        target: 80
    environment:
      PGADMIN_DEFAULT_EMAIL: admin1@example.com
      PGADMIN_DEFAULT_PASSWORD: ${MRS_C_PGADMIN_DEFAULT_PASSWORD}
    volumes:
      - pgadmin_data:/var/lib/pgadmin

volumes:
  postgres_data:
  pgadmin_data:
```

---

### 1.5 Démarrage minimal pour le Keyloack :
```bash
docker compose up -d database idp
```
Attends que Keycloak soit disponible sur http://localhost:9003. Tu verras apparaître une page demandant la création de l’utilisateur admin temporaire.

Renseigne :
```python
Utilisateur : admin
```
```python
Mot de passe : admin123 (via .env → KC_BOOTSTRAP_ADMIN_PASSWORD)
```

---

### 1.6 Une fois connecté à Keycloak :

- Selectionner SLICES en haut à gauche.

Ajout d'un client : 

Clients > Create Client
Client ID : `slices-mrs-backend-swagger`
Client Auth : `ON`
Root URL : `http://localhost:9001`
Valid redirect URIs : `http://localhost:9001/swagger/oauth2-redirect.html`
Web origins : `+`
Puis save.

Le client est alors crée.

On peut récupérer son mot de passe dans la catégorie `Credentials`.

- Puis aller dans Realm settings :
Frontend URL 
```bash
http://localhost:9003
```

---

### 1.7 Lancement des containers
```bash
docker compose up -d
```

Pour vérifier que tout est build :
```bash
docker ps
```
> Backend API	9001	http://localhost:9001	
>
> Frontend SPA	9002	http://localhost:9002	
> 
> Keycloak (IDP)	9003	http://localhost:9003	
> 
> PgAdmin	9004	http://localhost:9004

---

### 1.8 Accès au swaguer http://localhost:9001/ :
Bouton `Authorize` et se connecter avec le client.

On est alors authentifié sur le swaguer.


═══════════════════════════════════════════════════════════════════════════════════

## ÉTAPE 2 — 
