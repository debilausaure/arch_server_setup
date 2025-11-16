This page will guide you through setting up a certificate for the server.
It is generated and renewed automatically by Traefik, which uses Lego under
the hood. The certificate is issued through an ACME DNS (DNS-01) challenge. 

The certificate is a SAN wildcard certificate from Let's Encrypt, valid for
`debilausau.re` and all subdomains matched by `*.debilausau.re`.
The certificate Common Name (CN) will be `*.debilausau.re`, while both
`debilausau.re` and `*.debilausau.re` will figure in the Subject Alternative
Name (SAN).

# DNS zone credentials

The first step of this guide is to set up the proper credentials for lego to
be able to edit the domain's DNS entries needed to solve the challenge.

## Remove pre-existing DNS credentials

Visit [https://eu.api.ovh.com/console/](https://eu.api.ovh.com/console/) and
select the `<GET> /me/api/application` route. Log in to be able to use the API
and make a request, which should return a table of application IDs:

```json
[
  12345678,
  16253427,
  ...
]
```

We will use the API to remove all credentials from this table.

Select the `<DEL> /me/api/application/{applicationId}` route, which takes a
parameter. Make a request to this route for every application ID previously
retrieved, which will revoke all previously setup credentials.

## Setup new DNS credentials

Visit [https://eu.api.ovh.com/createToken/](https://eu.api.ovh.com/createToken/)
and log in. You are greeted with an UI, fill it with the following values.

```
Application Name        : Traefik
Application Description : Let's Encrypt DNS challenge
Validity                : Unlimited
Rights                  : <POST> /domain/zone/*
                          <DEL>  /domain/zone/*
--------------------------------------------------------------------
Rights                  : <POST> /domain/zone/debilausau.re/record
(stricter but untested)   <POST> /domain/zone/debilausau.re/refresh
                          <DEL>  /domain/zone/debilausau.re/record/*
```

Now write the different values returned by the UI into the corresponding files:
```
Application key    -> OVH/dns_challenge_creds/application_key
Application secret -> OVH/dns_challenge_creds/application_secret
Consumer Key       -> OVH/dns_challenge_creds/consumer_key
```

The endpoint is `ovh-eu` by default, you can change that in
`OVH/dns_challenge_creds/endpoint` if needed.

# Traefik configuration

First tell Traefik to use Let's Encrypt staging certificate servers.
Uncomment the following lines in the `compose.yml` file:
```
    ### Debug servers
    # - "--certificatesresolvers.le-cert-resolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
```

Then, uncomment the following `whoami` service, which as a special route to generate the certificate.
```
## To enable when setting up Let's Encrypt certificates
#  whoami:
#    image: "traefik/whoami"
#    container_name: "whoami"
#    networks:
#      - traefik
#    labels:
#      - "traefik.enable=true"
#      - "traefik.http.routers.whoami.entrypoints=websecure"
#      - "traefik.http.routers.whoami.tls.certresolver=le-cert-resolver"
#      - "traefik.http.routers.whoami.tls.domains[0].main=*.debilausau.re"
#      - "traefik.http.routers.whoami.tls.domains[0].sans=debilausau.re"
#      - "traefik.http.routers.whoami.rule=Host(`whoami.debilausau.re`)"
```

Start Traefik by running `sudo docker compose up -d`, and check the generated certificate.
If everything is fine, you can stop Traefik with `sudo docker compose down` and switch 
to the production servers by commenting the staging server configuration and removing the 
`letsencrypt/wildcardv2.json` file:
```
    ### Debug servers
    # - "--certificatesresolvers.le-cert-resolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
```

Once the certificate is generated, you can comment back the `whoami` service.

# Share the certificates with a KVM

If, like me, you've got a KVM hooked-up to your server, you might want to make it available through the same domain name.
Traefik handles the certificate generation, and stores the relevant informations inside a JSON file.
We'll extract the key and the certificate from Traefik's JSON, copy them through `ssh` to the KVM, and restart it.
This process will be set into a script, which will be called from a systemd service, automatically started everytime the file gets updated. 

## Preliminary set-up
We'll use `jq` to extract the informations from the JSON. Hence, we need `jq`:
```sh
pacman -S jq
```

Then, we'll need a key-pair to be able to connect without passwords to the KVM.
Generate a key-pair on the server:
```sh
ssh-keygen -f ~/.ssh/kvm_id_ed25519 -t ed25519 -C "kvm" -N ''
```

Copy the key pair to the KVM:
```sh
ssh-copy-id -i ~/.ssh/kvm_id_25519.pub root@kvm.local
```

Add an entry in your ssh config file to specify to use the appropriate key when connecting to the kvm:
```
# Use an absolute path for the identity file
Host kvm
    HostName 192.168.1.27
    User root
    IdentityFile ~/.ssh/kvm_id_ed25519
    IdentitiesOnly yes
```
## Automatic update setup

### Create a script

Create a script file under a folder of your liking (e.g. /usr/local/bin/copy_certs_to_kvm.sh):
```sh
#!/bin/sh

CRT_FILE_PATH=/path/to/letsencrypt/folder/server.crt
KEY_FILE_PATH=/path/to/letsencrypt/folder/server.key

if [ -e $CRT_FILE_PATH ]; then
  :> $CRT_FILE_PATH
else
  install -m 300 /dev/null $CRT_FILE_PATH
fi
if [ -e $KEY_FILE_PATH ]; then
  :> $KEY_FILE_PATH
else
  install -m 300 /dev/null $KEY_FILE_PATH
fi
jq -r '."le-cert-resolver"."Certificates"[] | select(.domain.sans[0] == "debilausau.re")."certificate"' /path/to/letsencrypt/folder/wildcard_acmev2.json | base64 -d >> $CRT_FILE_PATH
jq -r '."le-cert-resolver"."Certificates"[] | select(.domain.sans[0] == "debilausau.re")."key"' /path/to/letsencrypt/folder/wildcard_acmev2.json | base64 -d >> $KEY_FILE_PATH
scp -F /path/to/your/user/.ssh/config $CRT_FILE_PATH $KEY_FILE_PATH kvm:/etc/kvm/
timeout 30 ssh -F /path/to/your/user/.ssh/config kvm '/etc/init.d/S95nanokvm restart'
rm $CRT_FILE_PATH
rm $KEY_FILE_PATH
```
### Update service

Create a service file under the system systemd folder (e.g. /etc/systemd/system/copy_certs_to_kvm.service):
```
[Unit]
Description="Extract the certificate and key from the Let's Encrypt ACME json and copies with through scp to the KVM"

[Service]
ExecStart=/usr/local/bin/copy_certs_to_kvm.sh
```

### Path watching service

Create a new path file `/etc/systemd/system/acme_json_monitor.path`.
```
[Unit]
Description="Monitor the Let's Encrypt ACME json file for changes"

[Path]
PathModified=/path/to/letsencrypt/folder/wildcard_acmev2.json
Unit=copy_certs_to_kvm.service

[Install]
WantedBy=multi-user.target
```
