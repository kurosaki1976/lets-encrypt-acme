# Instalación y configuración de acme.sh

## Instalar paquetes necesarios

	apt install git socat dnsutils bsd-mailx

## Instalar acme.sh

	git clone https://github.com/acmesh-official/acme.sh.git
	cd acme.sh
	./acme.sh --install \
		--home /opt/acme.sh \
		--config-home /opt/acme.sh/data \
		--cert-home /opt/acme.sh/certs \
		--accountemail "yoel@iju.jovenclub.cu" \
		--accountkey /opt/acme.sh/jabber.iju.jovenclub.cu.key \
		--accountconf /opt/acme.sh/jabber.iju.jovenclub.cu.conf

## Crear archivo de dnskey

	vim /opt/acme.sh/iju.key

	key "iju" {
	  ...
	};

## Obtener certificados

### Exportar variables de entorno necesarias

	export NSUPDATE_SERVER=10.20.5.14
	export NSUPDATE_KEY=/opt/acme.sh/iju.key

### Comprobar

	acme.sh --issue \
		-d mail.iju.jovenclub.cu --challenge-alias mail.iju.jovenclub.cu \
		-d correo.iju.jovenclub.cu --challenge-alias correo.iju.jovenclub.cu \
		-d imap.iju.jovenclub.cu --challenge-alias imap.iju.jovenclub.cu \
		-d imaps.iju.jovenclub.cu --challenge-alias imaps.iju.jovenclub.cu \
		-d pop3.iju.jovenclub.cu --challenge-alias pop3.iju.jovenclub.cu \
		-d pop3s.iju.jovenclub.cu --challenge-alias pop3s.iju.jovenclub.cu \
		-d smtp.iju.jovenclub.cu --challenge-alias smtp.iju.jovenclub.cu \
		-d jabber.iju.jovenclub.cu --challenge-alias jabber.iju.jovenclub.cu \
		-d pubsub.jabber.iju.jovenclub.cu --challenge-alias pubsub.jabber.iju.jovenclub.cu \
		-d conference.jabber.iju.jovenclub.cu --challenge-alias conference.jabber.iju.jovenclub.cu \
		-d echo.jabber.iju.jovenclub.cu --challenge-alias echo.jabber.iju.jovenclub.cu \
		--dns dns_nsupdate --dnssleep 60 --notify-level 2 \
		--notify-mode 0 --notify-hook mail --test

### Obtener

	acme.sh --issue \
		-d mail.iju.jovenclub.cu --challenge-alias mail.iju.jovenclub.cu \
		-d correo.iju.jovenclub.cu --challenge-alias correo.iju.jovenclub.cu \
		-d imap.iju.jovenclub.cu --challenge-alias imap.iju.jovenclub.cu \
		-d imaps.iju.jovenclub.cu --challenge-alias imaps.iju.jovenclub.cu \
		-d pop3.iju.jovenclub.cu --challenge-alias pop3.iju.jovenclub.cu \
		-d pop3s.iju.jovenclub.cu --challenge-alias pop3s.iju.jovenclub.cu \
		-d smtp.iju.jovenclub.cu --challenge-alias smtp.iju.jovenclub.cu \
		-d jabber.iju.jovenclub.cu --challenge-alias jabber.iju.jovenclub.cu \
		-d pubsub.jabber.iju.jovenclub.cu --challenge-alias pubsub.jabber.iju.jovenclub.cu \
		-d conference.jabber.iju.jovenclub.cu --challenge-alias conference.jabber.iju.jovenclub.cu \
		-d echo.jabber.iju.jovenclub.cu --challenge-alias echo.jabber.iju.jovenclub.cu \
		--dns dns_nsupdate --dnssleep 60 --notify-level 2 \
		--notify-mode 0 --notify-hook mail --force

		--key-file /etc/ejabberd/key.pem \
		--fullchain-file /etc/ejabberd/cert.pem \
		--reloadcmd "ejabberdctl reload-config"

## Instalar tarea de cron

	acme.sh --install-cronjob

## Configurar proxy

Añadir en `/opt/acme.sh/acme.sh.env`:

	export HTTPS_PROXY="http://proxy.iju.jovenclub.cu:3128"

## Configurar notificaciones

	export MAIL_BIN="mail"
	export MAIL_FROM="postmaster@iju.jovenclub.cu"
	export MAIL_TO="yoel@iju.jovenclub.cu"

	acme.sh --set-notify --notify-hook mail --notify-level 2 --notify-mode 0



jabber.iju.jovenclub.cu
conference.jabber.iju.jovenclub.cu
pubsub.jabber.iju.jovenclub.cu
echo.jabber.iju.jovenclub.cu
mail.iju.jovenclub.cu
correo.iju.jovenclub.cu
imap.iju.jovenclub.cu
imaps.iju.jovenclub.cu
pop3.iju.jovenclub.cu
pop3s.iju.jovenclub.cu
smtp.iju.jovenclub.cu