+++
title = "Free Yourself - From Zero to Self-Hosted Cloud IDE"
subtitle = "Running Theia + Traefik v2 + traefik-forward-auth with Docker and Docker Compose in the cloud"
date = "2020-04-18T13:00:00+02:00"
draft = "true"
+++
Ever since I first learned about the concept of a cloud IDE through Cloud9 back in 2012-ish (before they were bought by AWS), I was drawn to the idea. [And while there are many good reasons to use a cloud IDE for coding](https://hellboundbloggers.com/need-cloud-based-ide/64800/) (professionally as well as recreatively), my personal number one reason was a different one: I wanted to do it, because I thought it was cool. Because I'm a nerd.
<!--more-->

For the very same reason, using an existing service like Codenvy, Codeanywhere or Koding is cool and all, but just not good enough. And while I certainly did *not* want to program my own web-based IDE from scratch, I *did* want to set it up myself, because I am that strange kind of person that considers these things "fun".

Then recently, [Theia 1.0 was released](https://dev.to/svenefftinge/theia-1-0-finally-a-good-browser-ide-3ok0), and I decided to give it a try. It took some fiddling, but right now I have something that I think I will be able to build upon, so I thought I'd share it with you, valued reader.

*Note: I started with [this guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-the-eclipse-theia-cloud-ide-platform-on-debian-10) for setting up Theia with [nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy/), but I quickly switched to Traefik as a reverse proxy, since setting up nginx-proxy to serve two different ports from the same container turned out to be overly complicated (I'm still not sure it is even possible). I also made various other additions and changes (like adding Google authentication), so I deemed it justified to make a new guide, even if it partially overlaps with the other one.*

## Goal
After following through this article, you will have
* A fully featured cloud IDE that can run VSCode extensions, reachable via `https://theia.[yourdomain.xyz]`
* The possibility to reach a developer preview server (such as the webpack dev server) via `https://prv.[yourdomain.xyz]`
* Both of these secured with OAuth (delegated to Google, but easily configurable to use any given OICD provider)

## Prerequisites
I know it says "From Zero" in the headline, but I lied. You need a few things:
* A server or virtual server with a public IP (I use a DigitalOcean droplet for $5/month)
* A domain that you have full control over (I bought a .xyz domain on Namecheap for ~$1/year)
* Four A-Records for your domain, pointing to said public IP:
  * `theia.[yourdomain.xyz]`
  * `prv.[yourdomain.xyz]`
  * `auth.[yourdomain.xyz]`
  * `traefik.[yourdomain.xyz]`
  * (alternatively a wildcard record: `*.[yourdomain.xyz]`)
* SSH access to said server with a user that is in the sudo group ([here](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04) is a guide on how to do that on a DigitalOcean droplet)
* [Docker](https://www.docker.com) and [Docker Compose](https://docs.docker.com/compose/) installed on said server
* *Some* knowledge about Docker and Docker Compose. Since the internet is full of guides and tutorials on them, I will not explain every detail about Docker and Docker Compose.

## Warmup
We want to use [Traefik](https://docs.traefik.io/) as a reverse proxy to easily route traffic to the different services we will create (one of them being the actual cloud IDE) and to handle authentication. Each service will run in its own container, and we will use Docker Compose to orchestrate this multi container environment. 

Connect to your server via SSH, then create a directory for your project and `cd` into it:

```shell {linenos=false}
mkdir ~/my-ide && cd ~/my-ide
```

### The .env file
Create a fille called `.env`, open it with your favourite editor...
```shell {linenos=false}
nano .env
```

...and paste the following content:
```shell
EMAIL=youremail@example.com
DOMAIN=example.com
```

Fill in your email and domain. Your email will be used to authenticate with Let's Encrypt, and the domain is used to configure Traefik. These variables will automatically be substituted in the `docker-compose.yaml` file we will be creating shortly. We are outsourcing them to a `.env` file to keep the actual `docker-compose.yaml` free of personal or sensitive information, so it can be shared and everyone can just use it out of the box by providing his/her own `.env` file.

### HTTP Basic Auth
For simplicity, we will be using HTTP basic auth to secure access to our services first, before switching to Google authentication later. Credentials for HTTP basic auth are provided in the form `username:hashed_password`. To generate such credentials, we can use the `htpasswd`command, which is part of the `apache2-utils` package on Debian and Ubuntu and part of the `httpd-tools` package for most RPM-based distributions. So make sure you have it installed. Now, to generate credentials and add them to your `.env` file under the key `BASICAUTH_USERS`, execute
```shell {linenos=false}
echo BASICAUTH_USERS=$(htpasswd -nbB username password) >> .env
```

Replace "username" and "password" with a username and password of your choosing.

## Traefik
Create a file called `docker-compose.yaml`...
```shell {linenos=false}
nano docker-compose.yaml
```

...and paste this content:

```yaml
version: "3.7"

networks:
  web:
    external: true

services:
  traefik:
    image: "traefik:v2.2"
    container_name: "traefik"
    command:
        # enable api and dashboard
      - "--api.dashboard=true"
        # use docker as provider
      - "--providers.docker=true"
        # don't automatically expose every container
      - "--providers.docker.exposedbydefault=false"
        # entrypoint for http
      - "--entrypoints.web.address=:80"
        # entrypoint for https
      - "--entrypoints.websecure.address=:443"
        # always redirect http to https
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
        # use Let's Encrypt to automatically issue and renew SSL certifiicates
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=${EMAIL}"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    labels:
        # tell traefik to expose this container
      - "traefik.enable=true"
        # expose the 'api@internal' service (automatically created by Traefik) under the subdomain 'traefik'
      - "traefik.http.routers.api.rule=Host(`traefik.${DOMAIN}`)"
      - "traefik.http.routers.api.service=api@internal"
        # secure access with HTTP basic auth
      - "traefik.http.routers.api.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=${BASICAUTH_USERS}"
        # access to this service only via HTTPS
      - "traefik.http.routers.api.entrypoints=websecure"
        # use Let's Encrypt to issue the SSL certificate
      - "traefik.http.routers.api.tls.certresolver=myresolver"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    networks:
      - "web"
```
We are configuring Traefik via commandline arguments that get passed to the executable (in the `commands` section of the YAML file) and via container labels (in the `labels` section). Consult the [Traefik documentation](https://docs.traefik.io/getting-started/configuration-overview/) for the exact difference between those two methods to pass config values to Traefik, but the gist is: Configuration done via container labels is dynamic, meaning you do not have to reboot Traefik if it changes, you just need to rebuild the container they are defined on. Traefik will automatically watch for and apply those changes.

This point is kind of defeated by the fact that we are using container labels *on the container running Traefik* (sometimes referred to as "Traefik-ception"), but this is a special case. We are configuring the endpoint for the API dashboard of Traefik itself.

If you now run
```shell {linenos=false}
docker-compose up -d 
```

and open `traefik.[yourdomain.xyz]` in your browser, you can see the Traefik dashboard. Nice.

## Theia

