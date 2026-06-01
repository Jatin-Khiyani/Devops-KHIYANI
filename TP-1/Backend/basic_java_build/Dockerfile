FROM eclipse-temurin:21-jdk

COPY Main.java /Main.java

RUN javac /Main.java

ENTRYPOINT ["java", "-cp", "/", "Main"]