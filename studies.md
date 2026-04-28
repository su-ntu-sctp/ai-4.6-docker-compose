# 4.6 Self Studies

**Estimated Preparation Time:** 65 minutes

---

## Task 1 — Docker Compose (25 minutes)

Watch the following video on Docker Compose:

- 📹 [Docker Compose — https://www.youtube.com/watch?v=-7b_jG9RY78&t=51s]

While watching, refer to **lesson.md Part 1 and Part 6** and pay attention to:
- Why Docker Compose is used instead of running individual `docker run` commands
- How the `docker-compose.yml` file is structured
- How services communicate with each other using service names
- What `depends_on` does and why it matters

**Guiding Questions:**
1. What is the main advantage of defining all services in a single `docker-compose.yml` file?
2. How do containers in the same Docker Compose network find each other — and why can't they use `localhost`?
3. What does `depends_on` guarantee, and what does it NOT guarantee?

---

## Task 2 — Volumes and Data Persistence (20 minutes)

No video for this task. Refer to **lesson.md Part 6** (Data Persistence section) and **Part 9** (Cleaning Up commands), then answer the following:

1. What happens to your PostgreSQL data if you run `docker compose down` without any flags?
2. What happens to your PostgreSQL data if you run `docker compose down -v`?
3. Why is data persistence important in a real-world application?
4. In the `docker-compose.yml` from the lesson, where is the named volume defined and where is it used?

**Guiding Questions:**
1. What is the difference between a named volume and mounting a host directory?
2. When would you deliberately want to remove volumes — and when would you not?
3. What would happen if the database container had no volume defined and it crashed?

---

## Task 3 — Docker Compose Commands Practice (20 minutes)

No video for this task. Refer to **lesson.md Part 9** and practise the following commands using your devops-demo project:

1. Start all services in detached mode:
```bash
docker compose up -d
```

2. Check the status of all services:
```bash
docker compose ps
```

3. View logs for the app service:
```bash
docker compose logs app
```

4. Follow live logs and then stop following with `Ctrl+C`:
```bash
docker compose logs -f app
```

5. Stop all services without removing them:
```bash
docker compose stop
```

6. Start them again without rebuilding:
```bash
docker compose start
```

7. Stop and remove containers and networks:
```bash
docker compose down
```

**Guiding Questions:**
1. What is the difference between `docker compose stop` and `docker compose down`?
2. When would you use `docker compose up -d --build` instead of just `docker compose up -d`?
3. What does the `-f` flag do in `docker compose logs -f`?

---

## Active Engagement Strategies

- Pause the video whenever a `docker-compose.yml` section is shown and compare it to the one in lesson.md
- After watching, close the video and try to write a basic `docker-compose.yml` from memory with two services
- For Tasks 2 and 3, run each command and observe the output rather than just reading through them

---

## Additional Reading Material

- [Docker Compose Overview — Docker Docs](https://docs.docker.com/compose/)
- [Docker Compose File Reference — Docker Docs](https://docs.docker.com/compose/compose-file/)
- [Networking in Docker Compose — Docker Docs](https://docs.docker.com/compose/networking/)
- [Managing Data with Volumes — Docker Docs](https://docs.docker.com/storage/volumes/)