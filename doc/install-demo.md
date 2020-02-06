# Installation of Demonstration Environment

This memo documents the installation of the Zone Manager for the LIGHTest
demonstration environment.


## Resources

The environment uses three Digital Ocean droplets (other cloud providers
are available) named as follows:

* lightest-primary
* lightest-secondary-sfo
* lightest-secondary-sgp

The lightest-primary droplet will be running the Zone Manager itself
providing the HTTPS API. It also runs NSD as the primary name server for
the zones served by the environment. The other two droplets will run NSD
as the secondary name servers for those zones.

As of writing this document, the droplets had the following addresses:

Droplet                 | IPv4 address    | IPv6 address
------------------------|-----------------|---------------
lightest-primary        | 167.71.42.146   | 2a03:b0c0:3:e0::bb:5001
lightest-secondary-sfo  | 167.71.127.223  | 2604:a880:2:d1::24:f001
lightest-secondary-sgp  | 206.189.82.24   | 2400:6180:0:d1::8e1:3001

We’ll use these addresses in the code snippets below.

All droplets were created using Debian 10.0 (buster) images.


## All Droplets

```
apt-get update
apt-get dist-upgrade
apt-get install nsd
```

## lightest-primary


### Install and Configure Zone Manager

(Because you need to `git clone` from the IAIK Gitlab instance, you need
to SSH into the machine with -A to pass on your credentials.)

We are installing the zone manager itself into `/usr/lib/zonemgr`. It’s
database and NSD configuration will be in `/var/lib/zonemgr/`.

```
apt-get install git gunicorn python-sqlalchemy python-falcon python-ldns \
  sqlite3
git clone git@github.com:H2020LIGHTest/ZoneManager.git /usr/lib/zonemgr
adduser --system --home /var/lib/zonemgr zonemgr
cp /usr/lib/zonemgr/etc/zonemgr.service /etc/systemd/system
```

Create the database and an environment called `lightest`:

```
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  init
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-environment \
    --environment lightest \
    --nsd-name ns0.lightest.nlnetlabs.nl \
    --nsd-conf /var/lib/zonemgr/nsd.zones.conf \
    --nsd-reload /usr/lib/zonemgr/reload-nsd.sh \
    --key-file /var/lib/zonemgr/private_key.tmp
```

Add the zone we are going to use, `lightest.nlnetlabs.nl`:

```
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-zone \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    --pattern lightest
```

The command will print the DS record for the zone. You will need that
record to configure delegation in the parent zone, in this case
`nlnetlabs.nl`.

Make everything owned by the `zonemgr` user it can access things later.

```
chown -R zonemgr: /var/lib/zonemgr
```

Add some necessary records to the zone:

```
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    lightest.nlnetlabs.nl NS \
      ns0.lightest.nlnetlabs.nl \
      ns1.lightest.nlnetlabs.nl \
      ns2.lightest.nlnetlabs.nl
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    lightest.nlnetlabs.nl A 167.71.42.146
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    lightest.nlnetlabs.nl AAAA 2a03:b0c0:3:e0::bb:5001
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    ns0.lightest.nlnetlabs.nl A 167.71.42.146
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    ns0.lightest.nlnetlabs.nl AAAA 2a03:b0c0:3:e0::bb:5001
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    ns1.lightest.nlnetlabs.nl A 167.71.127.223
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    ns1.lightest.nlnetlabs.nl AAAA 2604:a880:2:d1::24:f001
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    ns2.lightest.nlnetlabs.nl A 206.189.82.24
/usr/lib/zonemgr/zonemanager.py \
  --database sqlite:////var/lib/zonemgr/zones.db \
  add-record \
    --environment lightest \
    --apex lightest.nlnetlabs.nl \
    ns2.lightest.nlnetlabs.nl AAAA 2400:6180:0:d1::8e1:3001
```

Install the DNSSEC resigning cronjob:

```
crontab -u zonemgr /usr/lib/zonemgr/etc/crontab
```

Enable and start the service:

```
systemctl enable zonemgr
systemctl start zonemgr
```


### Install and Configure NSD

Configure NSD to pick up our zones:

```
cat > /etc/nsd/nsd.conf.d/server.conf <<EOF
server:
    ip-address: 0.0.0.0
    ip-address: ::0
    verbosity: 3
    pidfile: "/var/run/nsd/nsd.pid"
EOF
cat > /etc/nsd/nsd.conf.d/zonemgr.conf <<EOF
pattern:
    name: lightest
    notify: 167.71.127.223 NOKEY
    notify: 206.189.82.24 NOKEY
    provide-xfr: 167.71.127.223 NOKEY
    provide-xfr: 206.189.82.24 NOKEY

include: /var/lib/zonemgr/nsd.zones.conf
EOF
cat > /etc/nsd/nsd.conf.d/remote.conf << EOF
remote-control:
    control-enable: yes
    control-interface: 127.0.0.1
    control-port: 8952
    server-key-file: "/etc/nsd/nsd_server.key"
    server-cert-file: "/etc/nsd/nsd_server.pem"
    control-key-file: "/etc/nsd/nsd_control.key"
    control-cert-file: "/etc/nsd/nsd_control.pem"
EOF
```

Enable all the services:

```
systemctl enable nsd
systemctl start nsd
```


### Install Nginx and Acmetool

Get the packages:

```
apt-get install nginx acmetool
```

Start nginx.

```
systemctl enable nginx
systemctl start nginx
```

Bootstrap acmetool. Run

```
acmetool quickstart
```

and then select ’Let’s Encrypt (live)’ and ‘WEBROOT’. Enter
`/var/www/html/.well-known/acme-challenge` is the webroot path (NB: This
is _not_ the default presented). Accept the terms, insert an email address
if you want to and agree to the cronjob to be installed.

Request a certificate:

```
acmetool want lightest.nlnetlabs.nl
```

Now finalize Nginx configuration:

```
cat > /etc/nginx/sites-available/lightest.nlnetlabs.nl <<EOF
server {
    listen 80;
    listen [::]:80;
    server_name lightest.nlnetlabs.nl;
    root /var/name/html;
    location / {
        return 301 https://$host$request_uri;
    }
}
server {
    listen 443 ssl;
	  listen [::]:443 ssl;
	  server_name lightest.nlnetlabs.nl;
	  ssl_certificate /var/lib/acme/live/lightest.nlnetlabs.nl/fullchain;
	  ssl_certificate_key /var/lib/acme/live/lightest.nlnetlabs.nl/privkey;
	  root /var/www/html;
	  location / {
		    proxy_pass http://127.0.0.1:8008$request_uri;
	  }
}
EOF
ln -s ../sites-available/lightest.nlnetlabs.nl /etc/nginx/sites-enabled/lightest.nlnetlabs.nl 
```

Reload nginx:

```
systemctl reload nginx
```



## lightest-secondary-sfo and lightest-secondary-sgp

The two secondary servers are identical. They only need NSD to be
configured to serve as a secondary name server for our zones.

```
mkdir -p /var/lib/nsd/secondary
chown nsd: /var/lib/nsd/secondary
cat > /etc/nsd/nsd.conf.d/server.conf <<EOF
server:
    ip-address: 0.0.0.0
    ip-address: ::0
    verbosity: 3
    pidfile: "/var/run/nsd/nsd.pid"
EOF
cat > /etc/nsd/nsd.conf.d/secondary.conf <<EOF
pattern:
    name: lightest
    allow-notify: 167.71.42.146 NOKEY
    request-xfr: 167.71.42.146 NOKEY

zone:
    name: "lightest.nlnetlabs.nl"
    zonefile: "/var/lib/nsd/secondary/lightest.nlnetlabs.nl"
    include-pattern: lightest
EOF
cat > /etc/nsd/nsd.conf.d/remote.conf << EOF
remote-control:
    control-enable: yes
    control-interface: 127.0.0.1
    control-port: 8952
    server-key-file: "/etc/nsd/nsd_server.key"
    server-cert-file: "/etc/nsd/nsd_server.pem"
    control-key-file: "/etc/nsd/nsd_control.key"
    control-cert-file: "/etc/nsd/nsd_control.pem"
EOF
systemctl enable nsd
systemctl start nsd
```

