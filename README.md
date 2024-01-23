# Geo-filtering for self-hosted DNS server

## Introduction

I've been self-hosting private `Adguard Home` DNS server for a while, I've noticed recently huge number of DNS requests targeting common domains like Cisco, Adobe, Atlassian ... 
which clearly means that I'm under **DNS Amplification** attack.

I noticed first that my connection was very slow, until it's almost stopped working! 

As my router is relatively old, It does not have the option to use DoH directly, I need to put my server IP directly.

Also, I don't have the ability to set a fixed IP to my internet, as it's not permitted in my country right now.

The target is to make geo-blocking, block every other country except for my country.

## Solutions 

I've tried some solutions : 

1. Add all country ranges to client blacklist/whitelist on Adguard Home access configuration, this solution didn't work because the attacker origins are almost everywhere on the planet, and country ip ranges are not accurate enough (I tried, but I actually blocked myself).
2. use firewall rules to block IPs, It actually has the same problems of the first solution, and I'm not an expert enough to use enterprise level WAF, I use Cloudflare WAF, but sadly It works only on domains, It has nothing to do with requests pointed at server IP.

## workaround

I had an idea : *what if I redirected DNS request to be DoH request, so I'll have the ability then to geo-filter it ?*

And I found fancy solution for this problem using [Control-D](https://github.com/Control-D-Inc/ctrld), the main idea is that I convert normal UDP request (DNS query) into DoH query that uses my domain, Then I added rule to Cloudflare WAF to accept requests only from my country.

`ctrld` should listen to port 53, so he can re-route all requests on this port.

## steps 

1. first, you should pull all images that you will use in the deployment, in my case :

   ```bash
   docker pull adguard/adguardhome:latest
   docker pull controldns/ctrld:latest
   ```
   You need to do this even If you're planning to use docker compose, because of the next step.

2. stop the `systemd-resolved` service, this is the default DNS system resolver, and it's listening to port 53, you won't be able to start before this step, note also that if you stopped it before pulling image, docker won't be able to resolve docker hub IP to pull the image
     ```bash
     sudo systemctl stop systemd-resolved
     ```

3. create a new `docker-compose.yml` file and add the following content to it :
   ```yaml
    version: "2"
    services:
       adguardhome:
         image: adguard/adguardhome:latest
         container_name: adguardhome
         ports:
           - 3000:3000/tcp
           - 80:80/tcp
           - 443:443/tcp
         volumes:
           - ./work:/opt/adguardhome/work
           - ./conf:/opt/adguardhome/conf
       ctrld:
         image: controldns/ctrld
         container_name: ctrld
         ports:
           - 53:53/tcp
           - 53:53/udp
         volumes:
           - ./controld:/config
         restart: always
         command: --config=/config/ctrld.toml
   ```
4. now, we need to add configuration type, we will use a modified version of it, first create the directory `./controld`
      ```bash
      mkdir controld
      ```
      then create `ctrld.toml` file using `nano ctrld.toml` and fill it with these configs :
   ```toml
   [listener]
   
     [listener.0]
       ip = ""
       port = 0
       restricted = false
   
   [network]
   
     [network.0]
       cidrs = ["0.0.0.0/0"]
       name = "Network 0"
   
   [service]
     log_level = "info"
     log_path = ""
   
   [upstream]
   
     [upstream.0]
       bootstrap_ip = "1.1.1.1"
       endpoint = "https://yourdns.com/dns-query"
       name = "my-dns"
       timeout = 5000
       type = "doh"
   ```
   Replace `yourdns` with your DNS server hostname.
6. start the containers with 
     ```bash
     docker compose up -d
     ```
7. On your account on Cloudflare, from Security -> WAF , add rule to block request if country is not equal to your country.

And that's it! I hope you find this information useful.


