* Out of place as in no terraform in this tut but ...
* This was on the road to terraform django by the same person
# Dockerizing Django with Postgres, Gunicorn and Nginx
[Terraform project](../django-ecs-terraform/app/notes.md)  That led to this as a stepping stone

Django 3.2.6
Docker 20.10.8
Python 3.9.6
Not trying to comply with the versions unless my current defaults fail

# Setup
```
$ mkdir django-on-docker && cd django-on-docker
$ mkdir app && cd app
$ python3.9 -m venv env
$ source env/bin/activate
(env)$

(env)$ pip install django==3.2.6
(env)$ django-admin.py startproject hello_django .
(env)$ python manage.py migrate
(env)$ python manage.py runserver
```
## Under app:\
* hello_django
    * wsgi.py
* manage.py
* requirements.txt


```
docker-compose up -d --build
docker-compose exec web python manage.py migrate --noinput
docker-compose exec db psql --username=hello_django --dbname=hello_django_dev
docker volume ls
docker volume inspect dockerize2_postgres_data
<!-- Note that the volume name seems to begin with the name of the containing directory (dockerize2 here) -->
```

## Beginning of _03_dev STUFF

### [Dockerfile](app/Dockerfile)
### [docker-compose.yml](docker-compose.yml)
https://docs.docker.com/compose/compose-file/
* Update [settings.py](app/hello_django/settings.py)
```python
SECRET_KEY = os.environ.get("SECRET_KEY")

DEBUG = int(os.environ.get("DEBUG", default=0))

# 'DJANGO_ALLOWED_HOSTS' should be a single string of hosts with a space between each.
# For example: 'DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]'
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS").split(" ")
```




```
docker ps
docker exec -it <name> /bin/sh
```
## _04_dev
Run a different image that uses postgres
```
docker build -f ./app/Dockerfile -t hello_django:latest ./app
docker run -d -p 8006:8000 -e "SECRET_KEY=please_change_me" -e "DEBUG=1" -e "DJANGO_ALLOWED_HOSTS=*" hello_django python /usr/src/app/manage.py runserver 0.0.0.0:8000
```

* The migrate and flush commands can be commented in entrypoint.sh to avoid running them every time and fun manually
```
docker-compose exec web python manage.py flush --no-input
docker-compose exec web python manage.py migrate
```
### Production 
#### Gunicorn
* gunicorn==20.1.0 to requirements.txt
* Create files 
    * docker-compose.prod.yml
    * .env.prod
    * .env.prod.db
* 3 changes to docker-compose, Refer to the two new .env files (no environemnt variables) and change the web service command to run gunicorn instead of runserver

```docker-compose -f docker-compose.prod.yml up -d --build```
```docker-compose -f docker-compose.prod.yml down -v```

end _04_prod

* Create entrypoint.prod.sh remove migrate and flush
* Create Dockerfile.prod to run it
* Alter docker-compose.prod.yml 
    * to run the Dockerfile.prod
    * remove the volumes: for /usr/src/app  --- The multi stage build in Dockerfile.prod uses the directory for build. Bui packages are created as wheels there and the finished packages are used the in final build. That is the point of the multistage build -- to make the final smaller
```
docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
```
* migrate is out of the build and has to be run manually

end _05_prod

## nginx
* add nginx/Dockerfile and nginx/nginx.conf
* Dockerfile replaces the default nginx.conf with this one
* nginx.conf instructs nginx to listen on :80 for hello_django
* replace the port : directive with expose: in docker-compose.prod and add nginx service that depends on web
    * port 1337:80 (?)
* rebuild ...
https://docs.nginx.com/nginx/admin-guide/web-server/app-gateway-uwsgi-django/
https://stackoverflow.com/questions/40801772/what-is-the-difference-between-docker-compose-ports-vs-expose
end _06_prod

### After the last one, the app is running on 1337 and we have lost static files. The /admin page is not styled
Edit settings.py 
```
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
```
Run the dev server and check that /admin works 
```
docker-compose -f docker-compose.prod.yml down -v
docker-compose up -d --build
```
end _07_dev

### collectstatic (must be run)
* In docker-compose.prod.yml add static_volume to targets
    * web
    * nginx
    * top level volumes
* In Dockerfile.prod add to 'create the appropriate directories' section
    * ```RUN mkdir $APP_HOME/staticfiles```
* In nginx.conf add location /static/ to server object

* If the developments server is still running
```docker-compose down -v```
```
docker-compose -f docker-compose.prod.yml up -d --build
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear
```
#### verify
* view to verify
    * localhost:1337
    * localhost:1337/admin
* look at chrome web tools to see files served
* Check logs
    * ```docker-compose -f docker-compose.prod.yml logs -f```

_end _08_prod

## Media files Development Server
Bring down prod
build dev
startapp upload on dev container

## Create new  app
* If prod is still running
```docker-compose -f docker-compose.prod.yml down -v```

```
docker-compose up -d --build
docker-compose exec web python manage.py startapp upload
```
#### Edit settings

* add upload to INSTALLED_APPS
```
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "mediafiles"
```
 #### Other files for upload app
* add to hello_django/urls.py
    *  ```path('', image_upload, name="upload"),```
* .gitignore 
    * add media to .gitignore (or create .gitignore -note the the .env files are in git already)
* upload/views.py
    * add upload_image to views.py
* upload/templates/upload.html

### Test and upload an image
* If the Dev server is up
```docker-compose up -d --build```

end _09_dev

##  Medial files Production specific
* add media_volume to  both web.volumes and nginix.volumes in docker-compose.prod.yml
    * ```- media_volume:/home/app/web/mediafiles```
    * also media_volume to the top level volumes
* Dockerfile.prod add mediafiles to the "create the appropriate directories section"
    * ```RUN mkdir $APP_HOME/mediafiles```
* nginx.conf ```location /media/``` to server object
* ```alias /home/app/web/mediafiles/;```

#### Test
* if development server is running
```docker-compose  -v```
* Required commands build, migrate, collectstatic
```
docker-compose -f docker-compose.prod.yml up -d --build
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --noinput --clear
```
end _10_prod


