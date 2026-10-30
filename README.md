# React + FastAPI on k3s

A starting point for deploying a React (Vite) frontend and a FastAPI backend to a k3s cluster with GitHub Actions. It follows the same layout as `mern-k3s-deployment-guide`. The differences are the Python backend image, pytest in CI, and a single-domain Ingress that sends `/api` to FastAPI.

Status: untested. Nothing here has run against a real cluster yet. Expect to fix things on the first deploy.

## Layout

```
backend/            FastAPI app, requirements.txt, Dockerfile
frontend/           Vite + React, Dockerfile, nginx.conf
kubernetes/         manifests with ${VAR} placeholders
.github/workflows/  deploy.yml
```

## How a request flows

The browser loads the React app from `https://DOMAIN/`. The app calls `/api/...` on the same domain, so there is no CORS setup. Traefik sends `/api` to the backend Service and `/` to the frontend Service. The backend runs with `--root-path /api`, and the Ingress does not strip the prefix.

## What your app must provide

- `backend/requirements.txt` with `pytest` in it (the Dockerfile drops it from the image)
- `GET /health` on the backend
- `DATABASE_URL` and `SECRET_KEY` read from environment variables
- frontend scripts named `lint`, `typecheck` and `build`
- `VITE_API_URL` read at build time, normally `/api`

## Setup steps

1. Copy `backend/Dockerfile`, `frontend/Dockerfile`, `frontend/nginx.conf`, `kubernetes/` and `.github/` into your project.
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

## Neon notes

Use the `-pooler` host in the `DATABASE_URL` the app runs with, plus `sslmode=require`. The app rewrites `postgresql://` to `postgresql+psycopg://` so SQLAlchemy picks psycopg 3. Run Alembic migrations with the direct (non-pooler) host from your own machine, not from the pod.

## Things to verify on the first deploy

- `/api/health` and `/health` both answer through the Ingress. If routes 404, the `--root-path` handling is the first suspect.
- Pod logs show a successful database connection: `kubectl logs -l app=$APP_NAME-backend`.
- The certificate becomes ready: `kubectl get certificate`. Switch the Ingress annotation to `letsencrypt-staging` while debugging to avoid rate limits.

## Background

The pipeline rules (single `KUBECONFIG` secret name, no `paths-ignore` on manifests, secrets applied with `--dry-run=client -o yaml | kubectl apply -f -`, pinned actions, image scan, environment approval) come from `mern-k3s-deployment-guide/docs/pipeline-design.md`.
