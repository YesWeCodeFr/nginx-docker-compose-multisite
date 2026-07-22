# Installer NGINX dans Docker avec Docker Compose

Ce guide accompagne deux vidéos : la première est consacrée au déploiement de NGINX dans Docker avec Docker Compose, et la seconde à sa sécurisation en HTTPS avec Let's Encrypt.

L'objectif est de mettre en place une installation structurée dans `/srv/nginx`, capable d'héberger plusieurs sites avec :

- une image NGINX dont la version est explicitement définie ;
- une configuration globale de NGINX ;
- une configuration distincte pour chaque site ;
- des contenus HTML et des journaux séparés ;
- un site par défaut ;
- un second site accessible avec le domaine d'exemple `www.example.com` ;
- un certificat TLS délivré par Let's Encrypt ;
- une redirection automatique de HTTP vers HTTPS ;
- le renouvellement des certificats avec Certbot.

> Les commandes ci-dessous sont prévues pour un serveur Ubuntu ou Debian sur lequel Docker Engine et le plugin Docker Compose sont déjà installés.
> Exécutez-les avec un utilisateur qui possède les droits `sudo` et peut utiliser la commande `docker`.

> Remplacez `www.example.com` et `ADRESSE_IP_PUBLIQUE_DU_VPS` par votre propre nom de domaine et l'adresse publique de votre serveur.
> Le domaine `example.com` est réservé à la documentation : les commandes qui l'utilisent servent de modèles et ne fonctionneront qu'après ce remplacement.

## Ressources liées

Cette page sert de support aux deux vidéos YouTube associées.

- YouTube — [Installer NGINX dans Docker avec Docker Compose](https://youtu.be/l7uX_mWikqI)
- YouTube — [Sécuriser NGINX dans Docker avec Let's Encrypt](https://youtu.be/0ywPugJZ2Mo)
- LinkedIn : [me suivre sur LinkedIn](https://www.linkedin.com/in/dominique-korzeczek-695033188)

## Prérequis

- Un serveur Ubuntu ou Debian accessible en SSH.
- Un utilisateur avec les droits `sudo`.
- Docker Engine et Docker Compose installés.
- Le port TCP `80` accessible depuis internet.
- Le port TCP `443` accessible depuis internet.
- Pour la partie consacrée au domaine :
  - un nom de domaine ;
  - un accès à sa zone DNS ;
  - une entrée DNS de type `A` pour `www`, pointant vers l'adresse IPv4 publique du serveur.

Vous pouvez vérifier Docker et Docker Compose avec :

```bash
docker --version
docker compose version
```

> Le port `80` doit rester accessible pour permettre à Let's Encrypt de vérifier le domaine avec le challenge HTTP-01.

## 1. Créer le groupe d'administration NGINX

L'arborescence appartiendra à l'utilisateur `root` et au groupe `nginx-admin`.
Ce groupe permet d'autoriser uniquement certains utilisateurs à administrer l'installation.

Créez le groupe :

```bash
sudo groupadd nginx-admin
```

Si ce groupe existe déjà, il n'est pas nécessaire de le recréer.

Ajoutez l'utilisateur actuellement connecté au groupe :

```bash
sudo usermod -aG nginx-admin "$USER"
```

Activez le groupe dans un nouveau shell sans vous reconnecter au serveur :

```bash
newgrp nginx-admin
```

## 2. Créer l'arborescence principale

Créez les dossiers du site par défaut :

```bash
sudo mkdir -p /srv/nginx/{html/default,config/conf.d,log/default}
```

Créez les premiers fichiers de configuration :

```bash
sudo touch /srv/nginx/compose.yaml
sudo touch /srv/nginx/config/nginx.conf
sudo touch /srv/nginx/config/conf.d/default.conf
```

Attribuez l'arborescence à `root` et au groupe `nginx-admin` :

```bash
sudo chown -R root:nginx-admin /srv/nginx
```

Autorisez les membres du groupe à lire et modifier son contenu :

```bash
sudo chmod -R g+rwX /srv/nginx
```

Activez l'héritage du groupe sur tous les dossiers :

```bash
sudo find /srv/nginx -type d -exec chmod g+s {} +
```

Les futurs fichiers et dossiers créés dans cette arborescence hériteront ainsi du groupe `nginx-admin`.

## 3. Créer le fichier Docker Compose

Placez-vous dans le dossier de l'installation :

```bash
cd /srv/nginx
```

Ouvrez le fichier `compose.yaml` :

```bash
nano compose.yaml
```

Ajoutez la configuration suivante :

```yaml
services:
  nginx:
    image: nginx:1.30.3-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config/conf.d:/etc/nginx/conf.d:ro
      - ./log:/var/log/nginx
```

### Version de NGINX et image Alpine

La ligne suivante définit précisément l'image utilisée :

```yaml
image: nginx:1.30.3-alpine
```

Le numéro `1.30.3` permet de figer la version de NGINX.
Le déploiement reste ainsi prévisible et reproductible, même si une nouvelle version de l'image est publiée plus tard.

Il est préférable d'utiliser une version explicite plutôt que le tag `latest`, qui peut pointer vers une nouvelle version lors d'un prochain téléchargement.

Le suffixe `alpine` indique que l'image repose sur Alpine Linux, une distribution conçue pour être compacte.
Cette variante est particulièrement adaptée ici, car nous avons uniquement besoin de NGINX pour servir nos sites :

- l'image est plus légère à télécharger ;
- elle occupe moins d'espace sur le serveur ;
- elle contient peu de logiciels supplémentaires inutiles à notre installation.

En contrepartie, la variante Alpine fournit moins d'outils que l'image standard.
Si vous devez installer des utilitaires, compiler des modules ou effectuer des diagnostics avancés dans le conteneur, une variante basée sur Debian peut parfois être plus pratique.

### Redémarrage automatique

La ligne suivante définit la politique de redémarrage du conteneur :

```yaml
restart: unless-stopped
```

Avec cette politique, Docker redémarre automatiquement NGINX :

- si le conteneur s'arrête à cause d'une erreur ;
- lorsque Docker redémarre après un redémarrage du serveur.

Ainsi, après un redémarrage du serveur, NGINX redémarre automatiquement.

Si vous arrêtez volontairement le conteneur, il reste arrêté jusqu'à ce que vous le redémarriez manuellement.

### Nom du conteneur

La ligne suivante attribue le nom `nginx` au conteneur :

```yaml
container_name: nginx
```

Ce nom est personnalisable. Vous pouvez par exemple utiliser :

```yaml
container_name: nginx-web
```

Autres exemples possibles :

- `nginx-proxy` ;
- `web-server` ;
- `production-nginx`.

Choisissez un nom descriptif et unique sur le serveur afin d'identifier facilement le conteneur dans les commandes et les journaux Docker.

Le nom du service reste cependant `nginx`, car il est défini par la clé située sous `services`.
Ainsi, une commande comme `docker compose exec nginx nginx -t` reste inchangée même si vous modifiez `container_name`.

Les chemins qui commencent par `./` sont relatifs au dossier `/srv/nginx`, dans lequel se trouve `compose.yaml`.

Les contenus HTML et les configurations sont montés en lecture seule avec l'option `ro`.
Le dossier des journaux reste accessible en écriture pour NGINX.

## 4. Configurer NGINX

Ouvrez le fichier de configuration globale :

```bash
nano config/nginx.conf
```

Ajoutez le contenu suivant :

```nginx
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
```

Ce fichier contient les réglages communs à toute l'installation.
La dernière directive `include` charge tous les fichiers de site présents dans `/etc/nginx/conf.d` à l'intérieur du conteneur.

## 5. Configurer le site par défaut

Ouvrez le fichier `default.conf` :

```bash
nano config/conf.d/default.conf
```

Ajoutez la configuration suivante :

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /usr/share/nginx/html/default;
    index index.html;

    access_log /var/log/nginx/default/access.log;
    error_log /var/log/nginx/default/error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

L'option `default_server` indique que ce bloc doit répondre lorsqu'aucun autre `server_name` ne correspond à la requête reçue.

## 6. Créer la page du site par défaut

Ouvrez le fichier HTML :

```bash
nano html/default/index.html
```

Ajoutez le contenu suivant :

```html
<!doctype html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Site par défaut</title>
</head>
<body>
    <h1>Bienvenue sur le site par défaut</h1>
</body>
</html>
```

## 7. Démarrer et tester le site par défaut

Démarrez NGINX en arrière-plan :

```bash
docker compose up -d
```

Vérifiez l'état du conteneur :

```bash
docker compose ps
```

Le service `nginx` doit apparaître comme actif.

Testez ensuite le serveur depuis le VPS :

```bash
curl http://localhost
```

La réponse doit contenir :

```text
Bienvenue sur le site par défaut
```

Vous pouvez également ouvrir l'adresse IP publique du VPS dans un navigateur :

```text
http://ADRESSE_IP_DU_VPS
```

## 8. Associer le domaine au serveur

Pour rendre le site `www.example.com` accessible, ouvrez la zone DNS du domaine `example.com`, puis créez une entrée de type `A` :

```text
Type  : A
Nom   : www.example.com
Valeur: ADRESSE_IP_PUBLIQUE_DU_VPS
```

Cette entrée associe le sous-domaine `www.example.com` à l'adresse IPv4 publique du serveur.

Selon le fournisseur DNS, le champ `Nom` peut attendre le nom complet `www.example.com` ou uniquement le préfixe `www`.
Vérifiez le format demandé dans son interface.

La propagation DNS peut demander un peu de temps.
Il reste cependant possible de tester la configuration NGINX localement avant qu'elle soit terminée.

## 9. Créer les fichiers du site `www.example.com`

Depuis `/srv/nginx`, créez les dossiers du nouveau site :

```bash
sudo mkdir -p html/www.example.com
sudo mkdir -p log/www.example.com
```

Créez le fichier HTML et le fichier de configuration NGINX :

```bash
sudo touch html/www.example.com/index.html
sudo touch config/conf.d/www.example.com.conf
```

Accordez au groupe les droits nécessaires :

```bash
sudo chmod -R g+rwX html/www.example.com
sudo chmod -R g+rwX log/www.example.com
sudo chmod g+rw config/conf.d/www.example.com.conf
```

## 10. Configurer le site `www.example.com`

Ouvrez son fichier de configuration :

```bash
nano config/conf.d/www.example.com.conf
```

Ajoutez le bloc `server` suivant :

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name www.example.com;

    root /usr/share/nginx/html/www.example.com;
    index index.html;

    access_log /var/log/nginx/www.example.com/access.log;
    error_log /var/log/nginx/www.example.com/error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Ce bloc ne contient pas `default_server`.
NGINX le sélectionne lorsque l'en-tête HTTP `Host` correspond à `www.example.com`.

## 11. Créer la page de `www.example.com`

Ouvrez le fichier HTML :

```bash
nano html/www.example.com/index.html
```

Ajoutez le contenu suivant :

```html
<!doctype html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Site d'exemple</title>
</head>
<body>
    <h1>Bienvenue sur www.example.com</h1>
</body>
</html>
```

## 12. Valider et recharger NGINX

Vérifiez la syntaxe de toute la configuration :

```bash
docker compose exec nginx nginx -t
```

Si le test réussit, rechargez NGINX sans redémarrer le conteneur :

```bash
docker compose exec nginx nginx -s reload
```

## 13. Tester le nouveau site

Avant la fin de la propagation DNS, testez le nouveau bloc `server` en envoyant explicitement l'en-tête HTTP `Host` :

```bash
curl -H "Host: www.example.com" http://localhost
```

Une fois le domaine disponible, testez son adresse directement :

```bash
curl http://www.example.com
```

Dans les deux cas, la réponse doit contenir :

```text
Bienvenue sur www.example.com
```

Vous pouvez enfin ouvrir le site dans un navigateur :

```text
http://www.example.com
```

## 14. Comprendre HTTPS, TLS et Let's Encrypt

Avant de modifier l'installation, distinguons les principaux éléments qui permettent de sécuriser un site.

### HTTPS

HTTPS chiffre les échanges entre le navigateur de l'utilisateur et le site.
Les informations transmises, comme des identifiants ou le contenu d'un formulaire, ne peuvent ainsi pas être lues simplement par une personne qui intercepterait la connexion.

### TLS et le certificat

TLS est le protocole qui permet d'établir cette connexion chiffrée.
Le certificat TLS est en quelque sorte la carte d'identité du site : il permet au navigateur de vérifier qu'il communique bien avec le domaine demandé.

### Let's Encrypt

Let's Encrypt est une autorité de certification gratuite et automatisée.
Elle vérifie que vous contrôlez le nom de domaine, puis délivre le certificat TLS nécessaire à HTTPS.

### Certbot

Certbot est un programme en ligne de commande qui communique avec Let's Encrypt.
Il automatise notamment la demande, l'installation et le renouvellement des certificats.

Dans cette installation, Certbot et NGINX partageront des dossiers Docker.
Certbot y déposera les fichiers du challenge et les certificats, puis NGINX les utilisera pour rendre le site accessible en HTTPS.

## 15. Créer les dossiers de Certbot

Depuis `/srv/nginx`, créez les dossiers utilisés par Certbot :

```bash
sudo mkdir -p certbot/{www,conf,log}
```

Ces dossiers ont chacun un rôle précis :

- `certbot/www` reçoit les fichiers temporaires du challenge HTTP-01 ;
- `certbot/conf` conserve les certificats, les clés privées et les données de renouvellement ;
- `certbot/log` conserve les journaux de Certbot.

Attribuez-les à `root` et au groupe `nginx-admin`, puis appliquez les droits utilisés par le reste de l'installation :

```bash
sudo chown -R root:nginx-admin certbot
sudo chmod -R g+rwX certbot
sudo find certbot -type d -exec chmod g+s {} +
```

## 16. Ajouter Certbot au fichier Docker Compose

Ouvrez le fichier :

```bash
nano compose.yaml
```

Remplacez son contenu par cette configuration :

```yaml
services:
  nginx:
    image: nginx:1.30.3-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config/conf.d:/etc/nginx/conf.d:ro
      - ./log:/var/log/nginx
      - ./certbot/www:/var/www/certbot:ro
      - ./certbot/conf:/etc/letsencrypt:ro

  certbot:
    profiles: ["certbot"]
    image: certbot/certbot:v5.7.0
    container_name: certbot
    volumes:
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/log:/var/log/letsencrypt
```

Le port `443` permet à NGINX de recevoir les connexions HTTPS.
Les deux services partagent le dossier du challenge et celui des certificats.
NGINX les monte en lecture seule, tandis que Certbot peut y écrire.

Le profil `certbot` empêche le service Certbot de démarrer avec la commande habituelle `docker compose up`.
Certbot sera exécuté ponctuellement pour demander ou renouveler les certificats.

Vérifiez la syntaxe du fichier :

```bash
docker compose config
```

Recréez ensuite le conteneur NGINX afin d'appliquer le nouveau port et les nouveaux volumes :

```bash
docker compose up -d nginx
```

## 17. Rendre le challenge Let's Encrypt accessible

Le certificat n'existe pas encore : conservez pour le moment la configuration HTTP du site.

Ouvrez son fichier :

```bash
nano config/conf.d/www.example.com.conf
```

Dans le bloc `server` existant, ajoutez cette `location` avant la `location /` principale :

```nginx
location ^~ /.well-known/acme-challenge/ {
    root /var/www/certbot;
}
```

Le bloc HTTP complet devient :

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name www.example.com;

    root /usr/share/nginx/html/www.example.com;
    index index.html;

    access_log /var/log/nginx/www.example.com/access.log;
    error_log /var/log/nginx/www.example.com/error.log warn;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Vérifiez la configuration, puis rechargez NGINX :

```bash
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
```

## 18. Demander le certificat Let's Encrypt

Avant de lancer la commande, vérifiez que le domaine pointe vers l'adresse publique du serveur et que le port `80` est accessible depuis internet.

Demandez ensuite le certificat :

```bash
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path /var/www/certbot \
  --email VOTRE_ADRESSE_EMAIL \
  --agree-tos \
  --no-eff-email \
  --non-interactive \
  -d www.example.com
```

Remplacez `VOTRE_ADRESSE_EMAIL` par une adresse valide et `www.example.com` par votre domaine réel.

Les options principales sont les suivantes :

- `certonly` demande un certificat sans modifier automatiquement la configuration NGINX ;
- `--webroot` sélectionne la méthode du challenge par dossier web ;
- `--webroot-path` indique le dossier partagé dans lequel déposer le jeton du challenge ;
- `--email` fournit l'adresse utilisée pour les notifications importantes ;
- `-d` indique le domaine à inclure dans le certificat.

L'option `--rm` supprime le conteneur temporaire lorsque la commande est terminée, sans supprimer les certificats enregistrés dans les volumes.

## 19. Vérifier le certificat

Listez les certificats gérés par Certbot :

```bash
docker compose run --rm certbot certificates
```

La commande affiche notamment les domaines couverts, la date d'expiration et les chemins des fichiers du certificat.

Pour `www.example.com`, NGINX utilisera les fichiers suivants dans le conteneur :

```text
/etc/letsencrypt/live/www.example.com/fullchain.pem
/etc/letsencrypt/live/www.example.com/privkey.pem
```

## 20. Activer HTTPS dans NGINX

Le certificat existe désormais. Ouvrez à nouveau la configuration du site :

```bash
nano config/conf.d/www.example.com.conf
```

Ajoutez ce second bloc `server` après le bloc HTTP :

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name www.example.com;

    ssl_certificate /etc/letsencrypt/live/www.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.example.com/privkey.pem;

    root /usr/share/nginx/html/www.example.com;
    index index.html;

    access_log /var/log/nginx/www.example.com/access.log;
    error_log /var/log/nginx/www.example.com/error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Vérifiez la syntaxe et rechargez NGINX :

```bash
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
```

Testez les deux protocoles :

```bash
curl http://www.example.com
curl https://www.example.com
```

À cette étape, le site répond à la fois en HTTP et en HTTPS.

## 21. Rediriger HTTP vers HTTPS

Dans le bloc HTTP, conservez la `location` du challenge et remplacez uniquement la `location /` principale par cette redirection :

```nginx
location / {
    return 301 https://$host$request_uri;
}
```

Le bloc HTTP final devient :

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name www.example.com;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

Vérifiez et rechargez une nouvelle fois NGINX :

```bash
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
```

Ouvrez ensuite `http://www.example.com` dans un navigateur.
L'adresse doit être automatiquement redirigée vers `https://www.example.com`.

## 22. Tester et renouveler les certificats

Testez d'abord le processus de renouvellement sans modifier le certificat réel :

```bash
docker compose run --rm certbot renew --dry-run
```

Pour lancer le renouvellement réel de tous les certificats gérés par Certbot :

```bash
docker compose run --rm certbot renew
```

Certbot ne renouvelle que les certificats suffisamment proches de leur expiration.
Pour un certificat valable 90 jours, le renouvellement intervient généralement lorsqu'il reste environ 30 jours de validité.

Après un renouvellement, rechargez NGINX afin qu'il relise les fichiers du certificat :

```bash
docker compose exec nginx nginx -s reload
```

En production, placez les commandes de renouvellement et de rechargement dans un script exécuté automatiquement, chaque jour par exemple, avec Cron.

## Arborescence finale

```text
/srv/nginx/
├── compose.yaml
├── certbot/
│   ├── www/
│   ├── conf/
│   │   ├── live/
│   │   ├── archive/
│   │   └── renewal/
│   └── log/
├── html/
│   ├── default/
│   │   └── index.html
│   └── www.example.com/
│       └── index.html
├── config/
│   ├── nginx.conf
│   └── conf.d/
│       ├── default.conf
│       └── www.example.com.conf
└── log/
    ├── error.log
    ├── default/
    │   ├── access.log
    │   └── error.log
    └── www.example.com/
        ├── access.log
        └── error.log
```

Les fichiers de journaux sont créés automatiquement après le démarrage ou le rechargement de NGINX.

## Récapitulatif rapide des commandes

```bash
# Groupe d'administration
sudo groupadd nginx-admin
sudo usermod -aG nginx-admin "$USER"
newgrp nginx-admin

# Arborescence principale
sudo mkdir -p /srv/nginx/{html/default,config/conf.d,log/default}
sudo touch /srv/nginx/compose.yaml
sudo touch /srv/nginx/config/nginx.conf
sudo touch /srv/nginx/config/conf.d/default.conf
sudo chown -R root:nginx-admin /srv/nginx
sudo chmod -R g+rwX /srv/nginx
sudo find /srv/nginx -type d -exec chmod g+s {} +

# Configuration principale
cd /srv/nginx
nano compose.yaml
nano config/nginx.conf
nano config/conf.d/default.conf
nano html/default/index.html

# Démarrage et premier test
docker compose up -d
docker compose ps
curl http://localhost

# Second site
sudo mkdir -p html/www.example.com
sudo mkdir -p log/www.example.com
sudo touch html/www.example.com/index.html
sudo touch config/conf.d/www.example.com.conf
sudo chmod -R g+rwX html/www.example.com
sudo chmod -R g+rwX log/www.example.com
sudo chmod g+rw config/conf.d/www.example.com.conf
nano config/conf.d/www.example.com.conf
nano html/www.example.com/index.html

# Validation, rechargement et tests
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
curl -H "Host: www.example.com" http://localhost
curl http://www.example.com

# Certbot et HTTPS
sudo mkdir -p certbot/{www,conf,log}
sudo chown -R root:nginx-admin certbot
sudo chmod -R g+rwX certbot
sudo find certbot -type d -exec chmod g+s {} +
nano compose.yaml
docker compose config
docker compose up -d nginx
nano config/conf.d/www.example.com.conf
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload

# Demande et contrôle du certificat
docker compose run --rm certbot certonly --webroot \
  --webroot-path /var/www/certbot \
  --email VOTRE_ADRESSE_EMAIL \
  --agree-tos --no-eff-email --non-interactive \
  -d www.example.com
docker compose run --rm certbot certificates

# Tests et renouvellement
curl https://www.example.com
docker compose run --rm certbot renew --dry-run
docker compose run --rm certbot renew
docker compose exec nginx nginx -s reload
```

## Points d'attention

- Les chemins `/etc/nginx`, `/usr/share/nginx/html` et `/var/log/nginx` sont les chemins utilisés à l'intérieur du conteneur.
- Les chemins `./html`, `./config` et `./log` sont relatifs au dossier qui contient `compose.yaml` sur le serveur.
- Le contenu HTML et la configuration sont montés en lecture seule dans le conteneur.
- Le dossier des journaux doit rester accessible en écriture.
- Le dossier `certbot/conf` contient notamment les clés privées : ne le publiez pas et limitez-en l'accès aux utilisateurs autorisés.
- Le groupe `nginx-admin` doit être réservé aux utilisateurs autorisés à modifier l'installation.
- Après `usermod`, utilisez `newgrp nginx-admin` ou reconnectez-vous au serveur pour prendre en compte le groupe.
- Exécutez toujours `nginx -t` avant de recharger NGINX après une modification de configuration.
- Le domaine doit pointer vers l'adresse publique du serveur et les ports `80` et `443` doivent être accessibles.
- La `location` du challenge HTTP-01 doit rester accessible sur le port `80`, même après l'activation de la redirection HTTPS.
- Exécutez périodiquement `certbot renew`, puis rechargez NGINX pour prendre en compte un certificat renouvelé.

## Aller plus loin

Cette base permet d'ajouter d'autres sites en créant, pour chacun :

- un dossier dans `html` ;
- un fichier dans `config/conf.d` ;
- un dossier dans `log` ;
- une entrée DNS adaptée ;
- un certificat Let's Encrypt correspondant à son ou ses noms de domaine.

Le même dossier `certbot/www` peut servir aux challenges de plusieurs sites.
Certbot conserve ensuite tous les certificats sous `certbot/conf`, dans une arborescence distincte pour chaque nom de certificat.
