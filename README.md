# acme-tiny-dns01

## Quick test
```sh
wget "https://raw.githubusercontent.com/bugness-chl/acme-tiny-dns01/dns/acme_tiny_dns01.py"
sudo apt install python python-dnspython openssl ca-certificates
openssl genrsa 4096 > account.key
openssl genrsa 2048 > domain.key
openssl req -new -sha256 -key domain.key -subj "/CN=yoursite.com" > domain.csr
python acme_tiny_dns01.py --account-key ./account.key --csr ./domain.csr --ca "https://acme-staging.api.letsencrypt.org" --quiet > ./signed.crt
# Copy-paste the DNS record to your DNS management interface and press [enter]
```

You can check the result with :
```
openssl x509 -noout -text -in signed.crt
```


## Intro

This script is an adaptation of [acme-tiny](https://github.com/diafygi/acme-tiny)
for the dns-01 challenge protocol (big thanks to [Nuxi](https://github.com/nuxi/acme-tiny),
most of the adaptation work was done by him). It is useful when your server can not or should not
expose a public HTTP port.

Similarly, this is a tiny, auditable script that you can use to issue
and renew [Let's Encrypt](https://letsencrypt.org/) certificates.
However, at its current state, it needs manual intervention and thus
cannot automate the renewal of a certificate.

Before launching it, you need to have :
* your account.key (see below for help, keep preciously)
* the CSR for the domain(s) (see below for help)
* write access to the DNS zone of the domain(s) (can't help you with that)

The only system prerequisites are python (+dns library) and openssl.
```sh
# On debian/ubuntu (ca-certificates needed to authenticate Let's Encrypt HTTPS servers)
sudo apt install python python-dnspython openssl ca-certificates
```

Cons /acme-tiny-http :
* at the moment, needs a manual intervention to update the DNS zone, which defeats the Let's Encrypt's "fire and forget" style.
* in consequence, you will also need to set a reminder or to monitor the validity of the certificate (see option `--contact-mail`)

Pros /acme-tiny-http :
* doesn't need access to a web server,
* doesn't need to run on a server, just copy-paste the DNS challenge to your DNS administration interface,
* Let's Encrypt doesn't even need to access the server which will receive the certificate. Given the distinction between a domain and a zone in DNS, you only need to be able to add the record `_acme-challenge.private-jabber-service.private-network.example.org. IN TXT 123challengeXYZ` in your public DNS zone file for `example.org`.

**PLEASE READ THE SOURCE CODE! YOU MUST TRUST IT WITH YOUR PRIVATE KEYS!**

## Donate

If this script is useful to you, please donate to the EFF. I don't work there,
but they do fantastic work.

[https://eff.org/donate/](https://eff.org/donate/)

## How to use this script

If you already have a Let's Encrypt issued certificate and just want to renew,
you should only have to do Steps 3 and 4.

### Step 1: Create a Let's Encrypt account private key (if you haven't already)

To reuse a key created by official Let's Encrypt Certbot key, see
[the annex below](#user-content-use-existing-lets-encrypt-key).

You must have an account key that this script will use to register/authenticate
you to Let's Encrypt and to sign your requests. If you don't understand what I
just said, this script likely isn't for you! Please use the official
[Let's Encrypt client](https://github.com/certbot/certbot).

To initially create your account key :
```
openssl genrsa 4096 > account.key
```
Like a password, keep this account key preciously !


### Step 2: Create a certificate signing request (CSR) for your domains.

The ACME protocol (what Let's Encrypt uses) requires a CSR file to be submitted
to it, even for renewals. You can use the same CSR for multiple renewals. NOTE:
you can't use your account private key as your domain private key!

```sh
#generate a domain private key (if you haven't already)
openssl genrsa 4096 > domain.key
# or, for ECDSA : openssl ecparam -name prime256v1 -genkey -out domain.key

#for a single domain
openssl req -new -sha256 -key domain.key -subj "/CN=yoursite.com" > domain.csr

#for multiple domains (use this one if you want both www.yoursite.com and yoursite.com)
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:yoursite.com,DNS:www.yoursite.com")) > domain.csr
```

### Step 3: Launch the script and insert the DNS challenge(s) in your zone

Let's Encrypt mechanism requires you to prove that you own the domain(s) for
which you want a certificate. This script will negotiate a challenge with
Let's Encrypt and will generate the DNS record(s). You will have to
temporarily insert this(those) record(s) yourself in the zone file of
each domain.

Launch the script with the account key and the domain CSR. For example :
```sh
python acme_tiny_dns01.py --account-key ./account.key --csr ./domain.csr --contact-mail webmaster@example.org > ./signed.crt
```
The `--contact-mail` option is optional but will allow Let's Encrypt to
contact you and alert you when the certificate will be about to expire
(caution, there is no guarantee: https://community.letsencrypt.org/t/expiration-emails-too-many-unnecessary-etc/9502).

It should go like that :
```
Parsing account key...
Parsing CSR...
Registering account...
Already registered!
Getting challenge for smtp.example.org...
_acme-challenge.smtp.example.org. 300 IN TXT QX4c_VKaZQnMnMhaxvQ0IH73JmOdFt5lOw8IMmmliyE
Press enter to continue after updating DNS server
# (see below note)

Local checks on smtp.example.org...
Locally checking challenge on 2a01:e35:1234:5678::baba:5353...
Locally checking challenge on 12.34.56.78...
Asking authority to verify challenge smtp.example.org...
smtp.example.org verified!
Signing certificate...
Certificate signed!
You can now remove the _acme-challenge records from your DNS zone.
```

Note on updating the DNS zone file :
This is the time where you copy-paste the challenge(s) to your DNS zone file.
If you used the `--skip-check` option, please manually check the record has
been published before pressing Enter. For example, with dig&nbsp;:
```
dig @ip.of.one.of.the.primary.nameservers.for.the.zone   _acme-challenge.smtp.example.org. TXT
# (must display an answer section with the correct challenge)
```

### Step 4: Clean the zone file

Once the challenge is over, you can remove the TXT record from the DNS zone file.

Install the new certificate to the appropriate location (see your software doc.
Nginx example given below).

You should keep the CSR file for the next renewals and **must** keep the account.key file.

## Annex
### Use existing Let's Encrypt key
If you switched from official Let's Encrypt Certbot to this script, you will have
some conversion to do.

The private account key from the Let's Encrypt client is saved in the
[JWK](https://tools.ietf.org/html/rfc7517) format. `acme-tiny` is using the PEM
key format. To convert the key, you can use the tool
[conversion script](https://gist.github.com/JonLundy/f25c99ee0770e19dc595) by JonLundy:

```sh
# Download the script
wget -O - "https://gist.githubusercontent.com/JonLundy/f25c99ee0770e19dc595/raw/6035c1c8938fae85810de6aad1ecf6e2db663e26/conv.py" > conv.py

# Copy your private key to your working directory
cp /etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/<id>/private_key.json private_key.json

# Create a DER encoded private key
openssl asn1parse -noout -out private_key.der -genconf <(python conv.py private_key.json)

# Convert to PEM
openssl rsa -in private_key.der -inform der > account.key
```

### Install the certificate on nginx

The signed certificate that is output by this script can be used along
with the domain private key to run an https server. You need to include them in the
https settings in your web server's configuration. Here's an example on how to
configure an nginx server:

```
#NOTE: For nginx, you need to append the Let's Encrypt intermediate cert to your cert
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained.pem
```

```nginx
server {
    listen 443;
    server_name yoursite.com, www.yoursite.com;

    ssl on;
    ssl_certificate /path/to/chained.pem;
    ssl_certificate_key /path/to/domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_session_cache shared:SSL:50m;
    ssl_dhparam /path/to/server.dhparam;
    ssl_prefer_server_ciphers on;

    ...the rest of your config
}

server {
    listen 80;
    server_name yoursite.com, www.yoursite.com;

    ...the rest of your config
}
```

## Feedback/Contributing

This project has a very, very limited scope and codebase. I'm happy to receive
bug reports and pull requests, but please don't add any new features. This
script must stay under 200 lines of code to ensure it can be easily audited by
anyone who wants to run it.

If you want to add features for your own setup to make things easier for you,
please do! It's open source, so feel free to fork it and modify as necessary.
