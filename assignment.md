# Assignment (Optional)

## Brief

Practice creating and managing multi-container applications using Docker Compose with a Spring Boot application and PostgreSQL database.

1. **Create Docker Compose Configuration**
   - Use your existing Spring Boot project (e.g., devops-demo)
   - Ensure you have a working Dockerfile from Lesson 4.4
   - Add PostgreSQL and JPA dependencies to your `pom.xml`:
```xml
     <dependency>
         <groupId>org.postgresql</groupId>
         <artifactId>postgresql</artifactId>
         <scope>runtime</scope>
     </dependency>
     <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-data-jpa</artifactId>
     </dependency>
```
   - Update `application.properties` to use environment variables:
```properties
     spring.datasource.url=${SPRING_DATASOURCE_URL}
     spring.datasource.username=${SPRING_DATASOURCE_USERNAME}
     spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}
     spring.jpa.hibernate.ddl-auto=none
     spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
```
   - Build your JAR file: `mvn clean package -DskipTests`
   - Create a `docker-compose.yml` file in your project root with two services:
     - **app service**: Build from your Dockerfile, expose port 8080, set database environment variables
     - **db service**: Use `postgres:15` image, expose port 5432, create a database called `myapp_db`
   - Include a named volume for PostgreSQL data persistence
   - Use `depends_on` to ensure database starts before the application
   - Start your services: `docker compose up -d`
   - Verify both containers are running: `docker compose ps`
   - Test your application endpoint in the browser or with curl
   - Take a screenshot showing both containers running

2. **Practice Docker Compose Commands**
   - View logs from your application service: `docker compose logs app`
   - View logs from your database service: `docker compose logs db`
   - Follow logs in real-time: `docker compose logs -f` (press Ctrl+C to stop)
   - Modify the database name in your docker-compose.yml (e.g., change `myapp_db` to `myapp_db_v2`)
   - Update the corresponding environment variable in the app service
   - Restart your services with rebuild: `docker compose up -d --build`
   - Verify the changes worked by checking logs
   - Stop all services: `docker compose stop`
   - Restart services: `docker compose start`
   - Write a brief explanation (4-5 sentences) describing:
     - The difference between Dockerfile and docker-compose.yml
     - How services communicate using service names (e.g., `db` instead of `localhost`)
     - Why volumes are important for database containers
     - What happens when you run `docker compose down -v`

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.

## References
- Java: https://docs.oracle.com/javase/
- Spring Boot: https://docs.spring.io/spring-boot/docs/current/reference/html/
- PostgreSQL: https://www.postgresql.org/docs/
- OWASP: https://cheatsheetseries.owasp.org/