# Domain Name System

- [How it works](#how-it-works)
    - [Prerequisites](#prerequisites)
    - [Docker](#docker)
- [Security](#security)
- [Resources](#resources)

# How it works

So you can easily learn it off some AI tool but i'm not doing this for you, i'm doing this for me. 

Whats in a Domain? Well [google.com](www.google.com) is a domain.topleveldomain. So the domain is `google` and the top level domain is `www`. [subsearch.google.com](#) contains the subdomain `subsearch`. Behind any of these FQDNs (which may also be treated as Universal Resource Locators - URLs) are IP addresses. Domains are human readable fronts for these IP addresses. IP addresses are `X.X.X.X/port`. DNS exists so humans can type out words, or more importantly **brands** instead of `45.62.3.124/80`. 

Whats more, say Google have more than one IP that can host their search engine at the eponymous address. Well they can switch the DNS over to another IP and Joe Schlomo won't know what happened. 

You can do this locally. Your own computer (likely) will be able to host `localhost` through your `/etc/hosts` file. So you can spoof Facebook on your own machine and host your *next killer app* locally with a **brand** name. 

In other situations you may dynamically provision many nodes or addresses and want each machine to have a subdomain and such automatically added. Things like `coreDNS` in kubernetes take care of this. We can achieve similar setups in a Docker environment with a DNS solution like `dnsmasq`.

## Prerequisites

We already have the images locally (or if not, just pull them from your docker repo) so we'll go with a slightly edited version of the compose yaml; removing the mapped volumes. So with `docker-compose up --no-build` we should have what we need. 

<span style="color: red;">**BUG:** </span> I had to run these commands after `docker-compose up --no-build`: `docker exec <image-id> python manage.py migrate && docker exec <image-id>  python manage.py migrate --run-syncdb`. I think it's that while the backend depends on the db to be ready, application-wise it may not have been and thus, couldn't migrate the models. 

## Docker

We'll change our compose file to now specifically name the Docker network we'll use, `web`. `external: false` is not necessary here, it's the default setting. It means if we're not using an 'outside' network, we're having Docker make its own called web. Here are the steps to now add DNS to our setup. 

1. Add a reverse proxy

We'll discuss this in another repo, but we'll add the following to our compose and include the `conf.d` file also.
```
  proxy:
    image: dockware/proxy:latest
    container_name: proxy
    ports:
      - "80:80"
      - "443:443"
    networks:
      - web
    volumes:
      - "./proxy.conf:/etc/nginx/conf.d/default.conf"
    depends_on:
      - redis
      - backend
      - frontend  
      - db
      - celery
```
```
server {
    listen          80;
    server_name     api.project.dev;
    return 301      https://$host$request_uri;
}

server {
    listen           443 ssl;
    server_name      api.project.dev;
    
    ssl_certificate /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;
    
    location / {
        proxy_set_header Host $host;
        proxy_pass https://api;
    }
}
```
With this we can still access the app on `localhost`. We now want to start using our `*.project.dev` domains. Before we do, we have to make sure our host will allow port 53 to be used for DNS. For me that meant changing the settings on Docker Desktop to disable 'Use host networking for UDP'. 

Now we need to tell the host to use the DNS we're about to setup, but **only for requests to project.dev**. We run these commands on a mac/linux:
```
sudo mkdir -p /etc/resolver
sudo rm -rf /etc/resolver/project.dev
sudo sh -c 'echo "nameserver 127.0.0.1" >> /etc/resolver/project.dev'
```
Next we can add the DNS server to our compose file:
```
  dnsmasq:
    image: jpillora/dnsmasq
    ports:
      - "53:53/udp"
    networks:
      - web
    depends_on:
      - proxy
```
Now we fix the reverse proxy to a static IP to do all the routing for us predictably. We add this to our network section:
```
    ipam:
      config:
        - subnet: 10.0.0.0/24
```
And set the IP address for our proxy service:
```
proxy:
  image: dockware/proxy:latest
  ...
  networks:
    web:
      ipv4_address: 10.0.0.100
```
Make a `dnsmasq.conf` with its only contents being: `address=/project.dev/10.0.0.100` and mount that file into the dnsmasq service:
```
  dnsmasq:
    image: jpillora/dnsmasq
    ...
    volumes:
      - "./dnsmasq.conf:/etc/dnsmasq.conf"
```
This also meant I had to move the networks section to the top of file as we were defining an IP in a subnet that didn't exist in the `proxy` service. Had to make it first at top of file so the config was recognized by the time we got to the proxy service. 

## Resources

[Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/azure-dns?tabs=ubuntu) <br>
[DNS Server on own machine](https://www.howtogeek.com/devops/how-to-run-your-own-dns-server-on-your-local-network/) <br>
[DNS Masq](https://boxblinkracer.com/blog/docker-dnsmasq)