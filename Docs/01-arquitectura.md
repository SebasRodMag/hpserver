# 01 - Arquitectura

> [!NOTE]
> Por motivos de seguridad y portabilidad, las direcciones, dominios,
> identificadores y credenciales específicos de la instalación han sido
> sustituidos por valores de ejemplo.

## 1. Objetivo

`hpserver` es un servidor doméstico destinado principalmente al almacenamiento, sincronización y respaldo de archivos personales y familiares mediante Nextcloud.

La arquitectura se ha diseñado buscando:

- simplicidad de administración;
- reutilización de hardware;
- separación entre sistema, datos y copias de seguridad;
- acceso tanto local como remoto;
- seguridad;
- automatización;
- capacidad de recuperación ante fallos;
- utilización de software libre.

El sistema debe poder funcionar de forma desatendida durante largos periodos, en caso de avería de algunos de sus componentes debe brindar información suficiente para saber que esta sucediendo y poder recuperar el funcionamiento.

Una de las decisiones fundamentales del proyecto es considerar que **almacenamiento, sincronización y backup son funciones diferentes**. Nextcloud proporciona almacenamiento y sincronización, mientras que BorgBackup mantiene una copia independiente que permite recuperar los datos si el almacenamiento principal resulta dañado, eliminado o comprometido.

---

## 2. Visión general

La arquitectura puede dividirse en cinco grandes bloques:

1. **Sistema base:** Debian 13 instalado directamente sobre el hardware.
2. **Servicios:** Nextcloud y sus dependencias ejecutados mediante Docker Compose.
3. **Almacenamiento:** separación física entre sistema, datos y backup.
4. **Conectividad:** acceso local directo y acceso remoto mediante Cloudflare Tunnel.
5. **Protección:** BorgBackup, SMART, verificaciones periódicas y alertas automáticas.

De forma simplificada:

```text
                              INTERNET
                                  │
                                  ▼
                           Cloudflare
                                  ▲
                                  │
                         Cloudflare Tunnel
                                  │
                           ┌──────┴──────┐
                           │ cloudflared │
                           └──────┬──────┘
                                  │
                         Red Docker interna
                                  │
                ┌─────────────────┼─────────────────┐
                │                 │                 │
           Nextcloud          PostgreSQL          Redis
                │
                │
                ▼
        /srv/storage
        HDD DATA interno
                │
                │ BorgBackup
                ▼
         /srv/backup
       HDD BACKUP externo
```

El sistema operativo, la configuración de los contenedores y los datos de PostgreSQL permanecen en el SSD del servidor.

Los archivos de los usuarios de Nextcloud se almacenan en un HDD independiente.

Las copias Borg se almacenan en un tercer dispositivo físico.

---

## 3. Arquitectura física

El servidor reutiliza un ordenador portátil como plataforma de hardware.

Esta elección permite aprovechar un equipo existente que dispone de potencia suficiente para la carga prevista, evitando adquirir inicialmente un NAS o servidor dedicado.

El almacenamiento se distribuye entre tres dispositivos.

| Dispositivo | Función principal | Punto de montaje |
| --- | --- | --- |
| SSD 240 GB | Debian, Docker, configuración y base de datos | `/` |
| HDD 500 GB interno | Datos de usuarios de Nextcloud | `/srv/storage` |
| HDD 500 GB externo | Repositorio BorgBackup | `/srv/backup` |

Esta separación es deliberada. Permite una mejor y más rápida recuperación en caso de avería

### 3.1 Disco de sistema

El **SSD** contiene:

- Debian;
- Docker Engine;
- Docker Compose;
- configuración de Nextcloud;
- aplicaciones de Nextcloud;
- datos persistentes de PostgreSQL;
- scripts de administración;
- unidades y timers de systemd;
- configuración del sistema.

El sistema puede, por tanto, beneficiarse de la menor latencia del SSD para las operaciones frecuentes de la aplicación y de la base de datos.

Los archivos personales de los usuarios no se almacenan principalmente en este dispositivo.

### 3.2 Disco DATA

El **HDD** interno está dedicado a los datos de los usuarios de Nextcloud.

Su punto de montaje es:

```text
/srv/storage
```

y el directorio utilizado por Nextcloud:

```text
/srv/storage/nextcloud-data
```

Separar DATA del disco de sistema facilita la administración del almacenamiento y permite sustituir o reconstruir el sistema operativo sin que el disco que contiene los archivos de los usuarios tenga que ser formateado.

Esta separación **no constituye un backup**.

La eliminación accidental de un archivo, corrupción, errores de aplicación o determinados fallos administrativos pueden afectar a los datos independientemente de que se encuentren en otro disco.

### 3.3 Disco BACKUP

Un **HDD** externo independiente contiene el repositorio Borg:

```text
/srv/backup/borg
```

El dispositivo se monta en:

```text
/srv/backup
```

BorgBackup mantiene copias cifradas y deduplicadas de los componentes necesarios para recuperar Nextcloud. ``Se elimina información repetida de las copias para ahorrar espacio``

La utilización de un dispositivo físico diferente permite conservar las copias aunque falle el disco DATA.

Sin embargo, al permanecer conectado normalmente al mismo servidor, este disco no debe considerarse una copia *off-site*. Un incidente que afecte físicamente al servidor y a los dispositivos conectados podría afectar simultáneamente a DATA y BACKUP.

---

## 4. Arquitectura de servicios

Los servicios de aplicación se ejecutan mediante Docker Compose.

La configuración principal se encuentra en:

```text
/srv/docker/nextcloud/
```

La arquitectura está formada por cinco servicios principales:

```text
Nextcloud
PostgreSQL
Redis
Nextcloud Cron
Cloudflared
```

### 4.1 Nextcloud

Nextcloud constituye la capa principal de aplicación.

Proporciona:

- almacenamiento y gestión de archivos;
- sincronización entre dispositivos;
- interfaz web;
- usuarios y permisos;
- acceso mediante clientes móviles y de escritorio;
- funcionalidades adicionales mediante aplicaciones.

Nextcloud no almacena toda la información en un único lugar.

Los archivos de usuario se encuentran en DATA, mientras que configuración, aplicaciones y otros componentes persistentes se conservan en el volumen correspondiente del SSD.

### 4.2 PostgreSQL

PostgreSQL actúa como base de datos de Nextcloud.

Almacena información estructurada necesaria para el funcionamiento de la aplicación, incluyendo metadatos, configuración, usuarios, permisos y relaciones internas.

La base de datos se encuentra físicamente en el SSD.

Para los backups no se copia directamente el directorio de PostgreSQL mientras la base de datos está en ejecución. Antes de crear cada backup se genera un dump consistente mediante `pg_dump`.

Esto permite disponer de una representación recuperable de la base de datos sin depender de una copia en caliente de sus archivos internos.

### 4.3 Redis

Redis proporciona almacenamiento temporal en memoria utilizado por Nextcloud principalmente para caché y mecanismos de bloqueo.

Una función especialmente importante es el *file locking*, que ayuda a coordinar accesos concurrentes y evita conflictos cuando diferentes procesos trabajan simultáneamente con determinados recursos.

Redis no contiene los archivos personales de los usuarios y no sustituye ni a PostgreSQL ni al almacenamiento DATA.

Su contenido es fundamentalmente operativo y temporal.

### 4.4 Nextcloud Cron

Nextcloud necesita ejecutar periódicamente tareas internas en segundo plano.

Para ello se utiliza un contenedor específico que ejecuta el mecanismo cron proporcionado por Nextcloud.

Entre estas tareas pueden encontrarse operaciones de mantenimiento, limpieza y procesamiento de trabajos pendientes.

Este componente no debe confundirse con los timers de systemd utilizados por `hpserver` para backups y monitorización.

```text
Nextcloud Cron
    └── tareas internas de Nextcloud

systemd timers
    ├── BorgBackup
    ├── comprobación Borg
    ├── SMART
    └── verificaciones de integridad
```

### 4.5 Cloudflared

`cloudflared` establece el Cloudflare Tunnel utilizado para el acceso remoto.

El túnel se inicia desde el propio servidor hacia la infraestructura de Cloudflare.

Esto evita necesitar una conexión entrante directa desde Internet hacia el servidor y permite publicar Nextcloud sin configurar *port forwarding* en el router.

El servicio público utilizado por Nextcloud es:

```text
https://cloud.example.com
```

---

## 5. Arquitectura de red

`hpserver` dispone de una dirección IPv4 estática dentro de la red local:

```text
IP_LAN_SERVIDOR
```

La dirección se encuentra fuera del rango DHCP utilizado por el router para reducir el riesgo de conflictos.

Nextcloud puede utilizarse mediante dos rutas diferentes.

### 5.1 Acceso local

Desde la red doméstica se puede acceder directamente mediante:

```text
http://IP_LAN_SERVIDOR:8080
```

Este camino no depende de Cloudflare ni de la conexión a Internet.

El puerto Docker se publica como:

```text
8080:80
```

en lugar de enlazarse específicamente a `IP_LAN_SERVIDOR`.

Esta decisión evita una condición de carrera durante el arranque: Docker puede iniciarse antes de que el sistema haya terminado de asignar la dirección estática a la interfaz de red.

### 5.2 Acceso remoto

Desde Internet se utiliza:

```text
https://cloud.example.com
```

El flujo simplificado es:

```text
Cliente
   │
   │ HTTPS
   ▼
Cloudflare
   │
   │ Tunnel
   ▼
cloudflared
   │
   │ Red Docker
   ▼
Nextcloud
```

No existen reglas de redirección de puertos en el router destinadas a publicar directamente Nextcloud.

### 5.3 Proxy inverso y HTTPS

Cloudflare termina la conexión HTTPS pública.

Nextcloud está configurado para reconocer el tráfico procedente del entorno Docker utilizado por el proxy/túnel y generar correctamente las URL externas HTTPS.

Al mismo tiempo, se conserva el acceso HTTP local.

Para ello, la sobrescritura del protocolo HTTPS se aplica de forma condicionada al tráfico procedente del proxy y no indiscriminadamente a todas las conexiones.

Esto evita que una petición local como:

```text
http://IP_LAN_SERVIDOR:8080
```

sea redirigida incorrectamente a:

```text
https://IP_LAN_SERVIDOR:8080
```

---

## 6. Arquitectura de almacenamiento

La separación del almacenamiento sigue el principio:

```text
SYSTEM
   │
   ├── Debian
   ├── Docker
   ├── Nextcloud
   └── PostgreSQL

DATA
   │
   └── archivos de usuarios

BACKUP
   │
   └── Borg repository
```

Cada función puede administrarse de forma relativamente independiente.

Esta organización facilita especialmente los procedimientos de sustitución de hardware.

Por ejemplo, la avería del SSD no implica necesariamente la pérdida del disco DATA, mientras que la avería del HDD DATA puede recuperarse utilizando el repositorio BACKUP.

La documentación de recuperación define procedimientos específicos para cada escenario.

---

## 7. Arquitectura de backup

BorgBackup es la herramienta principal de copia de seguridad.

Cada backup incluye los elementos necesarios para reconstruir el servicio:

```text
/srv/storage/nextcloud-data
/srv/docker/nextcloud/volumes/nextcloud
/srv/docker/nextcloud/compose.yml
/srv/docker/nextcloud/.env
/var/backups/nextcloud/nextcloud.dump
```

El directorio vivo de PostgreSQL no forma parte directamente del backup.

En su lugar:

```text
PostgreSQL
    │
    ▼
 pg_dump
    │
    ▼
nextcloud.dump
    │
    ▼
BorgBackup
```

Antes de realizar la copia, el procedimiento automatizado:

1. comprueba que el disco BACKUP correcto está realmente montado;
2. comprueba el espacio libre disponible;
3. activa temporalmente el modo mantenimiento de Nextcloud cuando corresponde;
4. detiene temporalmente Nextcloud Cron cuando corresponde;
5. genera el dump de PostgreSQL;
6. valida el dump;
7. crea el archivo Borg;
8. aplica la política de retención;
9. compacta el repositorio;
10. restaura el estado anterior de Cron y del modo mantenimiento.

La comprobación del dispositivo mediante UUID es una medida especialmente importante.

Si `/srv/backup` existiera como un directorio normal pero el HDD externo no estuviera montado, un proceso de backup sin esta protección podría escribir accidentalmente los datos en el SSD del sistema.

---

## 8. Integridad, monitorización y alertas

Crear backups no es suficiente para considerar protegidos los datos.

La arquitectura incorpora diferentes niveles de comprobación:

```text
Backup diario
      │
      ├── Borg check semanal
      │
      ├── Borg --verify-data trimestral
      │
      └── restauraciones de prueba
```

Los discos DATA y BACKUP también son supervisados mediante SMART.

Las comprobaciones monitorizan, entre otros indicadores:

- sectores reasignados;
- sectores pendientes;
- sectores no corregibles;
- errores reportados;
- errores CRC;
- estado general SMART;
- resultado de self-tests.

Las tareas se ejecutan mediante servicios y timers de systemd.

Los servicios críticos utilizan:

```ini
OnFailure=hpserver-alert@%n.service
```

Si uno de ellos termina con error, systemd ejecuta el servicio general de alertas.

```text
Fallo
  │
  ▼
systemd
  │
  ▼
hpserver-alert
  │
  ▼
msmtp
  │
  ▼
Brevo
  │
  ▼
Correo electrónico
```

El envío de alertas se realiza directamente desde Debian y no a través de Nextcloud.

De esta manera, una avería de Nextcloud no impide necesariamente que el servidor informe del problema.

---

## 9. Automatización

Las principales tareas de mantenimiento se programan mediante timers de systemd.

| Tarea | Periodicidad |
| --- | --- |
| Backup Nextcloud | Diaria, 03:30 |
| Comprobación Borg | Semanal, domingo 04:30 |
| SMART | Mensual, primer sábado 05:00 |
| Verificación completa Borg | Trimestral, primer domingo de enero, abril, julio y octubre a las 06:00 |

Los timers utilizan `Persistent=true`.

Esto permite que systemd tenga en cuenta determinadas ejecuciones perdidas debido a que el servidor estuviera apagado en el momento programado.

El portátil también está configurado para ignorar el cierre de la tapa, evitando que entre en suspensión durante su funcionamiento como servidor.

---

## 10. Principios de diseño

La arquitectura se apoya en varios principios.

### Separar sistema, datos y backup

Los tres elementos cumplen funciones distintas y, siempre que es posible, utilizan dispositivos físicos diferentes.

### Evitar considerar sincronización como backup

Nextcloud mantiene y sincroniza datos, pero una modificación o eliminación puede propagarse.

Borg proporciona una segunda capa independiente con historial de versiones.

### Verificar los backups

Un backup no se considera fiable únicamente porque el proceso haya terminado sin errores.

Se realizan comprobaciones de integridad y restauraciones de prueba.

### Minimizar exposición a Internet

Nextcloud no se publica mediante una redirección directa de puertos del router.

El acceso remoto utiliza un túnel iniciado desde el propio servidor.

### Automatizar sin perder visibilidad

Las tareas periódicas están automatizadas, pero sus fallos generan alertas.

La automatización no debe convertir un error en un problema silencioso.

### Mantener capacidad de recuperación

La configuración, scripts y procedimientos se documentan y versionan independientemente del servidor.

Los secretos y claves necesarios para una recuperación se conservan mediante mecanismos separados del repositorio Git.

### No destruir la última copia válida

Durante una migración o sustitución de hardware:

> **El dispositivo antiguo no debe borrarse, formatearse ni reutilizarse hasta que el nuevo sistema haya sido validado y se haya realizado una restauración real satisfactoria.**

---

## 11. Dependencias y puntos de fallo

La arquitectura reduce algunos riesgos, pero no elimina todos los puntos de fallo.

La disponibilidad local depende principalmente de:

- hardware del servidor;
- SSD del sistema;
- HDD DATA;
- Docker;
- Nextcloud;
- PostgreSQL;
- red local.

El acceso remoto añade además dependencia de:

- conexión a Internet;
- Cloudflare;
- Cloudflare Tunnel;
- DNS público.

Las copias de seguridad dependen de:

- HDD BACKUP;
- BorgBackup;
- disponibilidad de la passphrase;
- disponibilidad de la clave de recuperación;
- integridad del repositorio.

Las alertas por correo dependen de:

- conectividad de red;
- `msmtp`;
- Brevo;
- servicio de correo del destinatario.

Por este motivo, la arquitectura no presupone que un único mecanismo sea suficiente para garantizar disponibilidad o recuperación.

---

## 12. Alcance y limitaciones

`hpserver` está diseñado como servidor doméstico y no pretende proporcionar alta disponibilidad.

Actualmente no existe:

- redundancia RAID;
- segundo servidor en caliente;
- replicación de PostgreSQL;
- alimentación mediante un UPS dedicado;
- copia completa *off-site* automatizada.

La batería del portátil puede proporcionar una protección limitada ante cortes breves de alimentación, pero no debe considerarse equivalente a una solución UPS administrada.

El disco BACKUP protege frente a múltiples escenarios de pérdida lógica o avería del almacenamiento DATA, pero al encontrarse normalmente junto al servidor no protege completamente frente a robo, incendio u otros incidentes físicos que afecten a toda la ubicación.

Estas limitaciones son conocidas y forman parte del equilibrio entre coste, complejidad y necesidades reales del proyecto.

---

## 13. Resultado de la arquitectura

La arquitectura resultante puede resumirse como:

```text
                     ┌───────────────────┐
                     │     Internet      │
                     └─────────┬─────────┘
                               │
                        Cloudflare Tunnel
                               │
              ┌────────────────▼────────────────┐
              │            hpserver             │
              │          Debian 13              │
              │                                 │
              │   ┌─────────────────────────┐   │
              │   │      Docker Compose     │   │
              │   │                         │   │
              │   │ Nextcloud   PostgreSQL  │   │
              │   │ Redis       Cron        │   │
              │   │ cloudflared             │   │
              │   └────────────┬────────────┘   │
              │                │                │
              │        ┌───────▼───────┐        │
              │        │   HDD DATA    │        │
              │        │ /srv/storage  │        │
              │        └───────┬───────┘        │
              │                │ Borg           │
              │        ┌───────▼───────┐        │
              │        │  HDD BACKUP   │        │
              │        │ /srv/backup   │        │
              │        └───────────────┘        │
              │                                 │
              │ systemd ──► monitorización      │
              │              │                  │
              │              ▼                  │
              │        alertas por correo       │
              └─────────────────────────────────┘
```

Esta estructura proporciona una base suficientemente sencilla para un entorno doméstico, pero incorpora mecanismos de separación, backup, verificación, monitorización y recuperación que permiten administrar el servidor de forma controlada y documentada.

Los siguientes capítulos describen individualmente cada uno de estos componentes y los procedimientos utilizados para configurarlos.

## 14. Referencias

La arquitectura y configuración descritas en este documento se han
contrastado principalmente con la documentación oficial de los
componentes utilizados.

### Nextcloud

- [Reverse proxy - Nextcloud Administration Manual](URL)
  - `trusted_proxies`
  - `overwriteprotocol`
  - `overwritecondaddr`
  - `overwrite.cli.url`

- [Backup - Nextcloud Administration Manual](URL)
  - Elementos necesarios para una copia completa.
  - Uso del modo mantenimiento.

### Cloudflare

- [Cloudflare Tunnel - Cloudflare Docs](URL)
  - Funcionamiento mediante conexiones salientes.
  - Publicación de servicios sin abrir puertos entrantes.

### BorgBackup

- [BorgBackup Documentation](URL)
  - Creación de repositorios.
  - Verificación de integridad.
  - Retención y compactación.

### systemd

- [systemd.timer](URL)
- [systemd.service](URL)
  - Programación de tareas.
  - Gestión de fallos mediante unidades.

### smartmontools

- [smartctl documentation](URL)
  - Estado SMART y self-tests.