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
docker exec -it <name> /bin/bash
```

Run a different image that uses postgres
```
docker build -f ./app/Dockerfile -t hello_django:latest ./app
docker run -d -p 8006:8000 -e "SECRET_KEY=please_change_me" -e "DEBUG=1" -e "DJANGO_ALLOWED_HOSTS=*" hello_django python /usr/src/app/manage.py runserver 0.0.0.0:8000
```

* The migrate and fluch commands can be commented in entrypoint.sh to avoid running them every time and fun manually
```
docker-compose exec web python manage.py flush --no-input
docker-compose exec web python manage.py migrate
```
### Production 
#### Gunicorn
* to requirements.txt
* [docker-compose.prod.yml](./docker-compose.prod.yml)
This differs from the final version. Observations
* command is gunicorn not runserver
* ports 8000:8000 differs from final
* .env.prod instead of variables (why are they in the dev version?)
* .env.prod.db   (seperate file???)
```docker-compose -f docker-compose.prod.yml up -d --build```