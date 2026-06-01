# Backend 
## Creating java file 

```
public class Main {

   public static void main(String[] args) {
       System.out.println("Hello World!");
   }
}
```

Compiling Main.java file using 
```
javac Main.java

```

## DockerFile

First creating a Dockerfile for java build 

```
FROM eclipse-temurin:22-jre-alpine

WORKDIR /usr/src/app

COPY Main.class .

CMD ["java", "Main"]

```

Building it using 

```
docker build -t java-hello-img . 
```

Running the container 

```
docker run java-hello-world
```

## Creating API 

Now changing Main.java and using code 

