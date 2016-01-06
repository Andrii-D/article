#Virtualization of development process with Docker and Vagrant
##Table of Contents
  * [Introduction](#introduction)
  * [Before we start](#before-we-start)
  * [Docker](#docker)
  * [Development process](#development-process)
  * [Vagrant](#vagrant)
  * [Additional information](#additional-information)
  * [The End](#the-end)


##Inroduction
Hello nerds, my name is Andrii Dvoiak and I'm a full stack web developer at Ukrainian startup called [Preply.com](http://preply.com).  
Preply - is a marketplace for tutoring. The platform where you can easily find a personal professional tutor  
based on your location and needs. We have more then 15 thousands tutors of 40 different subjects.  

We have been growing pretty fast for the last year in terms of customers and also our team,  
so we decided to push our development process to the next level.  
First of all we decided to organize and standardize our development environment so we can easily onboard new dev team members.  

We spent a lot of time researching the best practices and talking to other teams to understand what is  
the most efficient way to organize sharable development environments especially when your product decoupled to many microservices.  
*DESCLIMER:  Yes, we are the big fans of microservices and Docker technology.*

So we ended up with two main technologies: __Vagrant__ and __Docker__.  
You can read about all aspects and differences from creators of these services [here on StackOverflow](http://stackoverflow.com/questions/16647069/should-i-use-vagrant-or-docker-for-creating-an-isolated-environment).  

In this guide we will show you how to Dockerize your application so you can easily share  
and deploy it at any machine which supports Docker.  
Also we will show you how to run Docker almost everywhere using Vagrant.  

*Note: We think that Vagrant is an overhead in this stack  
and you only need to use it on Windows or other platforms which doesn't support Docker natively.*


This guide will help you to Dockerize your Django App  
for production deployment and create an isolated development environment based on [Vagrant](https://docs.vagrantup.com/v2/getting-started/).  
You will be able to quickly deploy your app at any production server which supports Docker technology.  

Also, we will teach you how to create a development environment based on virtual machine   
which you can easily pass to your co-workers with no worrying about their local OS's.

By the end of the day you will have your docker container to deploy on production server and development VM in the same way.

##Before we start
In this guide we'll be dealing with a simple Django app with [PostgerSQL](http://www.postgresql.org) Database  
and [Redis](http://redis.io) as a broker for [Celery](http://docs.celeryproject.org/en/latest/django/first-steps-with-django.html) Tasks.  
Also we are using [Supervisor](https://github.com/rfk/django-supervisor) to run our [Gunicorn](https://docs.djangoproject.com/en/1.8/howto/deployment/wsgi/gunicorn/) server.  

We are going to use [Docker Compose](http://docs.docker.com/compose/) technology to orchestrate our multi-container application.   
*Note that Compose 1.5.1 requires Docker 1.8.0 or later.*  
This will help us to run Django app, PostgreSQL and Redis Server and Celery Worker in separated containers and link them between each other.

To make it all happen, we only need to create a few files in root directory of your Django project (next to __manage.py__ file):

1. `Dockerfile` - (to build a final image and push it to DockerHub)
2. `redeploy.sh` - (to make redeploy in both DEV and PROD environments)
3. `docker-compose.yml` - (to orchestrate two containers)
4. `Vagrantfile` - (to provision a virtual machine for development environment)


We are using environment variable RUN_ENV to specify current environment.  
You type `export RUN_ENV=PROD` on production server and `export RUN_ENV=DEV` in Vagrant VM.


##Docker
###Introduction
As you may know, there are two main things in Docker to know: images and containers.    
We are going to create an image based on Ubuntu with installed Python, PIP and other Tools needed to run your Django app.  
Also, this basic image will contain all you requirements pre-installed.  
We will push this image to public repository in DockerHub.  
Note that this image __doesn't contain any of your project files__!  

We all agreed to keep our image on DockerHub always up-to-date.
From that image we are going to run a container exposing specific ports   
and mounting your local directory with project to some folder inside the container.  
This means that all your project files will be accessible within the container. No need to copy files!  

*Note: this is very useful for development process, because you can change your files and they will instantly change in running Docker container,  
but it unacceptable on production environment.*
 
Once we run a container, we will start supervisor with your gunicorn server.  

Once you wanted to do redeploy, we will pull new data from github,  
stop and remove existing container and run brand-new container! Magic!

Let's start!!!

###Docker and Docker-Compose installation
Before we start we need to install Docker and Docker-Compose on our local computer or server.  
Here is a code to do it on Ubuntu 14.04:  
    
    sudo -i
    echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
    curl -sSL https://get.docker.com/ | sh
    curl -L https://github.com/docker/compose/releases/download/1.5.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose
    usermod -aG docker ubuntu
    sudo reboot

*where __ubuntu__ is your current user*  

If your current development environment is not Ubuntu 14.04 you better to use Vagrant   
to create this environment. Start from [Vagrant section](#vagrant) of this tutorial.

###Docker image
Firstly, to prepare project for deployment with docker we need to build an image  
with only Python, PIP, and some pip requirements needed to run Django.

Lets create new dockerfile called `Dockerfile` in root directory of a project.  
It will looks like this:

    FROM ubuntu:14.04
    MAINTAINER Andrii Dvoiak

    RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
    RUN apt-get update
    RUN apt-get install -y python-pip python-dev python-lxml libxml2-dev libxslt1-dev libxslt-dev libpq-dev zlib1g-dev && apt-get build-dep -y python-lxml && apt-get clean
    # Specify your own RUN commands here (e.g. RUN apt-get install -y nano)

    ADD requirements.txt requirements.txt
    RUN pip install -r requirements.txt

    WORKDIR /project

    EXPOSE 80

*You can specify your own RUN commands, for example, to install other needed tools*

Then you need to build an image from this, using command

    docker build -t username/image .
*(where 'username' - your Dockerhub username and 'image' - name of your new image for this project)*

When you successfully built a basic image, you better to push it to your DockerHub cloud:

    docker push username/image

and don't worry, this image doesn't have any information of your project (besides *requiremets.txt* file).  
*Note: if you use private DockerHub repository be sure to execute `docker login` before pushing and pulling images.*


###Orchestrating containers
So at that moment we can run our container with django app,  
but also we need to run a few more containers with Redis server, PostgreSQL database and Celery Worker.  

To make this process simple we are going to use [Docker Compose](http://docs.docker.com/compose/) technology  
which allows us to create a simple YML file with instructions of what containers to run and how to link them between each other.

Lets create this magic file and call it by default `docker-compose.yml`:

    django:
      image: username/image:latest
      command: python manage.py supervisor
      environment:
        RUN_ENV: "$RUN_ENV"
      ports:
       - "80:8001"
      volumes:
       - .:/project
      links:
       - redis
       - postgres
       
    celery_worker:
      image: username/image:latest
      command: python manage.py celery worker -l info
      links:
       - postgres
       - redis
       
    postgres:
      image: postgres:9.1
      volumes:
        - local_postgres:/var/lib/postgresql/data
      ports:
       - "5432:5432"
      environment:
        POSTGRES_PASSWORD: "$POSTGRES_PASSWORD"
        POSTGRES_USER: "$POSTGRES_USER"  
         
    redis:
      image: redis:latest
      command: redis-server --appendonly yes

As you can see, we are going to run four projects called `django`, `celery_worker`, `postgres` and `redis`.  
These names are important for us.

Firstly, it will pull Redis image from dockerhub and run container from it.  
Secondly it will pull Postgres image and run container with mounted data from volume `local_postgres`.  
(See the [Additional information](#additional-information) section about how to create a local database).  
Then, it will start container with our Django app, forward 8001 port from inside to 80 outside,  
mount your current directory straight to `/project` folder inside container,  
link it with Redis and Postgres containers and run a supervisor.
And the last but not least is our container with celery worker, which is also linked with Postgres and Redis.  

*You can forward any number of ports if needed - just add new lines to __links__ section.  
Also you can link any number of containers, for example container with your database.*

*Use tag `:latest` for automatic checking for updated image on DockerHub*

__Important!__  
If you named project with Redis Server in YML file `redis` - you need to specify `redis` instead of `localhost`   
in your __settings.py__ to allow your Django app to connect to redis:

    REDIS_HOST = "redis"
    BROKER_URL = "redis"

The same with Postgres database: use `postgres` instead of `localhost`.

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': 'database_name',
            'USER': os.getenv('DATABASE_USER', ''),
            'PASSWORD': os.getenv('DATABASE_PASSWORD', ''),
            'HOST': 'postgres',
            'PORT': '5432',
        }
    }


###Redeploy script

Lets create re-deploy script to do redeploy in one click (lets call it `redeploy.sh`):

    #!/bin/sh
    if [ -z "$RUN_ENV" ]; then
        echo 'Please set up RUN_ENV variable'
        exit 1
    fi

    if [ "$RUN_ENV" = "PROD" ]; then
        git pull
    fi

    docker-compose stop
    docker-compose rm -f
    docker-compose up -d

Lets check what it does:

1. It checks if variable RUN_ENV is set and exit if it doesn't
2. If RUN_ENV is set to PROD, it will do 'git pull' command to get a new version of your project
3. It will stop all projects, specified in `docker-compose.yml` file
4. It will remove all existing containers
5. It will start new containers


So it looks pretty simple: to do re-deploy you just need to run `./redeploy.sh` !
*Do not forget to grant rights for execution(chmod +x redeploy.sh) for this script all other .sh scripts listed in manual*

To make a quick redeploy use this command:

    docker-compose up --no-deps -d django
    
But we don't know how it will start working inside the container. Lets take a look in the next section!

### Deploying inside the container

We just need to run __supervisor__ to actually start our service inside the container:

    python manage.py supervisor

If you don't use supervisor you can just start your server instead of supervisor.

If you use supervisor lets take a look on 'supervisord.conf' file:

    [supervisord]
    environment=C_FORCE_ROOT="1"

    [program:__defaults__]
    redirect_stderr=true
    startsecs=10
    autorestart=true

    [program:gunicorn_server]
    command=gunicorn -w 4 -b 0.0.0.0:8001 YourApp.wsgi:application
    directory={{ PROJECT_DIR }}
    stdout_logfile={{ PROJECT_DIR }}/gunicorn.log


So the supervisor will start your Gunicorn server with 4 workers and bind on 8001 port.  

##Development process

1. To access your local server you just go to http://127.0.0.1:8000
2. To redeploy local changes do `sh redeploy.sh`
3. To see all logs of your project run:

        docker-compose logs
4. To quickly redeploy changes use:

        docker-compose restart django

5. To connect to Django shell just do (if you have running container called CONTAINER):

        docker exec -it CONTAINER python manage.py shell
    and run this command to create initial superuser:

        from django.contrib.auth.models import User; User.objects.create_superuser('admin', 'admin@example.com', 'admin')

6. To make migrations:

        docker exec -it CONTAINER python manage.py schemamigration blabla --auto  

7. Or you can connect to bash inside the container:

        docker exec -it CONTAINER /bin/bash
        
8. You can dump your local database to .json file (you can specify a table to dump):

        docker exec -it CONTAINER python manage.py dumpdata > testdb.json

9. Or you can load data to your database from file:

        docker exec -it CONTAINER python manage.py loaddata testdb.json  

10. Use this command to monitor status of your running containers:  

        docker stats $(docker ps -q)

11. Use this command to delete all stopped containers:

        docker rm -v `docker ps -a -q -f status=exited`

*Where __CONTAINER__ suppose to be `project_django_1`*

You can play with your containers as you want. Here is a useful [Docker cheat sheet](https://github.com/wsargent/docker-cheat-sheet).  


##Vagrant
###Intro
The only reason we use Vagrant is to be able to create isolated development environment based on virtual machine  
with all needed services which is going to reproduce a real production environment.  
In other words, just to be able to run Docker.  
This environment can be easily and quickly installed to any machine of your co-workers.  

To do this you only need to install Vagrant and VirtualBox (or another provider) to local machine.

###Vagrantfile
To set up a virtual machine we need to create a file called `Vagrantfile` in root dir of our project.

This is a good example of Vagrantfile:

    # -*- mode: ruby -*-
    # vi: set ft=ruby :

    Vagrant.configure(2) do |config|

      config.vm.box = "ubuntu/trusty64"

      # We are going to run docker container on port 80 inside vagrant and expose it to port 8000 outside vagrant
      config.vm.network :forwarded_port, guest: 80, host: 8000

      #You can forward any number of ports:
      #config.vm.network :forwarded_port, guest: 5555, host: 5555

      # All files from current directory will be available in /project directory inside vagrant
      config.vm.synced_folder ".", "/project"

      # We are going to give VM 1/4 system memory & access to all cpu cores on the host
      config.vm.provider "virtualbox" do |vb|
        host = RbConfig::CONFIG['host_os']
        if host =~ /darwin/
          cpus = `sysctl -n hw.ncpu`.to_i
          mem = `sysctl -n hw.memsize`.to_i / 1024 / 1024 / 4
        elsif host =~ /linux/
          cpus = `nproc`.to_i
          mem = `grep 'MemTotal' /proc/meminfo | sed -e 's/MemTotal://' -e 's/ kB//'`.to_i / 1024 / 4
        else
          cpus = 2
          mem = 1024
        end
        vb.customize ["modifyvm", :id, "--memory", mem]
        vb.customize ["modifyvm", :id, "--cpus", cpus]
      end

      config.vm.provision "shell", inline: <<-SHELL
        # Install docker and docker compose
        sudo -i
        echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
        curl -sSL https://get.docker.com/ | sh
        usermod -aG docker vagrant
        curl -L https://github.com/docker/compose/releases/download/1.5.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
        chmod +x /usr/local/bin/docker-compose

      SHELL
    end

We are going to use this file to start our virtual machine by typing this command:

    vagrant up --provision

*If you see the warning about new version of your initial box just do:*

    vagrant box update

Lets look closer what it does:

1. It will pull an image of ubuntu/trusty64 operation system from vagrant repository
2. It will expose 80 from inside the machine to 8000 outside
3. It will mount your current directory to '/project' directory inside the machine
4. It will give 1/4 system memory & access to all cpu cores from the virtual machine
5. It will install Docker and Docker-Compose inside the virtual machine

This is it. Now you just need go inside this VM by typing:

    vagrant ssh
Lets check if everything is properly installed:

    docker --version
    docker-compose --version
*If not, try to install it manually! [Docker](https://docs.docker.com/v1.8/installation/ubuntulinux/) and [Docker-Compose](https://docs.docker.com/compose/install/)*

Lets go to see our project dir inside the VM and set environment variable RUN_ENV to DEV:

    cd /project
    export RUN_ENV=DEV

Now you can do local redeploy as simple as:

    sh redeploy.sh

Enjoy your local server on [http://127.0.0.1:8000](http://127.0.0.1:8000)

###Closing down virtual machine
You can use three different commands to stop working with your VM:

    vagrant suspend     # - to freeze your VM with saved RAM
    vagrant halt        # - to stop your VM but save all files
    vagrant destroy     # - to fully delete your virtual machine!


##Additional information
###Creating local database in container
####Preperation
We assume that you use Docker 1.9 and Docker-Compose 1.5.1 or later. 

It was a mess with volumes in Docker before ver. 1.9. And now Docker introduced __Volume__!  
You can create __volume__ independently from containers and images.  
It allows you to have persistent volume accessible from any number of mounted containers and keep data safe even when containers are dead.  

It's very simple. To list all your local volumes do:

    docker volume ls
    
To inspect the volume:

    docker volume inspect <volume_name>
   
To create new volume:

    docker volume create --name=<volume_name>
    
And to delete volume just do:

    docker volume rm <volume_name>
    
*TIP: Now you can remove old volumes created by some containers in past.*

####Creating Local database

We are going to run Docker container with PostgreSQL and mount it to already created Docker Volume.

Lets create a local volume for our database:

    docker volume create --name=local_postgres
    
We are going to use temporary file `docker-compose.database.yml` file:
    
      postgres:
          image: postgres:9.1
          container_name: some-postgres
          volumes:
            - local_postgres:/var/lib/postgresql/data
          ports:
            - "5432:5432"
          environment:
            POSTGRES_PASSWORD: "$POSTGRES_PASSWORD"
            POSTGRES_USER: "$POSTGRES_USER"
    
And run container with PostgreSQL:
    
    docker-compose -f docker-compose.database.yml up -d
    
Now you have an empty database running in container and accessible on `localhost:5432` (or another host if you run Docker on Mac).

You can connect to this container:

    docker exec -it some-postgres /bin/bash

Switch to right user:

    su postgres

Run `PSQL`:

    psql
    
And, for example, create a new database:

    CREATE DATABASE test;
    
To exit  `PSQL` type:
    
    \q

You can easily remove this container by:
        
    docker-compose -f docker-compose.database.yml stop
    docker-compose -f docker-compose.database.yml rm -f
    
All created data will be stored in `local_postgres` volume.  
Next time when you will need a database you can run a new container and you will already have your `test` database created.


####Dumping local database to file
Let's say you want to dump data from you local database.  
You need to run a simple container and mount a your local directory to some directory in container.  
Then you go inside the container and dump data to file in this mounted directory.

Lets do it:

    postgres:
      image: postgres:9.1
      container_name: some-postgres
      volumes:
        - local_postgres:/var/lib/postgresql/data
        - .:/data
      ports:
        - "5432:5432"
      environment:
        POSTGRES_PASSWORD: "$POSTGRES_PASSWORD"
        POSTGRES_USER: "$POSTGRES_USER"
        
In this example we are going to mount our current directory to directory `/data` inside the container.

Go inside:
    
    docker exec -it some-postgres /bin/bash
    su postgres

And dump:

    pg_dump --host 'localhost' -U postgres test -f /data/test.out
    
This command will dump test database to file test.out in your current directory.  
You can delete this container now.

####Loading data from file

Use the same technique with mounting directories to load data from file. 

Use this command to load data if you are already in postgres container and have this `/data/test.out` mounted.
    
    psql --host 'localhost' --username postgres test < /data/test.out

Don't forget to create database `test` before this.


###Git
When the server is running it will create additional files such as `.log`, `.pid`, etc.  
You don't need them to be committed!  
Don't forget to create .gitignore file:

    .idea
    db.sqlide3
    *.pyc
    *.ini
    *.log
    *.pid
    /static
    .vagrant/

###Static files
You might have some problems with serving static files on production only with gunicorn so check this out.  
Don't forget to create an empty folder `/static/` in your project root directory with file `__init__.py` to serve static from it.  
With this schema your settings.py file should include:

    STATIC_URL = '/static/'
    STATIC_ROOT = os.path.join(BASE_DIR, 'static')

And to serve static with gunicorn on production add this to the end of `urls.py`:

    SERVER_ENVIRONMENT = os.getenv('RUN_ENV', '')
    if SERVER_ENVIRONMENT == 'PROD':
        urlpatterns += patterns('', (r'^static/(?P<path>.*)$', 'django.views.static.serve', {'document_root': settings.STATIC_ROOT}), )

Otherwise you may use additional services to serve static files, for example, to use nGinx server.

##The end
Please don't be shy to share your experience of organizing development process in your team  
and best practices of working with Docker and Vagrant.  
Drop me a feedback at:  
__andrii@preply.com__ or [facebook.com/dvoyak](https://www.facebook.com/dvoyak)  

Thank you!


