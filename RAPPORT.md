# TP2 : Déploiement d'un Portfolio Next.js avec Coolify et Forgejo

**Étudiante :** Nour Kidoudi
**Formation :** Master DevOps et Cloud Computing, ISET Tozeur
**Date de réalisation :** 20 et 21 octobre 2026

---

## 1. Objectif du TP

Déployer le portfolio Next.js du TP1 avec une approche **PaaS auto-hébergée** :

- **Forgejo** héberge le dépôt Git et émet un webhook à chaque `git push` ;
- **Coolify** clone, construit, déploie et expose l'application (proxy Traefik intégré) ;
- le tout s'exécute **en local**, dans une machine virtuelle, sans domaine DNS public ni compte GitHub.

## 2. Environnement

| Élément | Valeur |
|---|---|
| VM | `vm-admin`, Ubuntu 24.04 (VM déjà utilisée pour Ansible et Semaphore au TP1) |
| IP de la VM | `192.168.237.135` |
| Docker | 29.1.3 |
| Coolify | v4.4.6 (dashboard sur le port 8000) |
| Proxy | Traefik v3.7 (conteneur `coolify-proxy`, port 80) |
| Forgejo | image `data.forgejo.org/forgejo/forgejo:16` (web sur 3001, SSH sur 2222) |
| Build | Nixpacks v1.41.0, Node 22, Next.js 16.4.0 |
| Nom d'hôte local | `portfolio.coolify.local` (fichier `hosts` du poste) |

Pour ne pas perdre le travail du TP1 (consultable ensuite par l'enseignant), la VM a été réutilisée en prenant un instantané au préalable et en vérifiant que le port 80 était libre pour le proxy de Coolify.

## 3. Réalisation

### Partie A : Installation de Coolify

Installation avec le script officiel, puis création du compte administrateur, choix du serveur **localhost** et démarrage du proxy. Le serveur apparaît en état **Ready** et le proxy `coolify-proxy` est **Healthy**.

> Capture : `captures/coolify-dashboard.png`

### Partie B : Forgejo et dépôt du portfolio

- Copie du `docker-compose.yml` fourni dans la VM et lancement de Forgejo avec `docker compose up -d`.
- Le réglage `FORGEJO__webhook__ALLOWED_HOST_LIST=external,private` est indispensable : par défaut, Forgejo refuse d'appeler une adresse privée avec un webhook.
- Assistant d'installation : SQLite3, domaine `192.168.237.135`, URL de base `http://192.168.237.135:3001/`.
- Création du dépôt public `portfolio`, puis push du code Next.js (copie propre du projet, sans `node_modules`, `.next` ni `.env.local`).

> Capture : `captures/forgejo-repo.png`

### Partie C : Déploiement du portfolio

Ressource **Public Git Repository** dans le projet `portfolio` (environnement `production`) :

| Paramètre | Valeur |
|---|---|
| URL du dépôt | `http://192.168.237.135:3001/nour/portfolio.git` (l'IP, car Coolify ne connaît pas le fichier `hosts`) |
| Branche | `main` |
| Build pack | Nixpacks |
| Port exposé | 3000 |
| Domaine | `http://portfolio.coolify.local` (protocole `http`, pas de certificat) |

Le premier déploiement s'est terminé avec le statut **Finished** (`Rolling update completed`). Le site est accessible sur `http://portfolio.coolify.local`.

> Captures : `captures/coolify-config.png`, `captures/coolify-deploy.png`, `captures/portfolio.png`

### Partie D : Redéploiement automatique (webhook Forgejo vers Coolify)

- **Coolify** : onglet *Webhooks*, section *Gitea*, saisie d'un secret de plus de 16 caractères ; option *Auto deploy* activée.
- **Forgejo** : webhook de type Forgejo, contenu `application/json`, même secret, événement de soumission (push), filtre de branche `main`.
- L'URL générée par Coolify utilisait l'IP publique détectée ; elle a été remplacée par l'IP locale de la VM : `http://192.168.237.135:8000/webhooks/source/gitea/events/manual`.
- La fonction **Tester l'envoi** de Forgejo renvoie une livraison réussie (coche verte).
- Test réel : modification du titre du portfolio, puis `git push forgejo main`. Un nouveau déploiement démarre sans intervention dans Coolify et la modification apparaît sur le site.

> Captures : `captures/forgejo-webhook.png`, `captures/coolify-autodeploy.png`

### D.4 Rollback et supervision

**Rollback**

1. Push volontaire d'une erreur de build : le déploiement automatique passe en **Failed** et le site reste en ligne avec la version précédente (Coolify ne remplace le conteneur que si le build réussit).
2. Retour à la version précédente depuis le menu **Rollback** de l'application.
3. Correction du dépôt, nouveau push, déploiement réussi.

> À compléter : résultat observé et capture `captures/coolify-rollback.png`.

**Supervision**

- Logs de l'application (*Runtime Logs*) et *Metrics* dans Coolify ;
- `docker ps` et `docker stats --no-stream` dans la VM.

> À compléter : observations (consommation CPU/mémoire du conteneur, état des services).

## 4. Comparaison PaaS (Coolify) et IaC (Ansible / Semaphore, TP1)

| Critère | Coolify (PaaS) | Ansible / Semaphore (IaC) |
|---|---|---|
| Mise en place | Installation guidée, configuration par interface web | Rôles et playbooks à écrire et à maintenir |
| Déploiement | Clone, build et déploiement pris en charge par l'outil | Chaque étape (paquets, build, service, proxy) décrite dans le code |
| Redéploiement | Automatique à chaque `git push` grâce au webhook | Lancement manuel ou planifié depuis Semaphore |
| Rollback | Intégré : retour à une version précédente en quelques clics | À prévoir explicitement dans le playbook |
| Proxy et routage | Traefik intégré, routage par nom d'hôte | Nginx à configurer soi-même |
| Contrôle fin | Limité aux options proposées par l'outil | Total sur le serveur (système, paquets, configuration) |
| Reproductibilité | Dépend de l'état de l'instance Coolify | Élevée : l'infrastructure est décrite dans le code versionné |
| Courbe d'apprentissage | Faible | Plus élevée |
| Ressources consommées | Plus lourd (Coolify, base de données, proxy, builds dans la VM) | Léger côté serveur |

**Conclusion :** Coolify est plus rapide à mettre en place et apporte directement le redéploiement automatique et le rollback. Ansible reste préférable lorsqu'on veut une infrastructure entièrement décrite par du code, reproductible et maîtrisée dans le détail. Les deux approches sont complémentaires : l'IaC pour préparer les serveurs, le PaaS pour déployer les applications.

## 5. Difficultés rencontrées

| # | Difficulté | Cause | Solution |
|---|---|---|---|
| 1 | L'installation de Coolify échoue à l'étape 8/9 : `Failed to parse docker-compose configuration` | Le plugin `docker compose` n'était pas installé sur la VM | Installation du plugin Compose, puis relance de l'installation |
| 2 | `connection reset by peer` pendant le téléchargement des images | Réseau instable sur la VM | Relance de la commande `docker compose up -d` (les couches déjà téléchargées sont conservées) |
| 3 | `network coolify declared as external, but could not be found` | Le script d'installation s'était arrêté avant la création du réseau | `docker network create --attachable coolify` |
| 4 | `permission denied` sur le socket Docker | L'utilisateur n'était pas dans le groupe `docker` | Ajout au groupe `docker` (ou utilisation de `sudo`) |
| 5 | Image `codeberg.org/forgejo/forgejo:16` injoignable | Registre inaccessible depuis la VM | Utilisation de `data.forgejo.org/forgejo/forgejo:16`, prévu dans le fichier fourni |
| 6 | Erreur « DNS is not configured for portfolio.coolify.local » | Coolify vérifie le domaine via un DNS public, alors que le nom n'existe que dans le fichier `hosts` | Désactivation de la validation DNS dans les réglages de Coolify |
| 7 | Premier déploiement très long (environ 50 minutes) | Téléchargements lents : image helper de Coolify, paquets Nix, image de build | Patience, le déploiement continue côté serveur ; les suivants sont plus rapides grâce au cache |
| 8 | Webhook Forgejo en échec (triangle rouge) | URL générée par Coolify avec l'IP publique détectée | Remplacement par l'IP locale de la VM dans l'URL du webhook |

**Autres observations (non bloquantes) :** avertissements `EBADENGINE` et `eslint deprecated` pendant `npm ci`, message `5 high severity vulnerabilities` de `npm audit` (dépendances de développement), avertissement Docker `UndefinedVar: NIXPACKS_PATH`.

## 6. Livrables

```
tp2-coolify-forgejo/
├── captures/
│   ├── coolify-dashboard.png
│   ├── forgejo-repo.png
│   ├── coolify-config.png
│   ├── coolify-deploy.png
│   ├── portfolio.png
│   ├── forgejo-webhook.png
│   ├── coolify-autodeploy.png
│   └── coolify-rollback.png
└── RAPPORT.md
```
