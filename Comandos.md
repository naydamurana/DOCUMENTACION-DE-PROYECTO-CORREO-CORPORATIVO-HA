# 📧 Proyecto: Servidor de Correo de Alta Disponibilidad (HA) y Seguro

Este proyecto implementa una solución de correo electrónico robusta y redundante utilizando Postfix, Dovecot, Roundcube, Keepalived, NFS, BIND9, y protecciones Antispam/Antivirus.

### I. Configuración de VM1 (`mail01`) - Nodo Master

```bash
# 1. Configuración de Red (Netplan)
sudo nano /etc/netplan/50-cloud-init.yaml
# (Contenido del archivo Netplan para 192.168.100.2)
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.100.2/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8]

# 2. Hostname y Dominio
sudo hostnamectl set-hostname mail01
sudo nano /etc/hosts
# 127.0.0.1      localhost
# 192.168.100.2  mail01.chocolatesparati.com.bo mail01
sudo nano /etc/mailname
# chocolatesparati.com.bo

# 3. Instalación de Servicios
apt update
apt install postfix dovecot-core dovecot-imapd dovecot-pop3d apache2 mariadb-server roundcube roundcube-mysql keepalived nfs-common -y

# 4. Configuración Postfix (General)
postconf -e 'myhostname = mail01.chocolatesparati.com.bo'
postconf -e 'mydestination = $myhostname, chocolatesparati.com.bo, mail01, localhost.localdomain, localhost'
postconf -e 'mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.100.0/24'
postconf -e 'message_size_limit = 52428800'

# 5. Configuración Postfix (SASL/Dovecot)
sudo nano /etc/postfix/main.cf
# smtpd_sasl_type = dovecot
# smtpd_sasl_path = private/auth
# smtpd_sasl_auth_enable = yes
# smtpd_sasl_security_options = noanonymous
# smtpd_sasl_local_domain = $myhostname
# smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination

# 6. Configuración Dovecot
sudo nano /etc/dovecot/conf.d/10-auth.conf
# disable_plaintext_auth = no
# auth_username_format = %n
sudo nano /etc/dovecot/conf.d/10-master.conf
# Configuración del socket de autenticación
# service auth {
#   unix_listener /var/spool/postfix/private/auth {
#     mode = 0666
#     user = postfix
#     group = postfix
#   }
# }

# 7. Configuración Keepalived (MASTER)
sudo nano /etc/keepalived/keepalived.conf
# vrrp_script check_script {
#   script "/usr/bin/killall -0 apache2"
#   interval 2
#   weight -20
# }
# vrrp_instance VI_1 {
#   state MASTER
#   interface ens33
#   virtual_router_id 51
#   priority 100
#   advert_int 1
#   authentication {
#     auth_type PASS
#     auth_pass 1111
#   }
#   virtual_ipaddress {
#     192.168.100.100/24
#   }
#   track_script {
#     check_script
#   }
# }
sudo systemctl restart keepalived

# 8. Integración con NFS (Almacenamiento Compartido)
sudo mkdir -p /var/vmail_mount
sudo mount 192.168.100.4:/srv/vmail /var/vmail_mount
sudo nano /etc/fstab
# 192.168.100.4:/srv/vmail /var/vmail_mount nfs defaults 0 0
sudo postconf -e 'home_mailbox ='
sudo postconf -e 'mail_spool_directory = /var/vmail_mount/'
sudo nano /etc/dovecot/conf.d/10-mail.conf
# mail_location = maildir:/var/vmail_mount/%u
systemctl restart postfix
systemctl restart dovecot
```
### II.Configuración de VM2 (mail02) - Nodo Esclavo
```bash
# 1. Configuración de Red (Netplan)
sudo nano /etc/netplan/50-cloud-init.yaml
# (Contenido del archivo Netplan para 192.168.100.3)
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.100.3/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8]

# 2. Hostname y Hosts
sudo hostnamectl set-hostname mail02
sudo nano /etc/hosts
# 127.0.0.1      localhost
# 192.168.100.2  mail01.chocolatesparati.com.bo mail01
# 192.168.100.3  mail02.chocolatesparati.com.bo mail02

# 3. Instalación de Servicios (Igual que VM1)
# ... omitido para brevedad (incluye postfix, dovecot, apache2, keepalived, nfs-common)

# 4. Configuración Postfix (General)
sudo postconf -e 'myhostname = mail02.chocolatesparati.com.bo' # ¡Diferente hostname!
# ... (El resto de la configuración de Postfix es similar a VM1)

# 5. Configuración Keepalived (BACKUP)
sudo nano /etc/keepalived/keepalived.conf
# vrrp_script check_script { ... } # Igual que VM1
# vrrp_instance VI_1 {
#   state BACKUP                    # ¡Diferente estado!
#   priority 90                     # ¡Diferente prioridad!
#   ... (El resto es igual)
# }
sudo systemctl restart keepalived

# 6. Integración con NFS
# (Los comandos de montaje NFS y configuración de Postfix/Dovecot son idénticos a VM1)
```
### III. Configuración de VM3 (storage01) - NFS, DNS, DHCP
```bash
# 1. Configuración de Red (Interfaces de Debian)
nano /etc/network/interfaces
# iface ens33 inet static
#         address 192.168.100.4
#         netmask 255.255.255.0
#         gateway 192.168.100.1
#         dns-nameservers 8.8.8.8 192.168.100.4
systemctl restart networking

# 2. Hosts
hostnamectl set-hostname storage01
nano /etc/hosts
# 192.168.100.4  storage01.chocolatesparati.com.bo storage01
# 192.168.100.2  mail01.chocolatesparati.com.bo mail01
# 192.168.100.3  mail02.chocolatesparati.com.bo mail02

# 3. Servidor NFS
apt install nfs-kernel-server -y
mkdir -p /srv/vmail
chmod 777 /srv/vmail
chown nobody:nogroup /srv/vmail
nano /etc/exports
# /srv/vmail 192.168.100.0/24(rw,sync,no_subtree_check)
exportfs -a
systemctl restart nfs-kernel-server

# 4. Servidor BIND9 (DNS)
apt install bind9 bind9-utils -y
# Configuración en named.conf.options y named.conf.local
# (Configuración detallada de la Zona: db.chocolates)
nano /etc/bind/db.chocolates
# @   IN    MX  10  mail.chocolatesparati.com.bo.
# mail      IN    A         192.168.100.100   # VIP
# www       IN    A         192.168.100.100   # VIP
# @   IN    TXT   "v=spf1 mx a:mail.chocolatesparati.com.bo ip4:192.168.100.0/24 -all" # Registro SPF
systemctl restart bind9

# 5. Servidor DHCP
apt install isc-dhcp-server -y
nano /etc/default/isc-dhcp-server
# INTERFACESv4="ens33"
nano /etc/dhcp/dhcpd.conf
# subnet 192.168.100.0 netmask 255.255.255.0 {
#   range 192.168.100.11 192.168.100.99;
#   option domain-name-servers 192.168.100.4, 8.8.8.8;
#   option routers 192.168.100.1;
#   # ... (Otras opciones de subred)
# }
systemctl restart isc-dhcp-server
```
### IV. Configuraciones Comunes (Roundcube, Seguridad, Listas)
```bash
  # 1. Configuración de Roundcube
sudo nano /etc/roundcube/config.inc.php
# $config['smtp_host'] = 'localhost:25';
# $config['imap_host'] = ["localhost:143"];
# $config['username_domain'] = 'chocolatesparati.com.bo';
# $config['mail_domain'] = 'chocolatesparati.com.bo';
# $config['product_name'] = 'Correo Chocolates Para Ti';

# 2. Acceso Directo al Webmail (Root de Apache)
sudo nano /etc/apache2/sites-available/000-default.conf
# DocumentRoot /usr/share/roundcube # Cambio de /var/www/html a Roundcube
sudo systemctl restart apache2

# 3. Antivirus y Antispam (Amavis, ClamAV, SpamAssassin)
sudo apt install amavisd-new spamassassin clamav-daemon ... -y
sudo postconf -e 'content_filter = smtp-amavis:[127.0.0.1]:10024'
sudo nano /etc/postfix/master.cf
# (Añadir las entradas smtp-amavis y 127.0.0.1:10025)
sudo systemctl restart postfix
sudo systemctl restart amavis

# 4. Hardening: SSL/TLS (Usando certificado snakeoil)
# Postfix TLS
sudo postconf -e 'smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem'
sudo postconf -e 'smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key'
sudo postconf -e 'smtpd_use_tls=yes'
# Dovecot TLS
sudo nano /etc/dovecot/conf.d/10-ssl.conf
# ssl = required
# ssl_cert = </etc/ssl/certs/ssl-cert-snakeoil.pem
# ssl_key = </etc/ssl/private/ssl-cert-snakeoil.key
sudo systemctl restart postfix
sudo systemctl restart dovecot

# 5. Crear usuarios y Listas de Distribución
sudo adduser ventas # Crear el usuario
sudo nano /etc/aliases
# todos: nayda, ventas
sudo newaliases
```
### V. Verificaciones Cruciales
```bash
# 1. Verificar DNS y VIP (Desde VM3 o cliente)
nslookup mail.chocolatesparati.com.bo 192.168.100.4

# 2. Verificar Montaje NFS (En VM1/VM2)
df -h | grep vmail

# 3. Probar Failover (En VM1 o VM2)
# Simular la caída del servicio web en el Master:
sudo systemctl stop apache2 

# 4. Probar Antivirus (En VM1/VM2 - ver logs mientras se envía el EICAR)
tail -f /var/log/mail.log | grep -E "amavis|Blocked"
# Enviar el siguiente texto por correo para prueba AV:
# X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*

# 5. Conexión TLS (Desde VM3 o cliente)
openssl s_client -connect 192.168.100.100:25 -starttls smtp
