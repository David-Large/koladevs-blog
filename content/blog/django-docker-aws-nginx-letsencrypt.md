---
title: "Deploy a Django App on AWS Lightsail: Docker, Docker Compose, PostgreSQL, Nginx & Github Actions"
date: 2021-08-31T15:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "awsdocker.png"
---

So you have written your Django Application and you are ready to deploy it? 

Although there are already existing solutions like Heroku, to help you deploy your application easily and quickly, it's always good for a developer to know how to deploy an application on a private server. 

Today, we'll learn how to deploy a Django App on AWS Lightsail. **This can also be applied to others VPS providers. **

## Table of content
- Setup
 - Add PostgreSQL
- Prepare the Django application for deployment
 - Environment variables
 - Testing
 - Docker Configuration
- Github Actions (testing)
- Preparing the server
- Github Actions (Deployment)

## 1 - Setup

For this project, we'll be using an already configured Django application. It's a project made for this article about   [ FullStack React & Django Authentication: Django REST, TypeScript, Axios, Redux & React Router ](https://dev.to/koladev/django-rest-authentication-cmh). 

You can directly clone the repo  [here](https://github.com/koladev32/django-auth-react-tutorial). 

Once it's done, make sure to create a virtual environment and run the following commands. 

```
cd django-auth-react-tutorial 
virtualenv --python=/usr/bin/python3.8 venv
source venv/bin/activate
```

### Add PostgreSQL

Actually, the project is running on sqlite3, which is very good in the local and development environments. Let's switch to PostgreSQL. 

Using this, I pretend that you have PostgreSQL installed on your machine and that the server is running. 
If that's not the case, feel free to check this resource to install the server. 

Once it's done, let's create the database we'll be using for this tutorial. Open your shell enter `psql` and let's start writing some SQL commands.

The CREATE DATABASE command lets us create a new database in PostgreSQL. 

```SQL
CREATE DATABASE coredb;
```

The CREATE USER command lets us create a user for our database along with a password.

```SQL
CREATE USER core WITH PASSWORD '12345678';
```
And finally, let's grand to our new user access to the database created earlier.

```SQL
GRANT ALL PRIVILEGES ON DATABASE coredb TO core;
```

Now let's install `psycopg2` using `pip install psycopg2` is a popular PostgreSQL database adapter for Python.

```
pip install psycopg2
```

Next step, let's set up the project to use PostgreSQL database instead of Sqlite3. 

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

Change the above code snippet to this:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'coredb',
        'USER': 'core',
        'PASSWORD': '12345678',
        'HOST': 'localhost',
        'PORT': 5432,
    }
}

```
What have we done? 

- `ENGINE`: We changed the database engine to use `postgresql_psycopg2` instead of `sqlite3`. 
- `NAME`: is the name of the database we created for our project. 
- `USER`: is the database user we've created during the database creation. 
- `PASSWORD`: is the password to the database we created. 

Now, make are that the Django application is connected to the PostgreSQL database. For this, we'll be running the `migrate` command which is responsible for executing the SQL commands specified in the migrations files. 

```
python manage.py migrate
```

You'll have a similar output: 

```
  Applying auth.0001_initial... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  ...
  Applying core_user.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying sessions.0001_initial... OK
```
Now, let's run the server to check if the application is working well.

```
python manage.py runserver
```

You will see something like this when you hit `http://127.0.0.1:8000` in your browser. 

![Django Server Well Running](https://cdn.hashnode.com/res/hashnode/image/upload/v1629734721362/2Wy0WpBJA.png)

Bravo for us! We've successfully configured our Django app to use PostgresSQL.

Now, let's prepare the application for deployment.

## 2 - Prepare application for deployment

Here, we'll configure the application to use env variables but also configure  [Docker](https://www.docker.com/)  as well. 

### Env variables

It's important to keep sensitive bits of code like API keys, passwords, and secret keys away from prying eyes. 
The best way to do it? Use environment variables. Here's how to do it in our application. 

First of all, let's install a python package named `django-environ`. 

```
pip install django-environ
```

Then, import it in the settings.py file, and let's initialize environment variables.

```python
# CoreRoot/settings.py

import environ

# Initialise environment variables

env = environ.Env()environ.Env.read_env()
```

Next step, create two files : 
- a `.env` file which will contain all environment variables that `django-environ` will read 
- and a `env.example` file which will contain the same content as `.env`. 

Actually, the `.env` file is ignored by git. The `env.example` file here represents a skeleton we can use to create our `.env` file in another machine. 

It'll be visible, so make sure to not include sensitive information.

```
# ./.env
SECRET_KEY=django-insecure-97s)x3c8w8h_qv3t3s7%)#k@dpk2edr0ed_(rq9y(rbb&_!ai%
DEBUG=0
DJANGO_ALLOWED_HOSTS="localhost 127.0.0.1 [::1]"
DB_ENGINE=django.db.backends.postgresql_psycopg2
DB_NAME=coredb
DB_USER=core
DB_PASSWORD=12345678
DB_HOST=localhost
DB_PORT=5432
CORS_ALLOWED_ORIGINS="http://localhost:3000 http://127.0.0.1:3000"
```
Now, let's copy the content paste it in `.env.example`, but make sure to delete the values.

```
./env.example
SECRET_KEY=
DEBUG=
DJANGO_ALLOWED_HOSTS=
DB_ENGINE=
DB_NAME=
DB_USER=
DB_PASSWORD=
DB_HOST=
DB_PORT=
CORS_ALLOWED_ORIGINS=
```
And now, let's go back to the settings file and add the env variables configurations as well.

```python
# ./CoreRoot/settings.py

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env('SECRET_KEY', default='qkl+xdr8aimpf-&x(mi7)dwt^-q77aji#j*d#02-5usa32r9!y')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = int(env("DEBUG", default=1))


ALLOWED_HOSTS = env("DJANGO_ALLOWED_HOSTS").split(" ")

DATABASES = {
    'default': {
        'ENGINE': env('DB_ENGINE', default='django.db.backends.postgresql_psycopg2'),
        'NAME': env('DB_NAME', default='coredb'),
        'USER': env('DB_USER', default='core'),
        'PASSWORD': env('DB_PASSWORD', default='12345678'),
        'HOST': env('DB_HOST', default='localhost'),
        'PORT': env('DB_PORT', default='5432'),
    }
}

CORS_ALLOWED_ORIGINS = env("CORS_ALLOWED_ORIGINS").split(" ")

```

### Testing

Testing in an application is the first assurance of maintainability and reliability of our Django server. 
We'll be implementing testing to make sure everything is green before pushing for deployment.

Let's write tests for our login and refresh endpoints. 
We'll also add one test to the `UserViewSet`. 
First of all, create a file named `test_runner.py` in `CoreRoot` directory. 

The goal here is to rewrite the `DiscoverRunner`, to load our custom fixtures in the test database. 

```python
# ./CoreRoot/test_runner.py
from importlib import import_module

from django.conf import settings
from django.db import connections
from django.test.runner import DiscoverRunner


class CoreTestRunner(DiscoverRunner):
    def setup_test_environment(self, **kwargs):
        """We set the TESTING setting to True. By default, it's on False."""
        super().setup_test_environment(**kwargs)
        settings.TESTING = True

    def setup_databases(self, **kwargs):
        """We set the database"""
        r = super().setup_databases(**kwargs)
        self.load_fixtures()
        return r

    @classmethod
    def load_fixtures(cls):
        try:
            module = import_module(f"core.fixtures")
            getattr(module, "run_fixtures")()
        except ImportError:
            return
``` 
Once it's done, we can add the TESTING configurations in the `settings.py` file. 

```python
# CoreRoot/settings.py
...
TESTING = False
TEST_RUNNER = "CoreRoot.test_runner.CoreTestRunner"
```
Now, we can start writing our tests. 

Let's start with the authentication tests. First of all, let's add the URLs and the data we'll be using.

```python
# core/auth/tests.py

from django.urls import reverse
from rest_framework.test import APITestCase
from rest_framework import status


class AuthenticationTest(APITestCase):
    base_url_login = reverse("core:auth-login-list")
    base_url_refresh = reverse("core:auth-refresh-list")

    data_register = {"username": "test", "password": "pass", "email": "test@appseed.us"}

    data_login = {
        "email": "testuser@yopmail.com",
        "password": "12345678",
    }
```
Great! We can add a test for login now.

```python
# core/auth/tests.py
...
       def test_login(self):
        response = self.client.post(f"{self.base_url_login}", data=self.data_login)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

To run the tests, open the terminal and enter the following command. 

```
python manage.py test
```
You should see a similar output: 

```
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.
----------------------------------------------------------------------
Ran 1 test in 0.287s

OK
Destroying test database for alias 'default'...
```
Let's add a test for the refresh endpoint. 

```python
# core/auth/tests.py
...

    def test_refresh(self):
        # Login

        response = self.client.post(f"{self.base_url_login}", data=self.data_login)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

        response_data = response.json()

        access_token = response_data.get('access')
        refresh_token = response_data.get('refresh')

        # Refreshing the token

        response = self.client.post(f"{self.base_url_refresh}", data={
            "refresh": refresh_token
        })

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        response_data = response.json()
        self.assertNotEqual(access_token, response_data.get('access'))
```

What we are doing here is pretty straightforward : 
- Login to retrieve the access and refresh tokens 
- Making a request with the refresh token to gain a new access token
- Comparing the old access token and the new obtained access token to make sure they are not equal.

Now let's move to the  [Docker](https://www.docker.com/)  configuration. 

### Dockerizing our app

 [Docker](https://www.docker.com/)  is an open platform for developing, shipping, and running applications inside containers. 
Why use Docker? 
It helps you separate your applications from your infrastructure and helps in delivering code faster. 

If it's your first time working with Docker, I highly recommend you go through a quick tutorial and read some documentation about it. 

Here are some great resources that helped me: 
-  [Docker Tutorial](https://www.youtube.com/watch?v=eN_O4zd4D9o&list=PLPoSdR46FgI5wOJuzcPQCNqS37t39zKkg) 
-  [Docker curriculum](https://docker-curriculum.com/) 

#### Dockerfile

The `Dockerfile` represents a text document containing all the commands that could call on the command line to create an image.

Add a Dockerfile to the project root:

```
# pull official base image
FROM python:3.9-alpine

# set work directory
WORKDIR /app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install python dependencies
COPY requirements.txt /app/requirements.txt
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt


# copy project
COPY . .
```
Here, we started with an **Alpine-based Docker Image for Python**. It's a lightweight Linux distribution designed for security and resource efficiency. 
After that, we set a working directory followed by two environment variables: 
 
1 - `PYTHONDONTWRITEBYTECODE` to prevent Python from writing `.pyc` files to disc
2 - `PYTHONUNBUFFERED` to prevent Python from buffering `stdout` and `stderr`

After that, we perform operations like: 
- Setting up environment variables
- Installing Postgres server
- Copying there `requirements.txt` file to our app path, upgrading pip, and installing the python package to run our application
- And last copying the entire project

Also, let's add a `.dockerignore` file. 

```
env
venv
.dockerignore
Dockerfile
```

#### Docker Compose

 [Docker Compose](https://docs.docker.com/compose/)  is a great tool (<3). You can use it to define and run multi-container Docker applications.
 
What do we need? Well, just a YAML file containing all the configuration of our application's services. 
Then, with the `docker-compose` command, we can create and start all those services.

Here, the `docker-compose.dev.yml` file will contain three services that make our app: nginx, web, and db. 

This file will be used for development.

As you guessed : 

```yaml
version: '3.7'
services:
  nginx:
    container_name: core_web
    restart: on-failure
    image: nginx:stable
    volumes:
      - ./nginx/nginx.dev.conf:/etc/nginx/conf.d/default.conf
      - static_volume:/app/static
    ports:
      - "80:80"
    depends_on:
      - web
  web:
    container_name: core_app
    build: .
    restart: always
    env_file: .env
    ports:
      - "5000:5000"
    command: >
      sh -c " python manage.py migrate &&
          gunicorn CoreRoot.wsgi:application --bind 0.0.0.0:5000"
    volumes:
     - .:/app
     - static_volume:/app/static
    depends_on:
     - db
  db:
    container_name: core_db
    image: postgres:12.0-alpine
    env_file: .env
    volumes:
      - postgres_data:/var/lib/postgresql/data/

volumes:
  static_volume:
  postgres_data:
```

- `nginx`:  [NGINX](https://www.nginx.com/)  is an open-source software for web serving, reverse proxying, caching, load balancing, media streaming, and more.
- `web`: We'll run and serve the endpoint of the Django application through Gunicorn. 
- `db`: As you guessed, this service is related to our PostgreSQL database.


And the next step, let's create the NGINX configuration file to proxy requests to our backend application. 
In the root directory, create a `nginx` directory and create a `nginx.dev.conf` file.

```
upstream webapp {
    server core_app:5000;
}
server {

    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://webapp;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
     alias /app/static/;
    }
}
```
Let's add `gunicorn` and some configurations before building our image.
```
pip install gunicorn
```

And add it as a requirement as well in the `requirements.txt`. 
Here's what my `requirements.txt` file looks like : 

```txt
Django==3.2.4
djangorestframework==3.12.4
djangorestframework-simplejwt==4.7.1
django-cors-headers==3.7.0
psycopg2==2.9.1
django-environ==0.4.5
gunicorn==20.1.0
```

And the last thing, let's add `STATIC_ROOT` in the `settings.py` file.

#### Docker Build

The setup is completed. Let's build our containers and test if everything works locally.

```
docker-compose -f docker-compose.dev.yml up -d --build 
```

Once it's done, hit `localhost/api/auth/login/` to see if your application is working. 
You should get a similar page. 

![Page GET Not allowed](https://cdn.hashnode.com/res/hashnode/image/upload/v1630022210893/4hCPqR6A4.png)

Great! Our Django application is successfully running inside a container.

Let's move to the Github Actions to run tests every time there is a push on the `main` branch.

## Github Actions (Testing)

 [GitHub actions](https://github.com/features/actions)  are one of the greatest features of Github. it helps you build, test or deploy your application and more. 

Here, we'll create a YAML file named `django.yml` to run some Django tests. 

In the root project, create a directory named `.github`. Inside that directory, create another directory named `workflows` and create the `django.yml` file. 

```yaml
name: Django CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:

    runs-on: ubuntu-latest
    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.9]
    services:
      postgres:
        image: postgres:12
        env:
          POSTGRES_USER: core
          POSTGRES_PASSWORD: 12345678
          POSTGRES_DB: coredb
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
    - uses: actions/checkout@v2
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}
    - name: psycopg2 prerequisites
      run: sudo apt-get install python-dev libpq-dev
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run Tests
      run: |
        python manage.py test
```

Basically, what we are doing here is setting rules for the  [GitHub action workflow](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions), installing dependencies, and running the tests. 

- Make sure that this workflow is triggered only when there is a push or pull_request on the main branch
- Choose `ubuntu-latest` as the OS and precise the Python version on which this workflow will run. 
- Next, we create a Postgres service, the database will be used to run our tests. 
- After that as we install the python dependencies and just run the tests. 

If you push the code in your repository, you'll see something similar when you go to your repository page. 

![Django Actions](https://cdn.hashnode.com/res/hashnode/image/upload/v1630024016358/wxAUXlykO.png)

After a moment, the yellow colors will turn to green, meaning that the checks have successfully completed.

## Setting up the AWS server

I'll be using a  [Lightsail server](https://aws.amazon.com/lightsail/) here. Note that these configurations can work with any VPS provider. 

If you want to set up a Lightsail instance, refer to the AWS  [documentation](https://aws.amazon.com/lightsail/).

Personally, I am my VPS is running on Ubuntu 20.04.3 LTS.

Also, you'll need  [Docker](https://docs.docker.com/engine/install/ubuntu/)  and  [docker-compose](https://docs.docker.com/compose/install/) installed on the machine.
 
After that, if you want to link your server to a domain name, make sure to add it to your DNS configuration panel. 

![Domain name configurations](https://cdn.hashnode.com/res/hashnode/image/upload/v1630162654342/mGaOf9REK.png)

Once you are done, we can start working on the deployment process. 

### Docker build script

To automate things here, we'll write a bash script to pull changes from the repo and also build the docker image and run the containers. 

We'll also be checking if there are any coming changes before pulling and re-building the containers again.

```
#!/usr/bin/env bash

TARGET='main'

cd ~/app || exit

ACTION='\033[1;90m'
NOCOLOR='\033[0m'

# Checking if we are on the main branch

echo -e ${ACTION}Checking Git repo
BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [ "$BRANCH" != ${TARGET} ]
then
  exit 0
fi

# Checking if the repository is up to date.

git fetch
HEADHASH=$(git rev-parse HEAD)
UPSTREAMHASH=$(git rev-parse ${TARGET}@{upstream})

if [ "$HEADHASH" == "$UPSTREAMHASH" ]
then
  echo -e "${FINISHED}"Current branch is up to date with origin/${TARGET}."${NOCOLOR}"
  exit 0
fi

# If that's not the case, we pull the latest changes and we build a new image

git pull origin main;

# Docker

docker-compose up -d --build

exit 0;
```

Good! Login on your server using SSH. We'll be creating some new directories: one for the repo and another one for our scripts. 

```
mkdir app .scripts
cd .scripts
vim docker-deploy.sh
```
And just paste the content of the precedent script and modify it if necessary.

```
cd ~/app
git clone <your_repository> .
```

Don't forget to add the `.`. Using this, it will simply clone the content of the repository in the current directory.

Great! Now we need to write the default `docker-compose.yml` file which will be run on this server.

We'll be adding an SSL certificate, by the way, so we need to create another `nginx.conf` file.

Here's the `docker-compose.yml` file.

```
version: '3.7'
services:
  nginx:
    container_name: core_web
    restart: on-failure
    image: jonasal/nginx-certbot:latest
    env_file:
      - .env.nginx
    volumes:
      - nginx_secrets:/etc/letsencrypt
      - ./nginx/user_conf.d:/etc/nginx/user_conf.d
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - web

  web:
    container_name: core_app
    build: .
    restart: always
    env_file: .env
    ports:
      - "5000:5000"
    command: >
      sh -c " python manage.py migrate &&
          gunicorn CoreRoot.wsgi:application --bind 0.0.0.0:5000"
    volumes:
     - .:/app
     - static_volume:/app/static
    depends_on:
     - db
  db:
    container_name: core_db
    image: postgres:12.0-alpine
    env_file: .env
    volumes:
      - postgres_data:/var/lib/postgresql/data/

volumes:
  static_volume:
  postgres_data:
  nginx_secrets:
```

If you noticed, we've changed the `nginx` service. Now, we are using the `docker-nginx-certbot` image. It'll automatically create and renew SSL certificates using the  [Let's Encrypt](https://letsencrypt.org/) free CA (Certificate authority) and its client `certbot`. 

Create a new directory `user_conf.d` inside the `nginx` directory and create a new file `nginx.conf`.

```
upstream webapp {
    server core_app:5000;
}

server {

    listen 443 default_server reuseport;
    listen [::]:443 ssl default_server reuseport;
    server_name dockerawsdjango.koladev.xyz;
    server_tokens off;
    client_max_body_size 20M;


    ssl_certificate /etc/letsencrypt/live/dockerawsdjango.koladev.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dockerawsdjango.koladev.xyz/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/dockerawsdjango.koladev.xyz/chain.pem;
    ssl_dhparam /etc/letsencrypt/dhparams/dhparam.pem;

    location / {
        proxy_pass http://webapp;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
     alias /app/static/;
    }
}
```
**Make sure to replace `dockerawsdjango.koladev.xyz` with your own domain name...**

And no troubles! I'll explain what I've done. 

```
server {
    listen 443 default_server reuseport;
    listen [::]:443 ssl default_server reuseport;
    server_name dockerawsdjango.koladev.xyz;
    server_tokens off;
    client_max_body_size 20M;
```

So as usual, we are listening on port `443` for **HTTPS**. 
We've added a `server_name` which is the domain name. We set the `server_tokens` to off to not show the server version on error pages. 
And the last thing, we set the request size to a **max of 20MB**. It means that requests larger than 20MB will result in errors with **HTTP 413** (Request Entity Too Large).

Now, let's write the job for deployment in the Github Action.

```
...
  deploy:
    name: Deploying
    needs: [test]
    runs-on: ubuntu-latest
    steps:
    - name: Deploying Application
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SSH_AWS_SERVER_IP }}
        username: ${{ secrets.SSH_SERVER_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        passphrase: ${{ secrets.SSH_PASSPHRASE }}
        script: |
          cd ~/.scripts
          ./docker-deploy.sh
```

Notice the usage of Github Secrets here. It allows the storage of sensitive information in your repository. Check this  [documentation](https://docs.github.com/en/actions/reference/encrypted-secrets)  for more information.

We also using here a GitHub action that requires the name of the host, the username, the key, and the passphrase. You can also use this action with a password but it'll require some configurations. 
Feel free to check the  [documentation](https://github.com/appleboy/ssh-action#setting-up-a-ssh-key)  of this action for more detail.

Also, notice the `needs: [build]` line. It helps us make sure that the precedent job is successful before deploying the new version of the app.

Once it's done, log via ssh in your server and create a .env file. 

```
cd app/
vim .env # or nano or whatever
```

And finally, create a `.env.nginx` file. This will contain the required configurations to create an SSL certificate.

```
# Required
CERTBOT_EMAIL=

# Optional (Defaults)
STAGING=1
DHPARAM_SIZE=2048
RSA_KEY_SIZE=2048
ELLIPTIC_CURVE=secp256r1
USE_ECDSA=0
RENEWAL_INTERVAL=8d
```

Add your email address. Notice here that `STAGING` is set to 1. We will test the configuration first with **Let’s encrypt** staging environment! It is important to not set staging=0 before you are 100% sure that your configuration is correct. 

This is because there is a limited number of retries to issue the certificate and you don’t want to wait till they are reset (once a week). 

Declare the environment variables your project will need. 

And we're nearly done. :)

Make a push to the repository and just wait for the actions to pass successfully.

![Successful Deployment](https://cdn.hashnode.com/res/hashnode/image/upload/v1630170291071/0t26MZ9nt.png)

And voilà. I can check https://dockerawsdjango.koladev.xyz/ and here's the result.


![HTTPS expired](https://cdn.hashnode.com/res/hashnode/image/upload/v1630280545520/4f2ttZjfw.png)

It looks like our configuration is clean! We can issue a production-ready certificate now. 
On your server, stop the containers.

```
docker-compose down
```

edit your `.env.nginx` file and set `STAGING=0`.

Then, start the containers again.

```
sudo docker-compose up -d --build
```

Let's refresh the page. 

![HTTPS Secure](https://cdn.hashnode.com/res/hashnode/image/upload/v1630281017853/jEnMx60C-.png)

And it's working like a charm! :)


## Conclusion

In this article,  we've learned how to use Github Actions to deploy a dockerized Django application on an AWS Lightsail server. Note that you can use these steps on any VPS. 

And as every article can be made better so your suggestion or questions are welcome in the comment section. 😉

Check the code of this tutorial  [here](https://github.com/koladev32/django-aws-docker-github-actions). 

