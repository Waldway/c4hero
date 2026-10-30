# c4hero (Wald°way setup): rules for agents

This is c4hero, a local C4 diagram editor, run in Docker on the local Mac.
The upstream code belongs to its maintainer. We only add the Docker files below.

## Rules

- Do not change c4hero's source code. Only add or edit the four Docker files.
- Do not push to the upstream repository.
- Do not add dependencies or change `package-lock.json`.
- Keep `.git` out of `.dockerignore`. The build reads the version number from Git.
- The port stays bound to `127.0.0.1`, so only this Mac can reach it.
- The security policy in `nginx.conf` allows only the page itself and the three AI providers. Never add tracking or analytics hosts.

## Setup

If the files below are missing, create them in the repo root with exactly this content.

`Dockerfile`

```dockerfile
FROM node:22-alpine AS build
RUN apk add --no-cache git
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts --no-audit --no-fund
COPY . .
RUN npm run build

FROM nginx:1.30-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

`compose.yaml`

```yaml
services:
  c4hero:
    build: .
    ports:
      - "127.0.0.1:3004:80"
    restart: unless-stopped
```

`.dockerignore`

```
node_modules
dist
.env
e2e
```

`nginx.conf`

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "no-referrer" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; connect-src 'self' https://api.anthropic.com https://api.openai.com https://generativelanguage.googleapis.com; font-src 'self'; media-src 'self' blob:; manifest-src 'self'; object-src 'none'; frame-src 'none'; worker-src 'self' blob:; base-uri 'self'; form-action 'none'; frame-ancestors 'none'" always;

    location / {
        try_files $uri /index.html;
    }
}
```

## Run

```sh
docker compose build --pull && docker compose up -d
```

Then open `http://localhost:3004` in Chrome or Brave. Diagrams are not stored in the container: the browser saves them into a folder you pick.

## Update from upstream

```sh
git fetch upstream --tags
git merge <new-tag>
docker compose build --pull && docker compose up -d
```

Merge tagged releases only, and read the changes before rebuilding.
