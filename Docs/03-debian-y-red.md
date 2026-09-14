# 03 - Debian y red

## 1. Objetivo

`hpserver` utiliza **Debian 13 (Trixie)** como sistema operativo base.

El objetivo de la instalación es disponer de un sistema:

- estable;
- ligero;
- administrable principalmente mediante terminal;
- adecuado para ejecución continua;
- compatible con Docker;
- accesible remotamente mediante SSH;
- con una configuración de red predecible;
- con el menor número posible de componentes innecesarios.

Debian se instala directamente sobre el SSD SYSTEM. Los servicios de aplicación, como Nextcloud, PostgreSQL y Redis, se ejecutan posteriormente mediante Docker.

> [!NOTE]
> Los valores específicos de la instalación pública, como direcciones IP, nombres de usuario o dominios, se sustituyen en este documento por ejemplos o variables.

---

## 2. Sistema operativo

La instalación utiliza:

```text
Debian GNU/Linux 13 (Trixie)
Arquitectura: amd64
Kernel: Linux 6.12
```

No se instala un entorno gráfico de escritorio.

La administración habitual se realiza mediante:

```text
SSH
 │
 ▼
Terminal
 │
 ├── systemctl
 ├── journalctl
 ├── Docker
 ├── smartctl
 └── herramientas GNU/Linux
```

La ausencia de escritorio reduce el número de servicios y paquetes que no aportan funcionalidad al propósito principal del equipo.

No obstante, al tratarse de un portátil, la pantalla y el teclado integrados permanecen disponibles para administración local en caso de pérdida de conectividad.

---

## 3. Identificación del servidor

El servidor utiliza un hostname fijo:

```text
hpserver
```

Puede comprobarse mediante:

```bash
hostname
```

o:

```bash
hostnamectl
```

El nombre identifica al equipo independientemente de la dirección IP que utilice.

Una configuración equivalente en otra instalación podría realizarse mediante:

```bash
sudo hostnamectl set-hostname hpserver
```

El hostname aparece también en distintos componentes del sistema de administración, como los logs y los correos automáticos de alerta.

---

## 4. Usuario administrativo y `sudo`

La administración diaria no se realiza iniciando sesión directamente como `root`.

Se utiliza un usuario normal con permisos administrativos mediante `sudo`.

El modelo es:

```text
usuario administrador
        │
        │ sudo
        ▼
operación privilegiada
```

Esto permite trabajar normalmente con privilegios limitados y elevarlos únicamente cuando una operación lo necesita.

### Incidencia: `sudo` no estaba instalado

Después de la instalación inicial se detectó que el comando:

```bash
sudo
```

no estaba disponible.

La situación se resolvió accediendo temporalmente como `root` e instalando el paquete:

```bash
apt update
apt install sudo
```

Posteriormente se añadió el usuario administrativo al grupo correspondiente:

```bash
usermod -aG sudo <ADMIN_USER>
```

Tras volver a iniciar sesión se comprobó el funcionamiento mediante:

```bash
sudo whoami
```

con el resultado esperado:

```text
root
```

### Lección

En una instalación mínima de Debian no debe asumirse que `sudo` estará necesariamente disponible o configurado para el usuario creado.

Antes de continuar con la administración remota conviene comprobar:

```bash
command -v sudo
groups
sudo -v
```

---

## 5. Acceso mediante SSH

OpenSSH permite administrar `hpserver` desde otro equipo de la red.

El servicio puede comprobarse mediante:

```bash
systemctl status ssh
```

y las conexiones siguen el esquema:

```text
Equipo administrador
        │
        │ SSH
        ▼
     hpserver
        │
        ▼
usuario no root
        │
        │ sudo
        ▼
administración
```

La configuración principal del servidor OpenSSH se encuentra en:

```text
/etc/ssh/sshd_config
```

y puede complementarse mediante archivos en:

```text
/etc/ssh/sshd_config.d/
```

### Acceso de `root`

Durante las primeras tareas de configuración fue necesario disponer temporalmente de acceso administrativo directo.

Una vez creado y comprobado un usuario con `sudo`, la administración habitual se realiza con ese usuario.

El acceso SSH directo como `root` no forma parte del flujo normal de administración.

Esta separación reduce la necesidad de utilizar permanentemente la cuenta con privilegios máximos.

> [!IMPORTANT]
> Antes de restringir el acceso remoto de `root`, debe comprobarse desde una segunda sesión que el usuario administrativo puede iniciar sesión correctamente y utilizar `sudo`. De lo contrario, un error de configuración podría dejar el servidor sin una vía de administración remota.

---

## 6. Actualización del sistema

Después de la instalación se actualizó el sistema utilizando los repositorios configurados de Debian.

El procedimiento habitual es:

```bash
sudo apt update
sudo apt upgrade
```

Cuando corresponde realizar una actualización más completa de dependencias:

```bash
sudo apt full-upgrade
```

El sistema debe mantenerse actualizado, especialmente en componentes expuestos a datos procedentes de la red, como:

- OpenSSH;
- Docker;
- kernel;
- bibliotecas del sistema;
- herramientas de seguridad.

Las actualizaciones importantes deben acompañarse de comprobaciones posteriores para confirmar que los servicios continúan funcionando correctamente.

---

## 7. Interfaz de red

La interfaz Ethernet utilizada por el servidor se identifica mediante un nombre predecible de Linux similar a:

```text
enp8s0
```

La configuración puede inspeccionarse mediante:

```bash
ip addr
```

y:

```bash
ip link
```

La interfaz cableada es la utilizada para proporcionar conectividad permanente al servidor.

Para un servicio de almacenamiento doméstico se prioriza Ethernet frente a Wi-Fi por su estabilidad y comportamiento más predecible.

---

## 8. Diseño de la red local

La red doméstica utiliza una subred privada IPv4 `/24`.

En la documentación pública utilizaremos:

```text
Red:       192.168.1.0/24
Gateway:   192.168.1.1
Servidor:  <SERVER_LAN_IP>
```

La dirección real del servidor se conserva únicamente en la documentación privada de la instalación.

El objetivo es que `hpserver` mantenga siempre la misma dirección dentro de la LAN.

Esto resulta especialmente importante porque diferentes componentes dependen de una ubicación predecible:

```text
clientes locales
      │
      ▼
SERVER_LAN_IP
      │
      ▼
   hpserver
```

---

## 9. Dirección IP estática

Durante la instalación inicial el servidor obtuvo una dirección mediante DHCP.

Posteriormente se configuró una dirección estática.

La dirección elegida se encuentra **fuera del pool DHCP del router**.

El principio es:

```text
LAN
│
├── direcciones reservadas / estáticas
│        └── hpserver
│
└── pool DHCP
         ├── móviles
         ├── ordenadores
         └── otros dispositivos
```

Esto reduce el riesgo de que el router asigne accidentalmente la misma dirección a otro dispositivo.

### Configuración con `dhcpcd`

La instalación utiliza `dhcpcd` para gestionar la configuración de la interfaz.

La configuración se encuentra en:

```text
/etc/dhcpcd.conf
```

Una versión saneada equivalente es:

```text
interface <SERVER_INTERFACE>
static ip_address=<SERVER_LAN_IP>/24
static routers=<ROUTER_IP>
static domain_name_servers=<ROUTER_IP>
```

Por ejemplo, en una red de laboratorio:

```text
interface enp8s0
static ip_address=192.168.1.10/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1
```

`dhcpcd` permite asignar explícitamente una dirección estática, gateway y servidores DNS a una interfaz.

> [!NOTE]
> Esta configuración documenta la implementación concreta de `hpserver`. Debian admite otros mecanismos de configuración de red y no debe interpretarse `dhcpcd` como la única opción disponible.

---

## 10. Validación de la red

Una configuración de red no se considera terminada simplemente porque el archivo de configuración pueda guardarse.

Se deben comprobar independientemente:

```text
interfaz
   +
dirección IP
   +
ruta por defecto
   +
DNS
   +
conectividad LAN
   +
conectividad Internet
```

### Dirección

```bash
ip addr show <SERVER_INTERFACE>
```

Debe aparecer la dirección esperada.

### Ruta

```bash
ip route
```

Debe existir una ruta por defecto similar a:

```text
default via <ROUTER_IP> dev <SERVER_INTERFACE>
```

### Conectividad con el router

```bash
ping -c 4 <ROUTER_IP>
```

### Conectividad IP externa

Por ejemplo:

```bash
ping -c 4 1.1.1.1
```

Esto permite comprobar conectividad sin depender todavía de DNS.

### Resolución DNS

Por ejemplo:

```bash
getent hosts debian.org
```

Una resolución correcta confirma que el sistema puede convertir nombres en direcciones.

### Validación tras reinicio

Finalmente:

```bash
sudo reboot
```

Después del arranque se repiten las comprobaciones.

Este último paso es importante: una configuración que funciona inmediatamente después de modificarla no está necesariamente correctamente integrada en el proceso de arranque.

---

## 11. DNS

El router actúa como resolver DNS principal para `hpserver`.

De forma simplificada:

```text
hpserver
   │
   ▼
router
   │
   ▼
DNS externo
```

Durante el diseño se valoró utilizar `hpserver` como servidor DNS para otros dispositivos de la vivienda.

Finalmente se decidió no convertirlo en una dependencia crítica de la red doméstica.

### Motivo

Si los dispositivos de la vivienda dependieran exclusivamente de `hpserver` para resolver nombres:

```text
hpserver apagado
       │
       ▼
DNS no disponible
       │
       ▼
otros dispositivos parecen
perder acceso a Internet
```

aunque la conexión del router siguiera funcionando.

La función principal de `hpserver` es proporcionar almacenamiento y servicios relacionados, no convertirse en un requisito para que el resto de la red pueda navegar.

Por ello se mantiene desacoplado el servicio DNS doméstico del servidor Nextcloud.

---

## 12. Dependencia entre red y Docker

Durante las pruebas apareció una consecuencia importante del orden de arranque entre la red y Docker.

Inicialmente Nextcloud publicaba su puerto mediante una asociación específica:

```text
<SERVER_LAN_IP>:8080:80
```

Después de un reinicio, Docker intentó iniciar el contenedor antes de que `dhcpcd` hubiera asignado la dirección estática al host.

El resultado fue un error equivalente a:

```text
failed to bind host port:
cannot assign requested address
```

### Causa

La secuencia era aproximadamente:

```text
arranque
   │
   ├── Docker inicia
   │      │
   │      └── intenta usar SERVER_LAN_IP
   │
   └── dhcpcd todavía no ha configurado SERVER_LAN_IP
```

Docker no podía asociar el puerto a una dirección que todavía no existía en la interfaz.

### Solución

La publicación se modificó a:

```yaml
ports:
  - "8080:80"
```

en lugar de:

```yaml
ports:
  - "<SERVER_LAN_IP>:8080:80"
```

De esta forma Docker no depende de que la IP concreta haya sido asignada antes de iniciar el contenedor.

### Resultado

Tras aplicar el cambio se realizó un reinicio completo y se comprobó:

- asignación correcta de la IP;
- inicio automático de Docker;
- inicio de Nextcloud;
- acceso desde LAN;
- acceso remoto;
- ausencia del error de `bind`.

### Lección

Los problemas de arranque pueden ser consecuencia no sólo de una configuración incorrecta, sino también de **dependencias temporales entre servicios correctamente configurados individualmente**.

---

## 13. Resolución DNS dentro de Docker

Durante la configuración SMTP apareció una segunda incidencia relacionada con el arranque de la red.

Nextcloud no podía resolver el servidor SMTP aunque Debian sí disponía de conectividad y resolución DNS.

Dentro del contenedor aparecía:

```text
nameserver 127.0.0.11
```

pero el resolver interno de Docker no disponía correctamente de servidores DNS externos.

El journal de Docker mostró un aviso indicando que no se habían definido resolvers externos.

### Diagnóstico

La diferencia fue importante:

```text
HOST Debian
   │
   └── DNS correcto

CONTENEDOR
   │
   └── DNS incorrecto
```

Por tanto, el fallo SMTP no estaba inicialmente en:

```text
Brevo
credenciales
TLS
Nextcloud
```

sino en la resolución DNS del contenedor.

Reiniciar Docker después de que la red estuviera disponible restauró temporalmente la resolución.

### Configuración persistente

Para evitar depender del estado DNS detectado durante el arranque, Docker dispone de una configuración explícita de resolvers.

Una versión pública equivalente de:

```text
/etc/docker/daemon.json
```

es:

```json
{
  "dns": ["<ROUTER_IP>", "<FALLBACK_DNS>"]
}
```

La configuración real utiliza el resolver de la red local y un resolver externo de respaldo.

### Lección

Al diagnosticar problemas de red en servicios Docker deben comprobarse por separado:

```text
host
  │
  └── conectividad / DNS

contenedor
  │
  └── conectividad / DNS
```

Que el host pueda resolver un nombre no demuestra que un contenedor pueda hacerlo.

---

## 14. Comportamiento de la tapa

Debido a que el servidor es un portátil, cerrar la tapa podría provocar la suspensión automática del sistema.

Ese comportamiento no es adecuado para un servidor que debe continuar proporcionando servicios mientras permanece físicamente cerrado.

`systemd-logind` permite definir la acción asociada al cierre de la tapa.

Se creó un archivo específico:

```text
/etc/systemd/logind.conf.d/10-hpserver-lid.conf
```

con:

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

El uso de un archivo *drop-in* permite mantener la modificación separada de la configuración principal distribuida por systemd.

### Validación

Después de aplicar la configuración se reinició completamente el servidor.

Posteriormente se comprobó:

- Debian arrancó correctamente;
- Docker inició automáticamente;
- todos los contenedores necesarios estaban activos;
- los timers systemd permanecían habilitados;
- Nextcloud era accesible;
- cerrar físicamente la tapa no suspendía el servidor.

### Motivo de las tres opciones

Se define explícitamente el comportamiento tanto para funcionamiento normal como con alimentación externa o en situaciones que systemd pueda interpretar como equipo acoplado.

El objetivo es inequívoco:

```text
cerrar tapa
     │
     ▼
no suspender
     │
     ▼
servicios continúan activos
```

---

## 15. Administración remota segura

La administración del sistema se realiza desde la red local mediante SSH.

El servicio Nextcloud puede ser accesible desde Internet mediante Cloudflare Tunnel, pero esto no implica que SSH deba publicarse igualmente.

Se mantiene la separación:

```text
Internet
   │
   ▼
Cloudflare Tunnel
   │
   ▼
Nextcloud


LAN
   │
   ▼
SSH
   │
   ▼
administración Debian
```

No se necesita una redirección pública del puerto SSH en el router para el funcionamiento normal del proyecto.

Esta separación reduce los servicios administrativos expuestos directamente a Internet.

---

## 16. Comprobaciones después de un reinicio

Los reinicios constituyen una prueba importante para un servidor administrado de forma desatendida.

Después de cambios relevantes se comprueba como mínimo:

```bash
ip addr
```

```bash
ip route
```

```bash
systemctl --failed
```

```bash
systemctl status docker
```

```bash
docker ps
```

y, cuando corresponda:

```bash
systemctl list-timers --all
```

Además se comprueba funcionalmente:

```text
SSH
LAN
Nextcloud
Cloudflare Tunnel
Docker
timers
montajes
```

Una configuración se considera correctamente integrada cuando continúa funcionando después de un reinicio completo, no únicamente durante la sesión en la que fue aplicada.

---

## 17. Principios aplicados

La configuración de Debian y red sigue varios principios.

### Administración con privilegios mínimos

Se utiliza un usuario normal y `sudo` en lugar de trabajar permanentemente como `root`.

### Dirección del servidor predecible

Los servicios locales pueden localizar `hpserver` de forma estable.

### Evitar dependencias domésticas innecesarias

El servidor Nextcloud no se convierte en un requisito para el funcionamiento básico de Internet del resto de la vivienda.

### No exponer administración innecesariamente

SSH permanece como mecanismo de administración local y no necesita publicarse directamente en Internet.

### Validar después de reiniciar

La persistencia forma parte de la validación.

### Diagnosticar por capas

En problemas de conectividad se diferencia entre:

```text
hardware
   ↓
interfaz
   ↓
IP / routing
   ↓
DNS del host
   ↓
Docker
   ↓
DNS del contenedor
   ↓
aplicación
```

Esto evita atribuir inmediatamente un problema a la aplicación cuando puede encontrarse en una capa inferior.

---

## 18. Incidencias y lecciones principales

Durante esta fase se produjeron varias incidencias útiles para futuras instalaciones.

| Incidencia | Causa / diagnóstico | Solución |
| --- | --- | --- |
| `sudo` no disponible | Instalación inicial mínima | Instalar `sudo` y añadir el usuario administrativo |
| Nextcloud no inicia tras reboot | Docker intenta enlazar una IP todavía no asignada | Publicar `8080:80` sin enlazar una IP concreta |
| SMTP no resuelve hostname | DNS del host correcto pero resolver Docker sin DNS externo | Configurar DNS explícito para Docker |
| Cerrar tapa podía suspender el servidor | Comportamiento de `systemd-logind` | Configurar `HandleLidSwitch*=ignore` |
| Riesgo al cambiar SSH | Posible pérdida de acceso administrativo | Validar usuario + `sudo` antes de restringir `root` |

Estas incidencias se describen también de forma resumida en el registro general de incidencias del proyecto.

---

## 19. Resumen

La configuración base de `hpserver` puede representarse como:

```text
                  Debian 13
                      │
           ┌──────────┴──────────┐
           │                     │
     Administración             Red
           │                     │
      usuario + sudo       Ethernet estática
           │                     │
          SSH              SERVER_LAN_IP
           │                     │
           └──────────┬──────────┘
                      │
                    Docker
                      │
                servicios hpserver
```

La configuración busca que el sistema sea sencillo de administrar, predecible después de cada reinicio y suficientemente independiente del resto de la infraestructura doméstica.

Las incidencias encontradas durante la configuración también dieron lugar a dos principios importantes para el resto del proyecto:

> **Un servicio no se considera correctamente configurado hasta haber comprobado su funcionamiento después de un reinicio.**

y:

> **La conectividad del host y la conectividad de los contenedores deben diagnosticarse como capas diferentes.**

---

## 20. Referencias

La configuración descrita se ha contrastado principalmente con documentación oficial o mantenida por los proyectos correspondientes.

### Debian

- **Debian Reference**
  - Administración del sistema.
  - Inicialización mediante systemd.
  - Usuarios y `sudo`.
  - Configuración de red.
  - OpenSSH.

- **Debian Reference — Network setup**
  - Infraestructura de red.
  - Configuración de interfaces.
  - Configuración estática.
  - Herramientas de diagnóstico.

- **Debian Reference — Network applications**
  - OpenSSH.
  - Archivos de configuración de cliente y servidor.

### dhcpcd

- **dhcpcd.conf(5)**
  - Configuración de interfaces.
  - `static ip_address`.
  - `static routers`.
  - `static domain_name_servers`.

### systemd

- **logind.conf(5)**
  - `HandleLidSwitch`.
  - `HandleLidSwitchExternalPower`.
  - `HandleLidSwitchDocked`.
  - Gestión de eventos de alimentación y tapa.

### Docker

- **Docker Engine documentation**
  - Configuración del daemon.
  - DNS de contenedores.
  - Publicación de puertos.
  - Inicio de servicios.

> Las referencias describen el funcionamiento de las tecnologías utilizadas. Las decisiones concretas, incidencias y procedimientos de validación documentados corresponden a la implementación de `hpserver`.