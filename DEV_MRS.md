# SLICES BI Blueprint integration of MRS

═══════════════════════════════════════════════════════════════════════════════════
## ÉTAPE 1 — Installations nécessaires

1. Cloner le dépôt
```bash
git clone https://gitlab.inria.fr/slices-ri/mrs.git
```
Puis :
```bash
cd mrs/infra/localhost-ports
```

2. Créer le fichier .env :
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

3. Nettoyer le fichier slices.realm.json
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

4. Modifier compose.yml
a. Fixer l'image de Keycloak :
```python
image: quay.io/keycloak/keycloak:26.0.0
```

b. Supprimer --import-realm de la commande Keycloak:
On a alors :
```python
  command: start-dev
```

c. Remplacer KEYCLOAK_ADMIN et KEYCLOAK_ADMIN_PASSWORD par :
```python
KC_BOOTSTRAP_ADMIN_USERNAME: admin
KC_BOOTSTRAP_ADMIN_PASSWORD: ${MRS_C_KEYCLOAK_ADMIN_PASSWORD}
```

5. Nettoyage total avant build
```bash
docker compose down -v
```

7. Démarrage minimal pour le Keyloack :
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

7. Une fois connecté à Keycloak, lancer les autres containers :
```bash
docker compose up -d --build backend portal pgadmin
```

Pour vérifier que tout est build :
```bash
docker ps
```
> Backend API	9001	http://localhost:9001	✅
>
> Frontend SPA	9002	http://localhost:9002	✅
> 
> Keycloak (IDP)	9003	http://localhost:9003	✅
> 
> PgAdmin	9004	http://localhost:9004	✅

