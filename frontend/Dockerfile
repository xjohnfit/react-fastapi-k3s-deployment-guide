# Build stage
FROM node:26-alpine AS builder
WORKDIR /app
COPY package*.json ./
# No package install scripts (the only one in the lock file, fsevents, is macOS-only).
RUN npm ci --ignore-scripts
COPY . .
# Relative URL: the browser calls /api on whatever domain served the page.
ARG VITE_API_URL=/api
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

# Production stage (static SPA, no Node in the final image)
FROM nginx:alpine
RUN apk add --no-cache dumb-init
# UID/GID 10001 must match runAsUser/runAsGroup in kubernetes/frontend-deployment.yml
RUN addgroup -g 10001 -S nginx-app && adduser -S -u 10001 -G nginx-app nginx-app
# Config and site files stay owned by root (read-only for nginx-app), so a
# compromised nginx process can't rewrite the pages it serves.
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=builder /app/dist /usr/share/nginx/html
# nginx-app only needs write access to its cache and log directories.
RUN mkdir -p /var/cache/nginx /var/log/nginx && \
    chown -R nginx-app:nginx-app /var/cache/nginx /var/log/nginx
USER 10001:10001
EXPOSE 5173
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://127.0.0.1:5173/ || exit 1
ENTRYPOINT ["dumb-init", "--"]
CMD ["nginx", "-g", "daemon off;"]
