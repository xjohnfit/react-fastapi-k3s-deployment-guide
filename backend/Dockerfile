FROM python:3.14-slim

WORKDIR /app
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1

# requirements.txt is the runtime lock file (test tools live in requirements-dev.txt).
# Every package is pinned with a hash and installed from a wheel, so no setup.py runs.
COPY requirements.txt .
RUN pip install --no-cache-dir --require-hashes --only-binary :all: -r requirements.txt

COPY app ./app

# UID/GID 10001 must match runAsUser/runAsGroup in kubernetes/backend-deployment.yml
RUN groupadd --system --gid 10001 appuser && useradd --system --uid 10001 --gid 10001 appuser
USER 10001:10001

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request,sys; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)" || exit 1

# --root-path /api tells FastAPI it is served under /api (for docs and redirect URLs).
# The proxy must strip /api before forwarding (kubernetes/middleware-strip-api.yml):
# uvicorn adds the root path back onto every request path, so an unstripped
# /api/health would arrive as /api/api/health and 404.
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--root-path", "/api", "--proxy-headers", "--forwarded-allow-ips", "*"]
