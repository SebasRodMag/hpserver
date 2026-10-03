# 12. Operación y mantenimiento

## 12.1 Objetivo y alcance

Este capítulo define los procedimientos de operación ordinaria y
mantenimiento de `hpserver`.

Su objetivo es disponer de una referencia práctica para administrar el
servidor sin tener que reconstruir mentalmente su arquitectura cada vez
que se realiza una intervención.

El capítulo cubre:

-   comprobación del estado general;
-   arranque, parada y reinicio;
-   operaciones habituales de Nextcloud;
-   administración básica de Docker;
-   actualizaciones;
-   almacenamiento;
-   backups;
-   PostgreSQL y Redis;
-   Cloudflare Tunnel;
-   correo y alertas;
-   logs y diagnóstico;
-   rutinas periódicas de mantenimiento.

No sustituye a los capítulos especializados. El diseño del backup se
documenta en el capítulo 9, la monitorización en el capítulo 10 y los
procedimientos de recuperación ante desastres en el capítulo 11.

La regla general de operación es:

``` text
COMPROBAR
   ↓
PROTEGER
   ↓
MODIFICAR
   ↓
VALIDAR
   ↓
REINICIAR SI PROCEDE
   ↓
VOLVER A VALIDAR
   ↓
DOCUMENTAR
```

Una intervención no debe considerarse terminada únicamente porque el
servicio funcione inmediatamente después del cambio. Cuando la
modificación afecte al arranque, almacenamiento, Docker, systemd o
configuración persistente, debe comprobarse también su comportamiento
después de un reboot.

------------------------------------------------------------------------

## 12.2 Estado normal del servidor

Antes de diagnosticar una anomalía es necesario conocer qué aspecto
tiene el sistema cuando funciona correctamente.

### 12.2.1 Filesystems esperados

La infraestructura utiliza tres funciones de almacenamiento:

  -----------------------------------------------------------------------
  Función                 Punto de montaje        Uso
  ----------------------- ----------------------- -----------------------
  SYSTEM                  `/`                     Debian, Docker,
                                                  configuración y
                                                  volúmenes de aplicación

  DATA                    `/srv/storage`          Datos de usuarios de
                                                  Nextcloud

  BACKUP                  `/srv/backup`           Repositorio Borg
  -----------------------------------------------------------------------

Los nombres `/dev/sdX` no deben utilizarse como identificadores
permanentes. Pueden cambiar entre arranques o al conectar dispositivos.

Comprobar:

``` bash
lsblk -f
findmnt /
findmnt /srv/storage
findmnt /srv/backup
```

DATA es obligatorio para iniciar Nextcloud. BACKUP utiliza una política
menos estricta para que su ausencia no impida arrancar el host.

### 12.2.2 Servicios systemd

Los componentes relevantes incluyen:

``` text
docker.service
nextcloud.service
nextcloud-backup.timer
borg-check.timer
borg-verify-data.timer
smart-check.timer
```

Además existe el mecanismo de alertas:

``` text
hpserver-alert@.service
```

Consultar:

``` bash
systemctl status docker.service --no-pager
systemctl status nextcloud.service --no-pager
systemctl list-timers --all
```

`nextcloud.service` controla el arranque seguro del stack y depende de
que DATA esté correctamente montado.

### 12.2.3 Contenedores Docker

El stack principal se encuentra en:

``` text
/srv/docker/nextcloud/
```

Sus componentes son:

``` text
app
db
redis
cron
cloudflared
```

Comprobar:

``` bash
cd /srv/docker/nextcloud
sudo docker compose ps
```

PostgreSQL y Redis disponen de healthchecks y deben alcanzar un estado
saludable.

### 12.2.4 Estado esperado de Nextcloud

``` bash
cd /srv/docker/nextcloud
sudo docker compose exec -u www-data app php occ status
```

En funcionamiento normal debe observarse, entre otros valores:

``` text
installed: true
maintenance: false
needsDbUpgrade: false
```

La versión exacta puede cambiar después de una actualización y no debe
utilizarse como indicador único de salud.

### 12.2.5 Acceso local y remoto

El acceso local está disponible mediante:

``` text
http://192.168.1.10:8080
```

El acceso externo utiliza Cloudflare Tunnel y HTTPS.

Ante un problema remoto debe probarse primero el acceso local. Si
Nextcloud funciona en LAN pero no externamente, el diagnóstico debe
centrarse después en `cloudflared`, DNS, Cloudflare o conectividad
externa, evitando modificar innecesariamente Nextcloud.

------------------------------------------------------------------------

## 12.3 Comprobación rápida del estado

Esta sección sirve como inspección inicial antes de realizar
mantenimiento o cuando se sospecha una anomalía.

### 12.3.1 Almacenamiento

``` bash
lsblk -f
df -h
findmnt /srv/storage
findmnt /srv/backup
sudo /usr/local/sbin/check-nextcloud-data.sh
echo $?
```

El guard debe devolver `0`.

También conviene comprobar inodos:

``` bash
df -i
```

Un filesystem puede quedarse sin inodos aunque todavía muestre espacio
disponible.

### 12.3.2 Docker

``` bash
systemctl is-active docker
cd /srv/docker/nextcloud
sudo docker compose ps
```

Para una inspección más general:

``` bash
sudo docker ps
sudo docker system df
```

### 12.3.3 Nextcloud

``` bash
sudo docker compose exec -u www-data app php occ status
```

Comprobar que no existe un modo mantenimiento inesperado ni una
actualización de base de datos pendiente.

### 12.3.4 PostgreSQL y Redis

PostgreSQL:

``` bash
sudo docker compose exec -T db sh -c \
  'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

Redis:

``` bash
sudo docker compose exec redis redis-cli ping
```

El resultado esperado de Redis es:

``` text
PONG
```

### 12.3.5 Backup y timers

``` bash
systemctl list-timers --all | grep -E \
  'nextcloud-backup|borg-check|borg-verify-data|smart-check'
```

Consultar la última ejecución:

``` bash
systemctl status nextcloud-backup.service --no-pager
journalctl -u nextcloud-backup.service -n 50 --no-pager
```

Un servicio `oneshot` puede aparecer como `inactive (dead)` después de
terminar correctamente. Debe revisarse el resultado de la ejecución y no
interpretar ese estado aisladamente como un fallo.

### 12.3.6 Logs y alertas

Una revisión rápida del boot actual:

``` bash
journalctl -b -p warning --no-pager
```

Esto no implica que todas las advertencias sean incidencias. Deben
interpretarse en su contexto.

------------------------------------------------------------------------

## 12.4 Arranque y parada controlada

### 12.4.1 Arranque normal

El arranque normal no requiere ejecutar manualmente `docker compose up`.

La secuencia diseñada es aproximadamente:

``` text
Debian
  ↓
mount de DATA
  ↓
Docker
  ↓
guard de DATA
  ↓
nextcloud.service
  ↓
stack Nextcloud
```

Comprobar después:

``` bash
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh
systemctl status nextcloud.service --no-pager
sudo docker ps
```

### 12.4.2 Parada normal

Para apagar completamente el servidor:

``` bash
sudo shutdown -h now
```

o:

``` bash
sudo systemctl poweroff
```

Debe evitarse cortar alimentación deliberadamente mientras PostgreSQL o
los discos están escribiendo.

### 12.4.3 Reinicio completo

``` bash
sudo reboot
```

Después:

``` bash
findmnt /srv/storage
findmnt /srv/backup
systemctl status nextcloud.service --no-pager
cd /srv/docker/nextcloud
sudo docker compose ps
```

Cuando el cambio realizado afecte al arranque, esta validación forma
parte de la propia intervención.

### 12.4.4 Arranque manual del stack

Para diagnóstico:

``` bash
cd /srv/docker/nextcloud
sudo docker compose up -d
```

Sin embargo, en producción debe preferirse:

``` bash
sudo systemctl start nextcloud.service
```

porque incorpora las dependencias y el guard diseñados para evitar
arrancar Nextcloud sobre un DATA incorrecto.

### 12.4.5 Qué no hacer

No debe utilizarse como procedimiento habitual:

``` bash
docker compose up -d
```

sin comprobar DATA cuando exista una incidencia de almacenamiento.

Tampoco deben crearse manualmente directorios vacíos para sustituir un
mount que no aparece. Esa acción puede ocultar la ausencia del
filesystem real y provocar que Nextcloud escriba sobre SYSTEM.

------------------------------------------------------------------------

## 12.5 Operaciones habituales de Nextcloud

Los comandos OCC deben ejecutarse como el usuario web del contenedor.

``` bash
cd /srv/docker/nextcloud
```

### 12.5.1 Estado con OCC

``` bash
sudo docker compose exec -u www-data app php occ status
```

Otros comandos útiles:

``` bash
sudo docker compose exec -u www-data app php occ config:list system
sudo docker compose exec -u www-data app php occ app:list
```

La salida de configuración puede contener información que no debe
publicarse sin revisión.

### 12.5.2 Modo mantenimiento

Activar:

``` bash
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --on
```

Desactivar:

``` bash
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --off
```

Comprobar siempre su estado al terminar una intervención.

### 12.5.3 Gestión básica de usuarios

Listar:

``` bash
sudo docker compose exec -u www-data app php occ user:list
```

Información:

``` bash
sudo docker compose exec -u www-data app \
  php occ user:info USUARIO
```

Las operaciones destructivas sobre usuarios deben realizarse con
especial precaución y no forman parte de una rutina de mantenimiento.

### 12.5.4 Cron

Comprobar el modo configurado:

``` bash
sudo docker compose exec -u www-data app \
  php occ background:cron
```

El contenedor `cron` debe permanecer activo:

``` bash
sudo docker compose ps cron
```

### 12.5.5 Logs de Nextcloud

Consultar desde el host:

``` bash
sudo tail -n 100 \
  /srv/storage/nextcloud-data/nextcloud.log
```

Para seguimiento:

``` bash
sudo tail -f \
  /srv/storage/nextcloud-data/nextcloud.log
```

Evitar publicar logs completos sin revisar previamente tokens, nombres
de usuario, direcciones o información privada.

------------------------------------------------------------------------

## 12.6 Gestión de Docker

### 12.6.1 Estado del stack

``` bash
cd /srv/docker/nextcloud
sudo docker compose ps
```

Inspección general:

``` bash
sudo docker ps -a
sudo docker images
sudo docker system df
```

### 12.6.2 Logs

Todos los servicios:

``` bash
sudo docker compose logs --tail=100
```

Servicio concreto:

``` bash
sudo docker compose logs --tail=100 app
sudo docker compose logs --tail=100 db
sudo docker compose logs --tail=100 redis
sudo docker compose logs --tail=100 cloudflared
```

Seguimiento:

``` bash
sudo docker compose logs -f app
```

### 12.6.3 Recrear un contenedor

Un contenedor es desechable; los datos persistentes no deben depender de
su capa writable.

Ejemplo:

``` bash
sudo docker compose up -d --force-recreate app
```

Antes de hacerlo debe comprobarse que los bind mounts y volúmenes
persistentes son correctos.

Recrear un contenedor no equivale a borrar su volumen persistente.

### 12.6.4 Actualización de imágenes

No actualizar indiscriminadamente todos los componentes.

Procedimiento general:

``` bash
sudo docker compose pull
sudo docker compose up -d
```

Debe utilizarse sólo después de revisar las versiones configuradas y el
procedimiento específico del componente.

Los servicios con estado, especialmente PostgreSQL, requieren mayor
precaución. Un cambio de versión mayor no debe tratarse como una simple
actualización de imagen.

### 12.6.5 Limpieza de recursos Docker

Inspeccionar primero:

``` bash
sudo docker system df
```

Una limpieza prudente de imágenes no utilizadas puede realizarse con:

``` bash
sudo docker image prune
```

No utilizar rutinariamente comandos agresivos como:

``` text
docker system prune -a --volumes
```

sin comprender exactamente qué recursos eliminarán.

------------------------------------------------------------------------

## 12.7 Actualización del sistema

Las actualizaciones deben realizarse por capas y con una copia válida
previa.

### 12.7.1 Política antes de actualizar

Antes de una actualización relevante:

``` text
[ ] comprobar salud actual
[ ] comprobar DATA
[ ] comprobar último backup
[ ] comprobar espacio libre
[ ] revisar notas de versión
[ ] identificar procedimiento de rollback
[ ] actualizar un componente cada vez cuando sea posible
```

Una actualización no es el momento adecuado para descubrir que el backup
llevaba varios días fallando.

### 12.7.2 Debian

``` bash
sudo apt update
apt list --upgradable
```

Después de revisar:

``` bash
sudo apt upgrade
```

Para cambios que puedan instalar/eliminar dependencias:

``` bash
sudo apt full-upgrade
```

debe revisarse cuidadosamente la propuesta antes de confirmarla.

Comprobar si es necesario reiniciar:

``` bash
test -f /var/run/reboot-required && \
  cat /var/run/reboot-required
```

### 12.7.3 Docker Engine y Compose

Docker procede del repositorio oficial configurado en APT.

Las actualizaciones se gestionan junto con Debian, pero deben revisarse
si incluyen cambios importantes en Engine, containerd o Compose.

Después:

``` bash
docker --version
docker compose version
systemctl status docker --no-pager
```

### 12.7.4 Nextcloud

No utilizar `latest` como mecanismo de actualización deliberada de
Nextcloud.

Antes:

``` bash
sudo docker compose exec -u www-data app php occ status
sudo systemctl start nextcloud-backup.service
```

Confirmar que el backup terminó correctamente.

Revisar la ruta de actualización admitida por Nextcloud y las notas de
la versión objetivo.

Después de cambiar la versión de imagen:

``` bash
sudo docker compose pull app cron
sudo docker compose up -d app cron
```

Comprobar:

``` bash
sudo docker compose exec -u www-data app php occ status
sudo docker compose logs --tail=100 app
```

Si la versión requiere pasos de actualización específicos, seguir la
documentación oficial de esa versión en lugar de asumir que recrear el
contenedor es suficiente.

### 12.7.5 PostgreSQL

PostgreSQL es un servicio con estado.

Una actualización dentro de la misma major puede gestionarse mediante la
imagen correspondiente después de disponer de backup válido.

Un salto de major debe tratarse como una migración planificada.

No cambiar simplemente:

``` text
postgres:18-alpine
```

por una major futura y arrancar esperando compatibilidad del directorio
de datos.

Antes de una migración deben existir, como mínimo:

``` text
nextcloud.dump
postgres-roles.sql
backup Borg validado
```

### 12.7.6 Redis

Redis contiene información auxiliar y sesiones, pero su actualización
también debe realizarse de forma controlada.

Comprobar después:

``` bash
sudo docker compose exec redis redis-cli ping
```

### 12.7.7 cloudflared

Si se mantiene una etiqueta flotante para `cloudflared`, una recreación
futura puede introducir una versión diferente.

Debe comprobarse:

``` bash
sudo docker compose logs --tail=100 cloudflared
```

y validar el acceso remoto después de cualquier actualización.

A largo plazo es preferible valorar el pinning de una versión concreta
para aumentar la reproducibilidad.

------------------------------------------------------------------------

## 12.8 Mantenimiento del almacenamiento

### 12.8.1 Espacio libre

``` bash
df -h
df -i
```

Para DATA:

``` bash
sudo du -sh /srv/storage/nextcloud-data
```

Para BACKUP:

``` bash
sudo du -sh /srv/backup/borg
```

No esperar a alcanzar el 100 % para actuar.

### 12.8.2 SMART

Los discos se revisan automáticamente mediante el servicio/timer
correspondiente.

Consulta manual:

``` bash
sudo smartctl -a /dev/sdX
```

El nombre `/dev/sdX` debe determinarse en ese momento mediante `lsblk`;
no debe copiarse ciegamente de documentación histórica.

Para una evaluación deben observarse tanto el estado global como los
atributos y errores registrados.

### 12.8.3 DATA

Comprobar:

``` bash
findmnt /srv/storage
ls -la /srv/storage/nextcloud-data/.ncdata
sudo /usr/local/sbin/check-nextcloud-data.sh
```

No modificar propietarios o permisos recursivamente como respuesta
automática a cualquier problema. Primero debe determinarse qué elemento
es incorrecto y por qué.

### 12.8.4 BACKUP

``` bash
findmnt /srv/backup
ls -ld /srv/backup/borg
```

La ausencia de BACKUP no debe confundirse con pérdida de DATA.

Si BACKUP falla, Nextcloud puede continuar funcionando, pero el servidor
queda temporalmente sin nuevas copias y la incidencia debe resolverse
con prioridad.

### 12.8.5 Mounts y UUID

``` bash
lsblk -f
sudo blkid
sudo findmnt --verify --verbose
```

Después de sustituir un disco deben reconciliarse todas las referencias
al UUID anterior.

Una búsqueda global controlada ayuda a detectar referencias olvidadas:

``` bash
sudo grep -R "UUID_ANTIGUO" \
  /etc \
  /usr/local/sbin \
  /srv/docker 2>/dev/null
```

------------------------------------------------------------------------

## 12.9 Mantenimiento del backup

### 12.9.1 Comprobar el último archive

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

Debe existir un archive reciente acorde con la programación.

### 12.9.2 Estado del timer

``` bash
systemctl status nextcloud-backup.timer --no-pager
systemctl list-timers --all | grep nextcloud-backup
```

### 12.9.3 Borg check

La comprobación automática se ejecuta mediante su timer.

Manual:

``` bash
sudo systemctl start borg-check.service
systemctl status borg-check.service --no-pager
journalctl -u borg-check.service -n 100 --no-pager
```

Un `borg check` satisfactorio es una comprobación importante del
repositorio, pero no sustituye a una restauración real.

### 12.9.4 Verificación periódica de datos

El sistema dispone de una verificación más profunda programada mediante:

``` text
borg-verify-data.timer
```

Comprobar:

``` bash
systemctl status borg-verify-data.timer --no-pager
```

Los Disaster Recovery Tests del capítulo 11 siguen siendo la validación
más fuerte de recuperabilidad.

### 12.9.5 Recovery bundle

Cada backup genera el bundle:

``` text
/var/backups/hpserver-config/
```

Contiene una selección curada de configuración necesaria para
reconstruir el host.

Debe mantenerse sincronizado con los cambios estructurales del servidor.
Si se añade un nuevo script, unit systemd o configuración imprescindible
para recuperación, debe revisarse el generador:

``` text
/usr/local/sbin/generate-hpserver-recovery.sh
```

### 12.9.6 Backup fallido

Ante un fallo:

``` text
1. no borrar el repositorio;
2. revisar el journal;
3. comprobar DATA;
4. comprobar BACKUP;
5. comprobar espacio libre;
6. comprobar Borg;
7. corregir la causa;
8. ejecutar una nueva copia manual;
9. validar el nuevo archive.
```

Consultar:

``` bash
journalctl -u nextcloud-backup.service --no-pager
```

No utilizar `borg check --repair` como primera respuesta ante una
inconsistencia. Una reparación modifica el repositorio y requiere
evaluar previamente el problema y preservar las copias disponibles.

------------------------------------------------------------------------

## 12.10 Mantenimiento de PostgreSQL y Redis

### 12.10.1 Comprobaciones básicas

PostgreSQL:

``` bash
sudo docker compose exec -T db sh -c \
  'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

Redis:

``` bash
sudo docker compose exec redis redis-cli ping
```

### 12.10.2 Logs

``` bash
sudo docker compose logs --tail=100 db
sudo docker compose logs --tail=100 redis
```

Los reinicios repetidos, errores de I/O, corrupción, falta de espacio o
fallos de autenticación deben investigarse antes de recrear servicios.

### 12.10.3 Dump manual

Para una operación excepcional puede generarse un dump manual:

``` bash
sudo docker compose exec -T db sh -c \
  'pg_dump -Fc -U "$POSTGRES_USER" "$POSTGRES_DB"' \
  > /ruta/segura/nextcloud-manual.dump
```

Los roles globales son independientes:

``` bash
sudo docker compose exec -T db sh -c \
  'pg_dumpall -U "$POSTGRES_USER" --roles-only' \
  > /ruta/segura/postgres-roles-manual.sql
```

El segundo archivo puede contener hashes de contraseñas y debe
protegerse adecuadamente.

El backup automatizado sigue siendo el mecanismo normal.

### 12.10.4 Operaciones que requieren precaución

No realizar sin un procedimiento específico:

``` text
borrar el volumen PostgreSQL
cambiar de major directamente
ejecutar SQL destructivo
cambiar propietarios masivamente
restaurar un dump sobre producción sin planificación
eliminar roles
```

------------------------------------------------------------------------

## 12.11 Cloudflare Tunnel y acceso remoto

### 12.11.1 Comprobación

``` bash
cd /srv/docker/nextcloud
sudo docker compose ps cloudflared
sudo docker compose logs --tail=100 cloudflared
```

Después comprobar acceso externo.

### 12.11.2 Desactivación temporal

Durante una recuperación, clonación o laboratorio con la configuración
real:

``` bash
sudo docker compose stop cloudflared
```

Esto es especialmente importante si `.env` contiene el token real del
túnel.

### 12.11.3 Reactivación

``` bash
sudo docker compose up -d cloudflared
sudo docker compose logs --tail=100 cloudflared
```

### 12.11.4 Diagnóstico local antes del externo

Orden recomendado:

``` text
¿DATA está correcto?
        ↓
¿Nextcloud funciona localmente?
        ↓
¿cloudflared está activo?
        ↓
¿el túnel conecta?
        ↓
¿DNS/Cloudflare son correctos?
        ↓
¿HTTPS externo responde?
```

No modificar `trusted_domains`, proxies o `overwriteprotocol` de forma
aleatoria para corregir una caída externa.

------------------------------------------------------------------------

## 12.12 Correo y alertas

### 12.12.1 SMTP de Nextcloud

Nextcloud utiliza SMTP para notificaciones.

Después de cambios en Docker, DNS o correo debe utilizarse la función de
prueba de correo de Nextcloud y revisar los logs si falla.

### 12.12.2 msmtp

Las alertas del host utilizan `msmtp`.

Los secretos se mantienen fuera del repositorio público y con permisos
restrictivos.

La configuración real no debe copiarse a Git.

### 12.12.3 Alertas systemd

Los servicios críticos pueden utilizar:

``` text
OnFailure=hpserver-alert@%n.service
```

Consultar:

``` bash
systemctl status hpserver-alert@NOMBRE.service --no-pager
```

cuando se esté diagnosticando una alerta concreta.

### 12.12.4 Prueba periódica

Después de modificar correo, DNS, msmtp o el sistema de alertas debe
realizarse una prueba controlada.

No es suficiente comprobar que el comando no muestra errores; debe
confirmarse la recepción real del mensaje.

------------------------------------------------------------------------

## 12.13 Logs y diagnóstico

### 12.13.1 journalctl

Boot actual:

``` bash
journalctl -b
```

Advertencias y errores:

``` bash
journalctl -b -p warning
```

Servicio:

``` bash
journalctl -u nextcloud.service
journalctl -u docker.service
journalctl -u nextcloud-backup.service
```

Intervalo temporal:

``` bash
journalctl --since "1 hour ago"
```

### 12.13.2 Docker logs

``` bash
cd /srv/docker/nextcloud
sudo docker compose logs --tail=200 SERVICIO
```

Usar `-f` sólo cuando se necesite seguimiento en tiempo real.

### 12.13.3 Nextcloud log

``` bash
sudo tail -n 200 \
  /srv/storage/nextcloud-data/nextcloud.log
```

Buscar el periodo exacto del incidente evita confundir errores
históricos con el problema actual.

### 12.13.4 Orden recomendado de diagnóstico

Ante una caída general:

``` text
1. host
2. discos y mounts
3. guard de DATA
4. Docker
5. PostgreSQL
6. Redis
7. Nextcloud
8. cloudflared
9. DNS/Internet/servicios externos
```

Ejemplo:

``` bash
uptime
lsblk -f
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh
systemctl status docker --no-pager
cd /srv/docker/nextcloud
sudo docker compose ps
sudo docker compose logs --tail=100 db
sudo docker compose logs --tail=100 app
```

El objetivo es localizar la capa que falla antes de modificarla.

------------------------------------------------------------------------

## 12.14 Rutina de mantenimiento

No todas las comprobaciones requieren intervención manual diaria. Los
timers automatizan gran parte del trabajo. La rutina manual sirve para
comprobar que esa automatización continúa funcionando.

### 12.14.1 Comprobaciones frecuentes

Cuando se administre el servidor o se reciba una alerta:

``` text
[ ] acceso Nextcloud correcto
[ ] contenedores esperados activos
[ ] DATA montado
[ ] espacio libre razonable
[ ] último backup reciente
[ ] ausencia de alertas pendientes
```

### 12.14.2 Mantenimiento mensual

Revisar:

``` text
[ ] actualizaciones Debian pendientes
[ ] versiones Docker
[ ] estado de contenedores
[ ] espacio SYSTEM
[ ] espacio DATA
[ ] espacio BACKUP
[ ] timers activos
[ ] últimos backups
[ ] logs de backup
[ ] SMART
[ ] alertas systemd
[ ] actualizaciones de Nextcloud disponibles
```

No es obligatorio instalar una actualización simplemente porque exista.
Debe evaluarse primero.

### 12.14.3 Mantenimiento trimestral

Revisar:

``` text
[ ] verificación profunda de Borg ejecutada
[ ] recovery bundle actualizado
[ ] secretos externos localizables
[ ] clave de recuperación Borg conservada fuera del servidor
[ ] procedimiento de Disaster Recovery vigente
[ ] cambios de arquitectura documentados
[ ] necesidad de nuevo DRT
```

### 12.14.4 Después de cambios importantes

Después de sustituir hardware, modificar mounts, actualizar una major,
cambiar el backup o alterar el arranque:

``` text
[ ] backup nuevo
[ ] recovery bundle actualizado
[ ] configuración pública actualizada
[ ] reboot de validación
[ ] alertas comprobadas
[ ] documentación actualizada
[ ] valorar repetir DRT
```

------------------------------------------------------------------------

## 12.15 Procedimiento antes de una intervención

Antes de realizar un cambio relevante:

### 12.15.1 Definir el cambio

Anotar:

``` text
qué se va a modificar
por qué
qué servicios afecta
qué podría salir mal
cómo volver atrás
```

### 12.15.2 Comprobar el estado previo

``` bash
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh

cd /srv/docker/nextcloud
sudo docker compose ps
sudo docker compose exec -u www-data app php occ status
```

### 12.15.3 Comprobar el backup

Verificar que existe una copia reciente y que el último servicio de
backup terminó correctamente.

Para cambios importantes puede ejecutarse una copia manual antes de
continuar:

``` bash
sudo systemctl start nextcloud-backup.service
```

No iniciar la intervención hasta comprobar su resultado.

### 12.15.4 Registrar el estado

Cuando sea útil, guardar versiones:

``` bash
cat /etc/debian_version
docker --version
docker compose version
sudo docker compose images
sudo docker compose exec -u www-data app php occ status
```

Esto facilita diagnosticar regresiones.

------------------------------------------------------------------------

## 12.16 Procedimiento después de una intervención

### 12.16.1 Validación inmediata

Comprobar:

``` bash
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh
cd /srv/docker/nextcloud
sudo docker compose ps
sudo docker compose exec -u www-data app php occ status
```

### 12.16.2 Prueba funcional

Cuando el cambio afecte a Nextcloud:

``` text
[ ] login
[ ] navegación
[ ] lectura
[ ] subida
[ ] descarga
```

Cuando afecte al acceso externo:

``` text
[ ] acceso LAN
[ ] túnel
[ ] HTTPS externo
```

Cuando afecte al correo:

``` text
[ ] envío
[ ] recepción real
```

### 12.16.3 Reboot cuando corresponda

Es especialmente recomendable después de cambios en:

``` text
fstab
systemd
Docker
red
mounts
arranque
kernel
hardware
```

Después del reboot debe repetirse la comprobación esencial.

### 12.16.4 Nuevo backup

Si la intervención modifica significativamente la configuración o los
datos persistentes, generar una nueva copia después de validar el
resultado.

### 12.16.5 Documentar

Actualizar:

``` text
documentación técnica
CHANGELOG
configuraciones de ejemplo
scripts públicos
incidencias y lecciones aprendidas
```

Nunca trasladar secretos reales al repositorio público al documentar un
cambio.

------------------------------------------------------------------------

## 12.17 Checklist de mantenimiento

### Estado general

``` text
[ ] servidor accesible
[ ] SYSTEM sin errores evidentes
[ ] DATA montado
[ ] BACKUP montado o ausencia explicada
[ ] guard DATA = 0
[ ] espacio libre suficiente
```

### Servicios

``` text
[ ] Docker activo
[ ] PostgreSQL healthy
[ ] Redis healthy
[ ] app activa
[ ] cron activo
[ ] cloudflared activo cuando corresponde
[ ] Nextcloud fuera de mantenimiento
[ ] needsDbUpgrade = false
```

### Protección

``` text
[ ] backup reciente
[ ] timer de backup activo
[ ] Borg check programado
[ ] verificación profunda programada
[ ] SMART programado
[ ] recovery bundle actualizado
[ ] secretos de recuperación externos disponibles
```

### Funcionalidad

``` text
[ ] acceso local
[ ] acceso remoto
[ ] login
[ ] lectura de archivos
[ ] escritura cuando sea necesario probarla
[ ] correo y alertas cuando hayan sido modificados
```

### Después de cambios

``` text
[ ] logs revisados
[ ] reboot realizado si procede
[ ] servicios recuperados automáticamente
[ ] datos persistentes
[ ] nuevo backup generado si procede
[ ] documentación actualizada
```

------------------------------------------------------------------------

## Resumen operativo

La administración de `hpserver` debe evitar dos extremos: intervenir sin
comprobar el estado previo y realizar cambios innecesarios ante
cualquier advertencia.

El procedimiento normal es:

``` text
OBSERVAR
   ↓
IDENTIFICAR LA CAPA
   ↓
COMPROBAR BACKUP
   ↓
REALIZAR EL CAMBIO MÍNIMO NECESARIO
   ↓
VALIDAR
   ↓
COMPROBAR PERSISTENCIA
   ↓
GENERAR NUEVO PUNTO DE RECUPERACIÓN
   ↓
DOCUMENTAR
```

El objetivo del mantenimiento no es únicamente mantener Nextcloud
accesible, sino conservar un sistema predecible, verificable y
recuperable.
