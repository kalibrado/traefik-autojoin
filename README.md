# traefik-autojoin

Configuration Docker Compose qui lance Traefik et le connecte automatiquement aux
réseaux Docker `bridge` créés par d'autres applications Compose. Une stack DNSMasq
optionnelle permet de rendre les noms d'applications accessibles depuis tout le LAN.

## Fonctionnement

- `traefik` est le reverse proxy Traefik v3.
- `traefik-autojoin` surveille les événements Docker et connecte `traefik` aux
  nouveaux réseaux `bridge`.
- Les réseaux système `bridge`, `host`, `none` et le réseau `proxy` sont ignorés.
- Les conteneurs sont exposés automatiquement (`exposedByDefault=true`).
- La règle par défaut utilise le nom du service Compose :
  `http://<service>.<DOMAIN>`.

Le réseau `proxy` est créé avec un nom fixe afin que Traefik puisse l'utiliser comme
réseau Docker par défaut. Le conteneur Traefik n'a pas besoin d'être ajouté
manuellement aux réseaux des applications.

## Prérequis

- Docker Engine avec le plugin Docker Compose.
- Le port `80` disponible pour les routes HTTP.
- Le port `8181` disponible pour l'accès direct au dashboard/API.
- Un nom DNS, ou des entrées dans `/etc/hosts`, pour les noms utilisés par Traefik.

La stack Traefik peut être lancée sur chaque machine Docker. La stack DNSMasq,
décrite plus bas, doit être lancée sur une seule machine du LAN.

## Installation

1. Créez le fichier `.env` à partir du modèle :

   ```sh
   cp .env.example .env
   ```

2. Définissez le suffixe de domaine dans `.env` :

   ```dotenv
   DOMAIN=example.com
   ```

3. Démarrez les services :

   ```sh
   docker compose up -d
   ```

4. Vérifiez l'état des conteneurs et les logs :

   ```sh
   docker compose ps
   docker compose logs -f traefik-autojoin
   ```

Avec `DOMAIN=example.com`, le dashboard peut être ouvert à l'adresse
`http://traefik.example.com` si ce nom résout vers l'hôte Docker. L'accès direct
au dashboard/API reste disponible sur `http://<hote-docker>:8181/dashboard/`.

Pour un test local, ajoutez par exemple dans `/etc/hosts` :

```text
127.0.0.1 traefik.example.com
```

## Publier une application

Une application Compose classique peut rester sur son propre réseau : le service
`traefik-autojoin` le détectera et y connectera Traefik.

Par exemple, un service nommé `whoami` sera accessible à :

```text
http://whoami.example.com
```

La règle par défaut est active pour tous les conteneurs Docker. Pour éviter
d'exposer un conteneur, ajoutez le label suivant à son service :

```yaml
labels:
  - traefik.enable=false
```

Pour définir une règle personnalisée, utilisez les labels Traefik habituels :

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.whoami.rule=Host(`app.example.com`)
  - traefik.http.services.whoami.loadbalancer.server.port=80
```

## DNSMasq pour le LAN

Le fichier [docker-compose.dnsmasq.yml](docker-compose.dnsmasq.yml) est une stack
distincte. Lancez-la uniquement sur la machine qui servira de DNS au réseau LAN.
Elle utilise `network_mode: host` et écoute sur le port DNS 53 en UDP et TCP.

### Configuration

Avant le démarrage, adaptez les valeurs dans `docker-compose.dnsmasq.yml`, ou
définissez-les dans le fichier `.env` de cette machine :

```dotenv
DNSMASQ_LISTEN_ADDRESS=192.168.1.5
TRAEFIK_HOST_IP=192.168.1.113
DEV_HOST_IP=192.168.1.43
TEST_HOST_IP=192.168.1.42
KASM_HOST_IP=192.168.1.5
```

`DNSMASQ_LISTEN_ADDRESS` doit être l'adresse LAN fixe de la machine DNS. Les autres
variables sont les adresses LAN des machines qui hébergent les applications. Ajoutez
une ligne `address` pour chaque nom qui doit être publié :

```ini
address=/whoami.local/192.168.1.113
address=/app2.local/192.168.1.43
```

Le nom DNS doit correspondre au nom utilisé par Traefik. Avec `DOMAIN=local`, un
service Compose nommé `whoami` est donc accessible avec `whoami.local`.

Démarrez ensuite cette stack sur la machine DNS :

```sh
docker compose -f docker-compose.dnsmasq.yml up -d
docker compose -f docker-compose.dnsmasq.yml logs -f dnsmasq
```

Configurez l'adresse IP de cette machine comme serveur DNS principal dans le
routeur DHCP. Tous les appareils qui récupèrent cette configuration DNS pourront
alors résoudre les noms `*.local` et atteindre les applications via Traefik.

Chaque hôte Docker doit avoir une adresse LAN stable, idéalement avec une
réservation DHCP. Le port 53 ne doit pas déjà être utilisé par un autre service
sur la machine DNS.

N'utilisez pas une règle globale comme `address=/.local/192.168.1.5` si plusieurs
machines hébergent des applications : elle enverrait tous les noms vers une seule
machine. Déclarez plutôt chaque nom avec l'adresse de l'hôte correspondant.

## Commandes utiles

```sh
# Afficher la configuration finale après interpolation
DOMAIN=example.com docker compose config

# Arrêter et supprimer les conteneurs
docker compose down

# Voir les réseaux auxquels Traefik est connecté
docker inspect traefik --format '{{json .NetworkSettings.Networks}}'
```

## Limites et sécurité

Cette configuration est adaptée à un environnement local ou de confiance :

- le socket Docker est monté dans les deux conteneurs ;
- `traefik-autojoin` dispose d'un accès en écriture au socket pour exécuter
  `docker network connect` ;
- le dashboard est activé avec `api.insecure=true`, sans authentification ;
- aucun HTTPS ni certificat TLS n'est configuré ;
- les conteneurs sont exposés par défaut.
- DNSMasq relaie les requêtes externes vers Cloudflare (`1.1.1.1`) et Quad9
  (`9.9.9.9`) ; adaptez ces serveurs si nécessaire.
- Le suffixe `.local` est également utilisé par mDNS. Pour éviter les conflits sur
  certains appareils, un suffixe privé comme `home.arpa` peut être préférable ;
  dans ce cas, utilisez la même valeur pour `DOMAIN` et les entrées DNS.

Avant une utilisation sur Internet, désactivez le mode insecure, ajoutez une
authentification et configurez TLS. Restreignez également les labels et l'accès
au socket Docker selon votre modèle de sécurité.
