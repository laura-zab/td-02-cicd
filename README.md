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
    <li>
      <a href="#ansible">Ansible</a>
      <ul>
        <li><a href="#introduction">Introduction</a></li>
        <li><a href="#playbooks">Playbooks</a></li>
        <li><a href="#deploy-your-app">Deploy your app</a></li>
      </ul>
    </li>
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

```dockerfile
# This step sets the base image for the build to Maven (with Amazon Corretto 17), and AS creates a name for the build stage to be referenced to.
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build

# Adds an environement variable called MYAPP_HOME which indicates the path to the app we are building.
ENV MYAPP_HOME /opt/myapp

# Points us to the directory we are working in, used for the COPY and RUN steps.
WORKDIR $MYAPP_HOME

# Copies the pom.xml and src directory to the docker container.
COPY pom.xml .
COPY src ./src

# Runs the mvn package command inside the container, therefore building the app and creating a jar file.
RUN mvn package -DskipTests


# Sets the base image to amazoncorretto
FROM amazoncorretto:17

# Sets the working directory to /opt/myapp using an environment variale.
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME

# Copies the result of the build (the jar) to the docker container.
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# Runs the java application by executing the jar.
ENTRYPOINT java -jar myapp.jar
```

## Docker-compose
### 1-3 Document docker-compose most important commands. 

The most important docker-compose commands are docker-compose up (-d if we want to run it detached from the terminal) to launch all the steps specified in the file (building the images if not already present and running the containers, with configurations from the file) and docker-compose down to kill the running containers.

### 1-4 Document your docker-compose file.

```yaml
version: '3.7'  # Specifies the version of the Compose file. This file is compatible with Compose version 3.7 and above.

services:  # Defines a collection of services that make up the application.
    api:  # Defines a service named "api".
        build:  # Specifies the build configuration for the service.
            context: ./simpleapi  # Indicates the directory containing the Dockerfile for the service.
        container_name: api  # Sets the name of the container for the service.
        ports:  # Exposes ports for the service.
          - 8080:8080  # Maps port 8080 on the host machine to port 8080 in the container.
        networks:  # Declares the networks to which the service should be connected.
          - app-network  # Connects the service to the "app-network" network.
        depends_on:  # Specifies services that the current service depends on.
          - db  # The "api" service depends on the "db" service.
        platform: linux/amd64  # Sets the platform for the container. In this case, it's a Linux machine with an AMD64 architecture because I had an issue building the images for the right platform because of my machine's platform.

    db:  # Defines a service named "db".
        build:  # Specifies the build configuration for the service.
            context: ./db  # Indicates the directory containing the Dockerfile for the service.
        container_name: db  # Sets the name of the container for the service.
        networks:  # Declares the networks to which the service should be connected.
          - app-network  # Connects the service to the "app-network" network.
        volumes:  # Mounts volumes to the container.
          - sql-data:/var/lib/postgresql/data  # Mounts the "sql-data" volume to the `/var/lib/postgresql/data` directory in the container.
        env_file:  # Specifies environment files to provide configuration values to the container.
          - .env  # Loads the `.env` file.
        platform: linux/amd64  # Sets the platform for the container.

    proxy:  # Defines a service named "proxy".
        build:  # Specifies the build configuration for the service.
            context: proxy  # Indicates the directory containing the Dockerfile for the service.
        container_name: proxy  # Sets the name of the container for the service.
        ports:  # Exposes ports for the service.
          - 80:80  # Maps port 80 on the host machine to port 80 in the container.
        networks:  # Declares the networks to which the service should be connected.
          - app-network  # Connects the service to the "app-network" network.
        depends_on:  # Specifies services that the current service depends on.
          - api  # The "proxy" service depends on the "api" service.
        platform: linux/amd64  # Sets the platform for the container.

networks:  # Defines a network named "app-network".
  app-network:

volumes:  # Defines a volume named "sql-data".
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

# Trigger the jobs when there is a push to the 'main' branch
on:
  push:
    branches:
      - 'main'
  pull_request:

# Define a job named "test-backend" that runs on Ubuntu 22.04
jobs:
  test-backend:
    runs-on: Ubuntu-22.04

    # Set the working directory to the 'simpleapi' directory
    env:
      working-directory: /simpleapi

    # Set the default working directory for all steps to the value of the 'working-directory' environment variable
    defaults:
      run:
        working-directory: ${{ env.working-directory }}

    # Define the steps for the job
    steps:
      # Checkout the repository code
      - Uses: actions/checkout@v4

      # Install JDK 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'
          cache: maven

      # Build and test the project using Maven
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
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=laura-zab_td-02-cicd
```

To use this we have to add secrets to Github containing the Sonar and Github tokens. The analysis runs when we push (at the same time as the pipeline), and we can see the results on the Sonar page.

<img width="1090" alt="Screenshot 2023-11-08 at 3 58 32 PM" src="https://github.com/laura-zab/td-02-cicd/assets/123493737/479252aa-8570-47ef-bb8a-b3c022b529ec">

# Ansible
## Introduction
### 3-1 Document your inventory and base commands

```yaml
# This inventory file defines which hosts Ansible will manage
all:
  # This group represents all hosts that Ansible will manage
  vars:
    # Set the Ansible user to centos for all hosts
    ansible_user: centos

    # Set the path to the SSH private key file 
    ansible_ssh_private_key_file: ../../../id_rsa

  # This group contains all hosts that Ansible will manage
  children:
    # The 'prod' group contains production hosts
    prod:
      hosts:
        # This host is named 'laura.zablit.takima.cloud' and is managed by Ansible
        - laura.zablit.takima.cloud
```

We can then ping the host to test the setup with this command:

```bash
ansible all -i inventories/setup.yml -m ping
```

We can request the server to get the OS distribution, thanks to the setup module:
```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

We can also remove the httpd server that we installed earlier:
```bash
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```


## Playbooks
### 3-2 Document your playbook

After running this command,
```bash
ansible-galaxy init roles/docker
```
we have defined a docker role in which we will write the whole docker task to execute.
We can then define the role in the playbook so that it's executed:

```yaml
# Define the target hosts for the playbook (here: all)
- hosts: all
  # Disable gathering of facts, as we are not using them in this playbook
  gather_facts: false
  # Enable becoming root user on all hosts, as we need to install packages and manage services
  become: yes
  # List the roles to be executed in this playbook. Apply the 'docker' role to all hosts
  roles:
    - docker
```

Here is the tasks/main.yml file that will be executed when calling the docker role from the playbook:

```yaml
# Install Docker
- name: Clean packages
  command:
    cmd: yum clean -y packages

- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

## Deploy your app
### Document your docker_container tasks configuration.


Creation of a network:

```yaml
# This task creates a Docker network named 'app-network'
- name: Create a network
  # Use the docker_network module to create the network
  docker_network:
    # Set the name of the network to 'app-network'
    name: app-network

```

Launch the database:

```yaml
# This task runs a Docker container named 'db' based on the db image we pushed earlier.
- name: Run Database
  docker_container:
    # Set the name of the container to 'db'
    name: db

    # Specify the Docker image to use for the container
    image: lzablit1/td-02-cicd-db:1.0

    # Connect the container to the 'app-network'
    networks:
      - name: app-network

    # Set environment variables for the database connection details
    env:
      POSTGRES_DB: "db"
      POSTGRES_USER: "usr"
      POSTGRES_PASSWORD: "pwd"

    # Pull the Docker image if it is not already present on the host
    pull: true

    # Recreate the container if it already exists
    recreate: true

    # Specify the platform for the container, which is linux/amd64 in this case
    platform: linux/amd64

```

Run the API:

```yaml
# This task runs a Docker container named 'api' based on the api image we pushed.
- name: Run App
  docker_container:
    # Set the name of the container to 'api'
    name: api

    # Specify the Docker image to use for the container
    image: lzablit1/td-02-cicd-api:1.0

    # Expose port 8080 of the container to port 8080 on the host, allowing external access to the application and for the proxy to access the backend
    ports: "8080:8080"

    # Connect the container to the 'app-network', enabling communication with other containers in the network
    networks:
      - name: app-network

    # Pull the Docker image if it is not already present on the host
    pull: true

    # Recreate the container if it already exists, ensuring it always runs with the latest image and configuration
    recreate: true

    # Specify the platform for the container, which is linux/amd64 in this case
    platform: linux/amd64

```

Set the proxy:

```yaml
# This task runs a Docker container named 'proxy' based on the proxy image we pushed.
- name: Run Proxy
  docker_container:
    # Set the name of the container to 'proxy'
    name: proxy

    # Specify the Docker image to use for the container
    image: lzablit1/td-02-cicd-proxy:1.0

    # Expose port 80 of the container to port 80 on the host, enabling external access to the proxy
    ports: "80:80"

    # Connect the container to the 'app-network', allowing it to communicate with other containers in the network
    networks:
      - name: app-network

    # Pull the Docker image if it is not already present on the host
    pull: true

    # Recreate the container if it already exists, ensuring it always runs with the latest image and configuration
    recreate: true

    # Specify the platform for the container, which is linux/amd64 in this case
    platform: linux/amd64
```

We can also add the frontend:

```yaml
# This task runs a Docker container named 'frontend' based on the frontend image we pushed.
- name: Run Frontend
  docker_container:
    # Set the name of the container to 'frontend'
    name: frontend

    # Specify the Docker image to use for the container
    image:  lzablit1/td-02-cicd-frontend:1.0

    # Expose port 8082 of the container to port 8082 on the host, enabling external access to the frontend and for the proxy to access the frontend
    ports: "8082:8082"

    # Connect the container to the 'app-network', allowing it to communicate with other containers in the network
    networks:
      - name: app-network

    # Pull the Docker image if it is not already present on the host
    pull: true

    # Recreate the container if it already exists, ensuring it always runs with the latest image and configuration
    recreate: true

    # Specify the platform for the container, which is linux/amd64 in this case
    platform: linux/amd64
```

We then have this playbook that calls all these roles to run our application:

```yaml
- hosts: all
  gather_facts: false
  become: yes
  roles:
    - docker
    - network
    - database
    - api
    - proxy
    - frontend
```


