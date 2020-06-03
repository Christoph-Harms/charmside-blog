+++
title = "Free Yourself - From Zero to Self-Hosted Cloud IDE"
subtitle = "Running Theia + Traefik v2 + traefik-forward-auth with Docker and Docker Compose in the cloud"
date = "2020-06-03T13:00:00+02:00"
tags = ["theia", "traefik", "traefik-forward-auth", "docker", "docker-compose", "cloudIDE"]
keywords = ["theia", "traefik", "traefik-forward-auth", "docker", "docker-compose", "cloudIDE"]
draft = "false"
+++
Ever since I first learned about the concept of a cloud IDE through Cloud9 back in 2012-ish (before they were bought by AWS), I was drawn to the idea. [And while there are many good reasons to use a cloud IDE for coding](https://hellboundbloggers.com/need-cloud-based-ide/64800/) (professionally as well as recreatively), my personal number one reason was a different one: I wanted to do it, because I thought it was cool. Because I'm a nerd.
<!--more-->

For the very same reason, using an existing service like Codenvy, Codeanywhere or Koding is cool and all, but just not good enough. And while I certainly did *not* want to program my own web-based IDE from scratch, I *did* want to set it up myself, because I am that strange kind of person that considers these things "fun".

Then recently, [Theia 1.0 was released](https://dev.to/svenefftinge/theia-1-0-finally-a-good-browser-ide-3ok0), and I decided to give it a try. It took some fiddling, but right now I have something that I think I will be able to build upon, so I thought I'd share it with you, valued reader.

*Note: I started with [this guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-the-eclipse-theia-cloud-ide-platform-on-debian-10) for setting up Theia with [nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy/), but I quickly switched to Traefik as a reverse proxy, since setting up nginx-proxy to serve two different ports from the same container turned out to be overly complicated (I'm still not sure it is even possible). I also made various other additions and changes (like adding Google authentication), so I deemed it justified to make a new guide, even if it partially overlaps with the other one.*

## Goal
After following through this article, you will have
* A fully featured cloud IDE that can run VSCode plugins, reachable via `https://theia.[yourdomain.xyz]`
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
We are configuring Traefik via commandline arguments that get passed to the executable (in the `commands` section of the YAML file) and via container labels (in the `labels` section). Consult the [Traefik documentation](https://docs.traefik.io/getting-started/configuration-overview/) for the exact difference between those two methods of passing config values to Traefik, but the gist is: Configuration done via container labels is dynamic, meaning you do not have to reboot Traefik if it changes, you just need to rebuild the container they are defined on. Traefik will automatically watch for and apply those changes.

This point is kind of defeated by the fact that we are using container labels *on the container running Traefik* (sometimes referred to as "Traefik-ception"), but this is a special case. We are configuring the endpoint for the API dashboard of Traefik itself.

If you now run
```shell {linenos=false}
docker-compose up -d 
```

and open `traefik.[yourdomain.xyz]` in your browser, then enter the username and password you [created earlier](#http-basic-auth), you can see the Traefik dashboard. Nice.

## Theia
Let's get to the juicy part of this. Add the highlighted code to your `docker-compose.yaml` after line 50:

```yaml {linenostart=46,hl_lines=["6-19"]}
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    networks:
      - "web"
  theia:
    image: "theiaide/theia"
    labels:
      # expose this container
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      # expose under the subdomain 'theia'
      - "traefik.http.routers.theia-ide.rule=Host(`theia.${DOMAIN}`)"
      # secure access with basic auth
      - "traefik.http.routers.theia-ide.middlewares=auth"
      # use Let's Encrypt for SSL
      - "traefik.http.routers.theia-ide.tls.certresolver=myresolver"
    networks:
      - "web"
```

Save the file, spin it up with `docker-compose up -d` and visit `theia.[yourdomain.xyz]`. After entering your [username and password configured for basic auth](#http-basic-auth), there it is: your very own cloud IDE. *Tadaaaaa*. But we still have some work to do.

### Persisting project storage and settings
At the moment, any files we create or settings we save in Theia only live *inside* the container. Which is fine as long as we never stop or rebuild the container, but having non-discardable containers is an antipattern. When we modify something in the future and have to rebuild, we want all our project files, settings and keybindings to still be there.

We will use [Docker volumes](https://docs.docker.com/storage/volumes/) to achieve what we need. Add the highlighted code after line 6 to your `docker-compose.yaml`:
```yaml {hl_lines=["7-9"]}
version: "3.7"

networks:
  web:
    external: true

volumes:
  theia-project:
  theia-settings:
```

and add this highlighted code after what is now line 56:
```yaml {linenostart=55,hl_lines=["3-5"]}
  theia:
    image: "theiaide/theia"
    volumes:
      - "theia-project:/home/project/"
      - "theia-settings:/home/theia/.theia/"
```
Bring it up wit `docker-compose up -d` again, and everything works, because Docker volumes are just GREA... wait, what? Actually, it does *not* work. The volumes are configured and mounted correctly, but if you create any files in your IDE and run `docker-compose down && docker-compose up -d`, they are gone. If you try to change a setting or keybinding, it doesn't even get saved. What the heck?

When I first came across this, I was very confused, but seeking help online quickly revealed that this is a permission issue (which are ["a recurring theme when dealing with docker volumes"](https://github.com/theia-ide/theia-apps/issues/345#issuecomment-615259600)). I also found [this article on Medium](https://medium.com/@nielssj/docker-volumes-and-file-system-permissions-772c1aee23ca) where the author explains in detail what is going wrong and how to fix it.

Basically, the volumes are created belonging to `root:root`, but the user trying to write to them is `theia`. The aforementioned Medium article tells us how to fix this: By `chown`ing the directories on the host to the GID of a group the  `theia` user in the container belongs to. It does not matter if the GID does not have an associated group on the host.

The GID of the `theia` group inside the container is `1001` (I found that out by running the `id` command inside the Theia console). Docker volumes normally live inside `/var/lib/docker/volumes/`, so if you run `sudo ls -lah /var/lib/docker/volumes` on the host, you should see something like this:
```shell {linenos=false}
total 40K
drwx------  4 root root 4.0K Apr 24 17:47 .
drwx--x--x 14 root root 4.0K Apr 20 18:49 ..
-rw-------  1 root root  32K Apr 24 17:47 metadata.db
drwxr-xr-x  3 root root 4.0K Apr 24 17:47 my-ide_theia-project
drwxr-xr-x  3 root root 4.0K Apr 24 17:47 my-ide_theia-settings
```

Every docker volume corresponds to a directory named in the form `[name of the directory containing the volume definition]_[defined volume name]`. Running the following series of commands will fix our issue (change the name of the volume folders if they differ on your machine):
```shell {linenos=false}
sudo chown -R :1001 /var/lib/docker/volumes/{my-ide_theia-project,my-ide_theia-settings} \
&& sudo chmod -R 775 /var/lib/docker/volumes/{my-ide_theia-project,my-ide_theia-settings} \
&& sudo chmod g+s /var/lib/docker/volumes/{my-ide_theia-project,my-ide_theia-settings}
```

Our IDE now persists project files and settings beyond the lifetime of the container.

### Exposing the dev-preview server
When developing, especially when developing web applications, we often want to be able to access a _preview server_, that shows us the current state of the app/website/etc under development, and ideally refreshes automatically when the code changes. A famous example for this is the [Webpack Dev Server](https://webpack.js.org/configuration/dev-server/).

These webservers normally listen to another port on the same machine. So basically, what we want to do is to reach another port in the  same container via another URL. More specific, if our preview server listens to port 7070 on `localhost` inside the container, we want Traefik to route the URL `https://prv.[yourdomain.xyz]` to that container and port.

To achieve this, we can use container labels to define another service on that port and attach it to another router with our desired URL in the `Host` rule. Add the highlighted code to your `docker-compose.yaml`:

```yaml {linenostart=55,hl_lines=["10-15", "18-21", 24, 27]}
  theia:
    image: "theiaide/theia"
    volumes:
      - "theia-project:/home/project/"
      - "theia-settings:/home/theia/.theia/"
    labels:
      # expose this container
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      # define one service (called "ide") on port 3000 (this is the Theia IDE)
      - "traefik.http.services.ide.loadbalancer.server.port=3000"
      # define another service (called "devprv") on port 7070 (this is the dev preview server)
      - "traefik.http.services.devprv.loadbalancer.server.port=7070"
      # define the router "theia-ide" and attach the "ide" service to it
      - "traefik.http.routers.theia-ide.service=ide"
      # expose under the subdomain 'theia'
      - "traefik.http.routers.theia-ide.rule=Host(`theia.${DOMAIN}`)"
      # define the router "theia-devprv" and attach the "devprv" service to it
      - "traefik.http.routers.theia-devprv.service=devprv"
      # expose under the subdomain 'prv'
      - "traefik.http.routers.theia-ide.rule=Host(`prv.${DOMAIN}`)"
      # secure access with basic auth
      - "traefik.http.routers.theia-ide.middlewares=auth"
      - "traefik.http.routers.theia-devprv.middlewares=auth"
      # use Let's Encrypt for SSL
      - "traefik.http.routers.theia-ide.tls.certresolver=myresolver"
      - "traefik.http.routers.theia-devprv.tls.certresolver=myresolver"
    networks:
      - "web"
```

We are defining two services (they are defined "on the fly" when their `loadbalancer.server.port` property is set). One is called "ide", which is basicially what the container did before. If no Traefik service is explicitly defined on a container, Traefik will assume a default service based on the exposed port of the container and attach it to the first router defined. That is why there was _no_ service definition before (although there _was_ a service, the default service) and now there are _two_ service definitions.

We associate port 3000 (which is the default port for the  Theia IDE) with one service and port 7070 with the other. Then we configure the routing rules for those two services, secure both of them with basic auth and enable SSL for them.

Now, if you want to reach a dev preview server via `https://prv.[yourdomain.xyz]`, you just have to configure it to run on port 7070. In the case of the Webpack Dev Server, that is as easy as passing `--port=7070` when starting it.

## traefik-forward-auth
HTTP Basic Auth really does not cut the slack. Luckily, Traefik comes with the possibility to [forward the authentication to any given  service](https://docs.traefik.io/middlewares/forwardauth/). So we could, for example, write a service that checks credentials against a database that we create ourselves. Or we could write a service that uses Google's OAuth to authenticate users. Oh wait. Someone already did that. :)

[traefik-forward-auth](https://github.com/thomseddon/traefik-forward-auth) is a service that runs in a Docker container and does exactly that: Delegate authentication to Google (or any OICD provider).

To use it, you will first have to create a Google client ID and client secret in the developer console. Instructions on how to do that can be found [here](https://github.com/thomseddon/traefik-forward-auth#provider-setup) or [here](https://www.smarthomebeginner.com/google-oauth-with-traefik-docker/) (or in a thousand more places, really). You will also need a randomly generated string. You can  obtain one by running `head /dev/urandom | tr -dc A-Za-z0-9 | head -c 32 ; echo ''` in the console.

After you obtained your randomly generated string, your client ID and secret, add the highlighted lines to your `docker-compose.yaml`:

```yaml {linenostart=79,hl_lines=["6-22"]}
      # use Let's Encrypt for SSL
      - "traefik.http.routers.theia-ide.tls.certresolver=myresolver"
      - "traefik.http.routers.theia-devprv.tls.certresolver=myresolver"
    networks:
      - "web"
  traefik-forward-auth:
    image: "thomseddon/traefik-forward-auth:2"
    environment:
      - "PROVIDERS_GOOGLE_CLIENT_ID=[YOUR_CLIENT_ID]"
      - "PROVIDERS_GOOGLE_CLIENT_SECRET=[YOUR_CLIENT_SECRET]"
      - "SECRET=[YOUR_RANDOMLY_GENERATED_STRING]"
      - "COOKIE_DOMAIN=${DOMAIN}"
      - "WHITELIST=${EMAIL}"
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.oauth.rule=Host(`auth.${DOMAIN}`)"
      - "traefik.http.routers.oauth.middlewares=auth"
      - "traefik.http.routers.oauth.service=oauth-service"
      - "traefik.http.services.oauth-service.loadbalancer.server.port=4181"
    networks:
      - "web"
```

Replace the placeholders in square brackets with the actual values. After this, add the following, highlighted lines (more exactly, _change_ lines 40 and 42, then add the next two lines after line 42):

```yaml {linenostart=34,hl_lines=[7,"9-11"]}
    labels:
        # tell traefik to expose this container
      - "traefik.enable=true"
        # expose the 'api@internal' service (automatically created by Traefik) under the subdomain 'traefik'
      - "traefik.http.routers.api.rule=Host(`traefik.${DOMAIN}`)"
      - "traefik.http.routers.api.service=api@internal"
        # secure access with Google Oauth
      - "traefik.http.routers.api.middlewares=auth"
      - "traefik.http.middlewares.auth.forwardauth.address=http://traefik-forward-auth:4181"
      - "traefik.http.middlewares.auth.forwardauth.authResponseHeaders=X-Auth-User, X-Secret, X-Forwarded-User, X-WebAuth-User"
      - "traefik.http.middlewares.auth.forwardauth.trustForwardHeader=true"
        # access to this service only via HTTPS
      - "traefik.http.routers.api.entrypoints=websecure"
        # use Let's Encrypt to issue the SSL certificate
      - "traefik.http.routers.api.tls.certresolver=myresolver"
```


Scroll to the end of this article for a complete `docker-compose.yaml`.

That's it! Restart everything with ` docker-compose down && docker-compose up -d`, visit your IDE and you will not be asked to pass  HTTP Basic Auth, but you will be required to be logged in to the google account that you added to the `WHITELIST` variable in the previous code example. You can add more email addresses there by separating them with a comma.

## Conclusion
If you want to develop a JS-based web application with your new cloud IDE, you're pretty much good to go now. However, there is still more to be done. For example, we can not install VSCode plugins as promised _yet_. This will be explained in a followup article that I will write shortly, so thank you for your patience.

This is how your `docker-compose.yaml` should look after following through this whole article:
```yaml
version: "3.7"

networks:
  web:
    external: true

volumes:
  theia-project:
  theia-settings:

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
        # secure access with Google Oauth
      - "traefik.http.routers.api.middlewares=auth"
      - "traefik.http.middlewares.auth.forwardauth.address=http://traefik-forward-auth:4181"
      - "traefik.http.middlewares.auth.forwardauth.authResponseHeaders=X-Auth-User, X-Secret, X-Forwarded-User, X-WebAuth-User"
      - "traefik.http.middlewares.auth.forwardauth.trustForwardHeader=true"
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
  theia:
    image: "theiaide/theia"
    volumes:
      - "theia-project:/home/project/"
      - "theia-settings:/home/theia/.theia/"
    labels:
      # expose this container
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      # define one service (called "ide") on port 3000 (this is the Theia IDE)
      - "traefik.http.services.ide.loadbalancer.server.port=3000"
      # define another service (called "devprv") on port 7070 (this is the dev preview server)
      - "traefik.http.services.devprv.loadbalancer.server.port=7070"
      # define the router "theia-ide" and attach the "ide" service to it
      - "traefik.http.routers.theia-ide.service=ide"
      # expose under the subdomain 'theia'
      - "traefik.http.routers.theia-ide.rule=Host(`theia.${DOMAIN}`)"
      # define the router "theia-devprv" and attach the "devprv" service to it
      - "traefik.http.routers.theia-devprv.service=devprv"
      # expose under the subdomain 'prv'
      - "traefik.http.routers.theia-ide.rule=Host(`prv.${DOMAIN}`)"
      # secure access with basic auth
      - "traefik.http.routers.theia-ide.middlewares=auth"
      - "traefik.http.routers.theia-devprv.middlewares=auth"
      # use Let's Encrypt for SSL
      - "traefik.http.routers.theia-ide.tls.certresolver=myresolver"
      - "traefik.http.routers.theia-devprv.tls.certresolver=myresolver"
    networks:
      - "web"
  traefik-forward-auth:
    image: "thomseddon/traefik-forward-auth:2"
    environment:
      - "PROVIDERS_GOOGLE_CLIENT_ID=[YOUR_CLIENT_ID]"
      - "PROVIDERS_GOOGLE_CLIENT_SECRET=[YOUR_CLIENT_SECRET]"
      - "SECRET=[YOUR_RANDOMLY_GENERATED_STRING]"
      - "COOKIE_DOMAIN=${DOMAIN}"
      - "WHITELIST=${EMAIL}"
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.oauth.rule=Host(`auth.${DOMAIN}`)"
      - "traefik.http.routers.oauth.middlewares=auth"
      - "traefik.http.routers.oauth.service=oauth-service"
      - "traefik.http.services.oauth-service.loadbalancer.server.port=4181"
    networks:
      - "web"
```
