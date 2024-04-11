# Documentation
This document describes how to get Keycloak working behind Traefik using Docker Compose. It is assumed that you are using a machine running Ubuntu.

## Prerequisites
1. Install Docker Engine and Docker Compose ([[1]](https://docs.docker.com/engine/install/ubuntu/),[[2]](https://docs.docker.com/compose/install/linux/#install-using-the-repository)):
```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
2. (Optional) Verify installation:
```
sudo docker run hello-world
```
3. Have some service running on `http://10.0.0.1:80` for Traefik to use, like a webserver. This assumes `10.0.0.1` is the IP address of the host you are using, change accordingly.
4. Have a publicly-accessible domain `service.domain.tld` for Traefik to use for the above service.
5. Have a publicly-accessible domain `keycloak.domain.tld` for Traefik to use for Keycloak as Keycloak needs a valid and trust-worthy TLS certificate, typically from Let's Encrypt.
6. Port 443 open on router.
## Installation
1. Create the directories that will hold the configuration files:
```
mkdir keycloak-traefik
mkdir keycloak-traefik/traefik_data
mkdir keycloak-traefik/traefik_letsencrypt
mkdir keycloak-traefik/keycloak_data
mkdir keycloak-traefik/postgres_data
cd keycloak-traefik
```
2. Create the `docker-compose.yml` file:
```
nano docker-compose.yml
```
3. Paste the following into the `docker-compose.yml` file:
```
version: '3'

services:
  postgres:
    container_name: keycloak-postgres
    image: postgres:latest
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: dbUSER
      POSTGRES_PASSWORD: dbPASS
      
  keycloak:
    container_name: keycloak-server
    image: quay.io/keycloak/keycloak:latest
    command: start-dev
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_PASSWORD: dbUSER
      KC_DB_USERNAME: dbPASS
      KC_DB_SCHEMA: public
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      PROXY_ADDRESS_FORWARDING: true
      KC_METRICS_ENABLED: true
      KC_PROXY: edge
    volumes:
      - ./keycloak_data:/data:rw
    ports:
      - 8081:8080
    depends_on:
      - postgres
      
  traefik:
    restart: always
    image: traefik:latest
    command:
      - "--log.level=DEBUG"
      - "--api.dashboard=true"
      - "--api.debug=true"
      - "--providers.docker=true"
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.http.address=:80"
      - "--entrypoints.https.address=:443"
      - "--log=true"
      - "--log.filepath=/etc/traefik/traefik.log"
      - "--providers.file=true"
      - "--providers.file.filename=/etc/traefik/dynamic.yml"
      - "--providers.file.watch=true"
      - "--experimental.plugins.keycloakopenid.modulename=github.com/Gwojda/keycloakopenid"
      - "--experimental.plugins.keycloakopenid.version=v0.1.35"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.dnschallenge=true"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.dnschallenge.provider=cloudflare"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.dnschallenge.resolvers=1.1.1.1:53"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.email=some-email@domain.tld"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.storage=/letsencrypt/acme.json"
    ports:  
      - 80:80
      - 443:443
    volumes:
      - ./traefik_data/:/etc/traefik
      - ./traefik_letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      - "traefik.http.routers.api.tls.certresolver=letsencrypt-dns-challenge"
      - "traefik.http.routers.api.tls.domains[0].main=domain.tld"
      - "traefik.http.routers.api.tls.domains[0].sans=*.domain.tld"
    environment:
      - "TZ=YOUR/TIMEZONE"
      - "CF_ZONE_API_TOKEN=YOUR-CLOUDFLARE-ZONE-API-TOKEN"
      - "CF_DNS_API_TOKEN=YOUR-CLOUDFLARE-DNS-API-TOKEN"
```
4. Visit [http://localhost:8081](http://localhost:8081) and login using `admin` and `admin` to access the `master` realm. From here, create a new realm by clicking the `master` drop-down menu on the upper left and click `Create realm`. In our example, name the ream `realm-name`.
5. In the `realm-name` realm, go to [http://localhost:8081/admin/master/console/#/realm-name/clients](http://localhost:8081/admin/master/console/#/realm-name/clients) or select `Clients` then `Create Client`. Select the following:
```
Client type: OpenID Connect
Client ID: client-id                                # replace if not using the same as the example
<Next>
Client Authentication: On
<Next>
Valid redirect URIs: https://service.domain.tld/*   # replace if not using the same as the example
Web Origins: *
```
6. Then, in the `Credentials` tab in the Client menu, copy the `Client Secret` which will look something like `9hgiD90dbqk54apjUl5LxAEDOkhZ80yB`.
7. Next, we begin setting up Traefik. Create the `dynamic.yml` Traefik configuration file:
```
nano traefik_data/dynamic.yml
```
8. Paste the template into the file and save:
```
http:
  middlewares:
    keycloak-middleware-name:
      plugin:
        keycloakopenid:
          ClientID: client-id
          ClientSecret: 9hgiD90dbqk54apjUl5LxAEDOkhZ80yB
          KeycloakRealm: realm-name
          KeycloakURL: https://keycloak.domain.tld            
          Scope: openid
          TokenCookieName: some-cookie-name
          UseAuthHeader: "false"
  routers:
    demo-service-route:
      rule: "Host(`service.domain.tld`)"
      service: "sample-service"
      entryPoints:
        - "https"
      middlewares:
        - "keycloak-middleware-name"
      tls:
        - certResolver: "letsencrypt-dns-challenge"
    keycloak-public-route:
      rule: "Host(`keycloak.domain.tld`)"
      service: "keycloak-server"
      entryPoints:
        - "https"
      tls:
        - certResolver: "letsencrypt-dns-challenge"
  services:
    sample-service:
      loadBalancer:
        servers:
          - url: "http://10.0.0.1:80"
    keycloak-server:
      loadBalancer:
        servers:
          - url: "http://10.0.0.1:8081"
```
9. Create and run the containers using
```
docker compose up -d
```

## docker-compose.yml File Details
1. We define the PostgreSQL service that Keycloak will use as its database. It will be named `keycloak-postgres`, store its data peristently in the `postgres_data` folder and use the database `keycloak` with username and password of `dbUSER` and `dbPASS`, respectively:
```
  postgres:
  container_name: keycloak-postgres
  image: postgres:latest
  volumes:
    - ./postgres_data:/var/lib/postgresql/data
  environment:
    POSTGRES_DB: keycloak
    POSTGRES_USER: dbUSER
    POSTGRES_PASSWORD: dbPASS
```
2. We then define the Keycloak service itself. The container name will be `keycloak-server` and use the `start-dev` command to start in development mode (`start` can be used for production mode). We then define the variables needed to use the previously-defined Postgres database with the `KC_DB` defining the database type, `KC_DB_URL` defining the location of the database, `KC_DB_USERNAME` and `KC_DB_PASSWORD` defining the credentials needed for the database. Here, we also set the default credentials to be used for the Keycloak `master` realm which will be `admin` and `admin` by default. `PROXY_ADDRESS_FORWARDING: true` and `KC_PROXY: edge` are needed for Keycloak to work behind a reverse proxy. Finally, the all the data for Keycloak will be written to the `keycloak_data`, if any, and we set the public-facing port to be `8081`. This means Keycloak will be accessible on `http://localhost:8081`. 
```
  keycloak:
    container_name: keycloak-server
    image: quay.io/keycloak/keycloak:latest
    command: start-dev
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_PASSWORD: dbUSER
      KC_DB_USERNAME: dbPASS
      KC_DB_SCHEMA: public
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      PROXY_ADDRESS_FORWARDING: true
      KC_METRICS_ENABLED: true
      KC_PROXY: edge
    volumes:
      - ./keycloak_data:/data:rw
    ports:
      - 8081:8080
    depends_on:
      - postgres
```
3. Lastly, we define the service for Traefik. We enable the API and give it a path to use the Docker socket. We then define the entry points and ports that Traefik will serve, which in this case is `http` with a port of 80 and `https` with a port of 443. You can add more if you want to serve more ports like 8000, 8080, etc. Next, we give Traefik the internal path that it will use to watch the `dynamic.yml` configuration file which can update Traefik's configuration in real time without having to restart the container. An important step is including the last two lines which import a plugin that helps Traefik handle OpenID and Keycloak. Lastly, we define the ports we are going to use (80 and 443 due to our entry points) and our volumes to store/access data.
There are also some configuration lines for automatic provisioning and renewing of certificates for `*.domain.tld` using Cloudflare's DNS and Let's Encrypt's DNS-01 challenge. This certificate resolver is given the name `letsencrypt-dns-challenge`. Any routes given this certificate resolver will get a valid and renewing Let's Encrypt certificate. Visit the [documentation](https://certbot-dns-cloudflare.readthedocs.io/en/stable/#credentials) for more details. 
```
  traefik:
    restart: always
    image: traefik:latest
    command:
      - "--log.level=DEBUG"
      - "--api.dashboard=true"
      - "--api.debug=true"
      - "--providers.docker=true"
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.http.address=:80"
      - "--entrypoints.https.address=:443"
      - "--log=true"
      - "--log.filepath=/etc/traefik/traefik.log"
      - "--providers.file=true"
      - "--providers.file.filename=/etc/traefik/dynamic.yml"
      - "--providers.file.watch=true"
      - "--experimental.plugins.keycloakopenid.modulename=github.com/Gwojda/keycloakopenid"
      - "--experimental.plugins.keycloakopenid.version=v0.1.35"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.dnschallenge=true"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.dnschallenge.provider=cloudflare"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.dnschallenge.resolvers=1.1.1.1:53"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.email=some-email@domain.tld"
      - "--certificatesresolvers.letsencrypt-dns-challenge.acme.storage=/letsencrypt/acme.json"
    ports:  
      - 80:80
      - 443:443
    volumes:
      - ./traefik_data/:/etc/traefik
      - ./traefik_letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      - "traefik.http.routers.api.tls.certresolver=letsencrypt-dns-challenge"
      - "traefik.http.routers.api.tls.domains[0].main=domain.tld"
      - "traefik.http.routers.api.tls.domains[0].sans=*.domain.tld"
    environment:
      - "TZ=YOUR/TIMEZONE"
      - "CF_ZONE_API_TOKEN=YOUR-CLOUDFLARE-ZONE-API-TOKEN"
      - "CF_DNS_API_TOKEN=YOUR-CLOUDFLARE-DNS-API-TOKEN"
```
## dynamic.yml File Details
1. For HTTP-based routes, we specify the `http` label as the "root". Under `http`, we define our `services`, `routers`, and `middlewares`.
2. Under `middlewares`, we define the plugin that we imported in the `docker-compose.yml` file along with the details and credentials we made earlier. Here, `keycloak-middleware-name` is the name of the middleware and can be changed to your liking. `ClientID`, `ClientSecret`, `KeycloakReam`, `KeycloakURL`, and `TokenCookieName` can be changed depending on your configuration:
```
http:
  middlewares:
    keycloak-middleware-name:
      plugin:
        keycloakopenid:
          ClientID: client-id
          ClientSecret: 9hgiD90dbqk54apjUl5LxAEDOkhZ80yB
          KeycloakRealm: realm-name
          KeycloakURL: https://keycloak.domain.tld            
          Scope: openid
          TokenCookieName: some-cookie-name
          UseAuthHeader: "false"
```
3. Next we define our `routers` which will map domains + ports (`service.domain.tld` + `443`) to backend `services` (`10.0.0.1:80`). It can also define `middlewares` (`keycloak-middleware-name`), if specified. We also define that this route will use TLS using the `letsencrypt-dns-challenge` certificate resolver. In this example, we make two routes, one for the demo service (`demo-service-route`) and one for Keycloak (`keycloak-public-route`). 
```
  routers:
    demo-service-route:
      rule: "Host(`service.domain.tld`)"
      service: "sample-service"
      entryPoints:
        - "https"
      middlewares:
        - "keycloak-middleware-name"
      tls:
        - certResolver: "letsencrypt-dns-challenge"
    keycloak-public-route:
      rule: "Host(`keycloak.domain.tld`)"
      service: "keycloak-server"
      entryPoints:
        - "https"
      tls:
        - certResolver: "letsencrypt-dns-challenge"
```
4. Lastly, we define the `services` that `routers` will route traffic to. 
```
  services:
    sample-service:
      loadBalancer:
        servers:
          - url: "http://10.0.0.1:80"
    keycloak-server:
      loadBalancer:
        servers:
          - url: "http://10.0.0.1:8081"
```
