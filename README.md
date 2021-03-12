# Docker Exim Relay Image

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![Layers](https://images.microbadger.com/badges/image/industrieco/exim-relay.svg)](https://microbadger.com/images/industrieco/exim-relay/) [![GitHub Tag](https://img.shields.io/github/tag/industrieco/docker-exim-relay.svg)](https://registry.hub.docker.com/u/industrieco/docker-exim-relay/) [![Docker Pulls](https://img.shields.io/docker/pulls/industrieco/exim-relay.svg)](https://registry.hub.docker.com/u/industrieco/exim-relay/)

A light weight Docker image for an Exim mail relay, based on the official Alpine image.

For extra security, the container runs as exim not root.

Official info about running Exim without privilege:

https://www.exim.org/exim-html-current/doc/html/spec_html/ch-security_considerations.html

Docker Hub:

https://hub.docker.com/r/laimison/exim-relay

## Docker

### Default setup

This will allow relay from all private address ranges and will relay directly to the internet receiving mail servers

```
docker run \
       --name smtp \
       --restart always \
       -h my.host.name \
       -d \
       -p 25:8025 \
       laimison/exim-relay
```

### Smarthost setup

To send forward outgoing email to a smart relay host

```
docker run \
       --restart always \
       -h my.host.name \
       -d \
       -p 25:8025 \
       -e SMARTHOST=some.relayhost.name \
       -e SMTP_USERNAME=someuser \
       -e SMTP_PASSWORD=password \
       laimison/exim-relay
```

## Docker Compose

```
version: "2"
  services:
    smtp:
      image: laimison/exim-relay
      restart: always
      ports:
        - "25:8025"
      hostname: my.host.name
      environment:
        - SMARTHOST=some.relayhost.name
        - SMTP_USERNAME=someuser
        - SMTP_PASSWORD=password
```

## Other Variables

###### LOCAL_DOMAINS

* List (colon separated) of domains that are delivered to the local machine
* Defaults to the hostname of the local machine
* Set blank to have no mail delivered locally

###### RELAY_FROM_HOSTS

* A list (colon separated) of subnets to allow relay from
* Set to "\*" to allow any host to relay - use this with RELAY_TO_DOMAINS to allow any client to relay to a list of domains
* Defaults to private address ranges: 10.0.0.0/8:172.16.0.0/12:192.168.0.0/16

###### RELAY_TO_DOMAINS

* A list (colon separated) of domains to allow relay to
* Defaults to "\*" to allow relaying to all domains
* Setting both RELAY_FROM_HOSTS and RELAY_TO_DOMAINS to "\*" will make this an open relay
* Setting both RELAY_FROM_HOSTS and RELAY_TO_DOMAINS to other values will limit which clients can send and who they can send to

###### RELAY_TO_USERS

* A whitelist (colon separated) of recipient email addresses to allow relay to
* This list is processed in addition to the domains in RELAY_TO_DOMAINS
* Use this for more precise whitelisting of relayable mail
* Defaults to "" which doesn't whitelist any addresses

###### SMARTHOST

* A relay host to forward all non-local email through

###### SMTP_USERNAME

* The username for authentication to the smarthost

###### SMTP_PASSWORD

* The password for authentication to the smarthost - leave this blank to disable authenticaion

## Docker Secrets

The smarthost password can also be supplied via docker swarm secrets / rancher secrets.  Create a secret called SMTP_PASSWORD and don't use the SMTP_PASSWORD environment variable

## Debugging

The logs are sent to /dev/stdout and /dev/stderr and can be viewed via docker logs

```shell
docker logs smtp
```

```shell
docker logs -f smtp
```

Exim commands can be run to check the status of the mail server as well

```shell
docker exec -ti smtp exim -bp
```

## Goals

Requirements

* hostname - sender hostname (e.g. exim, example.com)

* server - same as hostname if server is hostname (not IP) / server is the relay server to send mails through

* port - port (e.g. 10025, 587) / port is the port used by the relay server to accept connections

* networks - networks is the list of allowed networks to send mail from (e.g. "127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16")

* senderDomains - senderDomains is the list of allowed domains to send mails from; set to `'*'` for any (e.g. `"*"`)

* recipientDomains - recipientDomains is the list of allowed domains to send mails to; leave blank for any. (e.g. example.local, example.com, "" )

Not needed

* username - username is the user to authenticate with the relay server if needed

* password - password is the password used to authenticate with the relay server if needed

## Common

### Push image to Docker Hub

```

docker login --username=laimison

hub_push_exim_relay () {
if echo $1 | grep -q '[a-zA-Z0-9]'; then tag=$1; else tag=latest; fi
docker build -t laimison/exim-relay .
docker images | grep laimison/exim-relay | head -n 1 | awk '{print $3}' | while read -r value; do docker tag $value laimison/exim-relay:"${tag}"; done
docker push laimison/exim-relay:"${tag}"
}

hub_push_exim_relay

hub_push_exim_relay 0.1

```

### Gain root while working

```
docker exec -u root -it smtp sh
```

### Send test email

```
docker run --rm -it ubuntu:18.04 bash
apt update && apt install ssmtp vim iputils-ping netcat -y
# docker host's IP
nc -v 192.168.2.239 1025
# or whatever your laptop IP is
nc -v 192.168.2.182 1025
vi /etc/ssmtp/ssmtp.conf
# mailhub=192.168.2.182:1025
read receiver
echo -e 'Subject: test\n\nTesting ssmtp' | sendmail -v $receiver
```

### Helm

```
cd helm
mkdir -p tmp
helm template -f values.yaml --output-dir tmp/exim-relay .
```
