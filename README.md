# Stack Arr - Gestion automatisée de médias

Stack Docker complète pour télécharger et gérer automatiquement vos films et séries TV avec Radarr, Sonarr, qBittorrent, Prowlarr et Plex.

## 🎯 Services inclus

- **Gluetun** : Conteneur VPN (ProtonVPN) pour sécuriser qBittorrent
- **Radarr** : Gestion et téléchargement automatique de films
- **Sonarr** : Gestion et téléchargement automatique de séries TV
- **qBittorrent** : Client torrent (via VPN Gluetun)
- **Prowlarr** : Gestionnaire d'indexers (trackers torrents)
- **FlareSolverr** : Contournement des protections Cloudflare pour les indexers
- **Plex** : Serveur de streaming média

Tous les services (sauf FlareSolverr) sont exposés via **Traefik** en HTTPS avec certificats Let's Encrypt automatiques.

## 📋 Prérequis

- Docker & Docker Compose installés
- Traefik configuré avec réseau `traefik-net` (voir `/home/Projects/Traefik`)
- **Compte ProtonVPN Plus** (requis pour le VPN et le port forwarding)
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

#### Configuration Gluetun VPN (ProtonVPN)

**Option A : WireGuard (Recommandé - Plus rapide)**

1. Allez sur https://account.protonvpn.com/downloads
2. Sélectionnez **WireGuard configuration**
3. Configurez :
   - Platform : **Router**
   - Protocol : **WireGuard**
   - Features : Cochez **NAT-PMP (Port Forwarding)**
   - VPN Accelerator : **Décochez** (important pour le port forwarding)
4. Cliquez sur **Create** et copiez la **PrivateKey**
5. Dans `.env`, configurez :
   ```bash
   VPN_TYPE=wireguard
   WIREGUARD_PRIVATE_KEY=votre_cle_privee_ici
   SERVER_COUNTRIES=United States,Canada,Netherlands
   ```

**Option B : OpenVPN (Alternative)**

1. Allez sur https://account.protonvpn.com/account
2. Copiez votre **OpenVPN username** et **password**
3. Dans `.env`, configurez :
   ```bash
   VPN_TYPE=openvpn
   OPENVPN_USER=votre_username+pmp    # IMPORTANT: Ajoutez "+pmp" à la fin !
   OPENVPN_PASSWORD=votre_password
   SERVER_COUNTRIES=United States,Canada,Netherlands
   ```

**Choix du pays serveur VPN :**
- Pour un serveur au **Canada** : `United States,Canada,Netherlands` (faible latence)
- Pour un serveur en **Europe** : `Netherlands,Switzerland,Spain` (lois favorables P2P)
- Liste complète : `docker run --rm qmcgaw/gluetun format-servers -protonvpn`

### 2. Création de la structure de dossiers

Les dossiers seront créés automatiquement au premier lancement, mais vous pouvez les créer manuellement :

```bash
mkdir -p data/movies
mkdir -p data/tv
mkdir -p data/downloads/torrents
mkdir -p data/downloads/usenet
mkdir -p gluetun/config
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

### 0. Vérification du VPN Gluetun (IMPORTANT)

Avant de configurer les autres services, vérifiez que le VPN fonctionne correctement :

```bash
# Vérifier les logs Gluetun
docker logs gluetun

# Vous devez voir :
# ✅ "VPN is running"
# ✅ "Public IP address is XXX.XXX.XXX.XXX (Pays)"
# ✅ "port forwarded is XXXXX"

# Vérifier l'IP de qBittorrent (doit être l'IP du VPN)
docker exec qbittorrent wget -qO- https://ipinfo.io/ip

# Vérifier le port forwardé
docker exec gluetun cat /tmp/gluetun/forwarded_port
```

**Résultat attendu :**
- L'IP affichée doit être **différente de votre IP réelle** (celle du VPN ProtonVPN)
- Un numéro de port doit être affiché (ex: `48768`)
- Dans qBittorrent WebUI → Options → Connection, le port doit être automatiquement configuré

**⚠️ Architecture importante :**
- **Gluetun** : Conteneur VPN qui expose le port de qBittorrent
- **qBittorrent** : Utilise le réseau de Gluetun (`network_mode: service:gluetun`)
- **Traefik** : Route vers Gluetun, qui redirige vers qBittorrent
- **Autres services** : Connexion directe (pas de VPN)

**Kill switch automatique :** Si le VPN est coupé, qBittorrent n'aura plus d'accès Internet.

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
   - Host : `gluetun` ⚠️ **Important : utiliser `gluetun` et non `qbittorrent`**
   - Port : `8080`
   - Username : `admin`
   - Password : (défini dans qBittorrent, par défaut voir logs : `docker logs qbittorrent | grep password`)
   - Category : `radarr` (recommandé)
3. **Settings → Media Management**
   - Root Folder : `/data/movies`
   - Minimum Free Space : `102400` (100 GB)

**💡 Pourquoi `gluetun` ?** Comme qBittorrent utilise le réseau de Gluetun (`network_mode: service:gluetun`), c'est Gluetun qui expose le port 8080. Radarr doit donc se connecter à `gluetun:8080` pour atteindre qBittorrent.

### 4. Sonarr - Configuration du client de téléchargement

1. Accédez à `https://sonarr.votredomaine.fr`
2. **Settings → Download Clients → Add → qBittorrent**
   - Host : `gluetun` ⚠️ **Important : utiliser `gluetun` et non `qbittorrent`**
   - Port : `8080`
   - Username : `admin`
   - Password : (même que pour Radarr)
   - Category : `sonarr` (recommandé)
3. **Settings → Media Management**
   - Root Folder : `/data/tv`
   - Minimum Free Space : `102400` (100 GB)
   - Episode Naming : Personnalisez selon vos préférences

**💡 Pourquoi `gluetun` ?** Même raison que pour Radarr : qBittorrent partage le réseau de Gluetun, donc on doit se connecter via `gluetun:8080`.

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
├── gluetun/config/           # Config Gluetun VPN
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

### Protection VPN (Gluetun)

**🛡️ Sécurité renforcée :**
- ✅ **IP masquée** : Votre IP réelle n'est jamais exposée aux trackers torrents
- ✅ **Kill switch automatique** : Si le VPN est coupé, qBittorrent perd l'accès Internet
- ✅ **Port forwarding automatique** : Configuration automatique du port dans qBittorrent
- ✅ **Chiffrement** : Tout le trafic torrent passe par le tunnel VPN chiffré
- ✅ **Isolation** : Seul qBittorrent utilise le VPN, les autres services restent en connexion directe

**⚠️ Important** : Les trackers torrents voient uniquement l'IP du VPN ProtonVPN, jamais votre IP réelle.

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
docker compose logs -f gluetun
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
- `gluetun/config/`
- `radarr/config/`
- `sonarr/config/`
- `qbittorrent/config/`
- `prowlarr/config/`
- `plex/config/`

### Commandes VPN utiles

```bash
# Vérifier l'IP du VPN
docker exec qbittorrent wget -qO- https://ipinfo.io/json

# Voir le port forwardé
docker exec gluetun cat /tmp/gluetun/forwarded_port

# Changer de pays VPN (dans .env)
SERVER_COUNTRIES=Netherlands,Switzerland

# Redémarrer le VPN
docker compose restart gluetun qbittorrent
```

## 🎬 Workflow d'utilisation

**Pour un film (Radarr) :**
1. **Ajoutez un film dans Radarr** (via recherche ou liste)
2. **Radarr recherche automatiquement** le film via les indexers Prowlarr
3. **Radarr envoie le torrent à qBittorrent**
4. **qBittorrent télécharge via le VPN** dans `/data/downloads/torrents/` 🔒
5. **Radarr importe le film** vers `/data/movies/` (hardlink)
6. **Radarr notifie Plex** qui scanne la nouvelle vidéo
7. **Le film est disponible sur Plex** pour visionnage !

**Pour une série TV (Sonarr) :**
1. **Ajoutez une série dans Sonarr** (via recherche ou liste)
2. **Sonarr recherche automatiquement** les épisodes via les indexers Prowlarr
3. **Sonarr envoie les torrents à qBittorrent**
4. **qBittorrent télécharge via le VPN** dans `/data/downloads/torrents/` 🔒
5. **Sonarr importe les épisodes** vers `/data/tv/` (hardlink)
6. **Sonarr notifie Plex** qui scanne les nouveaux épisodes
7. **La série est disponible sur Plex** pour visionnage !

**🔒 Note** : Tous les téléchargements torrents passent automatiquement par le VPN (Gluetun). Votre IP réelle n'est jamais exposée.

## ❓ Dépannage

### VPN : Gluetun ne se connecte pas

**Symptômes** : Logs montrent "Authentication failed" ou "Cannot connect"

**Solutions** :
1. **WireGuard** : Vérifiez que VPN Accelerator est bien **décoché** dans la config ProtonVPN
2. **OpenVPN** : Vérifiez que vous avez bien ajouté `+pmp` à la fin du username
3. **Pays** : Essayez un autre pays dans `SERVER_COUNTRIES`
4. **Identifiants** : Vérifiez qu'il n'y a pas d'espaces dans `.env`

### VPN : Port forwarding ne fonctionne pas

**Symptômes** : Port affiché comme "fermé" (icône rouge) dans qBittorrent

**Solutions** :
1. **Normal sans torrent** : Le port apparaît fermé tant qu'aucun torrent n'est actif. Ajoutez un torrent et attendez.
2. **Vérifier le port** : `docker exec gluetun cat /tmp/gluetun/forwarded_port`
3. **Logs Gluetun** : `docker logs gluetun | grep "port forwarded"`
4. **Redémarrer** : `docker compose restart gluetun qbittorrent`

### VPN : qBittorrent inaccessible

**Symptômes** : Impossible d'accéder à l'interface Web

**Solutions** :
1. **Vérifier Gluetun** : `docker logs gluetun` - doit montrer "VPN is running"
2. **Vérifier qBittorrent** : `docker logs qbittorrent` - doit montrer "WebUI started"
3. **Port conflict** : Le port 8081 est-il libre ? `netstat -tulpn | grep 8081`
4. **Restart** : `docker compose restart gluetun qbittorrent`

### Radarr/Sonarr : "Unable to communicate with qBittorrent"

**Symptômes** : Erreur "Resource temporarily unavailable (qbittorrent:8080)"

**Solution** : Utilisez `gluetun` comme Host au lieu de `qbittorrent`
- Settings → Download Clients → qBittorrent
- Host : `gluetun` (pas `qbittorrent`)
- Port : `8080`
- Test → Save

**Raison** : qBittorrent utilise `network_mode: service:gluetun`, donc il partage le réseau de Gluetun. Le nom `qbittorrent` n'est plus accessible directement, il faut passer par `gluetun`.

### qBittorrent : "download client places downloads in /downloads"

**Solution** : Configurez le Remote Path Mapping dans Radarr/Sonarr
- Settings → Download Clients → Remote Path Mappings
- Host : `gluetun` (pas `qbittorrent`)
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
