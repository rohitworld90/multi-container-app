# multi-container-app

A multi-service Fibonacci calculator built to practice orchestrating several containers
as one application: React frontend, Express API, a Redis-backed worker for the actual
computation, Postgres for permanent storage, and Nginx routing it all.

> Built in 2020 while learning Docker and CI/CD. Kept as-is (aside from docs cleanup)
> as a snapshot of that work.

## Architecture

```
                       ┌──────────────┐
        browser ─────► │    Nginx     │
                       └──────┬───────┘
                     /        │ /api
              ┌──────▼─────┐ ┌▼───────────┐
              │   React    │ │  Express   │
              │   client   │ │   server   │
              └────────────┘ └─┬────────┬─┘
                               │        │
                       ┌───────▼──┐  ┌──▼───────┐
                       │ Postgres │  │  Redis   │
                       │ (seen    │  │ (values) │
                       │ indexes) │  └──▲───────┘
                       └──────────┘     │ pub/sub
                                  ┌─────▼──────┐
                                  │   Worker   │
                                  │ (computes  │
                                  │  fib(n))   │
                                  └────────────┘
```

- Submitting an index stores it in Postgres and publishes it to Redis.
- The **worker** subscribes to Redis, computes the Fibonacci value (deliberately
  recursive, to have something slow worth caching), and writes the result back.
- The client polls the API, which reads current values from Redis and seen indexes
  from Postgres.

## Services

| Directory | Role |
|---|---|
| `client/`  | React UI (Create React App), served by its own Nginx in production |
| `server/`  | Express API — reads/writes Postgres and Redis |
| `worker/`  | Redis subscriber that computes values |
| `nginx/`   | Front-door reverse proxy: `/` → client, `/api` → server |

Each service has a `Dockerfile.dev` (hot reload, mounted source) and a production
`Dockerfile`.

## Running it

```bash
docker compose up --build
```

Then open http://localhost:3050.

## CI/CD

`.travis.yml` runs the client test suite in a container on every push, then builds all
four production images and pushes them to Docker Hub. An AWS Elastic Beanstalk
multi-container deploy (`Dockerrun.aws.json`) was used originally and is kept for
reference with the deploy step disabled.

All configuration (Postgres/Redis hosts and credentials) is injected via environment
variables — see `server/keys.js` and `docker-compose.yml`.
