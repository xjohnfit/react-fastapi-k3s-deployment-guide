# React + FastAPI on k3s

A starting point for deploying a React (Vite) frontend and a FastAPI backend to a k3s cluster with GitHub Actions. It follows the same layout as `mern-k3s-deployment-guide`. The differences are the Python backend image, pytest in CI, and a single-domain Ingress that sends `/api` to FastAPI.

Status: the images and the `/api` routing have been tested locally (see [Test locally](#test-locally)), but nothing here has run against a real cluster yet. Expect to fix things on the first deploy.

Versions (2026-11): Python 3.14, Node 26 (LTS), nginx:alpine, kubectl 1.37, and the latest release of every GitHub Action, pinned by commit SHA. Python 3.15 is out, but the image installs wheels only (see [Supply-chain hardening](#supply-chain-hardening)) and PyYAML, which `uvicorn[standard]` pulls in, has no 3.15 wheel yet. Check before moving the Dockerfile and `test-backend` to a newer Python together: `docker run --rm -v "$PWD/backend/requirements.txt:/r.txt:ro" python:3.15-slim pip install --require-hashes --only-binary :all: -r /r.txt`.

## Layout

```text
backend/            FastAPI app, requirements.txt, Dockerfile
frontend/           Vite + React, Dockerfile, nginx.conf
kubernetes/         manifests with ${VAR} placeholders
.github/workflows/  deploy.yml
```

## How a request flows

The browser loads the React app from `https://DOMAIN/`. The app calls `/api/...` on the same domain, so there is no CORS setup. Traefik sends `/api` to the backend Service and `/` to the frontend Service.

The `/api` prefix is stripped before the request reaches FastAPI, by a Traefik `StripPrefix` middleware (`kubernetes/middleware-strip-api.yml`, attached through the Ingress's `router.middlewares` annotation). The backend runs with `--root-path /api` so FastAPI still knows its public prefix (the docs page loads `/api/openapi.json`). Both halves are needed. Uvicorn adds the root path back onto every request path, so if the Ingress forwarded `/api/health` unchanged, FastAPI would see `/api/api/health` and return 404.

## What your app must provide

- `backend/requirements.txt`: a lock file of the runtime dependencies, every package pinned with `--hash` (the Docker image installs it)
- `backend/requirements-dev.txt`: the same plus `pytest` (CI installs it). Generate both from `.in` files with `uv pip compile --universal --python-version 3.14 --generate-hashes`, or `pip-compile --generate-hashes`
- a frontend that builds without npm install scripts (`npm ci --ignore-scripts`)
- `GET /health` on the backend
- `DATABASE_URL` and `SECRET_KEY` read from environment variables
- frontend scripts named `lint`, `typecheck` and `build`
- `VITE_API_URL` read at build time, normally `/api`

## Setup steps

1. Copy `backend/Dockerfile`, `backend/.dockerignore`, `frontend/Dockerfile`, `frontend/.dockerignore`, `frontend/nginx.conf`, `kubernetes/`, `.trivyignore` and `.github/` (the workflow and `dependabot.yml`) into your project. If you add a manifest to `kubernetes/`, also add a `kubectl apply` line for it in the workflow's "Deploy to Kubernetes" step.
2. On the VPS, make the kubeconfig and base64 it:

   ```bash
   MY_IP=$(curl -s ifconfig.me)
   sudo cp /etc/rancher/k3s/k3s.yaml ~/k3s-config.yaml
   sudo chown $USER:$USER ~/k3s-config.yaml
   sed -i "s/127.0.0.1/$MY_IP/g" ~/k3s-config.yaml
   base64 -w 0 ~/k3s-config.yaml; echo
   rm ~/k3s-config.yaml
   ```

3. Check cert-manager is running: `kubectl get pods -n cert-manager`.
4. Add DNS A records for `DOMAIN`. The default ingress serves `DOMAIN` only; add a `www` rule to `kubernetes/ingress.yml` if you want it.
5. In the GitHub repo, Settings, Secrets and variables, Actions.

   Secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `KUBECONFIG`, `DATABASE_URL`, `SECRET_KEY`

   Variables: `APP_NAME`, `BACKEND_PORT` (8000), `HEALTH_CHECK_PATH` (/health), `DOMAIN`, `LETSENCRYPT_EMAIL`, `VITE_API_URL` (/api)
6. Create the `production` environment with a required reviewer, then confirm it saved:

   ```bash
   gh api -X PUT repos/OWNER/REPO/environments/production -f "reviewers[][type]=User" -F "reviewers[][id]=$(gh api user --jq .id)"
   gh api repos/OWNER/REPO/environments/production --jq '.protection_rules'
   ```

   An empty array means the gate did not save.
7. Push to `main`. The jobs run in this order: test-backend and test-frontend, trivy-fs-scan, build-and-push, scan-images, deploy.

## Supply-chain hardening

These pass SonarCloud's security rules for workflows and Dockerfiles:

- **Actions pinned to commit SHAs**, with the version in a comment (`uses: actions/checkout@<sha> # v7.0.1`). A tag can be moved to different code; a SHA can't. `.github/dependabot.yml` opens monthly PRs to bump the pins, the Docker base images and npm packages.
- **Secrets enter scripts through `env:`**, never as `${{ secrets.X }}` inside `run:`, so a secret's contents are never pasted into shell code.
- **Locked, wheel-only Python installs:** `pip install --require-hashes --only-binary :all:` installs only the exact files in the lock file and never runs a package's `setup.py`.
- **`npm ci --ignore-scripts`:** package install scripts don't run, in CI or in the image build.
- **Read-only site files:** nginx's config and `dist/` stay owned by root; the non-root nginx user can write only its cache and log directories.

And Trivy's Kubernetes checks (`KSV-*`):

- **Locked-down pods:** both run as UID/GID 10001 (the Dockerfiles' `USER 10001:10001` must match `runAsUser`/`runAsGroup`), with `readOnlyRootFilesystem: true`, seccomp `RuntimeDefault` and all capabilities dropped. The only writable paths are `emptyDir` volumes: `/tmp`, plus `/var/cache/nginx` for the frontend. If your app writes anywhere else, add an `emptyDir` for that path, or the pod crashes with a read-only filesystem error.
- **`.trivyignore`** lists findings accepted on purpose, each with a reason. `KSV-0125` (trusted registries) needs a cluster admission policy, not a manifest change.
- **Still open:** `KSV-0110`, workloads in the `default` namespace. Moving to a dedicated namespace touches every manifest, the middleware annotation (`<namespace>-<name>@kubernetescrd`) and the deploy commands.

## Neon notes

Use the `-pooler` host in the `DATABASE_URL` the app runs with, plus `sslmode=require`. The app rewrites `postgresql://` to `postgresql+psycopg://` so SQLAlchemy picks psycopg 3. Run Alembic migrations with the direct (non-pooler) host from your own machine, not from the pod.

## Test locally

Build both images from your app, then run them behind a proxy that routes the way the Ingress does: `/api/*` to the backend with the prefix stripped, everything else to the frontend. A Caddyfile for that:

```text
:8080 {
    handle_path /api* {
        reverse_proxy backend:8000
    }
    handle {
        reverse_proxy frontend:5173
    }
}
```

```bash
docker build -t myapp-backend backend
docker build -t myapp-frontend frontend
docker network create app-test
# The same limits the pods get in k3s: read-only image, UID 10001, no capabilities.
# mode=1777 makes the scratch dirs writable like a k8s emptyDir (Docker's tmpfs is root-only by default).
LIMITS=(--read-only --user 10001:10001 --cap-drop ALL --security-opt no-new-privileges)  # an array, so it works in bash and zsh
TMP=size=64m,mode=1777
# SQLite keeps the test self-contained if your app supports it (TripLedger does).
# If yours needs Postgres, run a postgres container on app-test and point DATABASE_URL at it.
docker run -d --rm --name backend --network app-test "${LIMITS[@]}" --tmpfs /tmp:$TMP -e SECRET_KEY=$(openssl rand -hex 32) -e DATABASE_URL=sqlite:////tmp/test.db myapp-backend
docker run -d --rm --name frontend --network app-test "${LIMITS[@]}" --tmpfs /tmp:$TMP --tmpfs /var/cache/nginx:$TMP myapp-frontend
docker run -d --rm --name proxy --network app-test -p 8080:8080 -v "$PWD/Caddyfile:/etc/caddy/Caddyfile:ro" caddy:2-alpine
# Wait until the backend answers through the proxy (gives up after 60 s).
for i in $(seq 1 60); do curl -sf localhost:8080/api/health && break; sleep 1; done
open http://localhost:8080           # the app, calling /api on the same origin
```

When you're done testing in the browser, clean up:

```bash
docker rm -f proxy frontend backend && docker network rm app-test
```

`docker ps` should show both app containers as `(healthy)`. The `HEALTHCHECK`s call `127.0.0.1`, not `localhost`, because in Alpine `localhost` resolves to IPv6 `::1` and nginx listens on IPv4 only.

## Things to verify on the first deploy

- `/api/health` answers through the Ingress, and `/api/docs` loads. If API routes 404, check that the strip-prefix middleware exists (`kubectl get middleware`) and that the Ingress annotation names it as `<namespace>-<name>@kubernetescrd`. Older k3s releases shipped Traefik v2, which uses the `traefik.containo.us/v1alpha1` API group instead of `traefik.io/v1alpha1`.
- Pod logs show a successful database connection: `kubectl logs -l app=$APP_NAME-backend`.
- The certificate becomes ready: `kubectl get certificate`. Switch the Ingress annotation to `letsencrypt-staging` while debugging to avoid rate limits.

## Background

The pipeline rules (single `KUBECONFIG` secret name, no `paths-ignore` on manifests, secrets applied with `--dry-run=client -o yaml | kubectl apply -f -`, SHA-pinned actions, image scan, environment approval) come from `mern-k3s-deployment-guide/docs/pipeline-design.md`.
