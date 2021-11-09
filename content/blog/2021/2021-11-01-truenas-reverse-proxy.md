+++
title = "Reverse proxy on TrueNAS"
+++
# Reverse Proxy on TrueNAS

On my [TrueNAS](https://www.truenas.com/) server I run two application servers in jails. In order to access them from outside my local network I could just forward specific ports from the WAN side of my router to the IP addresses of the servers, but this is an inelegant solution. It would require that I specify the port in the address and I would have to manage certificates for each application server. What I want is to use a subdomain as address like `app.example.com`. I also want TLS encryption without having to deal with certificates for each subdomain and server.

Both of these requirements can be achieved by using a reverse proxy, so let's get started.

This guide is heavily based on [Samuel Dowling's guide "How to set up an nginx reverse proxy with SSL termination in FreeNAS"](https://www.samueldowling.com/2020/01/18/nginx-reverse-proxy-freenas-ssl-tls/), so go read that first. Instead of [`certbot`](https://certbot.eff.org/) I use [`acme.sh`](https://github.com/acmesh-official/acme.sh) for certificate management, and I set the SSL config to be used globally instead of per domain since this reverse proxy only manages a single domain.


## Jail for nginx

There is no built-in solution or plugin for reverse proxy functionality on TrueNAS, so we roll our own using [nginx](https://nginx.org/en/). First go to the "Jails" section in TrueNAS and create a new jail. I recommend enabling `allow_raw_sockets` under "Jail Properties" so it's possible to ping and traceroute the server. You can also create the jail using the `iocage` command in a terminal.

The new jail has no services running so the next step is to make administration and configuration a bit easier by adding a user and enabling the sshd service. An alternative is to log in to the TrueNAS server via a terminal and use the `jexec` command to enter the jail, but I like having the jail servers self-contained.

Go to the Jail section in TrueNAS, expand your newly created jail and press the Shell button (or use `jexec`). You now have a shell with root access.

Type `adduser` and follow the instructions. Name the user whatever is suitable. When asked about additional groups, add the user to the `wheel` group. This enables use of the `su` and `sudo` commands.

Now enable the `sshd` service and start it:

```sh
$ sysrc sshd_enable=YES
$ service sshd start

# Verify sshd is running
$ service sshd status
```

You can now log in to the server with the user created, and run commands as root with `sudo` and switch to the root user using `su -`.


## nginx and SSL certificate

Next install nginx. For SSL certificate management install `acme.sh`. All commands are assumed to be run as root.

```sh
$ pkg update
$ pkg install nginx
$ pkg install acme.sh
```

Since we want a single certificate for all subdomains we'll need to issue a wildcard certificate:

```sh
$ acme.sh --register-account -m name@example.com
$ acme.sh --issue --dns dns_yourprovider -d '*.example.com'
```

Replace `dns_yourprovider` with the plugin name of your DNS provider. Follow the instructions from your provider on how to enable API access. You will at least have to create some sort of API key. If you can't find your provider plugin [in the list](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) then you can [issue a certificate manually](https://github.com/acmesh-official/acme.sh#9-use-dns-manual-mode) but you will then lose the ability to do automatic updates of the certificate using `acme.sh`.

Note that the files downloaded using `acme.sh` should _not_ be used directly. Install them somewhere the nginx config can find them:

```sh
$ mkdir /usr/local/etc/nginx/domains
$ acme.sh --install-cert -d "*.example.com" \
--cert-file /usr/local/etc/nginx/domains/example.com/cert.pem \
--key-file /usr/local/etc/nginx/domains/example.com/key.pem \
--fullchain-file /usr/local/etc/nginx/domains/example.com/fullchain.pem \
--reloadcmd "service nginx reload"
```


## Domain configuration

Mozilla has a nice [web page for generating the SSL configuration](https://ssl-config.mozilla.org/). A modern configuration will look something like this:

```
# generated 2021-10-31, Mozilla Guideline v5.6, nginx 1.20.1, OpenSSL 1.1.1l, modern configuration
# https://ssl-config.mozilla.org/#server=nginx&version=1.20.1&config=modern&openssl=1.1.1l&guideline=5.6
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    ssl_certificate /path/to/signed_cert_plus_intermediates;
    ssl_certificate_key /path/to/private_key;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
    ssl_session_tickets off;

    # modern configuration
    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;

    # HSTS (ngx_http_headers_module is required) (63072000 seconds)
    add_header Strict-Transport-Security "max-age=63072000" always;

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;

    # verify chain of trust of OCSP response using Root CA and Intermediate certs
    ssl_trusted_certificate /path/to/root_CA_cert_plus_intermediates;

    # replace with the IP address of your resolver
    resolver 127.0.0.1;
}
```

Copy and paste the configuration to `/usr/local/etc/nginx/snippets/ssl-params.conf`. We'll also need to configure proxy behaviour. Copy the following to `/usr/local/etc/nginx/snippets/proxy-params.conf`.

```
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $server_name;
proxy_set_header X-Forwarded-Ssl on;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_http_version 1.1;
```

Now we need a configuration for the subdomain. Put the following in `/usr/local/etc/nginx/domains/app.example.com.conf`:

```
server {
    listen 443 ssl http2;

    server_name app.example.com;
    access_log /var/log/nginx/app.example.com.access.log;
    error_log /var/log/nginx/app.example.com.error.log;

    location / {
        include snippets/proxy-params.conf;
        proxy_pass https://name-or-ip-address;
    }
}
```

The `proxy_pass` setting is the forwarding target on the internal network. It can be an address or IP with optional port. If you need additional subdomains just add similar files for each one.

The last piece of the puzzle is the nginx configuration itself. Make a backup of the existing `/usr/local/etc/nginx/nginx.conf` and replace or modify it with the following:

```
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;
    server_tokens off;

    include "snippets/ssl-params.conf";
    include "domains/*.conf";
}
```

Reload nginx to enable the configuration.

```sh
$ service nginx reload
```

If the reverse proxy is behind NAT you will have to forward port 443 (HTTPS) from the WAN side to the proxy server. You can forward port 80 (HTTP) too for convenience if you want, but with the configuration we just wrote it will just get redirected to port 443.
