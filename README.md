# docker-frontend-stack

Déploiement d'une stack frontend complète et sécurisée qui combine quatre composants essentiels : Traefik, Authelia, CrowdSec et Redis.

> 📖 **Article complet** : Ce dépôt accompagne l'article détaillé [Perfect Homelab - Série 08](https://f84.allfab.fr/posts/20251127-perfect-homelab-serie-08/) publié sur le blog [f84.allfab.fr](https://f84.allfab.fr)

## 📋 Description

Cette stack Docker Compose fournit une solution complète de reverse proxy sécurisé avec :

- **Traefik** : Reverse proxy avec gestion automatique des certificats SSL/TLS (Let's Encrypt)
- **Authelia** : Serveur d'authentification et d'autorisation (SSO, 2FA)
- **CrowdSec** : Système de détection et de prévention d'intrusions (IDS/IPS)
- **Redis** : Cache pour les sessions Authelia

## ⚠️ Configuration initiale

### 1. Cloner le projet

```bash
git clone https://github.com/allfab/docker-frontend-stack.git
cd docker-frontend-stack
```

### 2. Créer les fichiers de configuration à partir des templates

**IMPORTANT** : Les fichiers de configuration contenant des secrets ne sont pas inclus dans le dépôt pour des raisons de sécurité.

Copiez les fichiers d'exemple et personnalisez-les avec vos propres valeurs :

```bash
# Copier le fichier d'environnement
cp .env.example .env

# Copier les configurations Authelia
cp authelia/configuration.yml.example authelia/configuration.yml
cp authelia/users/config.yml.example authelia/users/config.yml

# Copier la configuration des notifications CrowdSec
cp crowdsec/config/notifications/ntfy.yaml.example crowdsec/config/notifications/ntfy.yaml
```

### 3. Éditer les fichiers de configuration

#### `.env`

Modifiez les variables suivantes avec vos propres valeurs :

```env
REDIS_PASSWORD="your_secure_redis_password_here"
OVH_APPLICATION_KEY="your_ovh_application_key"
OVH_APPLICATION_SECRET="your_ovh_application_secret"
OVH_CONSUMER_KEY="your_ovh_consumer_key"
CF_DNS_API_TOKEN="your_cloudflare_api_token"
BOUNCER_KEY_TRAEFIK="generate_with_cscli_bouncers_add"
```

#### `authelia/configuration.yml`

Remplacez les placeholders suivants :

- `jwt_secret`: Générer avec `openssl rand -base64 32`
- `secret`: Générer avec `openssl rand -base64 32`
- `encryption_key`: Générer avec `openssl rand -base64 32`
- `password`: Mot de passe de votre compte Gmail ou serveur SMTP
- Remplacer `mydomain.com` par votre domaine réel
- Adapter les plages d'IPs internes à votre réseau

#### `authelia/users/config.yml`

Créez un hash de mot de passe avec :

```bash
docker run --rm -it authelia/authelia:latest authelia crypto hash generate argon2 --password 'votre_mot_de_passe'
```

Puis remplacez le hash dans le fichier.

#### `crowdsec/config/notifications/ntfy.yaml`

- Remplacer l'URL de votre serveur Ntfy
- Remplacer le token Bearer par votre token Ntfy

### 4. Générer le bouncer CrowdSec pour Traefik

Après le premier démarrage de CrowdSec :

```bash
docker exec crowdsec cscli bouncers add crowdsec-bouncer-traefik-plugin
```

Copiez la clé générée dans `.env` → `BOUNCER_KEY_TRAEFIK`

## 🚀 Déploiement

### Lancer la stack complète

```bash
docker-compose up -d
```

### Vérifier les logs

```bash
# Tous les services
docker-compose logs -f

# Service spécifique
docker-compose logs -f traefik
docker-compose logs -f authelia
docker-compose logs -f crowdsec
```

### Vérifier que tous les services sont démarrés

```bash
docker-compose ps
```

## 🔧 Configuration post-déploiement

### Traefik

- Dashboard accessible sur : `https://traefik.mydomain.com` — servi via le routeur interne `api@internal`, protégé par une double barrière : Authelia (héritée de l'entryPoint `websecure`) **et** le middleware `dashboard-ipallow` (filtrage IP). L'API est en `insecure: false` et le port `8080` n'est **pas** exposé.
- Adaptez la `sourceRange` du middleware `dashboard-ipallow` (`traefik/rules/00-middlewares.yml`) à vos réseaux avant de déployer.
- Les certificats Let's Encrypt sont stockés dans `traefik/acme.json`

### Authelia

- Interface d'authentification : `https://auth.mydomain.com`
- Premier login : utilisez les credentials configurés dans `authelia/users/config.yml`

### CrowdSec

Vérifier le statut :

```bash
docker exec crowdsec cscli metrics
docker exec crowdsec cscli decisions list
```

## 📁 Structure du projet

```
docker-frontend-stack/
├── docker-compose.yml           # Orchestration principale
├── .env.example                 # Template des variables d'environnement
├── .gitignore                   # Fichiers à ignorer par Git
├── authelia/
│   ├── configuration.yml.example
│   ├── users/
│   │   └── config.yml.example
│   ├── db/                      # Base de données SQLite
│   └── logs/                    # Logs Authelia
├── crowdsec/
│   ├── config/
│   │   ├── acquis.yaml
│   │   ├── profiles.yaml
│   │   └── notifications/
│   │       └── ntfy.yaml.example
│   └── data/                    # Données CrowdSec
├── traefik/
│   ├── config.yml               # Configuration statique
│   ├── rules/                   # Règles de routing dynamiques
│   ├── logs/                    # Logs Traefik
│   └── acme.json                # Certificats Let's Encrypt
└── redis/
    └── data/                    # Données Redis persistantes
```

## 🔒 Sécurité

### Fichiers sensibles

Les fichiers suivants contiennent des données sensibles et **ne doivent jamais être commités** :

- `.env`
- `authelia/configuration.yml`
- `authelia/users/config.yml`
- `crowdsec/config/notifications/ntfy.yaml`
- `traefik/acme.json`
- Tous les dossiers `data/`, `db/`, `logs/`

Le `.gitignore` est configuré pour les exclure automatiquement.

### Bonnes pratiques

- ✅ Utiliser des mots de passe forts et uniques
- ✅ Générer des secrets aléatoires avec `openssl rand -base64 32`
- ✅ Activer la 2FA sur Authelia pour tous les utilisateurs
- ✅ Surveiller régulièrement les logs et métriques CrowdSec
- ✅ Mettre à jour régulièrement les images Docker

## 🔍 Troubleshooting

### Les conteneurs ne démarrent pas

```bash
# Vérifier les logs
docker-compose logs

# Vérifier la configuration
docker-compose config
```

### Erreur de certificat SSL

Vérifiez que :
- Le DNS pointe correctement vers votre serveur
- Les ports 80 et 443 sont bien ouverts
- Le challenge DNS est correctement configuré (OVH ou Cloudflare)

### Authelia ne se connecte pas à Redis

Vérifiez que le mot de passe Redis dans `authelia/configuration.yml` correspond à celui dans `.env`

## 📚 Documentation

- [Traefik](https://doc.traefik.io/traefik/)
- [Authelia](https://www.authelia.com/)
- [CrowdSec](https://docs.crowdsec.net/)
- [Redis](https://redis.io/documentation)

## 📝 Licence

Ce projet est open source sous licence MIT.

## 🤝 Contribution

Les contributions sont les bienvenues ! N'hésitez pas à ouvrir une issue ou une pull request.

## 📬 Contact

Pour toute question ou support :
- 📖 Consultez l'article complet : [Perfect Homelab - Série 08](https://f84.allfab.fr/posts/20251127-perfect-homelab-serie-08/)
- 🌐 Visitez le blog : [f84.allfab.fr](https://f84.allfab.fr)
- 💬 Laissez un commentaire sur l'article pour poser vos questions
