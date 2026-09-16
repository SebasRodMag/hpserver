# 06 - Nextcloud

## 1. Objetivo

Nextcloud constituye la aplicación principal de `hpserver`.

Su función es proporcionar a los usuarios una plataforma privada para:

- almacenar archivos;
- sincronizar dispositivos;
- acceder a los datos desde la red local;
- acceder remotamente mediante HTTPS;
- recuperar archivos mediante las funciones propias de Nextcloud;
- administrar usuarios y cuotas;
- ampliar funcionalidades mediante aplicaciones.

Nextcloud no se ejecuta directamente sobre Debian.

Forma parte de la pila Docker descrita en el capítulo anterior:

```text
Debian
   │
   ▼
Docker
   │
   ├── Nextcloud
   ├── PostgreSQL
   ├── Redis
   ├── Cron
   └── cloudflared
```

En el momento documentado, la instalación utiliza la rama **Nextcloud 34** mediante la imagen Apache oficial.

> [!NOTE]
> Dominios, direcciones IP, nombres de usuario, credenciales e identificadores específicos de la instalación se sustituyen por valores de ejemplo.

---

## 2. Arquitectura de Nextcloud

La arquitectura lógica es:

```text
                   Clientes
                      │
             ┌────────┴────────┐
             │                 │
            LAN             Internet
             │                 │
             │          Cloudflare Tunnel
             │                 │
             └────────┬────────┘
                      ▼
                  Nextcloud
                      │
       ┌──────────────┼──────────────┐
       │              │              │
       ▼              ▼              ▼
 PostgreSQL         Redis          DATA
       │              │              │
   metadatos       cache /      archivos
   aplicación       locking      usuarios
```

Nextcloud actúa como capa de aplicación.

Los archivos no se almacenan dentro del ciclo de vida del contenedor.

---

## 3. Imagen utilizada

El servicio principal utiliza la variante Apache de la imagen oficial.

Una definición pública equivalente es:

```yaml
app:
  image: nextcloud:<NEXTCLOUD_VERSION>-apache
  restart: unless-stopped
```

Se evita depender únicamente de:

```text
nextcloud:latest
```

porque una actualización automática de una versión mayor podría introducir cambios que requieran migraciones o modificaciones de configuración.

La actualización de Nextcloud se trata como una operación administrada:

```text
backup válido
     │
     ▼
revisar versión
     │
     ▼
actualizar imagen
     │
     ▼
ejecutar migraciones necesarias
     │
     ▼
validar
```

---

## 4. Persistencia

La instalación diferencia entre:

```text
aplicación Nextcloud
        │
        ▼
/srv/docker/nextcloud/volumes/nextcloud

archivos de usuarios
        │
        ▼
/srv/storage/nextcloud-data

base de datos activa
        │
        ▼
/srv/docker/nextcloud/volumes/postgres
```

Esta separación permite gestionar cada tipo de información según su función.

El directorio persistente de Nextcloud contiene elementos como:

```text
config/
custom_apps/
themes/
core/
3rdparty/
...
```

El directorio DATA permanece fuera de la estructura principal de Docker.

---

## 5. DATA externo

Nextcloud utiliza como directorio de datos:

```text
/srv/storage/nextcloud-data
```

que pertenece al HDD identificado mediante el rol:

```text
DATA
```

El contenedor recibe este almacenamiento mediante un bind mount.

Conceptualmente:

```yaml
volumes:
  - /srv/storage/nextcloud-data:/var/www/html/data
```

De esta forma:

```text
Nextcloud
    │
    ▼
/var/www/html/data
    │
    │ bind mount
    ▼
/srv/storage/nextcloud-data
    │
    ▼
HDD DATA
```

La eliminación o recreación del contenedor no elimina por sí misma los archivos almacenados en DATA.

---

## 6. Validación de DATA

Después de completar la instalación se verificó que Nextcloud utilizaba realmente el HDD DATA.

La comprobación no se limitó a revisar el archivo Compose.

Se cargó un archivo mediante Nextcloud y posteriormente se confirmó su existencia física dentro de:

```text
/srv/storage/nextcloud-data/<USER>/files/
```

También se comprobó mediante:

```bash
findmnt /srv/storage
```

y:

```bash
df -h /srv/storage
```

que el directorio correspondía al dispositivo DATA.

Esta validación evita un fallo potencialmente grave:

```text
configuración parece correcta
        │
        ▼
Nextcloud escribe realmente en SYSTEM
        │
        ▼
SSD se llena
```

---

## 7. PostgreSQL

Nextcloud utiliza PostgreSQL como base de datos.

La base de datos almacena información estructurada necesaria para la aplicación, como:

- usuarios;
- configuración;
- metadatos;
- referencias de archivos;
- actividad;
- aplicaciones;
- comparticiones;
- estados internos.

Los archivos de los usuarios no deben confundirse con la base de datos.

```text
archivo físico
     │
     └── DATA

metadatos y estado
     │
     └── PostgreSQL
```

Una instalación recuperable necesita ambos.

Por ello, copiar únicamente `nextcloud-data` no constituye un backup completo de Nextcloud.

---

## 8. Redis

Redis se utiliza para mejorar el funcionamiento de Nextcloud y gestionar especialmente el bloqueo transaccional de archivos.

La configuración conceptual incluye:

```php
'memcache.locking' => '\OC\Memcache\Redis',
'memcache.distributed' => '\OC\Memcache\Redis',
```

y una conexión equivalente a:

```php
'redis' => [
    'host' => 'redis',
    'port' => 6379,
],
```

Dentro de Docker, `redis` es el nombre del servicio Compose.

No es necesario utilizar una IP interna fija.

### Validación

Redis puede comprobarse mediante:

```bash
docker compose exec redis redis-cli ping
```

con resultado:

```text
PONG
```

La configuración de Nextcloud puede inspeccionarse mediante `occ`.

Por ejemplo:

```bash
docker compose exec -u www-data app \
  php occ config:system:get redis host
```

El resultado esperado es:

```text
redis
```

---

## 9. `occ`

Nextcloud proporciona la herramienta administrativa:

```text
occ
```

En la instalación Docker se ejecuta dentro del contenedor `app` como el usuario del servidor web.

El patrón utilizado es:

```bash
docker compose exec -u www-data app php occ <comando>
```

Por ejemplo:

```bash
docker compose exec -u www-data app php occ status
```

`occ` permite administrar numerosos aspectos sin modificar manualmente `config.php`.

Entre otros:

```text
configuración
usuarios
aplicaciones
mantenimiento
background jobs
migraciones
estado
```

Siempre que exista un comando `occ` apropiado se prefiere éste frente a modificar manualmente archivos internos durante el funcionamiento normal.

---

## 10. Estado de la instalación

El estado general puede comprobarse mediante:

```bash
docker compose exec -u www-data app php occ status
```

Esto permite verificar aspectos como:

```text
installed
version
maintenance
needsDbUpgrade
```

También resulta útil después de:

- actualizaciones;
- restauraciones;
- reinicios;
- cambios importantes de configuración.

---

## 11. Background jobs

Nextcloud necesita ejecutar periódicamente tareas internas.

Estas tareas incluyen operaciones de mantenimiento que no deben depender de que un usuario abra la interfaz web.

Nextcloud admite diferentes mecanismos:

```text
AJAX
Webcron
Cron
```

Para una instalación de servidor se utiliza:

```text
Cron
```

El modo se configuró mediante:

```bash
docker compose exec -u www-data app php occ background:cron
```

---

## 12. Contenedor Cron

La ejecución periódica se realiza mediante un segundo contenedor basado en la misma imagen de Nextcloud.

Conceptualmente:

```yaml
cron:
  image: nextcloud:<NEXTCLOUD_VERSION>-apache
  restart: unless-stopped
  entrypoint: /cron.sh
```

`app` y `cron` deben compartir la persistencia necesaria de Nextcloud.

La arquitectura es:

```text
app
 │
 ├──── persistencia Nextcloud
 │
cron
```

El contenedor Cron se ocupa únicamente de los trabajos internos de Nextcloud.

No debe confundirse con los timers systemd utilizados para:

```text
Borg
SMART
integridad
alertas
```

---

## 13. Ventana de mantenimiento

Nextcloud permite definir una hora UTC a partir de la cual ejecutar determinados trabajos diarios no sensibles al tiempo.

La configuración se realiza mediante:

```bash
docker compose exec -u www-data app \
  php occ config:system:set maintenance_window_start \
  --type=integer \
  --value=<UTC_HOUR>
```

En `hpserver` se selecciona una franja de baja utilización.

La opción no significa que Nextcloud entre diariamente en modo mantenimiento.

Su función es permitir que determinados background jobs no urgentes se concentren en una ventana de cuatro horas a partir de la hora configurada.

```text
maintenance_window_start
          │
          ▼
inicio ventana UTC
          │
          ▼
4 horas para determinados
trabajos no urgentes
```

---

## 14. Modo mantenimiento

Nextcloud proporciona un modo específico para operaciones que requieren evitar actividad de usuarios.

Puede activarse mediante:

```bash
docker compose exec -u www-data app \
  php occ maintenance:mode --on
```

y desactivarse:

```bash
docker compose exec -u www-data app \
  php occ maintenance:mode --off
```

Cuando está activo, se evita la actividad normal de los usuarios mientras se realizan operaciones sensibles.

Se utiliza, entre otros casos, durante el procedimiento de backup de `hpserver`.

### Importancia para los scripts

El script de backup no debe asumir que Nextcloud estaba inicialmente fuera de mantenimiento.

El procedimiento conserva el estado original:

```text
¿maintenance ya estaba activo?
          │
       ┌──┴──┐
       │     │
      Sí     No
       │     │
       │     ▼
       │   activarlo
       │
       ▼
     backup
       │
       ▼
desactivar sólo si
el script lo activó
```

Esto evita que un backup desactive accidentalmente un mantenimiento iniciado previamente por un administrador.

---

## 15. `trusted_domains`

Nextcloud limita los hostnames desde los que acepta acceso mediante:

```text
trusted_domains
```

Esta protección evita aceptar arbitrariamente cualquier valor de la cabecera `Host`.

La configuración puede inspeccionarse mediante:

```bash
docker compose exec -u www-data app \
  php occ config:system:get trusted_domains
```

Una instalación equivalente puede incluir:

```text
localhost
<SERVER_LAN_IP>
cloud.example.com
```

La configuración conceptual es:

```php
'trusted_domains' => [
    'localhost',
    '<SERVER_LAN_IP>',
    'cloud.example.com',
],
```

### Función

```text
petición HTTP
     │
     ▼
Host
     │
     ▼
¿trusted_domains?
   ┌─┴─┐
   │   │
  Sí   No
   │   │
   ▼   ▼
aceptar rechazar
```

El dominio público utilizado por la instalación debe estar incluido.

También se permite el acceso local mediante la dirección correspondiente.

---

## 16. Reverse proxy

El acceso remoto llega a Nextcloud a través de un proxy.

La arquitectura es:

```text
Cliente HTTPS
      │
      ▼
Cloudflare
      │
      ▼
cloudflared
      │
      ▼
app:80
```

Nextcloud recibe internamente una conexión HTTP aunque el cliente haya utilizado HTTPS.

Esto requiere que la aplicación conozca qué proxies puede considerar de confianza y bajo qué condiciones debe generar URLs HTTPS.

---

## 17. `trusted_proxies`

Nextcloud permite definir los proxies que pueden proporcionar información sobre el cliente original.

Una configuración pública equivalente es:

```php
'trusted_proxies' => [
    '<DOCKER_NETWORK_CIDR>',
],
```

En `hpserver`, el proxy se encuentra dentro de la red Docker.

La red real no necesita publicarse en la documentación.

### Seguridad

No debe configurarse:

```text
trusted_proxies = cualquier dirección
```

sin necesidad.

Nextcloud utiliza esta información para interpretar cabeceras reenviadas por el proxy.

Una configuración excesivamente permisiva puede permitir falsificar información sobre el cliente.

---

## 18. `overwriteprotocol`

El proxy termina HTTPS antes de enviar la petición a Nextcloud mediante HTTP.

Sin configuración adicional:

```text
Cliente
  │ HTTPS
  ▼
Proxy
  │ HTTP
  ▼
Nextcloud
```

Nextcloud podría interpretar que el protocolo original es HTTP y generar URLs incorrectas.

Se configuró:

```text
overwriteprotocol=https
```

Conceptualmente:

```php
'overwriteprotocol' => 'https',
```

Esto indica a Nextcloud que genere URLs utilizando HTTPS cuando se cumplen las condiciones configuradas.

---

## 19. Incidencia: HTTPS remoto frente a HTTP local

Inicialmente `overwriteprotocol=https` se aplicaba globalmente.

Esto solucionaba el acceso mediante el proxy, pero introducía un problema en el acceso local.

El usuario accedía mediante:

```text
http://<SERVER_LAN_IP>:8080
```

pero Nextcloud generaba una redirección equivalente a:

```text
https://<SERVER_LAN_IP>:8080
```

El puerto local sirve HTTP, no HTTPS.

El navegador terminaba mostrando un error de protocolo SSL.

### Diagnóstico

La configuración tenía dos caminos válidos:

```text
REMOTO
cliente
  │ HTTPS
  ▼
proxy
  │ HTTP
  ▼
Nextcloud


LOCAL
cliente
  │ HTTP
  ▼
Nextcloud
```

Por tanto, no era correcto forzar HTTPS para todas las peticiones.

---

## 20. `overwritecondaddr`

Nextcloud proporciona:

```text
overwritecondaddr
```

precisamente para aplicar los parámetros `overwrite*` sólo cuando la petición procede de determinadas direcciones.

Se configuró una expresión regular que identifica las peticiones procedentes de la red Docker utilizada por el proxy.

Una versión pública equivalente sería:

```php
'overwriteprotocol' => 'https',
'overwritecondaddr' => '^<PROXY_NETWORK_PATTERN>',
```

El resultado es:

```text
petición desde proxy
       │
       ▼
overwriteprotocol=https


petición LAN directa
       │
       ▼
detección automática
       │
       ▼
HTTP
```

Esto permite mantener simultáneamente:

```text
https://cloud.example.com
```

y:

```text
http://<SERVER_LAN_IP>:8080
```

---

## 21. `overwrite.cli.url`

También se configuró una URL pública para operaciones ejecutadas desde CLI:

```php
'overwrite.cli.url' => 'https://cloud.example.com',
```

o mediante `occ`:

```bash
docker compose exec -u www-data app \
  php occ config:system:set overwrite.cli.url \
  --value="https://cloud.example.com"
```

Esta URL se utiliza cuando Nextcloud necesita generar enlaces desde procesos que no disponen de una petición HTTP de usuario desde la que inferir el hostname.

---

## 22. Configuración resultante del proxy

La configuración conceptual queda:

```text
trusted_domains
    ├── localhost
    ├── LAN
    └── dominio público

trusted_proxies
    └── red del proxy

overwriteprotocol
    └── https

overwritecondaddr
    └── sólo proxy

overwrite.cli.url
    └── https://cloud.example.com
```

La combinación permite:

```text
LAN HTTP
   +
Internet HTTPS
```

sin forzar incorrectamente un protocolo sobre el otro.

---

## 23. Validación de la configuración

Los valores pueden consultarse mediante:

```bash
docker compose exec -u www-data app \
  php occ config:system:get trusted_domains
```

```bash
docker compose exec -u www-data app \
  php occ config:system:get trusted_proxies
```

```bash
docker compose exec -u www-data app \
  php occ config:system:get overwriteprotocol
```

```bash
docker compose exec -u www-data app \
  php occ config:system:get overwritecondaddr
```

```bash
docker compose exec -u www-data app \
  php occ config:system:get overwrite.cli.url
```

Después se realizan pruebas funcionales independientes:

```text
LAN
 │
 └── acceso HTTP correcto

Internet
 │
 └── acceso HTTPS correcto
```

La configuración no se considera validada si sólo funciona uno de los dos caminos.

---

## 24. Avisos de administración

Nextcloud muestra comprobaciones de seguridad y configuración en:

```text
Administración
    │
    ▼
Vista general
```

Durante la puesta en marcha aparecieron distintos avisos.

Estos avisos no se ignoraron automáticamente.

Para cada uno se determinó:

```text
qué significa
      │
      ▼
si aplica al servidor
      │
      ▼
acción necesaria
```

Entre los avisos tratados se encontraron:

- ventana de mantenimiento;
- migraciones de tipos MIME;
- HSTS;
- configuración de región telefónica;
- configuración de correo;
- AppAPI / Deploy Daemon;
- disponibilidad de 2FA.

Algunos requieren una modificación.

Otros corresponden a funcionalidades que no forman parte de la arquitectura actual.

---

## 25. Migraciones de tipos MIME

Nextcloud puede indicar que existen migraciones de tipos MIME disponibles.

Estas operaciones pueden ser costosas y por ello no siempre se ejecutan automáticamente durante una actualización.

El comando utilizado es:

```bash
docker compose exec -u www-data app \
  php occ maintenance:repair --include-expensive
```

Antes de ejecutar operaciones de mantenimiento costosas en una instalación con datos importantes debe existir un backup válido.

---

## 26. Región telefónica

Para evitar ambigüedades al interpretar números sin prefijo internacional se puede definir una región predeterminada.

En una instalación equivalente:

```bash
docker compose exec -u www-data app \
  php occ config:system:set default_phone_region \
  --value="<ISO_COUNTRY_CODE>"
```

El valor real depende del país de la instalación y no constituye una característica arquitectónica del proyecto.

---

## 27. AppAPI

Nextcloud puede mostrar avisos relacionados con AppAPI y la ausencia de un Deploy Daemon.

AppAPI se utiliza para aplicaciones externas o `ExApps`.

`hpserver` no necesita actualmente desplegar ExApps como parte de su funcionamiento principal.

Por ello no se añadió infraestructura adicional únicamente para eliminar el aviso.

### Principio

> **Un aviso de administración debe entenderse antes de corregirse; no toda funcionalidad disponible necesita desplegarse.**

Añadir servicios que el proyecto no utiliza aumentaría innecesariamente:

```text
complejidad
+
superficie de mantenimiento
+
superficie de ataque
```

---

## 28. Usuarios

Nextcloud proporciona cuentas independientes para los miembros que utilizan el servidor.

El administrador principal dispone de privilegios administrativos.

Los usuarios familiares normales no necesitan pertenecer al grupo de administradores.

Conceptualmente:

```text
Administrador
     │
     ├── configuración
     ├── usuarios
     └── mantenimiento

Usuarios
     │
     └── archivos personales
```

El objetivo es mantener espacios privados independientes antes de incorporar mecanismos adicionales de compartición.

---

## 29. Cuotas

Nextcloud permite limitar la capacidad máxima utilizada por cada usuario.

Las cuotas se administran desde la gestión de usuarios.

La estrategia de `hpserver` reserva una cuota mayor para la cuenta administrativa y cuotas individuales para los usuarios familiares.

Los valores concretos pueden modificarse a medida que cambien las necesidades y no son esenciales para reproducir la arquitectura.

### Motivo

Sin cuotas:

```text
un usuario
   │
   ▼
consume DATA
   │
   ▼
afecta a todos
```

Con cuotas:

```text
capacidad DATA
     │
     ├── usuario A
     ├── usuario B
     ├── usuario C
     └── administrador
```

Las cuotas no sustituyen a la monitorización global del espacio disponible.

---

## 30. Autenticación de dos factores

La cuenta administrativa utiliza autenticación de dos factores mediante TOTP.

El flujo es:

```text
contraseña
    │
    ▼
TOTP
    │
    ▼
acceso
```

Durante la activación también se generaron códigos de recuperación.

Estos códigos se almacenan fuera de Nextcloud.

### Validación

Después de activar 2FA se realizó una nueva autenticación en una sesión independiente para comprobar que:

- la contraseña era aceptada;
- el segundo factor funcionaba;
- el administrador podía acceder correctamente.

Los códigos de recuperación no se almacenan en:

```text
Git
repositorio público
Nextcloud como única copia
```

porque deben seguir disponibles precisamente si se pierde el acceso normal a Nextcloud.

---

## 31. 2FA para otros usuarios

La activación de TOTP para el administrador se considera validada.

No se fuerza inicialmente 2FA globalmente para todos los usuarios antes de completar su incorporación y comprobar sus dispositivos.

Esto evita crear una política de acceso que pueda bloquear innecesariamente a usuarios todavía no configurados.

La ampliación futura puede seguir:

```text
crear usuario
     │
     ▼
validar acceso
     │
     ▼
configurar cliente
     │
     ▼
activar 2FA
     │
     ▼
guardar recuperación
```

---

## 32. Correo electrónico

Nextcloud utiliza SMTP para enviar mensajes.

La instalación utiliza un proveedor SMTP externo.

Conceptualmente:

```text
Nextcloud
    │
    ▼
SMTP STARTTLS
    │
    ▼
proveedor
    │
    ▼
destinatario
```

La configuración incluye:

```text
modo SMTP
servidor SMTP
puerto 587
STARTTLS
autenticación
remitente del dominio
```

Las credenciales SMTP no forman parte de este repositorio.

La configuración completa del correo y la autenticación del dominio se documenta en:

```text
docs/08-seguridad-y-correo.md
```

---

## 33. Incidencia SMTP causada por DNS

Durante la configuración inicial, el envío de correo desde Nextcloud falló con un error de resolución del hostname SMTP.

En un primer momento podría parecer un problema de:

```text
credenciales
TLS
servidor SMTP
```

Sin embargo, el diagnóstico demostró que el host Debian resolvía correctamente mientras el contenedor no podía hacerlo.

La causa se encontraba en el resolver DNS de Docker después del arranque.

La incidencia se solucionó en la capa Docker.

### Lección

> **Un error producido por Nextcloud no implica necesariamente que su causa se encuentre en Nextcloud.**

La aplicación depende de:

```text
Nextcloud
    │
    ▼
Docker
    │
    ▼
Debian
    │
    ▼
red
```

y el diagnóstico debe respetar esas capas.

---

## 34. HSTS

Nextcloud detectó inicialmente que la respuesta HTTPS pública no contenía un encabezado HSTS con el tiempo mínimo recomendado.

En esta arquitectura HTTPS termina en la infraestructura situada delante de Nextcloud.

Por ello, HSTS se implementó en la capa externa y no directamente en Apache dentro del contenedor.

La respuesta pública se validó posteriormente comprobando la presencia de:

```text
Strict-Transport-Security
```

La configuración completa se documenta en:

```text
docs/07-cloudflare-tunnel.md
```

---

## 35. Logs

Nextcloud dispone de sus propios logs de aplicación.

Docker también proporciona logs de los contenedores.

Por tanto, el diagnóstico puede utilizar diferentes niveles:

```text
Nextcloud log
      │
      ▼
Docker app log
      │
      ▼
Docker daemon
      │
      ▼
systemd journal
```

Comandos útiles:

```bash
docker compose logs app
```

```bash
docker compose logs --tail=100 app
```

y:

```bash
journalctl -u docker
```

La fuente adecuada depende del tipo de problema investigado.

---

## 36. Actualizaciones

Las actualizaciones de Nextcloud no se tratan como una operación automática sin supervisión.

El procedimiento general es:

```text
comprobar versión actual
        │
        ▼
revisar versión destino
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
recrear contenedores
        │
        ▼
ejecutar actualización
        │
        ▼
revisar warnings
        │
        ▼
validar archivos
        │
        ▼
validar LAN/remoto
```

Después de una actualización también deben comprobarse:

```text
PostgreSQL
Redis
Cron
apps
logs
background jobs
```

---

## 37. Qué debe incluir el backup

La documentación oficial de Nextcloud identifica como elementos fundamentales para una copia:

```text
config
custom apps
data
theme
database
```

La estrategia de `hpserver` protege:

```text
/srv/docker/nextcloud/volumes/nextcloud
        │
        └── aplicación/config/custom apps/etc.

/srv/storage/nextcloud-data
        │
        └── DATA

PostgreSQL
        │
        ▼
pg_dump
        │
        ▼
nextcloud.dump

compose.yml
.env
```

La base de datos activa no se considera sustituible únicamente por los archivos de DATA.

---

## 38. Validación funcional

Después de cambios relevantes se comprueba Nextcloud desde el punto de vista del usuario, no únicamente desde Docker.

Las validaciones incluyen:

```text
login
  │
  ├── LAN
  └── remoto

archivos
  │
  ├── upload
  ├── download
  ├── edición
  └── eliminación

servicios
  │
  ├── PostgreSQL
  ├── Redis
  └── Cron

seguridad
  │
  ├── HTTPS remoto
  └── 2FA
```

Una pila Docker con todos los contenedores en estado `running` no demuestra por sí sola que Nextcloud funcione correctamente.

---

## 39. Aplicaciones

Nextcloud permite ampliar su funcionalidad mediante aplicaciones.

La instalación contiene aplicaciones oficiales y puede incorporar otras según las necesidades.

Las aplicaciones adicionales deben considerarse parte del ciclo de mantenimiento.

Antes de una actualización importante debe comprobarse su compatibilidad.

Las aplicaciones personalizadas o instaladas adicionalmente forman también parte de la estrategia de backup mediante la persistencia de Nextcloud.

---

## 40. Principios aplicados

### DATA está fuera del ciclo de vida del contenedor

Los archivos de usuario permanecen en almacenamiento dedicado.

### Base de datos y archivos forman conjuntamente el estado

No se considera suficiente conservar sólo uno de ellos.

### Redis mejora concurrencia y locking

No almacena los archivos de usuario.

### Cron no depende de actividad web

Los background jobs se ejecutan mediante el mecanismo recomendado para producción.

### LAN y remoto son caminos diferentes

Nextcloud debe poder distinguir HTTP local de HTTPS mediante proxy.

### Los avisos deben entenderse

No se instalan componentes únicamente para conseguir una pantalla de administración sin advertencias.

### Las configuraciones se validan funcionalmente

```text
valor correcto en config.php
          ≠
servicio necesariamente correcto
```

### La seguridad de acceso incluye recuperación

2FA debe acompañarse de códigos de recuperación almacenados fuera del propio servicio.

---

## 41. Incidencias y lecciones principales

| Incidencia | Causa / diagnóstico | Solución / lección |
| --- | --- | --- |
| HTTPS forzado también en LAN | `overwriteprotocol=https` global | Condicionarlo mediante `overwritecondaddr` |
| Acceso local generaba error SSL | HTTP local redirigido a HTTPS | Separar comportamiento proxy/LAN |
| SMTP fallaba desde Nextcloud | DNS del contenedor, no credenciales | Diagnosticar por capas |
| Aviso HSTS | Header ausente en respuesta pública | Configurarlo en la capa HTTPS externa |
| Migraciones MIME pendientes | Operaciones costosas no ejecutadas automáticamente | `maintenance:repair --include-expensive` |
| Aviso AppAPI | No existe Deploy Daemon | No desplegarlo mientras ExApps no sean necesarias |
| Ubicación de cuotas indicada incorrectamente | Diferencia de interfaz/versiones | Validar instrucciones contra Nextcloud 34 |
| Riesgo de perder acceso tras 2FA | Segundo factor único | Guardar recovery codes fuera de Nextcloud |

---

## 42. Resumen

Nextcloud constituye la capa de aplicación que une el resto de componentes:

```text
                    Usuarios
                       │
             ┌─────────┴─────────┐
             │                   │
            LAN                HTTPS
             │                   │
             │                 Proxy
             │                   │
             └─────────┬─────────┘
                       ▼
                   Nextcloud
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
      PostgreSQL     Redis         DATA
          │
          ▼
       metadatos
```

La instalación no se considera protegida únicamente porque Nextcloud sea accesible.

Su funcionamiento depende de una combinación coherente de:

```text
DATA
+
PostgreSQL
+
configuración
+
Redis
+
Cron
+
proxy
+
backup
+
autenticación
```

La arquitectura intenta mantener estos componentes separados y documentados para que puedan diagnosticarse, actualizarse y recuperarse independientemente.

---

## 43. Referencias

### Nextcloud Administration Manual

- **Nextcloud 34 Administration Manual**  
  https://docs.nextcloud.com/server/stable/admin_manual/

  - Administración general.
  - Configuración del servidor.
  - Mantenimiento.
  - Seguridad.
  - Background jobs.

### Configuración del servidor

- **Configuration Parameters**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/config_sample_php_parameters.html

  - `trusted_domains`.
  - `trusted_proxies`.
  - `overwriteprotocol`.
  - `overwrite.cli.url`.
  - Redis.
  - `maintenance_window_start`.

- **Apps, background jobs & config commands**  
  https://docs.nextcloud.com/server/stable/admin_manual/occ_apps.html

  - Gestión de configuración mediante `occ`.
  - Arrays de configuración.
  - Configuración de Redis.
  - `trusted_domains`.

### Reverse proxy

- **Reverse proxy — Nextcloud Administration Manual**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/reverse_proxy_configuration.html

  - `trusted_proxies`.
  - `overwriteprotocol`.
  - `overwritecondaddr`.
  - `overwrite.cli.url`.
  - Acceso mediante reverse proxy.
  - Acceso simultáneo directo y mediante proxy.

  `overwritecondaddr` resulta especialmente relevante para `hpserver`, ya que permite aplicar los parámetros `overwrite*` únicamente a las conexiones procedentes del proxy y mantener detección automática para el acceso HTTP local.

### Redis

- **Memory caching — Nextcloud Administration Manual**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/caching_configuration.html

  - Redis.
  - Distributed cache.
  - Transactional File Locking.
  - `memcache.locking`.
  - `memcache.distributed`.

### Background jobs

- **Background jobs — Nextcloud Administration Manual**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/background_jobs_configuration.html

  - AJAX.
  - Webcron.
  - Cron.
  - `maintenance_window_start`.

- **System & maintenance commands — `background:cron`**  
  https://docs.nextcloud.com/server/stable/admin_manual/occ_system.html

  - Configuración del modo Cron mediante `occ`.
  - Comandos de mantenimiento.

### Backup

- **Backup — Nextcloud Administration Manual**  
  https://docs.nextcloud.com/server/stable/admin_manual/maintenance/backup.html

  - Directorio de configuración.
  - Custom apps.
  - DATA.
  - Themes.
  - Base de datos.
  - Maintenance mode.

### Avisos de seguridad

- **Warnings on admin page**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/security_setup_warnings.html

  - Comprobaciones de seguridad.
  - HTTPS.
  - Reverse proxy.
  - Cookies seguras.

### AppAPI

- **AppAPI and External Apps**  
  https://docs.nextcloud.com/server/stable/admin_manual/exapps_management/AppAPIAndExternalApps.html

  - ExApps.
  - Deploy Daemons.
  - Arquitectura AppAPI.

### Imagen Docker

- **Nextcloud Docker — repositorio oficial**  
  https://github.com/nextcloud/docker

  - Imagen oficial.
  - Variables de entorno.
  - Persistencia.
  - Configuración mediante Docker.

> Las referencias describen el comportamiento oficial de Nextcloud y de su imagen Docker. Las decisiones arquitectónicas, valores concretos, procedimientos de validación e incidencias corresponden a la implementación de `hpserver`.