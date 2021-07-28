# Let's Encrypt SSL wildcard certificates with acme.sh auto renewal

## Author

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Installation

- Git clone and install

```bash
apt install git socat
git clone https://github.com/acmesh-official/acme.sh.git
cd acme.sh/
./acme.sh --install \
	--home /opt/acme.sh \
	--config-home /opt/acme.sh/data \
	--cert-home /opt/acme.sh/certs \
	--accountemail "john.doe@example.tld" \
	--accountkey /opt/acme.sh/example.tld.key \
	--accountconf /opt/acme.sh/example.tld.conf
```

> **NOTE**: The installation process takes place on a Bind9 DNS server running GNU/Linux Debian 10 Buster.

## Issuing/renewing certificates automatically with nsupdate

- Generate a key for updating the zone

```bash
b=$(dnssec-keygen -a hmac-sha512 -b 512 -n USER -K /tmp acme)
cat > /etc/bind/nsupdate.key <<EOF
key "letsencrypt" {
    algorithm hmac-sha512;
    secret "$(awk '/^Key/{print $2}' /tmp/$b.private)";
};
EOF
rm -f /tmp/$b.{private,key}
```

- Secure the key

```bash
chmod 640 /etc/bind/nsupdate.key
chown bind.bind /etc/bind/nsupdate.key
```

- Include the key in bind9 main configuration file

```bash
nano /etc/bind/named.conf

...
include "/etc/bind/nsupdate.key";
...
```

- Configure the zone to allow dynamic updates

```bash
zone "example.tld" {
    type master;
    update-policy {
        grant "letsencrypt" name _acme-challenge.example.tld. TXT;
    };
}
```

- Make the DNS server and update key available to acme.sh

```bash
export NSUPDATE_SERVER="127.0.0.1"
export NSUPDATE_KEY="/etc/bind/nsupdate.key"
export NSUPDATE_ZONE="example.tld"
```

- Issue a wildcard certificate

```bash
acme.sh --issue -d example.tld -d *.example.tld --days 90 --dns dns_nsupdate --dnssleep 60
```

If everything succeeded, it should get two `TXT` records temporarily added to zone `example.tld` similarily to:

```bash
_acme-challenge.example.tld. 60 IN TXT "W_-Qk9a2e5xlMWEJHfbl5Sp_vw8T1oLsIaIthzDgcDs"
_acme-challenge.example.tld. 60 IN TXT "NQ9KX3PSo0T_qhIKyAYQoBq7XRng3WwfnV58YyeI9k0"
```

> **NOTE**: To allow Let’s Encrypt certificate authority the issuance of SSL certificates for `example.tld`, add the following `CAA` record:
>
> ```bash
> example.tld. 60 IN CAA 0 issuewild "letsencrypt.org"
>                    CAA 0 iodef "mailto:john.doe@example.tld"
> ```

## Automate certificate renewal

```bash
crontab -e

0 0 1 */2 * "/opt/acme.sh"/acme.sh --renew -d example.tld -d *.example.tld --days 90 --dns dns_nsupdate --dnssleep 60 > /dev/null
```

## Stop certificate renewal

```bash
acme.sh --remove -d example.tld -d *.example.tld
```

## Deploy the Let's Encrypt SSL Certificate on services

### Let's Encrypt SSL Certificate on Zimbra

```bash
mkdir /opt/zimbra/ssl/letsencrypt
mv example.tld.key fullchain.cer /opt/zimbra/ssl/letsencrypt/
cd /opt/zimbra/ssl/letsencrypt/
wget https://letsencrypt.org/certs/trustid-x3-root.pem.txt
cat fullchain.cer trustid-x3-root.pem.txt > chain.pem
chown zimbra.zimbra *
```
```bash
su - zimbra
$ cd /opt/zimbra/ssl/letsencrypt/
$ /opt/zimbra/bin/zmcertmgr verifycrt comm example.tld.key fullchain.cer chain.pem
cp -a /opt/zimbra/ssl/zimbra /opt/zimbra/ssl/zimbra.$(date "+%Y%m%d")
cp example.tld.key /opt/zimbra/ssl/zimbra/commercial/commercial.key
```
```bash
su - zimbra
$ cd /opt/zimbra/ssl/letsencrypt/
$ /opt/zimbra/bin/zmcertmgr deploycrt comm fullchain.cer chain.pem
$ zmcontrol restart
```

Test the certificate

```bash
su - zimbra
$ echo QUIT | openssl s_client -connect mail.example.tld:443 | openssl x509 -noout -text | less
```

### Let's Encrypt SSL Certificate on iRedMail Server

```bash
mkdir /opt/letsencrypt
mv example.tld.{cer,key} /opt/letsencrypt
chmod 0444 /opt/letsencrypt/example.tld.cer
chmod 0400 /opt/letsencrypt/example.tld.key
```
```bash
mv /etc/ssl/certs/iRedMail.crt{,.bak}
mv /etc/ssl/private/iRedMail.key{,.bak}
ln -s /opt/letsencrypt/example.tld.cer /etc/ssl/certs/iRedMail.crt
ln -s /opt/letsencrypt/example.tld.key /etc/ssl/private/iRedMail.key
```

Restart related services

```bash
systemctl restart postfix.service dovecot.service nginx.service
```

### Let's Encrypt SSL Certificate on Proxmox Mail Gateway

#### Mail certificate

```bash
mv /etc/pmg/pmg-tls.pem{,.org}
cat example.tld.key fullchain.cer > /etc/pmg/pmg-tls.pem
chmod 0600 /etc/pmg/pmg-tls.pem
chown root.root /etc/pmg/pmg-tls.pem
```

#### HTTPs certificate

```bash
mv /etc/pmg/pmg-api.pem{,.org}
cat example.tld.key fullchain.cer > /etc/pmg/pmg-api.pem
chmod 0640 /etc/pmg/pmg-api.pem
chown root.www-data /etc/pmg/pmg-api.pem
```

Restart related service

```bash
systemctl restart pmgproxy.service
```

### Let's Encrypt SSL Certificate on Proxmox Vitual Environment

Use the Web GUI to deploy the files `fullchain.cer` and `example.tld.key`.

```
(Datacenter/"Proxmox Node"/System/Certificates/Upload Custom Certificate)
```

Tough the recommended method is by using the Web GUI, the command line could be used as well:

```bash
cp fullchain.cer /etc/pve/local/pveproxy-ssl.pem
cp example.tld.key /etc/pve/local/pveproxy-ssl.key
chmod 0640 /etc/pve/local/pveproxy-ssl.*
chown root.www-data /etc/pve/local/pveproxy-ssl.*
```

Restart related service

```bash
systemctl restart pveproxy.service
```

### Let's Encrypt SSL Certificate on pfSense Firewall

Use the Web GUI to deploy the files `fullchain.cer` and `example.tld.key`.

```
(System/Certificate Manager/Certificates/"Add/Sign Button"/Method "Import an existing Certificate")

(System/Advanced/Admin Access/SSL/TLS Certificate)
```

### Let's Encrypt SSL Certificate on Web Server

```bash
mv fullchain.cer /etc/ssl/certs/
mv example.tld.cer /etc/ssl/certs/
mv example.tld.key /etc/ssl/private/
chmod 0444 /etc/ssl/certs/{example.tld,fullchain}.cer
chmod 0400 /etc/ssl/private/example.tld.key
```

#### Apache

```bash
nano /etc/apache2/sites-available/exampleTLD.conf

...
SSLEngine on
SSLCertificateFile /etc/ssl/certs/example.tld.cer
SSLCertificateKeyFile /etc/ssl/private/example.tld.key
SSLCertificateChainFile "/etc/ssl/certs/fullchain.cer"
SSLProtocol -all +TLSv1.3 +TLSv1.2
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM
SSLHonorCipherOrder on
SSLOpenSSLConfCmd Curves X25519:secp521r1:secp384r1:prime256v1
SSLOpenSSLConfCmd DHParameters "/etc/ssl/dh4096.pem"
Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Frame-Options SAMEORIGIN
Header always set X-Content-Type-Options nosniff
Header always set Content-Security-Policy "default-src 'self';"
Header always set X-XSS-Protection "1; mode=block"
Header always set Set-Cookie "HttpOnly;Secure"
SSLCompression off
SSLSessionTickets off
...
```

Test settings, if syntax returns `OK`, restart the web service:

```bash
apache2ctl -t
systemctl restart apache2.service
```

#### Nginx

```bash
nano /etc/nginx/sites-available/exampleTLD

...
ssl on;
ssl_certificate /etc/ssl/certs/fullchain.cer;
ssl_certificate_key /etc/ssl/private/example.tld.key;
ssl_dhparam /etc/ssl/dh4096.pem;
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
ssl_ecdh_curve secp384r1;
ssl_prefer_server_ciphers on;
ssl_session_tickets off;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block";
add_header X-Content-Type-Options "nosniff";
add_header Content-Security-Policy "default-src 'self';" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Set-Cookie "HttpOnly;Secure" always;
...
```

Test settings, if syntax returns `OK`, restart the web service:

```bash
nginx -t
systemctl restart nginx.service
```

## References

* [An ACME Shell script: acme.sh](https://github.com/acmesh-official/acme.sh)
* [cerbot](https://certbot.eff.org/)
* [Installing a Let's Encrypt SSL Certificate](https://wiki.zimbra.com/wiki/Installing_a_LetsEncrypt_SSL_Certificate)
* [Deploy Commercial SSL Certificate on Proxmox Mail Gateway](https://dhenandi.com/deploy-commercial-ssl-certificate-on-proxmox-mail-gateway/)
* [Certificate Management](https://pve.proxmox.com/wiki/Certificate_Management)
* [How-To -- Lets Encrypt and PMG](https://forum.proxmox.com/threads/how-to-lets-encrypt-and-pmg.41493/)
* [How To Secure Apache with Let's Encrypt on Debian 10](https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-debian-10)
* [Request a free cert from Let's Encrypt](https://docs.iredmail.org/letsencrypt.html)
* [Update: Using Free Let’s Encrypt SSL/TLS Certificates with NGINX](https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/)
* [Installing Let’s Encrypt SSL Certificate with pfSense](https://gainanov.pro/eng-blog/linux/installing-lets-encrypt-pfsense/)
* [Let's Encrypt on pfSense](https://www.netgate.com/resources/videos/lets-encrypt-on-pfsense.html)
* [How to issue Let’s Encrypt wildcard certificate with acme.sh and Cloudflare DNS](https://www.cyberciti.biz/faq/issue-lets-encrypt-wildcard-certificate-with-acme-sh-and-cloudflare-dns/)
* [CAA Records](https://support.dnsimple.com/articles/caa-record/)
* [CAA Record Helper](https://sslmate.com/caa/)
* [SSL/TLS Strong Encryption: How-To](https://httpd.apache.org/docs/2.4/ssl/ssl_howto.html)
* [Apache Module mod_ssl](https://httpd.apache.org/docs/current/mod/mod_ssl.html#sslcacertificatefile)
* [Cipherli.st Strong Ciphers for Apache, nginx and Lighttpd](https://syslink.pl/cipherlist/)
* [SSL Server Test](https://www.ssllabs.com/ssltest)
* [SSL and TLS Deployment Best Practices](https://github.com/ssllabs/research/wiki/SSL-and-TLS-Deployment-Best-Practices)
* [SSL Server Rating Guide](https://github.com/ssllabs/research/wiki/SSL-Server-Rating-Guide)
* [pfSense as Name Server (bind9) with Let’s Encrypt/acme DNS-NSupdate/RFC 2136](https://forum.level1techs.com/t/pfsense-as-name-server-bind9-with-lets-encrypt-acme-dns-nsupdate-rfc-2136/168097)
* [Creating Wildcard Certificates on pfSense with Let’s Encrypt](https://www.danielcolomb.com/2019/08/29/creating-wildcard-certificates-on-pfsense-with-lets-encrypt/)
* [pfSense setup ACME Lets Encrypt](https://www.youtube.com/watch?v=Tc_8PAE8S28)
