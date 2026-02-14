# Configuration Repository - Teranga Biz Platform

Ce dépôt contient les configurations centralisées pour tous les microservices de la plateforme Teranga Biz.

## Structure des Configurations

```
configurations-teranga-biz/
├── application.yml              # Configuration globale (tous les services)
├── eureka-server.yml           # Service Discovery
├── config-server.yml           # Config Server
├── api-gateway.yml             # API Gateway + Security
├── user-service.yml            # Users & Authentication
├── .env.example                # Template variables d'environnement
├── ARCHITECTURE.md             # Documentation architecture complète
├── CORRECTIONS-RAPPORT.md      # Rapport des corrections appliquées
└── README.md                   # Ce fichier
```

## Services Configurés

| Service | Fichier | Port | Description |
|---------|---------|------|-------------|
| Eureka Server | `eureka-server.yml` | 8761 | Service Discovery |
| Config Server | `config-server.yml` | 8888 | Configuration centralisée |
| API Gateway | `api-gateway.yml` | 8080 | Point d'entrée, routing, sécurité |
| User Service | `user-service.yml` | 8081 | Gestion utilisateurs & authentification |

## Configuration Globale (application.yml)

Contient les paramètres communs à tous les services :
- Configuration Eureka Client
- Configuration Spring Cloud Config
- Configuration Actuator (monitoring)
- Configuration logs

## Utilisation

### 1. En développement local

Chaque microservice charge automatiquement sa configuration depuis ce repository via le Config Server :

```yaml
spring:
  cloud:
    config:
      uri: http://config-server:8888
      name: ${spring.application.name}
```

### 2. Mise à jour des configurations

1. Modifier le fichier de configuration souhaité
2. Commit et push sur la branche `main`
3. Les services rechargent automatiquement via Spring Cloud Config

```bash
git add .
git commit -m "update: configuration user-service"
git push origin main
```

### 3. Rafraîchissement manuel (si nécessaire)

```bash
# Déclencher le rechargement sans redémarrage
curl -X POST http://localhost:8081/actuator/refresh
```

## Variables d'Environnement

Les informations sensibles sont gérées via variables d'environnement :

### Secrets
- `JWT_SECRET` : Clé secrète JWT (générer avec `openssl rand -hex 32`)
- `POSTGRES_PASSWORD` : Mot de passe PostgreSQL
- `REDIS_PASSWORD` : Mot de passe Redis

### Base de données
- `POSTGRES_HOST` : Hôte PostgreSQL (défaut: `postgres`)
- `POSTGRES_PORT` : Port PostgreSQL (défaut: `5432`)
- `POSTGRES_DB` : Nom de la base (défaut: `postgres`)
- `POSTGRES_USER` : Utilisateur PostgreSQL (défaut: `postgres`)

### Configuration JPA
- `DDL_AUTO` : Mode Hibernate (défaut: `validate`)
  - `validate` : Production (vérifie le schéma)
  - `update` : Développement (met à jour automatiquement)
  - `create-drop` : Tests (recrée à chaque démarrage)

Voir `.env.example` pour la liste complète.

## Sécurité

### ⚠️ IMPORTANT - Production

1. **Ne jamais commiter de secrets** dans ce repository
2. Utiliser des variables d'environnement pour toutes les valeurs sensibles
3. En production, utiliser :
   - Secrets Kubernetes
   - AWS Secrets Manager
   - HashiCorp Vault
   - Spring Cloud Config Encryption (optionnel)

### Chiffrement (optionnel)

Pour chiffrer les valeurs sensibles dans les fichiers YAML :

```yaml
# Config Server
spring:
  cloud:
    config:
      server:
        encrypt:
          enabled: true

# Dans les fichiers de config
password: '{cipher}AQBvzOcvVm...'
```

## Architecture Complète

Voir `ARCHITECTURE.md` pour :
- Diagramme d'architecture
- Liste de tous les microservices planifiés
- Plan de développement par phases
- Bonnes pratiques appliquées

## Rapport de Corrections

Voir `CORRECTIONS-RAPPORT.md` pour :
- Liste des incohérences détectées
- Corrections appliquées
- État final du système
- Prochaines étapes

## Support

- **Repository principal** : https://github.com/AIMEROU-DIA/plateforme-teranga-biz
- **Configuration** : https://github.com/AIMEROU-DIA/configurations-teranga-biz
- **Documentation** : `ARCHITECTURE.md`

---

**Dernière mise à jour** : 2026-02-14  
**Version** : 1.0.0-MVP  
**Statut** : ✅ Production-ready