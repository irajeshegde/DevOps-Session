# db-service

FastAPI microservice backed by PostgreSQL, with CRUD endpoints for an `items` resource.

## Run locally with Docker

```
docker compose up --build
```

This starts Postgres on `5432` and the API on `8000`. Tables are created automatically on startup.

## Test it

```
curl http://localhost:8000/health

curl -X POST http://localhost:8000/items \
  -H "Content-Type: application/json" \
  -d '{"name":"widget","description":"a thing"}'

curl http://localhost:8000/items
```

Interactive API docs: http://localhost:8000/docs

## Stop / reset

```
docker compose down        # stop, keep data
docker compose down -v     # stop and wipe the Postgres volume
```

## Configuration

The API reads `DATABASE_URL` (set in `docker-compose.yml`). Default format:

```
postgresql+psycopg2://<user>:<password>@<host>:5432/<db>
```
