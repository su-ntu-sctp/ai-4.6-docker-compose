# Lesson: Cloud-Native Applications with Docker Compose (Running Multiple Containers)

---

## Lesson Overview

In this lesson, you will learn to run **multiple containers together** using **Docker Compose**. You will run your Spring Boot application with a PostgreSQL database to understand multi-container orchestration.

---

## Learning Objectives

By the end of this lesson, you will be able to:

1. Explain why Docker Compose is needed for cloud-native applications  
2. Configure a Docker Compose file to run multiple containers  
3. Run and verify a Spring Boot application and database using Docker Compose  
4. Inspect running services and logs using Docker Compose commands  

---

## Prerequisites

Before starting this lesson, ensure you have:

### 1. Your Spring Boot DevOps Demo Project Ready

Your project should have:
- A working Dockerfile (from Lesson 2)
- A `/hello` endpoint that returns "DevOps demo application is running!"
- Successfully builds with Maven

### 2. Project Structure

```
devops-demo/
 ├── src/
 │   └── main/
 │       ├── java/
 │       │   └── com/example/devopsdemo/
 │       │       └── controller/
 │       │           └── DemoController.java
 │       └── resources/
 │           └── application.properties
 ├── Dockerfile
 ├── pom.xml
 └── docker-compose.yml (you will create this)
```

### 3. Verify Your Dockerfile

Your Dockerfile from Lesson 2 should look like this:

```dockerfile
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
ENV PORT=8080
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

### 4. Important: Check Your .dockerignore File

If you have a `.dockerignore` file from Lesson 2, **verify it does NOT block the JAR file**.

Your `.dockerignore` should look like this:

```
.git
.gitignore
.idea/
.vscode/
target/classes/
target/test-classes/
```

**If you see `target/` (without subdirectories), remove it.** We need the JAR file for Docker to copy.

---

## Why Do We Need Docker Compose?

Until now, you have used `docker run` to start a single container. Real applications usually need multiple services:

- a backend application (Spring Boot)
- a database (PostgreSQL, MySQL, etc.)

Managing these containers one by one becomes difficult:
- you must start them in the correct order
- you must connect them to the same network
- you must pass environment variables correctly
- you must remember many commands

**Docker Compose solves this** by defining the entire system in **one file** and starting everything with **one command**.

### Real-World Scenario

```
Traditional Approach (Manual):
1. Create network: docker network create demo-network
2. Start database: docker run -d --network demo-network postgres...
3. Wait for database to be ready
4. Start application: docker run -d --network demo-network app...
5. Configure environment variables for both
6. Troubleshoot networking issues
7. Remember all these commands for next time

Docker Compose Approach:
1. Create docker-compose.yml
2. Run: docker compose up -d
3. Done!
```

---

## What Is Docker Compose?

Docker Compose allows you to:
- define multiple containers (services)
- describe how they connect to each other
- run them together as one application

Everything is defined in a file called `docker-compose.yml`.

When you run Docker Compose:
- Docker creates a private network
- containers can talk to each other using service names
- services start together in a controlled way

---

## Our Demo Scenario

You will run **two containers**:

1. **Spring Boot application**
   - same DevOps demo project
   - exposes `/hello` endpoint
2. **PostgreSQL database**
   - runs as a separate container
   - used only to demonstrate dependency

There will be:
- no database entities
- no repositories
- no CRUD APIs

The database exists to show **multi-container orchestration**, not data access.

### Architecture Overview

```
┌─────────────────────────────────────────────┐
│         Docker Compose Environment          │
│                                             │
│  ┌─────────────────┐   ┌─────────────────┐ │
│  │  Spring Boot    │   │   PostgreSQL    │ │
│  │  Application    │──>│   Database      │ │
│  │  (port 8080)    │   │   (port 5432)   │ │
│  └─────────────────┘   └─────────────────┘ │
│                                             │
└─────────────────────────────────────────────┘
         │                      │
         └──────────────────────┘
         Accessible from localhost
```

---

## Step 1: Add Required Dependencies

Even though we won't use the database for data operations yet, Spring Boot needs the proper drivers and libraries to establish a connection.

Open your `pom.xml` and add these dependencies inside the `<dependencies>` section:

```xml
<!-- PostgreSQL Driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

### Why do we need these?

**PostgreSQL Driver:**
- Allows Java applications to connect to PostgreSQL databases
- Without it, Spring Boot cannot communicate with PostgreSQL

**Spring Data JPA:**
- Provides database abstraction layer
- Required for the JPA configuration properties to work
- Even though we have no entities yet, having this dependency prevents configuration errors

### Verify Dependencies

After adding these dependencies, your `<dependencies>` section should look similar to this:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

---

## Step 2: Configure Application Properties

Open `src/main/resources/application.properties` and add the following configuration:

```properties
# Server configuration
server.port=8080

# Database configuration (from Docker Compose environment variables)
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}

# JPA configuration - prevent startup failure when no entities exist
spring.jpa.hibernate.ddl-auto=none
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.show-sql=false
```

### Understanding the Configuration

**Environment Variables:**

```properties
spring.datasource.url=${SPRING_DATASOURCE_URL}
```

This syntax means:
- Read the value from the `SPRING_DATASOURCE_URL` environment variable
- When running with Docker Compose, these variables are set automatically
- If the variable is not set, the application will fail to start (this ensures we only run with Docker Compose)

**JPA Configuration:**

```properties
spring.jpa.hibernate.ddl-auto=none
```
- Tells Hibernate **not** to create, update, or validate database schema
- Essential because we have no database entities yet
- Prevents startup errors

```properties
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
```
- Explicitly tells Hibernate we're using PostgreSQL
- Helps with SQL optimization

```properties
spring.jpa.show-sql=false
```
- Disables SQL logging in console
- Keeps logs clean during this lesson

---

## Step 3: Build the Project

Before creating Docker images, you must build the Spring Boot JAR file.

Run this command in your project directory:

```bash
mvn clean package -DskipTests
```

### Understanding the Command

- `clean` - removes previous build artifacts
- `package` - compiles code and creates JAR file
- `-DskipTests` - skips running tests to speed up build

### Verify the Build

```bash
ls -lh target/*.jar
```

**Expected output:**

```
-rw-r--r--  1 user  staff    25M Jan 22 10:30 target/devops-demo-0.0.1-SNAPSHOT.jar
```

---

## Step 4: Create docker-compose.yml

Create a file named `docker-compose.yml` in the project root (same level as `pom.xml`).

```yaml
version: "3.9"

services:
  app:
    build: .
    container_name: devops-demo-app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/demo_db
      SPRING_DATASOURCE_USERNAME: demo_user
      SPRING_DATASOURCE_PASSWORD: demo_password
    depends_on:
      - db

  db:
    image: postgres:15
    container_name: devops-demo-db
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: demo_db
      POSTGRES_USER: demo_user
      POSTGRES_PASSWORD: demo_password
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

---

## Step 5: Understand the Compose File

Let's break down each section:

### Version Declaration

```yaml
version: "3.9"
```

Specifies the Docker Compose file format version.

### Services Section

```yaml
services:
  app:
    build: .              # Build image from Dockerfile in current directory
    container_name: devops-demo-app
    ports:
      - "8080:8080"       # Map host port 8080 to container port 8080
```

**Key Points:**
- `build: .` tells Docker Compose to use your existing Dockerfile
- Container name is set explicitly for easy identification
- Port mapping allows access from your host machine

### Environment Variables

```yaml
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/demo_db
      SPRING_DATASOURCE_USERNAME: demo_user
      SPRING_DATASOURCE_PASSWORD: demo_password
```

**Critical Detail:**

Notice `jdbc:postgresql://db:5432/demo_db`:
- `db` is the **service name** of the PostgreSQL container (defined below)
- Docker Compose creates a network where services can find each other by name
- This replaces `localhost` when containers talk to each other
- From inside the app container, `db` resolves to the database container's IP

### Service Name Resolution

```
Inside app container:
  ping db → resolves to PostgreSQL container
  ping localhost → resolves to app container itself
  
From your computer:
  localhost:8080 → reaches app container
  localhost:5432 → reaches database container
```

### Dependency Management

```yaml
    depends_on:
      - db
```

**What this does:**
- Ensures database starts **before** the application
- However, it only waits for the container to start, not for PostgreSQL to be ready
- Spring Boot will retry connection automatically, so this usually works fine

### Database Service

```yaml
  db:
    image: postgres:15    # Use official PostgreSQL 15 image from Docker Hub
    container_name: devops-demo-db
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: demo_db          # Database name to create
      POSTGRES_USER: demo_user      # Database user to create
      POSTGRES_PASSWORD: demo_password  # Password for the user
```

**Key Points:**
- Uses official PostgreSQL image (no need to build)
- Environment variables are PostgreSQL-specific
- These must match the Spring Boot datasource configuration

### Data Persistence

```yaml
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

**Why volumes are important:**
- Without volumes, database data is lost when container stops
- Named volume `postgres-data` persists data on the host machine
- Data survives container restarts, rebuilds, and even `docker compose down`
- Only removed with `docker compose down -v`

---

## Step 6: Run the Application with Docker Compose

Now start both containers with a single command:

```bash
docker compose up -d
```

### Understanding the Command

- `up` - creates and starts containers
- `-d` - detached mode (runs in background)

### What Happens When You Run This?

```
1. Docker Compose reads docker-compose.yml
2. Creates a network: devops-demo_default
3. Pulls postgres:15 image (if not present locally)
4. Starts PostgreSQL container
5. Builds Spring Boot image from your Dockerfile
6. Starts Spring Boot container
7. Application connects to database using service name "db"
```

### Expected Output

```
[+] Running 3/3
 ✔ Network devops-demo_default        Created    0.1s
 ✔ Container devops-demo-db           Started    0.5s
 ✔ Container devops-demo-app          Started    1.2s
```

---

## Step 7: Verify Containers Are Running

Check the status of your containers:

```bash
docker compose ps
```

### Expected Output

```
NAME                IMAGE               COMMAND                  STATUS              PORTS
devops-demo-app     devops-demo-app     "java -jar app.jar"      Up 10 seconds       0.0.0.0:8080->8080/tcp
devops-demo-db      postgres:15         "docker-entrypoint.s…"   Up 15 seconds       0.0.0.0:5432->5432/tcp
```

### What to Look For

- **STATUS** should be `Up` for both containers
- **PORTS** shows the port mappings
- If status shows `Exited` or `Restarting`, check logs (next step)

---

## Step 8: Check Container Logs

View logs to ensure everything started correctly:

### Application Logs

```bash
docker compose logs app
```

**Look for these success indicators:**

```
Started DevopsDemoApplication in 3.456 seconds (process running for 4.123)
Tomcat started on port 8080
```

### Database Logs

```bash
docker compose logs db
```

**Look for:**

```
database system is ready to accept connections
```

### Follow Logs in Real-Time

```bash
docker compose logs -f app
```

**What this does:**
- `-f` flag means "follow" (like `tail -f`)
- Shows new log lines as they appear
- Useful for debugging
- Press `Ctrl+C` to stop following (doesn't stop containers)

### View Logs for All Services

```bash
docker compose logs
```

Shows logs for both app and database.

---

## Step 9: Test the Application Endpoint

Verify the application is working correctly.

### Using a Web Browser

Open your browser and navigate to:

```
http://localhost:8080/hello
```

### Using curl

```bash
curl http://localhost:8080/hello
```

### Expected Response

```
DevOps demo application is running!
```

### What This Proves

- ✅ Spring Boot application is running
- ✅ Application is accessible from your host machine
- ✅ Port mapping is working correctly
- ✅ Application started successfully with database connection configured

---

## Important Clarification About the Database

**The database is running, but no API uses it yet.**

This is intentional and mirrors real-world workflows:

### Why This Approach?

1. **Infrastructure First**: Set up the database infrastructure
2. **Verify Connection**: Ensure the application can connect
3. **Add Features Later**: Implement database entities and repositories in future lessons

### What's Happening Behind the Scenes?

- Spring Boot creates a database connection pool
- Connection is established to PostgreSQL
- No queries are executed because there are no entities
- This proves the multi-container setup works

### Real-World Parallel

This is similar to DevOps practices:
- Set up infrastructure (databases, networks, services)
- Verify connectivity and configuration
- Then add application features incrementally

---

## How Containers Start and Why You Do Not See a JDK Image

You may wonder why PostgreSQL appears in Docker Compose, but the JDK does not.

### In docker-compose.yml

```yaml
services:
  app:
    build: .              # References Dockerfile

  db:
    image: postgres:15    # Uses ready-made image
```

PostgreSQL uses a **ready-made image** from Docker Hub, so it appears directly in docker-compose.yml.

### In Dockerfile

```dockerfile
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
ENV PORT=8080
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

The JDK is defined **inside the Dockerfile** (FROM eclipse-temurin:17-jdk-alpine), so it doesn't appear in Docker Compose.

### Key Difference

| Aspect | Dockerfile | Docker Compose |
|--------|-----------|----------------|
| Purpose | Defines **what is inside** a container | Defines **how containers work together** |
| Contains | Base image, application code, dependencies | Service definitions, networks, volumes |
| Used For | Building images | Running multi-container applications |

### The Relationship

```
docker-compose.yml (orchestration layer)
    │
    ├── app service → build: .
    │       │
    │       └── Dockerfile (image definition)
    │               │
    │               └── FROM eclipse-temurin:17-jdk-alpine (base image with JDK)
    │
    └── db service → image: postgres:15 (ready-made image)
```

---

## Container Startup Flow

Here's what happens when you run `docker compose up -d`:

### Step-by-Step Process

```
1. Parse Configuration
   └─> Docker Compose reads docker-compose.yml

2. Network Setup
   └─> Creates network: devops-demo_default

3. Pull Images
   └─> Pulls postgres:15 from Docker Hub (if not present)

4. Start Database
   └─> Starts PostgreSQL container
   └─> Initializes database with POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
   └─> Database becomes ready to accept connections

5. Build Application Image
   └─> Reads your Dockerfile
   └─> Uses eclipse-temurin:17-jdk-alpine as base
   └─> Copies JAR file into image
   └─> Creates devops-demo-app image

6. Start Application
   └─> Waits for database due to depends_on
   └─> Starts Spring Boot container
   └─> Spring Boot reads environment variables
   └─> Connects to database using service name "db"
   └─> Application becomes ready

7. Final State
   └─> Both containers running
   └─> Connected via Docker network
   └─> Accessible from localhost
```

---

## Useful Docker Compose Commands

Here are essential commands for managing your multi-container application:

### Starting and Stopping

```bash
# Start all services (build if needed)
docker compose up -d

# Start services without rebuilding images
docker compose start

# Stop all services (containers remain)
docker compose stop

# Restart services
docker compose restart

# Restart a specific service
docker compose restart app
```

### Rebuilding and Cleaning

```bash
# Rebuild images and restart
docker compose up -d --build

# Force rebuild (ignore cache)
docker compose build --no-cache

# Stop and remove containers, networks
docker compose down

# Stop and remove containers, networks, and volumes
docker compose down -v
```

### Monitoring and Debugging

```bash
# View running services
docker compose ps

# View all services (including stopped)
docker compose ps -a

# View logs
docker compose logs

# View logs for specific service
docker compose logs app
docker compose logs db

# Follow logs in real-time
docker compose logs -f

# View last 50 lines
docker compose logs --tail=50 app

# View resource usage
docker compose top
```

### Executing Commands

```bash
# Open bash shell in running container
docker compose exec app sh

# Run a command in running container
docker compose exec app ls -la

# Connect to PostgreSQL
docker compose exec db psql -U demo_user -d demo_db
```

---

## Activity: Update Database Configuration

Practice modifying the Docker Compose configuration:

### Task

1. Change the database name from `demo_db` to `demo_db_v2`
2. Update the application environment variable to match
3. Restart the containers
4. Verify everything works

### Step-by-Step Instructions

**1. Edit docker-compose.yml**

Change these lines:

```yaml
services:
  app:
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/demo_db_v2  # Changed

  db:
    environment:
      POSTGRES_DB: demo_db_v2  # Changed
```

**2. Stop and Remove Old Containers**

```bash
docker compose down
```

**Why `down` instead of `stop`?**
- `stop` only stops containers
- `down` stops and removes containers
- This ensures a clean restart with new configuration

**3. Start with New Configuration**

```bash
docker compose up -d
```

**4. Verify Containers Are Running**

```bash
docker compose ps
```

Both should show `Up` status.

**5. Check Application Logs**

```bash
docker compose logs app | grep "Started"
```

Look for: `Started DevopsDemoApplication in X.XXX seconds`

**6. Test the Endpoint**

```bash
curl http://localhost:8080/hello
```

**Expected:** `DevOps demo application is running!`

**7. Verify Database Name**

```bash
docker compose exec db psql -U demo_user -l
```

You should see `demo_db_v2` in the list of databases.

### What Did You Learn?

- ✅ How to modify Docker Compose configuration
- ✅ How to safely restart services with new settings
- ✅ How to verify configuration changes
- ✅ The importance of keeping environment variables synchronized

---

## Troubleshooting Common Issues

### Issue 1: "Failed to configure a DataSource"

**Full Error Message:**

```
Failed to configure a DataSource: 'url' attribute is not specified
```

**Cause:**
- PostgreSQL driver is missing from pom.xml
- Or JPA dependency is missing
- Or environment variables are not being passed correctly

**Solution:**

1. Verify both dependencies are in pom.xml:

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

2. Rebuild the project:

```bash
mvn clean package -DskipTests
```

3. Rebuild and restart Docker Compose:

```bash
docker compose down
docker compose up -d --build
```

---

### Issue 2: "JAR file not found" during Docker build

**Full Error Message:**

```
COPY failed: file not found in build context or excluded by .dockerignore
```

**Cause:**
- JAR file was not built
- Or .dockerignore is excluding the target folder incorrectly

**Solution:**

1. Build the JAR:

```bash
mvn clean package -DskipTests
```

2. Verify JAR exists:

```bash
ls -lh target/*.jar
```

3. Check your Dockerfile uses the correct pattern:

```dockerfile
COPY target/*.jar app.jar
```

4. If you have a `.dockerignore` file, make sure it doesn't exclude the JAR:

```
# .dockerignore should NOT have:
# target/

# It's OK to have:
.git
.gitignore
.idea/
.vscode/
target/classes/
target/test-classes/
```

5. Rebuild Docker images:

```bash
docker compose up -d --build
```

---

### Issue 3: Port 8080 already in use

**Full Error Message:**

```
Bind for 0.0.0.0:8080 failed: port is already allocated
```

**Cause:**
- Another application is using port 8080
- Previous container is still running

**Solution:**

**Option 1:** Stop the conflicting application

Check what's using the port:

```bash
# On macOS/Linux
lsof -i :8080

# On Windows
netstat -ano | findstr :8080
```

Stop the application or container using that port.

**Option 2:** Change the port in docker-compose.yml

```yaml
services:
  app:
    ports:
      - "8081:8080"  # Use port 8081 on host instead
```

Then access: `http://localhost:8081/hello`

**Option 3:** Stop all Docker containers

```bash
docker ps -q | xargs docker stop
```

---

### Issue 4: Database connection refused

**Full Error Message:**

```
Connection to localhost:5432 refused
```

**Cause:**
- Using `localhost` instead of service name `db`
- Database container is not running

**Solution:**

1. Verify you're using service name in docker-compose.yml:

```yaml
environment:
  SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/demo_db  # "db", not "localhost"
```

2. Check database is running:

```bash
docker compose ps
```

3. Check database logs:

```bash
docker compose logs db
```

Look for: `database system is ready to accept connections`

4. If database failed to start, restart it:

```bash
docker compose restart db
```

---

### Issue 5: Application keeps restarting

**Symptoms:**

```bash
docker compose ps
```

Shows status: `Restarting (1) 10 seconds ago`

**Diagnosis:**

Check logs:

```bash
docker compose logs app
```

**Common Causes and Solutions:**

**Cause 1: Missing dependency**

Error: `ClassNotFoundException: org.postgresql.Driver`

Solution: Add PostgreSQL driver to pom.xml and rebuild.

**Cause 2: Database connection timeout**

Error: `Connection refused` or `Connection timeout`

Solution: 
- Database may be slow to start
- Wait 30 seconds and check again
- Check database logs: `docker compose logs db`

**Cause 3: Application.properties misconfiguration**

Error: `Could not resolve placeholder 'SPRING_DATASOURCE_URL'`

Solution: 
- Verify environment variables are set in docker-compose.yml
- Check spelling matches exactly: `SPRING_DATASOURCE_URL`

**Cause 4: Wrong JAR filename**

Error: `Error: Unable to access jarfile app.jar`

Solution: 
- Verify JAR was built: `ls target/*.jar`
- Your Dockerfile uses `COPY target/*.jar app.jar` which should work

---

### Issue 6: "Cannot connect to Docker daemon"

**Full Error Message:**

```
Cannot connect to the Docker daemon. Is the docker daemon running?
```

**Solution:**

1. Start Docker Desktop (on Mac/Windows)
2. Or start Docker service (on Linux):

```bash
sudo systemctl start docker
```

---

### Issue 7: Old data persists after changing configuration

**Symptom:**
- Changed database name but old data still appears

**Cause:**
- Docker volumes persist data
- Old volume is still attached

**Solution:**

```bash
# Stop containers and remove volumes
docker compose down -v

# Start with clean state
docker compose up -d
```

**Warning:** This deletes all database data!

---

## Clean Up Commands

When you're done with the lesson:

### Option 1: Stop Containers (Keep Everything)

```bash
docker compose stop
```

**What this does:**
- Stops containers
- Keeps containers, images, volumes
- Quick restart with `docker compose start`

### Option 2: Remove Containers (Keep Images and Volumes)

```bash
docker compose down
```

**What this does:**
- Stops and removes containers
- Removes network
- Keeps images and volumes (data persists)

### Option 3: Remove Everything Except Images

```bash
docker compose down -v
```

**What this does:**
- Stops and removes containers
- Removes network
- Removes volumes (database data is deleted)
- Keeps images (faster next startup)

### Option 4: Complete Cleanup

```bash
# Remove containers and volumes
docker compose down -v

# Remove images
docker rmi devops-demo-app
docker rmi postgres:15

# Remove all unused Docker resources
docker system prune -a
```

**Warning:** `docker system prune -a` removes all unused Docker resources system-wide, not just for this project.

---

## Key Takeaways

### 1. Docker Compose Orchestrates Multiple Containers

- One YAML file defines the entire system
- One command starts everything
- Simplifies complex application architectures

### 2. Dockerfile vs Docker Compose

| Aspect | Dockerfile | Docker Compose |
|--------|-----------|----------------|
| Defines | What is **inside** a container | How containers **work together** |
| Contains | Base image, code, dependencies | Services, networks, volumes |
| Scope | Single image | Multiple containers |
| Example | JDK, JAR file, runtime | App + Database + Network |

### 3. Service Names Enable Container Communication

- Containers use service names instead of localhost
- `db` in the connection URL resolves to the database container
- Docker Compose creates a private network automatically

### 4. Environment Variables Configure Containers

- Pass configuration without hardcoding
- Different values for different environments
- Can be overridden at runtime

### 5. Data Persistence Requires Volumes

- Without volumes, data is lost when containers stop
- Named volumes survive container restarts
- Removed only with `-v` flag

### 6. Separation of Infrastructure and Application

- Set up infrastructure first (databases, networks)
- Verify connectivity
- Add features incrementally
- Mirrors real-world DevOps practices
