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
