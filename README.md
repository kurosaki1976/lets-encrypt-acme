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
_acme-challenge.example.tld. 60 IN TXT "NQ9KX3PSo0T_qhIKyAYQoBq7XRng3WwfnV58YyeI9k0"
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

## References

* [acme.sh A pure Unix shell script implementing ACME client protocol](https://github.com/acmesh-official/acme.sh)
