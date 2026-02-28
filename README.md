# Hosting-Website

This project is a lightweight GitHub-to-hosting platform that lets users deploy static sites from a Git repository with a single URL. It uses a small microservice architecture backed by Redis and S3-compatible object storage to clone, build, and serve user projects.

## Architecture Overview

The repository contains the following services (each runs independently):

- `frontend/` - React UI where users submit a GitHub repository URL to deploy.
- `upload-service/` - Receives the repo URL, clones it using `simple-git`, uploads repository files to object storage, and enqueues a build job into Redis.
- `deploy-service/` - Worker that blocks on the Redis build queue, downloads the uploaded repo, runs `npm install` and `npm run build`, and uploads the built `dist` artifacts back to object storage.
- `request-handler/` - Lightweight HTTP layer that serves built files from object storage by mapping incoming subdomains to deployed site IDs.

## Service Details

- Frontend (`frontend/`)
  - Built with React and a small UI component set.
  - Lets users enter a GitHub repo URL and trigger deployment by calling the backend `POST /deploy`.
  - Polls `GET /status?id=<id>` to show deployment progress and the deployed URL.

- Upload Service (`upload-service/`)
  - Endpoint: `POST /deploy`
  - Workflow:
    1. Generate a short `id` for the deployment.
    2. Clone the repository into a temporary `output/<id>` folder via `simple-git`.
    3. Use a recursive file walker to gather files and upload them to object storage (S3/R2) under a prefix like `output/<id>/...`.
    4. Push the `id` into the Redis `build-queue` and set `status[id] = "uploaded"`.

- Deploy Service (`deploy-service/`)
  - Worker that performs builds and publishes final assets.
  - Workflow:
    1. Block on Redis `brPop` from `build-queue` to obtain an `id`.
    2. Download all objects from object storage under `output/<id>/` to a local folder (streamed, with directories created as needed).
    3. Run `npm install` and `npm run build` (via `child_process`) inside `output/<id>`.
    4. Collect files from the produced `dist/` folder and upload them back to object storage under `dist/<id>/...`.
    5. Update `status[id] = "deployed"` in Redis.

- Request Handler (`request-handler/`)
  - Serves built sites by mapping incoming hostnames to an `id` (subdomain → id).
  - Retrieves objects from object storage at `dist/<id><path>` and returns them with an appropriate `Content-Type` header (HTML, CSS, JS).

## Typical Request Flow

1. User enters a GitHub repository URL in the `frontend` UI and clicks deploy.
2. `frontend` calls `upload-service` `POST /deploy` with the repo URL.
3. `upload-service` clones the repo, uploads repository files to object storage under `output/<id>/...`, sets `status[id] = "uploaded"`, and pushes `id` to Redis `build-queue`.
4. `deploy-service` blocks on `brPop(build-queue)` and receives the `id`.
5. `deploy-service` downloads `output/<id>/...`, runs `npm install && npm run build`, and uploads build artifacts to `dist/<id>/...` in object storage.
6. `deploy-service` sets `status[id] = "deployed"`.
7. `frontend` polls `GET /status?id=<id>` and, once it sees `deployed`, shows the final URL (for example `http://<id>.your-domain.com/index.html`).
8. Incoming requests to that hostname are routed to `request-handler`, which serves the static files from `dist/<id>/...`.

## Key Implementation Highlights

- Redis is used as a simple durable queue (`LPUSH` / `BRPOP`) and a hash store for deployment status (`HSET` / `HGET`).
- Object storage (S3-compatible, e.g. AWS S3 or Cloudflare R2) stores both the uploaded repository (`output/<id>/`) and the published artifacts (`dist/<id>/`).
- The deploy worker uses streaming downloads/uploads and `Promise.all` to handle files concurrently and efficiently.
- Builds run inside a child process (`npm install` + `npm run build`) with stdout/stderr forwarded to logs.
- The frontend uses a polling strategy with `setInterval` to check `GET /status?id=` until the service reports `deployed`.

## Tech Stack

- Node.js + Express for services
- Redis for queueing and status
- S3-compatible object storage for file persistence (credentials should be stored securely)
- React for the frontend
- simple-git for cloning repositories

## Running Locally (high level)

1. Start a Redis server.
2. Configure credentials for object storage via environment variables (do not hard-code keys).
3. From each service folder, install and start the service (example):

```bash
cd upload-service && npm install && npm run dev
cd deploy-service && npm install && npm run dev
cd request-handler && npm install && npm run dev
cd frontend && npm install && npm run dev
```

Adjust commands to match your `package.json` scripts.

## Security & Notes

- This repository currently contains hard-coded credentials in some example files. Do not keep keys in source — move them to environment variables or a secrets manager and rotate them.
- Running arbitrary repositories' `npm install` / `npm run build` is potentially unsafe; consider sandboxing builds (containers, VMs) and applying resource/time limits in production.

## Contributing

If you want to improve this project, consider:

- Replacing hard-coded credentials with environment configuration.
- Adding authentication and rate-limiting to the `upload-service`.
- Containerizing the build worker for isolation.

---

For file-level details, inspect the service entry points in `upload-service/src`, `deploy-service/src`, `request-handler/src`, and the frontend in `frontend/`.
