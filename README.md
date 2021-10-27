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
- Compose: add a Postgres database
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

## Compose: add a container for NginX

Since we will then have two containers, one for Django + Gunicorn, and one for NginX, it's time 
to start our composition with Docker Compose and `docker-compose.yml`. Create your 
`docker-compose.yml` file at the root of the project, like following:

```text
.                           # Your current directory
├── backend                 # The Django project
│   ├── backend             # The main Django app of your project
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── manage.py
│   ├── Dockerfile
│   └── requirements.txt        # The requirements.txt file you created. 
└── docker-compose.yml
```

We are gonna use the version 3 of the configuration syntax. First, we add the Django+Gunicorn 
service:

```yaml
version: '3'

services:
  backend:
    build: ./backend
    container_name: django-backend
    stdin_open: true # Allows access to sdin
    tty: true # Allows code reloading
    volumes:
          - ./backend/:/app/src # maps host directory to internal container directory
    ports:
      - 8000:8000 
```

We simply tell Docker Compose that the backend service must use an image that is built from the 
current directory, therefore looking for our `Dockerfile`. The volumes directive tells to bind 
the current directory of the host to the `/app/src` directory of the container. The changes in 
our current directory will be reflected in real-time in the container directory. 
And reciprocally, changes that occur in the container directory will occur in our current 
directory as well.

Build and run the service with `docker-compose up`. The name of the image will be created as the 
container name `django-backend`. Note I am using backend as my service since there will be a frontend 
service in future. You can name your service accordingly.

Ok, let's add the NginX service now. Remember that we need a bridge to make our services 
communicate.

Update the `docker-compose.yml` as follows:

```yaml
version: '3'

services:

  backend:
    build: ./backend
    container_name: django-backend
    stdin_open: true
    tty: true
    volumes:
      - ./backend/:/app/src 
    networks:
      - nginx_network

  nginx:
    image: nginx:1.13
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
    ports:
      - 8000:80
    depends_on:
      - backend
    networks:
      - nginx_network

networks:
  nginx_network:
    driver: bridge
```

Note that we removed the ports directive from our backend service. We will now communicate with 
NginX instead of the Django server. We still want to access our app at `http://localhost:8000`, 
and we want NginX to listen to the port 80 in the container, so we use `ports: - 8000:80`.

We also bind a local directory to the `/etc/nginx/conf.d` container directory. Let's create it 
and add some contents:

```text
# first we declare our upstream server, which is our Gunicorn application
upstream backend_server {
    # docker will automatically resolve this to the correct address
    # because we use the same name as the service: "backend"
    server backend:8000;
}

# now we declare our main server
server {

    listen 80;
    server_name localhost;

    location / {
        # everything is passed to Gunicorn
        proxy_pass http://backend_server;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
}
```

Run `docker-compose up` and see if you can still see the Django default page at 
[http://localhost:8000](http://localhost:8000).

Note that trying to access your server with [http://0.0.0.0:8000](http://0.0.0.0:8000) will take 
you to the NginX page.

## Compose: add a Postgres database

We now want to use Postgres instead of the starting default SQLite database. We will need to 
update several things: our `requirements.txt`, because we need the `psycopg2` Python package, 
the Postgres driver; our Django project settings; and our `docker-compose.yml` file.

The `requirements.txt` file becomes:

```text
Django ~=3.2.8
psycopg2 ~=2.9.1
```

In the Django project settings, update the DATABASE setting from:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}
```

to

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'django_nginx',
        'USER': 'django_nginx_role',
        'PASSWORD': 'django_nginx_password',
        'HOST': 'postgres',  # <-- IMPORTANT: same name as docker-compose service!
        'PORT': '5432',
    }
}
```

As you can see, we used `django_nginx` everywhere, for the name, user, password and host. 
In fact, we can change these values to whatever suits us. But we must ensure the database 
container will use the same values! To do that, we will copy these values in a configuration 
file destined to be read by our database container.

Create an `env.db` directory in the base:

```dotenv
POSTGRES_USER=django_nginx_role
POSTGRES_PASSWORD=django_nginx_password
POSTGRES_DB=django_nginx
```

It means that, when started, the Postgres container will create a database called django_nginx, 
assigned to the role django_nginx_role with password django_nginx_password. If you change 
these values, remember to also change them in the `DATABASES` setting.

We are now ready to add our service in `docker-compose.yml`. The added service must have the 
same name than what is declared in the `DATABASES` setting:

```yaml
version: '3'

services:

  backend:
    build: ./backend
    container_name: django-backend
    stdin_open: true
    tty: true
    volumes:
      - ./backend/:/app/src # maps host directory to internal container directory
    networks:
      - nginx_network
      - postgres_network  # <-- connect to the bridge
    depends_on:
      - postgres  # <-- wait for db to be "ready" before starting the app

  nginx:
    image: nginx:1.13
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
    ports:
      - 8000:80
    depends_on:
      - backend
    networks:
      - nginx_network

  postgres:  # <-- IMPORTANT: same name as in DATABASES setting, otherwise Django won't find the database!
    image: postgres:12
    env_file:
      - env.db
    networks:  
      - postgres_network  # <-- connect to the bridge
    volumes:
      - postgres_volume:/var/lib/postgresql/data

networks:
  nginx_network:
    driver: bridge
  postgres_network:  # <-- add the bridge
    driver: bridge

volumes:
  postgres_volume:
```

You need to declare your volumes in the root `volumes:` directive if you want them to be 
kept persistently. Then, you can bind a volume to a directory in the container. Here, we 
bind our declared `postgres_volume` to the `postgres` container's `/var/lib/postgresql/data` 
directory.

OK, let's try it. As we are using Django, we need to "migrate" the database first. 
To do this, we will simply use Docker Compose to start our backend service and run the 
migration command inside it:

```bash
docker-compose build  # to make sure everything is up-to-date
docker-compose run --rm backend /bin/bash -c "./manage.py migrate"
```

## Static files: collecting, storing and serving

In order for NginX to serve them, we will update the `config/nginx/conf.d/local.conf` file, 
as well as our `docker-compose.yml` file. Static files will be stored in volumes. We also need 
to set the `STATIC_ROOT` and `MEDIA_ROOT` variables in the Django project settings.

NginX configuration becomes:

```text
# first we declare our upstream server, which is our Gunicorn application
upstream backend_server {
    # docker will automatically resolve this to the correct address
    # because we use the same name as the service: "backend"
    server backend:8000;
}

# now we declare our main server
server {

    listen 80;
    server_name localhost;

    location / {
        # everything is passed to Gunicorn
        proxy_pass http://backend_server;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /app/src/static/;
    }

    location /media/ {
        alias /app/src/media/;
    }
}
```

Add the following to Django project settings remember to `import os`:

```python
# as declared in NginX conf, it must match /app/static/
STATIC_ROOT = os.path.join(BASE_DIR, 'static')

# do the same for media files, it must match /app/media/
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

Add the corresponding volumes in `docker-compose.yml`:

```yaml 
version: '3'

services:

  backend:
    build: ./backend
    container_name: django-backend
    stdin_open: true
    tty: true
    volumes:
      - ./backend/:/app/src # maps host directory to internal container directory
      - static_volume:/app/src/static  # <-- bind the static volume
      - media_volume:/app/src/media  # <-- bind the media volume
    networks:
      - nginx_network
      - postgres_network
    depends_on:
      - postgres

  nginx:
    image: nginx:1.13
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - static_volume:/app/src/static  # <-- bind the static volume
      - media_volume:/app/src/media  # <-- bind the media volume
    ports:
      - 8000:80
    depends_on:
      - backend
    networks:
      - nginx_network

  postgres:
    image: postgres:12
    env_file:
      - env.db
    networks:
      - postgres_network
    volumes:
      - postgres_volume:/var/lib/postgresql/data

networks:
  nginx_network:
    driver: bridge
  postgres_network:
    driver: bridge

volumes:
  postgres_volume:
  static_volume:  # <-- declare the static volume
  media_volume:  # <-- declare the media volume
```

Finally, collect the static files each time you need to update them, by running:

```docker-compose run backend python manage.py collectstatic --no-input```

## Resources

Here are the resources I used to write this tutorial:

[Dockerizing Django with Postgres, Gunicorn, and Nginx](https://testdriven.io/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/)
[Nginx+Flask+Postgres multi-container setup with Docker Compose](https://www.ameyalokare.com/docker/2017/09/20/nginx-flask-postgres-docker-compose.html)
[Django tutorial using Docker, Nginx, Gunicorn and PostgreSQL](https://github.com/andrecp/django-tutorial-docker-nginx-postgres)
[Deploy Django, Gunicorn, NGINX, Postgresql using Docker](https://ruddra.com/docker-django-nginx-postgres/)
[Docker, how to expose a socket over a port for a Django Application](https://stackoverflow.com/questions/32180589/docker-how-to-expose-a-socket-over-a-port-for-a-django-application)


## You have multiple github accounts: now what?

[Debug ssh access](https://support.atlassian.com/bitbucket-cloud/docs/troubleshoot-ssh-issues/)