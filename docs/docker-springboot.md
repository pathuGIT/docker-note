

## Project Structure

```

Springboot-backend/
├─ src/
│  └─ main/
│     ├─ java/
│     └─ resources/
│        └─ application.properties
├─ .env
└─ Dockerfile

````

---

## Dockerfile

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
COPY target/eatsmeet-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
````

This uses a **multi-stage build**:

* **Build stage:** uses Maven to compile and package the Spring Boot app.
* **Runtime stage:** uses lightweight Java runtime to run the packaged `.jar`.

---

## application.properties

```properties
spring.application.name=eatsmeet

# Database
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=update

# Mail
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=${MAIL_USERNAME}
spring.mail.password=${MAIL_PASSWORD}
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

# JWT
jwt.secret=${JWT_SECRET}
```

> Note: Using environment variables allows secure configuration without hardcoding sensitive data.

---

## .env

```env
DB_URL=jdbc:mysql://host.docker.internal:3306/eatsmeet
DB_USERNAME=USER
DB_PASSWORD=1234
MAIL_USERNAME=myemail@gmail.com
MAIL_PASSWORD=my-app-password
JWT_SECRET=MN##...
```

> Replace placeholders with your actual database credentials, email, and JWT secret.

---

## Build & Run the Docker Container

1. **Package the Spring Boot app**

```bash
mvn clean package -DskipTests spring-boot:repackage
```

2. **Build the Docker image**

```bash
docker build -t eatsmeet-server:latest .
```

3. **Run the container with environment variables**

```bash
docker run --env-file .env -p 8080:8080 eatsmeet-server:latest
```

* `-p 8080:8080` maps container port 8080 to host port 8080.
* `--env-file .env` loads environment variables from your `.env` file.

---

## Notes

* Use **host.docker.internal** in DB URL for local Docker connections to your host machine’s database.
* For production, update `DB_URL` to the database container or cloud database hostname.
* Multi-stage Dockerfile reduces image size by only including the runtime in the final image.
* Environment variables are used for **security** and **flexibility**.

---
