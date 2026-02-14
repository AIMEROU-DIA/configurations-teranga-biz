# ARCHITECTURE - PLATEFORME TERANGA BIZ

## Vue d'ensemble

La plateforme Teranga Biz est une architecture microservices moderne construite avec Spring Boot et Spring Cloud, conçue pour être scalable, résiliente et prête pour la production.

## Statut Actuel (Phase MVP)

### Microservices Déployés ✅

| Service | Port | Rôle | Statut |
|---------|------|------|--------|
| **eureka-server** | 8761 | Service Discovery | ✅ Opérationnel |
| **config-server** | 8888 | Configuration centralisée | ✅ Opérationnel |
| **api-gateway** | 8080 | Point d'entrée unique | ✅ Opérationnel |
| **user-service** | 8081 | Gestion utilisateurs & auth | ✅ Opérationnel |

### Infrastructure

| Composant | Version | Rôle |
|-----------|---------|------|
| **PostgreSQL** | 15-alpine | Base de données principale |
| **Redis** | 7-alpine | Cache & Rate limiting |

---

## Architecture Technique

### Pattern Architectural
```
┌─────────────────────────────────────────────────┐
│              Client (Web/Mobile)                │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │   API Gateway (8080)  │ ◄── Redis (Rate Limiting)
          │  - Routage            │
          │  - Auth JWT           │
          │  - Rate Limiting      │
          └──────────┬────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │  User   │  │ Produits│  │ Commandes│
   │ Service │  │ Service │  │  Service │
   │ (8081)  │  │  (TODO) │  │  (TODO)  │
   └────┬────┘  └─────────┘  └─────────┘
        │
        ▼
   PostgreSQL
```

### Communication Inter-Services

1. **REST synchrone** : Requêtes API classiques
2. **Event-driven (TODO)** : RabbitMQ/Kafka pour événements asynchrones
3. **Service Discovery** : Eureka pour la découverte de services
4. **Configuration centralisée** : Config Server (GitHub)

---

## Configuration & Sécurité

### Configuration Centralisée

Dépôt Git : `https://github.com/AIMEROU-DIA/configurations-teranga-biz.git`

Structure :
```
configurations-teranga-biz/
├── application.yml          # Config commune à tous les services
├── eureka-server.yml        # Config Eureka
├── config-server.yml        # Config Config Server
├── api-gateway.yml          # Config Gateway
├── user-service.yml         # Config User Service
└── [futurs-services].yml    # Configs à venir
```

### Sécurité JWT

- **Algorithme** : HS256 (HMAC avec SHA-256)
- **Access Token** : 15 minutes (configurable via JWT_EXPIRATION)
- **Refresh Token** : 7 jours (configurable via JWT_REFRESH_EXPIRATION)
- **Secret partagé** : Entre Gateway et User Service

⚠️ **IMPORTANT** : En production, utiliser des secrets robustes via variables d'environnement.

---

## Déploiement

### Prérequis

- Docker & Docker Compose
- Java 17+
- Maven 3.8+

### Commandes

```bash
# Lancer l'infrastructure complète
docker-compose up -d

# Vérifier les services
docker-compose ps

# Logs d'un service
docker-compose logs -f user-service

# Arrêter tous les services
docker-compose down

# Rebuild après modification du code
docker-compose up -d --build
```

### Ordre de Démarrage (géré automatiquement)

1. PostgreSQL & Redis (healthcheck)
2. Eureka Server (healthcheck)
3. Config Server (healthcheck + depends_on Eureka)
4. User Service (depends_on Postgres + Config + Eureka)
5. API Gateway (depends_on Config + Eureka + Redis)

---

## Monitoring & Observabilité

### Endpoints Actuator (tous les services)

- **Health** : `/actuator/health`
- **Info** : `/actuator/info`
- **Metrics** : `/actuator/metrics`
- **Prometheus** : `/actuator/prometheus`

### Dashboard Eureka

URL : `http://localhost:8761`

Liste tous les services enregistrés et leur état.

---

## Plan de Développement Futur

### Phase 1 - MVP (EN COURS ✅)
- [x] Infrastructure de base (Eureka, Config, Gateway)
- [x] User Service (Auth, Roles)
- [x] PostgreSQL & Redis
- [x] Configuration production-ready

### Phase 2 - E-Commerce Core (À VENIR)
- [ ] **produits-service** : Catalogue produits
- [ ] **boutiques-service** : Mini-boutiques vendeurs
- [ ] **inventaire-service** : Gestion stocks
- [ ] **commandes-service** : Gestion commandes
- [ ] **paiements-service** : Intégration Stripe/PayPal
- [ ] **avis-service** : Notations & commentaires

### Phase 3 - Social & Engagement (À VENIR)
- [ ] **social-service** : Publications, likes, commentaires
- [ ] **messagerie-service** : Chat temps réel (WebSocket)
- [ ] **academie-service** : Formations & tutoriels
- [ ] **certification-service** : Badges & gamification

### Phase 4 - Monétisation & IA (À VENIR)
- [ ] **partenaires-service** : Prestataires externes
- [ ] **publicite-service** : Campagnes sponsorisées
- [ ] **ia-service** : Recommandations & chatbot (Python/FastAPI)
- [ ] **statistiques-service** : Analytics & dashboards

### Phase 5 - Scalabilité & Production (À VENIR)
- [ ] BFF GraphQL (agrégation de données frontend)
- [ ] Message Broker (RabbitMQ/Kafka)
- [ ] Kubernetes (orchestration production)
- [ ] CI/CD Pipeline (GitHub Actions)
- [ ] Distributed Tracing (Zipkin/Jaeger)
- [ ] Centralized Logging (ELK Stack)

---

## Bonnes Pratiques Appliquées

### Microservices
- ✅ Base de données par service (isolation)
- ✅ Communication via API REST
- ✅ Service Discovery (Eureka)
- ✅ Configuration centralisée (Config Server)
- ✅ Health checks (Actuator)

### Production
- ✅ Restart policies (`unless-stopped`)
- ✅ Resource limits (CPU, Memory)
- ✅ Health checks Docker
- ✅ Variables d'environnement (secrets)
- ✅ Logs structurés
- ✅ Monitoring (Prometheus-ready)

### Sécurité
- ✅ JWT Authentication
- ✅ Rate Limiting (Redis)
- ✅ CORS configuré
- ✅ Secrets externalisés (.env)
- ⚠️ TODO : HTTPS/TLS en production
- ⚠️ TODO : Secrets encryption (Spring Cloud Config)

---

## Endpoints Actuels

### API Gateway (Port 8080)

```bash
# Santé
GET http://localhost:8080/actuator/health

# Routes découvertes automatiquement via Eureka
GET http://localhost:8080/user-service/api/v1/auth/...
```

### User Service (Port 8081)

```bash
# Inscription
POST http://localhost:8081/api/v1/auth/inscription

# Connexion
POST http://localhost:8081/api/v1/auth/connexion

# Profil (JWT requis)
GET http://localhost:8081/api/v1/utilisateurs/{id}/profil
```

---

## Variables d'Environnement

Référence complète dans `.env.example`

### Critiques pour Production
```bash
POSTGRES_PASSWORD=<mot_de_passe_robuste>
REDIS_PASSWORD=<mot_de_passe_redis>
JWT_SECRET=<secret_64_caracteres_aleatoires>
DDL_AUTO=validate  # JAMAIS 'update' en prod
```

---

## Support & Contact

- **Repository** : https://github.com/AIMEROU-DIA/plateforme-teranga-biz
- **Config Repo** : https://github.com/AIMEROU-DIA/configurations-teranga-biz
- **Documentation** : Ce fichier + README.md dans chaque service

---

**Dernière mise à jour** : 2026-02-14  
**Version** : 1.0.0-MVP  
**Auteur** : Teranga Biz Platform Team
