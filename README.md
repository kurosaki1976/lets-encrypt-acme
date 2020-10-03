# Let's Encrypt Certificates with acme.sh

## Author

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Installation

```bash
apt install git socat
git clone https://github.com/acmesh-official/acme.sh.git
cd ./acme.sh
./acme.sh --install \
	--home /opt/acme.sh \
	--config-home /opt/acme.sh/data \
	--cert-home /opt/acme.sh/certs \
	--accountemail "john.doe@example.tld" \
	--accountkey /opt/acme.sh/example.tld.key \
	--accountconf /opt/acme.sh/example.tld.conf
```

> **NOTE**: The installation process takes place on a Bind9 DNS server.

## Issue a certificate

```bash
acme.sh --issue -d example.tld -d *.example.tld --days 90 --yes-I-know-dns-manual-mode-enough-go-ahead-please --dns
```

You should get an output like below:

```bash
Add the following txt record:
Domain:_acme-challenge.example.tld
Txt value:W_-Qk9a2e5xlMWEJHfbl5Sp_vw8T1oLsIaIthzDgcDs

Add the following txt record:
Domain:_acme-challenge.example.tld
Txt value:NQ9KX3PSo0T_qhIKyAYQoBq7XRng3WwfnV58YyeI9k0
```

Next, you should add these two `TXT` records to your `example.tld` forward zone:

```bash
_acme-challenge.example.tld. 60 IN TXT "W_-Qk9a2e5xlMWEJHfbl5Sp_vw8T1oLsIaIthzDgcDs"
                                   TXT "NQ9KX3PSo0T_qhIKyAYQoBq7XRng3WwfnV58YyeI9k0"
```

To allow Let’s Encrypt certificate authority the issuance of SSL certificates for `example.tld`, add the following CAA record:

```bash
example.tld. 60 IN CAA 0 issue "letsencrypt.org"
                   CAA 0 issuewild "letsencrypt.org"
                   CAA 0 iodef "mailto:postmaster@example.tld"
```

## Renew the certificate

```bash
acme.sh --renew -d example.tld -d *.example.tld --days 90 --yes-I-know-dns-manual-mode-enough-go-ahead-please --dns

Your cert is in  /opt/.acme.sh/example.tld/example.tld.cer
Your cert key is in  /opt/.acme.sh/example.tld/example.tld.key
The intermediate CA cert is in  /opt/.acme.sh/example.tld/ca.cer
And the full chain certs is there:  /opt/.acme.sh/example.tld/fullchain.cer
```

## Automate certificate renewal

```bash
crontab -e

0 0 1 */3 * "/opt/acme.sh"/acme.sh --renew -d example.tld -d *.example.tld --days 90 --yes-I-know-dns-manual-mode-enough-go-ahead-please > /dev/null
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

```bash
mv /etc/pmg/pmg-api.pem{,.org}
cat example.tld.key example.tld.cer > /etc/pmg/pmg-api.pem
chmod 0640 /etc/pmg/pmg-api.pem
chown root.www-data /etc/pmg/pmg-api.pem
```

Restart related service

```bash
systemctl restart pmgproxy.service
```

### Let's Encrypt SSL Certificate on Proxmox Vitual Environment

Use the Web GUI to deploy the files `example.tld.cer` and `example.tld.key`.

```
(Datacenter/"Proxmox Node"/System/Certificates/Upload Custom Certificate)
```

### Let's Encrypt SSL Certificate on pfSense Firewall

Use the Web GUI to deploy the files `example.tld.cer` and `example.tld.key`.

```
(System/Certificate Manager/Certificates/"Add/Sign Button"/Method "Import an existing Certificate")

(System/Advanced/Admin Access/SSL/TLS Certificate)
```

### Let's Encrypt SSL Certificate on Apache Web Server

```bash
mv fullchain.cer /etc/ssl/certs/exampleTLD.fullchain.cer
mv example.tld.cer /etc/ssl/certs/
mv example.tld.key /etc/ssl/private/
chmod 0444 /etc/ssl/certs/{example.tld,xampleTLD.fullchain}.cer
chmod 0400 /etc/ssl/private/example.tld.key
```
```bash
nano /etc/apache2/sites-available/exampleTLD.conf

...
SSLEngine on
SSLCertificateFile /etc/ssl/certs/example.tld.cer
SSLCertificateKeyFile /etc/ssl/private/example.tld.key
SSLCACertificateFile "/etc/ssl/certs/exampleTLD.fullchain.cer"
SSLCACertificatePath "/etc/ssl/certs/"
SSLProtocol -all +TLSv1.3 +TLSv1.2
SSLOpenSSLConfCmd Curves X25519:secp521r1:secp384r1:prime256v1
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
#SSLCipherSuite RC4-SHA:AES128-SHA:HIGH:!aNULL:!MD5
SSLHonorCipherOrder On
Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
SSLCompression off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
SSLSessionTickets Off
...
```

Test settings, if syntax returns `OK`, restart the web service:

```bash
apache2ctl -t
systemctl restart apache2.service
```

### Let's Encrypt SSL Certificate on Nginx Web Server

```bash
mv fullchain.cer /etc/ssl/certs/exampleTLD.fullchain.cer
mv example.tld.key /etc/ssl/private/
chmod 0444 /etc/ssl/certs/exampleTLD.fullchain.cer
chmod 0400 /etc/ssl/private/example.tld.key
```
```bash
nano /etc/nginx/sites-available/exampleTLD

...
ssl on;
ssl_certificate /etc/ssl/certs/exampleTLD.fullchain.cer;
ssl_certificate_key /etc/ssl/private/example.tld.key;
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH;
ssl_prefer_server_ciphers on;
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/ssl/certs/exampleTLD.fullchain.cer;
...
```

Test settings, if syntax returns `OK`, restart the web service:

```bash
nginx -t
systemctl restart nginx.service
```

## References

* [acme.sh A pure Unix shell script implementing ACME client protocol](https://github.com/acmesh-official/acme.sh)
* [cerbot](https://certbot.eff.org/)
* [Installing a Let's Encrypt SSL Certificate](https://wiki.zimbra.com/wiki/Installing_a_LetsEncrypt_SSL_Certificate)
* [Deploy Commercial SSL Certificate on Proxmox Mail Gateway](https://dhenandi.com/deploy-commercial-ssl-certificate-on-proxmox-mail-gateway/)
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
