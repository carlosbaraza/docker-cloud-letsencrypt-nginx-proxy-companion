# Docker Cloud Support

This fork adds Docker Cloud support to `docker-letsencrypt-nginx-proxy-companion`.

The only differences are in [`funcitons.sh`](app/functions.sh) to the `reload_nginx()`
function where the [`docker_kill`](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/blob/master/app/functions.sh#L75) function call is replaced by the use of the `docker-cloud` CLI to redeploy
the Service referenced by `NGINX_DOCKER_GEN_CONTAINER`. In using the `docker-cloud`
CLI, we need to add it to the `docker-letsencrypt-nginx-proxy-companion` image
via the `Dockerfile`.

See [this](https://blog.switchbit.io/developing-a-ghost-theme-with-gulp-part-5/) blog post for more context.

---

letsencrypt-nginx-proxy-companion is a lightweight companion container for the [nginx-proxy](https://github.com/jwilder/nginx-proxy). It allow the creation/renewal of Let's Encrypt certificates automatically. See [Let's Encrypt section](#lets-encrypt) for configuration details.

### Features:
* Automatic creation/renewal of Let's Encrypt certificates using original nginx-proxy container.
* Support creation of Multi-Domain ([SAN](https://www.digicert.com/subject-alternative-name.htm)) Certificates.
* Automatically creation of a Strong Diffie-Hellman Group (for having an A+ Rate on the [Qualsys SSL Server Test](https://www.ssllabs.com/ssltest/)).
* Work with all versions of docker.

***NOTE***: The first time this container is launch it generate a new Diffie-Hellman group file. This process can take several minutes to complete (be patient).

#### Usage

To use it with original [nginx-proxy](https://github.com/jwilder/nginx-proxy) container you must declare 3 writable volumes from the [nginx-proxy](https://github.com/jwilder/nginx-proxy) container:
* `/etc/nginx/certs` to create/renew Let's Encrypt certificates
* `/etc/nginx/vhost.d` to change the configuration of vhosts (need by Let's Encrypt)
* `/usr/share/nginx/html` to write challenge files.

Example of use:

* First start nginx with the 3 volumes declared:
```bash
$ docker run -d -p 80:80 -p 443:443 \
    --name nginx-proxy \
    -v /path/to/certs:/etc/nginx/certs:ro \
    -v /etc/nginx/vhost.d \
    -v /usr/share/nginx/html \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    jwilder/nginx-proxy
```

* Second start this container:
```bash
$ docker run -d \
    -v /path/to/certs:/etc/nginx/certs:rw \
    --volumes-from nginx-proxy \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    jrcs/letsencrypt-nginx-proxy-companion
```

Then start any containers you want proxied with a env var `VIRTUAL_HOST=subdomain.youdomain.com`

    $ docker run -e "VIRTUAL_HOST=foo.bar.com" ...

The containers being proxied must [expose](https://docs.docker.com/reference/run/#expose-incoming-ports) the port to be proxied, either by using the `EXPOSE` directive in their `Dockerfile` or by using the `--expose` flag to `docker run` or `docker create`. See [nginx-proxy](https://github.com/jwilder/nginx-proxy) for more informations. To generate automatically Let's Encrypt certificates see next section.

#### Separate Containers (recommended method)
nginx proxy can also be run as two separate containers using the [jwilder/docker-gen](https://github.com/jwilder/docker-gen)
image and the official [nginx](https://hub.docker.com/_/nginx/) image.

You may want to do this to prevent having the docker socket bound to a publicly exposed container service (avoid to mount the docker socket in the nginx exposed container). It's better in a security point of view.

To run nginx proxy as a separate container you'll need:

1) To mount the template file [nginx.tmpl](https://github.com/jwilder/nginx-proxy/blob/master/nginx.tmpl) into the docker-gen container. You can get the latest official [nginx.tmpl](https://github.com/jwilder/nginx-proxy/blob/master/nginx.tmpl) with a command like:
```bash
curl https://raw.githubusercontent.com/jwilder/nginx-proxy/master/nginx.tmpl > /path/to/nginx.tmpl
```

2) Set the `NGINX_DOCKER_GEN_CONTAINER` environment variable to the name or id of the docker-gen container.

Examples:

* First start nginx (official image) with volumes:
```bash
$ docker run -d -p 80:80 -p 443:443 \
    --name nginx \
    -v /etc/nginx/conf.d  \
    -v /etc/nginx/vhost.d \
    -v /usr/share/nginx/html \
    -v /path/to/certs:/etc/nginx/certs:ro \
    nginx
```

* Second start the docker-gen container with the shared volumes and the template file:
```bash
$ docker run -d \
    --name nginx-gen \
    --volumes-from nginx \
    -v /path/to/nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    jwilder/docker-gen \
    -notify-sighup nginx -watch -only-exposed -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
```

* Then start this container (NGINX_DOCKER_GEN_CONTAINER variable must contain the docker-gen container name or id):
```bash
$ docker run -d \
    --name nginx-letsencrypt \
    -e "NGINX_DOCKER_GEN_CONTAINER=nginx-gen" \
    --volumes-from nginx \
    -v /path/to/certs:/etc/nginx/certs:rw \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    jrcs/letsencrypt-nginx-proxy-companion
```
Then start any containers to be proxied as described previously.

#### Let's Encrypt

To use the Let's Encrypt service to automatically create a valid certificate for virtual host(s).

Set the following environment variables to enable Let's Encrypt support for a container being proxied. This environment variables need to be declared in each to-be-proxied application containers.

- `LETSENCRYPT_HOST`
- `LETSENCRYPT_EMAIL`

The `LETSENCRYPT_HOST` variable most likely needs to be the same as the `VIRTUAL_HOST` variable and must be publicly reachable domains. Specify multiple hosts with a comma delimiter.

##### multi-domain ([SAN](https://www.digicert.com/subject-alternative-name.htm)) certificates
If you want to create multi-domain ([SAN](https://www.digicert.com/subject-alternative-name.htm)) certificates add the base domain as the first domain of the `LETSENCRYPT_HOST` environment variable.

##### test certificates
If you want to create test certificates that don't have the 5 certs/week/domain limits define the `LETSENCRYPT_TEST` environment variable with a value of `true`.

##### Automatic certificate renewal
Every hour (3600 seconds) the certificates are checked and every certificate that will expire in the next [30 days](https://github.com/kuba/simp_le/blob/ecf4290c4f7863bb5427b50cdd78bc3a5df79176/simp_le.py#L72) (90 days / 3) are renewed.

##### Example:
```bash
$ docker run -d \
    --name example-app \
    -e "VIRTUAL_HOST=example.com,www.example.com,mail.example.com" \
    -e "LETSENCRYPT_HOST=example.com,www.example.com,mail.example.com" \
    -e "LETSENCRYPT_EMAIL=foo@bar.com" \
    tutum/apache-php
```

#### Optional container environment variables

Optional letsencrypt-nginx-proxy-companion container environment variables for custom configuration.

* `ACME_CA_URI` - Directory URI for the CA ACME API endpoint (default: ``https://acme-v01.api.letsencrypt.org/directory``). If you set it's value to `https://acme-staging.api.letsencrypt.org/directory` letsencrypt will use test servers that don't have the 5 certs/week/domain limits. You can also create test certificates per container (see [let's encrypt test certificates](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/blob/doc/README.md#test-certificates))

For example

```bash
$ docker run -d \
    -e "ACME_CA_URI=https://acme-staging.api.letsencrypt.org/directory" \
    -v /path/to/certs:/etc/nginx/certs:rw \
    --volumes-from nginx-proxy \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    jrcs/letsencrypt-nginx-proxy-companion
```

* `DEBUG` - Set it to `true` to enable debugging of the entrypoint script and generation of LetsEncrypt certificates, which could help you pin point any configuration issues.

* `NGINX_PROXY_CONTAINER`- If for some reason you can't use the docker --volumes-from option, you can specify the name or id of the nginx-proxy container with this variable.

#### Examples:
If you want other examples how to use this container, look at [docker-letsencrypt-nginx-proxy-companion-examples] (https://github.com/fatk/docker-letsencrypt-nginx-proxy-companion-examples).
