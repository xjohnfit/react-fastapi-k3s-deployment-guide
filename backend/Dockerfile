FROM python:3.13-slim

WORKDIR /app
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1

COPY requirements.txt .
# pytest is a dev dependency, so it stays out of the image.
RUN grep -v '^pytest' requirements.txt > runtime.txt \
    && pip install --no-cache-dir -r runtime.txt

COPY app ./app

# UID 1001 must match runAsUser in kubernetes/backend-deployment.yml
RUN useradd --system --uid 1001 appuser
USER 1001

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request,sys; urllib.request.urlopen('http://localhost:8000/health', timeout=2)" || exit 1

# --root-path /api: the Ingress sends /api/* to this pod without stripping the prefix.
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--root-path", "/api", "--proxy-headers", "--forwarded-allow-ips", "*"]
