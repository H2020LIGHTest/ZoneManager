# LIGHTest DNS ZoneManager

![LIGHTest](https://www.lightest.eu/static/LIGHTestLogo.png)

### Disclaimer

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND. This
software is the output of a research project. It was developed as
proof-of-concept to explore, test & verify various components created in
the LIGHTest project. It can thus be used to show the concepts of
LIGHTest, and as a reference implementation.

# LIGHTest

Lightweight Infrastructure for Global Heterogeneous Trust management in
support of an open Ecosystem of Stakeholders and Trust schemes. For more
information please visit the LIGHTest website: https://www.lightest.eu/

Detailed documentation of LIGHTest and its software components can be
found in the [LIGHTest Deliverables](https://www.lightest.eu/downloads/pub_deliverables/index.html).

# ZoneManager

This is the LIGHTest Zone Manager, a simple REST server that maintains and
signs the DNS zones used by the LIGHTest framework for publishing trust
data.

It is written in Python 2 using Falcon, gunicorn, LDNS, and SQL Alchemy.
See `requirements.txt` for the exact version we’ve used.

Zone Manager consists of a Python 2 module, `zonedb`, and a executable
script, `zonemanager.py` that uses the module to provide the server.


## Installation

ZoneManager can almost be installed from the repository using pip.
Unfortunately, LDNS’ Python bindings are not available via pypi.org, so
you will have to install them separately.

If you are on a Debian-ish system:

```
apt-get install apt-get install python-falcon python-gunicorn python-sqlalchemy
```

Now you can install Zone Manager:

```
pip2 install git+http://github.com/H2020LIGHTest/ZoneManager
```

On non-Debian systems, you might need to install dependencies separately.


## Usage

While primarily a web service, Zone Manager comes with a number of
maintenance commands. For each command, it requires the URI to a database
given via the `--database` (or `-d`, for short) option. This needs to be
given before the actual command. The URI is in SQL Alchemy format which
isn’t entirely intuitive. For SQLite, if you provide an absolute file
name, you need to start with four (!) slashes, e.g., 
`sqlite:////var/lib/zonemgr/zones.db`.

Before we can start, we need to initialize the database:

```
zonemanager.py -d sqlite:////var/lib/zonemgr/zones.db init
```

Next we need to add an environment. This tells Zone Manager where it finds
the instance of NSD to talk to and where to store the configuration for
it:

```
zonemanager.py -d sqlite:////var/lib/zonemgr/zones.db add-environment \
    --environment default \
    --nsd-name ns1.lightest.example.com \
    --nsd-reload /var/lib/zonemgr/reload-nsd.sh \
    --nsd-conf /var/lib/zonemgr/nsd.zones.conf \
    --key-file /var/lib/zonemgr/private_key.tmp
```

There’s five arguments to this. First, we need to give the new environment
so we can refer to it later. This happens via the `--environment` option.
Next, we need to provide the host name of the name server that is run by
our NSD instance via the `--nsd-name` option. This name will appear in the
SOA record of the zones we created.

Whenever Zone Manager updated its zones, it needs to inform NSD to reload
them. This happens via the script provided via the `--nsd-reload` option.
A simple version of that script that assumes that you have installed and
correctly setup the `nsd-control` program is included in the repository.

Zone Manager will create one NSD config file that needs to be included
in your ‘real’ NSD config. The location of this file is provided via the
`--nsd-conf` option.

Finally, we need to have the path to a file where we temporarily store the
private key for signing the zones. This is because of a peculiarity of the
LDNS Python bindings.

Next, we need to add a zone to be controlled by Zone Manager:

```
zonemanager.py -d sqlite:////var/lib/zonemgr/zones.db add-zone \
    --environment default \
    --apex lightest.example.com \
    --pattern lightest
```

We need to provide the name of the argument we created earlier, the apex
of the zone, i.e., the domain name of the top of the domain, and a
pattern. NSD uses patterns to provide configuration for multiple zones.
This needs to be configured in your ‘real’ NSD config. Here, we assume
that you created a pattern named `lightest`.

A correctly configured zone needs a bunch of records. Zone Manager only
creates the SOA record by itself. You will have to manually add at least
the NS records:

```
zonemanager.py -d sqlite:////var/lib/zonemgr/zones.db add-record \
    --environment default \
    --apex lightest.example.com \
    --ttl 86400 \
    lightest.example.com NS \
    ns1.lightest.example.com \
    ns2.lightest.example.com
```

Here we add two NS records. The last two arguments are the record data for
two records with the same name and type. These record data arguments are
in master file format. If that requires spaces, you need to quote them.
This way, you can add multiple records at once.

We also need at least A records for our name servers:

```
zonemanager.py -d sqlite:////var/lib/zonemgr/zones.db add-record \
    --environment default --apex lightest.example.com --ttl 86400 \
    ns1.lightest.example.com A 192.0.2.1
zonemanager.py -d sqlite:////var/lib/zonemgr/zones.db add-record \
    --environment default --apex lightest.example.com --ttl 86400 \
    ns2.lightest.example.com A 192.0.2.2
```

Finally we need to set up authentication for our clients. This happens via
simple bearer tokens that are exchanged out of band. Currently, you can
only create new tokens like so:

```
zonemanager.py -d sqlite:////var/lib/zonemgr/zones.db create-token \
    --environment default \
    user-alpha \
    lightest.example.com
```

This needs the name of the environment, a name of the token – this is only
for your own bookkeeping – and the (apex of the) zone this token is
allowing access to. The token will be printed on the terminal so you can
copy it and give it to your user.

Now you can start the server:

```
zonemanager.py -d sqlite:////var/lib/zonemgr/zones.db server 127.0.0.1:8080
```

This will run the server on localhost’s port 8080. Since Zone Manager
doesn’t do any TLS, you will need to run it behind nginx or Apache.


# Licence

* Apache License 2.0 (see [LICENSE](./LICENSE))
* © LIGHTest Consortium
