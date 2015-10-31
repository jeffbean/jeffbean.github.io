---
layout: post
title:  "Django in Docker"
date:   2015-10-30 17:09:24 -0700
categories: docker django python
---

# Django project using Docker

## Pre-requisites and assumptions

* Using Django 1.7, Python 3.3, Docker 1.5, docker-compose 1.5
* Assume that python, setuptools, installed
* Assume Docker and docker-compose installed

## Basic setup

First we need to have a Django project that we can build and run in Docker.

    django-admin startproject django_demo
    cd django_demo
    ./manage.py startapp blog

This will provide us a base project and site to expand upon using Docker.

### Reference structure

Reference structure at a glance.

`tree -I *.pyc`

    .
    ├── blog
    │   ├── admin.py
    │   ├── __init__.py
    │   ├── migrations
    │   │   ├── 0001_initial.py
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   ├── urls.py
    │   └── views.py
    ├── django_demo
    │   ├── __init__.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── docker-compose.yml
    ├── Dockerfile
    ├── manage.py
    ├── requirements.txt
    └── run_web.sh

#### requirements.txt

Add the Python packages we need for the demo.

    django>=1.7,<1.8
    psycopg2
    gunicorn
    dj-static==0.0.6

#### run_web.sh

A run script that is convenient for development and production.

We do the **collectstatic** here and not in the Dockerfile so you only have to restart the container to update any static files. I have seen people putting static files in the image build process.

```bash
#!/bin/bash
##
# Version: 1.1
# Author:  jeffreyrobertbean@gmail.com
# Date:    3/21/2015
##

# Simple retry function
function retry {
  local n=1
  local max=5
  local delay=10
  while true; do
    "$@" && break || {
      if [[ ${n} -lt ${max} ]]; then
        ((n++))
        echo "Migration failed. Attempt $n/$max:"
        sleep ${delay};
      else
        echo "The command has failed after $n attempts."
        exit 1
      fi
    }
  done
}
## Validate the django project is going to load properly (does not mean it will run)
python manage.py validate

###
# migrate db, so we have the latest db schema
#
#  Need to retry so if postgres is starting for the first time.
#  Also allows time to see any errors before it attempts to start the server.
##
retry python manage.py migrate --noinput

## Collect all the static files.
python manage.py collectstatic --noinput

##
# start server on the docker ip interface, port 8001
#  Also handles if we are going to start the production gunicorn server or the develop server
##
if [ -z ${DJANGO_DEBUG_MODE} ]; then
    su -m djuser -c "gunicorn django_demo.wsgi -w 1 -b 0.0.0.0:8001 --chdir=/code --enable-stdio-inheritance --error-logfile -"
else
    su -m djuser -c "python manage.py runserver 0.0.0.0:8001"
fi
```

#### Dockerfile

```bash
FROM python:3.3

ENV PYTHONUNBUFFERED 1
RUN mkdir /code
WORKDIR /code

ADD requirements.txt /code/
RUN pip install -r requirements.txt

COPY . /code/
CMD ./run_web.sh
```

#### docker-compose.yml

You dont want to build your own postgres image so lets pick the offical postgres image. Here we pick 9.4 to make sure the demo works.

The following `docker-compose` file here is used for **DEVELOPMENT**.

```yaml
db:
  image: postgres:9.4
web:
  build: .
  ports:
    - "8000:8001"
  links:
    - db
  volumes:
    - .:/code
  environment:
    - DJANGO_DEBUG_MODE=True
```


#### docker-compose.production.yml

The following `docker-compose` file here is used for **PRODUCTION**.

You can see here it is assumed that your built image is now either on the Docker Hub or a private registry. This will run the server in the production settings using gunicorn to be the server for the django app.

```yml
db:
  image: postgres:9.4
web:
  image: <your_django_docker_image>
  ports:
    - "8000:8001"
  links:
    - db
```

#### django_demo/settings.py

The **settings.py** needs to be updated to work nicely with the Docker DB container that will be running. To do this we use the environment to fill in our conection info.

We also need to add our new blog app to the `INSTALLED_APPS` confguration variable.

```python
"""
Django settings for django_demo project.

For more information on this file, see
https://docs.djangoproject.com/en/1.7/topics/settings/

For the full list of settings and their values, see
https://docs.djangoproject.com/en/1.7/ref/settings/
"""

# Build paths inside the project like this: os.path.join(BASE_DIR, ...)
import os
BASE_DIR = os.path.dirname(os.path.dirname(__file__))


# Quick-start development settings - unsuitable for production
# See https://docs.djangoproject.com/en/1.7/howto/deployment/checklist/

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = 'somethingdemosecretkey'

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = os.getenv('DJANGO_DEBUG_MODE', False)

TEMPLATE_DEBUG = True

ALLOWED_HOSTS = []


# Application definition

INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',
)

MIDDLEWARE_CLASSES = (
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.auth.middleware.SessionAuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
)

ROOT_URLCONF = 'django_demo.urls'

WSGI_APPLICATION = 'django_demo.wsgi.application'


# Database
# https://docs.djangoproject.com/en/1.7/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': os.environ.get('DB_ENV_DB', 'postgres'),
        'USER': os.environ.get('DB_ENV_POSTGRES_USER', 'postgres'),
        'PASSWORD': os.environ.get('DB_ENV_POSTGRES_PASSWORD', ''),
        'HOST': os.environ.get('DB_PORT_5432_TCP_ADDR', ''),
        'PORT': os.environ.get('DB_PORT_5432_TCP_PORT', ''),
    },
}
# Internationalization
# https://docs.djangoproject.com/en/1.7/topics/i18n/

LANGUAGE_CODE = 'en-us'

TIME_ZONE = 'UTC'

USE_I18N = True

USE_L10N = True

USE_TZ = True


# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/1.7/howto/static-files/
MESSAGE_STORAGE = 'django.contrib.messages.storage.session.SessionStorage'

MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
MEDIA_URL = '/media/'

STATIC_ROOT = os.path.join(BASE_DIR, 'cstatic')
STATIC_URL = '/static/'

# List of finder classes that know how to find static files in
# various locations.
STATICFILES_FINDERS = (
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
    # 'django.contrib.staticfiles.finders.DefaultStorageFinder',
)
```

#### django_demo/wsgi.py

We want to use a simple static file server that will serve out static files from the Django app.
The **wsgi** settings need to modified to do this. This is not recommended for production applications but for the purpose of this demo it makes things a little easier.

```python
import os
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "django_demo.settings")

from django.core.wsgi import get_wsgi_application
from dj_static import Cling

application = Cling(get_wsgi_application())
```

### Build

    docker-compose build
    docker-compose pull
    docker-compose run web ./manage.py makemigrations
    docker-compose up


### Finish

Now you can navigate to http://localhost:8000 or if you are using boot2docker the IP address of the VM:8000.

Congrats! you are running Django in Docker.

## Summary

This touturial gets a nice Django project up and running in Docker.

Next post will be making a Blog application on top of this infrastructure.
