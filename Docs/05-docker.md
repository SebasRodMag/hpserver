# 05 - Docker

## 1. Objetivo

`hpserver` utiliza **Docker Engine** y **Docker Compose** para ejecutar los principales servicios de la plataforma Nextcloud.

La utilización de contenedores permite separar el sistema operativo base de las aplicaciones y sus dependencias.

La arquitectura general es:

```text
Debian 13
   │
   ▼
Docker Engine
   │
   ▼
Docker Compose
   │
   ├── Nextcloud
   ├── PostgreSQL
   ├── Redis
   ├── Nextcloud Cron
   └── cloudflared
```

Debian continúa siendo responsable de:

- almacenamiento;
- red;
- Docker;
- systemd;
- scripts de backup;
- SMART;
- alertas;
- administración del servidor.

Docker se utiliza principalmente como capa de ejecución de los servicios de aplicación.

---

## 2. Por qué utilizar Docker

Nextcloud y sus dependencias podrían instalarse directamente sobre Debian.

Sin embargo, se eligió Docker para mantener una separación más clara entre:

```text
HOST
│
├── Debian
├── discos
├── red
├── systemd
└── administración

CONTENEDORES
│
├── Nextcloud
├── PostgreSQL
├── Redis
├── Cron
└── cloudflared
```

Esta separación aporta varias ventajas.

### Aislamiento de dependencias

Cada servicio utiliza las bibliotecas y componentes incluidos en su propia imagen.

Esto reduce la necesidad de instalar directamente en Debian:

```text
Apache
PHP
PostgreSQL
Redis
```

para ejecutar Nextcloud.

### Reproducibilidad

La definición de los servicios se conserva en:

```text
compose.yml
```

Esto permite reconstruir la arquitectura de contenedores de forma relativamente predecible.

### Actualización controlada

Las imágenes pueden actualizarse independientemente del sistema operativo.

### Recuperación

En una reconstrucción del servidor es posible reinstalar Docker, recuperar la configuración Compose y volver a crear los contenedores.

Los contenedores en sí mismos no se consideran información que deba conservarse.

Lo importante es preservar:

```text
configuración
+
datos persistentes
+
base de datos
+
secretos
+
backups
```

---

## 3. Docker Engine frente a Docker Desktop

`hpserver` es un servidor Debian sin entorno gráfico.

Por ello se utiliza:

```text
Docker Engine
+
Docker Compose plugin
```

y no Docker Desktop.

La administración se realiza mediante comandos como:

```bash
docker ps
docker compose ps
docker compose up -d
docker compose logs
```

Docker Engine se ejecuta como servicio del sistema y puede comprobarse mediante:

```bash
systemctl status docker
```

---

## 4. Instalación desde el repositorio oficial

Docker se instaló utilizando el repositorio APT oficial de Docker.

No se utilizó el paquete `docker.io` proporcionado directamente por Debian.

La documentación oficial de Docker recomienda eliminar paquetes que puedan entrar en conflicto antes de instalar Docker Engine desde su propio repositorio.

La instalación utiliza los paquetes:

```text
docker-ce
docker-ce-cli
containerd.io
docker-buildx-plugin
docker-compose-plugin
```

El flujo general es:

```text
Debian
   │
   ▼
repositorio oficial Docker
   │
   ▼
Docker Engine
   │
   ├── CLI
   ├── containerd
   ├── Buildx
   └── Compose plugin
```

---

## 5. Configuración del repositorio

La instalación requiere incorporar la clave utilizada para verificar los paquetes del repositorio Docker.

Un procedimiento equivalente al utilizado es:

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
```

Posteriormente:

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
```

y:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

El repositorio se configura siguiendo las instrucciones oficiales correspondientes a Debian.

Después:

```bash
sudo apt update
```

y se instalan los paquetes:

```bash
sudo apt install \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

---

## 6. Incidencia: continuación de línea en `curl`

Durante la incorporación de la clave GPG se introdujo accidentalmente un espacio después de la barra invertida:

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \ -o /etc/apt/keyrings/docker.asc
```

La barra invertida se utiliza en el shell para continuar un comando en la siguiente línea únicamente cuando se encuentra inmediatamente antes del salto de línea.

El comando incorrecto provocó que la salida de la clave apareciera en el terminal en lugar de guardarse correctamente en el archivo esperado.

La forma correcta es:

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
```

o, en una única línea:

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
```

### Lección

En comandos multilínea del shell:

```text
\ + salto de línea → continuación

\ + espacio        → comportamiento diferente
```

Cuando se ejecutan comandos que modifican repositorios, claves o configuración del sistema, debe comprobarse el resultado antes de continuar con el siguiente paso.

---

## 7. Validación de Docker

Después de la instalación se comprueba el servicio:

```bash
sudo systemctl status docker
```

La instalación oficial permite además validar el motor mediante:

```bash
sudo docker run hello-world
```

Este comando:

```text
Docker CLI
    │
    ▼
Docker daemon
    │
    ▼
descarga imagen
    │
    ▼
crea contenedor
    │
    ▼
ejecuta hello-world
```

y permite verificar que el motor puede descargar y ejecutar imágenes.

Compose puede comprobarse mediante:

```bash
docker compose version
```

---

## 8. Usuario y grupo `docker`

El usuario administrativo se añadió al grupo:

```text
docker
```

Esto permite ejecutar:

```bash
docker ps
```

sin anteponer `sudo` continuamente.

Sin embargo, pertenecer al grupo `docker` no debe interpretarse como un privilegio menor.

Un usuario con acceso al daemon Docker puede crear contenedores con acceso privilegiado, montar directorios del host y realizar operaciones equivalentes en la práctica a obtener privilegios elevados sobre el sistema.

Por ello:

> **La pertenencia al grupo `docker` debe considerarse prácticamente equivalente a disponer de acceso administrativo al servidor.**

No se debe añadir al grupo a usuarios que no sean administradores del sistema.

---

## 9. Docker Compose

Docker Compose permite describir varios servicios relacionados en un único archivo declarativo.

En `hpserver` se utiliza:

```text
compose.yml
```

La estructura general es:

```yaml
services:
  db:
    ...

  redis:
    ...

  app:
    ...

  cron:
    ...

  cloudflared:
    ...
```

La pila puede administrarse mediante:

```bash
docker compose up -d
```

```bash
docker compose ps
```

```bash
docker compose logs
```

```bash
docker compose stop
```

```bash
docker compose down
```

> [!WARNING]
> `docker compose down` y especialmente sus variantes que eliminan volúmenes deben utilizarse con conocimiento de qué información persistente depende de Docker. Los contenedores son reemplazables; los datos persistentes no deben tratarse de la misma manera.

---

## 10. Estructura del proyecto Nextcloud

La configuración de producción se encuentra bajo una estructura equivalente a:

```text
/srv/docker/nextcloud/
├── compose.yml
├── .env
└── volumes/
    ├── nextcloud/
    └── postgres/
```

Las responsabilidades son:

```text
compose.yml
    │
    └── definición de servicios

.env
    │
    └── variables y secretos

volumes/nextcloud
    │
    └── persistencia Nextcloud

volumes/postgres
    │
    └── base de datos activa
```

Los archivos de usuario no se encuentran dentro de esta estructura.

Se almacenan independientemente en:

```text
/srv/storage/nextcloud-data
```

Esta separación evita mezclar:

```text
aplicación
base de datos
datos de usuario
backup
```

---

## 11. El archivo `.env`

Compose utiliza un archivo:

```text
.env
```

para proporcionar determinados valores de configuración.

Este archivo contiene información sensible y **no debe incluirse en el repositorio Git**.

Puede contener valores equivalentes a:

```text
POSTGRES_DB=...
POSTGRES_USER=...
POSTGRES_PASSWORD=...
NEXTCLOUD_ADMIN_USER=...
NEXTCLOUD_ADMIN_PASSWORD=...
CLOUDFLARE_TUNNEL_TOKEN=...
```

En el repositorio público sólo debe existir:

```text
.env.example
```

con valores ficticios:

```dotenv
POSTGRES_DB=nextcloud
POSTGRES_USER=<POSTGRES_USER>
POSTGRES_PASSWORD=<POSTGRES_PASSWORD>

NEXTCLOUD_ADMIN_USER=<NEXTCLOUD_ADMIN_USER>
NEXTCLOUD_ADMIN_PASSWORD=<NEXTCLOUD_ADMIN_PASSWORD>

CLOUDFLARE_TUNNEL_TOKEN=<CLOUDFLARE_TUNNEL_TOKEN>
```

El archivo real debe tener permisos restrictivos.

Por ejemplo:

```bash
chmod 600 .env
```

### Principio

> **Git conserva la estructura necesaria para reconstruir el servicio, pero no los secretos necesarios para acceder a él.**

Los secretos forman parte del procedimiento privado de recuperación.

---

## 12. Servicios de la pila

La arquitectura utiliza cinco servicios.

```text
Docker Compose
│
├── app
│   └── Nextcloud Apache
│
├── db
│   └── PostgreSQL
│
├── redis
│   └── Redis
│
├── cron
│   └── tareas internas Nextcloud
│
└── cloudflared
    └── Cloudflare Tunnel
```

Cada servicio tiene una responsabilidad específica.

---

## 13. Nextcloud `app`

El servicio principal ejecuta la imagen Apache de Nextcloud.

La versión se fija explícitamente a una versión concreta de la rama utilizada por el servidor, en lugar de depender de una etiqueta completamente flotante.

Una representación equivalente es:

```yaml
app:
  image: nextcloud:<NEXTCLOUD_VERSION>-apache
  restart: unless-stopped
```

Nextcloud necesita acceso a:

```text
PostgreSQL
Redis
volumen Nextcloud
DATA
```

Los archivos de usuario se incorporan mediante el montaje correspondiente hacia el directorio de datos utilizado por Nextcloud.

---

## 14. PostgreSQL `db`

PostgreSQL almacena la base de datos de Nextcloud.

La imagen se encuentra fijada a una versión mayor concreta:

```yaml
db:
  image: postgres:18-alpine
```

Esto es consecuencia directa de una incidencia ocurrida durante el despliegue inicial.

La base de datos activa se almacena de forma persistente fuera del ciclo de vida del contenedor.

---

## 15. Incidencia: `postgres:alpine` y PostgreSQL 18

Inicialmente se utilizó:

```yaml
image: postgres:alpine
```

junto con un montaje pensado para versiones anteriores:

```yaml
volumes:
  - ./volumes/postgres:/var/lib/postgresql/data
```

La etiqueta `postgres:alpine` no fija una versión mayor concreta de PostgreSQL.

Durante el despliegue se obtuvo PostgreSQL 18.

La imagen oficial de PostgreSQL 18 introdujo un cambio relevante en la organización del almacenamiento del contenedor, utilizando como punto de montaje recomendado:

```text
/var/lib/postgresql
```

en lugar de mantener la estrategia anterior basada directamente en:

```text
/var/lib/postgresql/data
```

El resultado fue un ciclo de reinicios del contenedor de base de datos.

Como consecuencia, Nextcloud tampoco podía establecer una conexión estable con PostgreSQL.

### Diagnóstico

El problema no estaba en:

```text
usuario PostgreSQL
contraseña
red Docker
Nextcloud
```

sino en la combinación:

```text
etiqueta flotante
       +
nueva versión mayor
       +
layout de almacenamiento cambiado
```

### Solución

Se fijó explícitamente:

```yaml
image: postgres:18-alpine
```

y el almacenamiento se adaptó a:

```yaml
volumes:
  - ./volumes/postgres:/var/lib/postgresql
```

Como la inicialización fallida todavía pertenecía a una instalación nueva sin datos de producción, se pudo limpiar el estado incompleto y volver a inicializar PostgreSQL correctamente.

### Lección

> **Una etiqueta de imagen aparentemente estable puede incorporar una nueva versión mayor con cambios incompatibles.**

Esto resulta especialmente importante para servicios con estado:

```text
PostgreSQL
MySQL/MariaDB
bases de datos
almacenamientos
```

En estos componentes, actualizar una imagen no equivale simplemente a reemplazar un binario.

Los datos persistentes deben seguir siendo compatibles con la nueva versión.

---

## 16. Política de versiones

Después de la incidencia de PostgreSQL se adoptó como criterio general fijar las versiones relevantes de los servicios.

La pila utiliza conceptualmente:

```text
Nextcloud   → versión concreta
PostgreSQL  → major fijada
Redis       → versión fijada
```

El objetivo es evitar que:

```bash
docker compose pull
```

introduzca silenciosamente una nueva versión mayor de un componente crítico.

### Etiquetas flotantes

Etiquetas como:

```text
latest
alpine
apache
```

pueden cambiar con el tiempo.

Esto no significa que nunca puedan utilizarse, pero deben emplearse entendiendo qué nivel de estabilidad proporcionan.

Para componentes con datos persistentes se prefiere una política conservadora.

### Actualización

Una actualización debe tratarse como una operación administrada:

```text
consultar release notes
        │
        ▼
comprobar compatibilidad
        │
        ▼
backup válido
        │
        ▼
actualizar imagen
        │
        ▼
recrear contenedor
        │
        ▼
validar servicio
```

---

## 17. Redis

Redis se ejecuta como servicio independiente.

Su función principal para Nextcloud es proporcionar:

- caché;
- locking distribuido.

Una definición equivalente es:

```yaml
redis:
  image: redis:<REDIS_VERSION>-alpine
  restart: unless-stopped
```

Su funcionamiento puede comprobarse desde el contenedor mediante:

```bash
redis-cli ping
```

con respuesta:

```text
PONG
```

Nextcloud utiliza el hostname interno:

```text
redis
```

para localizarlo dentro de la red Compose.

No es necesario publicar Redis hacia la red local ni hacia Internet.

---

## 18. Nextcloud Cron

El servicio `cron` utiliza la misma imagen y persistencia de Nextcloud que `app`.

Conceptualmente:

```yaml
cron:
  image: nextcloud:<NEXTCLOUD_VERSION>-apache
  restart: unless-stopped
  entrypoint: /cron.sh
```

La imagen oficial de Nextcloud proporciona `/cron.sh` para ejecutar las tareas periódicas de la aplicación.

Es importante que `app` y `cron` compartan la misma persistencia de Nextcloud.

La propia configuración de ejemplo oficial de Nextcloud advierte que los volúmenes de ambos servicios deben coincidir.

El contenedor `cron` no sustituye a los timers systemd del servidor.

```text
cron container
     │
     └── tareas internas Nextcloud

systemd timers
     │
     ├── Borg
     ├── SMART
     └── integridad
```

---

## 19. Cloudflared

`cloudflared` se ejecuta dentro de la misma pila Compose.

Una definición equivalente es:

```yaml
cloudflared:
  image: cloudflare/cloudflared:<VERSION>
  restart: unless-stopped
  command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
```

El token se obtiene desde `.env` y nunca debe almacenarse directamente en `compose.yml` ni publicarse en Git.

Al encontrarse en la misma red Docker, `cloudflared` puede alcanzar Nextcloud utilizando el nombre del servicio:

```text
app:80
```

sin necesidad de salir hacia la interfaz LAN.

El funcionamiento de Cloudflare Tunnel se documenta en detalle en su capítulo específico.

---

## 20. Red interna de Docker

Docker Compose crea una red para los servicios del proyecto.

Conceptualmente:

```text
             Docker network
                   │
      ┌────────────┼────────────┐
      │            │            │
     app           db          redis
      │
      ├── cron
      │
      └── cloudflared
```

Los servicios pueden localizarse mediante sus nombres Compose:

```text
db
redis
app
```

Por ejemplo:

```text
Nextcloud → db:<POSTGRES_PORT>
Nextcloud → redis:<REDIS_PORT>
cloudflared → app:80
```

No es necesario conocer la IP interna concreta asignada a cada contenedor.

### Direcciones internas

Docker puede asignar direcciones como:

```text
172.x.x.x
```

pero, igual que ocurre con `/dev/sdX` para los discos, las direcciones dinámicas concretas no deben convertirse innecesariamente en parte de la configuración.

Se utilizan los nombres de servicio cuando es posible.

---

## 21. Publicación de puertos

El único servicio que necesita acceso HTTP directo desde la LAN es Nextcloud.

Inicialmente se utilizó una asociación equivalente a:

```yaml
ports:
  - "<SERVER_LAN_IP>:8080:80"
```

Esto provocó la condición de carrera descrita en `03-debian-y-red.md`.

La configuración se cambió a:

```yaml
ports:
  - "8080:80"
```

De esta forma Docker puede iniciar aunque la dirección LAN estática todavía no haya sido asignada.

### Implicación

La publicación:

```text
8080:80
```

hace que Docker escuche en las interfaces disponibles del host, no únicamente en la IP LAN concreta.

Esto debe tenerse en cuenta desde el punto de vista de seguridad.

En `hpserver`:

- no existe port forwarding del router para `8080`;
- el acceso remoto utiliza Cloudflare Tunnel;
- la publicación se utiliza principalmente para acceso LAN.

> [!NOTE]
> Docker modifica reglas de red para implementar la publicación de puertos. La documentación oficial advierte que esto debe tenerse en cuenta cuando se utilizan herramientas de firewall como `ufw` o `firewalld`.

---

## 22. Dependencias y healthchecks

Nextcloud depende de PostgreSQL y Redis.

Una definición simple:

```yaml
depends_on:
  - db
  - redis
```

puede establecer orden de creación, pero que un contenedor haya arrancado no significa necesariamente que su aplicación interna esté preparada para aceptar conexiones.

La diferencia es:

```text
container running
       ≠
service ready
```

Por ello se utilizan healthchecks para los servicios que lo permiten.

### PostgreSQL

Una comprobación típica utiliza:

```text
pg_isready
```

### Redis

Una comprobación típica utiliza:

```text
redis-cli ping
```

Compose puede utilizar:

```yaml
depends_on:
  db:
    condition: service_healthy
  redis:
    condition: service_healthy
```

para esperar a que las dependencias estén realmente preparadas.

Esto reduce condiciones de carrera durante el inicio de la pila.

---

## 23. Política de reinicio

Los servicios utilizan una política de reinicio adecuada para un servidor que debe funcionar sin intervención habitual.

Por ejemplo:

```yaml
restart: unless-stopped
```

Esto permite que los contenedores vuelvan a iniciarse después de:

- reinicios del servidor;
- reinicios de Docker;
- determinados fallos del proceso.

Al mismo tiempo, un servicio detenido deliberadamente puede permanecer detenido.

### Validación

Después de configurar la pila se realizaron reinicios completos del servidor y se comprobó:

```bash
docker compose ps
```

El resultado esperado es:

```text
app          running
db           running / healthy
redis        running / healthy
cron         running
cloudflared  running
```

La recuperación automática después de un reboot forma parte de los criterios de aceptación de la instalación.

---

## 24. DNS de Docker

Los contenedores utilizan el resolver interno de Docker, normalmente visible como:

```text
nameserver 127.0.0.11
```

Este resolver permite tanto resolver nombres de servicios Compose como reenviar consultas externas.

Durante la configuración apareció una incidencia en la que Docker se inició sin disponer correctamente de resolvers externos.

El host Debian podía resolver nombres, pero Nextcloud no.

Como medida defensiva se configuró:

```text
/etc/docker/daemon.json
```

con una estructura pública equivalente a:

```json
{
  "dns": [
    "<ROUTER_DNS>",
    "<FALLBACK_DNS>"
  ]
}
```

Después de modificar la configuración del daemon se reinicia Docker de forma controlada y se valida la conectividad de los contenedores.

Esta incidencia se describe con más detalle en `03-debian-y-red.md`.

---

## 25. Persistencia

Una distinción fundamental es:

```text
contenedor
    │
    └── reemplazable

datos persistentes
    │
    └── imprescindibles
```

Eliminar y recrear:

```text
nextcloud-app-1
```

no debería implicar perder los datos de Nextcloud si los montajes persistentes están correctamente configurados.

Por el contrario, eliminar accidentalmente:

```text
volumes/nextcloud
volumes/postgres
/srv/storage/nextcloud-data
```

puede provocar pérdida de información.

Por este motivo, la documentación de recuperación se centra en los datos persistentes y no en conservar contenedores concretos.

---

## 26. PostgreSQL y backup

El directorio:

```text
/srv/docker/nextcloud/volumes/postgres
```

contiene la base de datos activa.

Sin embargo, este directorio **no se copia directamente como mecanismo principal de backup mientras PostgreSQL está funcionando**.

Antes del backup se genera:

```text
PostgreSQL activo
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

El dump se valida posteriormente mediante `pg_restore`.

Esto separa:

```text
persistencia operacional
        │
        └── volumes/postgres

formato de recuperación
        │
        └── nextcloud.dump
```

La estrategia completa se documenta en `09-backup-borg.md`.

---

## 27. Configuración que debe conservarse

Para reconstruir la pila no es necesario conservar los contenedores existentes.

Sí debe conservarse o poder reconstruirse:

```text
compose.yml
.env / secretos
volumen Nextcloud
dump PostgreSQL
DATA
configuración Docker
```

El repositorio Git contiene versiones saneadas de:

```text
docker/nextcloud/compose.yml
docker/nextcloud/.env.example
config/docker/daemon.json
```

Los secretos reales se conservan fuera de Git.

---

## 28. Comandos habituales

La administración cotidiana utiliza principalmente:

```bash
docker compose ps
```

para consultar servicios.

```bash
docker compose logs
```

para revisar logs.

```bash
docker compose logs app
```

para un servicio concreto.

```bash
docker compose pull
```

para descargar imágenes definidas en Compose.

```bash
docker compose up -d
```

para crear o actualizar la pila.

```bash
docker compose config
```

para validar y mostrar la configuración Compose resuelta.

Antes de aplicar cambios importantes es especialmente útil:

```bash
docker compose config
```

porque permite detectar problemas sintácticos y revisar la configuración resultante.

---

## 29. Diagnóstico

Cuando un servicio falla, se intenta diagnosticar por capas:

```text
Debian
   │
   ▼
Docker daemon
   │
   ▼
contenedor
   │
   ▼
red Docker
   │
   ▼
dependencias
   │
   ▼
aplicación
```

Por ejemplo, ante un Nextcloud inaccesible:

```bash
systemctl status docker
```

```bash
docker compose ps
```

```bash
docker compose logs app
```

```bash
docker compose logs db
```

```bash
docker compose logs redis
```

permiten determinar progresivamente dónde se encuentra el fallo.

No debe asumirse que un error mostrado por Nextcloud tiene necesariamente su origen en Nextcloud.

---

## 30. Incidencias y lecciones principales

| Incidencia | Causa | Solución / lección |
| --- | --- | --- |
| Clave GPG mostrada en terminal | Espacio después de `\` en comando multilínea | Corregir sintaxis y validar archivo |
| PostgreSQL reiniciándose | `postgres:alpine` obtuvo PostgreSQL 18 con cambio de layout | Fijar versión y adaptar volumen |
| Nextcloud no inicia tras reboot | Puerto enlazado a IP todavía no disponible | Publicar `8080:80` |
| DNS funciona en host pero no contenedor | Docker inició sin resolver externo adecuado | Configurar DNS del daemon |
| `depends_on` no garantiza readiness por sí solo | Contenedor iniciado ≠ servicio preparado | Healthchecks + `service_healthy` |
| Riesgo de actualizaciones inesperadas | Etiquetas flotantes | Fijar versiones críticas |
| `.env` contiene secretos | Configuración sensible necesaria para Compose | `.env` fuera de Git + `.env.example` |

---

## 31. Principios aplicados

### Los contenedores son reemplazables

La recuperación no depende de conservar un contenedor concreto.

### La persistencia no es reemplazable

Configuración, base de datos y datos deben protegerse independientemente.

### Las versiones de servicios con estado deben controlarse

Una actualización mayor de PostgreSQL requiere más atención que reemplazar un contenedor sin estado.

### `running` no significa `ready`

Los healthchecks permiten comprobar disponibilidad real de dependencias.

### Los secretos no pertenecen al repositorio

Git contiene plantillas reproducibles, no credenciales.

### Los nombres de servicio son preferibles a IP internas

Compose proporciona resolución interna entre servicios.

### Las actualizaciones deben ser operaciones controladas

```text
backup
→ revisar compatibilidad
→ actualizar
→ validar
```

no simplemente:

```text
latest → pull → esperar que funcione
```

---

## 32. Resumen

Docker proporciona la capa de ejecución de los servicios de `hpserver`:

```text
                       Debian
                          │
                          ▼
                    Docker Engine
                          │
                          ▼
                    Docker Compose
                          │
        ┌─────────┬───────┼───────┬───────────┐
        │         │       │       │           │
        ▼         ▼       ▼       ▼           ▼
    Nextcloud PostgreSQL Redis   Cron    cloudflared
        │         │
        │         ▼
        │    persistencia DB
        │
        ▼
      DATA
```

La utilización de Docker no elimina la necesidad de comprender los servicios que se ejecutan dentro de los contenedores.

Las incidencias encontradas durante el despliegue demostraron especialmente la importancia de:

```text
versiones controladas
       +
persistencia identificada
       +
healthchecks
       +
validación tras reboot
       +
backups independientes
```

Docker facilita reconstruir la capa de aplicación, mientras que la estrategia de almacenamiento y backup permite recuperar la información que los contenedores por sí solos no pueden proporcionar.

---

## 33. Referencias

### Docker Engine

- **Docker Engine — Install on Debian**  
  https://docs.docker.com/engine/install/debian/

  - Repositorio oficial.
  - Clave de firma.
  - Paquetes Docker CE.
  - Validación mediante `hello-world`.

- **Docker Engine — Linux post-installation steps**  
  https://docs.docker.com/engine/install/linux-postinstall/

  - Grupo `docker`.
  - Inicio automático.
  - Consideraciones de seguridad.
  - Privilegios equivalentes a `root` asociados al grupo `docker`.

### Docker Compose

- **Docker Compose documentation**  
  https://docs.docker.com/compose/

  - Definición de aplicaciones multi-contenedor.
  - Servicios.
  - Redes.
  - Volúmenes.
  - Variables de entorno.

- **Control startup and shutdown order in Compose**  
  https://docs.docker.com/compose/how-tos/startup-order/

  - `depends_on`.
  - `healthcheck`.
  - `service_healthy`.
  - Diferencia entre contenedor iniciado y servicio preparado.

### PostgreSQL

- **PostgreSQL — Docker Official Image**  
  https://hub.docker.com/_/postgres

  - Variables de inicialización.
  - Persistencia.
  - Configuración de `PGDATA`.
  - Cambios introducidos en PostgreSQL 18.

  PostgreSQL 18 cambió el valor de `PGDATA` a una ruta dependiente de la versión y modificó el volumen declarado por la imagen oficial a:

  ```text
  /var/lib/postgresql

> Las referencias describen el comportamiento oficial de los componentes. La arquitectura Compose, política de versiones, incidencias y procedimientos de recuperación corresponden a la implementación real de `hpserver`.