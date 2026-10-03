# 13. Migración de hardware

## 13.1 Objetivo y alcance

Este capítulo define el procedimiento para trasladar `hpserver` a nuevo
hardware o sustituir de forma planificada alguno de sus dispositivos de
almacenamiento.

A diferencia de una recuperación ante desastres, una migración parte de
una situación controlada en la que el servidor original continúa
disponible. Esto permite preparar, validar y probar el destino antes de
retirar el origen.

La regla principal es:

> **El servidor original debe conservarse intacto hasta que el nuevo
> entorno haya sido validado y exista un punto de recuperación nuevo y
> comprobado.**

Una migración planificada debe aprovechar esta ventaja para reducir al
mínimo las operaciones irreversibles.

El procedimiento cubre:

-   sustitución del servidor completo;
-   sustitución de SYSTEM;
-   sustitución o ampliación de DATA;
-   sustitución de BACKUP;
-   migración de Nextcloud;
-   reconstrucción de PostgreSQL;
-   reconciliación de UUID y configuración;
-   cambio controlado a producción;
-   rollback;
-   retirada segura del hardware anterior.

El capítulo 11 continúa siendo la referencia cuando la migración deja de
ser planificada y pasa a convertirse en una recuperación por pérdida o
avería.

------------------------------------------------------------------------

## 13.2 Cuándo realizar una migración

### 13.2.1 Sustitución del servidor

Puede ser necesaria por:

-   renovación del portátil actual;
-   mayor capacidad de CPU o RAM;
-   necesidad de más interfaces o almacenamiento;
-   mejora de eficiencia energética;
-   deterioro físico;
-   búsqueda de una plataforma más apropiada para funcionamiento
    permanente.

En este escenario es preferible preparar el nuevo equipo en paralelo.

### 13.2.2 Sustitución de SYSTEM

SYSTEM contiene Debian, Docker, configuración del host y los componentes
persistentes de aplicación que no pertenecen a DATA.

Una sustitución planificada de SYSTEM puede realizarse mediante una
instalación limpia y reconstrucción a partir de la documentación,
recovery bundle y backup.

No debe asumirse que clonar el disco es siempre la mejor opción. Una
reconstrucción limpia evita trasladar errores históricos y ha sido
validada conceptualmente mediante los procedimientos de Disaster
Recovery.

### 13.2.3 Sustitución de DATA

DATA contiene los archivos de usuarios de Nextcloud.

Puede sustituirse por:

-   avería preventiva;
-   aumento de capacidad;
-   migración de HDD a SSD;
-   cambio de interfaz o formato físico.

La prioridad es mantener una copia íntegra del DATA original hasta
validar el nuevo dispositivo.

### 13.2.4 Sustitución de BACKUP

BACKUP puede sustituirse sin migrar Nextcloud, pero debe preservarse el
historial Borg siempre que sea posible.

Si el repositorio se copia a un nuevo dispositivo, la copia debe
validarse antes de retirar el anterior.

### 13.2.5 Ampliación de almacenamiento

Una ampliación de DATA debe tratarse como una migración de
almacenamiento, no como una simple sustitución de ruta.

Debe comprobarse:

``` text
filesystem
UUID
mount
permisos
.ncdata
guard
capacidad
arranque
backup
```

------------------------------------------------------------------------

## 13.3 Principios de la migración

### 13.3.1 No destruir el origen

Mientras el nuevo servidor no esté validado:

``` text
NO formatear SYSTEM antiguo
NO borrar DATA antiguo
NO reutilizar BACKUP antiguo
NO eliminar secretos de recuperación
NO modificar innecesariamente producción
```

El servidor anterior constituye el rollback más sencillo.

### 13.3.2 Mantener una copia Borg independiente

Antes de una migración relevante debe existir un backup reciente y
validado.

Para una sustitución completa es recomendable que al menos una copia no
dependa físicamente del hardware que se está manipulando.

### 13.3.3 Identificar discos por función y UUID

No asumir:

``` text
/dev/sda = SYSTEM
/dev/sdb = DATA
/dev/sdc = BACKUP
```

Comprobar en cada host:

``` bash
lsblk -f
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
sudo blkid
```

Los nombres de dispositivo son dinámicos.

### 13.3.4 Evitar dos servidores en producción simultáneamente

Durante la preparación pueden existir dos instalaciones de Nextcloud,
pero no deben competir por:

-   la misma IP;
-   el mismo Cloudflare Tunnel;
-   el mismo DATA en escritura;
-   tareas automáticas que afecten al mismo repositorio;
-   la misma identidad de producción sin aislamiento.

El nuevo host debe utilizar inicialmente una IP diferente y mantener
`cloudflared` desactivado.

### 13.3.5 Validar antes del cutover

El cambio de producción sólo debe realizarse después de comprobar
localmente:

``` text
PostgreSQL
Redis
Nextcloud
login
lectura
escritura
descarga
DATA
systemd
reboot
```

------------------------------------------------------------------------

## 13.4 Inventario previo

Antes de migrar debe registrarse el estado real del servidor.

### 13.4.1 Hardware

``` bash
lscpu
free -h
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

No es necesario replicar exactamente el hardware anterior, pero el
destino debe disponer de recursos suficientes para la carga prevista.

### 13.4.2 Almacenamiento

``` bash
df -h
df -i
findmnt /
findmnt /srv/storage
findmnt /srv/backup
```

Registrar:

``` text
SYSTEM → filesystem y UUID
DATA   → filesystem, UUID y capacidad utilizada
BACKUP → filesystem, UUID y capacidad utilizada
```

### 13.4.3 Red

``` bash
ip addr
ip route
cat /etc/dhcpcd.conf
```

La IP de producción actual no debe asignarse al nuevo servidor mientras
el antiguo continúe conectado con esa misma dirección.

### 13.4.4 Docker y Nextcloud

``` bash
cd /srv/docker/nextcloud
sudo docker compose ps
sudo docker compose images
sudo docker compose exec -u www-data app php occ status
```

### 13.4.5 Versiones

Registrar:

``` bash
cat /etc/debian_version
uname -r
docker --version
docker compose version
borg --version
```

Y las imágenes:

``` bash
cd /srv/docker/nextcloud
sudo docker compose images
```

### 13.4.6 Configuración externa

Confirmar que siguen disponibles fuera del servidor:

``` text
passphrase Borg
clave de recuperación Borg exportada
credenciales administrativas
acceso al dominio
acceso a Cloudflare
credenciales SMTP
2FA y códigos de recuperación cuando corresponda
```

No incluir estos secretos en la documentación pública.

------------------------------------------------------------------------

## 13.5 Preparación de la migración

### 13.5.1 Comprobar salud del servidor

Antes de copiar nada:

``` bash
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh

cd /srv/docker/nextcloud
sudo docker compose ps
sudo docker compose exec -u www-data app php occ status
```

No es recomendable utilizar como origen un sistema que ya presenta
errores no comprendidos.

### 13.5.2 Ejecutar backup

``` bash
sudo systemctl start nextcloud-backup.service
```

Después:

``` bash
systemctl status nextcloud-backup.service --no-pager
journalctl -u nextcloud-backup.service -n 100 --no-pager
```

El servicio `oneshot` puede aparecer como `inactive (dead)` tras
finalizar. Debe comprobarse el resultado real de la ejecución.

### 13.5.3 Validar Borg

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

Confirmar el archive recién creado.

Cuando proceda:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg check --show-rc /srv/backup/borg
```

### 13.5.4 Comprobar recovery bundle

Revisar:

``` text
/var/backups/hpserver-config/
```

El bundle debe reflejar el estado actual del host.

Especial atención a:

``` text
fstab
Docker
systemd
scripts
red
alertas
manifiesto de paquetes
```

### 13.5.5 Registrar UUID y configuración

Guardar temporalmente un inventario de referencia:

``` bash
lsblk -f
sudo blkid
findmnt
```

Los UUID del servidor original sirven para identificar el origen, pero
no deben copiarse a ciegas al nuevo host.

### 13.5.6 Preparar rollback

Antes de comenzar debe responderse:

``` text
¿qué condiciones harán abortar la migración?
¿cómo se vuelve al servidor anterior?
¿qué datos pueden haber cambiado durante las pruebas?
¿en qué momento se bloquearán las escrituras?
```

Hasta el cutover, el servidor original debe seguir siendo la fuente
autoritativa.

------------------------------------------------------------------------

## 13.6 Estrategias de migración

No existe una única estrategia válida para todos los cambios.

### 13.6.1 Trasladar físicamente DATA

Consiste en retirar el disco DATA del servidor original y conectarlo al
nuevo.

Ventajas:

``` text
rápido
no requiere copiar todo DATA
mantiene filesystem y UUID
```

Inconvenientes:

``` text
reduce la facilidad de rollback
manipula físicamente la copia principal
puede introducir diferencias de interfaz o alimentación
```

No es la opción preferida cuando existe un disco nuevo disponible.

### 13.6.2 Copiar DATA a un disco nuevo

Permite conservar el DATA original intacto.

Una estrategia eficiente consiste en realizar una copia inicial mientras
producción sigue funcionando y una sincronización final durante la
ventana de mantenimiento.

Debe preservarse:

``` text
estructura
propietarios
grupos
permisos
timestamps
contenido
```

Para este tipo de operación puede utilizarse `rsync` con opciones
adecuadas, pero el comando final debe revisarse para las rutas concretas
antes de ejecutarlo.

### 13.6.3 Restaurar DATA desde Borg

Es más lento que una copia directa cuando el origen está disponible,
pero tiene ventajas:

-   prueba simultáneamente la recuperabilidad;
-   desacopla la migración del disco DATA original;
-   reproduce el procedimiento de Disaster Recovery.

Debe seleccionarse un archive coherente con la base de datos que se vaya
a restaurar.

### 13.6.4 Reutilizar BACKUP

El disco BACKUP puede trasladarse al nuevo servidor una vez que se haya
dejado de utilizar en el antiguo.

Antes:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg check --show-rc /srv/backup/borg
```

Después del traslado debe comprobarse nuevamente el mount y el acceso al
repositorio.

### 13.6.5 Reconstruir SYSTEM

Para una renovación completa se recomienda una instalación limpia de
Debian y reconstrucción de servicios.

Esto evita depender de una clonación bit a bit del sistema antiguo y
mejora la reproducibilidad de la infraestructura.

### 13.6.6 Migración completa a hardware nuevo

La estrategia más conservadora es:

``` text
servidor original permanece intacto
          ↓
nuevo SYSTEM
          ↓
nuevo DATA
          ↓
BACKUP disponible
          ↓
restauración/configuración
          ↓
pruebas locales
          ↓
ventana de mantenimiento
          ↓
sincronización final
          ↓
cutover
          ↓
validación
          ↓
nuevo backup
          ↓
retirada diferida del origen
```

------------------------------------------------------------------------

## 13.7 Preparación del nuevo servidor

### 13.7.1 Debian

Instalar Debian estable equivalente o una versión previamente validada
para el stack.

Configurar:

``` text
hostname temporal o definitivo
usuario administrativo
sudo
SSH
actualizaciones
zona horaria
```

No es necesario copiar toda la configuración del antiguo host sin
revisión.

### 13.7.2 Red temporal

Asignar al nuevo servidor una IP distinta de producción.

Ejemplo conceptual:

``` text
hpserver actual → 192.168.1.10
nuevo servidor  → otra IP libre
```

No reutilizar `192.168.1.10` mientras el original siga conectado.

### 13.7.3 Docker

Instalar Docker desde el repositorio oficial y comprobar:

``` bash
docker --version
docker compose version
systemctl status docker --no-pager
```

Restaurar o reconciliar:

``` text
/etc/docker/daemon.json
override systemd de Docker
```

### 13.7.4 Filesystems

Preparar DATA y BACKUP según la estrategia seleccionada.

Ejemplo para identificar:

``` bash
lsblk -f
sudo blkid
```

Crear los puntos:

``` bash
sudo mkdir -p /srv/storage
sudo mkdir -p /srv/backup
```

`fstab` debe utilizar los UUID reales del nuevo hardware.

### 13.7.5 Estructura de directorios

Preparar:

``` text
/srv/docker/nextcloud/
/srv/storage/nextcloud-data/
/srv/backup/
```

Los permisos deben proceder de la configuración real y no de valores
improvisados.

### 13.7.6 Recovery bundle

El recovery bundle del servidor original se utiliza como referencia.

No debe copiarse íntegramente sobre `/etc` sin revisar.

Deben reconciliarse especialmente:

``` text
UUID
red
interfaces
hostname
systemd
rutas
versiones
hardware
```

------------------------------------------------------------------------

## 13.8 Migración de DATA

### 13.8.1 Copia inicial

Si se utiliza un disco nuevo, puede realizarse una primera copia antes
de la ventana de mantenimiento.

El destino no debe exponerse como servidor de producción durante esta
fase.

Antes:

``` bash
sudo /usr/local/sbin/check-nextcloud-data.sh
```

en el servidor original.

### 13.8.2 Detener escrituras para la sincronización final

La copia inicial puede quedar desactualizada mientras los usuarios
siguen utilizando Nextcloud.

Para el corte definitivo debe impedirse que se generen nuevas
diferencias.

Activar mantenimiento en el origen:

``` bash
cd /srv/docker/nextcloud

sudo docker compose exec -u www-data app \
  php occ maintenance:mode --on
```

También deben controlarse procesos automáticos que puedan modificar el
estado.

### 13.8.3 Sincronización final

Realizar la última sincronización sólo después de detener las
escrituras.

La dirección debe comprobarse dos veces antes de ejecutar cualquier
herramienta con opciones de borrado.

Regla:

``` text
ORIGEN = servidor antiguo
DESTINO = servidor nuevo
```

Una inversión de origen y destino puede destruir el conjunto correcto.

### 13.8.4 Propietarios y permisos

Comprobar:

``` bash
sudo ls -ld /srv/storage/nextcloud-data
sudo ls -la /srv/storage/nextcloud-data/.ncdata
```

No aplicar un `chown -R` o `chmod -R` como medida preventiva si no
existe una necesidad demostrada.

### 13.8.5 `.ncdata`

Debe existir:

``` text
/srv/storage/nextcloud-data/.ncdata
```

La presencia y contenido esperado de este marcador forma parte del guard
de DATA.

### 13.8.6 Validación

En el nuevo host:

``` bash
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh
echo $?
sudo du -sh /srv/storage/nextcloud-data
```

Cuando exista una referencia fiable, comparar hashes de archivos
seleccionados.

------------------------------------------------------------------------

## 13.9 Migración de PostgreSQL

La migración reproducible no se basa en copiar arbitrariamente el
directorio interno de PostgreSQL.

El procedimiento preferido es:

``` text
dump
  +
roles globales
  ↓
PostgreSQL limpio
  ↓
roles
  ↓
base de datos
  ↓
validación
```

### 13.9.1 Dump de la base

El backup automatizado genera:

``` text
/var/backups/nextcloud/nextcloud.dump
```

en formato custom de PostgreSQL.

Puede validarse con una versión compatible:

``` bash
pg_restore -l nextcloud.dump >/dev/null
echo $?
```

Si el host no dispone de `pg_restore`, puede utilizarse una imagen
PostgreSQL de la misma major.

### 13.9.2 Roles globales

El backup incluye:

``` text
/var/backups/nextcloud/postgres-roles.sql
```

generado mediante:

``` text
pg_dumpall --roles-only
```

Este archivo puede contener hashes de contraseñas.

Debe mantenerse con permisos restrictivos y no mostrarse ni publicarse.

### 13.9.3 PostgreSQL limpio

En el nuevo host, iniciar PostgreSQL con el volumen persistente nuevo y
vacío.

La inicialización de Compose puede crear ya el rol indicado mediante
`POSTGRES_USER`.

Antes de restaurar roles debe comprobarse qué roles existen para evitar
conflictos de `CREATE ROLE`.

### 13.9.4 Restauración de roles

Si la inicialización ya creó `nextcloud`, el SQL recuperado debe
reconciliarse para no intentar crearlo por segunda vez.

El procedimiento probado consiste en eliminar únicamente la línea
exacta:

``` text
CREATE ROLE nextcloud;
```

del archivo de restauración temporal y conservar los `ALTER ROLE`
recuperados.

Ejemplo:

``` bash
sed '/^CREATE ROLE nextcloud;$/d' \
  postgres-roles.sql > roles-restore.sql

chmod 600 roles-restore.sql
```

Aplicar con parada ante error:

``` bash
docker compose exec -T db sh -c \
  'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d postgres' \
  < roles-restore.sql
```

La orden debe adaptarse a la ubicación de staging utilizada.

### 13.9.5 Restauración del dump

Una vez disponibles los roles originales:

``` bash
docker compose exec -T db sh -c \
  'pg_restore --exit-on-error -U "$POSTGRES_USER" -d "$POSTGRES_DB"' \
  < nextcloud.dump
```

No utilizar `--no-owner` como sustituto automático de los roles que
realmente forman parte del estado de producción.

### 13.9.6 Propietarios y ACL

Después deben comprobarse:

``` text
roles
propietarios de objetos
schema public
ACL
tablas esenciales
```

No basta con que `pg_restore` finalice si se ha alterado
involuntariamente el modelo de ownership.

### 13.9.7 Validación

``` bash
docker compose exec -T db sh -c \
  'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

Después validar Nextcloud con OCC.

------------------------------------------------------------------------

## 13.10 Migración de Nextcloud

### 13.10.1 `compose.yml`

Restaurar:

``` text
/srv/docker/nextcloud/compose.yml
```

y validar:

``` bash
cd /srv/docker/nextcloud
sudo docker compose config --quiet
echo $?
```

### 13.10.2 `.env`

Restaurar el `.env` real únicamente en el servidor privado.

Debe conservar permisos restrictivos.

No debe incorporarse a Git.

### 13.10.3 Volumen de aplicación

Restaurar:

``` text
/srv/docker/nextcloud/volumes/nextcloud/
```

desde la fuente seleccionada.

DATA continúa separado en:

``` text
/srv/storage/nextcloud-data/
```

### 13.10.4 Redis

Redis puede inicializarse mediante Compose.

Comprobar:

``` bash
sudo docker compose up -d redis
sudo docker compose exec redis redis-cli ping
```

### 13.10.5 Cron

Arrancar sólo cuando aplicación y base estén validadas:

``` bash
sudo docker compose up -d cron
```

### 13.10.6 Modo mantenimiento

El estado de mantenimiento puede estar activo en una copia generada
durante el backup.

Comprobar:

``` bash
sudo docker compose exec -u www-data app php occ status
```

Desactivar sólo después de validar:

``` bash
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --off
```

------------------------------------------------------------------------

## 13.11 Reconciliación con el nuevo hardware

### 13.11.1 `/etc/fstab`

Nunca copiar el `fstab` antiguo y reiniciar sin revisar.

Comprobar los UUID nuevos:

``` bash
lsblk -f
sudo blkid
```

Editar y validar:

``` bash
sudo findmnt --verify --verbose
```

### 13.11.2 UUID de DATA

Actualizar cualquier referencia al UUID antiguo en:

``` text
/etc/fstab
check-nextcloud-data.sh
backup-nextcloud.sh
otros guards o scripts
```

### 13.11.3 UUID de BACKUP

Reconciliar también BACKUP en:

``` text
/etc/fstab
scripts de backup
scripts Borg
scripts de comprobación
```

### 13.11.4 Búsqueda global

Una comprobación útil:

``` bash
sudo grep -R "UUID_ANTIGUO" \
  /etc \
  /usr/local/sbin \
  /srv/docker 2>/dev/null
```

La búsqueda global evita depender de una lista mental de archivos.

### 13.11.5 systemd

Restaurar y revisar:

``` text
nextcloud.service
override de docker.service
timers de backup
timers Borg
SMART
alertas
```

Después:

``` bash
sudo systemctl daemon-reload
sudo systemd-analyze verify \
  /etc/systemd/system/nextcloud.service
```

### 13.11.6 Red

Mientras se realizan pruebas debe mantenerse la IP temporal.

La IP definitiva se asignará durante el cutover.

Si cambia el nombre de la interfaz de red, la configuración debe
adaptarse al nuevo hardware.

------------------------------------------------------------------------

## 13.12 Cloudflare Tunnel durante la migración

### 13.12.1 Riesgo de dos instancias

El `.env` restaurado puede contener el token real del túnel.

Si el nuevo host arranca `cloudflared` antes del cutover, puede existir
más de un conector utilizando la identidad de producción.

Por tanto:

``` text
cloudflared = DESACTIVADO durante preparación y pruebas
```

### 13.12.2 Mantenerlo desactivado

No ejecutar inicialmente:

``` bash
docker compose up -d
```

si eso implica arrancar todos los servicios sin revisar.

Arrancar explícitamente:

``` bash
sudo docker compose up -d db redis app cron
```

### 13.12.3 Cambio de producción

`cloudflared` sólo debe activarse cuando:

``` text
servidor antiguo ya no presta el servicio
nuevo servidor está validado
DATA definitivo está sincronizado
PostgreSQL definitivo está restaurado
IP/configuración final están correctas
```

### 13.12.4 Reactivación

``` bash
sudo docker compose up -d cloudflared
sudo docker compose logs --tail=100 cloudflared
```

Después comprobar HTTPS externo.

------------------------------------------------------------------------

## 13.13 Pruebas antes del cambio

### 13.13.1 PostgreSQL

``` bash
sudo docker compose exec -T db sh -c \
  'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

Comprobar además roles, tablas, ownership y ACL.

### 13.13.2 Nextcloud

``` bash
sudo docker compose exec -u www-data app php occ status
```

Esperado:

``` text
installed: true
needsDbUpgrade: false
```

El modo mantenimiento dependerá de la fase de la migración.

### 13.13.3 Login

Acceder por LAN utilizando la IP temporal del nuevo servidor.

No depender de Cloudflare para esta validación.

### 13.13.4 Lectura

Abrir y descargar archivos existentes.

### 13.13.5 Escritura

Cuando el conjunto de datos de prueba ya sea coherente y la prueba no
pueda confundirse con producción:

``` text
crear directorio
subir archivo
descargar archivo
comprobar contenido
```

Si todavía se prevé una sincronización final desde el origen, recordar
que estos cambios de prueba en el destino pueden ser reemplazados. No
deben considerarse datos de usuario definitivos.

### 13.13.6 Persistencia

Realizar un reboot del nuevo host antes del cutover cuando ya estén
instaladas las unidades systemd definitivas:

``` bash
sudo reboot
```

Después:

``` bash
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh
systemctl status nextcloud.service --no-pager
sudo docker ps
```

`cloudflared` debe continuar desactivado durante esta fase.

------------------------------------------------------------------------

## 13.14 Cutover

El cutover es el momento en el que el nuevo servidor pasa a ser
producción.

Debe realizarse como una operación breve y controlada.

### 13.14.1 Ventana de mantenimiento

Informar a los usuarios si procede y evitar nuevas escrituras.

En el servidor antiguo:

``` bash
cd /srv/docker/nextcloud
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --on
```

### 13.14.2 Detener procesos que puedan escribir

Detener o controlar:

``` text
cron
operaciones de usuarios
tareas automáticas
```

La finalidad es crear un punto final coherente.

### 13.14.3 Sincronización final

Si DATA se ha copiado previamente, realizar la sincronización final
desde el servidor antiguo al nuevo.

Verificar cuidadosamente:

``` text
origen
destino
opciones de rsync
opciones de borrado
```

Antes de cualquier `--delete`, comprobar dos veces la dirección.

### 13.14.4 Estado final de PostgreSQL

La base de datos utilizada por el nuevo servidor debe corresponder al
mismo punto lógico que DATA.

Si se han producido cambios desde el dump utilizado durante las pruebas,
generar un dump final y roles actualizados durante la ventana de
mantenimiento y restaurarlos en el destino.

No utilizar como producción una base de datos de prueba desactualizada.

### 13.14.5 Detener el servidor antiguo

Cuando la copia final esté completa:

``` text
detener stack antiguo
o apagar servidor antiguo
```

No permitir que ambos acepten escrituras después del corte.

### 13.14.6 Asignar la identidad de producción

Configurar en el nuevo host:

``` text
IP definitiva
hostname si procede
configuración de red definitiva
```

Si se reutiliza `192.168.1.10`, el antiguo servidor debe estar
desconectado o sin esa IP.

### 13.14.7 Activar el nuevo servicio

Comprobar:

``` bash
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh
```

Arrancar los componentes de producción y, finalmente, `cloudflared`.

### 13.14.8 Salir del modo mantenimiento

Después de las comprobaciones:

``` bash
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --off
```

------------------------------------------------------------------------

## 13.15 Validación posterior

### 13.15.1 LAN

Comprobar acceso mediante la IP definitiva.

Validar:

``` text
login
navegación
lectura
escritura
descarga
```

### 13.15.2 HTTPS externo

Después de activar el túnel comprobar el dominio público y HTTPS.

Un fallo externo con LAN funcional debe investigarse como problema de
acceso remoto antes de modificar la aplicación.

### 13.15.3 Correo

Realizar una prueba de correo de Nextcloud cuando la migración haya
cambiado Docker, red, DNS o configuración SMTP.

### 13.15.4 Backup

Comprobar:

``` bash
findmnt /srv/backup
systemctl status nextcloud-backup.timer --no-pager
```

No ejecutar todavía tareas destructivas sobre el servidor antiguo.

### 13.15.5 Timers

``` bash
systemctl list-timers --all | grep -E \
  'nextcloud-backup|borg-check|borg-verify-data|smart-check'
```

### 13.15.6 Reboot

Una migración completa no se considera validada hasta superar un
reinicio del nuevo servidor:

``` bash
sudo reboot
```

Después:

``` bash
findmnt /srv/storage
findmnt /srv/backup
sudo /usr/local/sbin/check-nextcloud-data.sh
systemctl status nextcloud.service --no-pager
cd /srv/docker/nextcloud
sudo docker compose ps
```

Comprobar también que los datos creados después del cutover persisten.

------------------------------------------------------------------------

## 13.16 Rollback

### 13.16.1 Cuándo abortar

Ejemplos:

``` text
DATA incorrecto
guard falla
PostgreSQL no restaura limpiamente
ownership/ACL incoherentes
Nextcloud no permite login
errores de escritura
reboot no supera validación
problemas de hardware nuevo
```

No continuar acumulando cambios sobre un destino cuyo estado no se
comprende.

### 13.16.2 Antes del cutover

El rollback es sencillo:

``` text
detener destino
mantener cloudflared desactivado
volver a utilizar el servidor original
```

El origen continúa siendo autoritativo.

### 13.16.3 Después del cutover

El rollback es más delicado porque pueden existir escrituras nuevas en
el servidor nuevo.

No debe simplemente encenderse el servidor antiguo y permitir acceso.

Primero:

``` text
detener nuevas escrituras
identificar qué datos cambiaron
preservar el nuevo estado
decidir cómo reconciliar DATA y PostgreSQL
```

### 13.16.4 Evitar divergencia

La regla es:

> **En cada momento debe existir una única instancia autoritativa que
> acepte escrituras de usuarios.**

Dos servidores escribibles con estados diferentes convierten un rollback
sencillo en un problema de reconciliación de datos.

------------------------------------------------------------------------

## 13.17 Primera copia en el nuevo servidor

Después de validar la migración debe crearse inmediatamente un nuevo
punto de recuperación.

Antes:

``` bash
findmnt /srv/storage
findmnt /srv/backup
sudo /usr/local/sbin/check-nextcloud-data.sh
```

Revisar que los scripts contienen los UUID del nuevo entorno.

Ejecutar:

``` bash
sudo systemctl start nextcloud-backup.service
```

Comprobar:

``` bash
systemctl status nextcloud-backup.service --no-pager
journalctl -u nextcloud-backup.service -n 100 --no-pager
```

Confirmar un archive nuevo:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

El archive debe contener, como mínimo, los componentes críticos
documentados en el capítulo 11:

``` text
DATA
compose.yml
.env
volumen de aplicación
nextcloud.dump
postgres-roles.sql
recovery bundle
```

Si se ha creado un repositorio Borg nuevo, deben exportarse y
conservarse externamente sus nuevos datos de recuperación.

------------------------------------------------------------------------

## 13.18 Retirada del servidor antiguo

### 13.18.1 Periodo de conservación

El servidor original no debe borrarse inmediatamente después del
cutover.

Debe conservarse sin prestar servicio durante un periodo prudente hasta
comprobar:

``` text
estabilidad del nuevo host
backup nuevo válido
reboot correcto
acceso local y remoto
correo
timers
datos de usuarios
```

Mantenerlo apagado reduce el riesgo de que vuelva accidentalmente a
producción.

### 13.18.2 Preservación antes del borrado

Antes de reutilizar discos antiguos:

``` text
confirmar que no contienen la única copia de ningún dato
confirmar que el nuevo backup es válido
confirmar que las credenciales externas están disponibles
documentar seriales y destino del hardware
```

### 13.18.3 Borrado seguro

El método depende del tipo de dispositivo.

No debe aplicarse automáticamente el mismo procedimiento a HDD y SSD.

En SSD deben considerarse mecanismos específicos del dispositivo, como
secure erase/sanitize cuando sean compatibles.

En HDD puede utilizarse un procedimiento de sobrescritura adecuado
cuando sea necesario.

El borrado debe planificarse sólo después de finalizar el periodo de
rollback.

### 13.18.4 Reutilización

El hardware antiguo puede destinarse a:

``` text
laboratorio
DRT
servidor de pruebas
backup adicional
otros servicios domésticos
```

pero sólo después de eliminar o aislar correctamente la configuración y
los secretos de producción.

------------------------------------------------------------------------

## 13.19 Checklist de migración

### Fase 1 --- Inventario

``` text
[ ] hardware registrado
[ ] SYSTEM identificado
[ ] DATA identificado
[ ] BACKUP identificado
[ ] UUID registrados
[ ] red registrada
[ ] versiones registradas
[ ] configuración externa disponible
```

### Fase 2 --- Protección

``` text
[ ] servidor origen saludable
[ ] backup reciente
[ ] backup satisfactorio
[ ] Borg accesible
[ ] recovery bundle actualizado
[ ] passphrase disponible
[ ] clave Borg externa disponible
[ ] rollback definido
```

### Fase 3 --- Preparar destino

``` text
[ ] Debian instalado
[ ] IP temporal
[ ] Docker instalado
[ ] DATA preparado
[ ] BACKUP preparado
[ ] fstab reconciliado
[ ] scripts instalados
[ ] UUID reconciliados
[ ] systemd instalado
[ ] cloudflared desactivado
```

### Fase 4 --- Restaurar/migrar

``` text
[ ] DATA copiado/restaurado
[ ] .ncdata presente
[ ] guard = 0
[ ] compose.yml restaurado
[ ] .env restaurado
[ ] volumen Nextcloud restaurado
[ ] PostgreSQL limpio
[ ] roles globales restaurados
[ ] dump restaurado
[ ] propietarios y ACL validados
[ ] Redis operativo
[ ] cron operativo
```

### Fase 5 --- Pruebas previas

``` text
[ ] Nextcloud status correcto
[ ] login local
[ ] lectura
[ ] escritura de prueba
[ ] descarga
[ ] reboot
[ ] persistencia
[ ] cloudflared continúa desactivado
```

### Fase 6 --- Cutover

``` text
[ ] ventana de mantenimiento
[ ] escrituras detenidas
[ ] sincronización final DATA
[ ] base final coherente con DATA
[ ] servidor antiguo detenido
[ ] IP de producción asignada
[ ] guard = 0
[ ] stack iniciado
[ ] cloudflared activado
[ ] mantenimiento desactivado
```

### Fase 7 --- Validación de producción

``` text
[ ] LAN
[ ] HTTPS externo
[ ] login
[ ] lectura
[ ] escritura
[ ] descarga
[ ] correo
[ ] timers
[ ] alertas
[ ] reboot final
[ ] persistencia
```

### Fase 8 --- Nuevo punto de recuperación

``` text
[ ] backup ejecutado
[ ] archive nuevo presente
[ ] nextcloud.dump presente
[ ] postgres-roles.sql presente
[ ] recovery bundle presente
[ ] Borg validado
```

### Fase 9 --- Cierre

``` text
[ ] servidor antiguo conservado temporalmente
[ ] documentación actualizada
[ ] CHANGELOG actualizado
[ ] nuevos UUID documentados de forma privada cuando corresponda
[ ] configuración pública sanitizada
[ ] periodo de rollback finalizado
[ ] hardware antiguo retirado o reutilizado
```

------------------------------------------------------------------------

## 13.20 Flujo resumido

Una migración completa debe seguir el principio:

``` text
INVENTARIAR
     ↓
PROTEGER
     ↓
PREPARAR DESTINO
     ↓
COPIAR / RESTAURAR
     ↓
RECONCILIAR
     ↓
PROBAR EN AISLAMIENTO
     ↓
REBOOT
     ↓
DETENER ESCRITURAS EN ORIGEN
     ↓
SINCRONIZACIÓN FINAL
     ↓
CUTOVER
     ↓
VALIDAR PRODUCCIÓN
     ↓
REBOOT FINAL
     ↓
NUEVO BACKUP
     ↓
CONSERVAR ORIGEN
     ↓
DOCUMENTAR
```

La principal ventaja de una migración frente a un Disaster Recovery es
disponer todavía del sistema original.

Esa ventaja debe conservarse durante todo el proceso: el origen no se
destruye para construir el destino; se mantiene como referencia y
rollback hasta que el nuevo `hpserver` haya demostrado que puede
funcionar de forma autónoma, persistente y recuperable.
