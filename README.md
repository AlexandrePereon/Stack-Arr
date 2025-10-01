# Stack Arr - Gestion automatisée de médias

Stack Docker complète pour télécharger et gérer automatiquement vos films et séries TV avec Radarr, Sonarr, qBittorrent, Prowlarr et Plex.

## 🎯 Services inclus

- **Radarr** : Gestion et téléchargement automatique de films
- **Sonarr** : Gestion et téléchargement automatique de séries TV
- **qBittorrent** : Client torrent
- **Prowlarr** : Gestionnaire d'indexers (trackers torrents)
- **FlareSolverr** : Contournement des protections Cloudflare pour les indexers
- **Plex** : Serveur de streaming média

Tous les services (sauf FlareSolverr) sont exposés via **Traefik** en HTTPS avec certificats Let's Encrypt automatiques.

## 📋 Prérequis

- Docker & Docker Compose installés
- Traefik configuré avec réseau `traefik-net` (voir `/home/Projects/Traefik`)
- Noms de domaine configurés (DNS pointant vers votre serveur) :
  - `radarr.votredomaine.fr`
  - `sonarr.votredomaine.fr`
  - `qbittorrent.votredomaine.fr`
  - `prowlarr.votredomaine.fr`
  - `plex.votredomaine.fr`
- Authelia configuré (optionnel, pour protection supplémentaire)

## 🚀 Installation

### 1. Configuration initiale

Copiez le fichier d'environnement exemple :

```bash
cp .env.example .env
```

Éditez `.env` et configurez vos paramètres :

```bash
nano .env
```

Variables à personnaliser :
- `DOMAIN` : Votre nom de domaine (ex: `alexandrepereon.fr`)
- `PUID` / `PGID` : Votre UID/GID (obtenez-les avec `id`)
- `PLEX_CLAIM` : Token Plex (obtenez-le sur https://www.plex.tv/claim/)

### 2. Création de la structure de dossiers

Les dossiers seront créés automatiquement au premier lancement, mais vous pouvez les créer manuellement :

```bash
mkdir -p data/movies
mkdir -p data/tv
mkdir -p data/downloads/torrents
mkdir -p data/downloads/usenet
mkdir -p radarr/config
mkdir -p sonarr/config
mkdir -p qbittorrent/config
mkdir -p prowlarr/config
mkdir -p plex/config
```

### 3. Démarrage des services

```bash
docker compose up -d
```

Vérifiez que tout fonctionne :

```bash
docker compose ps
```

## ⚙️ Configuration

### 1. Prowlarr - Configuration de FlareSolverr

**FlareSolverr** permet de contourner les protections Cloudflare sur certains indexers.

1. Accédez à `https://prowlarr.votredomaine.fr`
2. **Settings → Indexers → Onglet "Indexer Proxies" → Add → FlareSolverr**
   - Name : `FlareSolverr`
   - Tags : `flaresolverr` (créez ce tag)
   - Host : `http://flaresolverr:8191`
   - Cliquez sur **Test** puis **Save**

**💡 Utilisation** : Pour qu'un indexer utilise FlareSolverr, ajoutez-lui le tag `flaresolverr`. FlareSolverr sera automatiquement utilisé si Cloudflare est détecté. Sans tag, le proxy est désactivé.

📚 **Plus de détails** : [TRaSH's Guide - How to setup FlareSolverr](https://trash-guides.info/Prowlarr/prowlarr-setup-flaresolverr/)

### 2. Prowlarr - Ajout des indexers

1. **Settings → Apps → Add Application (Radarr)**
   - Type : **Radarr**
   - Prowlarr Server : `http://prowlarr:9696`
   - Radarr Server : `http://radarr:7878`
   - API Key : Copiez depuis Radarr (Settings → General → API Key)
2. **Settings → Apps → Add Application (Sonarr)**
   - Type : **Sonarr**
   - Prowlarr Server : `http://prowlarr:9696`
   - Sonarr Server : `http://sonarr:8989`
   - API Key : Copiez depuis Sonarr (Settings → General → API Key)
3. **Indexers → Add Indexer**
   - Ajoutez vos trackers préférés (YGG, 1337x, The Pirate Bay, etc.)
   - **Si un indexer est protégé par Cloudflare** : Ajoutez-lui le tag `flaresolverr`
   - Les indexers seront automatiquement synchronisés vers Radarr et Sonarr

### 3. Radarr - Configuration du client de téléchargement

1. Accédez à `https://radarr.votredomaine.fr`
2. **Settings → Download Clients → Add → qBittorrent**
   - Host : `qbittorrent`
   - Port : `8080`
   - Username : `admin`
   - Password : (défini dans qBittorrent, par défaut voir logs : `docker logs qbittorrent | grep password`)
3. **Settings → Media Management**
   - Root Folder : `/data/movies`
   - Minimum Free Space : `102400` (100 GB)

### 4. Sonarr - Configuration du client de téléchargement

1. Accédez à `https://sonarr.votredomaine.fr`
2. **Settings → Download Clients → Add → qBittorrent**
   - Host : `qbittorrent`
   - Port : `8080`
   - Username : `admin`
   - Password : (même que pour Radarr)
3. **Settings → Media Management**
   - Root Folder : `/data/tv`
   - Minimum Free Space : `102400` (100 GB)
   - Episode Naming : Personnalisez selon vos préférences

### 5. qBittorrent - Configuration des chemins

1. Accédez à `https://qbittorrent.votredomaine.fr`
2. **Options → Downloads**
   - Default Save Path : `/data/downloads/torrents`
   - Keep incomplete torrents in : `/data/downloads/torrents/incomplete`
3. **Options → BitTorrent → Torrent Queueing**
   - Maximum active downloads : `2-3` (pour limiter l'espace utilisé)
   - Maximum active torrents : `5-10`

### 6. Plex - Configuration des bibliothèques

1. Accédez à `https://plex.votredomaine.fr` ou `http://votre-ip:32400/web`
2. Connectez-vous avec votre compte Plex
3. **Ajouter une bibliothèque Films**
   - Type : Films
   - Dossier : `/movies`
4. **Ajouter une bibliothèque Séries TV**
   - Type : Séries TV
   - Dossier : `/tv`
5. **Settings → Network**
   - Activer l'accès distant (port 32400 déjà exposé)

### 7. Radarr/Sonarr → Plex - Notifications automatiques

**Pour Radarr (films) :**
1. Dans Radarr : **Settings → Connect → Add → Plex Media Server**
   - Host : `plex`
   - Port : `32400`
   - Auth Token : Trouvez-le dans Plex
     - Méthode 1 : Settings → en bas de l'URL dans le navigateur
     - Méthode 2 : `docker exec plex cat /config/Library/Application\ Support/Plex\ Media\ Server/Preferences.xml | grep -oP 'PlexOnlineToken="\K[^"]+'`
   - ✅ Update Library : Activé

**Pour Sonarr (séries) :**
2. Dans Sonarr : **Settings → Connect → Add → Plex Media Server**
   - Même configuration que pour Radarr
   - Host : `plex`
   - Port : `32400`
   - Auth Token : (même token que pour Radarr)
   - ✅ Update Library : Activé

Plex sera maintenant notifié automatiquement à chaque nouveau film ou épisode !

## 📂 Structure des dossiers

```
/home/Projects/Arr/
├── docker-compose.yml          # Configuration Docker
├── .env                        # Variables d'environnement (à personnaliser)
├── .env.example               # Template de configuration
├── data/                      # Données partagées entre services
│   ├── movies/               # Films finaux (lu par Plex)
│   ├── tv/                   # Séries TV finales (lu par Plex)
│   └── downloads/            # Téléchargements en cours
│       ├── torrents/         # Torrents qBittorrent
│       └── usenet/           # Usenet (si configuré)
├── radarr/config/            # Config Radarr
├── sonarr/config/            # Config Sonarr
├── qbittorrent/config/       # Config qBittorrent
├── prowlarr/config/          # Config Prowlarr
└── plex/config/              # Config Plex
```

### Importance des chemins

**⚠️ IMPORTANT** : Tous les services doivent utiliser `/data` comme base pour permettre les **hardlinks** et **atomic moves** :

- **Radarr** : `/data` → voit `/data/movies` et `/data/downloads`
- **Sonarr** : `/data` → voit `/data/tv` et `/data/downloads`
- **qBittorrent** : `/data` → télécharge dans `/data/downloads/torrents`
- **Plex** : `/movies` et `/tv` → lit depuis `/data/movies` et `/data/tv`

Cette structure évite les copies inutiles et économise l'espace disque.

## 🔒 Sécurité

### Authentification

Tous les services ont leur propre authentification intégrée :
- **Radarr** : Configurable dans Settings → General → Authentication
- **Sonarr** : Configurable dans Settings → General → Authentication
- **qBittorrent** : Username/Password (voir logs au premier démarrage)
- **Prowlarr** : Configurable dans Settings → General → Authentication
- **Plex** : Compte Plex requis

### Protection supplémentaire (optionnel)

Si vous avez configuré **Authelia** dans Traefik, ajoutez ce label aux services :

```yaml
labels:
  - "traefik.http.routers.SERVICE.middlewares=authelia@docker"
```

## 🛠️ Maintenance

### Mise à jour des services

```bash
docker compose pull
docker compose up -d
```

### Logs

Voir les logs d'un service :

```bash
docker compose logs -f radarr
docker compose logs -f sonarr
docker compose logs -f qbittorrent
docker compose logs -f prowlarr
docker compose logs -f flaresolverr
docker compose logs -f plex
```

### Redémarrer un service

```bash
docker compose restart radarr
```

### Sauvegarder la configuration

Les dossiers à sauvegarder régulièrement :
- `radarr/config/`
- `sonarr/config/`
- `qbittorrent/config/`
- `prowlarr/config/`
- `plex/config/`

## 🎬 Workflow d'utilisation

**Pour un film (Radarr) :**
1. **Ajoutez un film dans Radarr** (via recherche ou liste)
2. **Radarr recherche automatiquement** le film via les indexers Prowlarr
3. **Radarr envoie le torrent à qBittorrent**
4. **qBittorrent télécharge** dans `/data/downloads/torrents/`
5. **Radarr importe le film** vers `/data/movies/` (hardlink)
6. **Radarr notifie Plex** qui scanne la nouvelle vidéo
7. **Le film est disponible sur Plex** pour visionnage !

**Pour une série TV (Sonarr) :**
1. **Ajoutez une série dans Sonarr** (via recherche ou liste)
2. **Sonarr recherche automatiquement** les épisodes via les indexers Prowlarr
3. **Sonarr envoie les torrents à qBittorrent**
4. **qBittorrent télécharge** dans `/data/downloads/torrents/`
5. **Sonarr importe les épisodes** vers `/data/tv/` (hardlink)
6. **Sonarr notifie Plex** qui scanne les nouveaux épisodes
7. **La série est disponible sur Plex** pour visionnage !

## ❓ Dépannage

### qBittorrent : "download client places downloads in /downloads"

**Solution** : Configurez le Remote Path Mapping dans Radarr
- Settings → Download Clients → Remote Path Mappings
- Host : `qbittorrent`
- Remote Path : `/downloads`
- Local Path : `/data/downloads/torrents`

## 📊 Performances et limites

### Espace disque

Par défaut, Radarr refusera d'importer si moins de **100 GB** restent disponibles.

Ajustez dans Radarr : Settings → Media Management → Minimum Free Space

### Limiter les téléchargements simultanés

Dans qBittorrent → Options → BitTorrent → Torrent Queueing :
- Maximum active downloads : `2-3`
- Maximum active torrents : `5-10`

### Ressources CPU/RAM recommandées

- **Minimum** : 2 CPU, 4 GB RAM
- **Recommandé** : 4 CPU, 8 GB RAM (surtout pour Plex avec transcodage)

## 📚 Ressources utiles

- [Radarr Wiki](https://wiki.servarr.com/radarr)
- [TRaSH Guides](https://trash-guides.info/) - Guides de configuration avancée
- [LinuxServer.io Docs](https://docs.linuxserver.io/)
- [Plex Support](https://support.plex.tv/)

---

**Note** : Assurez-vous d'utiliser cette stack uniquement avec du contenu dont vous possédez les droits ou qui est légalement téléchargeable dans votre juridiction.
