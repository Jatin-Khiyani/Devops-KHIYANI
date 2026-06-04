# TP-1 Docker 3-tier application

In this part I created a 3-tier application with Docker.

The application contains:

1. A Postgres database.
2. A Springboot backend API.
3. An Apache httpd reverse proxy.
4. A Vue front-end added later.
5. Docker compose to run everything together.

## Folder content

```
TP-1/
├── database/
├── Backend/
│   ├── basic_java_build/
│   └── Springboot/
├── httpd/
├── frontend/
├── .env
└── docker-compose.yml
```

## Database

The database is in `TP-1/database`.

`Dockerfile`:

```
FROM postgres:17.2-alpine

COPY 01_CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02_InsertData.sql /docker-entrypoint-initdb.d/
```

`01_CreateScheme.sql` creates the tables.

`02_InsertData.sql` inserts the first data.

### Commands

```
cd TP-1/database
docker build -t postgres .
docker network create my-network
docker run -d --name postgres --network my-network -p 5432:5432 -e POSTGRES_PASSWORD=pwd -e POSTGRES_USER=usr -e POSTGRES_DB=db postgres
```

Adminer:

```
docker run \
  -p "8090:8080" \
  --net=my-network \
  --name=adminer \
  -d \
  adminer
```

### 1-1 For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?

It is better because passwords and private configuration should not be stored in the Dockerfile. If the image is pushed online, the Dockerfile values can be exposed.

With `-e`, the values are provided when the container starts. It keeps the image reusable and avoids hardcoding secrets.

### 1-2 Why do we need a volume to be attached to our postgres container?

A database needs persistent storage. If the data is only stored inside the container, it can be lost when the container is removed.

A volume stores the database data outside of the container lifecycle. This allows the data to survive restarts and rebuilds.

### 1-3 Document your database container essentials: commands and Dockerfile.

The main database commands are:

```
docker build -t postgres .
docker run -d --name postgres --network my-network -p 5432:5432 -e POSTGRES_PASSWORD=pwd -e POSTGRES_USER=usr -e POSTGRES_DB=db postgres
docker logs postgres
docker exec -it postgres psql -U usr -d db
```

The Dockerfile copies SQL scripts inside `/docker-entrypoint-initdb.d/` so Postgres executes them on first startup.

## Basic Java backend

The first backend test is in:

```
TP-1/Backend/basic_java_build
```

Commands:

```
docker build -t basic_java .
docker run basic_java
```

This was used to check that Java can be compiled and executed inside a Docker container.

## Springboot backend API

The real backend is in:

```
TP-1/Backend/Springboot
```

It exposes students and departments APIs. The configuration uses environment variables for the database:

```
DB_HOST
DB_NAME
DB_USER
DB_PASSWORD
```

### Commands

```
cd TP-1/Backend/Springboot
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

A multistage build keeps the final image smaller.

The first stage uses a JDK image and Maven. It copies the source code and builds the jar.

The second stage uses a JRE image. It only copies the jar from the first stage and runs it.

So the final image contains only what is needed to run the application, not the full build tools.

## HTTP server

The HTTP server is in:

```
TP-1/httpd
```

It is based on `httpd:2.4`. I copied my own `httpd.conf` into the image.

### Commands

```
docker run --rm httpd:2.4 cat /usr/local/apache2/conf/httpd.conf > httpd.conf
docker build -t apache-reverse-proxy .
docker run --rm -p 80:80 -e BACKEND_HOST=host.docker.internal apache-reverse-proxy
```

### 1-5 Why do we need a reverse proxy?

A reverse proxy gives one public entry point to the application.

It hides internal containers like backend and database. It also helps to route requests, serve the front-end, configure SSL later and load balance between multiple backend containers.

## Docker compose

The compose file is:

```
TP-1/docker-compose.yml
```

Current services:

```
database
backend-blue
backend-green
frontend
httpd
```

Start everything:

```
docker compose up --build
```

Stop everything:

```
docker compose down
```

### 1-6 Why is docker-compose so important?

Docker compose is important because the project uses several containers. It is not practical to start database, backend, frontend and proxy manually every time.

With compose the configuration is written once and the whole application starts with one command.

### 1-7 Document docker-compose most important commands.

```
docker compose up
```
Starts all services.

```
docker compose up -d
```
Starts services in background.

```
docker compose up --build
```
Rebuilds images and starts the containers.

```
docker compose down
```
Stops and removes containers and networks.

```
docker compose ps
```
Shows running services.

```
docker compose logs -f
```
Follows logs.

```
docker compose exec <service> <command>
```
Runs a command inside a service.

### 1-8 Document your docker-compose file.

The database service builds `./database`, reads variables from `.env` and uses `pg_isready` as a healthcheck.

The backend is duplicated into `backend-blue` and `backend-green`. Both use the Springboot app and connect to the database.

The frontend builds the Vue app.

The httpd service exposes port 80 and forwards requests to frontend or to the backend balancer.

The backend and database ports are not exposed directly to the host in the final version. Only the proxy is public.

## Publish

### 1-9 Document your publication commands and published images in dockerhub.

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

Images used in the final pipeline:

```
jatinkhiyani/tp-devops-database:latest
jatinkhiyani/tp-devops-simple-api:latest
jatinkhiyani/tp-devops-httpd:latest
jatinkhiyani/frontend:latest
```

### 1-10 Why do we put our images into an online repo?

We put images online so other machines can pull them directly.

This is needed for deployment because the server does not need to rebuild the image. It just pulls the image from DockerHub and runs it.

## Frontend

The frontend is in:

```
TP-1/frontend
```

Development commands:

```
yarn install
yarn serve
yarn build
```

Docker command:

```
docker build -t frontend .
```

For production the frontend uses:

```
VUE_APP_API_URL=/api
```

This is important because Apache receives `/api` requests and forwards them to the backend.

## Load balancing extra

I added two backend containers in compose:

```
backend-blue
backend-green
```

In `TP-1/httpd/httpd.conf`, Apache uses:

```
<Proxy "balancer://backend-cluster">
    BalancerMember http://backend-blue:8080  route=blue
    BalancerMember http://backend-green:8080 route=green
    ProxySet lbmethod=byrequests
</Proxy>
```

Then `/api/` is forwarded to the balancer.

We can load balance easily because the backend is stateless. The data is stored in the database, not in the backend container memory. If the app used sticky sessions, load balancing would be more difficult.

Grafana was not implemented in this repository.

