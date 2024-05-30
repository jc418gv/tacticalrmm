# This is not Tactical-RMM. [This is Tactical RMM](https://github.com/amidaware/tacticalrmm)

## Then what's this?

# This is a Docker Tweak To Reverse Proxy Tactical-RMM behind Traefik

**One Time Caveat: Consider all code and command blocks to be examples that you should modify for your environment**.

While assessing Tactical RMM, I noticed that the project offers an impressive reverse proxy for a host and no other great ingress. \
That project must, quite understandably, focus on that primary configuration for support. What suprised me was the lack of obvious \
success by the community to run [Traefik](https://traefik.io/) with a [Docker installation](https://docs.tacticalrmm.com/install_docker/). 

I created this fork to track my changes to the compose files, which I assumed would be extensive. It turns out that \
most of the tricks to make this work happen outside of the project. I will document notes on those here for my own \
reference. Maybe others will have a similar traefik environment or the concepts will help their troubleshooting. 

This guide will be most relevant if you:
 - Are already running https services in Traefik
 - Work primarily with docker containers instead of host installation
 - Do reverse proxy by host matching
 - Use labels to define routers / services


## Challenge 1: SSL Layers Everywhere

Tactical offers a ```--insecure``` install without encryption. This would be no challenge if we had a docker image of that. It is an unfortunate \
reality that would also result in far more people operating insecurely and I'll respect the absence of an external SSL option. The \
result is that everything in and out of the TRMM docker stack is encrypted. 

Also note that the application has quite a bit of internal communication that it wants to perform over https. It will fail if \
the authority is invalid. The application must be able to encrypt its communication. 

The challenge this poses to traefik is that the information it needs to route the request _is also encrypted_. 

The adherence to https in TRMM is laudable for being beginner-friendly. If you're reading this, you're already getting certs and \
this is actually another thing to deal with instead. There are environment variable options but between formatting, renewals, and \
having the right domains available I ended up using an alternate approach. 

## Challenge 2: Host Matching Everywhere

Tactical's docker stack is designed to serve https schema over port 443. 

If you're reading this, traefik needs to bind to 443 for other services. 

## Solving both

Let's walk through a [Docker install](https://docs.tacticalrmm.com/install_docker/) bearing in mind that we need this to play well \
with other https services on traefik. 

## Acquiring Certs

I'm documenting an alternate approach to this section. The documentated approach strikes me as having a high priority \
of making things as beginner-friendly and easy as possible. I have no criticism of that. It's also suitable for many \
non-beginners. If it's not suitable for you then this section might help:

#### TRMM Doesn't Need A Wildcard Certificate

TRMM needs a chain that covers the api, mesh, and rmm subdomains of example.com. The official docs use certbot. Install \
the plugin for your provider if needed and put together a command that might look like this:

```
sudo certbot certonly\
--dns-cloudflare \
--dns-cloudflare-credentials /opt/appdata/traefik/certs/cloudflare.ini \
-d rmm.example.com \
-d api.example.com \
-d mesh.example.com
```

The (```-rw-------```) cloudflare.ini file contains the api token to read/edit zones. All three certificates will be generated together. They will \
output to ```/etc/letsencrypt/live/rmm.example.com/``` in one chain that will be valid for all requests. It is a \
wildcard as far as TRMM knows. 

The modified docker-compose.yml in this project will ready-only mount those certificates in the ```trmm-nginx``` container \ 
to use them directly. This replaces install directions including the step ```echo "CERT_PUB_KEY=$(sudo base64 -w 0 \
/path/to/pub/key)" >> .env``` as a method to provide the certificate. This also means that TRMM will always have \
(read-only) access to the most recent certificates without additional mechanisms which pleases me. I am not going to \
claim this makes sense in your environment. You could follow their docs or get them out of traefik's acme.json. \
_It's important that TRMM get valid SSL certificates for all 3 domains for this, but it's not important exactly how._ \
If you're thinking that you "oughtta be able to just" then give it a shot. 

## Setting up the environment

We're going to put TRMM in its own Docker stack. Your ```Setting up the environment``` command block should resemble \
something like:

```
sudo mkdir -p /opt/stacks/tacticalrmm/
sudo chown -R 1000:1000 /opt/stacks/tacticalrmm/
/opt/stacks/tacticalrmm/
wget https://raw.githubusercontent.com/amidaware/tacticalrmm/master/docker/docker-compose.yml
wget https://raw.githubusercontent.com/amidaware/tacticalrmm/master/docker/.env.example
mv .env.example .env
```

When you use docker compose you can provide ```-f _file_``` to specify another compose file, \
which will be a "project" or stack, e.g. ```sudo docker compose -f /opt/stacks/tacticalrmm/docker-compose.yml pull``` \
which you may as well start now if you don't have an existing installation. 

## Base64 encoding certificates to pass as env variables

Skip this if you are using my docker-compose and certbot approach above. If you are \
going this route then you'll have to set up something to watch source cert files, \
remove the existing entries for these variables (sed), and then run those commands \
to concatenate the renewed certificates on. There is a traefik cert dumper and \
we will create Host rules that likely configure your configured resolver to renew the \
certificates as needed after watching acme.json for changes. 

## Starting the Environment

We're not done, but you can bring the containers online with \
```sudo docker compose -f /opt/stacks/tacticalrmm/docker-compose.yml up -d```. 

Visiting https://rmm.example.com/ won't work yet, but you should now be able to visit \
https://rmm.example.com:8443/login with working SSL. 

## TRMM and Traefik Both Apply SSL

We're going to make a special and unique service @file for rmm. We can provide a url \
with https schema and the port that was available for binding this way. 

```fileConfig.yml```
```
services:
  rmmsnowflake:
    loadBalancer:
      servers:
        - url: "https://rmm.example.com:8443/"
```

This does not need to be repeated for api and mesh. The requested url hostname is not modified by traefik to \
reflect the service loadBalancer and the IP address is the same. 

Add the following to your docker-compose.yml's traefik service's label section:
```
      traefik.http.routers.apissl.rule: Host(`api.$DOMAIN`) # for TRMM
      traefik.http.routers.apissl.tls: true
      traefik.http.routers.apissl.tls.certresolver: myresolver
      traefik.http.routers.apissl.service: rmmsnowflake@file
      traefik.http.routers.rmmssl.rule: Host(`rmm.$DOMAIN`) # for TRMM
      traefik.http.routers.rmmssl.tls: true
      traefik.http.routers.rmmssl.tls.certresolver: myresolver
      traefik.http.routers.rmmssl.service: rmmsnowflake@file
      traefik.http.routers.meshssl.rule: Host(`mesh.$DOMAIN`) # for TRMM
      traefik.http.routers.meshssl.tls: true
      traefik.http.routers.meshssl.tls.certresolver: myresolver
      traefik.http.routers.meshssl.service: rmmsnowflake@file
```

Now rebuild traefik and confirm everything looks good. 

Requests come in, Traefik decrypts, matches to rmmsnowflake@file, heads over to trmm-nginx container with the original \
hostname for an encrypted ask of the answer possibly with a different certificate, then comes on back and answers \
the original request. I think. All I really know is it works. 

### You may also find answers in the official [documentation](https://docs.tacticalrmm.com)
