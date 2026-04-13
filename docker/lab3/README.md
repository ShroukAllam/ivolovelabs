# Lab 3: Docker Container for Spring Boot App

## Steps:
1. Clone code
2. Write Dockerfile with Maven + Java 17
3. Build image: `docker build -t app1 .`
4. Run container: `docker run -d --name container1 -p 8082:8080 app1`
5. Test: `curl http://localhost:8082`
6. Clean: `docker stop container1 && docker rm container1`

## Dockerfile:
FROM maven:3.8.4-openjdk-17
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests
EXPOSE 8080
CMD ["java", "-jar", "target/demo-0.0.1-SNAPSHOT.jar"]
