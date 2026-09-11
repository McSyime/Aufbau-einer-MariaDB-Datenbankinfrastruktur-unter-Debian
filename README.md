# Aufbau einer sicheren MariaDB-Datenbankinfrastruktur unter Debian

## 1. Présentation du projet
### Contexte

Ce projet est réalisé dans le cadre de la modularbeit consacrée à l'implémentation d'une base de données. Le travail se concentre sur l'infrastructure nécessaire au fonctionnement d'un serveur de base de données. Il ne prévoit pas le développement d'une application complète.

Une petite base de données sert uniquement à configurer, administrer et tester l'infrastructure.

## 2. Buts du projet

Le projet poursuit les objectifs suivants :

- déployer une machine virtuelle Debian reproductible ;
- installer et exploiter un serveur de base de données MariaDB ;
- proposer une interface Web d'administration avec phpMyAdmin ;
- créer une base relationnelle minimale pour vérifier le fonctionnement du serveur ;
- séparer les droits d'administration, d'écriture et de lecture ;
- limiter les accès réseau au moyen d'un pare-feu ;
- préparer ultérieurement une solution de sauvegarde et de restauration ;
- documenter les opérations et les tests réalisés.

Le résultat attendu est une infrastructure fonctionnelle, sécurisée et vérifiable. La base de démonstration ne constitue pas une application métier terminée.

## 3. Outils utilisés

| Outil | Rôle dans le projet |
|---|---|
| VirtualBox | Exécution de la machine virtuelle |
| Vagrant | Création et configuration reproductible de la VM |
| Debian 12 Bookworm | Système d'exploitation du serveur |
| MariaDB 10.11 | Système de gestion de base de données relationnelle |
| Apache HTTP Server | Serveur Web utilisé par phpMyAdmin |
| PHP | Exécution de phpMyAdmin côté serveur |
| phpMyAdmin | Administration graphique de MariaDB |
| nftables | Filtrage des connexions réseau |
| SSH | Administration distante de la VM |
| SQL | Création des données, des utilisateurs et des privilèges |

Ces technologies correspondent aux outils et aux thèmes abordés dans le cours, notamment Debian, VirtualBox, Vagrant, le modèle relationnel, MariaDB, LAMP, phpMyAdmin, SQL, les utilisateurs et l'intégrité des données.

## 4. Architecture actuelle

| Composant | Configuration |
|---|---|
| Machine hôte | Windows avec VirtualBox et Vagrant |
| Machine virtuelle | Debian 12 |
| Nom d'hôte | `db-server` |
| Adresse privée | `192.168.56.20` |
| Mémoire vive | 2 Go |
| Processeurs virtuels | 2 |
| Accès SSH | Commande `vagrant ssh` |
| Interface Web | Apache et phpMyAdmin |
| Accès depuis l'hôte | `http://localhost:8080/phpmyadmin` |
| Base de données | `infrastructure_db` |
| Port MariaDB | `3306`, limité à `127.0.0.1` |

Le port 8080 de la machine hôte est redirigé vers le port 80 de la VM. MariaDB n'est pas directement publié sur la machine hôte.

## 5. Création de la machine virtuelle

La VM a été définie avec un `Vagrantfile` utilisant l'image `debian/bookworm64`.

Configuration principale :

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

## 6. Installation de MariaDB

MariaDB a été installé sur Debian :

```bash
sudo apt update
sudo apt install -y mariadb-server
sudo systemctl enable --now mariadb
```

L'installation a été sécurisée avec :

```bash
sudo mariadb-secure-installation
```

Les principales mesures appliquées sont :

- protection de l'administration MariaDB ;
- suppression des utilisateurs anonymes ;
- refus de la connexion distante du compte `root` ;
- suppression de la base de test inutile ;
- actualisation des privilèges.

MariaDB utilise actuellement l'adresse d'écoute locale :

```text
127.0.0.1:3306
```

Cela empêche une connexion directe au serveur MariaDB depuis une autre machine.

## 7. Base de données de démonstration

La base suivante a été créée :

```sql
CREATE DATABASE infrastructure_db;
```

Cette base sert à tester l'administration, les comptes, les autorisations, les futures sauvegardes et les restaurations. Elle pourra contenir quelques tables simples, mais le projet reste centré sur l'infrastructure.

## 8. Installation d'Apache, PHP et phpMyAdmin

Les composants Web ont été installés avec :

```bash
sudo apt install -y apache2 php libapache2-mod-php php-mysql
sudo systemctl enable --now apache2
```

phpMyAdmin a ensuite été installé et associé à Apache. L'interface est accessible depuis la machine hôte à l'adresse suivante :

```text
http://localhost:8080/phpmyadmin
```

## 9. Utilisateurs MariaDB et séparation des droits

Trois niveaux d'accès ont été prévus :

| Compte | Fonction | Droits attendus |
|---|---|---|
| `db_admin` | Administration de la base du projet | Administration de `infrastructure_db` |
| `db_writer` | Utilisation avec modification des données | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| `db_reader` | Consultation uniquement | `SELECT` |

Les utilisateurs limités sont créés depuis la console d'administration MariaDB :

```sql
CREATE USER IF NOT EXISTS 'db_writer'@'localhost'
IDENTIFIED BY '<mot_de_passe_writer>';

CREATE USER IF NOT EXISTS 'db_reader'@'localhost'
IDENTIFIED BY '<mot_de_passe_reader>';

GRANT SELECT, INSERT, UPDATE, DELETE
ON infrastructure_db.*
TO 'db_writer'@'localhost';

GRANT SELECT
ON infrastructure_db.*
TO 'db_reader'@'localhost';
```

Les privilèges peuvent être contrôlés avec :

```sql
SHOW GRANTS FOR 'db_admin'@'localhost';
SHOW GRANTS FOR 'db_writer'@'localhost';
SHOW GRANTS FOR 'db_reader'@'localhost';
```

La création de `db_writer` et `db_reader` avec `db_admin` a d'abord échoué. Ce comportement s'explique par le fait que `CREATE USER` est un privilège global. Les comptes ont donc été créés correctement avec l'administrateur système au moyen de `sudo mariadb`. Cette séparation respecte le principe du moindre privilège.

## 10. Configuration du pare-feu

`nftables` a été installé et activé :

```bash
sudo apt install -y nftables
sudo cp /etc/nftables.conf /etc/nftables.conf.backup
```

Configuration appliquée dans `/etc/nftables.conf` :

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

### Flux autorisés

| Service | Port | Utilité |
|---|---:|---|
| SSH | 22/TCP | Administration de Debian |
| HTTP | 80/TCP | Accès à Apache et phpMyAdmin |
| ICMP/ICMPv6 | Sans port | Tests réseau et messages de contrôle |
| DHCP | 67/UDP vers 68/UDP | Attribution de la configuration réseau |

Le port MariaDB `3306/TCP` n'est pas autorisé en entrée. phpMyAdmin communique avec MariaDB localement dans la VM.

## 14. Prochaines étapes

Les prochaines étapes techniques sont :

1. terminer et tester les comptes `db_writer` et `db_reader` ;
2. créer quelques tables et données minimales pour les tests ;
3. mettre en place une sauvegarde automatisée avec `mariadb-dump` ;
4. tester une restauration complète ;
5. vérifier la persistance après un redémarrage de la VM ;
6. produire un tableau final des tests et de leurs résultats.

## 15. Compétences mises en évidence

Le projet permet de démontrer les compétences suivantes :

- virtualisation d'un serveur Linux ;
- automatisation de la création d'une VM avec Vagrant ;
- installation et administration d'un système de base de données ;
- administration Web avec phpMyAdmin ;
- gestion des comptes et des privilèges SQL ;
- application du principe du moindre privilège ;
- configuration d'un pare-feu Linux ;
- contrôle des services avec systemd ;
- vérification des ports et des accès réseau ;
- diagnostic des erreurs d'authentification ;
- documentation d'une infrastructure technique.

Cette réalisation répond au thème de la modularbeit en présentant une implémentation de base de données principalement orientée infrastructure, administration et sécurité.
