# Lab 5: Multi-Stage Build for Java Spring Boot App

# Stage 1: Build stage with Maven and Java 17
FROM maven:3.8.4-eclipse-temurin-17-alpine AS build

WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# Stage 2: Runtime stage with Java 17 only
FROM eclipse-temurin:17-jdk-alpine

WORKDIR /app
COPY --from=build /app/target/demo-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
