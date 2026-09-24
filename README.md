# Aufbau einer sicheren MariaDB-Datenbankinfrastruktur unter Debian

## 1. Projektvorstellung

### 1.1 Kontext

Dieses Projekt wird im Rahmen der Modularbeit zur Implementierung einer Datenbank durchgeführt. Es befasst sich mit der Infrastruktur, die für den Betrieb eines Datenbankservers erforderlich ist. Die Entwicklung einer vollständigen Anwendung gehört nicht zum Projektumfang.

Eine kleine relationale Datenbank dient dazu, die Installation, die Administration, die Berechtigungen, die Netzwerksicherheit, die Sicherungen und die Wiederherstellung zu überprüfen.

## 2. Ziele

- eine Debian-VM mit Vagrant und VirtualBox bereitstellen;
- MariaDB 10.11 installieren und betreiben;
- die Datenbank mit phpMyAdmin administrieren;
- das Prinzip der geringsten Rechte anwenden;
- MariaDB auf die lokale Schnittstelle beschränken;
- Verbindungen mit nftables filtern;
- die Datenbank automatisch sichern;
- die Integrität der Archive überprüfen;
- die Daten in einer separaten Datenbank wiederherstellen;

## 3. Verwendete Werkzeuge

| Werkzeug | Verwendung |
|---|---|
| Windows | Host-Betriebssystem |
| VirtualBox | Ausführung der VM |
| Vagrant | Reproduzierbare Erstellung und Verwaltung der VM |
| Debian 12 Bookworm | Betriebssystem des Servers |
| MariaDB 10.11 | Relationales Datenbankmanagementsystem |
| Apache HTTP Server | Webserver für phpMyAdmin |
| PHP | Ausführung von phpMyAdmin |
| phpMyAdmin | Grafische Administration von MariaDB |
| nftables | Filterung der Netzwerkverbindungen |
| OpenSSH | Fernadministration von Debian |
| SQL | Erstellung von Tabellen, Konten und Berechtigungen |
| `mariadb-dump` | Logischer Export der Datenbank |
| gzip | Komprimierung und Überprüfung der Sicherungen |
| cron | Planung der Sicherungen |

Diese Technologien entsprechen den Themen des Unterrichts: Debian, VirtualBox, Vagrant, relationales Modell, MariaDB, LAMP, phpMyAdmin, SQL, Sicherheit, Benutzer und Datenintegrität.

## 4. Architektur

| Element | Konfiguration |
|---|---|
| Host-Rechner | Windows |
| Hypervisor | VirtualBox |
| VM-Verwaltung | Vagrant |
| Virtuelle Maschine | Debian 12 |
| Hostname | `db-server` |
| NAT-Schnittstelle | `eth0`, Adresse `10.0.2.15/24` |
| Private Schnittstelle | `eth1`, Adresse `192.168.56.20/24` |
| Arbeitsspeicher | 2 GB |
| Virtuelle Prozessoren | 2 |
| SSH-Zugriff | `vagrant ssh`, Port 22/TCP |
| Weboberfläche | Apache und phpMyAdmin, Port 80/TCP |
| Vagrant-Portweiterleitung | `127.0.0.1:8080` zum Port 80 der VM |
| Hauptdatenbank | `infrastructure_db` |
| Wiederherstellungsdatenbank | `infrastructure_restore_test` |
| MariaDB | nur `127.0.0.1:3306` |

Die VM verwendet NAT für den Internetzugang und ein Host-only-Netzwerk für die Kommunikation mit Windows. Der MariaDB-Port wird nicht auf dem Host veröffentlicht.

## 5. Erstellung der virtuellen Maschine

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


## 6. Installation und Absicherung von MariaDB

```bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb
sudo mariadb-secure-installation
```

Umgesetzte Massnahmen:

- Entfernung der anonymen Konten;
- Verbot der Remote-Anmeldung von `root`;
- Entfernung der standardmässigen Testdatenbank;
- lokale Administration mit `sudo mariadb`;
- individuelle Konten für die Administration und Nutzung der Datenbank.

### 6.1 Beschränkung der Netzwerk-Listening-Adresse

Die Datei `/etc/mysql/mariadb.conf.d/99-local-bind.cnf` enthält:

```ini
[mysqld]
bind-address = 127.0.0.1
```

Der von systemd verwaltete Netzwerk-Socket wurde deaktiviert und MariaDB anschliessend neu gestartet:

```bash
sudo systemctl disable --now mariadb.socket
sudo systemctl restart mariadb
sudo ss -lntp | grep 3306
```

Validiertes Ergebnis: `127.0.0.1:3306`. MariaDB ist somit von Windows aus nicht direkt erreichbar.

## 7. Apache, PHP und phpMyAdmin

```bash
sudo apt install -y apache2 php libapache2-mod-php php-mysql
sudo systemctl enable --now apache2
sudo apt install -y phpmyadmin
sudo a2enconf phpmyadmin
sudo systemctl reload apache2
```

Adresse unter Windows:

```text
http://localhost:8080/phpmyadmin
```

Überprüfung von Apache:

```bash
sudo apache2ctl configtest
```

Erwartetes Ergebnis: `Syntax OK`.

## 8. Demonstrationsdatenbank

```sql
CREATE DATABASE infrastructure_db
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

### 8.1 Tabelle zur Kontrolle der Dienste

```sql
CREATE TABLE server_status (
    status_id INT AUTO_INCREMENT PRIMARY KEY,
    service_name VARCHAR(100) NOT NULL,
    service_status VARCHAR(20) NOT NULL,
    checked_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 8.2 Tabelle zum Testen der Berechtigungen

```sql
CREATE TABLE permission_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(100) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Die Zeile `Créé par db_admin` wird für die Demonstration beibehalten.

## 9. Benutzer und Berechtigungen

| Konto | Funktion | Berechtigungen auf `infrastructure_db` |
|---|---|---|
| `db_admin` | Administration der Datenbank | Alle Berechtigungen auf der Datenbank |
| `db_writer` | Lesen und Ändern | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| `db_reader` | Nur Lesen | `SELECT` |

Erstellung über `sudo mariadb`:

```sql
CREATE USER IF NOT EXISTS 'db_admin'@'localhost'
IDENTIFIED BY '<Modularbeit_admin>';

CREATE USER IF NOT EXISTS 'db_writer'@'localhost'
IDENTIFIED BY '<Modularbeit_writer>';

CREATE USER IF NOT EXISTS 'db_reader'@'localhost'
IDENTIFIED BY '<Modularbeit_reader>';

GRANT ALL PRIVILEGES ON infrastructure_db.*
TO 'db_admin'@'localhost';

GRANT SELECT, INSERT, UPDATE, DELETE ON infrastructure_db.*
TO 'db_writer'@'localhost';

GRANT SELECT ON infrastructure_db.*
TO 'db_reader'@'localhost';
```


Überprüfung:

```sql
SHOW GRANTS FOR 'db_admin'@'localhost';
SHOW GRANTS FOR 'db_writer'@'localhost';
SHOW GRANTS FOR 'db_reader'@'localhost';
```

### 9.1 Testergebnisse

| Operation | `db_admin` | `db_writer` | `db_reader` |
|---|:---:|:---:|:---:|
| `SELECT` | Erlaubt | Erlaubt | Erlaubt |
| `INSERT` | Erlaubt | Erlaubt | Verweigert |
| `UPDATE` | Erlaubt | Erlaubt | Verweigert |
| `DELETE` | Erlaubt | Erlaubt | Verweigert |
| `CREATE TABLE` | Erlaubt | Verweigert | Verweigert |
| Globale Kontoverwaltung | Verweigert | Verweigert | Verweigert |

Die verweigerten Operationen von `db_reader` und `db_writer` erzeugten den MariaDB-Fehler `1142`. Die anfängliche Erstellung der Konten mit `db_admin` wurde ebenfalls verweigert, da `CREATE USER` eine globale Berechtigung ist. Diese Ergebnisse bestätigen das Prinzip der geringsten Rechte.

## 10. nftables-Firewall

```bash
sudo apt install -y nftables
sudo cp /etc/nftables.conf /etc/nftables.conf.backup
sudo systemctl enable nftables
```

Konfiguration von `/etc/nftables.conf`:

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

Validierung:

```bash
sudo nft -c -f /etc/nftables.conf
sudo systemctl restart nftables
sudo nft list ruleset
```

| Dienst | Port | Funktion |
|---|---:|---|
| SSH | 22/TCP | Administration von Debian |
| HTTP | 80/TCP | Apache und phpMyAdmin |
| ICMP/ICMPv6 | Kein Port | Netzwerk- und Kontrolltests |
| DHCP | 67/UDP zu 68/UDP | Netzwerkkonfiguration |

Der Port `3306/TCP` ist für eingehende Verbindungen nicht freigegeben.

Test unter Windows:

```powershell
Test-NetConnection 192.168.56.20 -Port 3306
```

Beobachtetes Ergebnis: `PingSucceeded: True` und `TcpTestSucceeded: False`. Die VM ist erreichbar, MariaDB jedoch nicht direkt zugänglich.

## 11. MariaDB-Sicherung

Geschütztes Verzeichnis:

```bash
sudo mkdir -p /var/backups/mariadb
sudo chown root:root /var/backups/mariadb
sudo chmod 700 /var/backups/mariadb
```

Skript `/usr/local/sbin/backup-infrastructure-db`:

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

Schutz und Ausführung:

```bash
sudo chown root:root /usr/local/sbin/backup-infrastructure-db
sudo chmod 700 /usr/local/sbin/backup-infrastructure-db
sudo bash -n /usr/local/sbin/backup-infrastructure-db
sudo /usr/local/sbin/backup-infrastructure-db
```

Während der Tests wurden zwei gültige Archive erstellt.

### 11.1 Überprüfung der Archive

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

gzip meldete keine Fehler. Die Anweisungen `CREATE TABLE` und `INSERT INTO` für `permission_test` waren vorhanden.

## 12. Automatisierung mit cron

```bash
sudo apt install -y cron
sudo systemctl enable --now cron
```

Datei `/etc/cron.d/mariadb-backup`:

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

Die Sicherung ist täglich um 02:00 Uhr geplant.

### 12.1 Zeitzone

```bash
sudo timedatectl set-timezone Europe/Zurich
```

Validiertes Ergebnis: Zeitzone `Europe/Zurich`, NTP-Synchronisierung aktiv. Cron wird um 02:00 Uhr Schweizer Zeit ausgeführt und berücksichtigt automatisch die Zeitumstellungen.

Die manuelle Ausführung wurde validiert. Die automatische Erstellung eines Archivs durch cron muss nach einer geplanten Ausführung noch in `/var/log/mariadb-backup.log` bestätigt werden.

## 13. Wiederherstellungstest

Zum Schutz des Originals wurde eine separate Datenbank erstellt:

```sql
CREATE DATABASE infrastructure_restore_test
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Wiederherstellung:

```bash
sudo sh -c \
'gzip -dc /var/backups/mariadb/infrastructure_db_2026-09-24_11-29-19.sql.gz |
mariadb infrastructure_restore_test'
```

Kontrollen:

```bash
sudo mariadb infrastructure_restore_test -e "SHOW TABLES;"
sudo mariadb infrastructure_restore_test -e \
"SELECT * FROM permission_test;"
```

Ergebnisse:

- `permission_test` wurde wiederhergestellt;
- `server_status` wurde wiederhergestellt;
- die Zeile `Créé par db_admin` wurde wiederhergestellt;
- die ursprüngliche Datenbank blieb unverändert.

Mit der Sicherung können somit die Struktur und die Daten wiederhergestellt werden.

## 14. Validierung nach dem Neustart

Die VM wurde mit `vagrant halt` angehalten und anschliessend mit `vagrant up` neu gestartet.

```bash
sudo systemctl is-active ssh apache2 mariadb nftables cron
sudo systemctl is-enabled ssh apache2 mariadb nftables cron
```

Alle fünf Dienste waren `active` und `enabled`.

Nach dem Neustart waren folgende Elemente weiterhin vorhanden:

- `infrastructure_db`;
- `infrastructure_restore_test`;
- `permission_test` und ihr Datensatz;
- die MariaDB-Listening-Adresse war weiterhin auf `127.0.0.1:3306` beschränkt;
- die nftables-Regeln und die `drop`-Richtlinien;
- die Dienste Apache, SSH, MariaDB, nftables und cron.

Die Persistenz wurde validiert.

## 15. Abschliessende Testtabelle

| Test | Beobachtetes Ergebnis | Status |
|---|---|:---:|
| Hostname `db-server` | Konform | Bestanden |
| Private Adresse `192.168.56.20` | Konform | Bestanden |
| Internet- und DNS-Zugang | 0 % Verlust zu `debian.org` | Bestanden |
| SSH | `active` und `enabled` | Bestanden |
| Apache | `active` und `enabled` | Bestanden |
| MariaDB | `active` und `enabled` | Bestanden |
| nftables | `active` und `enabled` | Bestanden |
| cron | `active` und `enabled` | Bestanden |
| phpMyAdmin | Anmeldung erfolgreich | Bestanden |
| MariaDB-Listening-Adresse | `127.0.0.1:3306` | Bestanden |
| Port 3306 von Windows aus | `TcpTestSucceeded: False` | Bestanden |
| Lesen mit `db_reader` | Erlaubt | Bestanden |
| Schreiben mit `db_reader` | Fehler 1142 | Bestanden |
| CRUD mit `db_writer` | Erlaubt | Bestanden |
| `CREATE TABLE` mit `db_writer` | Fehler 1142 | Bestanden |
| Administration mit `db_admin` | Konform | Bestanden |
| Manuelle Sicherung | Zwei Archive erstellt | Bestanden |
| gzip-Integrität | Kein Fehler | Bestanden |
| SQL-Inhalt | Tabellen und Daten vorhanden | Bestanden |
| Wiederherstellung | Tabellen und Daten wiederhergestellt | Bestanden |
| Persistenz nach Neustart | Dienste und Daten erhalten | Bestanden |
| Automatische cron-Ausführung | Um 02:00 Uhr zu beobachten | Ausstehend |

## 16. Nachgewiesene Kompetenzen

- Virtualisierung eines Linux-Servers;
- Verwendung von Vagrant und VirtualBox;
- Konfiguration von NAT- und Host-only-Netzwerken;
- Installation und Administration von MariaDB;
- Verwendung von SQL und einer relationalen Datenbank;
- Administration mit phpMyAdmin;
- Erstellung von Konten und Verwaltung von Berechtigungen;
- Anwendung des Prinzips der geringsten Rechte;
- Beschränkung der Netzwerk-Listening-Adresse eines Dienstes;
- Konfiguration und Validierung von nftables;
- Verwaltung von Diensten mit systemd;
- Analyse von Ports mit `ss` und PowerShell;
- Automatisierung mit Bash und cron;
- Sicherung, Komprimierung und Wiederherstellung;
- Diagnose von Authentifizierungs- und Berechtigungsfehlern;
- Validierung der Persistenz nach einem Neustart;
- Dokumentation einer technischen Infrastruktur.

