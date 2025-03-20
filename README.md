# Introduction to Traefik - Part Three

a blog post series about my homelab.

![Traefik](https://balaskas.gr/blog/fp-content/images/d23d82a2.png)

check here for [Introduction to Traefik - Part Two](https://github.com/ebal/traefik-intro-02)
check here for [Introduction to Traefik - Part Four](https://github.com/ebal/traefik-intro-04)

## Part Three

In this blog post series, I will connect several docker containers and a virtual machine behind the Traefik reverse proxy on my homelab, and set up Let's Encrypt for TLS.

In this article, I will try to connect a virtual machine to the Traefik reverse proxy. In Linux, Docker containers and virtual machines (VMs) run on different networks due to the way their networking is set up. To enable communication between Docker containers and VMs, we need to configure somekind of network bridging, port forwarding, or use a common network interface that allows them to communicate with each other. To simplify the setup, I will try to put Traefik docker container to a common network with the virtual machine.

> **Disclaimer**: This homelab is intended for testing and learning purposes, as are the services we'll explore. Make sure it fits your needs; I'm not suggesting you copy/paste everything here as-is. In the end, I may decide to delete this homelab and create a new one! But for now, let's enjoy this journey together. Thank you!

I've also made a short video to accompany this blog post:

[![Watch the video](https://img.youtube.com/vi/9FvMLj59P5A/maxresdefault.jpg)](https://youtu.be/9FvMLj59P5A)

## Virtual Machine

I use Qemu/KVM (kernel virtual machine) in my home lab. I also use terraform with libvirtd to automate my entire setup. That said, this post is not about that !

For the purpose of this article, I created an ubuntu 24.04 LTS running transmission service. It's IP is: `192.168.122.79` and listens to TCP Port: `9091`. Transmission also has configured with a Basic Auth authentication mechanism which username and password are the defaults: `transmission:transmission`.

## Setup diagram

something similar to the below scheme

```
                       ┌─────┐
                       │     │
                       │     │      192.168.122.x:9091
                       │     │       with Basic Auth
    ┌────┐             │     │        ┌───────┐
    │    │             │     │        │       │
    │    │  ─────────► │     ├───────►│       │
  ┌─└────┘─┐           │     │        │       │
  └────────┘           │     │        └───────┘
  192.168.1.3          │     │           VM
                       │     │
                       │     │        ┌──┐┌──┐
                       │     ├───────►│  ││  │whoami
                       └─────┘        └──┘└──┘   172.19.0.x
                                      ┌──┐┌──┐
                       Traefik        │  ││  │
                                      └──┘└──┘
                                       docker
                                     containers
```

## Traefik Network Mode

By default the network mode in docker compose is **bridge** mode which isolates the docker containers from the host. In the bridge mode our docker containers can communicate with each other directly.

But we need to either bridge our VM network (192.168.122.x/24) to the 172.19.0.x/24 network, or—what seems easier to me—change Traefik's network mode from bridge to host.

In docker compose yaml file, in traefik service we need to make two changes:

first add the host network mode:

```yaml
    # Very important in order to access the VM
    network_mode: host
```

and by using **host** we can now remove any port declaration

`remove`

```yaml
    ports:
      # The Web UI (enabled by --api.insecure=true)
      - 8080:8080
      # The HTTP port
      - 80:80
```

so our **docker-compose.yml** now looks like:

```yaml
---
services:
  traefik:
    image: traefik:v3.3
    container_name: traefik
    hostname: traefik
    env_file:
      - path: ./default.env
        required: true
    restart: unless-stopped
    volumes:
      # The configuration directory for traefik
      - ./traefik:/etc/traefik
      # So that Traefik can listen to other Docker Containers
      - /var/run/docker.sock:/var/run/docker.sock:ro
    # Add health check
    healthcheck:
       test: ["CMD", "traefik", "healthcheck"]
      interval: 30s
      retries: 3
      timeout: 10s
      start_period: 10s
    # Very important in order to access the VM
    network_mode: host

  # A container that exposes an API to show its IP address
  whoami:
    image: traefik/whoami
    container_name: whoami
    hostname: whoami
    depends_on:
      traefik:
        condition: service_healthy # Wait for Traefik to be healthy
    labels:
          # To enable whoami to Traefik
        - "traefik.enable=true"
          # Declare the host rule for this service
        - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
          # Declare the EntryPoint
        - "traefik.http.routers.whoami.entrypoints=web"

```

Start services:

```bash
docker compose up -d
```

### Test whoami

let us test now if whoami service works

```bash
curl -H Host:whoami.localhost http://127.0.0.1
```

output:

```
Hostname: whoami
IP: 127.0.0.1
IP: ::1
IP: 172.19.0.2
RemoteAddr: 172.19.0.1:43968
GET / HTTP/1.1
Host: whoami.localhost
User-Agent: curl/8.12.1
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 127.0.0.1
X-Forwarded-Host: whoami.localhost
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik
X-Real-Ip: 127.0.0.1
```

okay, seems it works as before.

http://whoami.localhost/

## Transmission

as mentioned above, our transmission application runs on the virtual machine. It is protected by a Basic Authentication setup and listens on a TCP port.

Let's test the connection for the command line

```bash
curl 192.168.122.79:9091
```

result is:

```
<h1>401: Unauthorized</h1>
```

To add **Basic Auth** to curl is simple:

```bash
curl -u transmission:transmission 192.168.122.79:9091
```

the output is:

```
<h1>301: Moved Permanently</h1>
```

we can add `-v` (verbose) to see more details:

```bash
curl -v -u transmission:transmission 192.168.122.79:9091
```

full output:

```
*   Trying 192.168.122.79:9091...
* Connected to 192.168.122.79 (192.168.122.79) port 9091
* using HTTP/1.x
* Server auth using Basic with user 'transmission'
> GET / HTTP/1.1
> Host: 192.168.122.79:9091
> Authorization: Basic dHJhbnNtaXNzaW9uOnRyYW5zbWlzc2lvbg==
> User-Agent: curl/8.12.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 301 Moved Permanently
< Server: Transmission
< Access-Control-Allow-Origin: *
< Location: /transmission/web/
< Date: Thu, 27 Feb 2025 15:47:02 GMT
< Content-Length: 31
< Content-Type: text/html; charset=ISO-8859-1
<
* Connection #0 to host 192.168.122.79 left intact
<h1>301: Moved Permanently</h1>
```

The **Location** section is interesting: `Location: /transmission/web/`
we will keep this in mind.

### Authorization header

by looking very careful the above output, we see that curl uses the Authorization header for Basic Auth. This is also interesting and we can use this.

Let's try the command:

```bash
curl -H "Authorization: Basic dHJhbnNtaXNzaW9uOnRyYW5zbWlzc2lvbg==" http://192.168.122.79:9091/
```

output:

```
<h1>301: Moved Permanently</h1>
```

okay !

So, another way to access transmission is via Authorization header. Curl sends the credentials through base64 encoding, which can be reproduced by

```bash
echo -n "transmission:transmission" | base64
```

and we verify the output:

```
dHJhbnNtaXNzaW9uOnRyYW5zbWlzc2lvbg==
```

## Traefik file provider

For the purpose of this lab, we want to access the application on the VM from localhost without providing any credentials, with Traefik handling everything.

```
  http://localhost/transmission
                  │
                  │
                  ▼
           ┌──────────────┐
           │              │
           │   Traefik    │
           │      &       │
           │  middleware  │
           └──────┬───────┘
                  │
                  │
                  ▼
                ┌────┐
                │ VM │
                └─┬──┘
              ┌───┴────┐
              └────────┘
  http://(internal IP:PORT)/transmision/
                  +
          Authorization header
```

To do that, we need to introduce a PathPrefix Rule to Traefik so it redirects every request for `/transmission` to the VM. And what a better place to introduce the file provider on our static Traefik configuration

```yaml
    # Enable file provider
    file:
        directory: /etc/traefik/dynamic/
        watch: true
```

under our docker provider.

so the entire **traefik/traefik.yml** should look like:

```yaml
# The /ping health-check URL
ping:

# API and dashboard configuration
api:
    insecure: true

# Debug log
log:
    filePath: /etc/traefik/traefik.log
    level: DEBUG

# Enable EntryPoints
entryPoints:
    web:
        address: ":80"
        reusePort: true

# Providers
providers:
    # Enable docker provider
    docker:
        exposedByDefault: false

    # Enable file provider
    file:
        directory: /etc/traefik/dynamic/
        watch: true

```

### Dynamic Directory

I am sure you have already noticed that we also have introduced a new directory under our Traefik folder and instructed Traefik to **watch** it. This is extremely useful because Traefik will automatically reload any configuration in that folder without the need to restart (stop/start or down/up) the Traefik service.

As this change is on static configuration, we need to stop/start the services:

```bash
docker compose down
```

Create the dynamic directory:

```bash
mkdir -pv ./traefik/dynamic/
ls    -la ./traefik/dynamic/
```

and

```bash
docker compose up -d
```

### Traefik Design

To help you understand how traefik works,

```
                     HTTP
┌───────────┐      ┌──────┐   ┌──────────┐   ┌───────┐
│           │      │      │   │          │   │       │
│EntryPoints│ ───► │Routes│──►│Middleware│──►│Service│
│           │      │      │   │          │   │       │
└───────────┘      └──────┘   └──────────┘   └───────┘
```

We have alredy explained EntryPoints on our previous article, so we are going to focus on routers, middlewares and services.

a better image ![Traefik Architecture](storage/Traefik_Architecture.png)

### Traefik Services

To avoid any complicated explanation, the Traefik Service is the final destination of an HTTP request.

For our example, should look like the below config. Please be aware, the `<service name>` is a placeholder for later.

```yaml
http:
  services:
    <service-name>:
      loadBalancer:
        servers:
          - url: 'http://192.168.122.79:9091'
```

### Traefik Middlewares

As we already mentioned above, we need to send login credentials to transmission. Be aware `<middleware name>` is a place holder for later.

```yaml
http:
  middlewares:
    <middlewar-name>:
      headers:
        customRequestHeaders:
          Authorization: "Basic dHJhbnNtaXNzaW9uOnRyYW5zbWlzc2lvbg=="

```
### Traefik Routes

Traefik HTTP Routes are part of Traefik's dynamic configuration and define how HTTP requests are handled and routed to the correct services.

Which means the **routers** is the component that connects everything (EntryPoint, Middleware, and Service) together. This is also where we add our PathPrefix rule for the Transmission location.

Be aware `<router|service|middleware name>` is a place holder for later.

eg.

```yaml
http:
  routers:
    <router-name>:
      entryPoints: web
      service: <service-name>
      rule: PathPrefix(`/transmission`)
      middlewares:
        - <middleware-name>
```

## Traefik dynamic configuration

We are ready to pull things together.

Create a file named transmission yaml under the dynamic configuration directory:

**./traefik/dynamic/transmission.yml**

```yaml
http:

  middlewares:
    middleware-transmission:
      headers:
        customRequestHeaders:
          Authorization: "Basic dHJhbnNtaXNzaW9uOnRyYW5zbWlzc2lvbg=="

  routers:
    router-transmission:
      entryPoints: web
      service: service-transmission
      rule: PathPrefix(`/transmission`)
      middlewares:
        - middleware-transmission

  services:
    service-transmission:
      loadBalancer:
        servers:
          - url: 'http://192.168.122.79:9091'

```

**NO need** to restart our services with dynamic configuration!

## Test Traefik new transmission route

from command line

```bash
curl localhost/transmission/
```

output:

```
<h1>301: Moved Permanently</h1>
```

traefik dashboard

![Traefik_transmission](storage/Traefik_transmission.png)

from browser

http://localhost/transmission/

![Transmission](storage/Transmission.png)

That's It !

```bash
docker compose down
```
