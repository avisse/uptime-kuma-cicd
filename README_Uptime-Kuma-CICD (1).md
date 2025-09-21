# Projet DevOps — CI/CD pour déployer **Uptime Kuma** sur une VM Debian (runner self‑hosted)

> **Rôle** : Ingénieur/DevOps  
> **Contexte** : Automatiser le déploiement d’une application Dockerisée (**Uptime Kuma**) sur une VM **Debian 12** isolée, avec **GitHub Actions** et un **runner auto‑hébergé** (self‑hosted) exécutant les jobs **localement** sur la VM.  
> **Objectif** : Un `git push` → déclenche un pipeline → met à jour/redémarre proprement le container Uptime Kuma → persistance des données → traçabilité via Git.

---

## 1) Contexte & Enjeux

J’ai mis en place un pipeline CI/CD complet pour démontrer :
- l’**automatisation fiable** des déploiements (push → déploiement),
- la **sécurisation des accès** (SSH, secrets chiffrés GitHub),
- la **traçabilité** (historique Git + logs d’exécutions),
- la **gestion de services Docker** sur une VM locale (ports, volumes, redémarrage idempotent),
- la **reproductibilité** (dépôt Git, fichiers infra-as-code).

L’application cible est **Uptime Kuma** (supervision de services, pings HTTP/ICMP, alertes).

---

## 2) Architecture (vue globale)

```
Poste dev (Git) ──push──▶ GitHub (repo)
                        └─▶ GitHub Actions (workflow)
                               └─▶ Runner self-hosted (VM Debian)
                                        ├─ Docker / Docker Compose
                                        └─ Conteneur Uptime Kuma (port 3010 → 3001)
```

- **Repo GitHub** : code, `docker-compose.yml`, workflow `.github/workflows/deploy.yml`.
- **Runner self-hosted** : installé **sur la VM** pour exécuter localement les jobs.
- **Uptime Kuma** : conteneur Docker, données persistées via volume.

---

## 3) Arborescence du dépôt

```
uptime-kuma-cicd/
├─ docker-compose.yml
├─ .github/
│  └─ workflows/
│     └─ deploy.yml
└─ README.md  (ce fichier)
```

---

## 4) Pré‑requis

- VM **Debian 12** (ou équivalent) avec :
  - **Docker + Docker Compose** installés et fonctionnels,
  - accès SSH (port 22) et un utilisateur (ex. `vboxuser`),
  - ports libres sur l’hôte (ex. `3010`).
- Un **dépôt GitHub**.
- Un **runner GitHub Actions self-hosted** **installé sur la VM** (voir §8).

---

## 5) Docker Compose (application & persistance)

Fichier : `docker-compose.yml`

```yaml
version: "3"

services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3010:3001"       # 3010 (VM) → 3001 (container)
    volumes:
      - ./data:/app/data  # persistance des données (config, états)
```

- **Pourquoi** : garder Uptime Kuma simple, avec un volume local `./data` pour **ne pas perdre** l’état (sondes / historiques) lors d’un redéploiement.
- **Pourquoi 3010** : éviter les conflits (ex. 3001 déjà utilisé par une autre app).

---

## 6) Sécurisation SSH (clé + autorisations)

Sur **mon poste**, génération d’une clé dédiée aux automatisations :

```bash
ssh-keygen -t ed25519 -C "github-actions-uptime"
# -> génère id_ed25519 (privée) et id_ed25519.pub (publique)
```

Sur la **VM**, j’autorise la clé publique :

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

> **Pourquoi** : le runner peut cloner/puller et déployer **sans mot de passe** tout en restant **sécurisé**.

---

## 7) Secrets GitHub (sécuriser les infos sensibles)

Dans **GitHub → Repo → Settings → Secrets and variables → Actions**, j’ai créé :

| Nom         | Exemple               | Rôle                                      |
|-------------|-----------------------|-------------------------------------------|
| `VM_HOST`   | `192.168.56.101`      | IP locale de la VM                        |
| `VM_USER`   | `vboxuser`            | Utilisateur Linux de la VM                |
| `VM_SSH_KEY`| *contenu de la clé*   | **Clé privée** (id_ed25519) **complète**  |

> Ces secrets sont **chiffrés** et **jamais** visibles dans les logs.

---

## 8) Runner GitHub Actions **self‑hosted** (sur la VM)

1. Télécharger le runner depuis l’UI GitHub (Repository → Settings → Actions → Runners → New self‑hosted runner).
2. Sur la VM :

```bash
# Exemple (les URLs et tokens sont fournis par GitHub lors de l’ajout du runner)
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -o actions-runner-linux-x64.tar.gz -L https://github.com/actions/runner/releases/download/vX.Y.Z/actions-runner-linux-x64-X.Y.Z.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

# Configuration
./config.sh --url https://github.com/<org>/<repo> --token <TOKEN_FOURNI>

# Lancement
./run.sh
```

> **Astuce** : Installer le runner en **service** (`./svc.sh install && ./svc.sh start`) pour qu’il survive aux redémarrages.

---

## 9) Workflow GitHub Actions (déploiement)

Fichier : `.github/workflows/deploy.yml`

```yaml
name: Deploy Uptime Kuma to VM

on:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Déployer Uptime Kuma (docker compose)
        run: |
          cd ~/projet-ecf/uptime-kuma-cicd
          docker compose pull
          docker compose up -d
```

- **Pourquoi `runs-on: self-hosted`** : le job s’exécute **directement sur la VM**, donc accès natif à Docker.
- **Pourquoi `pull` puis `up -d`** : pour **récupérer** la dernière image et **redéployer** sans downtime inutile.

> **Alternative** (si vous ne souhaitez pas de runner self-hosted) : exécuter des commandes **SSH** depuis GitHub Actions (via `appleboy/ssh-action`), mais cela expose plus de surface de configuration.

---

## 10) Déroulé d’un déploiement (du push à la prod locale)

1. Je modifie `docker-compose.yml` (ou la config du repo).
2. `git add . && git commit -m "Update" && git push`.
3. GitHub déclenche le workflow **sur `main`**.
4. Le **runner self-hosted** récupère le job et exécute :
   - `docker compose pull` (MAJ image),
   - `docker compose up -d` (recréation si nécessaire).
5. Uptime Kuma est disponible sur **`http://<VM_HOST>:3010`**.

---

## 11) Validation post-déploiement

Sur la VM :

```bash
docker ps
# vérifier que 'uptime-kuma' tourne

docker logs -f uptime-kuma
# vérifier qu'il démarre correctement
```

Dans le navigateur : `http://<VM_HOST>:3010`

Configurer vos **sondes** (GLPI, Rocket.Chat, Heimdall, etc.) :
- ping HTTP/HTTPS,
- fréquence (ex : 60s),
- notifications, etc.

---

## 12) Problèmes rencontrés & solutions

- **Conflit de ports** : 3001 utilisé par une autre app → j’ai exposé **3010** côté hôte.  
- **Droits SSH** : `~/.ssh` doit être en `700` et `authorized_keys` en `600`.  
- **Chemins VM** : le `cd` dans le workflow doit **pointer** vers le **vrai** dossier du repo sur la VM.  
- **Runner éphémère** : j’ai installé le runner en **service** pour éviter d’avoir à relancer `./run.sh` après reboot.  

---

## 13) Sécurité & bonnes pratiques

- Clés SSH **dédiées** aux automatisations (révocation simple en cas d’incident).
- Secrets **GitHub** pour toute donnée sensible (jamais en clair dans le repo).
- `restart: unless-stopped` pour relancer le conteneur si la VM redémarre.
- Volumes **persistants** pour **ne pas perdre** l’état d’Uptime Kuma.
- Runner **isolé** sur VM locale non exposée publiquement.

---

## 14) Extensions possibles

- Ajout d’un **Image Renderer**/export PDF de rapports (via sidecar si besoin).
- Provisioning automatique de sondes (script API d’Uptime Kuma).
- Ajout d’étapes **lint/test** dans le pipeline avant déploiement.
- Monitoring du **runner** lui‑même (sonde Uptime Kuma vers `:8080` etc.).

---

## 15) Commandes utiles (mémo)

```bash
# Démarrer/Mettre à jour
docker compose pull && docker compose up -d

# Voir les conteneurs
docker ps

# Logs Uptime Kuma
docker logs -f uptime-kuma

# Arrêter
docker compose down

# Purger l’image (optionnel)
docker rmi louislam/uptime-kuma:latest
```

---

## 16) Licence / Crédits

- **Uptime Kuma** : https://github.com/louislam/uptime-kuma
- **GitHub Actions** : https://docs.github.com/actions
- **Docker** : https://docs.docker.com/

---

### TL;DR

- Un **push** sur `main` déclenche un **workflow**,
- exécuté par un **runner self‑hosted** sur la **VM**,
- qui **déploie/met à jour** Uptime Kuma via **Docker Compose**,
- avec **données persistantes**, **ports maîtrisés**, **sécurité SSH/Secrets**,
- et une **traçabilité complète** des déploiements.
