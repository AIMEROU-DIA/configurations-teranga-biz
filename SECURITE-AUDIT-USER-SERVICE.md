# AUDIT SÉCURITÉ USER-SERVICE - RAPPORT COMPLET

Date : 2026-02-14  
Statut : ✅ Corrections critiques terminées | 🔄 Corrections majeures en cours

---

## RÉSUMÉ EXÉCUTIF

### Problèmes Identifiés

| Sévérité | Nombre | Statut |
|----------|--------|--------|
| **CRITIQUE** | 1 | ✅ Corrigé |
| **MAJEUR** | 5 | 🔄 En cours |
| **MINEUR** | 3 | ⏳ Planifié |

---

## CORRECTIONS CRITIQUES APPLIQUÉES ✅

### 1. ✅ Faille Exposition Mots de Passe dans Logs (CRITIQUE)

**Problème** :  
L'endpoint `POST /api/v1/users/{id}/change-password` utilisait `@RequestParam` pour `oldPassword` et `newPassword`.  
Les paramètres de requête apparaissent dans :
- URL (logs serveurs web, proxys, navigateurs)
- Historique bash/curl
- Logs de monitoring

**Impact** :  
🔴 **Exposition de mots de passe en clair dans plusieurs couches de logging**

**Solution** :
1. Créé `ChangePasswordRequest.java` DTO avec validation robuste :
   - `@NotBlank` sur tous les champs
   - `@Size(min=8, max=100)` sur newPassword
   - `@Pattern` pour complexité (majuscule, minuscule, chiffre, spécial)
   - Champ `confirmPassword` pour double vérification

2. Modifié `UserController.changePassword()` :
   - `@RequestParam` → `@Valid @RequestBody ChangePasswordRequest`
   - Mots de passe maintenant dans le corps HTTP (non loggé par défaut)

3. Amélioré `UserService.changePassword()` :
   - Vérification confirmation = nouveau mot de passe
   - Vérification nouveau ≠ ancien mot de passe
   - Messages d'erreur en français explicites

**Fichiers modifiés** :
- ✅ `/dto/ChangePasswordRequest.java` (nouveau)
- ✅ `/controller/UserController.java`
- ✅ `/service/UserService.java`

---

## CORRECTIONS MAJEURES EN COURS 🔄

### 2. ⚠️ Lombok @Builder.Default Non Utilisé (MAJEUR)

**Problème** :  
Plusieurs entités et DTOs définissent des valeurs par défaut mais n'utilisent pas `@Builder.Default`.  
Conséquence : le builder Lombok ignore ces valeurs et crée des objets avec `null`.

**Fichiers concernés** :
- `/dto/AuthResponse.java` : `tokenType = "Bearer"` → devient `null`
- `/entity/User.java` : `isActive = true`, `isVerified = false`, etc. → deviennent `null`
- `/entity/Role.java` : `isSystem = false`, `isActive = true` → deviennent `null`
- `/entity/RefreshToken.java` : `isRevoked = false` → devient `null`

**Solution requise** :
```java
// AVANT (incorrect)
@Builder
public class AuthResponse {
    private String tokenType = "Bearer";  // Ignoré par builder
}

// APRÈS (correct)
@Builder
public class AuthResponse {
    @Builder.Default
    private String tokenType = "Bearer";  // Utilisé par builder
}
```

**Impact** :
- NullPointerException potentielles
- Comportement imprévisible (utilisateurs inactifs par défaut)
- Tokens de type `null` au lieu de `"Bearer"`

**Action requise** : Ajouter `@Builder.Default` sur **tous les champs** avec initialisation.

---

### 3. ⚠️ Imports Inutilisés (MINEUR - Code Quality)

**Fichiers concernés** :
- `User.java` : `@CreationTimestamp`, `@UpdateTimestamp` (ligne 5-6)
- `UserService.java` : `List`, `Collectors` (supprimés ✅)

**Action requise** : Nettoyer les imports inutilisés.

---

### 4. ⚠️ Warnings Null Safety (MINEUR - Type Safety)

**Problème** :  
Le compilateur détecte des conversions non sûres dans :
- `AuthService.java` ligne 60, 191
- `RoleService.java` lignes 63, 82, 99, 122

**Exemple** :
```java
User user = userRepository.findByUsername(username)
    .orElseThrow(...);  // Type 'User' pas garanti @NonNull
```

**Solution** :
Ajouter annotations `@NonNull` ou utiliser `Objects.requireNonNull()`.

---

## CORRECTIONS MINEURES PLANIFIÉES ⏳

### 5. CORS Trop Permissif (MINEUR)

**Fichier** : `SecurityConfig.java`

**Problème actuel** :
```java
setAllowedOriginPatterns(List.of("*"))  // Accepte tous les domaines
```

**Solution production** :
```java
// Dans application-prod.yml
cors:
  allowed-origins:
    - https://teranga-biz.com
    - https://app.teranga-biz.com
```

---

### 6. Refresh Token Rotation Manquante (MINEUR)

**Fichier** : `AuthService.java`

**Problème** :  
La méthode `refreshToken()` renvoie le **même** refresh token au lieu d'en générer un nouveau.

**Bonne pratique (OAuth2 RFC 6749)** :  
Refresh Token Rotation = invalider l'ancien token et en créer un nouveau.

**Avantage** :  
Limite la fenêtre d'exploitation en cas de vol de token.

**Solution** :
```java
@Transactional
public AuthResponse refreshToken(String refreshTokenValue) {
    RefreshToken oldToken = // ...trouver et valider...
    
    // INVALIDER l'ancien
    oldToken.setIsRevoked(true);
    refreshTokenRepository.save(oldToken);
    
    // CRÉER un nouveau
    RefreshToken newToken = createRefreshToken(oldToken.getUser());
    
    // Retourner nouveau access + nouveau refresh
    return AuthResponse.builder()
        .accessToken(jwtService.generateToken(user))
        .refreshToken(newToken.getToken())
        .tokenType("Bearer")
        .user(mapToDTO(user))
        .build();
}
```

---

### 7. Logout Multi-Devices Ambigu (MINEUR)

**Fichier** : `AuthService.java`

**Problème** :
```java
public void logout(UUID userId) {
    refreshTokenRepository.findTopByUserIdOrderByCreatedAtDesc(userId)
        .ifPresent(refreshTokenRepository::delete);
}
```

Cette implémentation supprime uniquement le **dernier** token créé.  
Si l'utilisateur est connecté sur plusieurs appareils, seul le plus récent est déconnecté.

**Solutions possibles** :

**Option A** : Logout device actuel (recommandé)
```java
public void logout(String refreshTokenValue) {
    refreshTokenRepository.findByTokenAndIsRevokedFalse(refreshTokenValue)
        .ifPresent(token -> {
            token.setIsRevoked(true);
            refreshTokenRepository.save(token);
        });
}
```

**Option B** : Logout tous les devices
```java
public void logoutAll(UUID userId) {
    List<RefreshToken> tokens = refreshTokenRepository.findByUserIdAndIsRevokedFalse(userId);
    tokens.forEach(token -> token.setIsRevoked(true));
    refreshTokenRepository.saveAll(tokens);
}
```

---

## BONNES PRATIQUES DÉJÀ APPLIQUÉES ✅

### Sécurité

✅ **BCrypt avec force 12** : Très robuste contre brute-force  
✅ **JWT secret externalisé** : Via `application.yml` et variables d'environnement  
✅ **Validation stricte** : Regex pour email, téléphone, complexité mot de passe  
✅ **UUID pour IDs** : Empêche l'énumération des ressources  
✅ **Soft delete** : `deletedAt` au lieu de suppression physique  
✅ **@PreAuthorize** : Contrôle d'accès au niveau méthode  
✅ **UserSecurityService** : Vérification ownership (isOwnerOrAdmin)  
✅ **Exception handling centralisé** : Pas de fuite de stacktraces  

### Architecture

✅ **Séparation DTO/Entity** : Aucune entité JPA exposée directement  
✅ **@Transactional readOnly** : Optimisation lectures base de données  
✅ **FETCH JOIN** : Évite problème N+1 lors du chargement des rôles  
✅ **Index base de données** : Sur email, username, deleted_at  
✅ **Optimistic locking** : `@Version` sur User  
✅ **Audit JPA** : `createdAt`, `updatedAt` automatiques  

### Gestion des Rôles

✅ **Many-to-Many explicite** : Table `user_roles` avec métadonnées (assigned_by, expires_at)  
✅ **Permissions JSON** : Flexibilité sans complexifier le schéma  
✅ **Rôle par défaut** : `BUYER` attribué automatiquement à l'inscription  
✅ **Validation hiérarchique** : Vérification `isOwnerOrAdmin` avant actions sensibles  

---

## PLAN D'ACTION IMMÉDIAT

### Phase 1 : Corrections Majeures (Priorité HAUTE)

1. **Ajouter @Builder.Default** :
   - [ ] `AuthResponse.java` : tokenType
   - [ ] `User.java` : isActive, isVerified, userRoles, refreshTokens
   - [ ] `Role.java` : isSystem, isActive
   - [ ] `RefreshToken.java` : isRevoked

2. **Nettoyer imports inutilisés** :
   - [x] `UserService.java` (fait ✅)
   - [ ] `User.java` : ligne 5-6

3. **Corriger warnings null safety** :
   - [ ] `AuthService.java` : ligne 60, 191
   - [ ] `RoleService.java` : lignes 63, 69, 82, 99, 122

### Phase 2 : Corrections Mineures (Priorité MOYENNE)

4. **Refresh Token Rotation** :
   - [ ] Implémenter rotation dans `AuthService.refreshToken()`
   - [ ] Ajouter index sur `RefreshToken.isRevoked`

5. **Logout multi-devices** :
   - [ ] Ajouter endpoint `POST /api/v1/auth/logout-all`
   - [ ] Modifier endpoint actuel pour logout device actuel

6. **CORS production** :
   - [ ] Créer `application-prod.yml` avec domaines autorisés
   - [ ] Tester avec domaine production réel

### Phase 3 : Améliorations (Priorité BASSE)

7. **Authentification 2FA** (optionnel)
8. **Rate limiting sur endpoints auth** (via Gateway)
9. **Logs d'audit** (connexions, changements de rôles)
10. **Tests de sécurité** (OWASP Top 10)

---

## RECOMMANDATIONS PRODUCTION

### Secrets

```bash
# Générer un nouveau JWT secret (32+ caractères)
openssl rand -hex 32

# Configurer dans .env
JWT_SECRET=<nouveau_secret_généré>
POSTGRES_PASSWORD=<mot_de_passe_robuste>
REDIS_PASSWORD=<mot_de_passe_robuste>
```

### Configuration CORS

```yaml
# user-service-prod.yml
cors:
  allowed-origins:
    - https://teranga-biz.com
    - https://app.teranga-biz.com
  allowed-methods:
    - GET
    - POST
    - PUT
    - DELETE
  allowed-headers:
    - Authorization
    - Content-Type
  max-age: 3600
```

### Base de Données

```bash
# Créer schéma avec Flyway/Liquibase
DDL_AUTO=validate  # JAMAIS 'update' en prod
```

### Monitoring Sécurité

- Activer logs d'audit (connexions, échecs auth)
- Configurer alertes (tentatives brute-force, JWT invalides)
- Monitorer temps de réponse endpoints auth (attaque timing)

---

## CONCLUSION

**Statut global** : 🟢 Système globalement sécurisé

Le user-service applique déjà **la majorité des bonnes pratiques** de sécurité.  
La faille critique (exposition mots de passe) a été **corrigée immédiatement**.

Les corrections majeures restantes sont principalement :
- **Qualité de code** (@Builder.Default, imports)
- **Améliorations incrémentales** (token rotation, logout)

**Le service est prêt pour la production** après application des corrections Phase 1.

---

**Auteur** : Audit Sécurité Teranga Biz  
**Version** : 1.0.0  
**Dernière mise à jour** : 2026-02-14
