# CI/CD 

CI stands for Continuous Integration and CD for Continuous Development. It is an [Agile](https://www.agilealliance.org/agile101/the-agile-manifesto/) concept closely related to Test Driven Development (TDD)

The idea is to run a tought ahead set of tests whenever we apply a change during development, in doing so we have a feedback loop as small as possible that allows us to quickly identify eventual new problems

A real-world situation would be, for example, using a versioning platform (like GitHub) with a pipeline service (like Travis CI) that automatically runs our test units in any needed environment, whenever we make a new commit

We'll immediately see if something goes wrong, and easily locate where the problem comes from depending of the quality of our tests set



## Docker

In this section we'll focus on the main role of docker: setting up a **replicable environment** for your project (with eQual)

[Docker](https://www.docker.com/get-started) is a very complete tool and could very well have its own dedicated page. It works similarly to a virtual machine (VM), the big difference is that it isolates every service into separate containers, thus ensuring you only use the ressources needed and nothing more

While it can feel less intuitive than a VM, Docker provides unrivalled performances plus the possibility to easily connect to the outside world

Below we'll give a specific example to use docker with eQual, if you're looking for more general informations please read the [getting started with docker](https://docs.docker.com/get-started/) guide

### Setup

Go ahead and [install Docker](https://docs.docker.com/get-docker/)

On windows it comes with **Docker Compose**, which aims to automate the deployment process

In its simplest form, all you need to use Docker is a few command lines plus those 2 files in your project :

- Dockerfile
- docker-compose.yml

Directory usually looks like this :

```
project@root/
	.docker/
 		Dockerfile
 		another-config-file.example
	docker-compose.yml
```

### Dockerfile

Dockerfile is used to build an image with additional requirements

The syntax :

**FROM** tells which image you're gonna use as starting point (it can be an existing image from [Docker Hub](https://hub.docker.com/search?q=&type=image), or homemade)

**COPY** tells what files of your project you want to bring inside the image, it can be specific config files aswell as the full project

**WORKDIR** changes your default location inside the directory

**RUN** tells what commands to run within the image; like additional requirements or rights management

> For more informations about dockerfile syntax, read [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

Here is an example using eQual with a php-apache image : 

```dockerfile
FROM php:7.3-apache   # official image taken from docker hub
COPY . /var/www/html	# copy all the project into apache html folder (must be at project's root)

RUN docker-php-ext-install pdo_mysql mysqli \	# installing required mysql service
    && chown -R www-data:www-data /var/www \ 	# granting permissions in apache folder
    && cp /etc/apache2/mods-available/rewrite.load /etc/apache2/mods-enabled/	# apache config
    
WORKDIR /var/www/html	# changing current location in directory

COPY .docker/vhost.conf /etc/apache2/sites-available/000-default.conf	# custom apache vhost
COPY .docker/config.inc.php /config/config.inc.php	  # changing DB host to "mysql"
COPY .docker/apache2.conf /etc/apache2/apache2.conf	  # apache config
```

To build an image out of this file, use the following command :

```bash
docker build --file .docker/Dockerfile -t todolist .
```

### docker-compose.yml

docker-compose.yml is used to deploy container(s) from one or multiple image(s)

For example, here is what we used to deploy an apache server along with its mysql service :

```yaml
version: '3'
services:
  localhost:
    build:
      context: .
      dockerfile: .docker/Dockerfile
    image: todolist		# using the image previously created with Dockerfile
    container_name: localhost
    restart: always
    ports:
      - 80:80
    volumes:
      - /var/www/html
    environment:
      - QN_DB_NAME=equal
      - QN_DB_HOST=localhost
      - QN_DB_PORT=3306
      - HTTPS_METHOD=noredirect
      - VIRTUAL_PORT=80
      - VIRTUAL_HOST=localhost
    links:
      - mysql
    
  mysql:
    image: mysql:5.7	# using an official image for mySQL
    container_name: mysql
    restart: always
    ports:
      - 3306:3306
    environment:
      MYSQL_DATABASE: equal
      MYSQL_ALLOW_EMPTY_PASSWORD: 'true'
```

To launch the associated Docker command :

```bash
docker-compose up --build --detach
```

With this, you have what you need to develop and test your application using docker

If you want to see the running containers :

```bash 
docker ps
```

To access the bash CLI of a container :

```bash
docker exec -ti CONTAINER_ID /bin/bash
```

To run a bash script within a container (instead of opening it as CLI) :

```bash
docker exec -ti CONTAINER_ID /bin/bash /full/directory/path/script.sh
```

The following section will cover how that command can be used

### Automating the process & database init

Unfortunately if you want to setup a database once a container has been created, you still need to run a few commands manually. We'll work this around and use it as an opportunity to automate the whole process, from image creation to deployment

What we'll do is create a .cmd script (for windows) that we'll launch from the root of the project, using a few of the commands listed above plus the ones needed for the DB setup of a project called "todolist"

```bash
docker build --file .docker/Dockerfile -t todolist .
docker-compose up --build --detach
sleep 10
docker exec -ti localhost /bin/bash .docker/init.sh
```

You've already seen the 2 first lines, they refer to Dockerfile and docker-compose.yml

Sleep 10 is important, granting time for the containers to establish a bridge before we connect the database

Last line launches the following bash script within the container "localhost"

```bash
php run.php --do=init_db
php run.php --do=init_package --package=core
php run.php --do=init_package --package=todolist
```

> Note: if it returns a "wrong host" error, try to increase the sleep time to 20 or 30 sec. If it still doesn't work it means some config for apache is either missing or uncorrect



## Travis CI

[Travis CI](https://docs.travis-ci.com/) is a testing environment offering many options and compatibilities

Configuration example (in YAML) for a **.travis.yml** file, running tests for a todo-list app running on eQual :

```yaml
dist: xenial
language: php
php: 
  - '7.2'  
  
services:
  - mysql

env:
  global:
    - DB_NAME="equal"
    
before_script:
  # install apache2
  - chmod +x .travis/*.sh
  - .travis/init-php.sh 
  
script: 
  # check that mandatory directories are present and have correct access rights set
  - php run.php --do=test_fs-consistency
  # check ability to connect to the dbms service
  - php run.php --do=test_db-connectivity
  # create an empty database 
  - php run.php --do=init_db  
  # initialise database & demo data
  - php run.php --do=init_package --package=core
  - php run.php --do=init_package --package=todolist
  # run test units
  - php run.php --do=test_package --package=todolist
```



## BitBucket Pipelines

[BitBucket](https://bitbucket.org/) has a built-in testing environment called Pipelines that relies on [Docker](https://www.docker.com/)

They have an amazing [tutorial to get started](https://support.atlassian.com/bitbucket-cloud/docs/get-started-with-bitbucket-pipelines/), aswell as a [YAML validator](https://bitbucket-pipelines.prod.public.atl-paas.net/validator) with all the information you need

Here is a minimal syntax example taken from their page about [build artifacts](https://support.atlassian.com/bitbucket-cloud/docs/publish-and-link-your-build-artifacts/) :

```yaml
image: python:3.5.1

pipelines:
 branches:
  master:
   - step:
    script:
     - python run_tests.py
```
