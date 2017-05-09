# Docker + Nginx + Let's Encrypt + Rstudio + Shiny

## NICE TO HAVE FIXED

**Multiple users in Rstudio Server**

Would be nice to be able to create multiple users in Rstudio when firing up the docker container. 
Maybe based on some of the information here: https://itsalocke.com/r-training-environment/

----

This code originated at https://github.com/gilyes/docker-nginx-letsencrypt-sample.

This simple example shows how to set up Rstudio Server and Shiny Server running behind a dockerized Nginx reverse proxy and served via HTTPS using free [Let's Encrypt](https://letsencrypt.org) certificates. New sites can be added on the fly by just modifying `docker-compose.yml` and then running `docker-compose up` as the main Nginx config is automatically updated and certificates (if needed) are automatically acquired.

Some of the configuration from the original repo is derived from <https://github.com/fatk/docker-letsencrypt-nginx-proxy-companion-examples> with some simplifications and updates to work with current `nginx.tmpl` from [nginx-proxy](https://github.com/jwilder/nginx-proxy) and docker-compose v2 files.

## Running the example
### Prerequisites
* [docker](https://docs.docker.com/engine/installation/) (>= 1.10)
* [docker-compose](https://github.com/docker/compose/releases) (>= 1.8.1)
* access to (sub)domain(s) pointing to a publicly accessible server (required for TLS)

### Preparation
* Clone the [repository](https://github.com/56north/encrypted_dashboard) on the server pointed to by your domain. 
* In `docker-compose.yml`: 
  * Change the **VIRTUAL_HOST** and **LETSENCRYPT_HOST** entries from *rstudio.mydomain.com* and *shiny.mydomain.com* to your domains.
  * Change **LETSENCRYPT_EMAIL** entries to the email address you want to be associated with the certificates. 
  * Change **USER** and **PASSWORD** entries to the user and password you want for Rstudio.

### Running
In the main directory run: 
```bash
docker-compose up
```

This will perform the following steps:

* Download the required images from Docker Hub ([nginx](https://hub.docker.com/_/nginx/), [docker-gen](https://hub.docker.com/r/jwilder/docker-gen/), [docker-letsencrypt-nginx-proxy-companion](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion/)).
* Create containers from them.
* Build and create containers for Rstudio Server and Shiny Server
* Start up the containers. 
  * *docker-letsencrypt-nginx-proxy-companion* inspects containers' metadata and tries to acquire certificates as needed (if successful then saving them in a volume shared with the host and the Nginx container).
  * *docker-gen* also inspects containers' metadata and generates the configuration file for the main Nginx reverse proxy

If everything went well then you should now be able to access Rstudio and Shiny and the given addresses.

### Troubleshooting
* To view logs run `docker-compose logs`.
* To view the generated Nginx configuration run `docker exec -ti nginx cat /etc/nginx/conf.d/default.conf`

## How does it work

The system consists of 4 main parts:

* Main Nginx reverse proxy container.
* Container that generates the main Nginx config based on container metadata.
* Container that automatically handles the acquisition and renewal of Let's Encrypt TLS certificates.
* The actual servers living in their own containers. In this example Rstudio and Shiny.

### The main Nginx reverse proxy container
This is the only publicly exposed container, routes traffic to the backend servers and provides TLS termination.

Uses the official [nginx](https://hub.docker.com/_/nginx/) Docker image.

It is defined in `docker-compose.yml` under the **nginx** service block:

```
services:
  nginx:
    restart: always
    image: nginx
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/etc/nginx/conf.d"
      - "/etc/nginx/vhost.d"
      - "/usr/share/nginx/html"
      - "./volumes/proxy/certs:/etc/nginx/certs:ro"
```

As you can see it shares a few volumes:
* Configuration folder: used by the container that generates the configuration file.
* Default Nginx root folder: used by the Let's Encrypt container for challenges from the CA. 
* Certificates folder: written to by the Let's Encrypt container, this is where the TLS certificates are maintained. 

### The configuration generator container
This container inspects the other running containers and based on their metadata (like **VIRTUAL_HOST** environment variable) and a template file it generates the Nginx configuration file for the main Nginx container. When a new container is spinning up this container detects that, generates the appropriate configuration entries and restarts Nginx. 

Uses the [jwilder/docker-gen](https://hub.docker.com/r/jwilder/docker-gen/) Docker image.

It is defined in `docker-compose.yml` under the **nginx-gen** service block:

```
services:
  ...

  nginx-gen:
    restart: always
    image: jwilder/docker-gen
    container_name: nginx-gen
    volumes:
      - "/var/run/docker.sock:/tmp/docker.sock:ro"
      - "./volumes/proxy/templates/nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro"
    volumes_from:
      - nginx
    entrypoint: /usr/local/bin/docker-gen -notify-sighup nginx -watch -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
```

The container reads the `nginx.tmpl` template file (source: [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy)) via a volume shared with the host.

It also mounts the Docker socket into the container in order to be able to inspect the other containers (the `"/var/run/docker.sock:/tmp/docker.sock:ro"` line). 
**Security warning**: mounting the Docker socket is usually discouraged because the container getting (even read-only) access to it can get root access to the host. In our case, this container is not exposed to the world so if you trust the code running inside it the risks are probably fairly low. But definitely something to take into account. See e.g. [The Dangers of Docker.sock](https://raesene.github.io/blog/2016/03/06/The-Dangers-Of-Docker.sock/) for further details.

NOTE: it would be preferrable to have docker-gen only handle containers with exposed ports (via `-only-exposed` flag in the `entrypoint` script above) but currently that does not work, see e.g. <https://github.com/jwilder/nginx-proxy/issues/438>.

### The Let's Encrypt container
This container also inspects the other containers and acquires Let's Encrypt TLS certificates based on the **LETSENCRYPT_HOST** and **LETSENCRYPT_EMAIL** environment variables. At regular intervals it checks and renews certificates as needed. 

Uses the [jrcs/letsencrypt-nginx-proxy-companion](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion/) Docker image.

It is defined in `docker-compose.yml` under the **letsencrypt-nginx-proxy-companion** service block:

```
services:
  ...

  letsencrypt-nginx-proxy-companion:
    restart: always
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: letsencrypt-nginx-proxy-companion
    volumes_from:
      - nginx
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./volumes/proxy/certs:/etc/nginx/certs:rw"
    environment:
      - NGINX_DOCKER_GEN_CONTAINER=nginx-gen
```

The container uses a volume shared with the host and the Nginx container to maintain the certificates.

It also mounts the Docker socket in order to inspect the other containers. See the security warning above in the docker-gen section about the risks of that.

### The Rstudio Server and Shiny Server
These two servers are running in their own respective containers. They are defined in `docker-compose.yml` under the **tidyverse** and **shiny** service blocks: 

```
services:
  ...

  tidyverse:
    restart: always
    image: rocker/tidyverse
    container_name: rstudio
    expose:
      - "8787"
    environment:
      - VIRTUAL_HOST=rstudio.mydomain.com
      - VIRTUAL_NETWORK=nginx-proxy
      - VIRTUAL_PORT=80
      - LETSENCRYPT_HOST=rstudio.mydomain.com
      - LETSENCRYPT_EMAIL=me@myemail.com
      - USER=test
      - PASSWORD=test
    volumes:
      - shiny-apps:/home/mikkel/apps
      - r-packages:/usr/local/lib/R/site-library

  shiny:
    restart: always
    image: rocker/shiny
    container_name: shiny
    expose:
      - "3838"
    environment:
      - VIRTUAL_HOST=shiny.mydomain.com
      - VIRTUAL_NETWORK=nginx-proxy
      - VIRTUAL_PORT=80
      - LETSENCRYPT_HOST=shiny.mydomain.com
      - LETSENCRYPT_EMAIL=me@myemail.com
    volumes:
      - shiny-apps:/srv/shiny-server/
      - ./volumes/shiny/logs:/var/log/
      - r-packages:/usr/local/lib/R/site-library
```
The important part here are the environment variables and the volumes. The environment variables are used by the config generator and certificate maintainer containers to set up the system.

### The data volumes

The volumes are used to ensure that Rstudio and Shiny share apps and packages in order for you to build apps in Rstudio and have them deployed on the Shiny server without too big a hassle.

### The Dockerfile

This is the docker image I'm using in conjunction with the
[Encrypted Dashboard](/) project.

Currently, [the new docker image for Rstudio](https://github.com/rocker-org/rocker-versioned)
Exposes both ports `8787` and `3838` which doesn't work with the Encrypted Dashboard setup.

## Conclusion
This can be a fairly simple way to have easy, reproducible deploys for a secure R-based dashboard solution with auto-renewing TLS certificates. 
