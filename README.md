# Social Media Backend

A REST API for registering users, publishing posts, and voting. FastAPI validates requests and exposes interactive OpenAPI documentation. PostgreSQL stores users, posts, and votes, and JWT bearer tokens authenticate post and vote requests.

## Features and architecture

- Register users with an email and password. Passwords are hashed with bcrypt, and user responses omit the hash.
- Create, list, read, update, and delete posts. Lists support title search and offset pagination. Only a post's owner can update or delete it.
- Add or remove a vote on a post. PostgreSQL enforces one vote per user and post. Read endpoints return post details and aggregate vote counts.
- Run integration tests with pytest and FastAPI's TestClient against a separate PostgreSQL database.

Requests pass through FastAPI routers and Pydantic schemas. SQLAlchemy manages database sessions and relationships. Alembic migrations create and update the PostgreSQL schema. Users own posts, and a vote links a user to a post through a composite primary key. The API runs under Uvicorn with Python 3.12 in Docker.

## Run locally with Docker

From the repository root, with Docker Compose and OpenSSL available, create a private local environment file:

```sh
cp .env.example .env
printf '\nDATABASE_PASSWORD=%s\nSECRET_KEY=%s\n' "$(openssl rand -hex 24)" "$(openssl rand -hex 32)" >> .env
docker compose -f docker-compose-dev.yml up -d --build
docker compose -f docker-compose-dev.yml exec api python -m alembic upgrade head
```

Open [Swagger UI](http://127.0.0.1:8000/docs) or [ReDoc](http://127.0.0.1:8000/redoc). PostgreSQL runs on the Compose network and persists data in a named volume. Keep the same database credentials when reusing that volume. Stop the services with `docker compose -f docker-compose-dev.yml down`.

## Run with local Python

Use Python 3.12 and a running PostgreSQL instance. Create a database named `fastapi` (or use the name configured in `.env`) and grant the configured user access to it. Create `.env` as above, then set the connection fields to match the local database.

```sh
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m alembic upgrade head
python -m uvicorn app.main:app --reload
```

| Variable | Purpose |
| --- | --- |
| `DATABASE_HOSTNAME`, `DATABASE_PORT` | PostgreSQL address. Development Compose supplies `postgres:5432` inside its network. |
| `DATABASE_USERNAME`, `DATABASE_PASSWORD`, `DATABASE_NAME` | PostgreSQL credentials and database name. |
| `SECRET_KEY` | Private JWT signing key. Use a fresh random value for each environment. |
| `ALGORITHM`, `ACCESS_TOKEN_EXPIRE_MINUTES` | JWT signing algorithm and access-token lifetime. The example uses HS256 and 30 minutes. |

## API and authentication

| Method | Path | Access | Behavior |
| --- | --- | --- | --- |
| GET | `/` | Public | Returns the app's greeting. |
| POST | `/users/` | Public | Registers a user from JSON containing `email` and `password`. |
| GET | `/users/{id}` | Public | Returns the user's ID, email, and creation time. |
| POST | `/login` | Public | Accepts form-encoded `username` (the user's email) and `password`. Returns `access_token` and `token_type`. |
| GET | `/posts/` | Bearer token | Lists posts and vote counts. Accepts `limit`, `skip`, and title `search`. |
| GET | `/posts/{id}` | Bearer token | Returns one post and its vote count. |
| POST | `/posts/` | Bearer token | Creates a post with `title`, `content`, and optional `published`. |
| PUT | `/posts/{id}` | Post owner | Updates a post. |
| DELETE | `/posts/{id}` | Post owner | Deletes a post and returns HTTP 204. |
| POST | `/vote/` | Bearer token | Adds (`dir: 1`) or removes (`dir: 0`) a vote for `post_id`. |

Register a local demo user, sign in, and use the returned access token:

```sh
curl -fsS http://127.0.0.1:8000/users/ \
  -H 'Content-Type: application/json' \
  -d '{"email":"reader@example.com","password":"local-demo-password"}'

curl -fsS http://127.0.0.1:8000/login \
  --data-urlencode 'username=reader@example.com' \
  --data-urlencode 'password=local-demo-password'

curl -fsS 'http://127.0.0.1:8000/posts/?limit=5&skip=0' \
  -H 'Authorization: Bearer REPLACE_WITH_ACCESS_TOKEN'
```

The login response has the shape `{"access_token":"<JWT>","token_type":"bearer"}`. Post reads return `Post` and `votes` fields. Post creation and updates return the post directly.

## Tests and CI

Tests connect to `${DATABASE_NAME}_test` and drop and recreate its application tables for each database fixture. Use a disposable test database.

With the development Compose stack running, create the test database once:

```sh
docker compose -f docker-compose-dev.yml exec -T postgres \
  sh -c 'createdb -U "$POSTGRES_USER" "${POSTGRES_DB}_test"'
docker compose -f docker-compose-dev.yml run --rm api python -m pytest
```

For local Python, create the corresponding disposable test database in PostgreSQL and run `python -m pytest` from the repository root. The suite covers signup, login, authorized post operations, ownership checks, and voting.

[GitHub Actions](.github/workflows/build-deploy.yml) runs the tests on pushes and pull requests using Python 3.12 and a disposable PostgreSQL 15 service. The legacy Heroku deploy job runs only when the workflow is manually dispatched on `main` and requires production credentials. This repository does not assert that a public instance is running.

## Known limitations

- The `published` field is stored and returned. The current read endpoints do not enforce draft visibility.
- Public user lookup exposes the email field, and CORS is configured broadly for development.
- Login issues an access token. The API does not implement refresh tokens, password reset, or rate limiting.
- Pagination uses offsets without a stable explicit sort order.
