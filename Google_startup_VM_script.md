#!/bin/bash
# Restore sudo/root access for existing user 'heftworld'

if id -u heftworld >/dev/null 2>&1; then
  usermod -aG google-sudoers heftworld || gpasswd --add heftworld google-sudoers
else
  useradd -m -s /bin/bash heftworld
  gpasswd --add heftworld google-sudoers
fi

# Optional: ensure sudoers config allows this group
if ! grep -q "google-sudoers" /etc/sudoers; then
  echo "%google-sudoers ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
fi


docker build -t stripe-flask-app:v1 .

docker build -t devlino-api:v1 .

docker build -t devlino-frontend:v1 .

docker network create devlino-net

docker run -d \
  --name traefik \
  --network devlino-net \
  -p 80:80 \
  -p 443:443 \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v $(pwd)/letsencrypt:/letsencrypt \
  traefik:v3.0 \
  --providers.docker=true \
  --providers.docker.exposedbydefault=false \
  --entrypoints.web.address=:80 \
  --entrypoints.websecure.address=:443 \
  --certificatesresolvers.letsencrypt.acme.email=YOUR_EMAIL \
  --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json \
  --certificatesresolvers.letsencrypt.acme.httpchallenge=true \
  --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web



docker run -d \
  --network devlino-net \
  -v $(pwd)/logs:/app/logs \
  -e ENVIRONMENT=production \
  --env-file .env \
  --label traefik.enable=true \
  --label "traefik.http.routers.stripe.rule=Host(`stripe.devlino.com`)" \
  --label traefik.http.routers.stripe.entrypoints=websecure \
  --label traefik.http.routers.stripe.tls.certresolver=letsencrypt \
  --label traefik.http.services.stripe.loadbalancer.server.port=5000 \
  stripe-flask-app:v1

  
  
docker run -d \
  --network devlino-net \
  --name devlino-app \
  -e ENVIRONMENT=production \
  --env-file .env \
  -v $(pwd)/logs:/app/logs \
  --label traefik.enable=true \
  --label "traefik.http.routers.devlino-app.rule=Host(`devlino.com`)" \
  --label traefik.http.routers.devlino-app.entrypoints=websecure \
  --label traefik.http.routers.devlino-app.tls.certresolver=letsencrypt \
  --label traefik.http.services.devlino-app.loadbalancer.server.port=5000 \
  devlino-frontend:v1

  
  
docker run -d \
--name devlino-api \
--network devlino-net \
--log-driver json-file \
--log-opt max-size=10m \
--log-opt max-file=3 \
--log-opt compress=true \
-e ENVIRONMENT=production \
--env-file .env \
devlino-api:v1


| Hostname           | Type | Value (your static IP) |
| ------------------ | ---- | ---------------------- |
| devlino.com        | A    | 34.xx.xx.xx            |
| stripe.devlino.com | A    | 34.xx.xx.xx            |
