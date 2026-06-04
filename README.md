# Devops-KHIYANI

This repository contains my work for the Devops labs. The work is done in chronological order from Docker basics, Docker compose, GitHub Actions, Ansible deployment and the extra part with load balancing and front-end.

The folder structure is kept as it was during the labs. Some folders are confusing but I did not change them because the repository was already built like this.

## Repository structure

```
.
├── TD-1/                         # First Docker discovery with Flask cat app
├── TP-1/
│   ├── database/                 # Postgres image and SQL init scripts
│   ├── Backend/
│   │   ├── basic_java_build/      # First Java container
│   │   └── Springboot/            # Simple API backend
│   ├── httpd/                    # Apache reverse proxy and load balancing
│   ├── frontend/                 # Vue frontend served with Nginx
│   └── docker-compose.yml        # Local orchestration
├── .github/workflows/            # CI, Docker build and deployment workflows
└── my-project/ansible/           # Ansible inventory, playbook and roles
```

## Chronology of the work

1. I started with Docker discovery in `TD-1`. I created a Flask application and dockerized it with Alpine, Python, pip and a virtual environment.
2. Then I worked on the 3-tier Docker application in `TP-1`: database, backend API, httpd reverse proxy, docker-compose and image publication.
3. After that I added GitHub Actions in `.github/workflows` to test the backend, run SonarCloud, build Docker images and push them to DockerHub.
4. Then I added Ansible in `my-project/ansible` to install Docker on the server and deploy the containers from DockerHub.
5. At the end I added the front-end and the extra load balancing part with two backend containers: `backend-blue` and `backend-green`.

## TD part 01 - Docker discovery

For the first Docker discovery I created the folder `TD-1`. The application is a small Flask app that displays a random cat gif. It contains:

```
TD-1/
├── Dockerfile
├── app.py
├── requirements.txt
└── templates/index.html
```

The `Dockerfile` uses `alpine:3.21.0`, installs Python, pip and the needed build dependencies, creates a virtual environment, installs Flask and then starts `app.py`.

Important commands used in this part:

```
docker build -t myfirstapp .
docker run -p 8888:5000 myfirstapp
```

The app is then available on:

```
http://localhost:8888
```

I also used basic Docker commands during the discovery part:

```
docker run hello-world
docker pull alpine
docker images
docker run alpine ls -l
docker run alpine echo "hello from alpine"
docker run -it alpine /bin/sh
docker ps
docker ps -a
```

This part was mainly to understand the difference between images and containers and how a container starts from an image.

## TD part 02 - GitHub discovery

For GitHub I used the repository with git commands and pushed my work regularly. I used the repository:

```
https://github.com/Jatin-Khiyani/Devops-KHIYANI.git
```

The important commands used were:

```
git clone <repository-url>
git status
git add .
git commit -m "message"
git push origin main
git log
```

I also used branches `main` and `develop` during the GitHub Actions part.

## TD part 03 - Ansible discovery

For Ansible I first tested the remote connection to the server manually with SSH, then used Ansible ping and ad-hoc commands.

The server used in the inventory is:

```
jatin.khiyani.takima.school
```

Basic Ansible commands:

```
ansible --version
ssh -i <path_to_private_key> admin@jatin.khiyani.takima.school
ansible all -i inventories/setup.yml -m ping
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

The `admin@` part is needed because the remote Debian server uses `admin` as the user. Ansible also uses SSH internally to connect to the server, so the SSH test was useful before writing the playbook.

# Lab 1 - Docker TP

## Database

In this part I created a Postgres database image in `TP-1/database`.

Files used:

```
TP-1/database/
├── Dockerfile
├── 01_CreateScheme.sql
└── 02_InsertData.sql
```

The Dockerfile is:

```
FROM postgres:17.2-alpine

COPY 01_CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02_InsertData.sql /docker-entrypoint-initdb.d/
```

Postgres executes the SQL files automatically because they are copied into `/docker-entrypoint-initdb.d/`.

Terminal commands:

```
docker build -t postgres .
docker network create my-network
docker run -d --name postgres --network my-network -p 5432:5432 -e POSTGRES_PASSWORD=pwd -e POSTGRES_USER=usr -e POSTGRES_DB=db postgres
```

Adminer command:

```
docker run \
  -p "8090:8080" \
  --net=my-network \
  --name=adminer \
  -d \
  adminer
```

### 1-1 For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?

It is better to use `-e` because passwords and private configuration should not be written directly inside the Dockerfile.

If the password is inside the Dockerfile it becomes part of the image definition and can be visible when the image is shared. With `-e`, the value is provided when running the container. It is still simple for local work but it avoids putting secrets directly into the image.

### 1-2 Why do we need a volume to be attached to our postgres container?

A database container must keep data even if the container is removed, rebuilt or updated. If no volume is used, the data is stored inside the container filesystem and can disappear when the container is destroyed.

A Docker volume stores the database files outside of the container lifecycle. This is why the database can keep the same data after restarting or recreating the container.

### 1-3 Document your database container essentials: commands and Dockerfile.

Database essentials:

```
docker build -t postgres .
docker network create my-network
docker run -d --name postgres --network my-network -p 5432:5432 -e POSTGRES_PASSWORD=pwd -e POSTGRES_USER=usr -e POSTGRES_DB=db postgres
docker logs postgres
docker exec -it postgres psql -U usr -d db
```

The Dockerfile is documented above. The SQL files create the `departments` and `students` tables and insert initial data.

## Backend - Basic Java

The first backend step is in:

```
TP-1/Backend/basic_java_build
```

It contains a simple `Main.java` and a Dockerfile. The Docker image compiles the Java file and runs the class.

Commands:

```
docker build -t basic_java .
docker run basic_java
```

The expected output is:

```
Hello World!
```

## Backend - Springboot API

Then I created the Springboot backend in:

```
TP-1/Backend/Springboot
```

The application exposes departments and students endpoints and connects to Postgres using environment variables.

Main useful endpoints:

```
/
/students
/students/{id}
/departments
/departments/{departmentName}
/departments/{departmentName}/students
/departments/{departmentName}/count
```

The database configuration is in `application.yml`:

```
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:postgres}:5432/${DB_NAME:db}
    username: ${DB_USER:usr}
    password: ${DB_PASSWORD:pwd}
```

Commands:

```
docker build -t springboot .
docker run --name springboot \
  --network my-network \
  -p 8080:8080 \
  -e DB_HOST=postgres \
  -e DB_NAME=db \
  -e DB_USER=usr \
  -e DB_PASSWORD=pwd \
  springboot
```

### 1-4 Why do we need a multistage build? And explain each step of this Dockerfile.

We need a multistage build because the application needs Maven and a JDK to build, but it does not need all of that to run.

In the first stage I use `eclipse-temurin:21-jdk-alpine`. This stage installs Maven, copies `pom.xml` and `src`, then runs Maven to create the jar.

In the second stage I use `eclipse-temurin:21-jre-alpine`. This image is smaller because it only contains the runtime. I copy only the jar from the build stage and start it with:

```
java -jar myapp.jar
```

This makes the final image smaller, cleaner and faster to deploy.

## HTTP server and reverse proxy

The reverse proxy is in:

```
TP-1/httpd
```

I used `httpd:2.4` and replaced the Apache configuration with my `httpd.conf`.

Commands used:

```
docker run --rm httpd:2.4 cat /usr/local/apache2/conf/httpd.conf > httpd.conf
docker build -t apache-reverse-proxy .
docker run --rm -p 80:80 -e BACKEND_HOST=host.docker.internal apache-reverse-proxy
```

### 1-5 Why do we need a reverse proxy?

A reverse proxy gives one public entry point to the application. The client does not need to know the backend container or database container.

It is useful because it hides internal services, keeps backend ports closed from the host, makes routing cleaner, and can also be used later for SSL, front-end routing and load balancing.

## Docker compose

The compose file is:

```
TP-1/docker-compose.yml
```

It starts the database, two backend instances, the frontend and the httpd proxy.

Command:

```
docker compose up --build
```

### 1-6 Why is docker-compose so important?

Docker compose is important because this application has several containers. Running them one by one with long `docker run` commands is not clean and is easy to break.

With compose I can describe the services, networks, environment variables, healthcheck and dependencies in one file. After that one command can start the full application.

### 1-7 Document docker-compose most important commands.

```
docker compose up
```
Starts the services.

```
docker compose up -d
```
Starts the services in the background.

```
docker compose up --build
```
Rebuilds images and starts the services.

```
docker compose down
```
Stops and removes compose containers and networks.

```
docker compose ps
```
Shows the running services.

```
docker compose logs
```
Shows logs.

```
docker compose logs -f
```
Follows logs in real time.

```
docker compose exec <service> <command>
```
Runs a command inside a running container.

### 1-8 Document your docker-compose file.

My compose file contains these services:

```
database
backend-blue
backend-green
frontend
httpd
```

The database uses the `.env` file and has a healthcheck with `pg_isready`.

The two backend containers use the same Springboot image but run as two different services. They both connect to the same database.

The frontend is built from `TP-1/frontend` and served with Nginx.

The `httpd` service exposes only port `80:80` to the host. Backend and database ports are not exposed to the host in the final compose file. This is cleaner because users access the application through the proxy.

## Publish images

### 1-9 Document your publication commands and published images in dockerhub.

Commands used:

```
docker login
docker images

docker tag tp-1-database:latest jatinkhiyani/my-database:1.0
docker tag tp-1-backend:latest jatinkhiyani/my-backend:1.0
docker tag tp-1-httpd:latest jatinkhiyani/my-httpd:1.0

docker push jatinkhiyani/my-database:1.0
docker push jatinkhiyani/my-backend:1.0
docker push jatinkhiyani/my-httpd:1.0
```

Images also used later in GitHub Actions and Ansible:

```
jatinkhiyani/tp-devops-database:latest
jatinkhiyani/tp-devops-simple-api:latest
jatinkhiyani/tp-devops-httpd:latest
jatinkhiyani/frontend:latest
```

### 1-10 Why do we put our images into an online repo?

We put Docker images in an online repository because another machine can pull the same image without rebuilding it locally.

This is useful for team work, CI/CD and production deployment. The server can pull a version from DockerHub and run it. It also makes rollback easier because images can be tagged with versions.

# Lab 2 - GitHub Actions

## CI pipeline

The CI workflow is:

```
.github/workflows/main.yml
```

It runs on push to `main` and `develop`, and also on pull requests.

The first job is `test-backend`. It checks out the code, installs Java 21 and runs:

```
mvn clean verify
```

from:

```
TP-1/Backend/Springboot
```

### 2-1 What are testcontainers?

Testcontainers are Java libraries used during tests to start real Docker containers for dependencies.

In this project the backend tests can start a PostgreSQL container for integration tests. This is better than testing with a fake database because the tests are closer to the real application behavior.

The dependencies are in the Springboot `pom.xml`:

```
testcontainers
jdbc
postgresql
```

## Secured variables

### 2-2 For what purpose do we need to use secured variables?

Secured variables are used to store private values like DockerHub token, DockerHub username, Sonar token, SSH private key and database passwords.

They should not be written directly in the repository because the repository can be public and the values would be exposed. GitHub Secrets lets the workflow use the values without printing them in the code.

## Docker build and push workflow

The Docker build workflow is:

```
.github/workflows/build-and-push.yml
```

It runs after the CI workflow has completed successfully on `main`.

It builds and pushes:

```
TP-1/database
TP-1/Backend/Springboot
TP-1/httpd
TP-1/frontend
```

### 2-3 Why did we put needs: build-and-test-backend on this job?

We use `needs` so the Docker image is built only after the backend tests are successful.

Without this dependency, GitHub Actions could build and push an image even if the tests are failing. That is not good because DockerHub could receive a broken version of the application.

In the split pipeline version I used `workflow_run` instead of `needs` because the Docker build is in another workflow. The idea is the same: publish only after the CI is green.

### 2-4 For what purpose do we need to push docker images?

We push Docker images so that the server can pull and run them during deployment.

The image built on GitHub Actions becomes the same image used by Ansible on the remote server. This avoids rebuilding on the server and gives a clear separation between build and deployment.

## Quality Gate

The SonarCloud job is also in:

```
.github/workflows/main.yml
```

It runs after `test-backend` and executes:

```
mvn -B verify sonar:sonar
```

The Sonar values are stored in GitHub Secrets:

```
PROJECT_KEY
ORGANIZATION_KEY
SONAR_TOKEN
```

The goal is to check code quality, maintainability and security problems before accepting the code.

## Split pipelines

I split the pipelines like this:

```
.github/workflows/main.yml              # test backend and Sonar
.github/workflows/build-and-push.yml    # build and push Docker images after CI success
.github/workflows/deploy.yml            # deploy with Ansible
```

This makes the workflows easier to read and avoids mixing testing, build and deployment in one file.

# Lab 3 - Ansible

## Inventory and base commands

The Ansible inventory is:

```
my-project/ansible/inventories/setup.yml
```

Content:

```
all:
  vars:
    ansible_user: admin
    ansible_python_interpreter: /opt/docker_venv/bin/python
  children:
    prod:
      hosts:
        jatin.khiyani.takima.school:
```

### 3-1 Document your inventory and base commands

Commands used:

```
cd my-project/ansible
ansible all -i inventories/setup.yml -m ping
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```

The inventory defines the remote production host and the user `admin`. I also set `ansible_python_interpreter` to use the Python virtual environment where the Docker SDK is installed.

## Playbook and roles

The main playbook is:

```
my-project/ansible/playbook.yml
```

Content:

```
- hosts: all
  gather_facts: true
  become: true

  roles:
    - docker
    - network
    - database
    - app
    - frontend
    - proxy
```

### 3-2 Document your playbook

The playbook runs all roles in order.

`docker` installs Docker and the Python Docker SDK.

`network` creates the Docker network used by all containers.

`database` starts the Postgres container.

`app` starts the Springboot API container.

`frontend` starts the Vue/Nginx front-end container.

`proxy` starts the httpd container and exposes port 80.

To run the playbook:

```
cd my-project/ansible
ansible-playbook -i inventories/setup.yml playbook.yml
```

To check syntax:

```
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
```

### 3-3 Document your docker_container tasks configuration.

The containers are launched with `community.docker.docker_container`.

Database role:

```
name: database
image: "{{ DOCKERHUB_USERNAME }}/tp-devops-database:latest"
restart_policy: unless-stopped
network: my-network
```

Backend role:

```
name: backend
image: "{{ DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest"
restart_policy: unless-stopped
network: my-network
env:
  DB_HOST: "{{ DB_HOST }}"
  DB_NAME: "{{ POSTGRES_DB }}"
  DB_USER: "{{ POSTGRES_USER }}"
  DB_PASSWORD: "{{ POSTGRES_PASSWORD }}"
```

Frontend role:

```
name: frontend
image: "{{ DOCKERHUB_USERNAME }}/frontend:latest"
restart_policy: unless-stopped
network: my-network
```

Proxy role:

```
name: httpd
image: "{{ DOCKERHUB_USERNAME }}/tp-devops-httpd:latest"
ports:
  - "80:80"
env:
  BACKEND_HOST: "{{ BACKEND_HOST }}"
  FRONTEND_HOST: "{{ FRONTEND_HOST }}"
```

## Continuous deployment

The deployment workflow is:

```
.github/workflows/deploy.yml
```

It installs Ansible in GitHub Actions, writes the SSH private key from GitHub Secrets into `private_key.pem`, and runs the Ansible playbook.

The deployment uses these secured values:

```
SSH_PRIVATE_KEY
DOCKERHUB_USERNAME
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD
DB_HOST
BACKEND_HOST
FRONTEND_HOST
```

### Is it really safe to deploy automatically every new image on the hub? explain. What can I do to make it more secure?

It is not always safe to automatically deploy every new image from DockerHub. If a bad image is pushed by mistake, or if the image is compromised, the server could deploy it directly.

To make it more secure I can deploy only after tests and Sonar are green, use GitHub environments with manual approval, use version tags instead of only `latest`, protect the `main` branch, use Docker image scanning, and keep all deployment secrets in GitHub Secrets.

In this repository the deploy workflow is separated in `deploy.yml`. This makes it easier to control when deployment happens.

# Front-end

The front-end is in:

```
TP-1/frontend
```

It is a Vue application. For production it uses:

```
VUE_APP_API_URL=/api
```

This means the browser calls `/api`, and Apache forwards `/api/` to the backend load balancer.

The front-end Dockerfile builds the Vue app with Node and then serves the built files with Nginx.

Commands:

```
cd TP-1/frontend
yarn install
yarn serve
yarn build
```

Docker command:

```
docker build -t frontend .
```

# TP Extra - Load balancing

For the extra part I added load balancing with two backend services in `docker-compose.yml`:

```
backend-blue
backend-green
```

The Apache configuration in `TP-1/httpd/httpd.conf` uses a balancer:

```
<Proxy "balancer://backend-cluster">
    BalancerMember http://backend-blue:8080  route=blue
    BalancerMember http://backend-green:8080 route=green
    ProxySet lbmethod=byrequests
</Proxy>

ProxyPass /api/ balancer://backend-cluster/
ProxyPassReverse /api/ balancer://backend-cluster/
```

## Why can we easily load balance between our backends?

We can load balance easily because the backend API is stateless. The request does not depend on memory stored in one specific backend container.

If one request goes to `backend-blue` and the next request goes to `backend-green`, the application still works because the shared state is in the database.

If the application used sticky sessions or stored user sessions in memory, load balancing would be more complicated because a user would need to keep going to the same backend.

## Checkpoint: do you loadbalance?

Yes, locally the compose file starts two backend containers and Apache distributes `/api/` requests between them using `mod_proxy_balancer`.

As the server crashed I was not able to run it on the web however it is working in local host here is a screen shot. 


![Load Balancing ](Load_Balancing.png
)