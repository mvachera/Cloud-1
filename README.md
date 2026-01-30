# Cloud-1 - Déploiement WordPress HTTPS automatisé

Stack WordPress + MySQL + PHPMyAdmin + Nginx sur AWS EC2 avec Ansible, Docker et Let's Encrypt.

## Architecture
```
Internet → DNS → Elastic IP → Nginx (reverse proxy + SSL)
    ├── Port 80 → Redirige vers 443
    ├── Port 443 → WordPress (HTTPS)
    └── Port 8080 → phpMyAdmin (HTTPS)
        ↓
    Docker Network (bridge)
        ├── MySQL (interne uniquement)
        ├── WordPress
        ├── PHPMyAdmin
        └── Certbot (renouvellement auto)
```

## Prérequis

**Local :**
- Ansible + Python 3
- Clé SSH AWS (`~/.ssh/cloud-1-key.pem`)

**AWS :**
- EC2 Ubuntu 24.04 + Elastic IP
- Security Group : ports 22 (SSH), 80, 443, 8080 ouverts

**DNS :**
- Nom de domaine avec enregistrement A vers l'Elastic IP

## Installation
```bash
# Config
git clone <repo>
cd cloud-1
cp .env.example .env

# Tester la connexion
ansible -i inventory.ini cloud_server -m ping

# Déploiement
set -a
source .env
set +a
ansible-playbook -i inventory.ini playbook.yml
```

## Fichiers du projet
```
cloud-1/
├── playbook.yml                  # Playbook Ansible principal
├── docker-compose.yml.j2         # Template Docker Compose
├── nginx-wordpress.conf.j2       # Config HTTP (validation Certbot)
├── nginx-wordpress-ssl.conf.j2   # Config HTTPS (production)
├── inventory.ini                 # Serveurs cibles
├── .env                          # Secrets (gitignore)
└── .gitignore                    # Protection fichiers sensibles
```

## Images Docker

Toutes les images utilisent le tag `:latest` pour avoir les dernières versions :
- `mysql:latest`
- `wordpress:latest`
- `phpmyadmin:latest`
- `nginx:latest`
- `certbot/certbot:latest`

## Templates Jinja2

Les fichiers `.j2` sont des templates Ansible qui permettent d'injecter dynamiquement les variables du `.env` :
```yaml
# Dans docker-compose.yml.j2
MYSQL_ROOT_PASSWORD: {{ mysql_root_password }}
```

devient après génération :
```yaml
# Dans docker-compose.yml sur le serveur
MYSQL_ROOT_PASSWORD: mon_password_secret
```

Cela permet de séparer le code de la configuration et de ne jamais commit les secrets.

## Variables .env requises
```bash
# MySQL
MYSQL_ROOT_PASSWORD=password
MYSQL_DATABASE=wordpress_db
MYSQL_USER=wordpress_user
MYSQL_PASSWORD=password

# WordPress
WORDPRESS_DB_HOST=cloud1_mysql:3306
WORDPRESS_DB_NAME=wordpress_db
WORDPRESS_DB_USER=wordpress_user
WORDPRESS_DB_PASSWORD=password

# phpMyAdmin
PMA_HOST=cloud1_mysql
PMA_PORT=3306

# SSL/HTTPS
DOMAIN_NAME=mon-domaine.ovh
CERTBOT_EMAIL=email@example.com
```

## Accès

- WordPress : `https://mon-domaine.ovh`
- phpMyAdmin : `https://mon-domaine.ovh:8080`

## Fonctionnement HTTPS

### Pourquoi 2 fichiers nginx ?

**`nginx-wordpress.conf.j2` (HTTP) :**
- Utilisé uniquement au premier déploiement
- Permet à Certbot de valider le domaine via le challenge ACME (port 80)
- Contient la route `/.well-known/acme-challenge/` pour Let's Encrypt

**`nginx-wordpress-ssl.conf.j2` (HTTPS) :**
- Configuration finale de production
- Active TLS avec les certificats Let's Encrypt
- Reverse proxy vers WordPress (443) et phpMyAdmin (8080)
- Redirection HTTP → HTTPS

### Processus de déploiement

**Premier déploiement :**
1. Playbook détecte absence de certificat
2. Génère config HTTP (`nginx-wordpress.conf.j2`)
3. Démarre les services Docker
4. Certbot valide le domaine (challenge ACME sur port 80)
5. Let's Encrypt génère les certificats
6. Playbook génère config HTTPS (`nginx-wordpress-ssl.conf.j2`)
7. Redémarre Nginx en mode HTTPS

**Déploiements suivants :**
- Playbook détecte certificat existant
- Génère directement config HTTPS
- Démarre les services en mode HTTPS

**Renouvellement :**
- Cron job automatique tous les 7 jours
- Certificats valides 90 jours

## Sécurité

- HTTPS obligatoire (redirection HTTP → HTTPS)
- MySQL accessible uniquement en interne (pas exposé)
- TLS 1.2/1.3 uniquement
- Secrets dans `.env` (gitignore)
- SSH EC2 restreint à mon IP

## Persistence

Les volumes Docker assurent la persistence :
- `mysql_data` : base de données
- `wordpress_data` : fichiers WordPress
- `/etc/letsencrypt` : certificats SSL

Tout persiste après reboot EC2.

## Troubleshooting
```bash
# Logs
ssh ubuntu@ELASTIC_IP -i ~/.ssh/cloud-1-key.pem
cd /home/ubuntu/cloud-1
docker compose logs -f

# Vérifier SSL
sudo certbot certificates

# Redémarrer
docker compose restart

# Reset complet
docker compose down -v
sudo rm -rf /etc/letsencrypt/*
# Relancer playbook
```

## Concepts clés

- **Reverse proxy** : Nginx intercepte les requêtes, gère SSL, transmet aux containers
- **SSL termination** : Nginx déchiffre HTTPS, communique en HTTP interne avec les containers
- **Templates Jinja2** : Génération dynamique de fichiers avec injection de variables
- **Idempotence** : Le playbook peut être relancé sans casser le système
- **Infrastructure as Code** : Configuration versionnée et reproductible

---

👤 **Auteur**

Matheo Vacherat - Projet cloud-1 - École 42