I've been working with Docker containers for a while now, and one thing I learned the hard way is that data doesn't stick around when you restart containers, unless you set up persistent volumes properly. Let me walk you through how I deploy PostgreSQL in Docker with data that actually survives container restarts.

## Why Persistent Volumes Matter

The first time I ran PostgreSQL in Docker, I was excited about how quick it was to get up and running. Then I restarted the container and... all my data was gone. Turns out, containers are ephemeral by nature. Everything inside them disappears when they're removed. Not ideal for a database!

That's where Docker volumes come in. They let you store data outside the container's filesystem, so your database persists even when containers come and go.

## The Basic Setup

Here's the straightforward approach I use now. First, I create a dedicated Docker volume:

```bash
docker volume create postgres_data
```

This creates a named volume that Docker manages for you. You can see it with `docker volume ls`.

Then I run PostgreSQL with this volume mounted:

```bash
docker run -d \
  --name my_postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=myapp \
  -v postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

Let me break down what's happening here:

- `-d` runs it in detached mode (background)
- `--name my_postgres` gives it a friendly name
- The `-e` flags set environment variables for the initial user, password, and database
- `-v postgres_data:/var/lib/postgresql/data` is the magic, it mounts our volume to PostgreSQL's data directory
- `-p 5432:5432` maps the container's port to your host machine

## Using Docker Compose (My Preferred Method)

Honestly, I don't like typing long docker run commands. I always mess something up. These days I use Docker Compose instead. Here's my typical setup in a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: my_postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mysecretpassword
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

Now I just run:

```bash
docker compose up -d
```

Much cleaner, and I can commit this file to version control so the whole team uses the same setup.

## Testing the Persistence

Want to make sure it actually works? Try this:

Connect to your database:
```bash
docker exec -it my_postgres psql -U myuser -d myapp
```

Create a test table and add some data:
```sql
CREATE TABLE test_persistence (
    id SERIAL PRIMARY KEY,
    message TEXT
);

INSERT INTO test_persistence (message) VALUES ('This data should persist!');
```

Now stop and remove the container:
```bash
docker compose down
# or if you used docker run:
docker stop my_postgres
docker rm my_postgres
```

Start it back up:
```bash
docker compose up -d
```

Connect again and check:
```sql
SELECT * FROM test_persistence;
```

Your data should still be there. If it is, you've successfully set up persistent storage!

## A Few Tips from Experience

**Don't store secrets in plain text.** That hardcoded password in my examples? Fine for local development, but use environment files or secrets management for anything real. Create a `.env` file:

```
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mysecretpassword
POSTGRES_DB=myapp
```

Then reference it in docker-compose.yml with `env_file: .env` and add `.env` to your `.gitignore`.

**Back up your data.** Volumes are great, but they're not backups. I run regular dumps:

```bash
docker exec my_postgres pg_dump -U myuser myapp > backup.sql
```

**Check your volume location.** Want to know where Docker actually stores your volume? Run:

```bash
docker volume inspect postgres_data
```

On Linux, it's usually somewhere like `/var/lib/docker/volumes/postgres_data/_data`.

## Wrapping Up

Setting up PostgreSQL with Docker and persistent volumes isn't complicated once you understand the basics. The key is mounting that volume to `/var/lib/postgresql/data`, and Docker handles the rest. I've been using this setup for both local development and smaller production deployments, and it's been rock solid.

Give it a try, and your future self will thank you when that container restarts and your data is still there.