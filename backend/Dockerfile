FROM maven:3.9-eclipse-temurin-17-alpine AS build

WORKDIR /app

COPY pom.xml ./
COPY src src/

RUN mvn clean package -DskipTests

# Run the app in a minimal Java container
FROM eclipse-temurin:17-jre-alpine

# Patch base OS packages
RUN apk update && apk upgrade --no-cache

WORKDIR /app

# Create a dedicated non-root user
RUN addgroup -S -g 10001 appgroup && \
    adduser -S -u 10001 -G appgroup appuser

COPY --from=build /app/target/*.jar app.jar

# Run Spring Boot as non-root
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD wget --spider -q http://127.0.0.1:8080/actuator/health || exit 1