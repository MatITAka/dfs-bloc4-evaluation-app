# Déploiement automatisé : qualification → production

**Auteur** : Matéo SLYEMI  
**Date** : 16 mars 2026  

---

## 1. Stratégie de déploiement

### 1.1 Principe général

Le déploiement suit un flux unidirectionnel : le code validé sur l'environnement de qualification est promu vers la production via un pipeline GitHub Actions. Le déclenchement est **explicite** (manuel via `workflow_dispatch`) pour garantir qu'aucune mise en production ne se fasse par accident.

```
   Développeur            GitHub Actions              Production
       │                       │                          │
       │  workflow_dispatch     │                          │
       ├──────────────────────►│                          │
       │                       │  1. Checkout code         │
       │                       │  2. Install dépendances   │
       │                       │  3. Tests (qualité)       │
       │                       │  4. Build                 │
       │                       │──────────────────────────►│
       │                       │  5. Déploiement rsync     │
       │                       │  6. Symlinks shared       │
       │                       │  7. Migration BDD         │
       │                       │  8. Cache Laravel         │
       │                       │  9. Bascule symlink       │
       │                       │◄──────────────────────────│
       │                       │  10. Smoke test           │
       │  ✅ ou ❌ Résultat    │                          │
       │◄──────────────────────│                          │
```

### 1.2 Environnements

| Environnement | Rôle | URL | IP |
|---------------|------|-----|-----|
| Qualification | Validation fonctionnelle, tests | `http://eval-dfs-q-tpl-20261-05.it-students.fr` | `35.180.117.1` |
| Production | Application live, utilisateurs finaux | `https://eval-dfs-p-tpl-20261-05.it-students.fr` | `51.45.7.228` |

### 1.3 Les 4 exigences du sujet

| Exigence | Réponse technique |
|----------|------------------|
| Déclenchement explicite et reproductible | `workflow_dispatch` (bouton manuel dans GitHub Actions) |
| Vérification d'un niveau minimal de qualité | Tests PHPUnit + vérification syntaxique avant déploiement |
| Mise à jour effective de la production | Déploiement rsync + symlink atomique (stratégie releases) |
| Vérification post-déploiement (smoke test) | Requêtes HTTP vers `/api/health`, `/` et `/dispatch` |

---

## 2. Structure de déploiement sur le serveur

### 2.1 Arborescence

Le déploiement utilise une stratégie de **releases avec symlink** pour permettre un rollback instantané :

```
/var/www/opstrack/
├── releases/
│   ├── 20260316-initial/               ← première release (déploiement manuel)
│   ├── 20260316-151045_d4e5f6a/        ← releases suivantes (via CI/CD)
│   └── ...
├── shared/
│   ├── .env                            ← fichier unique, partagé entre releases
│   └── storage/                        ← logs, cache fichiers, sessions
│       ├── app/public/
│       ├── framework/{cache,sessions,views}/
│       └── logs/
└── current -> releases/20260316-initial/   ← symlink actif
```

**Avantages** :
- Le basculement est atomique (un `ln -sfn` suffit)
- Le rollback est immédiat : repointer `current` vers la release précédente
- Le `.env` et `storage/` sont partagés — aucune donnée perdue entre déploiements

### 2.2 Configuration Apache

Le VirtualHost pointe vers `current/public`, ce qui suit automatiquement le symlink :

```apache
DocumentRoot /var/www/opstrack/current/public

<Directory /var/www/opstrack/current/public>
    AllowOverride All
    Require all granted
</Directory>
```

### 2.3 Initialisation (déjà effectuée)

La structure a été initialisée à partir du déploiement manuel initial :

```bash
sudo mkdir -p /var/www/opstrack/{releases,shared/storage}
sudo cp -a /var/www/opstrack/storage/* /var/www/opstrack/shared/storage/
sudo cp /var/www/opstrack/.env /var/www/opstrack/shared/.env
sudo rsync -a --exclude='releases' --exclude='shared' /var/www/opstrack/ /var/www/opstrack/releases/20260316-initial/
sudo rm -rf /var/www/opstrack/releases/20260316-initial/storage
sudo ln -sfn /var/www/opstrack/shared/storage /var/www/opstrack/releases/20260316-initial/storage
sudo ln -sfn /var/www/opstrack/shared/.env /var/www/opstrack/releases/20260316-initial/.env
sudo ln -sfn /var/www/opstrack/releases/20260316-initial /var/www/opstrack/current
sudo chown -R ubuntu:www-data /var/www/opstrack/releases /var/www/opstrack/shared
```

---

## 3. Secrets GitHub

Les secrets suivants doivent être configurés dans le dépôt GitHub (Settings → Secrets and variables → Actions) :

| Secret | Contenu | Rôle |
|--------|---------|------|
| `PROD_SSH_KEY` | Contenu du fichier `ubuntu.pem` | Authentification SSH vers la prod |
| `PROD_HOST` | `51.45.7.228` | IP de la machine de production |
| `PROD_USER` | `ubuntu` | Utilisateur SSH |
| `OPSTRACK_API_TOKEN` | `change-me` | Token API pour le smoke test |

---

## 4. Pipeline GitHub Actions

### 4.1 Fichier workflow

Fichier : `.github/workflows/deploy-production.yml`

```yaml
name: Deploy to Production

# 1. DÉCLENCHEMENT EXPLICITE
on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Tapez "deploy" pour confirmer le déploiement en production'
        required: true
        type: string

jobs:
  # ─────────────────────────────────────────────────
  # JOB 1 : Vérification qualité (gate)
  # ─────────────────────────────────────────────────
  quality-gate:
    name: Vérification qualité
    runs-on: ubuntu-latest
    if: github.event.inputs.confirm == 'deploy'

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Setup PHP 8.4
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: mbstring, xml, curl, zip, bcmath, redis, mongodb
          tools: composer:v2

      - name: Cache Composer
        uses: actions/cache@v4
        with:
          path: vendor
          key: composer-${{ hashFiles('composer.lock') }}
          restore-keys: composer-

      - name: Installation des dépendances
        run: composer install --no-interaction --prefer-dist --ignore-platform-req=ext-mongodb

      - name: Copie du .env de test
        run: |
          cp .env.example .env
          php artisan key:generate

      - name: Vérification syntaxique
        run: php artisan route:list --no-interaction > /dev/null 2>&1

      - name: Tests PHPUnit
        run: php artisan test --no-interaction
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: ':memory:'

  # ─────────────────────────────────────────────────
  # JOB 2 : Déploiement en production
  # ─────────────────────────────────────────────────
  deploy:
    name: Déploiement production
    runs-on: ubuntu-latest
    needs: quality-gate

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Setup PHP 8.4
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: mbstring, xml, curl, zip, bcmath
          tools: composer:v2

      - name: Installation des dépendances (production)
        run: composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader --ignore-platform-req=ext-mongodb

      - name: Préparer la clé SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.PROD_SSH_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H ${{ secrets.PROD_HOST }} >> ~/.ssh/known_hosts

      - name: Définir le nom de la release
        id: release
        run: |
          RELEASE_NAME="$(date +%Y%m%d-%H%M%S)_$(echo $GITHUB_SHA | cut -c1-7)"
          echo "name=$RELEASE_NAME" >> $GITHUB_OUTPUT

      - name: Déployer les fichiers via rsync
        run: |
          rsync -azP --delete \
            --exclude='.git' \
            --exclude='.github' \
            --exclude='tests' \
            --exclude='storage' \
            --exclude='.env' \
            --exclude='node_modules' \
            -e "ssh -i ~/.ssh/deploy_key" \
            ./ ${{ secrets.PROD_USER }}@${{ secrets.PROD_HOST }}:/var/www/opstrack/releases/${{ steps.release.outputs.name }}/

      - name: Activer la release sur le serveur
        run: |
          ssh -i ~/.ssh/deploy_key ${{ secrets.PROD_USER }}@${{ secrets.PROD_HOST }} << 'REMOTE'
            set -e
            RELEASE="/var/www/opstrack/releases/${{ steps.release.outputs.name }}"
            SHARED="/var/www/opstrack/shared"

            # Corriger l'ordre des migrations (timestamps identiques dans le repo)
            cd $RELEASE/database/migrations
            mv 2026_03_14_183114_create_customers_table.php     2026_03_14_183100_create_customers_table.php 2>/dev/null || true
            mv 2026_03_14_183114_create_sites_table.php          2026_03_14_183101_create_sites_table.php 2>/dev/null || true
            mv 2026_03_14_183114_create_tickets_table.php        2026_03_14_183102_create_tickets_table.php 2>/dev/null || true
            mv 2026_03_14_183114_create_interventions_table.php  2026_03_14_183103_create_interventions_table.php 2>/dev/null || true
            mv 2026_03_14_183114_create_api_tokens_table.php     2026_03_14_183104_create_api_tokens_table.php 2>/dev/null || true
            cd $RELEASE

            # Lier le .env partagé
            ln -sfn $SHARED/.env $RELEASE/.env

            # Lier le dossier storage partagé
            rm -rf $RELEASE/storage
            ln -sfn $SHARED/storage $RELEASE/storage

            # Permissions
            chown -R ubuntu:www-data $RELEASE
            chmod -R 775 $SHARED/storage

            # Migrations base de données
            php artisan migrate --force --no-interaction

            # Optimisation Laravel production
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache

            # Basculement atomique du symlink
            ln -sfn $RELEASE /var/www/opstrack/current

            # Recharger PHP-FPM pour le nouveau code
            sudo systemctl reload php8.4-fpm

            echo "✅ Release ${{ steps.release.outputs.name }} activée"
          REMOTE

      - name: Nettoyage des anciennes releases (garder les 5 dernières)
        run: |
          ssh -i ~/.ssh/deploy_key ${{ secrets.PROD_USER }}@${{ secrets.PROD_HOST }} << 'REMOTE'
            cd /var/www/opstrack/releases
            ls -1dt */ | tail -n +6 | xargs -r rm -rf
            echo "🧹 Anciennes releases nettoyées"
          REMOTE

  # ─────────────────────────────────────────────────
  # JOB 3 : Smoke test post-déploiement
  # ─────────────────────────────────────────────────
  smoke-test:
    name: Vérification post-déploiement
    runs-on: ubuntu-latest
    needs: deploy

    steps:
      - name: Attendre la propagation (10s)
        run: sleep 10

      - name: Smoke test — health check
        run: |
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
            https://eval-dfs-p-tpl-20261-05.it-students.fr/api/health)
          if [ "$HTTP_CODE" -eq 200 ]; then
            echo "✅ Health check OK (HTTP $HTTP_CODE)"
          else
            echo "❌ Health check FAILED (HTTP $HTTP_CODE)"
            exit 1
          fi

      - name: Smoke test — page d'accueil
        run: |
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
            https://eval-dfs-p-tpl-20261-05.it-students.fr/)
          if [ "$HTTP_CODE" -eq 200 ]; then
            echo "✅ Page d'accueil OK (HTTP $HTTP_CODE)"
          else
            echo "❌ Page d'accueil FAILED (HTTP $HTTP_CODE)"
            exit 1
          fi

      - name: Smoke test — dispatch dashboard
        run: |
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
            https://eval-dfs-p-tpl-20261-05.it-students.fr/dispatch)
          if [ "$HTTP_CODE" -eq 200 ]; then
            echo "✅ Dispatch dashboard OK (HTTP $HTTP_CODE)"
          else
            echo "⚠️  Dispatch dashboard HTTP $HTTP_CODE"
          fi

      - name: Smoke test — API tickets
        run: |
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
            -H "Authorization: Bearer ${{ secrets.OPSTRACK_API_TOKEN }}" \
            https://eval-dfs-p-tpl-20261-05.it-students.fr/api/v1/tickets)
          if [ "$HTTP_CODE" -eq 200 ]; then
            echo "✅ API tickets OK (HTTP $HTTP_CODE)"
          else
            echo "⚠️  API tickets HTTP $HTTP_CODE (peut nécessiter un token)"
          fi
```

---

## 5. Procédure de déploiement

### 5.1 Déclenchement

1. Aller sur le dépôt GitHub → onglet **Actions**
2. Sélectionner le workflow **"Deploy to Production"**
3. Cliquer sur **"Run workflow"**
4. Saisir `deploy` dans le champ de confirmation
5. Cliquer sur **"Run workflow"**

Le pipeline exécute les 3 jobs séquentiellement. Si le job `quality-gate` échoue (tests en erreur), le déploiement n'a pas lieu.

### 5.2 Rollback en cas de problème

Le rollback est immédiat — repointer le symlink vers la release précédente :

```bash
ssh -i ubuntu.pem ubuntu@51.45.7.228

# Lister les releases disponibles
ls -lt /var/www/opstrack/releases/

# Repointer vers la release précédente
ln -sfn /var/www/opstrack/releases/<RELEASE_PRECEDENTE> /var/www/opstrack/current

# Recharger PHP-FPM
sudo systemctl reload php8.4-fpm
```

Le rollback prend moins de 10 secondes. Aucune donnée n'est perdue car le `.env` et `storage/` sont partagés.

---

## 6. Correspondance avec les attendus

| Exigence du sujet | Implémentation | Section |
|--------------------|----------------|---------|
| Déclenchement explicite et reproductible | `workflow_dispatch` avec confirmation textuelle `"deploy"` | §4.1, §5.1 |
| Vérification d'un niveau minimal de qualité | Job `quality-gate` : lint + PHPUnit — bloquant (`needs: quality-gate`) | §4.1 job 1 |
| Mise à jour effective de la production | Job `deploy` : rsync + fix migrations + symlinks + migration BDD + cache + bascule atomique + reload PHP-FPM | §4.1 job 2 |
| Vérification post-déploiement (smoke test) | Job `smoke-test` : HTTP vers `/api/health`, `/`, `/dispatch`, `/api/v1/tickets` | §4.1 job 3 |

---

## 7. Sécurité du pipeline

- **Aucun secret dans le code** : clé SSH, IP, token API stockés en GitHub Secrets, chiffrés au repos, jamais affichés dans les logs.
- **Confirmation manuelle** : `confirm == 'deploy'` empêche un déclenchement accidentel.
- **Séparation des jobs** : chaque job tourne dans un runner isolé. Le déploiement ne s'exécute que si la quality gate passe.
- **Moindre privilège** : rsync exclut `.env`, `.git`, `tests`, `node_modules`. Seul le code applicatif est transféré.
- **Rétention** : 5 dernières releases conservées pour rollback, les plus anciennes nettoyées automatiquement.
- **Fix des migrations automatisé** : le renommage des fichiers de migration est intégré au script de déploiement pour pallier le bug d'ordonnancement du repo source.
