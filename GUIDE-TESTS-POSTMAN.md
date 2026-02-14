# Guide de Tests Postman - User Service

## Prérequis

### 1. Démarrage des services

```bash
cd /home/aimerou/Documents/TERANGA-BIZ/plateforme-teranga-biz
docker-compose up -d
```

Vérifier que tous les services sont actifs :
- PostgreSQL (port 5432)
- Redis (port 6379)
- Eureka Server (http://localhost:8761)
- Config Server (http://localhost:8888)
- API Gateway (http://localhost:8080)
- User Service (http://localhost:8081)

### 2. Configuration Postman

#### Variables d'environnement
Créer un environnement "Teranga-Biz-Local" avec ces variables :

| Variable | Valeur initiale | Valeur actuelle |
|----------|----------------|-----------------|
| `base_url` | `http://localhost:8080` | |
| `user_service_url` | `http://localhost:8081` | |
| `access_token` | | (auto-rempli) |
| `refresh_token` | | (auto-rempli) |
| `user_id` | | (auto-rempli) |
| `admin_token` | | (auto-rempli) |

---

## Tests à effectuer (ordre recommandé)

### TEST 1 : Inscription d'un utilisateur standard

**Endpoint** : `POST /api/v1/auth/register`  
**URL** : `{{base_url}}/api/v1/auth/register`  
**Authentification** : Aucune (public)

**Headers** :
```
Content-Type: application/json
```

**Body (JSON)** :
```json
{
  "firstName": "Jean",
  "lastName": "Dupont",
  "email": "jean.dupont@example.com",
  "phoneNumber": "+221771234567",
  "password": "Password123!",
  "confirmPassword": "Password123!"
}
```

**Réponse attendue** (201 Created) :
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "550e8400-e29b-41d4-a716-446655440000",
  "tokenType": "Bearer",
  "user": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "firstName": "Jean",
    "lastName": "Dupont",
    "email": "jean.dupont@example.com",
    "phoneNumber": "+221771234567",
    "isActive": true,
    "isVerified": false,
    "createdAt": "2026-02-14T10:30:00",
    "updatedAt": "2026-02-14T10:30:00"
  }
}
```

**Script Post-response (Tests tab)** :
```javascript
pm.test("Status code is 201", function () {
    pm.response.to.have.status(201);
});

pm.test("Response has access token", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.accessToken).to.be.a('string');
    pm.environment.set("access_token", jsonData.accessToken);
});

pm.test("Response has refresh token", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.refreshToken).to.be.a('string');
    pm.environment.set("refresh_token", jsonData.refreshToken);
});

pm.test("Response has user ID", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.user.id).to.be.a('string');
    pm.environment.set("user_id", jsonData.user.id);
});
```

**Cas d'erreur à tester** :

- Email déjà existant (400)
- Mot de passe trop faible (400)
- Téléphone invalide (400)
- Passwords non identiques (400)

---

### TEST 2 : Inscription d'un administrateur

**Endpoint** : `POST /api/v1/auth/register`  
**URL** : `{{base_url}}/api/v1/auth/register`

**Body (JSON)** :
```json
{
  "firstName": "Admin",
  "lastName": "Principal",
  "email": "admin@teranga-biz.com",
  "phoneNumber": "+221779876543",
  "password": "AdminSecure2026!",
  "confirmPassword": "AdminSecure2026!"
}
```

**Script Post-response** :
```javascript
pm.test("Status code is 201", function () {
    pm.response.to.have.status(201);
});

var jsonData = pm.response.json();
pm.environment.set("admin_token", jsonData.accessToken);
pm.environment.set("admin_refresh_token", jsonData.refreshToken);
pm.environment.set("admin_id", jsonData.user.id);
```

**Note** : Après cette inscription, vous devrez manuellement attribuer le rôle ADMIN via la base de données :

```sql
-- Récupérer l'ID du rôle ADMIN
SELECT id FROM roles WHERE name = 'ADMIN';

-- Attribuer le rôle (remplacer {admin_user_id} et {admin_role_id})
INSERT INTO user_roles (user_id, role_id, assigned_at) 
VALUES ('{admin_user_id}', '{admin_role_id}', NOW());
```

---

### TEST 3 : Connexion utilisateur

**Endpoint** : `POST /api/v1/auth/login`  
**URL** : `{{base_url}}/api/v1/auth/login`  
**Authentification** : Aucune (public)

**Body (JSON)** :
```json
{
  "email": "jean.dupont@example.com",
  "password": "Password123!"
}
```

**Réponse attendue** (200 OK) :
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "660e9500-f39c-51e5-b827-557766551111",
  "tokenType": "Bearer",
  "user": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "firstName": "Jean",
    "lastName": "Dupont",
    "email": "jean.dupont@example.com",
    "phoneNumber": "+221771234567",
    "isActive": true,
    "isVerified": false
  }
}
```

**Script Post-response** :
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Tokens are different from registration", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.accessToken).to.not.equal(pm.environment.get("access_token"));
    pm.environment.set("access_token", jsonData.accessToken);
    pm.environment.set("refresh_token", jsonData.refreshToken);
});
```

**Cas d'erreur à tester** :

- Email inexistant (401)
- Mot de passe incorrect (401)
- Compte désactivé (403)

---

### TEST 4 : Récupération du profil utilisateur

**Endpoint** : `GET /api/v1/users/{id}/profile`  
**URL** : `{{base_url}}/api/v1/users/{{user_id}}/profile`  
**Authentification** : Bearer Token

**Headers** :
```
Authorization: Bearer {{access_token}}
```

**Réponse attendue** (200 OK) :
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "firstName": "Jean",
  "lastName": "Dupont",
  "email": "jean.dupont@example.com",
  "phoneNumber": "+221771234567",
  "isActive": true,
  "isVerified": false,
  "createdAt": "2026-02-14T10:30:00",
  "updatedAt": "2026-02-14T10:30:00"
}
```

**Script Post-response** :
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("User ID matches", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.id).to.equal(pm.environment.get("user_id"));
});

pm.test("Email is correct", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.email).to.equal("jean.dupont@example.com");
});
```

**Cas d'erreur à tester** :

- Sans token (401)
- Avec token expiré (401)
- Accès au profil d'un autre utilisateur sans être admin (403)
- ID utilisateur inexistant (404)

---

### TEST 5 : Mise à jour du profil utilisateur

**Endpoint** : `PUT /api/v1/users/{id}/profile`  
**URL** : `{{base_url}}/api/v1/users/{{user_id}}/profile`  
**Authentification** : Bearer Token

**Headers** :
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

**Body (JSON)** :
```json
{
  "firstName": "Jean-Pierre",
  "lastName": "Dupont-Martin",
  "phoneNumber": "+221771234999"
}
```

**Réponse attendue** (200 OK) :
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "firstName": "Jean-Pierre",
  "lastName": "Dupont-Martin",
  "email": "jean.dupont@example.com",
  "phoneNumber": "+221771234999",
  "isActive": true,
  "isVerified": false,
  "createdAt": "2026-02-14T10:30:00",
  "updatedAt": "2026-02-14T10:35:00"
}
```

**Script Post-response** :
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("FirstName is updated", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.firstName).to.equal("Jean-Pierre");
});

pm.test("Phone is updated", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.phoneNumber).to.equal("+221771234999");
});
```

**Cas d'erreur à tester** :

- Numéro téléphone invalide (400)
- Mise à jour email (devrait être refusé - 400)
- Sans autorisation (403)

---

### TEST 6 : Changement de mot de passe

**Endpoint** : `POST /api/v1/users/{id}/change-password`  
**URL** : `{{base_url}}/api/v1/users/{{user_id}}/change-password`  
**Authentification** : Bearer Token

**Headers** :
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

**Body (JSON)** :
```json
{
  "currentPassword": "Password123!",
  "newPassword": "NewPassword456!",
  "confirmNewPassword": "NewPassword456!"
}
```

**Réponse attendue** (200 OK) :
```json
{
  "message": "Mot de passe changé avec succès"
}
```

**Script Post-response** :
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Success message is returned", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.message).to.include("succès");
});
```

**Cas d'erreur à tester** :

- Ancien mot de passe incorrect (401)
- Nouveau mot de passe trop faible (400)
- Mots de passe confirmation non identiques (400)
- Nouveau mot de passe identique à l'ancien (400)

**Important** : Après ce test, se reconnecter avec le nouveau mot de passe pour les tests suivants.

---

### TEST 7 : Re-connexion après changement de mot de passe

**Endpoint** : `POST /api/v1/auth/login`  
**URL** : `{{base_url}}/api/v1/auth/login`

**Body (JSON)** :
```json
{
  "email": "jean.dupont@example.com",
  "password": "NewPassword456!"
}
```

**Script Post-response** :
```javascript
pm.test("Login with new password successful", function () {
    pm.response.to.have.status(200);
    var jsonData = pm.response.json();
    pm.environment.set("access_token", jsonData.accessToken);
    pm.environment.set("refresh_token", jsonData.refreshToken);
});
```

---

### TEST 8 : Refresh Token (rotation)

**Endpoint** : `POST /api/v1/auth/refresh-token`  
**URL** : `{{base_url}}/api/v1/auth/refresh-token`  
**Authentification** : Aucune (public)

**Headers** :
```
Content-Type: application/json
```

**Body (JSON)** :
```json
{
  "refreshToken": "{{refresh_token}}"
}
```

**Réponse attendue** (200 OK) :
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "770f0600-g40d-62f6-c938-668877662222",
  "tokenType": "Bearer",
  "user": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "firstName": "Jean-Pierre",
    "lastName": "Dupont-Martin",
    "email": "jean.dupont@example.com"
  }
}
```

**Script Post-response** :
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("New access token received", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.accessToken).to.be.a('string');
    pm.expect(jsonData.accessToken).to.not.equal(pm.environment.get("access_token"));
    pm.environment.set("access_token", jsonData.accessToken);
});

pm.test("New refresh token received (rotation)", function () {
    var jsonData = pm.response.json();
    var oldRefreshToken = pm.environment.get("refresh_token");
    pm.expect(jsonData.refreshToken).to.not.equal(oldRefreshToken);
    pm.environment.set("old_refresh_token", oldRefreshToken);
    pm.environment.set("refresh_token", jsonData.refreshToken);
});
```

**Cas d'erreur à tester** :

- Refresh token invalide (404)
- Refresh token expiré (400)
- Réutilisation ancien refresh token après rotation (400 - révoqué)

---

### TEST 9 : Vérification rotation (ancien token révoqué)

**Endpoint** : `POST /api/v1/auth/refresh-token`  
**URL** : `{{base_url}}/api/v1/auth/refresh-token`

**Body (JSON)** :
```json
{
  "refreshToken": "{{old_refresh_token}}"
}
```

**Réponse attendue** (400 Bad Request) :
```json
{
  "error": "Refresh token révoqué"
}
```

**Script Post-response** :
```javascript
pm.test("Old refresh token is revoked", function () {
    pm.response.to.have.status(400);
});

pm.test("Error message indicates revoked token", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.error).to.include("révoqué");
});
```

---

### TEST 10 : Logout (device actuel uniquement)

**Endpoint** : `POST /api/v1/auth/logout`  
**URL** : `{{base_url}}/api/v1/auth/logout`  
**Authentification** : Bearer Token

**Headers** :
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

**Body (JSON)** :
```json
{
  "refreshToken": "{{refresh_token}}"
}
```

**Réponse attendue** (200 OK) :
```json
{
  "message": "Déconnexion réussie"
}
```

**Script Post-response** :
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Success message received", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.message).to.include("Déconnexion");
});
```

**Vérification** : Essayer d'utiliser le refresh token juste après le logout (devrait échouer).

---

### TEST 11 : Vérification token révoqué après logout

**Endpoint** : `POST /api/v1/auth/refresh-token`  
**URL** : `{{base_url}}/api/v1/auth/refresh-token`

**Body (JSON)** :
```json
{
  "refreshToken": "{{refresh_token}}"
}
```

**Réponse attendue** (400 Bad Request) :
```json
{
  "error": "Refresh token révoqué"
}
```

**Script Post-response** :
```javascript
pm.test("Logged out token is revoked", function () {
    pm.response.to.have.status(400);
});
```

---

### TEST 12 : Re-connexion pour test multi-devices

**Endpoint** : `POST /api/v1/auth/login`  
**URL** : `{{base_url}}/api/v1/auth/login`

**Body (JSON)** :
```json
{
  "email": "jean.dupont@example.com",
  "password": "NewPassword456!"
}
```

**Script Post-response** :
```javascript
var jsonData = pm.response.json();
pm.environment.set("access_token", jsonData.accessToken);
pm.environment.set("refresh_token_device1", jsonData.refreshToken);
```

---

### TEST 13 : Connexion depuis un 2ème device (simulation)

**Endpoint** : `POST /api/v1/auth/login`  
**URL** : `{{base_url}}/api/v1/auth/login`

**Headers** :
```
User-Agent: PostmanDevice2
```

**Body (JSON)** :
```json
{
  "email": "jean.dupont@example.com",
  "password": "NewPassword456!"
}
```

**Script Post-response** :
```javascript
var jsonData = pm.response.json();
pm.environment.set("access_token_device2", jsonData.accessToken);
pm.environment.set("refresh_token_device2", jsonData.refreshToken);
```

**Note** : Maintenant l'utilisateur a 2 refresh tokens actifs (device1 et device2).

---

### TEST 14 : Logout ALL devices (admin ou propriétaire)

**Endpoint** : `POST /api/v1/auth/logout-all`  
**URL** : `{{base_url}}/api/v1/auth/logout-all`  
**Authentification** : Bearer Token

**Headers** :
```
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

**Body (JSON)** :
```json
{
  "userId": "{{user_id}}"
}
```

**Réponse attendue** (200 OK) :
```json
{
  "message": "Déconnexion de tous les appareils réussie"
}
```

**Script Post-response** :
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("All devices logout successful", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.message).to.include("tous les appareils");
});
```

---

### TEST 15 : Vérification révocation device 1

**Endpoint** : `POST /api/v1/auth/refresh-token`  
**URL** : `{{base_url}}/api/v1/auth/refresh-token`

**Body (JSON)** :
```json
{
  "refreshToken": "{{refresh_token_device1}}"
}
```

**Réponse attendue** (400 Bad Request) :
```json
{
  "error": "Refresh token révoqué"
}
```

---

### TEST 16 : Vérification révocation device 2

**Endpoint** : `POST /api/v1/auth/refresh-token`  
**URL** : `{{base_url}}/api/v1/auth/refresh-token`

**Body (JSON)** :
```json
{
  "refreshToken": "{{refresh_token_device2}}"
}
```

**Réponse attendue** (400 Bad Request) :
```json
{
  "error": "Refresh token révoqué"
}
```

---

### TEST 17 : Connexion Admin

**Endpoint** : `POST /api/v1/auth/login`  
**URL** : `{{base_url}}/api/v1/auth/login`

**Body (JSON)** :
```json
{
  "email": "admin@teranga-biz.com",
  "password": "AdminSecure2026!"
}
```

**Script Post-response** :
```javascript
var jsonData = pm.response.json();
pm.environment.set("admin_token", jsonData.accessToken);
pm.environment.set("admin_refresh_token", jsonData.refreshToken);
pm.environment.set("admin_id", jsonData.user.id);
```

---

### TEST 18 : Admin accède au profil d'un autre utilisateur

**Endpoint** : `GET /api/v1/users/{id}/profile`  
**URL** : `{{base_url}}/api/v1/users/{{user_id}}/profile`  
**Authentification** : Bearer Token (admin)

**Headers** :
```
Authorization: Bearer {{admin_token}}
```

**Réponse attendue** (200 OK) : Profil de l'utilisateur Jean-Pierre

**Script Post-response** :
```javascript
pm.test("Admin can access other user profile", function () {
    pm.response.to.have.status(200);
});

pm.test("Retrieved correct user", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.email).to.equal("jean.dupont@example.com");
});
```

---

### TEST 19 : Admin supprime un utilisateur (soft delete)

**Endpoint** : `DELETE /api/v1/users/{id}`  
**URL** : `{{base_url}}/api/v1/users/{{user_id}}`  
**Authentification** : Bearer Token (admin)

**Headers** :
```
Authorization: Bearer {{admin_token}}
```

**Réponse attendue** (204 No Content) : Aucun body

**Script Post-response** :
```javascript
pm.test("Status code is 204", function () {
    pm.response.to.have.status(204);
});
```

---

### TEST 20 : Vérification utilisateur supprimé (soft delete)

**Endpoint** : `POST /api/v1/auth/login`  
**URL** : `{{base_url}}/api/v1/auth/login`

**Body (JSON)** :
```json
{
  "email": "jean.dupont@example.com",
  "password": "NewPassword456!"
}
```

**Réponse attendue** (403 Forbidden ou 401 Unauthorized) :
```json
{
  "error": "Compte désactivé ou supprimé"
}
```

**Script Post-response** :
```javascript
pm.test("Deleted user cannot login", function () {
    pm.expect([401, 403]).to.include(pm.response.code);
});
```

---

## Scénarios de tests d'erreur supplémentaires

### Sécurité

1. **Injection SQL** : Tester avec `email: "' OR '1'='1"`
2. **XSS** : Tester avec `firstName: "<script>alert('XSS')</script>"`
3. **Access Token manipulation** : Modifier manuellement un token JWT
4. **CSRF** : Vérifier que les endpoints modifiant l'état nécessitent un token

### Validation

1. **Email invalide** : `email: "notanemail"`
2. **Téléphone invalide** : `phoneNumber: "123"`
3. **Mot de passe faible** : `password: "123"`
4. **Champs manquants** : Body incomplet

### Limites

1. **Nom trop long** : `firstName: "a".repeat(300)`
2. **Email trop long** : `email: "a".repeat(250) + "@test.com"`

---

## Ordre recommandé d'exécution

1. Tests 1-2 : Inscriptions
2. Test 3 : Première connexion
3. Tests 4-5 : Profil (lecture/modification)
4. Tests 6-7 : Changement mot de passe
5. Tests 8-9 : Refresh token rotation
6. Tests 10-11 : Logout simple
7. Tests 12-16 : Multi-devices et logout-all
8. Tests 17-20 : Fonctionnalités admin

---

## Notes importantes

### Access Token Expiration
- Les access tokens expirent après 15 minutes
- Si un test échoue avec 401, rafraîchir le token avec TEST 8

### Refresh Token Expiration
- Les refresh tokens expirent après 7 jours
- Après expiration, l'utilisateur doit se reconnecter

### Nettoyage Base de données
Entre les séries de tests, vous pouvez nettoyer :

```sql
-- Supprimer tous les utilisateurs de test
DELETE FROM user_roles WHERE user_id IN (SELECT id FROM users WHERE email LIKE '%@example.com');
DELETE FROM refresh_tokens WHERE user_id IN (SELECT id FROM users WHERE email LIKE '%@example.com');
DELETE FROM users WHERE email LIKE '%@example.com';
```

### Vérifications dans la base de données

Après certains tests, vérifier manuellement :

```sql
-- Vérifier les refresh tokens actifs
SELECT u.email, rt.token, rt.is_revoked, rt.expires_at 
FROM refresh_tokens rt 
JOIN users u ON rt.user_id = u.id 
WHERE u.email = 'jean.dupont@example.com';

-- Vérifier les rôles assignés
SELECT u.email, r.name 
FROM users u 
JOIN user_roles ur ON u.id = ur.user_id 
JOIN roles r ON ur.role_id = r.id 
WHERE u.email = 'admin@teranga-biz.com';

-- Vérifier soft delete
SELECT email, deleted_at 
FROM users 
WHERE email = 'jean.dupont@example.com';
```

---

## Checklist finale

- [ ] Tous les 20 tests passent avec succès
- [ ] Refresh token rotation fonctionne (ancien token révoqué)
- [ ] Logout multi-devices fonctionne
- [ ] Admin peut gérer les autres utilisateurs
- [ ] Utilisateurs ne peuvent pas accéder aux profils des autres
- [ ] Changement mot de passe sécurisé (pas de logs)
- [ ] Soft delete fonctionne (utilisateur ne peut plus se connecter)
- [ ] Validation des données fonctionne (emails, téléphones, mots de passe)
- [ ] Tests d'erreur passent correctement
- [ ] Tokens expirés sont rejetés

---

## Support

Si un test échoue, vérifier :

1. Tous les services Docker sont actifs : `docker-compose ps`
2. Les logs des services : `docker-compose logs user-service`
3. La base de données PostgreSQL est accessible
4. Redis est actif (requis par api-gateway)
5. Les variables d'environnement Postman sont correctes
6. Le token n'est pas expiré

Pour reset complet :
```bash
docker-compose down -v
docker-compose up -d
```
