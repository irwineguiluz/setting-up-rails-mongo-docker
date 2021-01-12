# Setting up Ruby on Rails with Docker and MongoDB

I assume that you have Docker and Docker Compose installed. So let's start!


# Start of the tutorial

First, let's make an empty directory called ***rails-dockerized***.

    mkdir rails-dockerized

cd into the directory

    cd rails-dockerized

The first file we want to make is the Gemfile, this is because we will have docker instantiate the entire dev environment instead of running rails commands directly through our host operating system.

```bash
touch Gemfile
```

Inside the Gemfile, we will write the source that we get our ruby gems.
```rb
source 'https://rubygems.org'
```
*Gemfile*

underneath the source add a line for the rails version we want.

```rb
gem 'rails', '5.0.7'
```
*Gemfile*

If you want the newest version of rails (_6.1.0 as of this writing_), you can install whichever version you want but this is the one that I have tested for this tutorial.

We also need and empty *Gemfile.lock*, this is where the Gemfile versions are recorded by rails.

**The app will not run without an empty *Gemfile.lock***

```bash
touch Gemfile.lock
```

Now we need to make a *Dockerfile*, this is where we define what technologies our docker image will have, such as ruby versions, node.js versions...

```bash
touch Dockerfile
```

Inside the Dockerfile we will specify the minimum requirements for docker to make an image to run a rails app. Each new command can be thought of like a script making layers that become an operating system.

First is the programming language that everything else will need to run on top of. In our case ruby, using the  [FROM](https://docs.docker.com/engine/reference/builder/#from)  command.

```Dockerfile
FROM ruby:2.5.1
```
*Dockerfile*

Next is the RUN command a typical suffix for RUN is apt-get update && apt-get install problems can arise because updating should happen outside your docker image on dockerhub. See a detailed explanation [here](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/).

```Dockerfile
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs
```
*Dockerfile*

[build-essential](https://hub.docker.com/r/continuul/build-essential/)  is a package of trusted compilers for our application.

[libpq-dev](https://zoomadmin.com/HowToInstall/UbuntuPackage/libpq-dev)  is a compiler system to help with port forwarding to databases from virtual machines. A detailed explanation is in the link.

[nodejs](https://hub.docker.com/_/node/)  is a javascript engine that will enable you to run javascript outside of a browser.

We will add more RUN commands which you can think of as the docker equivalent of a command-line script.

```Dockerfile
RUN mkdir /app
```
*Dockerfile*

This will make a directory only within the context of our docker-image.

Now we want to make the */app* directory the working directory with the  [WORKDIR](https://docs.docker.com/engine/reference/builder/#workdir)  command, it's the bash equivalent of cd into a folder. This means that all the commands that you run after this line will happen in the */app* directory.

```Dockerfile
WORKDIR /app
```
*Dockerfile*

We have to ADD the *Gemfile* and the *Gemfile.lock* for docker to reference for the build of our rails app.

```Dockerfile
ADD Gemfile /app/Gemfile
ADD Gemfile.lock /app/Gemfile.lock
```
*Dockerfile*

Since we have our *Gemfile* and the *Gemfile.lock* inside the directory where we will build, the next command is to RUN bundle install. This adds all of the rails dependencies.

```Dockerfile
RUN bundle install
```
*Dockerfile*

Now we want to ADD the current directory to the application directory after that is installed like so.

```Dockerfile
ADD . /app
```
*Dockerfile*

The whole file should look like this:

```Dockerfile
FROM ruby:2.5.1
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs
RUN mkdir /app
WORKDIR /app
ADD Gemfile /app/Gemfile
ADD Gemfile.lock /app/Gemfile.lock
RUN bundle install
ADD . /app
```
*Dockerfile*

This is the file that will be referenced every time we use the build command on the command-line.

As the next step in our configuration, let’s create a *docker-compose.yml*.

```bash
touch docker-compose.yml
```

In this file we will define a config object for the services involved in this app such as the database.

Within this file, because of how docker objects are defined, improper indentation can cause the build to fail.

I will provide a full list of the file after I walk you through what is going on with proper indentation.

```yml
version: '2'
services:
```
*docker-compose.yml*

under services, we will define the external services for the app such as the database (Mongo).

```yml
  db:
    image: mongo:4.4.3
```
*docker-compose.yml*

This is how we can pull an image database from docker hub, in our case Mongo.

Some configuration for MongoDB client connection.

```yml
    restart: always
    environment:
      # mongodb client connection
      MONGO_INITDB_ROOT_USERNAME: mongoroot
      MONGO_INITDB_ROOT_PASSWORD: mongosecret
```
*docker-compose.yml*

We want to give our app a port forwarding service so that the process that docker is running is linked to the process the host operating system is running.

```yml
    ports:
      - "27017:27017"
```
*docker-compose.yml*

Let's add our next service called app:

```yml
  app:
```
*docker-compose.yml*

below app, we want our app to reference the rails application. This means that instead of calling on an image from DockerHub like we did with Mongo. We are going to tell docker to build the app that we configured in the *Dockerfile* with the build:

We will add the current directory with build: .

```yml
    build: .
```
*docker-compose.yml*

Now we will run rails with the command: then bind it to port 0.0.0.0

```yml
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
```
*docker-compose.yml*

We need a thing called volumes:

It is used to say, the directory where the host system shares the */app* directory.

```yml
    volumes:
      - ".:/app"
```
*docker-compose.yml*

We want to set up the port forwarding so that when docker refers its port 3000 in the container, it will forward that to 3001 to the operating system in the same way that we did so for the database.

```yml
    ports:
      - "3001:3000"
```
*docker-compose.yml*

Let's make the app service depend_on the db: service so that when we use our rails app it refers to the database to store data and it is linked to the database.

```yml
    depends_on:
      - db
    links:
      - db
```
*docker-compose.yml*

The whole file should look like this:

```yml
version: '2'
services:
  db:
    image: mongo:4.4.3
    restart: always
    environment:
      # mongodb client connection
      MONGO_INITDB_ROOT_USERNAME: mongoroot
      MONGO_INITDB_ROOT_PASSWORD: mongosecret
    ports:
      - "27017:27017"
  app:
    build: .
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
    volumes:
      - ".:/app"
    ports:
      - "3001:3000"
    depends_on:
      - db
    links:
      - db
```
*docker-compose.yml*

## Let’s move to the Rails part now!

We are going to be interacting with the command-line again in a way that we prefix all of our ordinary rails commands with  _docker-compose run app_

This is the command to download all of the boiler-plate rails.

**note: this will take a few minutes the first time but will be quick next time. the --api flag can be added, is optional and but it will be easier to get started if you use embedded ruby views instead of a front end framework.**

```bash
docker-compose run app rails new . --skip-active-record
```

If you see a confirmation to overwrite Gemfile, just type yes.

If you have permission troubles you can execute this command:
```bash
sudo chown -R $USER:$USER .
```

If you notice, there is no _database.yml_ and no _sqlite3_ gem is added automatically. Now we have to add two gems which will be a bridge for us between Rails and MongoDB.

Add the following gems to  _Gemfile_.

```rb
gem 'mongoid', '~> 6.0'
gem 'bson_ext'
```

Now execute in the command line:

```bash
docker-compose build
```
```bash
sudo chown -R $USER:$USER .
```

Now we have to generate _mongoid.yml_ file which is similar to _database.yml_ file for us.

```bash
docker-compose run app rails g mongoid:config
```
```bash
sudo chown -R $USER:$USER .
```

You have to update *mongoid.yml* file to connect your Mongo database

```yml
     database: nameofyourdatabase
```
*mongoid.yml*

You have to change *localhost* to *db* (name of Mongo container in *docker-compose.yml*)
```yml
     hosts:
       - db:27017
```
*mongoid.yml*

Now you must uncomment and set user and password of your database
```yml
      # The name of the user for authentication.
      user: 'user'

      # The password of the user for authentication.
      password: 'password'
```
*mongoid.yml*

** (Optional) You can generate a Article model to make a test
```bash
docker-compose run app rails g scaffold article title:string detail:text
```
```bash
sudo chown -R $USER:$USER .
```
**

Now run _docker-compose up_ on the command-line in order to start both the Mongo database and the rails application.

```bash
docker-compose up
```

## Let’s move to the MongoDB part now!

If you ran `docker-compose up` , now you must open your MongoDB Client to set your host connection (credentials in *docker-compose.yml*), after that, don't forget to create your collection and its specific user for it.
