# Cloud-1 - Déploiement automatisé WordPress

Déploiement automatisé d'une stack WordPress + MySQL + PHPMyAdmin sur AWS EC2 avec Ansible et Docker.

## 📋 Prérequis

### Local
- **Ansible** installé (`pip install ansible`)
- **Python 3** 
- Clé SSH AWS dans `~/.ssh/cloud-1-key.pem` (chmod 400)

### AWS
- Instance EC2 Ubuntu 24.04 LTS
- Elastic IP configurée et associée
- Security Group avec ports ouverts : 22 (SSH), 80 (HTTP), 8080 (PHPMyAdmin)
- Port 3306 (MySQL) **fermé** pour sécurité

## 🚀 Déploiement

### 1. Configuration initiale

```bash
# Cloner le projet
git clone <votre-repo>
cd cloud-1

# Créer et configurer .env avec vos passwords
cp .env.example .env
nano .env

# Mettre à jour inventory.ini avec votre Elastic IP
nano inventory.ini
```

### 2. Charger les variables d'environnement

```bash
source .env
```

### 3. Tester la connexion SSH

```bash
ansible -i inventory.ini cloud_server -m ping
```

**Résultat attendu :** `aws_server | SUCCESS => { "ping": "pong" }`

### 4. Lancer le déploiement

```bash
ansible-playbook -i inventory.ini playbook.yml
```

**Durée :** ~5-10 minutes

## 📁 Structure du projet

```
cloud-1/
├── .env                      # Variables d'environnement (secrets)
├── .gitignore               # Protection des fichiers sensibles
├── inventory.ini            # Configuration serveurs Ansible
├── playbook.yml             # Playbook Ansible principal
├── docker-compose.yml.j2    # Template Docker Compose (Jinja2)
└── README.md                # Documentation
```

## 🏗️ Architecture

```
Internet
    ↓
Elastic IP (15.x.x.x)
    ↓
AWS EC2 Ubuntu
    ↓
Docker Network (bridge)
    ├── Container MySQL (port 3306 - interne)
    ├── Container WordPress (port 80)
    └── Container PHPMyAdmin (port 8080)
```

## 🔐 Sécurité

### Bonnes pratiques appliquées
- ✅ Passwords dans `.env` (jamais en dur)
- ✅ `.gitignore` protège `.env` et `.pem`
- ✅ MySQL accessible uniquement en interne (pas exposé sur Internet)
- ✅ Templates Jinja2 pour injection sécurisée des variables
- ✅ Security Group AWS restreint les accès

### Fichiers sensibles à NE JAMAIS commit
```bash
.env                    # Passwords
*.pem                   # Clés SSH
cloud-1-key.pem        # Clé privée AWS
```

## 🌐 Accès aux services

Une fois déployé, accédez aux services :

- **WordPress** : http://ELASTIC_IP/
- **PHPMyAdmin** : http://ELASTIC_IP:8080/
- **MySQL** : Accessible uniquement depuis les containers Docker (sécurisé)

### Identifiants PHPMyAdmin

```
Utilisateur : root
Password : Voir MYSQL_ROOT_PASSWORD dans .env

OU

Utilisateur : wordpress_user
Password : Voir MYSQL_PASSWORD dans .env
```

## 🔄 Persistance des données

Les volumes Docker assurent la persistance :
- `mysql_data` : Base de données MySQL
- `wordpress_data` : Fichiers WordPress (thèmes, plugins, uploads)

**Les données persistent même après :**
- Redémarrage de l'instance EC2 ✅
- Redémarrage des containers ✅
- Crash des services ✅

## 🐛 Troubleshooting

### Voir les logs des containers

```bash
ssh ubuntu@ELASTIC_IP -i ~/.ssh/cloud-1-key.pem
cd /home/ubuntu/cloud-1
docker compose logs -f
```

### Redémarrer les services

```bash
docker compose restart
```

### Vérifier l'état des containers

```bash
docker compose ps
docker compose top
```

### Nettoyer et redéployer

```bash
# Sur le serveur
docker compose down -v    # Supprime tout (containers + volumes)

# Depuis votre machine
ansible-playbook -i inventory.ini playbook.yml
```

### Erreur "Connection timeout"

**Cause :** Security Group bloque le port 22

**Solution :** AWS Console → EC2 → Security Groups → Autoriser votre IP sur port 22

### Erreur "Access denied" MySQL

**Cause :** Mauvais identifiants ou anciennes données

**Solution :** Nettoyer les volumes (`docker compose down -v`) puis redéployer

## 📊 Monitoring

### CloudWatch (AWS)
Métriques disponibles automatiquement :
- CPU Utilization
- Network In/Out
- Disk Read/Write

**Accès :** AWS Console → CloudWatch → Metrics → EC2

### Vérifier l'état des services

```bash
# Depuis le serveur
systemctl status docker
docker compose ps
curl http://localhost:80        # WordPress
curl http://localhost:8080      # PHPMyAdmin
```

## 💾 Backup & Restore

### Créer un snapshot AWS

```bash
AWS Console → EC2 → Instances → Actions → Images et modèles → Créer une image
```

**Note :** Le snapshot sauvegarde tout (système + volumes Docker + base de données)

### Restaurer depuis un snapshot

```bash
EC2 → Images → AMI → Sélectionner l'image → Lancer une instance
```

## 🔧 Maintenance

### Mises à jour système

```bash
ssh ubuntu@ELASTIC_IP
sudo apt update && sudo apt upgrade -y
```

### Mises à jour des images Docker

```bash
cd /home/ubuntu/cloud-1
docker compose pull      # Télécharge nouvelles images
docker compose up -d     # Redémarre avec nouvelles versions
```

## 📝 Notes importantes

### Idempotence
Le playbook est **idempotent** : peut être relancé plusieurs fois sans casser le système.

## 🎓 Concepts clés

- **Infrastructure as Code** : Configuration versionnée et reproductible
- **Variables Ansible** : Séparation code/configuration
- **Templates Jinja2** : Génération dynamique de fichiers
- **Handlers Ansible** : Restart automatique seulement si nécessaire
- **Volumes Docker** : Persistance des données
- **Networks Docker** : Communication inter-containers

## 👤 Auteur

Matheo Vacherat - Projet cloud-1 - École 42