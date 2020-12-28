[![](https://img.shields.io/docker/pulls/alexanderkrause/rpi-letsencrypt-nginx-proxy-companion.svg)](https://hub.docker.com/r/alexanderkrause/rpi-letsencrypt-nginx-proxy-companion "Click to view the image on Docker Hub")

This is a [**fork**](https://github.com/Alexander-Krause/rpi-docker-letsencrypt-nginx-proxy-companion) that enables usage on a armhf architecture (tested on RPI 3). Have a look at Yves Blusseau's [original](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion) repository and README. The following part does not include all available options of the original project.

### Why do you want to use this?
Reasons and examples for using a reverse proxy are discussed [by Jason Wilder](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/) or [here](https://stackoverflow.com/a/366212/3250397).
With this companion container for automatically creating/renewing *Let's Encrypt certificates* you can host and expose your dockerized TLS-secured applications on a Raspberry Pi. Examples:

* [Home Assistant](https://www.home-assistant.io/) with a custom nginx template for your sweet home automation (*tested*)
* [Nextcloud](https://github.com/nextcloud/docker) with [Passman](https://github.com/nextcloud/passman) extension and [MySQL](https://github.com/hypriot/rpi-mysql) (*tested*)
* [Nginx](https://github.com/armhf-docker-library/nginx) hosting your web sites (*tested*)
* Own Mailserver, e.g. [tomav/docker-mailserver](https://github.com/tomav/docker-mailserver), [hardware/mailserver](https://github.com/hardware/mailserver) or [Poste.io](https://poste.io/) (*not tested*)

### How to use
Built image is hosted on [Dockerhub](https://hub.docker.com/r/alexanderkrause/rpi-letsencrypt-nginx-proxy-companion). Declare three writable volumes for the [rpi-nginx-proxy](https://github.com/Alexander-Krause/rpi-nginx-proxy) container:
* `/etc/nginx/certs` to create/renew Let's Encrypt certificates
* `/etc/nginx/vhost.d` to change the configuration of vhosts (needed by Let's Encrypt)
* `/usr/share/nginx/html` to write challenge files.

Exemplary usage:

* First start nginx with the 3 volumes declared (you need to build this image as shown in the respective repository):
```bash
$ docker run -d -p 80:80 -p 443:443 \
    --name nginx-proxy \
    -v /path/to/certs:/etc/nginx/certs:ro \
    -v /etc/nginx/vhost.d \
    -v /usr/share/nginx/html \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    --label com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy \
    alexanderkrause/rpi-nginx-proxy
```
The "com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy" label is needed so that the letsencrypt container knows which nginx proxy container to use.

* Second start this container:
```bash
$ docker run -d \
    --name nginx-letsencrypt \
    -v /usr/ssl:/etc/nginx/certs:rw \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    --volumes-from nginx-proxy \
    alexanderkrause/rpi-letsencrypt-nginx-proxy-companion
```

Then start any containers you want proxied with a env var `VIRTUAL_HOST=yourdomain.com`, e.g.

```bash
$ docker run -d \
    --name example-app \
    -e "VIRTUAL_HOST=example.com" \
    -e "LETSENCRYPT_HOST=example.com" \
    -e "LETSENCRYPT_EMAIL=foo@bar.com" \
    tutum/apache-php
```

#### Regarding Certificate Aquiring
The acquiring of a certificate requires a nginx-reverse-proxy container with a mapping of the default ports, i.e.,  '80:80' and '443:443', as shown above. If you don't want to expose those ports, you need to apply a workaround:

Initially start a nginx-reverse-proxy container as shown below with those port mappings, then shutdown all three containers (reverse-proxy, companion and your application). Remove the reverse-proxy container and start a new one with your desired port mappings, e.g. '5050:80' and '5060:443'. Finally, start the companion and your application container.

### How to build the image yourself
1. Clone this repository `$ git clone https://github.com/Alexander-Krause/rpi-docker-letsencrypt-nginx-proxy-companion.git`
2. `$ cd rpi-docker-letsencrypt-nginx-proxy-companion`
3. `$ docker build -t alexanderkrause/rpi-docker-letsencrypt-nginx-proxy-companion:latest .`

### DynDNS
Tested with [duckdns](https://www.duckdns.org) as DynDNS provider. Configure the update url in your router or device (with [ddclient](https://sourceforge.net/p/ddclient/wiki/Home/)) and (!) enable port forwarding (e.g. 443 of your Pi / docker container) in your router. Do the steps from above and enter `yourducksubdomain.duckdns.org` in `VIRTUAL_HOST` and `LETSENCRYPT_HOST`.

