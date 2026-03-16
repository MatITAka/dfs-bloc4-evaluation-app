# Architecture cible et choix de l'hébergement

**Auteur** : Matéo SLYEMI  
**Date** : 16 mars 2026  

---

## 1. Analyse des besoins de l'application

### 1.1 Composants identifiés

L'analyse de l'environnement de qualification et du code source révèle une application multi-composants :

| Composant | Technologie | Rôle | Port / Protocole |
|-----------|------------|------|-------------------|
| Application principale | Laravel 12 / PHP 8.4 | Cœur métier, interface web, API REST, webhook | HTTP/HTTPS (80/443) |
| Base de données relationnelle | MySQL | Données transactionnelles : utilisateurs, tickets, interventions, commentaires | 3306 |
| Base de données documentaire | MongoDB | Journalisation technique, événements applicatifs, traçabilité | 27017 |
| Cache applicatif | Redis | Cache disponible, mécanisme de performance (sessions, queues et cache utilisent MySQL par défaut) | 6379 |
| Microservice dashboard | Next.js (Node.js) | Tableau de bord dispatch, consomme l'API Laravel | 3000 |
| Webhook | `public/hooks.php` | Réception d'événements externes | via le serveur web |
| API tierce | Open-Meteo | Enrichissement météo des données d'intervention | HTTPS sortant |

### 1.2 Besoins fonctionnels

OpsTrack est une application de gestion d'interventions terrain (field service). Les besoins identifiés sont :

- **Disponibilité continue** : les techniciens terrain dépendent de l'application pour consulter et mettre à jour leurs tickets en temps réel. Une indisponibilité impacte directement l'activité opérationnelle.
- **Performance en lecture** : le tableau de bord principal et le dispatch-dashboard doivent afficher des données à jour rapidement. Par défaut, les sessions, queues et cache utilisent MySQL, mais Redis est présent dans la stack et peut être activé pour améliorer les performances. Redis peut influencer la cohérence des données affichées s'il est activé comme cache.
- **Traçabilité** : MongoDB stocke les événements techniques, ce qui est essentiel pour le diagnostic d'incidents et pour les obligations de conformité.
- **Intégrations** : le webhook reçoit des événements externes (systèmes tiers, IoT terrain), l'API publique Open-Meteo enrichit les fiches d'intervention.

### 1.3 Besoins non fonctionnels

- **Sécurité** : données clients et interventions soumises au RGPD, authentification par token API (`OPSTRACK_API_TOKEN`), basic auth sur le webhook.
- **Scalabilité** : en phase de croissance, le nombre de techniciens et le volume de tickets augmenteront — l'architecture doit pouvoir évoluer sans refonte.
- **Budget maîtrisé** : démarrage sur une infrastructure raisonnable avec un chemin de migration clair vers du scaling horizontal.
- **Conformité** : protection des données personnelles (RGPD), journalisation des accès et des traitements.

---

## 2. Architecture cible pour une application en croissance

### 2.1 Vue d'ensemble

L'architecture cible s'organise en 3 couches : exposition publique, traitement applicatif, et données. Chaque composant est isolé dans son propre service pour permettre un scaling indépendant.

```
                        ┌─────────────────────┐
                        │    Utilisateurs      │
                        │  (navigateurs, API)  │
                        └──────────┬──────────┘
                                   │
                        ┌──────────▼──────────┐
                        │   Load Balancer /    │
                        │   Reverse Proxy      │
                        │   (ALB + Apache2)    │
                        │   TLS termination    │
                        └───┬─────────────┬───┘
                            │             │
                ┌───────────▼───┐   ┌─────▼───────────┐
                │  Laravel 12   │   │  Next.js         │
                │  (PHP-FPM)    │   │  Dispatch         │
                │  App + API    │   │  Dashboard        │
                │  + Webhook    │   │  (Node.js)        │
                └───┬───┬───┬──┘   └──────┬────────────┘
                    │   │   │             │ (consomme API)
           ┌────────┘   │   └────────┐    │
           │            │            │    │
    ┌──────▼──┐  ┌──────▼──┐  ┌─────▼────▼─┐
    │  MySQL  │  │ MongoDB │  │   Redis     │
    │  (RDS)  │  │(DocumentDB│ │(ElastiCache)│
    │         │  │ ou EC2)  │  │             │
    └─────────┘  └─────────┘  └─────────────┘
```

### 2.2 Détail des couches

**Couche d'exposition (publique)** :
Un Application Load Balancer (ALB) AWS assure la terminaison TLS, distribue le trafic vers les instances applicatives et permet de séparer le routage : les requêtes vers `/dispatch` sont dirigées vers le service Next.js, le reste vers Laravel. C'est aussi le point d'entrée unique qui simplifie la gestion des certificats. En configuration mono-machine (phase actuelle), Apache2 assure ce rôle de reverse proxy avec `proxy_http` pour le microservice Next.js et `proxy_fcgi` pour PHP-FPM.

**Couche applicative (privée)** :
Laravel et Next.js tournent sur des instances EC2 dans un sous-réseau privé, accessibles uniquement via l'ALB. En phase initiale, une seule instance de chaque suffit. En croissance, on ajoute un Auto Scaling Group pour Laravel (le composant le plus sollicité). Le webhook `hooks.php` reste servi par la même instance Laravel — il n'a pas besoin d'un service dédié à ce stade.

**Couche données (privée, non exposée)** :
Les bases de données sont dans un sous-réseau privé sans accès internet. MySQL via Amazon RDS (gestion automatique des backups, failover, patching), Redis via ElastiCache, MongoDB via une instance EC2 dédiée (DocumentDB étant une alternative mais plus coûteuse pour ce volume).

### 2.3 Isolation réseau (VPC)

| Sous-réseau | Composants | Accès entrant | Accès sortant |
|-------------|-----------|---------------|---------------|
| Public | ALB | Internet (80, 443) | Vers sous-réseau applicatif |
| Applicatif (privé) | EC2 Laravel, EC2 Next.js | Depuis ALB uniquement | Vers données + Internet (NAT Gateway pour Open-Meteo, Composer, npm) |
| Données (privé) | RDS MySQL, ElastiCache Redis, EC2 MongoDB | Depuis sous-réseau applicatif uniquement | Aucun |

Les Security Groups appliquent le principe du moindre privilège : chaque composant n'accepte que le trafic strictement nécessaire sur les ports attendus.

---

## 3. Justification du fournisseur et des services retenus

### 3.1 Choix d'AWS

Le choix d'Amazon Web Services se justifie par plusieurs facteurs :

- **Cohérence avec l'existant** : les environnements de qualification et production fournis sont déjà sur AWS (EC2, région eu-west-3 Paris). Rester sur le même fournisseur évite une migration coûteuse et capitalise sur les compétences acquises.
- **Étendue des services managés** : RDS, ElastiCache, ALB, Certificate Manager (ACM) permettent de déléguer la maintenance des couches bas niveau (patching, backups, haute disponibilité) tout en conservant le contrôle.
- **Région eu-west-3 (Paris)** : hébergement des données en France, conformité RGPD simplifiée (pas de transfert hors UE).
- **Élasticité** : Auto Scaling Groups + ALB permettent d'absorber les pics de charge sans sur-provisionner.

### 3.2 Services retenus

| Besoin | Service AWS | Justification |
|--------|------------|---------------|
| Compute applicatif | EC2 (t3.medium) | Bon ratio coût/perf pour PHP-FPM + Node.js, burstable |
| Base relationnelle | RDS MySQL (db.t3.small) | Backups automatiques, Multi-AZ disponible, maintenance déléguée |
| Cache / Sessions | ElastiCache Redis (cache.t3.micro) | Latence <1ms, persistance optionnelle, pas de gestion serveur |
| NoSQL / Logs | EC2 dédiée (t3.small) + MongoDB | Plus économique que DocumentDB pour ce volume, migration possible ultérieurement |
| Load Balancer | ALB | Routage L7, terminaison TLS, health checks, intégration ACM |
| TLS / Certificats | ACM (Certificate Manager) | Certificats gratuits, renouvellement automatique, intégré à l'ALB |
| DNS | Route 53 | Gestion du domaine, alias vers ALB, basculement DNS possible |
| Stockage objets | S3 | Backups BDD, assets statiques, logs archivés |
| Réseau sortant | NAT Gateway | Permet aux instances privées d'accéder à Internet (Open-Meteo, mises à jour) |

---

## 4. Estimation du coût annuel

L'estimation ci-dessous correspond à la phase initiale (une instance par composant) en région eu-west-3, tarif On-Demand. Les prix sont indicatifs et basés sur la grille AWS de début 2026.

| Service | Dimensionnement | Coût mensuel estimé | Coût annuel estimé |
|---------|----------------|--------------------|--------------------|
| EC2 Laravel (t3.medium) | 1 instance, 24/7 | ~34 € | ~408 € |
| EC2 Next.js (t3.small) | 1 instance, 24/7 | ~17 € | ~204 € |
| EC2 MongoDB (t3.small) | 1 instance, 24/7, 50 Go EBS | ~22 € | ~264 € |
| RDS MySQL (db.t3.small) | Single-AZ, 20 Go | ~28 € | ~336 € |
| ElastiCache Redis (cache.t3.micro) | 1 nœud | ~13 € | ~156 € |
| ALB | 1, trafic modéré | ~20 € | ~240 € |
| NAT Gateway | 1, trafic faible | ~35 € | ~420 € |
| Route 53 | 1 zone hébergée | ~1 € | ~12 € |
| S3 (backups) | ~50 Go | ~2 € | ~24 € |
| ACM | Certificats TLS | Gratuit | Gratuit |
| **Total** | | **~172 €/mois** | **~2 064 €/an** |

**Pistes d'optimisation** :
- Passer en Reserved Instances (1 an) réduit le coût EC2/RDS de ~30%, ramenant le total sous les 1 500 €/an.
- Le NAT Gateway est le poste le plus cher pour un trafic faible — en phase initiale, un NAT Instance (t3.nano) serait 4x moins cher.
- Consolider Laravel + Next.js sur une seule EC2 (t3.medium) avec Nginx en reverse proxy interne est viable tant que la charge reste modérée.

---

## 5. Exigences qualitatives et réglementaires

### 5.1 Sécurité

- **Chiffrement en transit** : TLS 1.2+ obligatoire sur tous les flux publics (ALB + ACM). Communications internes en réseau privé VPC.
- **Chiffrement au repos** : activation du chiffrement AES-256 sur RDS (option native), EBS (volumes EC2), et S3 (SSE-S3).
- **Gestion des secrets** : les credentials BDD, tokens API (`OPSTRACK_API_TOKEN`, `WEBHOOK_BASIC_USER/PASSWORD`) sont stockés dans AWS Systems Manager Parameter Store (SecureString) ou dans un fichier `.env` non versionné avec permissions restrictives. Jamais dans le code source.
- **Accès SSH** : clé privée uniquement (`ubuntu.pem`), pas d'authentification par mot de passe, port 22 restreint aux IP d'administration.

### 5.2 Sauvegarde et reprise

- **MySQL** : backups automatiques RDS (rétention 7 jours) + snapshots manuels avant chaque déploiement.
- **MongoDB** : script `mongodump` quotidien via cron, stocké sur S3 avec lifecycle policy (30 jours de rétention).
- **Redis** : par défaut, Redis n'est pas utilisé pour les sessions ni les queues (drivers `database`). S'il est activé comme cache, ses données sont reconstituables — pas de backup nécessaire. En cas de migration vers Redis pour les sessions/queues, activer la persistance RDB.
- **RPO (Recovery Point Objective)** : 24h maximum (fréquence backup quotidienne).
- **RTO (Recovery Time Objective)** : 1h (restauration RDS automatique, redéploiement applicatif via CI/CD).

### 5.3 Supervision

- **Monitoring infrastructure** : Amazon CloudWatch pour les métriques EC2 (CPU, mémoire, disque), RDS (connexions, latence), ElastiCache (hit rate).
- **Monitoring applicatif** : endpoint `/api/health` de Laravel comme health check ALB. Logs applicatifs centralisés via MongoDB (déjà prévu dans l'architecture de l'application).
- **Alertes** : CloudWatch Alarms sur les seuils critiques (CPU > 80%, espace disque < 20%, health check échoué) → notification SNS (email).

### 5.4 Conformité RGPD et traçabilité

- **Localisation des données** : toute l'infrastructure est en eu-west-3 (Paris, France). Aucun transfert de données hors UE.
- **Données personnelles** : les entités utilisateurs et interventions dans MySQL contiennent potentiellement des données personnelles (noms, adresses de sites d'intervention). L'accès en BDD est limité à l'utilisateur applicatif (`user1`) avec des privilèges restreints (pas de `GRANT`, pas de `DROP`).
- **Journalisation** : MongoDB trace les événements applicatifs, ce qui permet d'auditer les actions effectuées dans l'application (qui a fait quoi, quand).
- **Droit à l'effacement** : l'architecture n'impose pas de contrainte technique bloquante — les données utilisateurs dans MySQL sont supprimables, les logs MongoDB peuvent être purgés par filtre utilisateur.
- **Sous-traitance** : AWS est un sous-traitant au sens du RGPD. Le Data Processing Agreement (DPA) AWS couvre les obligations de l'article 28.

### 5.5 Élasticité et évolution

L'architecture est conçue pour évoluer progressivement :

| Phase | Déclencheur | Action |
|-------|-------------|--------|
| Phase 1 (actuelle) | Lancement, <50 utilisateurs | 1 instance par composant, Single-AZ |
| Phase 2 | Croissance, 50-200 utilisateurs | Auto Scaling Group Laravel (2-4 instances), RDS Multi-AZ |
| Phase 3 | Charge élevée, >200 utilisateurs | Migration MongoDB vers DocumentDB, CDN CloudFront pour les assets statiques, séparation du webhook en Lambda si pics |

Chaque phase est un ajout incrémental, pas une refonte. L'isolation en sous-réseaux et la séparation des composants permettent de scaler chaque couche indépendamment.

---

## 6. Synthèse

L'architecture proposée répond aux 6 exigences minimales du sujet :

| Exigence | Réponse |
|----------|---------|
| Exposition publique des services | ALB avec terminaison TLS, routage vers Laravel et Next.js |
| Isolation entre composants | VPC 3 sous-réseaux, Security Groups par service |
| Élasticité / capacité d'évolution | Auto Scaling Group, chemin de migration en 3 phases |
| Gestion des données relationnelles et NoSQL | RDS MySQL + MongoDB sur EC2 dédiée |
| Sécurité, sauvegarde et supervision | TLS, chiffrement au repos, backups automatisés, CloudWatch |
| Conformité (RGPD, traçabilité) | Données en France (eu-west-3), journalisation MongoDB, DPA AWS |

Le budget annuel estimé de ~2 064 € (optimisable sous 1 500 € avec Reserved Instances) reste cohérent pour une PME exploitant une application métier critique.
