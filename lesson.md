# Lesson 4.6: Cloud-Native Applications with Docker Compose.

## Learning Objectives

By the end of this lesson, you will be able to:

1. Explain why Docker Compose is needed for cloud-native applications  
2. Configure a Docker Compose file to run multiple containers  
3. Run and verify a Spring Boot application and database using Docker Compose  
4. Inspect running services and logs using Docker Compose commands  

---

## Prerequisites

Before starting this lesson, ensure you have:

### 1. Completed Previous Lessons

- Lesson 4.4: Docker Local (Dockerfile, building images)
- Lesson 4.5: Docker Hub (pushing/pulling images)

### 2. Your Spring Boot DevOps Demo Project Ready

Your project should have:
- A working Dockerfile (from Lesson 4.4)
- A `/hello` endpoint that returns "DevOps demo application is running!"
- Successfully builds with Maven

### 3. Important: Stop Any Running Containers

Stop containers from previous lessons to avoid port conflicts:

```bash
# Stop all running containers
docker ps -q | xargs docker stop

# Verify nothing is running
docker ps
```
### 4. Important: Stop WSL PostgreSQL (If Installed)

If you installed PostgreSQL on WSL for Java/Spring Boot lessons, stop it to avoid port 5432 conflict:

**For WSL users:**
```bash
sudo service postgresql stop
```

**For Mac users (Homebrew):**
```bash
brew services stop postgresql@16
```

**Verify PostgreSQL is stopped:**
```bash
# Should show no output or "not running"
sudo service postgresql status
```


### 5. Project Structure

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
 ├── .dockerignore
 ├── pom.xml
 └── docker-compose.yml (you will create this)
```

### 6. Verify Your Dockerfile

Your Dockerfile from Lesson 4.4 should look like this:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
ENV PORT=8080
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

### 7. Verify Your .dockerignore File

From Lesson 4.4, your `.dockerignore` should look like this (allows JAR file):

```
target/classes/
target/test-classes/
target/generated-sources/
target/maven-status/
.git
.gitignore
.idea/
.vscode/
```

**Important:** Do NOT use `target/` alone - this blocks the JAR file!

---

## Part 1: Why Docker Compose? (20 minutes)

### The Problem with Multiple Containers

Until now, you used `docker run` to start a single container. Real applications need multiple services:

- Backend application (Spring Boot)
- Database (PostgreSQL, MySQL)
- Cache (Redis)
- Message queue (RabbitMQ)

Managing these manually becomes difficult:

```bash
# Traditional approach (tedious!)
docker network create demo-network
docker run -d --network demo-network --name db postgres:15 ...
docker run -d --network demo-network --name app myapp:latest ...
# Remember all these commands!
# Repeat every time!
```

### Docker Compose Solution

**One file + One command:**

```bash
docker compose up -d
```

### Real-World Scenario

**Manual Approach:**
```
1. Create network: docker network create demo-network
2. Start database: docker run -d --network demo-network postgres...
3. Wait for database to be ready
4. Start application: docker run -d --network demo-network app...
5. Configure environment variables for both
6. Troubleshoot networking issues
7. Remember all these commands for next time
```

**Docker Compose Approach:**
```
1. Create docker-compose.yml
2. Run: docker compose up -d
3. Done!
```

### What Docker Compose Provides

- ✅ Define multiple containers in one file
- ✅ Describe how they connect
- ✅ Start everything with one command
- ✅ Automatic networking
- ✅ Easy to share and reproduce

---

## Part 2: Our Demo Scenario (10 minutes)

You will run **two containers**:

1. **Spring Boot application**
   - DevOps demo project
   - Exposes `/hello` endpoint
2. **PostgreSQL database**
   - Runs as a separate container
   - Demonstrates multi-container orchestration

**Important:** No database entities or CRUD APIs yet. Database exists to show orchestration.

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

## Part 3: Add Dependencies (15 minutes)

Even though we won't use the database for data operations yet, Spring Boot needs drivers to establish connection.

### Step 1: Add PostgreSQL and JPA Dependencies

Open `pom.xml` and add these inside `<dependencies>`:

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

### Why These Dependencies?

**PostgreSQL Driver:**
- Allows Java to connect to PostgreSQL databases
- Without it, Spring Boot cannot communicate with PostgreSQL

**Spring Data JPA:**
- Provides database abstraction layer
- Required for JPA configuration properties
- Even without entities, prevents configuration errors
- **Note:** We'll create entities and repositories in Lesson 4.11

### Verify Dependencies

Your `<dependencies>` section should include:

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
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## Part 4: Configure Application Properties (15 minutes)

### Step 1: Update application.properties

Open `src/main/resources/application.properties` and add:

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

- Reads value from environment variable
- Docker Compose sets these automatically
- If missing, application fails (ensures we use Docker Compose)

**JPA Configuration:**

```properties
spring.jpa.hibernate.ddl-auto=none
```
- Tells Hibernate NOT to create/update schema
- Essential because we have no entities yet
- Prevents startup errors

```properties
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
```
- Explicitly tells Hibernate we're using PostgreSQL

```properties
spring.jpa.show-sql=false
```
- Disables SQL logging
- Keeps logs clean

---

## Part 5: Build the Project (10 minutes)

Before creating Docker images, build the JAR file.

### Step 1: Build

```bash
mvn clean package -DskipTests
```

**Understanding the command:**
- `clean` - removes previous build artifacts
- `package` - compiles code and creates JAR
- `-DskipTests` - skips tests to speed up build

### Step 2: Verify the Build

```bash
ls -lh target/*.jar
```

**Expected output:**

```
-rw-r--r--  1 user  staff    25M Jan 22 10:30 target/devops-demo-0.0.1-SNAPSHOT.jar
```

---

## Part 6: Create docker-compose.yml (30 minutes)

### Step 1: Create the File


Create `docker-compose.yml` in project root (same level as `pom.xml`):

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

### Step 2: Understand Each Section

**Version Declaration:**
```yaml
version: "3.9"
```
Specifies Docker Compose file format version.

**App Service:**
```yaml
services:
  app:
    build: .              # Build from Dockerfile in current directory
    container_name: devops-demo-app
    ports:
      - "8080:8080"       # Map host:container ports
```

**Environment Variables:**
```yaml
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/demo_db
      SPRING_DATASOURCE_USERNAME: demo_user
      SPRING_DATASOURCE_PASSWORD: demo_password
```

**Critical:** Notice `jdbc:postgresql://db:5432/demo_db`
- `db` is the service name of PostgreSQL container
- Docker Compose creates network where services find each other by name
- From inside app container, `db` resolves to database container IP

**Service Name Resolution:**
```
Inside app container:
  ping db → resolves to PostgreSQL container
  
From your computer:
  localhost:8080 → reaches app container
  localhost:5432 → reaches database container
```

**Dependency Management:**
```yaml
    depends_on:
      - db
```
Ensures database starts before application.

**Database Service:**
```yaml
  db:
    image: postgres:15    # Official PostgreSQL image
    container_name: devops-demo-db
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: demo_db          # Database name to create
      POSTGRES_USER: demo_user      # User to create
      POSTGRES_PASSWORD: demo_password  # User password
```

**Data Persistence:**
```yaml
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

**Why volumes matter:**
- Without volumes, data lost when container stops
- Named volume persists data on host
- Survives container restarts and rebuilds
- Only removed with `docker compose down -v`

---

## Part 7: Run with Docker Compose (20 minutes)

### Step 1: Start All Services

```bash
docker compose up -d
```

**Understanding the command:**
- `up` - creates and starts containers
- `-d` - detached mode (runs in background)

### What Happens?

```
1. Docker Compose reads docker-compose.yml
2. Creates network: devops-demo_default
3. Pulls postgres:15 image (if not present)
4. Starts PostgreSQL container
5. Builds Spring Boot image from Dockerfile
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

### Step 2: Verify Containers Running

```bash
docker compose ps
```

**Expected output:**

```
NAME                IMAGE               COMMAND                  STATUS              PORTS
devops-demo-app     devops-demo-app     "java -jar app.jar"      Up 10 seconds       0.0.0.0:8080->8080/tcp
devops-demo-db      postgres:15         "docker-entrypoint.s…"   Up 15 seconds       0.0.0.0:5432->5432/tcp
```

**What to look for:**
- STATUS should be `Up` for both
- If `Exited` or `Restarting`, check logs

---

## Part 8: Check Logs and Test (20 minutes)

### Step 1: View Application Logs

```bash
docker compose logs app
```

**Look for success indicators:**

```
Started DevopsDemoApplication in 3.456 seconds
Tomcat started on port 8080
```

### Step 2: View Database Logs

```bash
docker compose logs db
```

**Look for:**

```
database system is ready to accept connections
```

### Step 3: Follow Logs in Real-Time

```bash
docker compose logs -f app
```

- `-f` flag means "follow"
- Press `Ctrl+C` to stop (doesn't stop containers)

### Step 4: Test the Application

**Using browser:**
```
http://localhost:8080/hello
```

**Using curl:**
```bash
curl http://localhost:8080/hello
```

**Expected response:**
```
DevOps demo application is running!
```

**What this proves:**
- ✅ Spring Boot running
- ✅ Accessible from host
- ✅ Port mapping works
- ✅ Database connection configured

---

## Part 9: Important Clarification (10 minutes)

### The Database is Running But Not Used Yet

**This is intentional and mirrors real-world workflows:**

1. **Infrastructure First:** Set up database infrastructure
2. **Verify Connection:** Ensure application can connect
3. **Add Features Later:** Implement entities and repositories in future lessons

### What's Happening?

- Spring Boot creates database connection pool
- Connection established to PostgreSQL
- No queries executed (no entities yet)
- Proves multi-container setup works

### Real-World Parallel

DevOps practice:
- Set up infrastructure (databases, networks, services)
- Verify connectivity and configuration
- Add application features incrementally

**Note:** In Lesson 4.11, you'll add database entities, repositories, and CRUD APIs.

---

## Part 10: Docker Compose Commands (15 minutes)

### Starting and Stopping

```bash
# Start all services
docker compose up -d

# Start without rebuilding
docker compose start

# Stop all services
docker compose stop

# Restart services
docker compose restart

# Restart specific service
docker compose restart app
```

### Rebuilding

```bash
# Rebuild and restart
docker compose up -d --build

# Force rebuild (no cache)
docker compose build --no-cache
```

### Cleaning Up

```bash
# Stop and remove containers, networks
docker compose down

# Also remove volumes (deletes data!)
docker compose down -v
```

### Monitoring

```bash
# View running services
docker compose ps

# View all services
docker compose ps -a

# View logs
docker compose logs

# View specific service logs
docker compose logs app
docker compose logs db

# Follow logs
docker compose logs -f

# Last 50 lines
docker compose logs --tail=50 app
```

### Executing Commands

```bash
# Open shell in container
docker compose exec app sh

# Run command in container
docker compose exec app ls -la

# Connect to PostgreSQL
docker compose exec db psql -U demo_user -d demo_db
```

---

## Part 11: Activity - Update Database Configuration (20 minutes)

Practice modifying Docker Compose configuration.

### Task

1. Change database name from `demo_db` to `demo_db_v2`
2. Update application environment variable
3. Restart containers
4. Verify everything works

### Instructions

**1. Edit docker-compose.yml**

```yaml
services:
  app:
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/demo_db_v2  # Changed

  db:
    environment:
      POSTGRES_DB: demo_db_v2  # Changed
```

**2. Stop and Remove**

```bash
docker compose down
```

**3. Rebuild and Start**

```bash
docker compose up -d --build
```

**Important:** Use `--build` flag to rebuild app image with changes!

**4. Verify Running**

```bash
docker compose ps
```

Both should show `Up` status.

**5. Check Logs**

```bash
docker compose logs app | grep "Started"
```

Look for: `Started DevopsDemoApplication`

**6. Test Endpoint**

```bash
curl http://localhost:8080/hello
```

Expected: `DevOps demo application is running!`

**7. Verify Database Name**

```bash
docker compose exec db psql -U demo_user -l
```

Should see `demo_db_v2` in list.

### What You Learned

- ✅ Modifying Docker Compose configuration
- ✅ Rebuilding images after changes
- ✅ Verifying configuration changes
- ✅ Keeping environment variables synchronized

---

## Part 12: Understanding Dockerfile vs Docker Compose (10 minutes)

### Why JDK Doesn't Appear in docker-compose.yml

**In docker-compose.yml:**
```yaml
services:
  app:
    build: .              # References Dockerfile

  db:
    image: postgres:15    # Uses ready-made image
```

PostgreSQL uses ready-made image, so it appears directly.

**In Dockerfile:**
```dockerfile
FROM eclipse-temurin:21-jdk-alpine
```

JDK is defined inside Dockerfile, so doesn't appear in Compose.

### Key Difference

| Aspect | Dockerfile | Docker Compose |
|--------|-----------|----------------|
| Purpose | What's **inside** container | How containers **work together** |
| Contains | Base image, code, dependencies | Services, networks, volumes |
| Scope | Single image | Multiple containers |

### The Relationship

```
docker-compose.yml (orchestration)
    │
    ├── app service → build: .
    │       │
    │       └── Dockerfile
    │               └── FROM eclipse-temurin:21-jdk-alpine
    │
    └── db service → image: postgres:15
```

---

## Troubleshooting

### Issue 1: "Failed to configure a DataSource"

**Cause:** Missing PostgreSQL or JPA dependency

**Solution:**
1. Verify both dependencies in pom.xml
2. Rebuild: `mvn clean package -DskipTests`
3. Restart: `docker compose up -d --build`

---

### Issue 2: "JAR file not found"

**Cause:** JAR not built or .dockerignore blocking it

**Solution:**
1. Build: `mvn clean package -DskipTests`
2. Verify: `ls -lh target/*.jar`
3. Check .dockerignore allows JAR (use `target/classes/` not `target/`)
4. Rebuild: `docker compose up -d --build`

---

### Issue 3: Port 8080 already in use

**Cause:** Another application using port or old container running

**Solution:**

**Option 1:** Stop old containers
```bash
docker ps -q | xargs docker stop
```

**Option 2:** Change port in docker-compose.yml
```yaml
ports:
  - "8081:8080"  # Use 8081 on host
```

---

### Issue 4: Database connection refused

**Cause:** Using `localhost` instead of service name `db`

**Solution:**
Verify docker-compose.yml uses:
```yaml
SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/demo_db
```

Not `localhost`!

---

### Issue 5: Application keeps restarting

**Diagnosis:**
```bash
docker compose logs app
```

**Common causes:**
- Missing dependency → Add to pom.xml
- Database not ready → Wait 30 seconds
- Wrong environment variables → Check spelling
- Wrong JAR filename → Verify build

---

## Clean Up

### Option 1: Stop (Keep Everything)

```bash
docker compose stop
```

Keeps containers, images, volumes. Quick restart with `start`.

### Option 2: Remove Containers

```bash
docker compose down
```

Removes containers and network. Keeps images and volumes.

### Option 3: Remove Everything

```bash
docker compose down -v
```

Removes containers, network, and volumes. Database data deleted!

### Option 4: Complete Cleanup

```bash
docker compose down -v
docker rmi devops-demo-app postgres:15
docker system prune -a
```

**Warning:** `prune -a` removes ALL unused Docker resources system-wide!

---

## Summary

### Key Takeaways

**1. Docker Compose Orchestrates Multiple Containers**
- One YAML file defines entire system
- One command starts everything
- Simplifies complex architectures

**2. Dockerfile vs Docker Compose**

| Aspect | Dockerfile | Docker Compose |
|--------|-----------|----------------|
| Defines | What's inside container | How containers work together |
| Scope | Single image | Multiple containers |
| Example | JDK, JAR, runtime | App + Database + Network |

**3. Service Names Enable Communication**
- Containers use service names (not localhost)
- `db` resolves to database container
- Docker Compose creates private network

**4. Environment Variables Configure Containers**
- Pass configuration without hardcoding
- Different values per environment
- Override at runtime

**5. Volumes Persist Data**
- Without volumes, data lost on stop
- Named volumes survive restarts
- Removed only with `-v` flag

**6. Infrastructure-First Approach**
- Set up infrastructure
- Verify connectivity
- Add features incrementally
- Mirrors real-world DevOps

---