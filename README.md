# Géocontrib

Voici la procédure ainsi que les fichiers configurés pour faire fonctionner l'application sur nos serveurs.

## Prérequis

Il est tout d'abord nécessaire de demander à la DSI de créer une entrée DNS pour le serveur sur lequel se trouvera l'application. Dans le cas de cette procédure, c'était créer "site.domaine.fr" pour le serveur xxx.xxx.xxx.xxx.

Ensuite, la procédure qui va suivre considère les points suivants :
- vous utilisez un seul serveur Linux (Debian) pour héberger le back et le front de l'application
- vous avez une base PostgreSQL à disposition

## Installation des outils et dépendances

Le back est une application django (Python) et le front du VueJS (node.js), nous allons donc installer les outils pour avoir ces 2 environnements à disposition pour faire tourner les applications. Il est également nécessaire d'installer des outils sur le serveur.

### Dépendances Linux

En suivant la [documentation du back](https://github.com/neogeo-technologies/geocontrib), il faut installer ces dépendances au 01/10/2024 (vérifier sur la page github du projet s'il y en a des nouvelles) :

```
# Ces dépendances sont requises pour les fonctionnalités de base du projet, telles que l'interaction avec une base de données PostgreSQL, la manipulation de fichiers géospatiaux, etc.
apt-get install -y libproj-dev gdal-bin ldap-utils libpq-dev libmagic1

# Ces dépendances sont nécessaires pour compiler et utiliser la bibliothèque Pillow, qui est utilisée pour le traitement d'images (depuis passage à python 3.12).
apt-get install -y libjpeg-dev zlib1g-dev libpng-dev
```

### Environnement Python

Au 01/10/2024, il est conseillé d'utiliser la version 3.12 de Python pour faire fonctionner le back d'après la documentation.

Celle-ci indique d'utiliser la version Python du système Linux pour générer un virtualenv, ce que nous n'allons pas faire pour faciliter la gestion des versions de Python pour d'autres projets sur la même machine.

Pour faire cela, nous allons utiliser l'outil ["uv" d'Astral](https://github.com/astral-sh/uv). Pour l'installer, il faut exécuter :

```
curl -LsSf https://astral.sh/uv/install.sh | sh
# Source ou redémarrer le terminal pour charger l'outil
# Pour vérifier l'Installation
uv --version
```

**Remarque :**
Si vous avez une erreur SSL avec uv pour créer un environnement ou installer des packages, vous pouvez rajouter l'argument suivant pour passer outre la vérification SSL, à utiliser avec précaution.

```
uv --native-tls <reste de la commande>
```

Ensuite, nous allons créer un dossier dans lequel nous allons placer les fichiers du front et du back. A la racine de celui-ci nous allons placer notre virtual env Python. Vous pouvez créer le dossier où vous voulez.

```
mkdir geocontrib
cd geocontrib
# Pour initialiser votre environnement uv
uv init
# on supprime le fichier hello.py
rm hello.py
# on crée l'environnement avec la version 3.12 de Python
uv venv .venv --python 3.12
# on active l'environnement
source .venv/bin/activate
```

Maintenant, nous allons passer à l'installer de l'environnement node.js pour le front.

### Environnement node.js

Comme pour l'environnement Python, nous allons installer un outil qui va permettre de gérer plusieurs versions de node.js facilement.

Pour cela, nous allons installer [nvm](https://github.com/nvm-sh/nvm), mais vous pouvez utiliser un autre outil (par exemple [Volta](https://github.com/volta-cli/volta?tab=readme-ov-file)).

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
# Source ou redémarrer le terminal pour charger l'outil
nvm --version
```

Nous allons installer la version 14 de node.js d'après la dernière configuration docker du front et du dernier commit indiquant une erreur en utilisant la version 18 ([bug version 18](https://git.neogeo.fr/geocontrib/geocontrib-frontend/-/commit/dcba2acae886bf9b5b152aab475d37db0edeba4a)). La dernière version disponible au 01/10/2024 est la v14.21.3.

```
nvm install 14
```

De là, il suffira de faire `nvm use 14` pour charger cette version pour lancer le front et le build.

Maintenant que nous avons les outils, nous allons télécharger le back et le front sur notre serveur.

## Installation du back et du front

Allez dans le dossier geocontrib créé précèdemment et faite :

```
# back
git clone https://git.neogeo.fr/geocontrib/geocontrib-django.git
# front
git clone https://git.neogeo.fr/geocontrib/geocontrib-frontend.git
```

De là, en faisant un `ls -a` vous devriez avoir ceci :

```
geocontrib-django  geocontrib-frontend  .git  .gitignore  pyproject.toml  .python-version  README.md  .venv
```

Avant de passer à la configuration du back, pensez à charger votre environnement Python `source .venv/bin/activate`.

## Configuration du back

Tout d'abord, allez dans le dossier `geocontrib-django` et installez les dépendances du back, puis créez le projet Django :

```
# Installer les dépendances
uv pip install -r requirements.txt
# Créer le projet
django-admin startproject config .
```

Le dossier nommé `config` est ainsi créé. Copiez `config-sample/settings.py` dans `config/settings.py`.

Ensuite, créez un fichier `.env` à la racine du dossier `geocontrib-django` (ou renommez le fichier `env_sample` en `.env`). Dedans, copiez-collez les lignes suivantes pour configurer votre instance Django :

```
SECRET_KEY= # à définir
DEBUG=False # False en production, True pour la mise en place.
ALLOWED_HOSTS=site.domaine.fr # DNS défini pour votre serveur permettant d'atteindre l'app
CSRF_TRUSTED_ORIGINS=https://site.domaine.fr # j'avais un message d'erreur me demandant d'ajouter cette valeur
DB_NAME=
DB_USER=
DB_PWD=
DB_HOST=
DB_PORT=
# EMAIL_BACKEND="django.core.mail.backends.smtp.EmailBackend" # la méthode par défaut de django qui est modifiable, remplacer smtp par console si on ne veut pas envoyer le mail, mais le voir quand même dans la console
# EMAIL_HOST="xxx.xxx.xxx.xxx"
# EMAIL_HOST_USER="" # laisser vide si envoie de mail interne
# EMAIL_HOST_PASSWORD="" # laisser vide si envoie de mail interne
# EMAIL_PORT=25
# EMAIL_USE_TLS=False
# DEFAULT_FROM_EMAIL="geocontrib@villeneuvedascq.fr"
BASE_URL="https://site.domaine.fr" # par défaut, "http://localhost:8000", mais en cas de production, il faut mettre le base url de votre site, sinon lors de l'envoie de mail, vous aurez http://localhost:8000/geocontrib/projet/2-localisation-des-projets-immobiliers/type-signalement/6-residence-etudiante/signalement/88252184-e570-4dcc-8853-90d0af446515/ ce qui n'est pas correct.
```

En allant sur la documentation officielle, ou dans les fichiers dans `config/`, vous verrez d'autres paramètres modifiable pour configurer votre application. Ajoutez ceux-ci dans votre fichier `.env` au besoin.

Copiez-collez ensuite le contenu de `config-sample/urls.py` dans `config/urls.py` et modifiez le si nécessaire.

Ensuite, nous allons créer les tables nécessaires pour l'application dans notre base PostgreSQL. Pour cela, il faut effectuer les commandes suivantes :

```
python manage.py migrate
python manage.py loaddata geocontrib/data/perm.json
python manage.py loaddata geocontrib/data/flatpages.json
python manage.py loaddata geocontrib/data/geocontrib_beat.json
python manage.py loaddata geocontrib/data/notificationmodel.json
```

Si vous avez le message suivant, ignorez le :

```
Sites not migrated yet. Please make sure you have Sites setup on Django Admin
```

Ensuite, la documentation conseille l'installation des images et icônes par défaut de l'application (que vous pouvez modifier au besoin):

```
mkdir media
cp geocontrib/static/geocontrib/img/default.png media/
cp geocontrib/static/geocontrib/img/logo.png media/
cp geocontrib/static/geocontrib/img/logo-neogeo*.png media/
```

Ensuite, nous allons créer le **superutilisateur** de l'application avec la commande :

```
python manage.py createsuperuser
```

Pensez à enregistrer les informations que vous allez écrire.

Enfin, en prévision de la mise en production, nous allons effectuer la commande :

```
python manage.py collectstatic
```

Celle-ci va générer un dossier à la racine du projet et nommé `static`. Ce dossier contient les fichiers qui seront servis directement par notre serveur web et non par le back à la volée. Ce dossier n'est pas nécessaire si vous laisser la variable `DEBUG=True`, ce qui n'est pas conseillé en production, raison pour laquelle nous le générons. 

Voilà, la configuration du back est terminé, vous pouvez à présent démarrer l'application en effectuant la commande suivante :

```
python manage.py runserver
```

Celle-ci va exécuter l'application sur le port par défaut, ce que nous ne voulons pas forcément, surtout si vous en avez indiqué un autre dans le fichier `.env` dans l'attribut `BASE_URL`. Dans ce cas, ajoutez le port dans la commande précédente, par exemple pour le port 9999 :

```
python manage.py runserver 10002
```

Maintenant, nous allons passer au front.

## Configuration du front

Allez dans le dossier du front et faite `nvm use 14` si ce n'est pas encore fait. Ensuite, faites `npm install` pour installer toutes les dépendances du projet (comme pur le projet Python).

Ensuite, selon la [documentation](https://git.neogeo.fr/geocontrib/geocontrib-frontend), il faut créer un fichier `.env` dans lequel nous allons mettre :

```
BASE_URL=/geocontrib/
NODE_ENV=development
```

Cela sert potentiellement lorsque nous faisons du développement en faisant un serveur local, mais ça n'a pas l'air utile en l'état, peut-être un héritage d'une ancienne version qui n'a pas été mis à jour.

Le fichier le plus important à modifier est `public/config/config.json.sample` qu'il faut renommer en `config.json` pour qu'il soit pris en compte par l'application. Dans ce fichier, nous allons remplacer les éléments suivants :

```
{
    "BASE_URL":"/geocontrib/",
    "DOMAIN":"http://localhost/", -> "/geocontrib/"
    "NODE_ENV":"development", -> Possible de supprimer la ligne
    "VUE_APP_LOCALE":"fr-FR",
    "VUE_APP_APPLICATION_NAME":"GéoContrib", -> "A modifier si envie, c'est le tête en haut à gauche de l'interface"
    "VUE_APP_APPLICATION_FAVICO":"/geocontrib/img/geo2f.ico", -> idem si envie
    "VUE_APP_APPLICATION_ABSTRACT":"Application de saisie d'informations géographiques contributive", -> idem si envie
    "VUE_APP_LOGO_PATH":"/geocontrib/img/logo-neogeo-circle.png", -> idem si envie
    "VUE_APP_DJANGO_BASE":"http://localhost:8010", -> "https://site.domaine.fr" qui est la racine web de votre application
    "VUE_APP_DJANGO_API_BASE":"http://localhost:8010/api/", -> "/geocontrib/api/" qui est le chemin à ajouter à l'url de base pour atteindre l'api
    "VUE_APP_CATALOG_NAME": "Datasud", -> Vous pouvez supprimer la ligne, de cette façon dans l'interface, quand vous allez vouloir créer un signalement, vous n'aurez pas la ligne vous proposant de charger une couche de Datasud
    "VUE_APP_IDGO": true, -> Vous pouvez mettre en False si vous n'avez pas la plateforme IDGO pour enlever le bouton de l'interface comme pour Datasud
    "VUE_APP_RELOAD_INTERVAL": 15000,
    "VUE_APP_DISABLE_LOGIN_BUTTON":false,
    "VUE_APP_LOGIN_URL":"",
    "VUE_APP_FONT_FAMILY":"",
    "VUE_APP_HEADER_COLOR":"",
    "VUE_APP_PRIMARY_COLOR":"",
    "VUE_APP_PRIMARY_HIGHLIGHT_COLOR" : "",
    "DEFAULT_BASE_MAP_SCHEMA_TYPE": "tms",
    "DEFAULT_BASE_MAP_SERVICE": "https://{s}.tile.openstreetmap.fr/osmfr/{z}/{x}/{y}.png",
    "DEFAULT_BASE_MAP_OPTIONS": {
            "attribution": "&copy; contributeurs d'<a href='https://osm.org/copyright'>OpenStreetMap</a>",
            "maxZoom": 20
    },
    "DEFAULT_MAP_VIEW" : {
        "center": [47.0, 1.0],
        "zoom": 4
    },
    "MAP_PREVIEW_CENTER" : [43.60882653785945, 1.447888090697796],
    "DISPLAY_FORBIDDEN_PROJECTS": true,
    "DISPLAY_FORBIDDEN_PROJECTS_DEFAULT": true,
    "VUE_APP_URL_DOCUMENTATION": "https://www.onegeosuite.fr/docs/module-geocontrib/intro",
    "VUE_APP_URL_DOCUMENTATION_FEATURE": "https://www.onegeosuite.fr/docs/module-geocontrib/project_settings"
}
```

Cette configuration proposée est minimale, vous pouvez compléter celui-ci selon va besoin de personnalisation, par exemple le thème de celle-ci en mettant des codes hexadécimaux de couleur pour les attributs "xxx_COLOR" ou en changeant les icônes et logos.

Ce fichier est diffusé par l'application quand vous regardez les éléments chargés dans le navigateur avec F12, donc faites attention à ce que vous mettez dedans. De ce fait, vous pouvez regarder ceux des autres services Géocontrib sur internet pour comparer et vous en inspirer.

Maintenant, nous allons build l'application pour son utilisation avec Apache2 (vous avez un fichier Nginx à la racine du projet si vous utilisez ce type de serveur) :

```
npm run build
```

En faisant cela, vous allez avoir un nouveau dossier nommé `dist` qui va contenir l'application et la configuration que vous avez défini. Vous devez déplacer ce fichier à l'endroit où vous mettez habituellement vos applications Apache2, dans mon cas c'est dans `/var/www/apps/` et je renomme le dossier en `geocontrib`.

Maintenant, nous allons passer à la configuration d'Apache2.

Remarque : si vous souhaitez faire du développement, allez sur la page Gitlab de l'application pour voir comment faire, je n'aborde pas cela dans cette documentation.

## Configuration d'Apache2

Si vous regardez dans les fichiers de l'application front, vous pouvez voir un fichier nommé `conf_apache_dev.md`, mais les informations qui se trouvent dedans ne sont pas correctes dans le cadre d'une mise en production, ni pour une utilisation conventionnelle d'Apache2 puisqu'ils conseillent de modifier directement le fichier Apache2.conf et non de faire un VirtualHost.

Donc, voici la "bonne" procédure à suivre pour faciliter la maintenabilité et la cohérence avec vos autres applications.

Ainsi, il faut déjà vérifier si vous avez installé et activé les extensions suivantes pour Apache2 :

* mod_headers : `sudo a2enmod headers`
* mod_proxy : `sudo a2enmod proxy`
* mod_ssl : `sudo a2enmod ssl`
* mod_proxy_http : `sudo a2enmod proxy_http`

Une fois fait, vous allez créer un fichier domaine.conf (ce n'est pas obligatoire, mais conseillé) dans `/sites-availables/`, qui est dans le cas présent `site.domaine.fr.conf`.

Dedans, vous allez copier-coller les lignes ci-dessous qui sont inspirées de la configuration Nginx de l'application :

```
<IfModule mod_ssl.c>
	<VirtualHost *:443>
		ServerName site.domaine.fr # A ADAPTER

		# Mail du webmaster
		ServerAdmin mail@xxx.fr # A ADAPTER

		DocumentRoot /var/www/apps/geocontrib # A ADAPTER

		AddDefaultCharset UTF-8

		LimitRequestBody 4294967296

		# Redirection de la racine vers /geocontrib/
		RedirectMatch 301 ^/$ /geocontrib/

		Alias /geocontrib /var/www/apps/geocontrib # A ADAPTER
		Alias /geocontrib/static/ /var/www/apps/geocontrib/static # A ADAPTER

		<Directory /var/www/apps/geocontrib> # A ADAPTER
			Options -MultiViews
			AllowOverride None
			Require all granted

			DirectoryIndex index.html
			FallbackResource /geocontrib/index.html
		</Directory>

		RequestHeader set X-Forwarded-Proto "https"
		RequestHeader set X-Forwarded-Port "443"

		# Délocaliser pour ce vhost les logs d'erreur
		ErrorLog ${APACHE_LOG_DIR}/error.log
		# Délocaliser pour ce vhost les logs d'accès
		CustomLog ${APACHE_LOG_DIR}/access.log combined

		# SSL
		SSLEngine on
		SSLCertificateFile      /etc/ssl/wildcard/server.crt # A ADAPTER
		SSLCertificateKeyFile   /etc/ssl/wildcard/server.key # A ADAPTER

		# Proxy permettant pour les applications qui tournent sur un port, de
		# faire l'équivalent d'un alias
		ProxyRequests off
		ProxyPreserveHost On
		ProxyVia On

		<Proxy *>
			Order Deny,Allow
			Allow from all
		</Proxy>

		# Geocontrib api django
		ProxyPass /geocontrib/api/ ws://localhost:10002/geocontrib/api/ # A ADAPTER SELON PORT DEFINI
		ProxyPassReverse /geocontrib/api/ ws://localhost:10002/geocontrib/api/ # A ADAPTER SELON PORT DEFINI

		ProxyPass /geocontrib/cas/ ws://localhost:10002/geocontrib/cas/ # A ADAPTER SELON PORT DEFINI
		ProxyPassReverse /geocontrib/cas/ ws://localhost:10002/geocontrib/cas/ # A ADAPTER SELON PORT DEFINI

		ProxyPass /geocontrib/admin/ ws://localhost:10002/geocontrib/admin/ # A ADAPTER SELON PORT DEFINI
		ProxyPassReverse /geocontrib/admin/ ws://localhost:10002/geocontrib/admin/ # A ADAPTER SELON PORT DEFINI

		ProxyPass /geocontrib/media/ ws://localhost:10002/geocontrib/media/ # A ADAPTER SELON PORT DEFINI
		ProxyPassReverse /geocontrib/media/ ws://localhost:10002/geocontrib/media/ # A ADAPTER SELON PORT DEFINI

		# Uniquement en DEBUG=True, sinon on utilise le static dir qui est placé dans le dossier sur /var/www/etc.
		# ProxyPass /geocontrib/static/ ws://localhost:10002/geocontrib/static/ # A ADAPTER SELON PORT DEFINI
		# ProxyPassReverse /geocontrib/static/ ws://localhost:10002/geocontrib/static/ # A ADAPTER SELON PORT DEFINI

	</VirtualHost>
</IfModule>
```

Cette configuration va rendre l'application accessible uniquement en https, quand on va sur l'url `site.domaine.fr` on est redirigé directement sur `site.domaine.fr/geocontrib/` et toutes les urls qui font appels à l'api sont redirigées vers elle, et les fichiers statics pointent bien vers le dossier "dist" que nous avons généré.

De là, il ne reste plus qu'à enable le site et restart/reload le serveur Apache2 :

```
sudo a2ensite <nom du fichier .conf>
sudo systemctl restart apache2
```

Et normalement, vous aurez votre site fonctionnel à présent, bravo !

Concernant l'utilisation de l'application, vous pouvez trouver la documentation à cette adresse : [https://www.onegeosuite.fr/docs/module-geocontrib/intro](https://www.onegeosuite.fr/docs/module-geocontrib/intro). Egalement, vous avez ce webinaire de GéoBretagne qui est sur une ancienne version, mais reste d'actualité : [https://cms.geobretagne.fr/content/replay-webinaire-presentation-dune-experimentation-sur-loutil-geocontrib](https://cms.geobretagne.fr/content/replay-webinaire-presentation-dune-experimentation-sur-loutil-geocontrib).

Il est possible de faire des choses en plus en allant sur le panneau de gestion de l'api accessible via domain.fr/geocontrib/admin/. N'hésitez pas à regarder les tables dans la base PostgreSQL pour comprendre le fonctionnement de l'application.

## Issues

- Lors de la création d'un signalement, il est possible de modifier de le formulaire tant qu'aucun signalement n'a été créé. SI vous souhaitez revoir le schéma, il est possible de passer par le panneau admin de l'api et d'aller dans "Champ personnalisé" pour ajouter un nouvel attribut à un signalement. Si vous souhaitez ajouter une liste de valeur, il faut utiliser les "," comme séparateur. Si vous avez des "," dans la valeur à ajouter, il faut l'entourer de guillemets pour indiquer que c'est une chaîne de caractères. En opérant ainsi, je ne vois pas de soucis, étant donné que les nouveaux signalements ont bien les nouveaux attributs, que les signalemnets déjà effectués ont les nouveaux attributs si on les modifies. Idem dans la base de données, tout semble correct.

- Lors de l'ajout d'un signalement de type ligne et polygone, lorqu'on souhaite créer l'entité directement l'entité, il y a un souci, tous les clics ne sont pas pris en compte. Il est donc conseillé au 07/10/2024 de terminer directement l'entité, en posant 2 noeuds (ou 4 pour les polygones), en supprimant celle-ci ensuite et en créant une nouvelle. De là, l'outil de création d'entité est parfaitement fonctionnel !

## TODO

- Voir comment stocker les new et old value lors d'une mise à jour d'un signalement pour retourner dans le mail de notification les champs qui ont été modifiés. Même principe que les audit postgresql (stocker dedans ? Genre avoir une table d'audit et prendre le dernier audit de type update pour l'entité en question ? Est-ce qu'il y a un statut "à envoyer" dans la bdd pour retrouver rapidement les features à envoyer et donc, sur lesquelles joindre des données ? https://blog.dalleryd.fr/posts/2023-11-26-audit-postgresql-postgis-qgis-1-3/)
