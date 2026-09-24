
# Aufbau einer sicheren MariaDB-Datenbankinfrastruktur unter Debian

## 1. Présentation du projet

### 1.1 Contexte

Ce projet est réalisé dans le cadre de la modularbeit consacrée à l'implémentation d'une base de données. Il porte sur l'infrastructure nécessaire au fonctionnement d'un serveur de base de données. Le développement d'une application complète ne fait pas partie du périmètre.

Une petite base relationnelle sert à vérifier l'installation, l'administration, les autorisations, la sécurité réseau, les sauvegardes et la restauration.

### 1.2 Périmètre

Le projet comprend :

- une machine virtuelle Debian reproductible ;
- un serveur MariaDB ;
- une interface d'administration phpMyAdmin ;
- une base de données de démonstration ;
- plusieurs comptes avec des droits distincts ;
- un pare-feu nftables ;
- une sauvegarde automatisée ;
- un test de restauration ;
- des tests techniques documentés.

## 2. Objectifs

- déployer une VM Debian avec Vagrant et VirtualBox ;
- installer et exploiter MariaDB 10.11 ;
- administrer la base avec phpMyAdmin ;
- appliquer le principe du moindre privilège ;
- limiter MariaDB à l'interface locale ;
- filtrer les connexions avec nftables ;
- sauvegarder automatiquement la base ;
- vérifier l'intégrité des archives ;
- restaurer les données dans une base séparée ;
- garantir la persistance après redémarrage ;
- documenter les opérations et leurs résultats.

## 3. Outils utilisés

| Outil | Utilisation |
|---|---|
| Windows | Système hôte |
| VirtualBox | Exécution de la VM |
| Vagrant | Création et gestion reproductible de la VM |
| Debian 12 Bookworm | Système d'exploitation du serveur |
| MariaDB 10.11 | Système de gestion de base de données relationnelle |
| Apache HTTP Server | Serveur Web de phpMyAdmin |
| PHP | Exécution de phpMyAdmin |
| phpMyAdmin | Administration graphique de MariaDB |
| nftables | Filtrage des connexions réseau |
| OpenSSH | Administration distante de Debian |
| SQL | Création des tables, comptes et permissions |
| `mariadb-dump` | Export logique de la base |
| gzip | Compression et contrôle des sauvegardes |
| cron | Planification des sauvegardes |
| systemd | Gestion des services |

Ces outils correspondent aux thèmes du cours : Debian, VirtualBox, Vagrant, modèle relationnel, MariaDB, LAMP, phpMyAdmin, SQL, sécurité, utilisateurs et intégrité des données.

## 4. Architecture

| Élément | Configuration |
|---|---|
| Machine hôte | Windows |
| Hyperviseur | VirtualBox |
| Gestion de la VM | Vagrant |
| Machine virtuelle | Debian 12 |
| Nom d'hôte | `db-server` |
| Interface NAT | `eth0`, adresse `10.0.2.15/24` |
| Interface privée | `eth1`, adresse `192.168.56.20/24` |
| Mémoire vive | 2 Go |
| Processeurs virtuels | 2 |
| Accès SSH | `vagrant ssh`, port 22/TCP |
| Interface Web | Apache et phpMyAdmin, port 80/TCP |
| Redirection Vagrant | `127.0.0.1:8080` vers le port 80 de la VM |
| Base principale | `infrastructure_db` |
| Base de restauration | `infrastructure_restore_test` |
| MariaDB | `127.0.0.1:3306` uniquement |
| Fuseau horaire | `Europe/Zurich` |

La VM utilise le NAT pour accéder à Internet et un réseau host-only pour communiquer avec Windows. Le port MariaDB n'est pas publié sur l'hôte.

## 5. Création de la machine virtuelle

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.hostname = "db-server"
  config.vm.network "private_network", ip: "192.168.56.20"

  config.vm.network "forwarded_port",
                    guest: 80,
                    host: 8080,
                    host_ip: "127.0.0.1",
                    auto_correct: true

  config.vm.provider "virtualbox" do |vb|
    vb.name = "database-server"
    vb.memory = 2048
    vb.cpus = 2
  end

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y curl vim openssh-server
    systemctl enable --now ssh
  SHELL
end
```

Commandes principales :

```powershell
vagrant up
vagrant ssh
vagrant halt
vagrant status
```

`vagrant halt` conserve la VM et ses données. `vagrant destroy` les supprimerait.

## 6. Installation et sécurisation de MariaDB

```bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb
sudo mariadb-secure-installation
```

Mesures appliquées :

- suppression des comptes anonymes ;
- interdiction de la connexion distante de `root` ;
- suppression de la base de test par défaut ;
- administration locale avec `sudo mariadb` ;
- comptes individuels pour administrer et utiliser la base.

### 6.1 Restriction de l'écoute réseau

Le fichier `/etc/mysql/mariadb.conf.d/99-local-bind.cnf` contient :

```ini
[mysqld]
bind-address = 127.0.0.1
```

La socket réseau gérée par systemd a été désactivée, puis MariaDB redémarré :

```bash
sudo systemctl disable --now mariadb.socket
sudo systemctl restart mariadb
sudo ss -lntp | grep 3306
```

Résultat validé : `127.0.0.1:3306`. MariaDB n'est donc pas directement joignable depuis Windows.

## 7. Apache, PHP et phpMyAdmin

```bash
sudo apt install -y apache2 php libapache2-mod-php php-mysql
sudo systemctl enable --now apache2
sudo apt install -y phpmyadmin
sudo a2enconf phpmyadmin
sudo systemctl reload apache2
```

Adresse depuis Windows :

```text
http://localhost:8080/phpmyadmin
```

Contrôle Apache :

```bash
sudo apache2ctl configtest
```

Résultat attendu : `Syntax OK`.

## 8. Base de données de démonstration

```sql
CREATE DATABASE infrastructure_db
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

### 8.1 Table de contrôle des services

```sql
CREATE TABLE server_status (
    status_id INT AUTO_INCREMENT PRIMARY KEY,
    service_name VARCHAR(100) NOT NULL,
    service_status VARCHAR(20) NOT NULL,
    checked_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 8.2 Table de test des permissions

```sql
CREATE TABLE permission_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(100) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

La ligne `Créé par db_admin` est conservée pour la démonstration.

## 9. Utilisateurs et privilèges

| Compte | Fonction | Droits sur `infrastructure_db` |
|---|---|---|
| `db_admin` | Administration de la base | Tous les privilèges sur la base |
| `db_writer` | Lecture et modification | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| `db_reader` | Consultation | `SELECT` |

Création depuis `sudo mariadb` :

```sql
CREATE USER IF NOT EXISTS 'db_admin'@'localhost'
IDENTIFIED BY '<mot_de_passe_admin>';

CREATE USER IF NOT EXISTS 'db_writer'@'localhost'
IDENTIFIED BY '<mot_de_passe_writer>';

CREATE USER IF NOT EXISTS 'db_reader'@'localhost'
IDENTIFIED BY '<mot_de_passe_reader>';

GRANT ALL PRIVILEGES ON infrastructure_db.*
TO 'db_admin'@'localhost';

GRANT SELECT, INSERT, UPDATE, DELETE ON infrastructure_db.*
TO 'db_writer'@'localhost';

GRANT SELECT ON infrastructure_db.*
TO 'db_reader'@'localhost';
```

Les mots de passe réels ne sont pas conservés dans la documentation.

Contrôle :

```sql
SHOW GRANTS FOR 'db_admin'@'localhost';
SHOW GRANTS FOR 'db_writer'@'localhost';
SHOW GRANTS FOR 'db_reader'@'localhost';
```

### 9.1 Résultats des tests

| Opération | `db_admin` | `db_writer` | `db_reader` |
|---|:---:|:---:|:---:|
| `SELECT` | Autorisé | Autorisé | Autorisé |
| `INSERT` | Autorisé | Autorisé | Refusé |
| `UPDATE` | Autorisé | Autorisé | Refusé |
| `DELETE` | Autorisé | Autorisé | Refusé |
| `CREATE TABLE` | Autorisé | Refusé | Refusé |
| Gestion globale des comptes | Refusée | Refusée | Refusée |

Les refus de `db_reader` et `db_writer` ont produit l'erreur MariaDB `1142`. La création initiale des comptes avec `db_admin` a également été refusée, car `CREATE USER` est un privilège global. Ces résultats confirment le principe du moindre privilège.

## 10. Pare-feu nftables

```bash
sudo apt install -y nftables
sudo cp /etc/nftables.conf /etc/nftables.conf.backup
sudo systemctl enable nftables
```

Configuration de `/etc/nftables.conf` :

```nft
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0;
        policy drop;

        iifname "lo" accept
        ct state established,related accept
        tcp dport 22 accept
        tcp dport 80 accept
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept
        udp sport 67 udp dport 68 accept
    }

    chain forward {
        type filter hook forward priority 0;
        policy drop;
    }

    chain output {
        type filter hook output priority 0;
        policy accept;
    }
}
```

Validation :

```bash
sudo nft -c -f /etc/nftables.conf
sudo systemctl restart nftables
sudo nft list ruleset
```

| Service | Port | Fonction |
|---|---:|---|
| SSH | 22/TCP | Administration de Debian |
| HTTP | 80/TCP | Apache et phpMyAdmin |
| ICMP/ICMPv6 | Sans port | Tests et contrôle réseau |
| DHCP | 67/UDP vers 68/UDP | Configuration réseau |

Le port `3306/TCP` n'est pas autorisé en entrée.

Test depuis Windows :

```powershell
Test-NetConnection 192.168.56.20 -Port 3306
```

Résultat observé : `PingSucceeded: True` et `TcpTestSucceeded: False`. La VM est joignable, mais MariaDB n'est pas accessible directement.

## 11. Sauvegarde MariaDB

Répertoire protégé :

```bash
sudo mkdir -p /var/backups/mariadb
sudo chown root:root /var/backups/mariadb
sudo chmod 700 /var/backups/mariadb
```

Script `/usr/local/sbin/backup-infrastructure-db` :

```bash
#!/bin/bash

set -euo pipefail
umask 077

BACKUP_DIR="/var/backups/mariadb"
DATABASE="infrastructure_db"
TIMESTAMP="$(date +'%Y-%m-%d_%H-%M-%S')"
TEMP_FILE="${BACKUP_DIR}/.${DATABASE}_${TIMESTAMP}.sql.gz.tmp"
BACKUP_FILE="${BACKUP_DIR}/${DATABASE}_${TIMESTAMP}.sql.gz"

mkdir -p "${BACKUP_DIR}"

mariadb-dump \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    "${DATABASE}" \
    | gzip > "${TEMP_FILE}"

gzip --test "${TEMP_FILE}"
mv "${TEMP_FILE}" "${BACKUP_FILE}"

find "${BACKUP_DIR}" \
    -type f \
    -name "${DATABASE}_*.sql.gz" \
    -mtime +7 \
    -delete

echo "Sauvegarde créée : ${BACKUP_FILE}"
```

Protection et exécution :

```bash
sudo chown root:root /usr/local/sbin/backup-infrastructure-db
sudo chmod 700 /usr/local/sbin/backup-infrastructure-db
sudo bash -n /usr/local/sbin/backup-infrastructure-db
sudo /usr/local/sbin/backup-infrastructure-db
```

Deux archives valides ont été créées durant les tests.

### 11.1 Vérification des archives

```bash
sudo find /var/backups/mariadb \
    -type f \
    -name 'infrastructure_db_*.sql.gz' \
    -exec gzip --test {} \;

sudo find /var/backups/mariadb \
    -type f \
    -name 'infrastructure_db_*.sql.gz' \
    -exec zgrep -H 'permission_test' {} \;
```

Aucune erreur gzip n'a été retournée. Les instructions `CREATE TABLE` et `INSERT INTO` de `permission_test` étaient présentes.

## 12. Automatisation avec cron

```bash
sudo apt install -y cron
sudo systemctl enable --now cron
```

Fichier `/etc/cron.d/mariadb-backup` :

```cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

0 2 * * * root /usr/local/sbin/backup-infrastructure-db >> /var/log/mariadb-backup.log 2>&1
```

```bash
sudo chown root:root /etc/cron.d/mariadb-backup
sudo chmod 644 /etc/cron.d/mariadb-backup
sudo systemctl restart cron
```

La sauvegarde est planifiée chaque jour à 02:00.

### 12.1 Fuseau horaire

```bash
sudo timedatectl set-timezone Europe/Zurich
```

Résultat validé : fuseau `Europe/Zurich`, synchronisation NTP active. Cron s'exécutera à 02:00 heure suisse et suivra les changements d'heure.

L'exécution manuelle est validée. La création automatique d'une archive par cron reste à confirmer dans `/var/log/mariadb-backup.log` après une exécution planifiée.

## 13. Test de restauration

Une base séparée a été créée pour protéger l'original :

```sql
CREATE DATABASE infrastructure_restore_test
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Restauration :

```bash
sudo sh -c \
'gzip -dc /var/backups/mariadb/infrastructure_db_2026-09-24_11-29-19.sql.gz |
mariadb infrastructure_restore_test'
```

Contrôles :

```bash
sudo mariadb infrastructure_restore_test -e "SHOW TABLES;"
sudo mariadb infrastructure_restore_test -e \
"SELECT * FROM permission_test;"
```

Résultats :

- `permission_test` restaurée ;
- `server_status` restaurée ;
- ligne `Créé par db_admin` restaurée ;
- base originale inchangée.

La sauvegarde permet donc de restaurer la structure et les données.

## 14. Validation après redémarrage

La VM a été arrêtée avec `vagrant halt`, puis redémarrée avec `vagrant up`.

```bash
sudo systemctl is-active ssh apache2 mariadb nftables cron
sudo systemctl is-enabled ssh apache2 mariadb nftables cron
```

Les cinq services étaient `active` et `enabled`.

Après redémarrage, les éléments suivants étaient toujours présents :

- `infrastructure_db` ;
- `infrastructure_restore_test` ;
- `permission_test` et sa donnée ;
- écoute MariaDB limitée à `127.0.0.1:3306` ;
- règles nftables et politiques `drop` ;
- services Apache, SSH, MariaDB, nftables et cron.

La persistance est validée.

## 15. Tableau final des tests

| Test | Résultat observé | État |
|---|---|:---:|
| Nom d'hôte `db-server` | Conforme | Validé |
| Adresse privée `192.168.56.20` | Conforme | Validé |
| Accès Internet et DNS | 0 % de perte vers `debian.org` | Validé |
| SSH | `active` et `enabled` | Validé |
| Apache | `active` et `enabled` | Validé |
| MariaDB | `active` et `enabled` | Validé |
| nftables | `active` et `enabled` | Validé |
| cron | `active` et `enabled` | Validé |
| phpMyAdmin | Connexion réussie | Validé |
| Écoute MariaDB | `127.0.0.1:3306` | Validé |
| Port 3306 depuis Windows | `TcpTestSucceeded: False` | Validé |
| Lecture avec `db_reader` | Autorisée | Validé |
| Écriture avec `db_reader` | Erreur 1142 | Validé |
| CRUD avec `db_writer` | Autorisé | Validé |
| `CREATE TABLE` avec `db_writer` | Erreur 1142 | Validé |
| Administration avec `db_admin` | Conforme | Validé |
| Sauvegarde manuelle | Deux archives créées | Validé |
| Intégrité gzip | Aucune erreur | Validé |
| Contenu SQL | Tables et données présentes | Validé |
| Restauration | Tables et données récupérées | Validé |
| Persistance après redémarrage | Services et données conservés | Validé |
| Exécution automatique cron | À observer à 02:00 | En attente |

## 16. Compétences mises en évidence

- virtualisation d'un serveur Linux ;
- utilisation de Vagrant et VirtualBox ;
- configuration réseau NAT et host-only ;
- installation et administration de MariaDB ;
- utilisation de SQL et d'une base relationnelle ;
- administration avec phpMyAdmin ;
- création de comptes et gestion des privilèges ;
- application du principe du moindre privilège ;
- limitation de l'écoute réseau d'un service ;
- configuration et validation de nftables ;
- gestion des services avec systemd ;
- analyse des ports avec `ss` et PowerShell ;
- automatisation avec Bash et cron ;
- sauvegarde, compression et restauration ;
- diagnostic des erreurs d'authentification et de permissions ;
- validation de la persistance après redémarrage ;
- documentation d'une infrastructure technique.

## 17. Conclusion et travaux restants

L'infrastructure est fonctionnelle. Les services, les données, les permissions, le pare-feu, les sauvegardes manuelles et la restauration ont été validés.

Travaux restants :

1. confirmer une exécution automatique de cron dans `/var/log/mariadb-backup.log` ;
2. tester une dernière fois les ports 22, 80 et 3306 depuis Windows après redémarrage ;
3. créer un instantané Vagrant nommé `infrastructure-validee` ;
4. préparer un diagramme d'architecture ;
5. préparer la présentation et la démonstration finale.

Cette réalisation répond au thème de la modularbeit en présentant une implémentation de base de données orientée infrastructure, administration, sécurité et continuité des données.
