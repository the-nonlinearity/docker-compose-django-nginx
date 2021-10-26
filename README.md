# docker-compose-django-nginx

## Run Django Application

    docker-compose up

## Migrations

Make migrations

    docker-compose build
    docker-compose run --rm backend /bin/bash -c "./manage.py migrate"