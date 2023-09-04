# grafana-borgia
A Grafana instance to monitor the Borgia application


# log

1. Installation des librairies

Sous l'environneemnt virtuel éxécutant votre application Django faire une installation du fichier requirements.txt.

```sh
pip -r requirements.txt
```

2. Copier le dossier log dans le dossier contenant toute votre application

> Cas de Borgia le copier sous : /borgia-app/Borgia/borgia

3. Dans le fichier settings.py de votre application Django, copier les lignes suivantes : 

```py

import os
from log.logger import CustomisedJSONFormatter

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "simple": {
            "format": "[%(asctime)s] %(levelname)s|%(name)s|%(module)s|%(funcName)s|%(message)s",
            "datefmt": "%Y-%m-%d %H:%M:%S",
        },
        "json": {
            "()": CustomisedJSONFormatter,
        },
    },
    "handlers": {
        "applogfile": {
            "level": "DEBUG",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "PATH_TO_YOUR_APP_LOG_FILE/app.log",
            "maxBytes": 1024 * 1024 * 15,  # 15MB
            "backupCount": 10,
            "formatter": "json",
        },
        "console": {
            "level": "DEBUG",
            "class": "logging.StreamHandler",
            "formatter": "simple",
        },
    },
    "root": {
        "handlers": ["applogfile", "console"],
        "level": "DEBUG",
    },
}

```

A la ligne : "filename": "PATH_TO_YOUR_APP_LOG_FILE/app.log" remplacer le PATH_TO_YOUR_APP_LOG_FILE avec le chemin vers votre fichier app.log.

Attention également a veiller bien inclure le fichier logger.py. 

Une fois toutes ces actions effectuées vous pouvez verifier que votre application Django écrit bien ses logs au format souhaités et dans le fichier app.log

Pour ecrire un log depuis une fonction : 

```py 
import logging
logger = logging.getLogger(__name__)
logger.debug("This is a debug log")
logger.critical("This is a critical log")
```

4. Installation de Grafana

Pour installer Grafana plusieurs options s'offrent à vous. L'installer directement sur l'OS, via Docker ou via un fichier Docker compose. 

```
docker run -d --name=grafana -p 3000:3000 grafana/grafana
```

Cette commande vous permet d'installer grafana dans un docker.

Une fois le conteneur lancé, un serveur grafana doit etre disponible à l'adresse http://localhost:3000/

> Lorsque que vous etablissez une connection SSH avec l'editeur VS Code, il est possible avec VS Code d'utiliser une redirection de port : https://code.visualstudio.com/docs/remote/ssh#_forwarding-a-port-creating-ssh-tunnel
> Dans votre cas creer une redirection du port 3000 vers le port 3000 de votre machine. 

5. Pour accéder à Grafana depuis le web il est necessaire d'utiliser un reverse proxy, NGINX est parfait pour cela. 

Installet NGINX : 

```sh

sudo apt-get update
sudo apt-get install nginx

```

Créez un fichier de configuration, par exemple, grafana.conf, dans le répertoire /etc/nginx/sites-available/ avec le contenu suivant :

```
server {
    listen 80;
    server_name grafana.mysite.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/grafana_access.log;
    error_log /var/log/nginx/grafana_error.log;
}
```

Une fois tout cela effectué creer un lien vers /etc/nginx/sites-enabled/

Verifier la configuration puis redémarrer NGINX, votre site devrait etre accessible à l'addresse : grafana.mysite.com 
```
sudo nginx -t
sudo systemctl restart nginx
```

--- 
TODO 

Si vous utiliser un gestionnaire de machine virtuel comme Proxmox par exemple, vous devrez utiliser un reverse proxy comme haproxy pour rediriger la connection entrante vers le site. Dans haproxy une declaration du backend est necessaire dans haconfig 

TODO

---

5. Installation de Loki et de Promtail 

Loki est un système de gestion des journaux open source conçu pour collecter, stocker et interroger des journaux de manière efficace et évolutive. 

Promtail est un agent de collecte de journaux développé par Grafana, conçu pour extraire des journaux de diverses sources, les enrichir et les expédier vers Loki. Il facilite la collecte centralisée et la recherche de journaux au sein d'un cluster ou d'une infrastructure distribuée. 

Promtail est l'instance qui va aller chercher les logs dans votre fichier app.log et les renvoyer vers un serveur Loki qui lui va transmettre ces logs vers une instance Grafana. 


Pour l'installation de Loki et Promtail nous allons passer par Docker compose. 

Pour cela un fichier docker-compose.yaml est disponible. 
Dans votre fichier docker-compose.yaml vous devez definir les deux paths de volumes suivant : LOCAL_PATH_TO_YOUR_DJANGO_LOG_FOLDER et PATH_TO_PROMTAIL_CFG. 

LOCAL_PATH_TO_YOUR_DJANGO_LOG_FOLDER est le path vers votre dossier log de votre application django. PATH_TO_PROMTAIL_CFG est le path vers le fichier promtail-config.yml telechargé.

- LOCAL_PATH_TO_YOUR_DJANGO_LOG_FOLDER/:/log/
- PATH_TO_PROMTAIL_CFG/promtail-config.yml:/etc/promtail/config.yml

Ce binding de volumes permet dans le container docker promtail d'utiliser le fichier promtail-config.yml comme source de configuration. Cela creer une copie de du fichier locale à l'intérieur de votre conteneur docker. 

Le binding des dossiers de log permet a promtail d'accéder au dossier de log dynamiquement et donc a chaque modification du fichier app.log, permettre a promtail de lire les mofications de ce fichier

Pour lancer le docker compose se placer dans le dossier contenant le fichier docker-compose.yaml et lancer la commande : 

docker-compose -f docker-compose.yaml up

Si vous souhaiter la lancer en mode daemon (en tache de fond) ajouter à l'issuer de la commande un -d

Si tout se passe bien lorsque que vous excuter la commande docker ps, 3 conteneurs actifs doivent apparaitre. Si rien ne s'affiche verifier si ils ne sont pas eteint via la commande docker ps -a. Si un d'eux est eteint, il y a surement une erreur, verifier les logs. 

6. Ajouter le conteneur Grafana au reseau loki 

Afin de simplifier la communication entre conteneur la creation d'un pont en le conteneur loki et le conteneur grafana est necessaire. Pour ce faire lancer la commande : 

docker network connect lokigrafana_loki grafana

Loki est maintenant visible depuis le conteneur Grafana. Vous pouvez tester la connection en executant la commande : docker exec -ti containerID1 ping containerID2

7. Ajout de Loki a Grafana. 

Rendez vous sur la page de votre serveur Grafana. Dans le menu acceder a la page datasources, cliquer sur ajouter une nouvelle sources de données, selectionner Loki. Dans l'input URL insérer l'url contenant votre instance loki : http://lokigrafana-loki-1:3100. Cliquer sur save. 

Vous pouvez accéder a la page Explore et faire des requetes sur vos logs : 

Voici quelques requêtes simples : 

Cette requete permet d'afficher les logs au format json 

{job="containerlogs"} | json 

Cette requete permet de filtrer les logs et afficher seulement les logs de niveau ERROR. 

{job="containerlogs"} | json json_level="level" | json_level = `ERROR`

Cette requete permet de filtrer en affichant les logs venant seulement du fichier views.py

{job="containerlogs"} | json |= "views.py"


Si vous voulez seulement visualiser un seul champ par exemple quand la func_name de django vaut = myfunc

Taper la requete : {job="containerlogs"} | json 

Aller ensuite dans un dans champ ou func_name = myfunc cliquer sur l'icone + 

{job="containerlogs"} | json | django_funcName=`myfunc` 



Installation de Prometheus. 

De même que Loki et Promtail il est possible d'installer une instance Prometheus afin de surveiller l'application Django avec des metrics comme par exemple le nombre de requête par seconde, le temps de réponse, la latence...

Lors de l'installation du fichier requirements.py le package django-prometheus a été installé ( https://github.com/korfuri/django-prometheus ). Pour l'utiliser rendez vous dans le fichier settings.py de votre application django. 

et modifier les lignes suivantes : 

```py

INSTALLED_APPS = [
   ...
   'django_prometheus',
   ...
]

MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',
    # All your other middlewares go here, including the default
    # middlewares like SessionMiddleware, CommonMiddleware,
    # CsrfViewmiddleware, SecurityMiddleware, etc.
    'django_prometheus.middleware.PrometheusAfterMiddleware',
]

```

dans le fichier urls.py : 

```py
urlpatterns = [
    ...
    path('', include('django_prometheus.urls')),
]
```

Il est egalement possible de monitorer votre base de données. Cependant dans mon cas j'obtiens une erreur. 

> Lien vers l'erreur Github : https://github.com/korfuri/django-prometheus/issues/414

```py

DATABASES = {
    'default': {
        'ENGINE': 'django_prometheus.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    },
}

```

Une votre application django correctement configurée vous pouvez relancer votre serveur et vous rendre à l'url http://your-django-app.com/metrics Si tout s'est passée correctement vous devriez alors observer une liste de metrics. 

Installation de Prometheus. 

Prometheus peut etre installer de différentes manières, une installation bare-metal ou via un conteneur Docker ou autre.

Si vous ne maitriser pas bien Docker une installation bare-metal est à privilégier sinon pour l'installer via docker : docker run -p 9090:9090 prom/prometheus. 


Un configuration possible en bare-metal est la suivante. 

global:
  scrape_interval: 15s
  evaluation_interval: 15s
scrape_configs:
  - job_name: borgiaMonitor
    static_configs:
      - targets:
        - localhost

Avec un conteneur docker il est necessaire de specifier --network="host"

docker run --network="host" -d -p 9090:9090 prom/prometheus

> Precision : J'ai rencontré pas mal de problèmes pour accèder à http://your-django-app.com/metrics si le site utilise HTTPS. Pour contourner ce problème je conseille deployer votre application django a la fois sur internet et à la fois sur le reseau local. Creer simplement un nouveau fichier de configuration NGINX avec le contenu suivant : 
```
server {

    listen 80;
    server_name localhost;

    location = /favicon.ico { access_log off; log_not_found off; }


    location /media  {
        alias  PATH_TO_YOUR_APP/static/media;
    }

    location /static {
        alias PATH_TO_YOUR_APP/static/static_root;
    }


    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }

}

```

Une fois votre serveur Prometheus lancé vous pouvez voir si il repond bien en accédant à http://localhost:9090/targets


Vous pouvez maintenant sous votre panel grafana ajouter une nouvelle source de données, selectionner Prometheus. Dans l'url entrer : http://172.17.0.1:9090

L'adresse 172.17.0.1 permet d'accéder au reseau local de la machine.

Creation du panneau de monitoring Django : 
Afin de visualiser proprement les metrics recupérée il est possible d'utiliser differents dashboard ici un pret à l'emploi : 
https://grafana.com/grafana/dashboards/17658-django/

Simplement telecharger le et importer le en tant que json dans grafana. 
Selectionner en source de données Prometheus et le tour est joué.
