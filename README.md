# docker-compose-django-nginx

## Summary

This repository serves as an example of how to setup your Docker configuration for a web application 
based on the Django framework. It is based on this [repository](https://github.com/Pawamoy/docker-nginx-postgres-django-example) 
and draws inspiration from several other tutorials and examples: you can check these resources with 
the links at the bottom. There are a few places where things will differ from the original repository,
but that is personal preference for folder structure etc. so keep that in mind. 

We will however keep things as vanilla as possible and instead use plain pip and requirements.txt files 
and remove the need for `gunicorn` so that we can take advantage of code reloading in development.

What is in this repository:

- Overview
- Dockerfile: Simple Django application
- Compose: add a container for NginX
- Compose: add containers for a Postgres database
- Static files: collecting, storing and serving
- Resources

We want three containers: one for NginX, one for Django + Gunicorn (they always go together), and 
one for our database. The NginX container communicate with the Django+Gunicorn one, which itself 
connects to the Postgres container. In the `docker-compose` configuration, it means we will 
declare three containers, or three services. However we need bridges between the containers, 
in order for them to communicate, therefore these will also be in the `docker-compose` config.

Each time you restart your containers or services, the data in the Postgres databases will be lost. 
We need data to be persistent, so we will use a volume to persist the database between restarts.

## Dockerfile: Simple Django application

Create a simple Django project with `django-admin startproject backend`. In order to run `django-admin` 
you need Django to be installed. Create and activate a virtual environment and install Django. Note
the Django version that has been installed, or install the version you prefer. We will be adding that to 
our `requirements.txt` file. In your project directory, run `touch requirements.txt` and add Django, it 
should look something like this:

```text
Django ~=3.2.8
```



Now that you have a working Django project, you can run it by going into the hello directory and 
type `python manage.py runserver`. Go to [http://localhost:8000](http://localhost:8000) to see the 
result.

Now let's create our Dockerfile by running, run `touch Dockerfile`. Paste the following contents
into it:

```dockerfile
FROM python:3

RUN mkdir -p /app/src/
WORKDIR /app/src/

# install requirements
ADD requirements.txt /app/src/
RUN pip install -r requirements.txt

# copy project code
COPY . /app/src/

# expose the port 8000
EXPOSE 8000

# define the default command to run when starting the container
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

With the directory structure looking like the sample below.

```text
.                           # Your current directory
└── backend                 # The Django project
    ├── backend             # The main Django app of your project
    │   ├── __init__.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    ├── Dockerfile
    └── requirements.txt        # The requirements.txt file you created. 

```

You should now be able to change into the backend directory and build the container with 
`docker build . -t backend`, and to start it with `docker run -p 8000:8000 backend`. 
The `-p 8000:8000` option says to bind the port 8000 of the host to the port 8000 of the 
container, allowing you to go to [http://localhost:8000](http://localhost:8000) and see your 
application running as if you were inside the container.

## Run Django Application

    docker-compose up

## Migrations

Make migrations

    docker-compose build
    docker-compose run --rm backend /bin/bash -c "./manage.py migrate"