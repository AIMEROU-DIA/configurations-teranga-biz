# Platform Configuration Repository

Ce dépôt contient les configurations centralisées pour tous les microservices de la plateforme.

## Structure
- `application.yml` : Configuration globale partagée par tous les services
- `[service-name].yml` : Configuration spécifique à chaque microservice

## Mise à jour des configurations
1. Modifier le fichier de configuration souhaité
2. Commit et push sur la branche `main`
3. Chaque service recharge automatiquement la configuration via Spring Cloud Config

## Variables d'environnement sensibles
Les informations sensibles (mots de passe, tokens) sont gérées via:
- Variables d'environnement dans les conteneurs Docker
- Secrets Kubernetes en production
- Encryption Spring Cloud Config (optionnel)