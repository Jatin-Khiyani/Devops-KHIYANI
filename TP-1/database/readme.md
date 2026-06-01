# Database
Directory 
```
postgres-lab/
├── Dockerfile
├── init-db/
│   ├── 01-CreateScheme.sql
│   └── 02-InsertData.sql
└── postgres-data/
```

## Creating Postgres 

Here I am creating a postgres database using the offical image on docker.
In the image, we create a database "db" with user as "usr" and password as "pwd" directly built into the docker file. 

The Docker file being used here is 

```
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY init-db/ /docker-entrypoint-initdb.d/
````
Then it is built using 

```
docker build -t my-postgres-img .
```

```
docker run \
  --name postgres-db \
  --network app-network \
  -p 5432:5432 \
  -v "$(pwd)/postgres-data:/var/lib/postgresql/data" \
  -d \
  my-postgres-img

  ```

We create a private network so Adminer and PostgreSQL can communicate with each other. 

```
docker network create app-network
```

We use docker run to build the container using the app-network.



Running Adminer 

```
docker run \
  -p 8090:8080 \
  --network app-network \
  --name adminer \
  -d \
  adminer
```

## Creating init files for the sql database

The offical image of Postgres automatically runs the sql file inside init-db folder 

Created 2 files 

1. CreateScheme.sql
1. InsertData.sql

Then removing the container and re running the image of postgre and admirer

## Question 1

We should be using -e because then the password is not hardcoded into the Dockerfile. Anyone having access to the docker file will be able to find the password. 

-e takes in enviornment variables and can take the password at runtime and will only need to be passed once during startup. 

We can use 

```
docker run \
  --name postgres-db \
  --network app-network \
  -p 5432:5432 \
  -e POSTGRES_DB=db \
  -e POSTGRES_USER=usr \
  -e POSTGRES_PASSWORD=pwd \
  -d \
  postgres:17.2-alpine
```


## Question 2
A containter can be stopped, removed and changed. Each time this happens the data will be lost. In a normal use case we update the container in each version and while maintaence. We would still require the data to be stored when this happens. If it is only stored in the container and not in a volume it will be lost forever 

## Question 3

In this readme I have documented the database creation and process 



