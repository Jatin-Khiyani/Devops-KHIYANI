# Database

In this part I create a database following the instructions given in the module. 

`Dockerfile` makes the Postgres database and assigns user name and password. It also indicates the location of `01_CreateSchema.sql` and `02_InsertData.sql` files to create tables and enter data. These are automatically execute by Postgres. 

## Terminal Comandas

1. To Build the Image

```
docker build -t postgres . 
```

2. To create a new network that will be used throughout the lab 

```
docker network create my-network
```

3. To run the container 

```
docker run -d --name postgres --network my-network -p 5432:5432 -e POSTGRES_PASSWORD=pwd -e POSTGRES_USER=usr -e POSTGRES_DB=db  postgres  
```

4. To run admirer

```
 docker run \                                                                                                                
    -p "8090:8080" \
    --net=my-network \
    --name=adminer \
    -d \
    adminer
```

## Questions 

1. **1-1 For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?**

Passwords and private keys are secrets and should no be available to the public. If they are mentioned in the image defination, thoes passwords and private keys will be avaialble to the public. 

When used wihth -e these varibles are only passed once in the admins terminal. They are not inculded in the image defination and not available to the public 

2. **1-2 Why do we need a volume to be attached to our postgres container?**

in a PostSQL container storage is not temporary. When the container is deleted, changed or update the data might be lost. This data is valuble to all parities envolved. 

A docker volume is permanent storage stored locally or on a server running the container. Volume is not effected by container shut downs, updated or duplication. 

3. **1-3 Document your database container essentials: commands and Dockerfile.**

For the database this readme contains all comands and information about the container