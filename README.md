# Installer NGINX dans Docker avec Docker Compose

Ce guide accompagne la vidéo consacrée au déploiement de NGINX dans Docker avec Docker Compose.

L'objectif est de mettre en place une installation structurée dans `/srv/nginx`, capable d'héberger plusieurs sites avec :

- une image NGINX dont la version est explicitement définie ;
- une configuration globale de NGINX ;
- une configuration distincte pour chaque site ;
- des contenus HTML et des journaux séparés ;
- un site par défaut ;
- un second site accessible avec le domaine d'exemple `www.example.com`.

> Les commandes ci-dessous sont prévues pour un serveur Ubuntu ou Debian sur lequel Docker Engine et le plugin Docker Compose sont déjà installés.
> Exécutez-les avec un utilisateur qui possède les droits `sudo` et peut utiliser la commande `docker`.

> Remplacez `www.example.com` et `ADRESSE_IP_PUBLIQUE_DU_VPS` par votre propre nom de domaine et l'adresse publique de votre serveur.
> Le domaine `example.com` est réservé à la documentation : les commandes qui l'utilisent servent de modèles et ne fonctionneront qu'après ce remplacement.

## Ressources liées

Cette page sert de support à la vidéo YouTube associée.

- YouTube : [regarder la vidéo](https://youtu.be/l7uX_mWikqI)
- LinkedIn : [me suivre sur LinkedIn](https://www.linkedin.com/in/dominique-korzeczek-695033188)

## Prérequis

- Un serveur Ubuntu ou Debian accessible en SSH.
- Un utilisateur avec les droits `sudo`.
- Docker Engine et Docker Compose installés.
- Le port TCP `80` accessible depuis internet.
- Pour la partie consacrée au domaine :
  - un nom de domaine ;
  - un accès à sa zone DNS ;
  - une entrée DNS de type `A` pour `www`, pointant vers l'adresse IPv4 publique du serveur.

Vous pouvez vérifier Docker et Docker Compose avec :

```bash
docker --version
docker compose version
```

> Cette installation utilise uniquement HTTP sur le port `80`.
> La mise en place de HTTPS avec un certificat TLS et Let's Encrypt sera traitée séparément.

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

## Arborescence finale

```text
/srv/nginx/
├── compose.yaml
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
```

## Points d'attention

- Les chemins `/etc/nginx`, `/usr/share/nginx/html` et `/var/log/nginx` sont les chemins utilisés à l'intérieur du conteneur.
- Les chemins `./html`, `./config` et `./log` sont relatifs au dossier qui contient `compose.yaml` sur le serveur.
- Le contenu HTML et la configuration sont montés en lecture seule dans le conteneur.
- Le dossier des journaux doit rester accessible en écriture.
- Le groupe `nginx-admin` doit être réservé aux utilisateurs autorisés à modifier l'installation.
- Après `usermod`, utilisez `newgrp nginx-admin` ou reconnectez-vous au serveur pour prendre en compte le groupe.
- Exécutez toujours `nginx -t` avant de recharger NGINX après une modification de configuration.
- Le domaine doit pointer vers l'adresse publique du serveur et le port `80` doit être accessible.
- Cette configuration ne fournit pas encore HTTPS.

## Aller plus loin

Cette base permet d'ajouter d'autres sites en créant, pour chacun :

- un dossier dans `html` ;
- un fichier dans `config/conf.d` ;
- un dossier dans `log` ;
- une entrée DNS adaptée.

L'étape suivante consiste à sécuriser les sites avec HTTPS et des certificats TLS obtenus avec Let's Encrypt.
