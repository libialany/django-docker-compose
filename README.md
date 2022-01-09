### Development the application

specifying the image

```
FROM python:3.9-alpine3.13
LABEL maintainer=""
```
create 

ENV PYTHONUNBUFFERED 1

copying the requirements and the application

```
COPY ./requirements.txt /requirements.txt
COPY ./app /app
```

```
WORKDIR /app
EXPOSE 8000
``` 

Note we use the forward slash to optimize the creation of layers because if we execute separate commands we are creating more layers

```
RUN python -m venv /py && \
	/py/bin/pip install --upgrade pip && \
	/py/bin/pip install -r /requirements.txt && \
	adduser --disabled-password --no-create-home app
```

we create a variable of environment so as not to have to write from the path / py / bin

```
ENV PATH="/py/bin:$PATH"
```
run as app not root

```
USER app
```



### Development the service side

build our dockerfile

```
    build:
      context: .
``` 

porst open 

```
    ports:
      - 8000:8000
```

###  create env variables 

1.set some values in settings.py

```
SECRET_KEY= os.environ.get('SECRET_KEY')

DEBUG = bool(int(os.environ.get('DEBUG'),0))

ALLOWED_HOSTS = []

ALLOWED_HOSTS.extend(
        filter(
            None,
            os.environ.get('ALLOWED_HOSTS','').split(','),
        )
)

``

### databases set up

1. db container shoud star before the app container

2. There should be anetwork connection set up the app and the db container .

3. DB settings must be the same.
 
```
    environment:
      - SECRET_KEY=devsecretkey
      - DEBUG=1
      - DB_HOST=db
      - DB_NAME=devdb
      - DB_USER=devuser
      - DB_PASS=changeme
    depends_on:
      - db
  db:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=devuser
      - POSTGRES_PASSWORD=changeme

```

### driver packages

- Install some packages that are needed for our  postgres driver.

```
RUN python -m venv /py && \

        /py/bin/pip install --upgrade pip && \

	# install

        apk add --update --no-cache postgresql-client && \

	# add

        apk add --update --no-cache --virtual .tmp-deps \


                build-base postgresql-dev musl-dev && \


        /py/bin/pip install -r /requirements.txt && \
	# delete 

        apk del .tmp-deps && \

        adduser --disabled-password --no-create-home app

``` 

change the default setup on settings.py .

```

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': os.environ.get('DB_HOST'),
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASS'),
    }
}
```
### adding an  app

```
docker-compose run --rm app sh -c "python manage.py startapp core"
```

create a model in core>models.py 

```
class Sample(models.Model):
    attachment = models.FileField()
```

register in core>admin.py

```
from core.models import Sample

admin.site.register(Sample)

```

add a management file for management the db.

-----
> mkdir management
>	mkdir commands
>		wait_for_db.py
-----


wait_for_db.py that file create a comand for handle exception


### settings static


1.you create the respective folders: static and media  you give them the permissions and the owner.

dockerfile
```
    adduser --disabled-password --no-create-home app && \
        mkdir -p /vol/web/static && \
        mkdir -p /vol/web/media && \
        chown -R app:app /vol && \

```
2. configure the storage path
docker...yml

```
    volumes:
      - ./app:/app
      - ./data/web:/vol/web

```

3. create the variables in setting.py

```
STATIC_URL = '/static/static/'
MEDIA__URL = '/static/media/'

MEDIA_ROOT = '/vol/web/media'
STATIC_ROOT = '/vol/web/static'

```

4. add those variables in urls.py

```
from django.contrib import admin
from django.urls import path
from django.conf.urls.static import static
from django.conf import settings

if settings.DEBUG:
    urlpatterns += static(
    settings.MEDIA_URL,
    document_root=settings.MEDIA_ROOT,
    )

```

#### create admin count

```
docker-compose  run --rm app sh -c "python manage.py createsuperuser"
``

### setting the proxy 

1. create proxy folder

2. create Dockerfile.

```
# 

FROM nginxinc/nginx-unprivileged:1-alphine

#


COPY ./default.conf.tpl /etc/nginx/default.conf.tpl
COPY ./uwsgi_params /etc/nginx/uwsgi_params
COPY ./run.sh /run.sh
# for second proxy
ENV APP_PORT=9000

USER root

RUN mkdir -p /vol/static && \
        chmod 755 /vol/static && \
        touch /etc/nginx/conf.d/default.conf && \
        chown nginx:nginx /etc/nginx/conf.d/default.conf && \
        chmod +x /run.sh
CMD [ "/run.sh" ]
```
create default.conf.tpl to configure nginx

```server {
        listen ${LISTEN_PORT};
	# first location
        location /static {
                alias /vol/static
        }
	# second location for uwsgi
        location / {
                uwsgi_pass      ${APP_HOST}:${APP_PORT};

                include         /etc/nginx/uwsgi_params;
		#max size requests 
                client_max_body_size    10M;
        }

}```

create a run.sh

```
#!/bin/sh #this is the shell of alphine

set -e 
# pass the configuration we made
envsubst < /etc/nginx/default.conf.tpl > /etc/nginx/conf.d/default.conf
# daemon off for more security
nginx -g 'daemon off;'
```



###  configure our Django app to run as a uWSGI service


create   a folder scripts add run.sh

explained more is comented

```
#!/bin/sh

set -e

# run the db first
python manage.py wait_for_db
# get all static files in all project
python manage.py collectstatic --noinput
# update 
python manage.py migrate

# this for uwsgy config which is placed in >app > wsgi.py
uwsgi --socket :9000 --workers 4  --master --enable-threads --module app.wsgi

```

last one add in  requirements.txt
```
uWSGI>=2.0.19.1,<2.1
```
configure in Dockerfile our main script

```
COPY ./scripts /scripts
      chmod -R +x /scripts
ENV PATH="/scripts:/py/bin:$PATH"
CMD ["run.sh"]
```

create docker-compose-deploy.yml and then set all configurations.

next , create a file for save environmnets variables  we set in docker-compose-deploy.yml  (.env).

last execute those commands

1. docker-compose -f docker-compose-deploy.yml down --volumes

2. docker-compose -f docker-compose-deploy.yml build

3. dont forget docker-compose -f docker-compose-deploy.yml run --rm app sh -c 'python manage.py createsuperuser'  
