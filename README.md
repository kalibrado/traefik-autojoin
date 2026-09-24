# traefik-autojoin

Reverse proxy **Traefik v3** qui découvre tout seul les conteneurs d'un serveur Docker
et leur attribue un nom d'hôte (`<service>.<domaine>`), sans rien déclarer application
par application. Un petit conteneur (`traefik-autojoin`) connecte Traefik aux réseaux
créés par les autres stacks Compose, et une stack **dnsmasq** optionnelle publie les
noms sur tout le LAN.

Conçu pour un homelab avec **plusieurs serveurs Docker** (prod, dev, test, ...) pilotés,
par exemple, depuis Portainer.

## Sommaire

- [Principe](#principe)
- [Prérequis](#prérequis)
- [Installation](#installation)
- [Un domaine par serveur](#un-domaine-par-serveur)
- [Publier une application](#publier-une-application)
- [DNS avec dnsmasq](#dns-avec-dnsmasq)
- [Utiliser les noms depuis Kasm Workspace](#utiliser-les-noms-depuis-kasm-workspace)
- [Utiliser les noms sur tout le réseau](#utiliser-les-noms-sur-tout-le-réseau)
- [Dashboard](#dashboard)
- [Dépannage](#dépannage)
- [Commandes utiles](#commandes-utiles)
- [Limites et sécurité](#limites-et-sécurité)

## Principe

```
   Client (Chrome dans Kasm, PC, ...)
        │  1. jellyfin.local ?
        ▼
   dnsmasq (192.168.1.5)
        │  2. *.local  →  192.168.1.113
        ▼
   Traefik (serveur prod, port 80)
        │  3. Host(`jellyfin.local`)  →  conteneur jellyfin
        ▼
   jellyfin
```

Deux rôles distincts :

| Rôle | Où | Ce qu'il fait |
|---|---|---|
| **DNS** (dnsmasq) | une seule machine du LAN | Envoie chaque suffixe (`local`, `dev.local`, ...) vers le bon serveur Docker |
| **Traefik + autojoin** | un par serveur Docker | Découvre les conteneurs locaux et route selon le nom d'hôte |

Un seul Traefik central ne pourrait pas découvrir les conteneurs des autres serveurs
sans exposer leur socket Docker en TCP, ce qui est risqué. D'où un Traefik par serveur.

### Ce que fait la stack Traefik

- `traefik` : reverse proxy Traefik v3, écoute sur le port `80` (HTTP).
- `traefik-autojoin` : surveille les événements Docker et connecte `traefik` à chaque
  nouveau réseau de type `bridge`. Les réseaux `bridge`, `host`, `none` et `proxy`
  sont ignorés.
- Les conteneurs sont exposés automatiquement (`exposedByDefault=true`).
- La règle par défaut est `Host(<nom-du-service>.<DOMAIN>)`, où le nom du service est
  celui de la clé sous `services:` dans le fichier Compose (label
  `com.docker.compose.service`). Pour un conteneur lancé avec `docker run`, c'est le
  nom du conteneur qui est utilisé.
- Le réseau `proxy` est créé avec un nom fixe. Il est utilisé comme réseau par défaut
  de Traefik ; les applications n'ont pas besoin de le rejoindre.

## Prérequis

- Docker Engine avec le plugin Docker Compose **v2.23 ou plus récent** (nécessaire pour
  `configs: content:` utilisé par la stack dnsmasq).
- Le port `80` libre sur chaque serveur Docker (pas d'autre reverse proxy dessus).
- Le port `8181` libre pour l'accès direct au dashboard.
- Pour dnsmasq : le port `53` (UDP et TCP) libre sur la machine DNS, et une **adresse IP
  fixe** (ou une réservation DHCP) pour chaque serveur Docker.

## Installation

À faire sur **chaque serveur Docker**.

### Avec Docker Compose

1. Créer le fichier `.env` à partir du modèle :

   ```bash
   cp .env.example .env
   ```

2. Choisir le domaine de ce serveur dans `.env` (voir
   [Un domaine par serveur](#un-domaine-par-serveur)) :

   ```
   DOMAIN=local
   ```

3. Démarrer :

   ```bash
   docker compose up -d
   ```

4. Vérifier :

   ```bash
   docker compose ps
   docker compose logs -f traefik-autojoin
   ```

Le dashboard est alors disponible sur `http://traefik.<DOMAIN>` (si le DNS est en place)
ou directement sur `http://<ip-du-serveur>:8181/dashboard/`.

### Avec Portainer

1. Sélectionner l'**environnement** du serveur (menu en haut de Portainer).
2. **Stacks → Add stack**, puis soit coller le contenu de `docker-compose.yml` dans
   l'éditeur web, soit choisir **Repository** avec l'URL de ce dépôt.
3. Dans **Environment variables**, ajouter `DOMAIN` avec la valeur du serveur.
   Le fichier `.env` n'est pas utilisé par Portainer : les variables se renseignent
   dans l'interface.
4. **Deploy the stack**.

La même stack se déploie sur chaque environnement, seule la valeur de `DOMAIN` change.

## Un domaine par serveur

Chaque serveur Docker reçoit son propre suffixe. Le DNS envoie tout `*.<suffixe>` vers
lui, donc aucune ligne n'est à ajouter quand une application apparaît.

| Serveur | IP | `DOMAIN` | Exemple |
|---|---|---|---|
| prod | 192.168.1.113 | `local` | `jellyfin.local` |
| dev | 192.168.1.43 | `dev.local` | `jellyfin.dev.local` |
| test | 192.168.1.42 | `test.local` | `jellyfin.test.local` |
| kasm | 192.168.1.5 | `kasm.local` | `homepage.kasm.local` |

Adaptez ces valeurs à votre réseau. Si `DOMAIN` n'est pas défini, `local` est utilisé.

> **Attention à `.local`.** Ce suffixe est aussi utilisé par mDNS (RFC 6762). Selon le
> client, la résolution peut partir en multicast au lieu d'interroger votre DNS, ce qui
> donne des résultats capricieux. Si vous constatez ce comportement, choisissez un autre
> suffixe (`home`, `lab`, `test`, ou `home.arpa` pour un suffixe officiellement réservé
> aux réseaux domestiques) et utilisez **la même valeur** dans `DOMAIN` et dans dnsmasq.
> Évitez `.dev` : c'est un vrai domaine Internet qui force le HTTPS dans Chrome.
>
> De plus, la règle `address=/local/...` de dnsmasq répond à tout nom en `.local` reçu
> par le DNS, y compris ceux d'appareils mDNS réels (imprimante, Chromecast...).

## Publier une application

Une application Compose ordinaire n'a **rien à ajouter** : elle garde son propre réseau,
`traefik-autojoin` y connecte Traefik, et le service est joignable sous son nom.

Par exemple, un service nommé `whoami` est accessible à `http://whoami.local` sur le
serveur prod. Le sous-domaine dépend du **nom du service** dans le fichier Compose, pas
de `container_name` ni du nom de la stack.

### Quand faut-il un label de port ?

Traefik détecte le port du conteneur seul quand il n'en expose qu'un. Il faut indiquer
le port quand l'image en expose plusieurs, ou aucun :

```yaml
services:
  filebrowser:
    image: filebrowser/filebrowser
    ports:
      - 192.168.1.113:8081:80
    labels:
      - traefik.http.services.filebrowser.loadbalancer.server.port=80
```

Le port du label est **toujours le port interne du conteneur** (celui de droite dans
`ports`), jamais le port publié sur l'hôte.

### Ne pas exposer un conteneur

```yaml
    labels:
      - traefik.enable=false
```

Comme Traefik est connecté à tous les réseaux, les bases de données et autres services
internes reçoivent aussi une route (`db.local`, `redis.local`...). C'est sans danger sur
un réseau de confiance, mais vous pouvez les masquer avec ce label, ou passer
`--providers.docker.exposedByDefault=false` dans le `command` de Traefik et n'ajouter
`traefik.enable=true` que sur les applications voulues.

Deux services portant le même nom dans des stacks différentes du même serveur produisent
la même règle. Donnez-leur des noms distincts, ou forcez une règle :

### Règle personnalisée

```yaml
    labels:
      - traefik.enable=true
      - traefik.http.routers.whoami.rule=Host(`app.local`)
      - traefik.http.services.whoami.loadbalancer.server.port=80
```

Les labels écrits en dur avec un ancien domaine sont à mettre à jour à la main si vous
changez de suffixe.

## DNS avec dnsmasq

La stack `docker-compose.dnsmasq.yml` est **distincte** : à lancer uniquement sur la
machine qui sert de DNS au LAN. Elle utilise `network_mode: host` et écoute sur le
port 53 (UDP et TCP).

### Configuration

Les valeurs se définissent dans le `.env` de la machine DNS (ou dans les variables
d'environnement de Portainer) :

```
DNSMASQ_LISTEN_ADDRESS=192.168.1.5
PROD_HOST_IP=192.168.1.113
DEV_HOST_IP=192.168.1.43
TEST_HOST_IP=192.168.1.42
KASM_HOST_IP=192.168.1.5
```

`DNSMASQ_LISTEN_ADDRESS` doit être l'adresse LAN fixe de la machine DNS. Les autres
variables sont les adresses des serveurs Docker.

Le fichier de configuration généré associe un suffixe à un serveur :

```
address=/local/192.168.1.113
address=/dev.local/192.168.1.43
address=/test.local/192.168.1.42
address=/kasm.local/192.168.1.5
```

dnsmasq applique **la règle la plus spécifique** : `jellyfin.dev.local` correspond à
`dev.local`, pas à `local`. Il n'y a donc jamais de ligne à ajouter par application.
Toutes les autres requêtes sont relayées vers Cloudflare (`1.1.1.1`) et Quad9
(`9.9.9.9`) ; modifiez ces serveurs si nécessaire.

### Démarrage

```bash
docker compose -f docker-compose.dnsmasq.yml up -d
docker compose -f docker-compose.dnsmasq.yml logs -f dnsmasq
```

Si le port 53 est déjà occupé (`systemd-resolved`), `bind-interfaces` avec
`listen-address` règle en général le conflit.

Après toute modification de la stack, vérifiez que le conteneur a bien été **recréé** :
dnsmasq ne relit pas sa configuration à chaud.

### Pare-feu

Sur un serveur avec `ufw`, les conteneurs et les autres machines du LAN ne peuvent pas
interroger le DNS par défaut. Autorisez le port 53 :

```bash
sudo ufw allow from 172.16.0.0/12 to any port 53     # réseaux Docker
sudo ufw allow from 192.168.1.0/24 to any port 53    # LAN
sudo ufw reload
```

Sans cette règle, le DNS répond en local sur le serveur mais pas depuis les conteneurs
(`Could not resolve host`).

### Tester

Depuis la machine DNS, puis depuis un conteneur :

```bash
nslookup jellyfin.local 192.168.1.5
docker run --rm alpine nslookup jellyfin.local 192.168.1.5
```

Les deux doivent renvoyer l'IP du serveur prod.

## Utiliser les noms depuis Kasm Workspace

Dans l'administration de Kasm : **Workspaces → Chrome → Edit → Docker Run Config
Override (JSON)** :

```json
{
  "dns": ["192.168.1.5"]
}
```

Si le champ contient déjà du JSON, fusionnez la clé `dns` avec l'existant. Détruisez puis
relancez la session Chrome pour que le changement soit pris en compte. Dans Chrome, tapez
l'adresse complète la première fois (`http://jellyfin.local`) : sans le `http://`, Chrome
peut lancer une recherche. Désactivez aussi **Utiliser un DNS sécurisé** dans
`chrome://settings/security` si la résolution échoue.

## Utiliser les noms sur tout le réseau

Pour que tous vos appareils profitent des noms, ils doivent utiliser la machine DNS.

- **Sur le routeur** : si l'interface de votre box permet de choisir les DNS distribués
  par le DHCP, renseignez l'IP de la machine DNS. Mettez la même IP en primaire et en
  secondaire : les clients utilisent parfois le secondaire au hasard, et un DNS public
  ne connaît pas vos noms. Tous les modèles de box ne proposent pas ce réglage.
- **Machine par machine** : renseignez l'IP du DNS dans les paramètres réseau de chaque
  appareil.

Si la machine DNS s'éteint, les appareils qui l'utilisent perdent la résolution de
noms, Internet compris.

## Dashboard

Chaque instance Traefik a son propre dashboard, en lecture seule :

- `http://traefik.<DOMAIN>` (via le DNS), par exemple `http://traefik.local` ;
- `http://<ip-du-serveur>:8181/dashboard/` en accès direct.

Traefik ne sait pas regrouper plusieurs instances. Pour une page unique qui pointe vers
les dashboards et les services de tous vos serveurs, un portail de type
[Homepage](https://gethomepage.dev) déployé sur une seule machine convient bien.

## Dépannage

Tester dans cet ordre, la première étape qui échoue indique la cause.

| Symptôme | Cause probable | Vérification |
|---|---|---|
| `NXDOMAIN` ou `Could not resolve host` avec le bon suffixe | dnsmasq tourne avec l'ancienne config, ou le client n'utilise pas ce DNS | `docker exec dnsmasq cat /etc/dnsmasq.conf`, puis `docker restart dnsmasq` |
| Le DNS répond sur le serveur mais pas depuis un conteneur ou le LAN | Pare-feu sur le port 53 | Règles `ufw` ci-dessus |
| Le conteneur Kasm ne résout rien | Override DNS absent ou session non relancée | `docker inspect <id> --format '{{.HostConfig.Dns}}'` doit afficher `[192.168.1.5]` |
| `504 Gateway Timeout` | Traefik n'atteint pas le conteneur (pas de réseau commun, ou mauvais port) | `docker inspect traefik --format '{{json .NetworkSettings.Networks}}'` ; vérifier le port du label |
| `502 Bad Gateway` | Traefik joint le conteneur mais le port n'écoute pas | Utiliser le port **interne** dans le label |
| `404 page not found` | Traefik répond mais ne connaît pas ce nom | Le nom du service ne correspond pas au sous-domaine, ou `traefik.enable=false` |
| Sous-domaine doublé (`dozzle-monitoring...`) | Ancienne règle basée sur le nom du conteneur | Utiliser la règle de ce dépôt (label `com.docker.compose.service`) |
| Résolution capricieuse avec `.local` | mDNS intercepte le suffixe | Changer de suffixe (voir plus haut) |

Pour voir chaque requête et le service ciblé, `--log.level=INFO` et `--accesslog=true`
sont déjà activés dans la stack : `docker logs traefik`.

## Commandes utiles

```bash
# Afficher la configuration finale après interpolation des variables
DOMAIN=local docker compose config

# Réseaux auxquels Traefik est connecté
docker inspect traefik --format '{{json .NetworkSettings.Networks}}'

# Ports que Traefik voit pour un conteneur
docker inspect <conteneur> --format '{{json .Config.ExposedPorts}}'

# Arrêter et supprimer la stack
docker compose down
```

## Limites et sécurité

Cette configuration est faite pour un environnement local ou de confiance :

- Le socket Docker est monté dans les deux conteneurs. `traefik-autojoin` y a un accès
  en **écriture** (nécessaire pour `docker network connect`), ce qui équivaut à des
  droits root sur l'hôte.
- Le dashboard est activé avec `api.insecure=true`, sans authentification, et le port
  `8181` est publié.
- Aucun HTTPS ni certificat TLS n'est configuré : tout passe en HTTP sur le port 80.
- Traefik étant connecté à tous les réseaux et les conteneurs exposés par défaut, tout
  service parlant HTTP est joignable sur le LAN.
- Les services non-HTTP (SSH, bases de données...) ne passent pas par Traefik.
- dnsmasq relaie les requêtes externes vers des DNS publics (`1.1.1.1`, `9.9.9.9`).

Avant toute exposition sur Internet : désactivez `api.insecure`, ajoutez une
authentification, configurez TLS, passez `exposedByDefault` à `false` et restreignez
l'accès au socket Docker (par exemple avec un proxy de socket en lecture seule pour
Traefik).