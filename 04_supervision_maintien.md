# Supervision, journalisation, sauvegarde et maintenance corrective

**Auteur** : Matéo SLYEMI  
**Date** : 16 mars 2026  

---

## 1. Journalisation exploitable des services

### 1.1 Journalisation applicative (Laravel)

Laravel est configuré pour journaliser les erreurs dans `storage/logs/laravel.log` via le driver `stack` :

```ini
LOG_CHANNEL=stack
LOG_STACK=single
LOG_LEVEL=error
```

En production, seules les erreurs sont loguées (`LOG_LEVEL=error`) pour éviter de saturer le disque (1.7 Go libres).

### 1.2 Journalisation métier (MongoDB)

L'application utilise un service dédié (`EventLogService`) qui enregistre chaque action métier dans MongoDB (collection `app_events` de la base `opstrack_logs`) :

| Canal | Événement | Déclencheur |
|-------|-----------|-------------|
| `api` | `tickets.index` | Consultation de la liste des tickets |
| `api` | `ticket.created` | Création d'un ticket via l'API |
| `api` | `ticket.updated` | Modification d'un ticket (avec détail des changements) |
| `webhook` | `intervention.synced` | Réception d'un événement webhook externe |
| `integration` | `weather.synced` | Appel à l'API météo Open-Meteo |

Chaque entrée contient : `channel`, `event_type`, `severity`, `payload` (données contextuelles), `recorded_at` (horodatage).

Le service est résilient : si MongoDB est indisponible, l'erreur est capturée silencieusement et redirigée vers le log Laravel (`Log::warning('MongoDB logging unavailable.')`). L'application continue de fonctionner.

**Consultation des logs MongoDB** :

```bash
mongosh opstrack_logs --eval "db.app_events.find().sort({recorded_at: -1}).limit(5).pretty()"
```

### 1.3 Journalisation système

| Service | Fichier de log | Commande de consultation |
|---------|---------------|-------------------------|
| Apache2 | `/var/log/apache2/opstrack_access.log` et `opstrack_error.log` | `tail -f /var/log/apache2/opstrack_error.log` |
| PHP-FPM | `/var/log/php8.4-fpm.log` | `journalctl -u php8.4-fpm` |
| MySQL | `/var/log/mysql/error.log` | `tail -f /var/log/mysql/error.log` |
| MongoDB | `/var/log/mongodb/mongod.log` | `tail -f /var/log/mongodb/mongod.log` |
| Redis | `/var/log/redis/redis-server.log` | `tail -f /var/log/redis/redis-server.log` |
| Système | journald | `journalctl -xe` |

---

## 2. Outils et configurations d'audit

### 2.1 Audit des accès API

Le middleware `EnsureApiTokenIsValid` enregistre automatiquement la dernière utilisation de chaque token API (`last_used_at`). Cela permet de détecter les tokens inactifs ou compromis :

```bash
# Vérifier les tokens et leur dernière utilisation
php artisan tinker --execute="App\Models\ApiToken::all(['token','is_active','last_used_at'])->toJson(JSON_PRETTY_PRINT);"
```

### 2.2 Audit des connexions SSH

```bash
# Dernières connexions réussies
last -10

# Tentatives de connexion échouées
sudo grep "Failed" /var/log/auth.log | tail -10
```

### 2.3 Audit du pare-feu

```bash
sudo ufw status verbose
sudo grep -i "ufw" /var/log/syslog | tail -10
```

---

## 3. Sondes et alertes

### 3.1 Sondes de disponibilité

| Sonde | Commande | Seuil d'alerte |
|-------|---------|----------------|
| Application Laravel | `curl -sf https://eval-dfs-p-tpl-20261-05.it-students.fr/api/health` | HTTP ≠ 200 |
| Dispatch dashboard | `curl -sf https://eval-dfs-p-tpl-20261-05.it-students.fr/dispatch` | HTTP ≠ 200 |
| MySQL | `mysqladmin -u user1 -p0000 ping` | ≠ "alive" |
| Redis | `redis-cli ping` | ≠ "PONG" |
| MongoDB | `mongosh --eval "db.runCommand({ping:1})" --quiet` | échec |
| Espace disque | `df -h / \| awk 'NR==2{print $5}'` | > 85% |

### 3.2 Script de monitoring (cron)

```bash
#!/bin/bash
# /usr/local/bin/opstrack-monitor.sh

ALERT_EMAIL="m.slyemi@it-students.fr"
ERRORS=""

# Health check Laravel
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://eval-dfs-p-tpl-20261-05.it-students.fr/api/health)
[ "$HTTP_CODE" != "200" ] && ERRORS+="Laravel health check FAILED (HTTP $HTTP_CODE)\n"

# MySQL
mysqladmin -u user1 -p0000 ping 2>/dev/null | grep -q "alive" || ERRORS+="MySQL is DOWN\n"

# Redis
redis-cli ping 2>/dev/null | grep -q "PONG" || ERRORS+="Redis is DOWN\n"

# Espace disque
DISK_USAGE=$(df / | awk 'NR==2{gsub(/%/,""); print $5}')
[ "$DISK_USAGE" -gt 85 ] && ERRORS+="Disk usage critical: ${DISK_USAGE}%\n"

# Envoyer alerte si erreurs
if [ -n "$ERRORS" ]; then
    echo -e "OpsTrack ALERT:\n$ERRORS" | mail -s "[OpsTrack] Alert" "$ALERT_EMAIL" 2>/dev/null
    echo -e "[$(date)] ALERT:\n$ERRORS" >> /var/log/opstrack-monitor.log
fi
```

Activation via cron (toutes les 5 minutes) :

```bash
sudo chmod +x /usr/local/bin/opstrack-monitor.sh
echo "*/5 * * * * root /usr/local/bin/opstrack-monitor.sh" | sudo tee /etc/cron.d/opstrack-monitor
```

---

## 4. Stratégie de sauvegarde et restauration

### 4.1 Sauvegarde MySQL

```bash
#!/bin/bash
# /usr/local/bin/opstrack-backup.sh

BACKUP_DIR="/var/backups/opstrack"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p $BACKUP_DIR

# Dump MySQL
mysqldump -u root -p0000 opstrack > $BACKUP_DIR/mysql_opstrack_$DATE.sql
gzip $BACKUP_DIR/mysql_opstrack_$DATE.sql

# Dump MongoDB
mongodump --db opstrack_logs --out $BACKUP_DIR/mongo_$DATE/
tar czf $BACKUP_DIR/mongo_opstrack_logs_$DATE.tar.gz -C $BACKUP_DIR mongo_$DATE/
rm -rf $BACKUP_DIR/mongo_$DATE/

# Rotation : garder les 7 derniers jours
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "[$(date)] Backup completed" >> /var/log/opstrack-backup.log
```

Activation via cron (quotidien à 2h du matin) :

```bash
sudo chmod +x /usr/local/bin/opstrack-backup.sh
echo "0 2 * * * root /usr/local/bin/opstrack-backup.sh" | sudo tee /etc/cron.d/opstrack-backup
```

### 4.2 Restauration

```bash
# Restauration MySQL
gunzip < /var/backups/opstrack/mysql_opstrack_YYYYMMDD-HHMMSS.sql.gz | mysql -u root -p0000 opstrack

# Restauration MongoDB
tar xzf /var/backups/opstrack/mongo_opstrack_logs_YYYYMMDD-HHMMSS.tar.gz -C /tmp/
mongorestore --db opstrack_logs /tmp/mongo_YYYYMMDD-HHMMSS/opstrack_logs/
```

### 4.3 Objectifs de reprise

| Métrique | Valeur | Justification |
|----------|--------|---------------|
| RPO (Recovery Point Objective) | 24h | Backup quotidien |
| RTO (Recovery Time Objective) | 30 min | Restauration dump + redéploiement CI/CD |

---

## 5. Diagnostic et correction de bugs

### 5.1 Bug 1 — Webhook : statut du ticket ignoré

**Symptôme** : lorsqu'un événement externe est reçu via le webhook (`/hooks.php`), le ticket est toujours mis à jour avec le statut `scheduled`, quel que soit le statut envoyé dans le payload.

**Diagnostic** : dans `app/Http/Controllers/WebhookController.php`, ligne 48 :

```php
// AVANT (bug)
$ticket->update(['status' => 'scheduled']);
```

Le statut transmis dans `$payload['status']` est ignoré. Le ticket est forcé en `scheduled`.

**Correctif appliqué** :

```php
// APRÈS (correctif)
$ticket->update(['status' => $payload['status']]);
```

Le ticket reflète désormais le statut réellement transmis par le système externe.

**Vérification** : envoi d'un webhook avec un statut `in_progress` → le ticket passe bien en `in_progress`.

### 5.2 Bug 2 — Dashboard : KPIs non actualisés (cache non invalidé)

**Symptôme** : les compteurs du tableau de bord (tickets ouverts, critiques, créés aujourd'hui) ne se mettent pas à jour immédiatement après la modification d'un ticket. L'affichage peut être décalé de jusqu'à 30 minutes.

**Diagnostic** : dans `app/Http/Controllers/DashboardController.php`, ligne 26 :

```php
// Le cache dure 30 minutes et n'est jamais invalidé
'kpis' => Cache::remember('dashboard.kpis', now()->addMinutes(30), function (): array {
```

Le cache `dashboard.kpis` n'est invalidé nulle part dans le code. Toute modification de ticket (création, changement de statut, clôture) n'est visible dans les KPIs qu'après expiration du cache.

**Correctif appliqué** : ajout d'un observer dans le modèle `Ticket` (`app/Models/Ticket.php`) :

```php
protected static function booted(): void
{
    static::saved(function () {
        \Illuminate\Support\Facades\Cache::forget('dashboard.kpis');
    });

    static::deleted(function () {
        \Illuminate\Support\Facades\Cache::forget('dashboard.kpis');
    });
}
```

Le cache est maintenant invalidé à chaque sauvegarde ou suppression de ticket. Le prochain affichage du dashboard recalcule les KPIs en temps réel.

---

## 6. Diagnostic et correction de faille de sécurité

### 6.1 Faille — Injection SQL dans la recherche de tickets

**Symptôme** : le paramètre `search` de l'endpoint `GET /api/v1/tickets?search=...` est injectable.

**Diagnostic** : dans `app/Http/Controllers/Api/TicketController.php`, ligne 26 :

```php
// AVANT (vulnérable)
$query->where('title', 'like', "%{$search}%")
    ->orWhereRaw("reference like '%{$search}%'");
```

La première clause (`where`) utilise le query builder de Laravel avec binding paramétré — elle est sécurisée. Mais la seconde clause (`orWhereRaw`) injecte la variable `$search` **directement dans la chaîne SQL** sans échappement. Un attaquant peut exploiter cette faille pour extraire des données, modifier des enregistrements ou compromettre la base.

**Exemple d'exploitation** :

```
GET /api/v1/tickets?search=' OR 1=1 --
```

Cette requête retournerait tous les tickets de la base, en contournant les filtres.

**Correctif appliqué** :

```php
// APRÈS (sécurisé — binding paramétré via le query builder)
$query->where('title', 'like', "%{$search}%")
    ->orWhere('reference', 'like', '%' . $search . '%');
```

Le query builder Laravel échappe automatiquement les paramètres. L'injection SQL n'est plus possible.

**Vérification** : la requête `?search=' OR 1=1 --` ne retourne plus aucun résultat (comportement attendu).

---

## 7. Correspondance avec les attendus

| Attendu du sujet | Section | Statut |
|-----------------|---------|--------|
| Journalisation exploitable des services | §1 (Laravel logs, MongoDB EventLog, logs système) | ✅ |
| Outils ou configurations d'audit | §2 (audit API tokens, SSH, UFW) | ✅ |
| Sondes et alertes pertinentes | §3 (script monitoring cron, health checks) | ✅ |
| Stratégie de sauvegarde et restauration | §4 (MySQL dump, MongoDB dump, cron quotidien, RPO/RTO) | ✅ |
| Identification d'un bug + correctif | §5 (webhook statut ignoré + cache dashboard non invalidé) | ✅ |
| Identification d'une faille + correctif | §6 (SQL injection dans la recherche de tickets) | ✅ |
