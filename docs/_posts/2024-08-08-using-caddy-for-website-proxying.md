---
layout: post
title:  "Using Caddy for Website Proxying"
date:   2024-08-08 13:40:00 +0300
categories: deployment caddy
---

Not long ago, I wrote a small library in `Go` for testing HTTP requests, something very similar to `httpbin`, but with the possibility of using it inside `httptest` package. `httptest` allows you to create a test web server that handle test requests so you don't depend on an external server while testing.

After some time, I wrote a working server based on this library.
Then it occurred to me that my library lacked some features, including determining the HTTP protocol version.

After I felt everything was ready, it was time to deploy this server.
For a web server to work over the `HTTP/2.0` protocol, you need **trusted** TLS certificates. Currently, you can create them for free using services like **Let's Encrypt**.

> A slight digression. For the last few years, I haven't particularly kept up with new developments in web servers (and not only them). Before this moment, I had used **nginx** and a couple of times **haproxy**, so **caddy** passed me by. And now a great opportunity came up to fix this situation – I  decided to use **caddy** as a proxy for my server.

Caddy can automatically create certificates and proxy` HTTP/1.1`, `HTTP/2.0`, and `HTTP/3` (and `H2C`) requests. And although this is far from all that caddy is capable of, I am mostly interested in these two points – certificates and proxying.
From the beginning, I decided to use` docker compose` to run caddy and my service (**httpbulb**) – since both have docker images. 
It is possible to expose the server to the internet to create its certificates but I'm going to reuse caddy's certificates for my server backend. This seems to be the simplest way for me.


In my server, there's nothing special, except for what it does, but if you're interested, here's the [code](https://github.com/niklak/httpbulb/blob/main/cmd/bulb/main.go).

I also decided to decline the redirect from port `80` to `443`. I can't imagine there is any significant reason for such redirects nowadays, besides good manners, of course. But that doesn't bother me much. I prefer to keep it simple: if there are certificates, the server serves only `https`, if not, then `http`.

Here's what the `docker-compose.yml` turned out to be:

```docker-compose.yml
name: caddyproxy
services:
  caddy:
    # There is nothing special here, everything just like in caddy's documentation
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

  web:
    image: ghcr.io/niklak/httpbulb:latest
    restart: unless-stopped
    volumes:
      # I want to use caddy's certificates, so I link caddy's data directory with certificates.
      - caddy_data:/data
      # But, to be able to read these files, I need root permissions.
    user: root
    ports:
      # I don't want to expose 8080 port to the world. It can be accessed only from the localhost.
      - 127.0.0.1:8080:8080
    environment:
      # specify server address, in my case it's just a port
      - SERVER_ADDR=:8080
      # Setting cert paths from caddy for httpbulb server
      - SERVER_CERT_PATH=/data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/httpbulb.net/httpbulb.net.crt
      - SERVER_KEY_PATH=/data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/httpbulb.net/httpbulb.net.key
      - SERVER_READ_TIMEOUT=120s
      - SERVER_WRITE_TIMEOUT=120s

volumes:
  caddy_data:
  caddy_config:

```

Also, for caddy to work, you need a `Caddyfile`.

Initially, it looked like this:

```
{
  # I don't require redirects from 80 to 443
  auto_https disable_redirects
}

httpbulb.net:443 {
  # caddy will accept connections on 443 port and dial them to 8080 port 
  reverse_proxy * {
    to web:8080
    transport http {
      # tell that we are using https 
      tls
      # because I use certificates for httpbulb.net, and my backend server is called web, I must set tls_server_name MANUALLY.
      tls_server_name httpbulb.net
    }
  }
}

```

After that, I ran the services with `docker compose up -d` and realized it was not exactly what I needed. The browser makes requests over `h3`, and caddy sends requests to the backend over `h2` (because my server does not support `h3`). Therefore, I need to completely disable  caddy's `h3` support until I add `h3` support for **httpbulb**.

I slightly modified the Caddyfile:

```
{
  auto_https disable_redirects

  servers :443 {
    # now caddy will perform only HTTP/1.1 and HTTP/2.0 requests
    protocols h1 h2
  }
}
```

I restarted and checked again – still not what I needed. If I send a regular `HTTP/1.1` request, my service shows proto `HTTP/2.0` – because caddy still sends requests to the backend over `HTTP/2.0`. **I need a clear match**: if the user-agent sent an `HTTP/1.1` request, then an` HTTP/1.1` request should be sent to the backend as well. The same applies to `HTTP/2.0`.


After digging through the caddy documentation, I realized that this can be solved using [request matchers](https://caddyserver.com/docs/caddyfile/matchers).

The final version looks like this:

```
{
  auto_https disable_redirects

  servers :443 {
    protocols h1 h2
  }
}

httpbulb.net:443 {
  # declaring a named matcher for HTTP/1.1
  @http1.1 protocol http/1.1

  # declaring a named matcher for HTTP/2.0
  @http2 protocol http/2+

  # if request's protocol matches HTTP/1.1 then
  # caddy will send HTTP/1.1 request to the backend
  reverse_proxy @http1.1 {
    to web:8080
    transport http {
      tls
      tls_server_name httpbulb.net
      # serve only HTTP/1.1
      versions 1.1
    }
  }

  # if request's protocol matches HTTP/2.0 then
  # caddy will try to send an HTTP/2.0 request to the backend
  reverse_proxy @http2 {
    to web:8080
    transport http {
      tls
      tls_server_name httpbulb.net
      # serve HTTP/1.1 and HTTP/2.0
      versions 1.1 2
    }
  }
}

```

After these changes, everything worked – `HTTP/1.1` requests reached the backend as `HTTP/1.1`, and` HTTP/2.0` as `HTTP/2.0`.
In the end, I was satisfied choosing **caddy**. There are some things that I didn't like (*json config*, lack of *rate limit* in the standard build), but they don't affect anything. Caddy is indeed a strong and convenient tool for serving websites and services.