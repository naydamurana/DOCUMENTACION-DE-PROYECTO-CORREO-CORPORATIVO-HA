# Proyecto Final SIS313: IMPLEMENTACIÓN DE CORREO CORPORATIVO DE ALTA DISPONIBILIDAD

**Asignatura:** SIS313: Infraestructura, Plataformas Tecnológicas y Redes  
**Semestre:** 2/2025  
**Docente:** Ing. Marcelo Quispe Ortega  
**Fecha del proyecto:** 21/08/2025  
**Integrantes:**  
- Coraite Yanaje Luz Clara (Maestro)  
- Muraña Pizarro Nayda Thatiana (Esclavo)  

---

## Introducción

El presente proyecto desarrolla la implementación completa de una plataforma de correo corporativo con alta disponibilidad (HA) utilizando tecnologías de software libre. Se integran servicios esenciales como SMTP, IMAP, POP3, Webmail y DNS dentro de una arquitectura redundante que garantiza continuidad operativa incluso ante fallos críticos de hardware o software, reforzada mediante mecanismos de seguridad como cifrado TLS, autenticación de dominio (SPF/DKIM/DMARC) y filtros de contenido (ClamAV / SpamAssassin). El diseño permite que la infraestructura responda a los exigentes niveles de confiabilidad, robustez y escalabilidad que requieren las organizaciones modernas.

---

## Máquinas del Proyecto (Roles y Responsabilidades)

| Nombre Completo | Rol en el Proyecto | Sistema Operativo | Responsabilidades Clave |
|-----------------|--------------------|-------------------|-------------------------|
| Coraite Yanaje Luz Clara | Maestro | **Ubuntu Server 22.04** | Despliegue y configuración de la **VM Maestro (mail01)**: instalación y ajuste de **iRedMail / Postfix / Dovecot / Roundcube** y configuración de **Keepalived en modo MASTER** para gestión de la VIP. Supervisión de servicios críticos y validación del nodo activo. |
| Muraña Pizarro Nayda Thatiana | Esclavo | **Ubuntu Server 22.04** | Despliegue y configuración de la **VM Esclavo (mail02)** con servicios espejo al Maestro. Configuración de **Keepalived en modo BACKUP** y validación de failover y acceso a buzones vía NFS. |
| Servidor Debian (storage01) | Backup / Infraestructura | **Debian** | Servidor orientado a soporte: **NFS** para buzones Maildir, **Bind9** (DNS corporativo), **ISC DHCP** (asignación IP). Garantiza persistencia de datos, resolución interna y disponibilidad de recursos de red. |

---

## Objetivo

Implementar una solución de correo electrónico empresarial con Alta Disponibilidad mediante servidores redundantes con failover automático, almacenamiento compartido, autenticación integrada y protección antispam/antivirus, permitiendo gestionar buzones corporativos, listas de distribución y accesos seguros para que el servicio permanezca operativo y accesible aun ante fallos en cualquiera de sus componentes.

---

## Justificación

El correo electrónico es un sistema crítico institucional. Su interrupción implica pérdida de productividad, fallos en la comunicación y riesgo operacional. Por ello se implementa una arquitectura Activo–Pasivo que elimina el Punto Único de Falla (SPOF), incorpora mecanismos de seguridad para proteger integridad, disponibilidad y confidencialidad y asegura la continuidad operacional mediante redundancia y failover automatizado.

---

## Tecnologías implementadas

| Categoría | Software / Tecnología | Función en el Proyecto |
|-----------|------------------------|------------------------|
| Tecnologías Principales | **iRedMail** | Suite de correo empresarial que integra Postfix, Dovecot, Amavisd, Nginx/Apache, MariaDB/PostgreSQL y herramientas de administración. Facilita despliegue completo en Maestro y Esclavo. |
| Servidores / SO | **Ubuntu Server 22.04** (Maestro y Esclavo) / **Debian** (Backup) | Ubuntu ejecuta iRedMail; Debian ofrece NFS, DNS y DHCP. |
| Capa de Aplicación | **Nginx / Apache** (según instalación de iRedMail) | Servidor web para Webmail y panel de administración. |
| Webmail | **Roundcube** (incluido en iRedMail) | Interfaz web para usuarios. |
| MTA / MDA | **Postfix / Dovecot** | Postfix como MTA (SMTP), Dovecot como MDA (IMAP/POP3). |
| Antivirus / Antispam | **Amavisd + ClamAV + SpamAssassin** | Filtrado y protección del flujo de correo. |
| Base de Datos | **MariaDB / PostgreSQL** | Almacena usuarios, dominios y configuraciones (según instalación). |
| Alta Disponibilidad (HA) | **Keepalived + VRRP** | Failover automático de la IP Virtual (VIP). |
| Almacenamiento Compartido | **NFS (Debian)** | Almacena Maildir accesible por Maestro y Esclavo. |
| DNS Corporativo | **Bind9 (Debian)** | Zona `chocolatesparati.com.bo`, A/MX/TXT (SPF). |
| Asignación de Red | **ISC DHCP Server (Debian)** | Entrega IPs y parámetros de red a los clientes. |

---

## Temas de la Asignatura Puestos en Práctica (T1–T5)

### T1 – Fundamentos de la Continuidad Operacional (CO)
- Identificación del SPOF: análisis de riesgos al usar un único servidor de correo; justificación del esquema Maestro–Esclavo.  
- Aseguramiento del servicio crítico: redundancia de servidores, VIP y almacenamiento compartido garantizan continuidad operativa.

### T2 – Alta Disponibilidad (HA) y Tolerancia a Fallos
- Failover Activo–Pasivo con VRRP (Keepalived): VIP que conmuta SMTP/IMAP/Webmail automáticamente.  
- Redundancia operacional mediante NFS en Debian en lugar de RAID10 o replicación compleja.  
- Arquitectura distribuida (Maestro, Esclavo, Backup) para mitigar fallos en múltiples capas.

### T4 – Optimización y Servicios Complejos
- Configuración integrada de SMTP / IMAP / POP3 usando iRedMail (Postfix + Dovecot + Roundcube).  
- Ajustes de rendimiento: tamaño de mensajes, límites, mail_location y políticas de autenticación.  
- Orquestación de servicios (iRedMail, Amavisd, ClamAV, SpamAssassin, DNS, DHCP, Keepalived).

### T5 – Seguridad y Hardening
- Cifrado TLS obligatorio para Postfix y Dovecot.  
- Publicación de SPF en DNS; implementación de DKIM y DMARC recomendada.  
- Integración Amavisd + ClamAV + SpamAssassin para defensa en profundidad.  
- Hardening general: UFW/iptables, permisos, deshabilitar autenticación insegura y monitoreo.

---

## Estrategia Adoptada para la Alta Disponibilidad

Arquitectura Maestro–Esclavo (Activo–Pasivo) con **Keepalived (VRRP)** gestionando una **IP Virtual (VIP: 192.168.100.100)**. El Maestro atiende tráfico; el Esclavo permanece en standby y asume la VIP al detectar fallo. El servidor Debian proporciona NFS para buzones, Bind9 y DHCP, evitando replicación de almacenamiento dentro de los nodos de correo y simplificando la coherencia de buzones.

---

## Componentes Clave y Función

| Componente | Función en la Arquitectura | Tecnología Clave |
|------------|----------------------------|------------------|
| Balanceador / Failover | Mantener VIP única apuntando al servidor activo | Keepalived (VRRP) |
| Servidor Maestro (Activo) | Atiende SMTP, IMAP, Webmail | Ubuntu + iRedMail |
| Servidor Esclavo (Pasivo) | Standby y asume servicio si Maestro falla | Ubuntu + iRedMail |
| Servidor Backup (Debian) | NFS, DNS (Bind9), DHCP | Debian + NFS + Bind9 + ISC-DHCP |
| Firewall | Controla puertos esenciales (25, 465, 587, 143, 993, 110, 995, 80, 443) | UFW / iptables |
| Antivirus / Antispam | Inspección y filtrado de mensajes | Amavisd + ClamAV + SpamAssassin |

---

## Guía de Implementación y Puesta en Marcha (versión completa y práctica)

> **Nota:** adapta nombres de interfaces (e.g., `ens33`) y dominios a tu entorno.

### Pre-requisitos
- 3 VMs: VM1 (mail01 — Ubuntu 22.04), VM2 (mail02 — Ubuntu 22.04), VM3 (storage01 — Debian).  
- Red interna `192.168.100.0/24`. VIP sugerida: `192.168.100.100`.  
- Usuarios con sudo.  
- Certificados TLS (snakeoil para pruebas; Let's Encrypt en producción).  
- Paquetes base: `curl`, `net-tools`, `nfs-common`/`nfs-kernel-server`, `keepalived`, `postfix`, `dovecot`, `roundcube`, `apache2` o `iRedMail` instalador.
