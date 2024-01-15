# forced-dns-over-https
Force dns server to work on HTTP protocol
## Introduction

I've been self-hosting private `Adguard Home` dns server for a while, I've noticed recently huge number of dns requests targeting common domains like Cisco, Adobe, Atlassian ... 
which clearly means that I'm under **DNS flooding** attack.

I noticed first that my connection was very slow unitl it's almost stopped working! 

as my router is relatively old, It does not have the option to use DoH directly, I need to put my server ip directly.

also I don't have the ability to set a fixed ip to my internet, as it's not permitted in my country right now.

the target is to make geo blocking, block every other country except for my country.

## Solutions 

I've tried some solutions : 

1. Add all country ranges to client blacklist/whitelist on Adguard Home access configuration, this solution didn't work because the attacker origins is almost every where on the planet, and country ip ranges are not accurate enough (I tried but I actually blocked myself).
2. use firewall rules to block ips, It actually has the same problems of the first solution, and I'm not an expert enough to use enterprise level WAF, I use Cloudflare WAF, but sadly It works only on domains, It has nothing to do with requests pointed at server ip.

## workaround

I had an idea : *what if I redirected dns request to be DoH request so I'll have the ability then to geo-filter it ?*

And I found fancy solution for this problem using [docker-cloudflared](https://github.com/crazy-max/docker-cloudflared/tree/master) , the main idea is that I convert normal UDP request (DNS query) into DoH query that uses my domain, Then I added rule to Cloudflare WAF to accept requests only from my country.

`Cloudflared` should listen to port 53, so he can re-route all requests on this port.

## steps 

1. first, you should pull all images that you will use in the deployment, in my case :

   ```bash
   docker pull adguard/adguardhome:latest
   docker pull crazymax/cloudflared:latest
   ```
   you need to do this even If you're planning to use docker compose, because of the next step.

2. stop the `systemd-resolved` service, this is the default dns system resolver and it's listening to port 53, you won't be able to start before this step, note also that if you stopped it before pulling image, docker won't be able to resolve docker hub ip to pull the image
     ```bash
     sudo systemctl stop systemd-resolved
     ```

3. create new `docker-compose.yml` file and add the following content to it :
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
    
       cloudflared:
         image: crazymax/cloudflared:latest
         container_name: cloudflared
         ports:
           - 53:5053
         environment:
           - "TZ=$YOUR_TZ"
           - "TUNNEL_DNS_UPSTREAM=https://$YOUR_DOMAIN/dns-query"
         restart: always
   ```
     replace `YOUR_TZ` with your timezone, and `YOUR_DOMAIN` with your domain
     and then :
     ```bash
     docker compose up -d
     ```
4. On your account on Cloudflare, from Security -> WAF , add rule to block request if country is not equal to your country.

and that's it! I hope you find this information useful.


