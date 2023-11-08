<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#docker">Docker</a>
      <ul>
        <li><a href="#database">Database</a></li>
        <li><a href="#backend-api">Backend API</a></li>
        <li><a href="#docker-compose">Docker-compose</a></li>
        <li><a href="#publish">Publish</a></li>
      </ul>
    </li>
    <li>
      <a href="#github-actions">Github Actions</a>
      <ul>
        <li><a href="#build-and-test-your-application">Build and test your application</a></li>
        <li><a href="#quality-gate">Quality Gate</a></li>
      </ul>
    </li>
    <li><a href="#ansible">Ansible</a></li>
  </ol>
</details>

# Docker

## Database
### 1-1 Document your database container essentials: commands and Dockerfile.

The database Dockerfile looks like this:

```
FROM postgres:14.1-alpine

COPY /sql /docker-entrypoint-initdb.d

ENV POSTGRES_DB=db \ POSTGRES_USER=usr \ POSTGRES_PASSWORD=pwd
```

It copies the sql files to the docker container so that it creates and fills up the database when run. It also contains the environment variables (for now) to create the database.

We also created a network so that we're able to link our containers, using the command

    docker network create app-network

The command to build the image is

    docker build -t lzablit/db .

and to run it, with a volume for data persistence

    docker run --network app-network --name db -v sql-data:/var/lib/postgresql/data -d lzablit/db

We can then run an adminer docker container on the same network so that we can access our database on port 8090:

    docker run \
    -p "8090:8080" \
    --network app-network \
    --name adminer \
    -d \
    adminer


## Backend API
### 1-2 Why do we need a multistage build? And explain each step of this dockerfile.

Multi-stage builds allow us to separate the build environment from the runtime environment by having multiple 'FROM's. 
It is useful if we need different dependencies during the build and the runtime, making smaller images that can be reused (if unchanged) without repeating the steps each time.
Unnecessary build tools and dependencies are not present in the final image, the Dockerfile is more organized, and it's easier to manage the build process.

```docker
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
// This step sets the base image for the build to Maven (with Amazon Corretto 17), and AS creates a name for the build stage to be referenced to.

ENV MYAPP_HOME /opt/myapp
// Adds an environement variable called MYAPP_HOME which indicates the path to the app we are building.

WORKDIR $MYAPP_HOME
// Points us to the directory we are working in, used for the COPY and RUN steps.

COPY pom.xml .
COPY src ./src
// Copies the pom.xml and src directory to the docker container.

RUN mvn package -DskipTests
// Runs the mvn package command inside the container, therefore building the app and creating a jar file.


FROM amazoncorretto:17
// Sets the base image to amawoncorretto

ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
// Copies the result of the build (the jar) to the docker container.

ENTRYPOINT java -jar myapp.jar
// Runs the java application by executing the jar.
```

## Docker-compose
### 1-3 Document docker-compose most important commands. 

The most important docker-compose commands are docker-compose up (-d if we want to run it detached from the terminal) to launch all the steps specified in the file (building the images if not already present and running the containers, with configurations from the file) and docker-compose down to kill the running containers.

### 1-4 Document your docker-compose file.

```yaml
version: '3.7'

services:
    api:
        build:
          context: ./simpleapi
        container_name: api
        ports:
          - 8080:8080
        networks:
          - app-network
        depends_on:
          - db
        platform: linux/amd64

    db:
        build:
          context: ./db
        container_name: db
        networks:
          - app-network
        volumes:
          - sql-data:/var/lib/postgresql/data
        env_file:
          - .env
        platform: linux/amd64

    proxy:
        build:
          context: proxy
        container_name: proxy
        ports:
          - 80:80
        networks:
          - app-network
        depends_on:
          - api
        platform: linux/amd64


networks:
  app-network:

volumes:
  sql-data:
```
## Publish

### 1-5 Document your publication commands and published images in dockerhub.

Connexion to Dockerhub

    docker login -u <username> -p <password>
    
Tag the images

    docker tag tp-01-api lzablit1/tp-01-api:1.0
    
Push the images to dockerhub

    docker push lzablit1/tp-01-api:1.0
    
We can then see the images on dockerhub:

<img width="936" alt="Screenshot 2023-11-08 at 1 10 58 AM" src="https://github.com/laura-zab/td-02-cicd/assets/123493737/9283db12-a20f-4c5e-ad11-99655e40ddc7">

# Github Actions

## Build and test your application
### 2-1 What are testcontainers?

Testcontainers is a Java library that provides a way to run Docker containers for integration testing.

### 2-2 Document your Github Actions configurations.

Below is main.yml in .github/workflows directory.
This gives the directives and steps for the pipeline that is launched when we push (here on main) to Github.
This job runs tests on the API.

```yaml
name: CI devops 2023 
on:
  push:
  branches:
    -'main' 
  pull_request:

jobs:
  test-backend:
    runs-on: Ubuntu-22.04 
    env:
      working-directory: /simpleapi 
    defaults:
      run:
        working-directory: ${{ env.working-directory }} 
      steps:
        - Uses: actions/checkout@v4
        
        - name: Set up JDK 17
          uses: actions/setup-java@v3
          with:
            distribution: 'temurin'
            java-version: '17' 
            cache: maven
            
        - name: Build and test with Maven 
          run: mvn clean verify
```
## Quality Gate  
### Document your quality gate configuration.

First we need to create an organization and a project in Sonar that includes this Github repo. We retrieve a project key that we'll use later.

We add this to the main.yml (replacing the build and test with Maven step) to use Sonar as a quality gate.

```yaml
  - name: Cache SonarCloud packages
    uses: actions/cache@v3
    with:
      path: ~/.sonar/cache
      key: ${{ runner.os }}-sonar

  - name: Cache Maven packages
    uses: actions/cache@v3
    with:
      path: ~/.m2
      key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
      restore-keys: ${{ runner.os }}-m2

  - name: Build and analyze
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=laura-zab_td-02-cicd
```

To use this we have to add secrets to Github containing the Sonar and Github tokens. The analysis runs when we push (at the same time as the pipeline), and we can see the results on the Sonar page.

# Ansible
## Introduction
### 3-1 Document your inventory and base commands

