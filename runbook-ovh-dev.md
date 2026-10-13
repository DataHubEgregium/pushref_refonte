# Pushref v2 — monter l'environnement de dev sur OVHcloud

> Runbook pas à pas, de la création du compte OVH à un environnement qui tourne.
> egregium • AIPushTech — 13 octobre 2026

**Ce qu'on monte** : un VPS OVH à 7,21 € par mois pour l'API Python et le front React, et
une base PostgreSQL **managée** à 54,46 € pour les données, en HTTPS, redéployé
automatiquement à chaque push sur `main`. Base accessible depuis DBeaver, API testable depuis
Swagger, front testable dans le navigateur. **61,67 € HT par mois au total.**

**Durée** : une demi-journée pour la première mise en route, en suivant les étapes dans
l'ordre. Ensuite, un déploiement est un `git push`.

```
        Internet                        DBeaver
            │                               ╎ tunnel SSH
┌───────────┼─── VPS-2 · SANS ÉTAT ──┐   ┌──╎── MANAGÉ ─────────┐
│      ┌────▼────┐                   │   │  ╎                   │
│      │  Caddy  │ 80/443, TLS auto  │   │ ┌▼────────────────┐  │
│      └────┬────┘                   │   │ │ PostgreSQL 17   │  │
│   /api/*  │  tout le reste         │   │ │ Essential       │  │
│  ┌────────┴────────┐               │   │ │ pgvector        │  │
│  ▼                 ▼               │   │ │ pg_cron         │  │
│ ┌──────────┐  ┌──────────┐         │   │ │ sauvegardes     │  │
│ │ API      │  │ /srv/web │         │   │ │ managées        │  │
│ │ FastAPI  │  │ build    │         │   │ └─────▲───────────┘  │
│ └────┬─────┘  └──────────┘         │   │       │              │
│      └─────────────────────────────┼───┼───────┘              │
└────────────────────────────────────┘   └──────────────────────┘
  Seule l'IP du VPS est autorisée à joindre la base.
  Le VPS perdu se remonte en 20 min depuis le dépôt : il ne porte aucune donnée.
```

**Décisions prises, et pourquoi**

| Décision | Raison |
|---|---|
| VPS plutôt que Public Cloud pour le calcul | 7,21 € contre 20,60 € pour la même machine, et une machine de dev tourne en continu. |
| PostgreSQL managé plutôt qu'en conteneur | 54 € pour le seul composant qu'un `git push` ne reconstruit pas. L'API et le front sont sans état : un VPS perdu se remonte en vingt minutes depuis le dépôt. |
| Caddy plutôt que Nginx | Le certificat TLS s'obtient et se renouvelle tout seul, sans une ligne de configuration. |
| Images Docker sur GHCR | Gratuit avec le dépôt, intégré à GitHub Actions, rien à installer. |
| Pas de MinIO au départ | Le code actuel ne fait aucun appel de stockage de fichiers. On l'ajoutera au besoin. |

---

## Partie 0 — Prérequis

Avant de commencer, il vous faut :

1. **Un compte GitHub** avec le dépôt `pushref-v2` (privé). Le palier gratuit donne 2 000
   minutes d'Actions par mois sur dépôt privé, largement suffisant.
2. **Une paire de clés SSH** sur votre machine. Si vous n'en avez pas :
   ```bash
   ssh-keygen -t ed25519 -C "pushref-dev" -f ~/.ssh/pushref_dev
   ```
   Ne mettez **pas** de passphrase : GitHub Actions devra utiliser cette clé sans
   interaction. Vous obtenez `~/.ssh/pushref_dev` (privée) et `~/.ssh/pushref_dev.pub`
   (publique).
3. **Un nom de domaine.** Comptez une dizaine d'euros par an. Ce n'est pas du confort : Entra
   ID refuse les URI de redirection en HTTP sauf sur `localhost`, donc sans domaine et sans
   HTTPS vous ne pourrez pas brancher l'authentification Acensi. Si vous voulez vraiment
   différer l'achat, voyez la variante `sslip.io` à l'étape 4.
4. **DBeaver** installé sur votre poste.

---

## Partie 1 — Le compte OVH et le serveur

### Étape 1 — Créer le compte OVH

1. Rendez-vous sur `ovhcloud.com` et créez un compte **professionnel** au nom d'AIPushTech,
   avec le numéro de TVA intracommunautaire. Un compte particulier vous ferait refacturer la
   TVA sans pouvoir la récupérer, et le changement après coup est pénible.
2. Activez l'authentification à deux facteurs sur le compte, tout de suite. Ce compte
   portera bientôt la production.
3. Enregistrez un moyen de paiement.

### Étape 2 — Commander le VPS

1. Dans l'espace client : **Bare Metal Cloud › VPS › Commander un VPS**.
2. Choisissez le modèle **VPS-2** — 4 vCPU, 8 Go de RAM, 75 Go NVMe, 7,21 € HT par mois.
   Le VPS-1 (2 vCPU, 4 Go) tiendrait, mais 4 Go partagés entre PostgreSQL, l'API et un build
   deviennent serrés dès que vous indexez des embeddings. Prenez le VPS-2.
3. **Datacentre** : Gravelines ou Strasbourg. La donnée reste en France, ce qui compte pour
   la revue de sécurité d'une BFI.
4. **Image système** : **Debian 13**, sans panneau de contrôle. Pas de cPanel, pas de Plesk,
   rien. Tout passera par Docker.
5. **Options** : sauvegarde automatique non — vos sauvegardes utiles seront les dumps
   PostgreSQL, et le code est sur GitHub. Snapshot non. Adresse IP additionnelle non.
6. Validez. La livraison prend de quelques minutes à une heure.

> **Et le CLI OVH ?** Inutile ici, et ce n'est pas un oubli : un VPS se commande dans l'espace client puis se pilote en SSH, il n'y a aucune ressource à créer par API. Le CLI et l'API OVH deviennent utiles au passage en Public Cloud, où l'infrastructure — instances, base managée, Object Storage, Load Balancer — gagne à être décrite en Terraform plutôt que cliquée. C'est le bon moment pour les introduire, pas maintenant.

#### La base de données managée

**Public Cloud › Bases de données › Créer un service.**

| Choix | Valeur |
|---|---|
| Moteur | PostgreSQL 17 |
| Offre | **Essential**, 1 nœud, DB1-4 — 2 vCPU, 4 Go, 80 Go, 54,46 € HT/mois |
| Région | la même que le VPS |
| Réseau | public, restreint par liste d'adresses |

Une fois le service créé, relevez dans l'onglet **Informations générales** le nom d'hôte, le
**port** — OVH en utilise un non standard — et l'utilisateur d'administration `avnadmin` avec
son mot de passe. Téléchargez aussi le certificat CA du projet.

Puis trois réglages, dans cet ordre :

1. **Connexions autorisées** — ajoutez l'IP du VPS en `/32`. Tant qu'aucune adresse n'est
   déclarée, le service est injoignable, y compris par vous.
2. **Bases de données** — créez `pushref`.
3. **Utilisateurs** — créez `pushref` et relevez son mot de passe.

Essential conserve 2 jours de sauvegardes, Business en conserve 14. Deux jours suffisent en
dev, d'autant qu'on ajoute un dump quotidien à l'étape 18.

### Étape 3 — Première connexion et durcissement

Le courriel de livraison contient l'**adresse IPv4** et un **lien sécurisé** vers le mot de
passe temporaire. L'utilisateur par défaut porte le nom de la distribution : pour Debian,
c'est `debian`. Le compte root est désactivé.

```bash
# Première connexion — on vous demandera de changer le mot de passe temporaire,
# puis la session se fermera. Reconnectez-vous avec le nouveau.
ssh debian@VOTRE_IP
```

Installez votre clé publique, puis coupez l'authentification par mot de passe :

```bash
# Depuis VOTRE poste, pas depuis le serveur
ssh-copy-id -i ~/.ssh/pushref_dev.pub debian@VOTRE_IP

# Vérifiez que la clé fonctionne AVANT de couper les mots de passe
ssh -i ~/.ssh/pushref_dev debian@VOTRE_IP
```

Une fois la connexion par clé confirmée, sur le serveur :

```bash
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

> ⚠️ **Ne fermez pas votre session actuelle** avant d'avoir ouvert une seconde session dans
> un autre terminal avec la clé. C'est la seule façon de ne pas se verrouiller dehors. Si
> cela arrive quand même, le mode rescue de l'espace client OVH vous sauve.

Mise à jour, pare-feu, et protection contre le bourrage de mots de passe :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ufw fail2ban rsync

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable --now fail2ban
```

**Le port 5432 n'est jamais ouvert.** DBeaver y accédera par un tunnel SSH, ce qui est à la
fois plus sûr et plus simple que de gérer une liste d'adresses autorisées depuis une
connexion domestique à IP variable.

### Étape 4 — Le domaine et le DNS

Dans l'espace client OVH : **Web Cloud › Noms de domaine › Commander**. Prenez le domaine du
produit, par exemple `pushref.fr`.

Une fois le domaine actif, ouvrez sa **zone DNS** et ajoutez un enregistrement :

| Type | Sous-domaine | Cible |
|---|---|---|
| A | `dev` | l'IPv4 de votre VPS |

Attendez la propagation, puis vérifiez :

```bash
dig +short dev.pushref.fr
# doit retourner l'IP de votre VPS
```

> **Variante sans domaine.** Le service `sslip.io` résout n'importe quelle IP encodée dans le
> nom : si votre VPS est en `51.68.42.17`, alors `51-68-42-17.sslip.io` pointe dessus, et
> Caddy saura obtenir un certificat valide pour ce nom. C'est gratuit et immédiat, mais ce
> n'est pas une adresse que vous montrerez à Acensi, et vous devrez tout reconfigurer en
> achetant le domaine. Achetez le domaine.

---

## Partie 2 — Le socle sur le serveur

### Étape 5 — Installer Docker

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
                        docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
```

Déconnectez-vous et reconnectez-vous pour que l'appartenance au groupe prenne effet, puis
vérifiez :

```bash
docker run --rm hello-world
docker compose version
```

### Étape 6 — L'arborescence de déploiement

```bash
sudo mkdir -p /opt/pushref/{web,backups}
sudo chown -R $USER:$USER /opt/pushref
cd /opt/pushref
```

Le fichier de secrets. Il ne va **jamais** dans Git.

```bash
cat > /opt/pushref/.env <<'EOF'
# --- Identité du déploiement ---
ENV=dev
DEV_HOST=dev.pushref.fr
GH_OWNER=votre-compte-github
API_TAG=latest

# --- Base managée OVH ---
PGHOST=postgresql-xxxxxxxx.database.cloud.ovh.net
PGPORT=20184
PGDATABASE=pushref
PGUSER=pushref
PGPASSWORD=le_mot_de_passe_releve
DATABASE_URL=postgresql+psycopg://pushref:le_mot_de_passe_releve@postgresql-xxxxxxxx.database.cloud.ovh.net:20184/pushref?sslmode=require

# --- API ---
CORS_ORIGINS=https://dev.pushref.fr,http://localhost:5173

# --- Sauvegardes vers Object Storage ---
BACKUP_BUCKET=pushref-dev-backups
S3_ENDPOINT=https://s3.gra.io.cloud.ovh.net

# --- Entra ID : à remplir quand Acensi aura fourni l'inscription d'application ---
OIDC_ISSUER=
OIDC_AUDIENCE=
EOF

chmod 600 /opt/pushref/.env
```

Le `sslmode=require` n'est pas optionnel : la base est jointe par le réseau public, et sans
lui la connexion partirait en clair. Les variables `PG*` font double emploi avec l'URL, mais
elles servent aux outils en ligne de commande — `psql`, `pg_dump`, `pg_isready` — qui ne
savent pas lire une URL SQLAlchemy.

### Étape 7 — Les extensions PostgreSQL

```bash
cd /opt/pushref && set -a && . ./.env && set +a

# Une seule fois, avec le compte d'administration
docker run --rm -it postgres:17 psql \
  "postgresql://avnadmin:MOT_DE_PASSE_AVNADMIN@$PGHOST:$PGPORT/pushref?sslmode=require" \
  -c 'CREATE EXTENSION IF NOT EXISTS vector;' \
  -c 'CREATE EXTENSION IF NOT EXISTS pg_trgm;' \
  -c 'CREATE EXTENSION IF NOT EXISTS "uuid-ossp";' \
  -c 'GRANT ALL ON SCHEMA public TO pushref;' \
  -c '\dx'
```

Les extensions s'installent une seule fois, et avec `avnadmin` : le compte applicatif
`pushref` n'a pas ce droit. La dernière commande doit afficher `vector`, `pg_trgm` et
`uuid-ossp`. Si vous ajoutez une extension plus tard, passez-la en migration Alembic plutôt
qu'à la main, pour qu'une base reconstruite la retrouve.

Bonne nouvelle au passage : `pg_cron` est disponible sur la base managée. Les tâches
planifiées peuvent rester dans PostgreSQL, sans contournement par un ordonnanceur applicatif
— c'est une contrainte de moins par rapport à une base en conteneur.

### Étape 8 — Le fichier Compose

```bash
cat > /opt/pushref/docker-compose.yml <<'EOF'
name: pushref-dev

services:
  api:
    image: ghcr.io/${GH_OWNER}/pushref-api:${API_TAG}
    restart: unless-stopped
    env_file: .env
    expose:
      - "8000"

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      DEV_HOST: ${DEV_HOST}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./web:/srv/web:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - api

volumes:
  caddy_data:
  caddy_config:
EOF
```

Deux services seulement, et aucun volume de données : la base vit ailleurs. C'est tout
l'intérêt de l'option retenue — ce qui tourne sur le VPS est entièrement jetable.

### Étape 9 — Caddy

```bash
cat > /opt/pushref/Caddyfile <<'EOF'
{$DEV_HOST} {
    encode gzip zstd

    handle /api/* {
        reverse_proxy api:8000
    }

    handle {
        root * /srv/web
        try_files {path} /index.html
        file_server
    }

    log {
        output file /data/access.log
    }
}
EOF
```

`try_files {path} /index.html` est ce qui fait fonctionner le routage côté client de React :
toute URL inconnue renvoie l'application, qui décide elle-même quoi afficher.

Vérifiez que le VPS joint bien la base avant d'aller plus loin :

```bash
cd /opt/pushref && set -a && . ./.env && set +a
docker run --rm postgres:17 pg_isready -h "$PGHOST" -p "$PGPORT"
```

Un `no response` ici veut presque toujours dire que l'IP du VPS n'est pas dans la liste des
connexions autorisées de la base.

---

## Partie 3 — Le code

### Étape 10 — Le squelette de l'API Python

Dans votre dépôt GitHub, l'arborescence :

```
pushref-v2/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── db.py
│   │   └── routers/
│   ├── alembic/
│   ├── alembic.ini
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   ├── package.json
│   └── vite.config.ts
└── .github/workflows/deploy-dev.yml
```

**`backend/requirements.txt`**

```
fastapi==0.115.*
uvicorn[standard]==0.32.*
sqlalchemy==2.0.*
alembic==1.14.*
psycopg[binary]==3.2.*
pydantic-settings==2.6.*
pgvector==0.3.*
python-jose[cryptography]==3.3.*
httpx==0.28.*
```

Les versions sont épinglées sur une majeure-mineure. Un build qui change de comportement
sans que vous ayez touché au code est la pire façon de perdre une matinée.

**`backend/Dockerfile`**

```dockerfile
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**`backend/app/config.py`**

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    env: str = "dev"
    database_url: str
    cors_origins: str = ""
    oidc_issuer: str = ""
    oidc_audience: str = ""

    @property
    def cors_list(self) -> list[str]:
        return [o.strip() for o in self.cors_origins.split(",") if o.strip()]


settings = Settings()
```

**`backend/app/db.py`**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.config import settings

engine = create_engine(settings.database_url, pool_pre_ping=True)
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)


def get_session():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

**`backend/app/main.py`**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy import text

from app.config import settings
from app.db import engine

app = FastAPI(
    title="Pushref API",
    docs_url="/api/docs",
    openapi_url="/api/openapi.json",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_list,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.get("/api/health")
def health():
    with engine.connect() as conn:
        version = conn.execute(text("SHOW server_version")).scalar()
        has_vector = conn.execute(
            text("SELECT 1 FROM pg_extension WHERE extname = 'vector'")
        ).scalar()
    return {
        "status": "ok",
        "env": settings.env,
        "postgres": version,
        "pgvector": bool(has_vector),
    }
```

Tous les chemins sont préfixés par `/api`, y compris la documentation. C'est ce qui permet à
Caddy de router sur un seul critère, sans règle d'exception.

**Initialiser Alembic**, depuis `backend/` sur votre poste :

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
alembic init alembic
```

Puis, dans `alembic/env.py`, remplacez la lecture de l'URL par la variable
d'environnement :

```python
import os
config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])
```

### Étape 11 — Le front React

Création, sur votre poste :

```bash
npm create vite@latest frontend -- --template react-ts
cd frontend && npm install
```

**`frontend/.env.development`** — pour travailler en local contre l'API de dev :

```
VITE_API_URL=https://dev.pushref.fr/api
```

**`frontend/.env.production`** — pour le build déployé, où le front et l'API partagent
l'origine :

```
VITE_API_URL=/api
```

Un appel type :

```ts
const API = import.meta.env.VITE_API_URL;

export async function getHealth() {
  const r = await fetch(`${API}/health`);
  if (!r.ok) throw new Error(`API ${r.status}`);
  return r.json();
}
```

---

## Partie 4 — Le déploiement automatique

### Étape 12 — Les secrets GitHub

Dans le dépôt : **Settings › Secrets and variables › Actions › New repository secret**.

| Nom | Valeur |
|---|---|
| `SSH_HOST` | l'IPv4 du VPS |
| `SSH_USER` | `debian` |
| `SSH_KEY` | le contenu **entier** de `~/.ssh/pushref_dev`, lignes `BEGIN` et `END` comprises |
| `GHCR_PAT` | un jeton personnel GitHub avec la portée `read:packages` |
| `DEV_HOST` | `dev.pushref.fr` |

Le `GHCR_PAT` se crée dans **Settings › Developer settings › Personal access tokens ›
Tokens (classic)**, avec la seule case `read:packages`. Il sert au serveur à tirer les
images ; la publication, elle, utilise le `GITHUB_TOKEN` automatique du workflow.

### Étape 13 — Le workflow

**`.github/workflows/deploy-dev.yml`**

```yaml
name: deploy-dev

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: deploy-dev
  cancel-in-progress: true

jobs:
  api:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: ./backend
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/pushref-api:latest
            ghcr.io/${{ github.repository_owner }}/pushref-api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
        working-directory: frontend
      - run: npm run build
        working-directory: frontend
      - uses: actions/upload-artifact@v4
        with:
          name: web-dist
          path: frontend/dist

  deploy:
    needs: [api, web]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: web-dist
          path: dist

      - name: Préparer la clé SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_KEY }}" > ~/.ssh/id_deploy
          chmod 600 ~/.ssh/id_deploy
          ssh-keyscan -H ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts

      - name: Envoyer le front
        run: |
          rsync -az --delete -e "ssh -i ~/.ssh/id_deploy" \
            dist/ ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }}:/opt/pushref/web/

      - name: Migrer et basculer
        run: |
          ssh -i ~/.ssh/id_deploy ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} '
            set -e
            cd /opt/pushref
            echo "${{ secrets.GHCR_PAT }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker compose pull api
            docker compose run --rm api alembic upgrade head
            docker compose up -d
            docker image prune -f
          '

      - name: Vérifier
        run: |
          sleep 5
          curl -fsS https://${{ secrets.DEV_HOST }}/api/health
```

Trois points à retenir dans ce fichier :

- Le **`concurrency`** empêche deux déploiements de se marcher dessus si vous poussez deux
  fois de suite.
- Les migrations tournent **avant** la bascule des conteneurs, dans un conteneur jetable.
  Une migration qui échoue arrête le déploiement et laisse la version précédente en ligne.
- La **vérification finale** est ce qui transforme « le workflow est vert » en « le service
  répond ». Sans elle, vous découvrirez la panne en ouvrant le navigateur.

### Premier déploiement

```bash
git add -A && git commit -m "socle dev" && git push origin main
```

Suivez l'exécution dans l'onglet **Actions**. Au premier passage, Caddy obtient son
certificat — comptez une poignée de secondes de plus.

---

## Partie 5 — Travailler au quotidien

### Étape 14 — Connecter DBeaver par tunnel SSH

La base n'accepte que l'IP du VPS. DBeaver passe donc par un tunnel SSH à travers le VPS, ce
qui a l'avantage de marcher depuis n'importe où, sans déclarer votre adresse du moment.

**Nouvelle connexion › PostgreSQL**, onglet **Main** :

| Champ | Valeur |
|---|---|
| Host | le nom d'hôte de la base OVH |
| Port | le port OVH, non standard |
| Database | `pushref` |
| Username | `pushref` |
| Password | celui relevé à la création de l'utilisateur |

Onglet **SSL** : cochez *Use SSL*, mode `require`.

Onglet **SSH** — cochez *Use SSH Tunnel* :

| Champ | Valeur |
|---|---|
| Host/IP | l'IPv4 du VPS |
| Port | `22` |
| User Name | `debian` |
| Authentication Method | `Public Key` |
| Private Key | `~/.ssh/pushref_dev` |

Contrairement à un montage où la base tournerait sur le VPS, l'hôte à renseigner dans
l'onglet Main est bien celui de la base OVH, pas `localhost` : le tunnel sert à sortir depuis
l'IP autorisée, pas à atteindre un port local.

### Étape 15 — Tester les API

FastAPI génère sa documentation interactive tout seul. Ouvrez :

```
https://dev.pushref.fr/api/docs
```

Vous y voyez tous les points d'entrée, leurs schémas, et un bouton *Try it out* qui exécute
l'appel pour de vrai. C'est votre outil de test principal, et il reste à jour sans que
personne ne l'entretienne.

En ligne de commande :

```bash
curl -s https://dev.pushref.fr/api/health | jq
```

### Étape 16 — Tester le front

Deux modes, et ils servent à des choses différentes.

**Le front déployé**, celui que verra Acensi :

```
https://dev.pushref.fr
```

**Le front local avec rechargement à chaud**, contre l'API de dev — c'est celui que vous
utiliserez 95 % du temps :

```bash
cd frontend && npm run dev
# http://localhost:5173
```

Ce second mode ne marche que parce que `http://localhost:5173` figure dans `CORS_ORIGINS`.
Si vous obtenez une erreur CORS dans la console du navigateur, c'est là qu'il faut
regarder, et il faut redémarrer l'API après modification :

```bash
ssh debian@VOTRE_IP 'cd /opt/pushref && docker compose up -d --force-recreate api'
```

### Étape 17 — Les commandes de survie

```bash
cd /opt/pushref && set -a && . ./.env && set +a

docker compose ps                    # qui tourne
docker compose logs -f api           # les logs de l'API en direct
docker compose logs --tail=100 caddy # pourquoi le certificat ne sort pas
docker compose restart api

# psql sur la base managée
docker run --rm -it postgres:17 psql \
  "postgresql://$PGUSER:$PGPASSWORD@$PGHOST:$PGPORT/$PGDATABASE?sslmode=require"

# Remise à zéro volontaire, par exemple après une refonte de schéma
docker compose run --rm api alembic downgrade base
docker compose run --rm api alembic upgrade head
docker compose run --rm api python -m seeds.load
```

### Étape 18 — La base de dev, et comment elle survit

Cette base sert tout le dev : les référentiels chargés, les jeux d'essai, les parcours construits au fil des tranches. Elle n'est pas jetable. Ce qui est jetable, c'est le contenu de la base Supabase actuelle — comptes, progression, conversations — pas celle-ci.

Deux protections, et la première compte plus que la seconde.

#### Reconstructible par conception

Tout ce qui définit la base vit dans Git : les migrations Alembic pour le schéma, et un dossier `seeds/` pour les référentiels — arbres de compétences, sondes, catalogue, glossaire — en fichiers versionnés chargés par une commande.

```
backend/
├── alembic/versions/      # le schéma
└── seeds/
    ├── 001_competences.yaml
    ├── 002_sondes.yaml
    └── load.py
```

Une base perdue se remonte alors en deux commandes, sans rien attendre d'une sauvegarde :

```bash
docker compose run --rm api alembic upgrade head
docker compose run --rm api python -m seeds.load
```

C'est la vraie protection, et elle a un corollaire : **un référentiel qui n'existe que dans la base est un référentiel que vous perdrez un jour.** Tout ce qui est chargé à la main devrait finir en seed versionné.

#### Sauvegardée hors de la machine

La base managée sauvegarde déjà toute seule, et conserve 2 jours sur l'offre Essential. On y
ajoute un dump quotidien copié dans l'Object Storage, pour deux raisons : une rétention plus
longue que deux jours, et la **portabilité** — un dump se restaure chez n'importe quel
hébergeur, une sauvegarde managée ne sort pas de chez OVH.

Créez un conteneur dans **Public Cloud › Object Storage**, classe standard, puis un
utilisateur S3 dans **Users & Roles**. Les variables `BACKUP_BUCKET` et `S3_ENDPOINT` sont
déjà dans le `.env` de l'étape 6. Le dump se prend depuis le VPS, puisque c'est lui qui a le
droit de joindre la base.

```bash
sudo apt install -y awscli
aws configure   # les clés de l'utilisateur S3 OVH

cat > /opt/pushref/backup.sh <<'EOF'
#!/bin/bash
set -euo pipefail
cd /opt/pushref
set -a && . ./.env && set +a

OUT="/opt/pushref/backups/pushref-$(date +%F).sql.gz"

docker run --rm -e PGPASSWORD="$PGPASSWORD" postgres:17 \
  pg_dump -h "$PGHOST" -p "$PGPORT" -U "$PGUSER" "$PGDATABASE" | gzip > "$OUT"

# la copie hors OVH-manage : c'est elle qui rend le dump portable
aws s3 cp "$OUT" "s3://$BACKUP_BUCKET/" --endpoint-url "$S3_ENDPOINT"

find /opt/pushref/backups -name '*.sql.gz' -mtime +14 -delete
EOF

chmod +x /opt/pushref/backup.sh
( crontab -l 2>/dev/null; \
  echo "0 3 * * * /opt/pushref/backup.sh >> /opt/pushref/backups/cron.log 2>&1" ) | crontab -
```

Quelques dizaines de mégaoctets de dumps coûtent moins d'un centime par mois. Vérifiez le
lendemain que le fichier est bien arrivé dans le conteneur : une sauvegarde jamais vérifiée
n'est pas une sauvegarde.

Restauration :

```bash
cd /opt/pushref && set -a && . ./.env && set +a
gunzip -c /opt/pushref/backups/pushref-2026-10-20.sql.gz \
  | docker run --rm -i -e PGPASSWORD="$PGPASSWORD" postgres:17 \
      psql -h "$PGHOST" -p "$PGPORT" -U "$PGUSER" "$PGDATABASE"
```

---

## Annexe A — Dépannage

| Symptôme | Cause la plus fréquente | Vérification |
|---|---|---|
| Caddy n'obtient pas de certificat | Le DNS ne pointe pas encore sur le VPS, ou le port 80 est fermé | `dig +short dev.pushref.fr` et `sudo ufw status` |
| L'API ne joint pas la base | L'IP du VPS absente des connexions autorisées, ou `sslmode=require` oublié | Le `pg_isready` de l'étape 9, puis l'onglet Connexions autorisées |
| `permission denied for schema public` | Le `GRANT` de l'étape 7 n'a pas été passé | Rejouer la commande avec `avnadmin` |
| DBeaver reste bloqué | Tunnel SSH non coché, ou l'hôte mis à `localhost` au lieu du nom OVH | Les deux onglets de la connexion |
| Erreur CORS dans le navigateur | L'origine locale absente de `CORS_ORIGINS` | La console du navigateur donne l'origine refusée |
| Le déploiement échoue sur `docker login` | `GHCR_PAT` sans la portée `read:packages` | Régénérer le jeton |
| `vector` introuvable | Extensions créées sur la mauvaise base, ou pas avec `avnadmin` | `\dx` sur `pushref` |
| 404 sur une route du front | `try_files` absent du Caddyfile | Comparer avec l'étape 9 |

## Annexe B — Ce qui changera au passage en production

Rien de ce que vous écrivez ici n'est à jeter. Ce qui bouge :

| Aujourd'hui en dev | En production |
|---|---|
| Base managée Essential, 1 nœud, 2 jours de sauvegarde | Business 2 nœuds, réplique et bascule automatique, 14 jours |
| Base jointe par le réseau public restreint par IP | Base en vRack, injoignable depuis Internet |
| Une seule machine, un VPS | 2 à 3 instances Public Cloud derrière un Load Balancer |
| Caddy sur la machine | Load Balancer OVH avec certificat, Caddy conservé derrière |
| Dossier `web` servi depuis le disque | Object Storage + CDN |
| `.env` sur le serveur | Gestionnaire de secrets |
| Infrastructure créée à la main dans l'espace client | Décrite en Terraform, avec le fournisseur OVH |

Le passage en Public Cloud est aussi le moment où le CLI et l'API OVH prennent leur sens :
des instances qui se créent et se détruisent gagnent à être décrites en code, un VPS unique
non.

Les deux seules choses à concevoir dès maintenant parce qu'elles coûtent cher à rattraper
sont le **`tenant_id` sur chaque table cloisonnée** et le **fait que l'API n'identifie jamais
le client autrement que par l'émetteur du jeton**.

## Annexe C — Checklist de mise en route

- [ ] Compte OVH professionnel, 2FA activée
- [ ] VPS-2 commandé en Debian 13, datacentre français
- [ ] Connexion par clé SSH vérifiée, mot de passe désactivé
- [ ] `ufw` actif : 22, 80, 443 uniquement
- [ ] Base managée Essential créée, IP du VPS autorisée, base et utilisateur `pushref` créés
- [ ] Domaine acheté, enregistrement A `dev` qui résout
- [ ] Docker installé, `hello-world` passe
- [ ] `/opt/pushref` créé, `.env` rempli, droits `600`
- [ ] `pg_isready` passe depuis le VPS et `\dx` montre `vector`
- [ ] Dépôt GitHub avec `backend/` et `frontend/`
- [ ] Cinq secrets GitHub créés
- [ ] Premier push : workflow vert et `/api/health` répond
- [ ] DBeaver connecté par tunnel
- [ ] `/api/docs` s'ouvre dans le navigateur
- [ ] `npm run dev` en local atteint l'API de dev
- [ ] Seeds des référentiels versionnés dans `backend/seeds/`
- [ ] Sauvegarde quotidienne en place, et un premier dump vérifié dans l'Object Storage

---

**Coût mensuel : 7,21 € pour le VPS et 54,46 € pour la base, soit 61,67 € HT**, plus une
dizaine d'euros par an pour le domaine.
