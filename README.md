# Go URL Shortener

## Overview

This is a small URL shortener service written in Go. It uses:
- `gin` for the HTTP API
- `go-redis/redis` as a Redis client to store short→long URL mappings

The service listens on port `9808` and uses Redis (default port `6379`) as the backing store.

## High-level flow / file map

- [main.go](main.go) — sets up the HTTP routes, initializes the store, and starts the server on port `:9808`.
- [handler/handlers.go](handler/handlers.go) — request handlers:
  - `CreateShortUrl` receives `POST /create-short-url`, validates JSON, calls the shortener and store.
  - `HandleShortUrlRedirect` handles `GET /:shortUrl` and issues the redirect to the original URL.
- [shortener/shorturl_generator.go](shortener/shorturl_generator.go) — generates a short token from the long URL + user ID.
- [store/store_service.go](store/store_service.go) — initializes a Redis client and exposes `InitializeStore()`, `SaveUrlMapping()` and `RetrieveInitialUrl()`.

## Prerequisites

- Go (1.20+ recommended)
- Docker (to run Redis quickly)
- Git (optional)

## Quick start (recommended)

1. Start Redis with Docker (map host 6379 to container 6379):

```bash
# pull and start Redis
docker run -d --name redis -p 6379:6379 redis
```

2. Connect to the running Redis container to verify or inspect keys (your requested step):

```bash
# open an interactive redis-cli shell inside the container
docker exec -it redis redis-cli

# inside redis-cli, you can run PING
> PING
PONG

# list keys (if any)
> KEYS *
```

Exit the `redis-cli` shell with `CTRL+C`.

3. Run the Go service (binds to localhost:9808):

```powershell
# from project root
go run main.go
```

You should see the server start and the `Redis started successfully` message.

4. Create a short URL (PowerShell-safe example):

```powershell
# PowerShell (escape JSON) using curl.exe
curl.exe -X POST -H "Content-Type: application/json" -d "{\"long_url\":\"https://example.com/very/long/path\",\"user_id\":\"your-user-id\"}" http://localhost:9808/create-short-url

# Or use PowerShell native cmdlet
Invoke-RestMethod -Uri "http://localhost:9808/create-short-url" -Method Post -Body (@{ long_url="https://example.com/very/long/path"; user_id="your-user-id" } | ConvertTo-Json) -ContentType "application/json"
```

The API will return a JSON response containing the shortened URL, e.g. `http://localhost:9808/abcd1234`.

5. Open the short URL in a browser or request via curl to get redirected:

```bash
curl -v http://localhost:9808/abcd1234
```

## Redis and ports explained

- Redis runs (by default) on port `6379`. The app connects to `localhost:6379` inside the host environment.
- The Go server listens on `:9808` (HTTP). When you run `go run main.go` it exposes port `9808` on localhost.
- If you run the Go app inside a Docker container, be sure to map the container port `9808` to the host, e.g. `-p 9808:9808`, and ensure the container can reach the Redis container (bridge network or link them).

## Troubleshooting

- "invalid character" JSON errors when using `curl` from PowerShell: PowerShell interprets quotes and braces. Use the `curl.exe` binary, use `curl --%` (stop parsing) or use `Invoke-RestMethod` as shown above.
- Redis connection errors: ensure Redis container is running and reachable at `localhost:6379`. You can check with `docker ps` and `docker exec -it redis redis-cli PING`.

## Development notes (how code interacts)

- `main.go` registers two routes and calls `store.InitializeStore()` early on. The order matters: `InitializeStore()` creates and pings a Redis client that is used by the store functions.
- `CreateShortUrl` in [handler/handlers.go](handler/handlers.go) uses:
  - `shortener.GenerateShortLink(longUrl, userId)` to create a stable short token
  - `store.SaveUrlMapping(shortUrl, longUrl, userId)` to persist the mapping in Redis
- `HandleShortUrlRedirect` in [handler/handlers.go](handler/handlers.go) uses `store.RetrieveInitialUrl(shortUrl)` and performs an HTTP redirect to the original URL.

## Files to inspect

- [main.go](main.go)
- [handler/handlers.go](handler/handlers.go)
- [shortener/shorturl_generator.go](shortener/shorturl_generator.go)
- [store/store_service.go](store/store_service.go)

## Next steps / Optional

- Add a `.gitignore` to ignore build artifacts and editor settings.
- Add a `.gitattributes` to normalize line endings if collaborating across OSes.
- Consider adding Dockerfiles / docker-compose to containerize both the service and Redis for easier deployment.