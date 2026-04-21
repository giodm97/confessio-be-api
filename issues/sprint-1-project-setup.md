---
repo: giodm97/confessio-be-api
title: "Sprint 1 — Spring Boot project setup + Docker Compose + application.properties"
labels: enhancement
status: pending
---

## Overview

Bootstrap the complete Spring Boot 3.5 / Java 21 project for confessio-be-api.
Generate via Spring Initializr, configure pom.xml with all required dependencies,
set up application.properties for local and test profiles, add Docker Compose for
local PostgreSQL, and add Procfile for Elastic Beanstalk.

## Changes required

### 1. Generate project via Spring Initializr

```bash
curl -s -L "https://start.spring.io/starter.zip?\
type=maven-project&language=java&bootVersion=3.5.0\
&groupId=com.confessio&artifactId=confessio-be-api\
&name=confessio-be-api&packageName=com.confessio.api\
&javaVersion=21\
&dependencies=web,data-jpa,postgresql,lombok,security,validation" \
-o confessio-be-api.zip
unzip -q confessio-be-api.zip
rm confessio-be-api.zip
```

Extract contents into the repo root (not into a subdirectory).

### 2. `pom.xml`

Add inside `<properties>`:
```xml
<mapstruct.version>1.6.3</mapstruct.version>
```

Add inside `<dependencies>`:
```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${mapstruct.version}</version>
</dependency>
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.6</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

Configure compiler plugin (Lombok before MapStruct):
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok-mapstruct-binding</artifactId>
                <version>0.2.0</version>
            </path>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>${mapstruct.version}</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

### 3. `src/main/resources/application.properties`

```properties
spring.application.name=confessio-be-api
server.servlet.context-path=/api

spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5432/confessio}
spring.datasource.username=${DB_USERNAME:confessio}
spring.datasource.password=${DB_PASSWORD:confessio}
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration

management.endpoints.web.exposure.include=health
management.endpoint.health.show-details=never

springdoc.api-docs.path=/v3/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.enabled=true

ai.base-url=${AI_BASE_URL:https://api.groq.com/openai/v1}
ai.api-key=${AI_API_KEY:}
ai.model=${AI_MODEL:llama-3.3-70b-versatile}
ai.max-tokens=${AI_MAX_TOKENS:400}
```

### 4. `src/test/resources/application.properties`

```properties
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.H2Dialect
spring.flyway.enabled=false
```

### 5. `docker-compose.yml` (repo root)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: confessio-postgres
    environment:
      POSTGRES_DB: confessio
      POSTGRES_USER: confessio
      POSTGRES_PASSWORD: confessio
    ports:
      - "5432:5432"
    volumes:
      - confessio-pg-data:/var/lib/postgresql/data
  adminer:
    image: adminer:latest
    container_name: confessio-adminer
    restart: unless-stopped
    ports:
      - "8090:8080"
    depends_on:
      - postgres
volumes:
  confessio-pg-data:
```

### 6. `Procfile` (repo root)

```
web: java -jar target/confessio-be-api-0.0.1-SNAPSHOT.jar
```

### 7. Package directories

Create the following empty directories under
`src/main/java/com/confessio/api/`:
`ai/`, `config/`, `constants/`, `controller/`, `dto/request/`, `dto/response/`,
`entity/`, `exception/`, `mapper/`, `repository/`, `service/`

Place a `.gitkeep` file in each empty directory.

## Acceptance criteria

- [ ] `./mvnw verify -DskipTests` completes without errors
- [ ] All required package directories exist under `src/main/java/com/confessio/api/`
- [ ] `docker-compose.yml` present in repo root
- [ ] `Procfile` present in repo root
- [ ] `application.properties` contains `server.servlet.context-path=/api`
- [ ] Test `application.properties` uses H2 with Flyway disabled
