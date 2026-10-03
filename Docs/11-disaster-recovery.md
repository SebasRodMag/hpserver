# 11. Disaster Recovery

## 11.1 Objetivo

El objetivo de este documento es definir el procedimiento de
recuperación del servidor `hpserver` ante una pérdida parcial o total de
la infraestructura.

El sistema aloja una instancia de Nextcloud utilizada como
almacenamiento privado y familiar. Aunque la disponibilidad del servicio
es importante, la prioridad principal ante una avería es preservar la
integridad de los datos.

Por este motivo, ante un incidente grave se priorizará:

1.  Evitar modificaciones innecesarias sobre los dispositivos afectados.
2.  Identificar correctamente el alcance de la avería.
3.  Preservar cualquier copia superviviente de los datos.
4.  Reconstruir la infraestructura desde un entorno conocido.
5.  Restaurar los datos desde una copia validada.
6.  Comprobar la integridad y funcionalidad del servicio antes de
    devolverlo a producción.
7.  Generar una nueva copia de seguridad una vez finalizada la
    recuperación.

El procedimiento descrito en este documento no depende de que el
hardware original siga siendo operativo.

La recuperación completa ha sido probada mediante ejercicios de Disaster
Recovery en un sistema independiente.

------------------------------------------------------------------------

## 11.2 Alcance y modelo de recuperación

La infraestructura se divide conceptualmente en tres elementos de
almacenamiento independientes:

-   `SYSTEM`
-   `DATA`
-   `BACKUP`

Esta separación permite tratar de forma diferente la pérdida de cada
componente.

``` text
                    ┌─────────────────────┐
                    │       SYSTEM        │
                    │ Debian + Docker     │
                    │ Nextcloud + DB      │
                    │ configuración       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │        DATA         │
                    │ archivos Nextcloud  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       BACKUP        │
                    │ repositorio Borg    │
                    └─────────────────────┘
```

El servidor puede reconstruirse sobre hardware diferente siempre que
sobrevivan una copia válida del repositorio Borg, la passphrase
necesaria para acceder al repositorio y las credenciales externas que no
forman parte deliberadamente del propio backup.

### 11.2.1 SYSTEM

`SYSTEM` contiene Debian, Docker Engine y Docker Compose, el volumen de
aplicación de Nextcloud, PostgreSQL, Redis, la configuración de Docker
Compose, scripts administrativos, unidades y temporizadores systemd y
configuración del sistema.

La pérdida de `SYSTEM` no implica necesariamente la pérdida de los
archivos de los usuarios, ya que éstos se encuentran almacenados en
`DATA`.

Los elementos críticos de configuración se incluyen en Borg mediante un
recovery bundle generado automáticamente antes de cada backup.

### 11.2.2 DATA

`DATA` contiene los archivos almacenados por los usuarios de Nextcloud.

Se monta en:

``` text
/srv/storage
```

y Nextcloud utiliza:

``` text
/srv/storage/nextcloud-data
```

La ausencia de DATA debe provocar un fallo controlado del servicio y
nunca el arranque silencioso contra un directorio incorrecto de SYSTEM.

Las protecciones incluyen `/etc/fstab`, dependencias systemd, el script
`check-nextcloud-data.sh`, validación del UUID y comprobación de
`.ncdata`.

### 11.2.3 BACKUP

`BACKUP` contiene el repositorio Borg:

``` text
/srv/backup/borg
```

La ausencia temporal de BACKUP no debe impedir el funcionamiento normal
de Nextcloud. Por ello su entrada en `/etc/fstab` utiliza:

``` text
nofail,x-systemd.device-timeout=10s
```

BACKUP debe considerarse independiente de DATA. Una sincronización o
segunda copia de los mismos archivos no sustituye a un backup con
histórico.

### 11.2.4 Información externa al servidor

Deben conservarse fuera del servidor:

-   passphrase del repositorio Borg;
-   copia exportada de la clave de recuperación Borg;
-   credenciales necesarias para servicios externos;
-   acceso administrativo a los proveedores utilizados;
-   documentación de recuperación.

El repositorio utiliza cifrado `repokey-blake2`. La passphrase y la
clave exportada no deben almacenarse únicamente dentro del mismo
repositorio que protegen.

------------------------------------------------------------------------

## 11.3 Principios de recuperación

### Preservar antes de reparar

No se debe formatear, reutilizar o modificar un dispositivo afectado
antes de determinar qué información contiene. Siempre que sea posible,
el disco sustituido se conservará sin modificaciones hasta completar y
validar la recuperación.

### No asumir la identidad de un disco por `/dev/sdX`

Los nombres `/dev/sda`, `/dev/sdb` y `/dev/sdc` no son identificadores
permanentes. Los sistemas de archivos se identificarán mediante UUID y,
cuando sea necesario, también mediante modelo, capacidad y número de
serie.

### Diagnosticar antes de restaurar

Una avería de SYSTEM, DATA y BACKUP requiere procedimientos diferentes.
No se iniciará automáticamente una restauración completa si el problema
puede resolverse recuperando sólo un componente.

### No modificar la única copia superviviente

Si BACKUP constituye la única copia disponible, se evitará trabajar
innecesariamente sobre el repositorio original. Siempre que sea posible
se realizará una copia independiente y se validará antes de una
reconstrucción compleja.

### No aplicar configuraciones antiguas de forma ciega

Antes de aplicar el recovery bundle deben revisarse especialmente UUID,
`/etc/fstab`, red, interfaces, scripts de validación, rutas, mounts
systemd y servicios externos.

### Recuperar primero, publicar después

Cloudflare Tunnel y otros accesos externos permanecerán desactivados
durante la recuperación.

``` text
restaurar
   ↓
validar almacenamiento
   ↓
validar base de datos
   ↓
validar Nextcloud
   ↓
validar lectura/escritura
   ↓
validar reinicio
   ↓
habilitar acceso externo
```

### La integridad tiene prioridad sobre el tiempo

No existe un RTO contractual que justifique omitir comprobaciones de
integridad.

------------------------------------------------------------------------

## 11.4 Material necesario para una recuperación

  Elemento                Función
  ----------------------- ------------------------------------------
  Repositorio Borg        Contiene datos y componentes respaldados
  Passphrase Borg         Permite acceder al repositorio cifrado
  Clave Borg exportada    Material adicional de recuperación
  Recovery bundle         Reconstrucción de configuración del host
  `compose.yml`           Infraestructura Nextcloud
  `.env`                  Variables y secretos de Docker Compose
  Volumen Nextcloud       Configuración y estado de la aplicación
  `nextcloud.dump`        Backup de PostgreSQL
  `postgres-roles.sql`    Roles globales de PostgreSQL
  DATA                    Archivos de usuarios
  Credenciales externas   Cloudflare, correo y otros servicios

### 11.4.1 Repositorio Borg

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg check --show-rc /srv/backup/borg
```

Un retorno `0` indica que la comprobación ejecutada por Borg ha
terminado correctamente. No debe interpretarse por sí sola como una
lectura exhaustiva de todos los bytes almacenados.

### 11.4.2 Passphrase y clave de recuperación

La recuperación no debe depender de poder acceder al SYSTEM averiado. La
información necesaria para acceder al repositorio debe disponer de una
copia externa.

### 11.4.3 Recovery bundle

Incluye, entre otros, `fstab`, hostname, red, DNS de Docker, repositorio
APT de Docker, unidades systemd, scripts administrativos, alertas,
`msmtp` e inventarios de paquetes, discos y mounts.

No contiene deliberadamente la passphrase ni la clave Borg exportada.

### 11.4.4 Base de datos PostgreSQL

La recuperación completa requiere:

``` text
nextcloud.dump
postgres-roles.sql
```

`nextcloud.dump` contiene la base Nextcloud y `postgres-roles.sql` los
roles globales necesarios para reproducir propietarios, permisos y ACL.
Esta necesidad fue confirmada durante las pruebas de Disaster Recovery
al detectar la dependencia del rol `oc_admin`.

------------------------------------------------------------------------

## 11.5 Identificación del almacenamiento

Antes de cualquier operación destructiva:

``` bash
lsblk -f
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
sudo blkid
```

Cuando sea necesario:

``` bash
sudo smartctl -i /dev/sdX
```

### 11.5.1 Identificación por función

Durante la recuperación se utilizarán los nombres conceptuales `SYSTEM`,
`DATA` y `BACKUP`, sin asumir un nombre `/dev/sdX`.

### 11.5.2 Hardware nuevo

Un nuevo filesystem tendrá normalmente un UUID diferente. Es un cambio
esperado; se actualizarán las configuraciones dependientes en lugar de
intentar conservar artificialmente el UUID anterior.

### 11.5.3 Reconciliación de UUID

Después de preparar DATA deben revisarse:

``` text
/etc/fstab
/usr/local/sbin/check-nextcloud-data.sh
```

Después de preparar BACKUP deben revisarse `/etc/fstab` y los scripts
que validen explícitamente su UUID.

Es recomendable buscar referencias antiguas:

``` bash
sudo grep -R "UUID_ANTIGUO" /etc /usr/local/sbin /srv/docker 2>/dev/null
```

------------------------------------------------------------------------

## 11.6 Diagnóstico inicial ante una avería

### Paso 1 --- Detener cambios innecesarios

Detener servicios afectados, evitar escrituras, no formatear
dispositivos y no ejecutar reparaciones destructivas sin conocer el
problema.

### Paso 2 --- Identificar dispositivos

``` bash
lsblk -f
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

Identificar SYSTEM, DATA y BACKUP.

### Paso 3 --- Comprobar mounts

``` bash
findmnt /
findmnt /srv/storage
findmnt /srv/backup
```

La mera existencia de `/srv/storage` no demuestra que DATA esté montado.

### Paso 4 --- Comprobar DATA

``` bash
sudo /usr/local/sbin/check-nextcloud-data.sh
```

Un fallo debe investigarse antes de arrancar Nextcloud.

### Paso 5 --- Comprobar servicios

``` bash
systemctl status docker.service --no-pager
systemctl status nextcloud.service --no-pager
sudo docker ps -a
```

### Paso 6 --- Consultar logs

``` bash
journalctl -b -p warning
journalctl -u nextcloud.service
journalctl -u docker.service
sudo docker logs NOMBRE_CONTENEDOR
```

### Paso 7 --- Clasificar el incidente

``` text
A. Pérdida o corrupción de DATA
B. Pérdida o corrupción de BACKUP
C. Pérdida de SYSTEM
D. Pérdida simultánea de SYSTEM y DATA
E. Pérdida lógica de archivos
```

> Preservar primero, diagnosticar después y restaurar únicamente cuando
> se conozca qué copia constituye la fuente de recuperación más segura.

Nunca debe destruirse el hardware o filesystem original simplemente
porque exista un backup aparentemente válido. La copia debe validarse
antes de descartar la última fuente potencial de información.

------------------------------------------------------------------------

## 11.7 Escenarios de recuperación

El procedimiento concreto depende del componente afectado. Antes de
restaurar se debe haber realizado el diagnóstico inicial descrito en la
sección anterior y haber identificado qué copia constituye la fuente de
recuperación más segura.

Los escenarios principales son:

``` text
A. Pérdida o corrupción de DATA
B. Pérdida o corrupción de BACKUP
C. Pérdida de SYSTEM
D. Pérdida simultánea de SYSTEM y DATA
E. Pérdida lógica de archivos
```

En todos los casos se aplican dos reglas:

1.  No destruir ni reutilizar el componente averiado hasta haber
    validado la recuperación.
2.  No devolver el servicio a producción hasta comprobar su integridad y
    funcionamiento.

### 11.7.1 Pérdida o corrupción de DATA

Este escenario se produce cuando el almacenamiento que contiene los
archivos de los usuarios deja de estar disponible, presenta errores o
debe ser sustituido, mientras SYSTEM y BACKUP continúan siendo
utilizables.

El objetivo es preparar un nuevo DATA y restaurar en él el directorio de
datos de Nextcloud desde Borg.

#### Detener Nextcloud

Antes de manipular DATA:

``` bash
sudo systemctl stop nextcloud.service
```

Comprobar:

``` bash
sudo docker ps
```

No debe mantenerse una instancia de Nextcloud escribiendo sobre DATA
durante la recuperación.

#### Identificar el dispositivo afectado

``` bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
findmnt /srv/storage
```

Si el dispositivo antiguo todavía es legible, no se debe formatear
inmediatamente. Debe conservarse hasta validar la restauración.

#### Preparar el nuevo DATA

Una vez identificado con certeza el nuevo dispositivo, se crea el
filesystem correspondiente y se obtiene su UUID.

Ejemplo conceptual:

``` bash
sudo mkfs.ext4 -L storage /dev/DEVICE_PARTITION
sudo blkid /dev/DEVICE_PARTITION
```

> `DEVICE_PARTITION` debe sustituirse por el dispositivo identificado
> durante la recuperación. Nunca se debe copiar literalmente un
> `/dev/sdX` de este documento.

Crear el punto de montaje si fuera necesario:

``` bash
sudo mkdir -p /srv/storage
```

Actualizar `/etc/fstab` con el nuevo UUID:

``` fstab
UUID=<UUID_DATA_NUEVO> /srv/storage ext4 defaults 0 2
```

DATA no utiliza `nofail`, ya que su ausencia debe impedir el arranque
normal del stack.

Validar:

``` bash
sudo mount -a
findmnt /srv/storage
```

#### Reconciliar las protecciones de DATA

El nuevo UUID debe actualizarse también en:

``` text
/usr/local/sbin/check-nextcloud-data.sh
```

Después debe buscarse cualquier referencia al UUID anterior:

``` bash
sudo grep -R "UUID_DATA_ANTIGUO" \
  /etc \
  /usr/local/sbin \
  /srv/docker 2>/dev/null
```

#### Restaurar DATA desde Borg

Seleccionar primero el archive adecuado:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

La restauración debe realizarse sobre el nuevo filesystem DATA.

El archive contiene la ruta:

``` text
srv/storage/nextcloud-data/
```

La extracción puede realizarse en un directorio temporal y trasladarse
después al mount definitivo, o directamente desde `/` cuando se haya
verificado cuidadosamente el destino.

Una opción controlada es:

``` bash
sudo mkdir -p /root/restore-data
cd /root/restore-data

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg extract \
  /srv/backup/borg::<ARCHIVE> \
  srv/storage/nextcloud-data
```

Tras la extracción, copiar el contenido al nuevo DATA preservando
metadatos:

``` bash
sudo rsync -aHAX \
  /root/restore-data/srv/storage/nextcloud-data/ \
  /srv/storage/nextcloud-data/
```

Comprobar propietario y permisos de acuerdo con la instalación
recuperada. En la configuración actual el directorio de datos pertenece
a `www-data:www-data`.

#### Validar DATA

Ejecutar:

``` bash
sudo /usr/local/sbin/check-nextcloud-data.sh
```

Debe finalizar correctamente.

Comprobar también:

``` bash
sudo head -n 1 /srv/storage/nextcloud-data/.ncdata
sudo du -sh /srv/storage/nextcloud-data
```

Una vez validado DATA se puede iniciar Nextcloud:

``` bash
sudo systemctl start nextcloud.service
```

La validación funcional se realizará según la sección 11.9.

------------------------------------------------------------------------

### 11.7.2 Pérdida o corrupción de BACKUP

Este escenario se produce cuando el dispositivo que contiene el
repositorio Borg deja de estar disponible, mientras SYSTEM y DATA
continúan funcionando correctamente.

La pérdida de BACKUP no implica por sí misma una pérdida inmediata del
servicio Nextcloud, pero elimina temporalmente la capacidad de
recuperación histórica.

#### Mantener Nextcloud operativo

BACKUP está configurado con:

``` text
nofail,x-systemd.device-timeout=10s
```

por lo que su ausencia no debe impedir el funcionamiento de Nextcloud.

No es necesario detener el servicio únicamente porque BACKUP haya
fallado.

#### Preservar el repositorio antiguo

Si el dispositivo todavía puede leerse, debe preservarse antes de
inicializar un nuevo repositorio.

No se ejecutarán operaciones de reparación destructiva sin disponer de
otra copia.

#### Preparar un nuevo BACKUP

Identificar el nuevo dispositivo:

``` bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

Crear el filesystem únicamente después de confirmar su identidad:

``` bash
sudo mkfs.ext4 -L BACKUP /dev/DEVICE_PARTITION
sudo blkid /dev/DEVICE_PARTITION
```

Actualizar `/etc/fstab`:

``` fstab
UUID=<UUID_BACKUP_NUEVO> /srv/backup ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2
```

Validar:

``` bash
sudo mount -a
findmnt /srv/backup
```

También deben actualizarse los scripts que comprueben explícitamente el
UUID de BACKUP.

#### Si existe otra copia del repositorio Borg

Si se dispone de una réplica válida del repositorio, ésta puede copiarse
al nuevo dispositivo.

Después debe validarse:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg check --show-rc /srv/backup/borg
```

y comprobar que los archives esperados son visibles:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

#### Si no existe otra copia del repositorio

Si BACKUP se ha perdido completamente pero SYSTEM y DATA sobreviven,
debe crearse un repositorio Borg nuevo y generar inmediatamente una
nueva copia completa.

En este caso se pierde el histórico almacenado exclusivamente en el
repositorio anterior.

La pérdida simultánea de DATA y de la única copia BACKUP constituye un
escenario no recuperable mediante esta infraestructura local y se trata
como una limitación en la sección 11.12.

------------------------------------------------------------------------

### 11.7.3 Pérdida de SYSTEM

Este escenario se produce cuando el disco del sistema operativo deja de
estar disponible, mientras DATA y BACKUP sobreviven.

El objetivo es reinstalar Debian y reconstruir los servicios utilizando
el recovery bundle y los componentes almacenados en Borg, conservando
DATA.

#### Preservar DATA y BACKUP

Antes de instalar el nuevo SYSTEM deben identificarse inequívocamente
los discos supervivientes.

``` bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

Durante la instalación del sistema operativo no deben seleccionarse DATA
ni BACKUP como destino de particionado.

#### Instalar Debian

Instalar una versión compatible de Debian y habilitar acceso
administrativo.

Después:

-   actualizar el sistema;
-   instalar `sudo` si fuera necesario;
-   configurar SSH;
-   instalar las dependencias necesarias;
-   instalar Docker desde su repositorio oficial;
-   instalar Borg, smartmontools y las herramientas administrativas
    utilizadas.

El recovery bundle contiene un inventario de paquetes que puede
utilizarse como referencia.

#### Montar DATA y BACKUP

Crear:

``` bash
sudo mkdir -p /srv/storage /srv/backup
```

Reconstruir `/etc/fstab` utilizando los UUID reales de los discos
supervivientes.

No debe copiarse el `fstab` histórico sin comprobar primero los
dispositivos actuales.

Validar:

``` bash
sudo findmnt --verify --verbose
sudo mount -a
findmnt /srv/storage
findmnt /srv/backup
```

#### Recuperar configuración y componentes de SYSTEM

Desde Borg se recuperarán, entre otros:

``` text
srv/docker/nextcloud/compose.yml
srv/docker/nextcloud/.env
srv/docker/nextcloud/volumes/nextcloud/
var/backups/nextcloud/nextcloud.dump
var/backups/nextcloud/postgres-roles.sql
var/backups/hpserver-config/
```

El recovery bundle se utilizará para reconstruir:

-   configuración del host;
-   scripts;
-   unidades systemd;
-   temporizadores;
-   Docker;
-   alertas;
-   red.

Los valores dependientes del nuevo SYSTEM deben reconciliarse antes de
instalarlos.

#### Reconstruir PostgreSQL

PostgreSQL debe inicializarse limpio y posteriormente restaurarse
utilizando:

``` text
postgres-roles.sql
nextcloud.dump
```

El procedimiento detallado se encuentra en la sección 11.8.

#### Validar antes de publicar

Cloudflare Tunnel permanecerá desactivado durante la reconstrucción.

El servicio se validará primero mediante acceso local y sólo después se
devolverá a producción.

------------------------------------------------------------------------

### 11.7.4 Pérdida simultánea de SYSTEM y DATA

Este es el escenario principal de Disaster Recovery y representa la
pérdida del sistema operativo, la configuración activa y los archivos de
usuario almacenados en DATA, mientras BACKUP sobrevive.

Este escenario fue reproducido durante DRT-04.

El proceso general es:

``` text
hardware nuevo o reparado
        ↓
Debian limpio
        ↓
preparar SYSTEM + DATA
        ↓
montar BACKUP
        ↓
validar Borg
        ↓
recuperar recovery bundle
        ↓
reconstruir host
        ↓
restaurar DATA
        ↓
restaurar Nextcloud
        ↓
reconstruir PostgreSQL + roles
        ↓
arrancar localmente
        ↓
validar lectura y escritura
        ↓
reboot completo
        ↓
validar persistencia
        ↓
habilitar servicios externos
```

Durante una recuperación de este tipo se recomienda, siempre que sea
posible, trabajar sobre una copia independiente del repositorio Borg y
preservar el BACKUP original.

Una vez validada la copia independiente, la recuperación debe poder
completarse sin utilizar el servidor de producción ni otras fuentes que
no sobrevivirían al escenario simulado.

El procedimiento completo se desarrolla en la sección 11.8.

------------------------------------------------------------------------

### 11.7.5 Pérdida lógica de archivos

Este escenario se produce cuando SYSTEM, DATA y BACKUP funcionan
correctamente, pero uno o varios archivos han sido:

-   eliminados accidentalmente;
-   sobrescritos;
-   modificados de forma no deseada;
-   eliminados por un cliente sincronizado;
-   afectados por un error lógico.

En este caso no debe reconstruirse el servidor completo.

#### Determinar qué debe recuperarse

Identificar:

-   usuario afectado;
-   ruta;
-   archivo o directorio;
-   momento aproximado anterior al incidente.

Consultar los archives disponibles:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

#### Restaurar en una ubicación temporal

No se recomienda sobrescribir directamente DATA durante la primera
extracción.

Ejemplo:

``` bash
sudo mkdir -p /root/restore-logical
cd /root/restore-logical

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg extract \
  /srv/backup/borg::<ARCHIVE> \
  <RUTA_DEL_ARCHIVO>
```

Comprobar el archivo recuperado antes de devolverlo a su ubicación
definitiva.

Cuando sea posible, comparar:

-   tamaño;
-   fecha;
-   tipo de archivo;
-   contenido;
-   hash.

#### Reintegrar el archivo

La forma de reintegración dependerá del alcance del incidente.

Debe evitarse introducir archivos directamente en el DATA de Nextcloud
sin tener en cuenta su base de datos y file cache.

Cuando se realice una recuperación administrativa sobre DATA, puede ser
necesario actualizar el índice de Nextcloud mediante las herramientas
`occ` apropiadas.

La recuperación debe limitarse al contenido afectado y validarse
posteriormente desde la propia interfaz de Nextcloud.

------------------------------------------------------------------------

### Resumen de decisión

``` text
¿Nextcloud funciona y sólo falta un archivo?
        │
        └── Sí → recuperación lógica

¿SYSTEM funciona pero DATA ha fallado?
        │
        └── Sí → sustituir/restaurar DATA

¿DATA funciona pero SYSTEM ha fallado?
        │
        └── Sí → reconstruir SYSTEM conservando DATA

¿SYSTEM y DATA se han perdido?
        │
        └── Sí → recuperación completa desde BACKUP

¿Sólo BACKUP ha fallado?
        │
        └── Sí → sustituir BACKUP y regenerar protección
```

En cualquier escenario, el hardware o filesystem sustituido no se
reutilizará hasta que la recuperación haya sido validada.

------------------------------------------------------------------------

## 11.7 Escenarios de recuperación

Una avería no implica necesariamente reconstruir el servidor completo.
El procedimiento debe elegirse en función del componente afectado y de
las copias que continúen disponibles.

Antes de ejecutar cualquiera de los procedimientos siguientes deben
cumplirse las reglas de la sección 11.6:

-   identificar correctamente SYSTEM, DATA y BACKUP;
-   preservar los dispositivos afectados;
-   determinar qué copia constituye la fuente de recuperación;
-   validar el repositorio Borg antes de depender de él;
-   evitar que una instancia reconstruida se publique externamente antes
    de ser validada.

Los comandos de esta sección son ejemplos operativos. Los nombres
`/dev/sdX`, UUID y nombres de archive deben sustituirse por los
identificados durante el incidente.

### 11.7.1 Escenario A --- Pérdida o corrupción de DATA

Este escenario se aplica cuando SYSTEM continúa operativo, pero el
almacenamiento que contiene los archivos de los usuarios ha fallado, ha
sido sustituido o debe reconstruirse.

La prioridad es impedir que Nextcloud escriba en el directorio
`/srv/storage` perteneciente al filesystem de SYSTEM mientras DATA no
está montado.

#### 1. Detener Nextcloud

``` bash
sudo systemctl stop nextcloud.service
```

Comprobar:

``` bash
sudo docker ps
```

No se debe continuar utilizando la instancia hasta disponer nuevamente
de un DATA válido.

#### 2. Identificar el nuevo dispositivo

``` bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

Confirmar físicamente qué dispositivo sustituirá a DATA antes de
particionarlo o formatearlo.

> **Advertencia:** los comandos de particionado y formateo destruyen el
> contenido del dispositivo seleccionado.

#### 3. Preparar el filesystem

El procedimiento concreto dependerá del estado del dispositivo. Para un
disco nuevo, un ejemplo sería:

``` bash
sudo mkfs.ext4 -L storage /dev/sdX1
```

Obtener su UUID:

``` bash
sudo blkid /dev/sdX1
```

Crear el punto de montaje si fuera necesario:

``` bash
sudo mkdir -p /srv/storage
```

#### 4. Actualizar `/etc/fstab`

DATA es un almacenamiento obligatorio para Nextcloud y no debe utilizar
`nofail`.

Ejemplo:

``` fstab
UUID=<UUID_DATA_NUEVO> /srv/storage ext4 defaults 0 2
```

Recargar systemd y montar:

``` bash
sudo systemctl daemon-reload
sudo mount /srv/storage
```

Validar:

``` bash
findmnt /srv/storage
lsblk -f
```

#### 5. Reconciliar las protecciones de DATA

Actualizar el UUID esperado en:

``` text
/usr/local/sbin/check-nextcloud-data.sh
```

Buscar referencias al UUID antiguo:

``` bash
sudo grep -R "UUID_DATA_ANTIGUO" \
  /etc \
  /usr/local/sbin \
  /srv/docker 2>/dev/null
```

No se debe continuar hasta haber revisado cualquier coincidencia
relevante.

#### 6. Restaurar DATA desde Borg

Seleccionar primero el archive:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

La restauración debe realizarse de forma que:

``` text
/srv/storage/nextcloud-data
```

vuelva a contener el árbol almacenado en el archive bajo:

``` text
srv/storage/nextcloud-data
```

Una forma segura es extraer inicialmente a un directorio de staging y
trasladar después el árbol validado a su destino.

Ejemplo:

``` bash
sudo mkdir -p /root/restore-data
cd /root/restore-data

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg extract --show-rc \
  /srv/backup/borg::<ARCHIVE> \
  srv/storage/nextcloud-data
```

Comprobar el código de retorno:

``` bash
echo $?
```

Debe ser `0`.

Tras validar la extracción, instalar el árbol restaurado en
`/srv/storage` conservando propietarios, permisos y estructura.

#### 7. Validar DATA

Comprobar como mínimo:

``` bash
sudo /usr/local/sbin/check-nextcloud-data.sh
```

y:

``` bash
sudo head -n 1 /srv/storage/nextcloud-data/.ncdata
sudo du -sh /srv/storage/nextcloud-data
```

También es recomendable comparar uno o varios hashes de archivos
conocidos cuando exista una referencia fiable.

#### 8. Arrancar y validar Nextcloud

``` bash
sudo systemctl start nextcloud.service
```

Después:

``` bash
cd /srv/docker/nextcloud
sudo docker compose exec -u www-data app php occ status
```

Debe verificarse la navegación y apertura de archivos desde Nextcloud
antes de considerar recuperado DATA.

------------------------------------------------------------------------

### 11.7.2 Escenario B --- Pérdida o corrupción de BACKUP

Este escenario se aplica cuando SYSTEM y DATA continúan funcionando,
pero el dispositivo o repositorio de backup ha dejado de estar
disponible.

La pérdida de BACKUP no debe detener Nextcloud. Sin embargo, mientras no
exista una nueva copia válida, los datos quedan sin la protección
histórica proporcionada por Borg.

#### 1. Confirmar que el problema está limitado a BACKUP

``` bash
findmnt /srv/storage
sudo /usr/local/sbin/check-nextcloud-data.sh
```

Comprobar también Nextcloud:

``` bash
cd /srv/docker/nextcloud
sudo docker compose exec -u www-data app php occ status
```

#### 2. Identificar o sustituir el dispositivo BACKUP

``` bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

Si se utiliza un nuevo dispositivo, preparar un filesystem ext4 y
obtener su nuevo UUID.

#### 3. Actualizar `/etc/fstab`

BACKUP puede utilizar:

``` fstab
UUID=<UUID_BACKUP_NUEVO> /srv/backup ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2
```

Validar:

``` bash
sudo findmnt --verify --verbose
sudo mount /srv/backup
findmnt /srv/backup
```

#### 4. Elegir el tipo de recuperación

Existen dos posibilidades.

**Existe otra copia del repositorio Borg**

Copiar el repositorio superviviente al nuevo BACKUP y validarlo antes de
reanudar las tareas automáticas.

**No existe otra copia del repositorio Borg**

Se ha perdido el histórico anterior. Debe inicializarse un repositorio
nuevo y generar inmediatamente una copia completa del estado actual.

En este segundo caso, DATA no se considera recuperado desde BACKUP:
constituye la fuente primaria superviviente a partir de la que se crea
una nueva cadena de copias.

#### 5. Validar el nuevo repositorio

Después de copiar o crear el repositorio:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

y cuando corresponda:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg check --show-rc /srv/backup/borg
```

Finalmente debe ejecutarse y comprobarse una nueva tarea de backup.

------------------------------------------------------------------------

### 11.7.3 Escenario C --- Pérdida de SYSTEM

Este escenario se aplica cuando el disco del sistema se pierde o debe
reinstalarse, pero DATA y BACKUP permanecen disponibles.

La ventaja de este escenario es que los archivos de usuario continúan
físicamente en DATA. No obstante, Nextcloud no debe ponerse en servicio
hasta reconstruir de forma coherente la aplicación y la base de datos.

#### 1. Preservar DATA y BACKUP

Durante la instalación del nuevo SYSTEM debe extremarse la
identificación de discos para evitar seleccionar DATA o BACKUP como
destino de instalación.

Siempre que sea posible, pueden desconectarse temporalmente los discos
que no sean necesarios durante la instalación inicial.

#### 2. Instalar Debian

Realizar una instalación mínima del sistema y aplicar las
actualizaciones necesarias.

No se deben copiar directamente configuraciones del host antiguo antes
de identificar las interfaces, UUID y características del nuevo entorno.

#### 3. Montar DATA y BACKUP

Identificar:

``` bash
lsblk -f
```

Crear los puntos de montaje:

``` bash
sudo mkdir -p /srv/storage /srv/backup
```

Construir `/etc/fstab` utilizando los UUID reales del nuevo entorno y
validar:

``` bash
sudo findmnt --verify --verbose
```

#### 4. Instalar Borg y acceder al backup

Instalar BorgBackup y proporcionar de forma segura la passphrase
externa.

Validar:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

#### 5. Recuperar la configuración de SYSTEM

Extraer del archive seleccionado:

-   recovery bundle;
-   `compose.yml`;
-   `.env`;
-   volumen de Nextcloud;
-   `nextcloud.dump`;
-   `postgres-roles.sql`.

El recovery bundle debe revisarse antes de instalar sus archivos.

Especialmente deben reconciliarse:

-   `/etc/fstab`;
-   configuración de red;
-   UUID de los scripts de validación;
-   unidades systemd relacionadas con almacenamiento;
-   configuración de servicios externos.

#### 6. Reconstruir Docker y Nextcloud

Instalar Docker desde su repositorio oficial y restaurar la estructura:

``` text
/srv/docker/nextcloud/
├── compose.yml
├── .env
└── volumes/
    ├── nextcloud/
    └── postgres/
```

El volumen PostgreSQL no debe restaurarse copiando un directorio de
datos activo. Se reconstruirá una instancia PostgreSQL limpia y se
restaurará mediante los dumps.

#### 7. Restaurar PostgreSQL

La recuperación requiere:

``` text
postgres-roles.sql
nextcloud.dump
```

Primero se inicializa PostgreSQL limpio.

La imagen crea automáticamente el rol indicado por `POSTGRES_USER`. Por
tanto, `postgres-roles.sql` no debe ejecutarse ciegamente si contiene un
`CREATE ROLE` para ese mismo usuario.

Debe reconciliarse el rol existente y crear los roles adicionales antes
de ejecutar `pg_restore`.

En la instalación validada, esto incluye el rol `oc_admin`.

Una vez disponibles los roles:

``` bash
sudo docker compose exec -T db sh -c \
  'pg_restore --exit-on-error -U "$POSTGRES_USER" -d "$POSTGRES_DB"' \
  < /ruta/segura/nextcloud.dump
```

El código de retorno debe ser `0`.

#### 8. Validar antes de publicar

Levantar inicialmente sólo los servicios internos necesarios:

``` bash
sudo docker compose up -d db redis app cron
```

Cloudflare Tunnel debe permanecer desactivado.

Validar:

``` bash
sudo docker compose exec -u www-data app php occ status
```

Si el backup fue capturado durante el modo mantenimiento puede aparecer:

``` text
maintenance: true
```

El modo mantenimiento es un estado persistente, no un proceso que deba
esperarse. Una vez validada la restauración puede desactivarse:

``` bash
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --off
```

Después se realizarán las pruebas funcionales indicadas en la sección
11.9.

------------------------------------------------------------------------

### 11.7.4 Escenario D --- Pérdida simultánea de SYSTEM y DATA

Este es el escenario principal de Disaster Recovery y representa la
pérdida del servidor operativo manteniendo únicamente BACKUP y la
información externa de recuperación.

Este procedimiento ha sido validado mediante DRT-04 sobre una
instalación limpia e independiente.

El flujo general es:

``` text
nuevo SYSTEM
     ↓
Debian limpio
     ↓
nuevo DATA
     ↓
acceso a BACKUP
     ↓
validación Borg
     ↓
recovery bundle
     ↓
restauración DATA
     ↓
restauración Nextcloud
     ↓
PostgreSQL + roles globales
     ↓
systemd y protecciones
     ↓
validación local
     ↓
reboot completo
     ↓
acceso externo
```

#### 1. Preservar el hardware original

No reutilizar ni formatear SYSTEM o DATA antiguos hasta completar la
recuperación.

Si BACKUP constituye la única copia superviviente, se recomienda
trabajar sobre una copia independiente del repositorio siempre que sea
posible.

#### 2. Instalar un SYSTEM limpio

Instalar Debian y las herramientas mínimas necesarias.

Crear un nuevo DATA y, si procede, conectar o copiar BACKUP.

Identificar todos los dispositivos mediante:

``` bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

#### 3. Validar BACKUP antes de depender de él

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg check --show-rc /srv/backup/borg
```

Seleccionar un archive válido y confirmar que contiene como mínimo:

``` text
srv/storage/nextcloud-data/.ncdata
srv/docker/nextcloud/compose.yml
srv/docker/nextcloud/.env
var/backups/nextcloud/nextcloud.dump
var/backups/nextcloud/postgres-roles.sql
```

#### 4. Restaurar DATA y los componentes de SYSTEM

Restaurar:

-   `/srv/storage/nextcloud-data`;
-   volumen de aplicación de Nextcloud;
-   `compose.yml`;
-   `.env`;
-   `nextcloud.dump`;
-   `postgres-roles.sql`;
-   recovery bundle.

No mezclar sin necesidad componentes procedentes de archives diferentes.
Para una reconstrucción completa debe preferirse un punto de
recuperación coherente.

#### 5. Reconciliar el nuevo hardware

Actualizar:

-   UUID de DATA;
-   UUID de BACKUP;
-   `/etc/fstab`;
-   scripts de guard;
-   red;
-   interfaces;
-   unidades systemd;
-   cualquier otra referencia dependiente del host.

Buscar referencias antiguas antes de continuar.

#### 6. Reconstruir PostgreSQL

Inicializar PostgreSQL 18 limpio.

Restaurar primero los roles globales, reconciliando el `POSTGRES_USER`
que la imagen haya creado automáticamente.

Después restaurar `nextcloud.dump` y exigir `pg_restore` con código de
retorno `0`.

Validar:

-   tablas;
-   datos;
-   propietarios;
-   ACL;
-   roles globales.

#### 7. Arrancar Nextcloud sin acceso externo

Levantar:

``` bash
sudo docker compose up -d db redis app cron
```

No levantar `cloudflared`.

Comprobar:

``` bash
sudo docker compose exec -u www-data app php occ status
```

Desactivar el modo mantenimiento cuando corresponda.

#### 8. Realizar una prueba funcional

Desde acceso local:

-   iniciar sesión con una cuenta recuperada;
-   navegar por archivos existentes;
-   abrir o descargar un archivo conocido;
-   crear un directorio de prueba;
-   subir un archivo;
-   descargarlo de nuevo;
-   comprobar su contenido.

Esta prueba confirma que la instancia no sólo arranca, sino que puede
leer y escribir correctamente sobre DATA y mantener la coherencia con
PostgreSQL.

#### 9. Reconstruir el arranque seguro

Restaurar y reconciliar:

``` text
nextcloud.service
docker.service.d/override.conf
check-nextcloud-data.sh
```

La dependencia de DATA debe mantenerse antes del arranque de Docker y
Nextcloud.

Durante una recuperación paralela a un servidor de producción, el
servicio debe adaptarse temporalmente para no arrancar `cloudflared`.

#### 10. Realizar un reboot completo

Después de validar `fstab`, systemd y los guards:

``` bash
sudo reboot
```

No levantar manualmente los servicios después del arranque.

Comprobar que:

-   DATA está montado;
-   BACKUP está montado o su ausencia es tolerada según diseño;
-   Docker ha arrancado;
-   PostgreSQL está healthy;
-   Redis está healthy;
-   Nextcloud está operativo;
-   cron está operativo;
-   los archivos creados antes del reboot persisten;
-   Cloudflare continúa desactivado durante el ensayo.

Sólo después de superar esta prueba debe considerarse reconstruido el
servidor.

------------------------------------------------------------------------

### 11.7.5 Escenario E --- Pérdida lógica de archivos

Este escenario cubre eliminaciones accidentales, sobrescrituras,
corrupción lógica o necesidad de recuperar una versión anterior cuando
el hardware continúa funcionando.

Antes de utilizar Borg deben revisarse primero los mecanismos propios de
Nextcloud cuando sean suficientes, como papelera o versiones.

Si es necesario recurrir al backup, no debe sobrescribirse directamente
DATA sin conocer el alcance de la restauración.

#### 1. Identificar el punto de recuperación

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

Localizar el archivo o directorio en el archive adecuado.

#### 2. Restaurar a staging

Crear un directorio temporal separado:

``` bash
sudo mkdir -p /root/restore-logical
cd /root/restore-logical
```

Extraer únicamente el contenido necesario:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg extract \
  /srv/backup/borg::<ARCHIVE> \
  <RUTA_DEL_ARCHIVO>
```

#### 3. Validar antes de devolver el archivo

Comprobar:

-   nombre;
-   tamaño;
-   fecha;
-   contenido;
-   hash, cuando exista una referencia fiable;
-   propietario y permisos.

La restauración a staging permite comprobar el archivo sin alterar
inmediatamente el estado actual de Nextcloud.

#### 4. Reintegrar de forma controlada

La forma de devolver un archivo dependerá de la naturaleza de la
pérdida.

No debe copiarse contenido arbitrariamente al data directory de
Nextcloud sin tener en cuenta su índice de archivos.

Cuando se modifique el filesystem fuera de Nextcloud puede ser necesario
utilizar las herramientas administrativas de Nextcloud para reconciliar
el estado.

La recuperación lógica debe limitarse al menor conjunto de datos posible
y validarse desde la interfaz de usuario una vez finalizada.

------------------------------------------------------------------------

### Resultado esperado de los escenarios

La recuperación se considera completada únicamente cuando el servicio ha
sido validado, no simplemente cuando los comandos de restauración han
terminado.

En todos los escenarios aplicables debe comprobarse:

``` text
almacenamiento correcto
        ↓
integridad de la restauración
        ↓
servicios operativos
        ↓
Nextcloud coherente
        ↓
lectura de datos
        ↓
escritura de datos
        ↓
persistencia
```

Los procedimientos detallados para una reconstrucción completa, las
comprobaciones posteriores y la generación de la primera copia tras una
recuperación se desarrollan en las secciones siguientes.

------------------------------------------------------------------------

## 11.8 Reconstrucción completa del servidor

Esta sección describe el procedimiento lineal de reconstrucción cuando
SYSTEM debe instalarse desde cero y la recuperación depende del
repositorio BACKUP.

El procedimiento está basado en la reconstrucción ejecutada durante
DRT-04. Los identificadores de dispositivos, UUID, direcciones IP y
nombres de archive mostrados son marcadores y deben sustituirse por los
valores del entorno de recuperación.

> **Objetivo:** partir de un Debian limpio y terminar con una instancia
> Nextcloud funcional, validada localmente y preparada para volver a
> producción.

### 11.8.1 Preparar el entorno de recuperación

Antes de instalar o modificar discos:

1.  preservar el hardware original;
2.  identificar qué copia de BACKUP se utilizará;
3.  disponer externamente de la passphrase Borg;
4.  disponer de las credenciales necesarias para los servicios externos;
5.  mantener Cloudflare Tunnel fuera del procedimiento hasta completar
    las pruebas locales.

Si el servidor original continúa funcionando parcialmente, no debe
utilizarse como dependencia oculta del proceso una vez preparada una
copia independiente de recuperación.

### 11.8.2 Instalar Debian

Realizar una instalación mínima de Debian sobre el nuevo SYSTEM.

Después del primer arranque:

``` bash
sudo apt update
sudo apt full-upgrade
```

Instalar las herramientas básicas necesarias para la recuperación:

``` bash
sudo apt install borgbackup smartmontools
```

La lista completa de paquetes del servidor anterior puede consultarse
posteriormente en el recovery bundle.

No aplicar todavía de forma automática la configuración de red, `fstab`
o unidades systemd del host anterior.

### 11.8.3 Identificar SYSTEM, DATA y BACKUP

Ejecutar:

``` bash
lsblk -f
```

y:

``` bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

Registrar qué dispositivo desempeñará cada función:

``` text
SYSTEM  → <dispositivo>
DATA    → <dispositivo>
BACKUP  → <dispositivo>
```

No continuar hasta que la identificación sea inequívoca.

### 11.8.4 Preparar DATA

Si DATA también se ha perdido, preparar un nuevo filesystem.

Ejemplo para una partición ya creada:

``` bash
sudo mkfs.ext4 -L storage /dev/sdX1
```

Obtener el UUID:

``` bash
sudo blkid /dev/sdX1
```

Crear el punto de montaje:

``` bash
sudo mkdir -p /srv/storage
```

Añadir a `/etc/fstab`:

``` fstab
UUID=<UUID_DATA> /srv/storage ext4 defaults 0 2
```

DATA no debe utilizar `nofail`.

Validar:

``` bash
sudo findmnt --verify --verbose
sudo mount /srv/storage
findmnt /srv/storage
```

### 11.8.5 Preparar BACKUP

Si se conecta directamente el dispositivo BACKUP superviviente,
identificar su UUID sin modificar su filesystem.

Crear:

``` bash
sudo mkdir -p /srv/backup
```

La entrada esperada en `/etc/fstab` es:

``` fstab
UUID=<UUID_BACKUP> /srv/backup ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2
```

Validar:

``` bash
sudo findmnt --verify --verbose
sudo mount /srv/backup
findmnt /srv/backup
```

Si se trabaja sobre una copia independiente del repositorio, validar
también el filesystem y el repositorio copiado antes de utilizarlo.

### 11.8.6 Configurar acceso a Borg

Crear un directorio protegido para la configuración:

``` bash
sudo install -d -o root -g root -m 700 /root/.config/borg
```

Instalar de forma segura la passphrase recuperada externamente:

``` text
/root/.config/borg/passphrase
```

con permisos:

``` bash
sudo chmod 600 /root/.config/borg/passphrase
```

No mostrar la passphrase en la terminal ni almacenarla en el repositorio
de documentación.

Comprobar el acceso:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

Seleccionar el archive que se utilizará como punto de recuperación.

### 11.8.7 Validar el archive seleccionado

Antes de restaurar:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg check --show-rc /srv/backup/borg
```

Comprobar el código:

``` bash
echo $?
```

Debe ser `0`.

Comprobar que el archive contiene los componentes críticos:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg::<ARCHIVE> \
  srv/storage/nextcloud-data/.ncdata

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg::<ARCHIVE> \
  srv/docker/nextcloud/compose.yml

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg::<ARCHIVE> \
  srv/docker/nextcloud/.env

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg::<ARCHIVE> \
  var/backups/nextcloud/nextcloud.dump

sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg::<ARCHIVE> \
  var/backups/nextcloud/postgres-roles.sql
```

También debe localizarse el recovery bundle:

``` text
var/backups/hpserver-config/
```

### 11.8.8 Extraer primero a staging

Para una recuperación completa es recomendable utilizar un directorio de
staging.

Ejemplo:

``` bash
sudo mkdir -p /root/dr-restore
cd /root/dr-restore
```

Extraer los componentes necesarios:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg extract --show-rc \
  /srv/backup/borg::<ARCHIVE> \
  srv/storage/nextcloud-data \
  srv/docker/nextcloud \
  var/backups/nextcloud/nextcloud.dump \
  var/backups/nextcloud/postgres-roles.sql \
  var/backups/hpserver-config
```

Comprobar:

``` bash
echo $?
```

Debe ser `0`.

El staging permite inspeccionar los archivos antes de instalarlos en sus
ubicaciones definitivas.

### 11.8.9 Revisar el recovery bundle

Inspeccionar:

``` bash
sudo find /root/dr-restore/var/backups/hpserver-config \
  -type f | sort
```

No copiar todo el bundle de forma automática.

Revisar especialmente:

``` text
/etc/fstab
/etc/dhcpcd.conf
/etc/docker/daemon.json
/etc/systemd/system/
/usr/local/sbin/
```

Los UUID y parámetros dependientes del hardware antiguo deben
sustituirse por los del nuevo servidor.

### 11.8.10 Restaurar DATA

Instalar el árbol recuperado en el nuevo filesystem DATA conservando
metadatos.

El resultado final debe ser:

``` text
/srv/storage/nextcloud-data
```

Comprobar:

``` bash
sudo ls -ld /srv/storage/nextcloud-data
sudo head -n 1 /srv/storage/nextcloud-data/.ncdata
sudo du -sh /srv/storage/nextcloud-data
```

No arrancar Nextcloud todavía.

### 11.8.11 Instalar Docker

Instalar Docker Engine utilizando el repositorio oficial correspondiente
a Debian.

El recovery bundle contiene como referencia:

``` text
/etc/apt/sources.list.d/docker.sources
/etc/apt/keyrings/docker.asc
```

Estos archivos pueden utilizarse para reproducir la configuración, pero
debe comprobarse que siguen siendo adecuados para la versión de Debian
instalada.

Instalar Docker Engine, CLI, containerd, Buildx y Docker Compose Plugin.

Validar:

``` bash
sudo docker version
sudo docker compose version
```

### 11.8.12 Restaurar la estructura de Nextcloud

Crear:

``` bash
sudo mkdir -p /srv/docker/nextcloud
```

Instalar desde staging:

``` text
compose.yml
.env
volumes/nextcloud/
```

El archivo `.env` debe permanecer protegido:

``` bash
sudo chmod 600 /srv/docker/nextcloud/.env
```

La estructura esperada es:

``` text
/srv/docker/nextcloud/
├── compose.yml
├── .env
└── volumes/
    ├── nextcloud/
    └── postgres/
```

El directorio `volumes/postgres` debe comenzar vacío para una
reconstrucción mediante dump.

Validar Compose:

``` bash
cd /srv/docker/nextcloud
sudo docker compose config --quiet
echo $?
```

Debe devolver `0`.

### 11.8.13 Mantener Cloudflare Tunnel desactivado

El `.env` recuperado puede contener un token válido del túnel real.

Durante la recuperación no debe ejecutarse:

``` bash
sudo docker compose up -d
```

si ello implica levantar `cloudflared`.

Los servicios se arrancarán explícitamente.

Durante la fase inicial:

``` bash
sudo docker compose up -d db
```

Cloudflare sólo se habilitará una vez validado completamente el
servidor.

### 11.8.14 Inicializar PostgreSQL limpio

Con `volumes/postgres` vacío:

``` bash
cd /srv/docker/nextcloud
sudo docker compose up -d db
```

Esperar a que el contenedor esté healthy:

``` bash
sudo docker compose ps db
```

Comprobar:

``` bash
sudo docker compose exec -T db sh -c \
  'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

La inicialización crea automáticamente el usuario definido mediante
`POSTGRES_USER` y la base indicada mediante `POSTGRES_DB`.

### 11.8.15 Validar los artefactos PostgreSQL

Antes de importar:

``` bash
sudo docker run --rm \
  -v /ruta/staging/postgresql:/backup:ro \
  postgres:18-alpine \
  pg_restore -l /backup/nextcloud.dump >/dev/null

echo $?
```

Debe devolver `0`.

Para comprobar los roles sin mostrar hashes de contraseña:

``` bash
sudo grep '^CREATE ROLE ' \
  /ruta/staging/postgresql/postgres-roles.sql
```

No utilizar `cat` sobre `postgres-roles.sql`, ya que puede contener
hashes de credenciales.

### 11.8.16 Restaurar los roles globales

La imagen PostgreSQL ya habrá creado el rol correspondiente a
`POSTGRES_USER`.

Si `postgres-roles.sql` contiene:

``` sql
CREATE ROLE nextcloud;
```

ejecutar el fichero completo provocaría un conflicto.

Debe generarse una copia de restauración que elimine únicamente ese
`CREATE ROLE`, conservando los `ALTER ROLE` y la creación de los demás
roles.

Ejemplo:

``` bash
sudo sed '/^CREATE ROLE nextcloud;$/d' \
  /ruta/staging/postgresql/postgres-roles.sql \
  > /ruta/staging/postgresql/roles-restore.sql

sudo chmod 600 /ruta/staging/postgresql/roles-restore.sql
```

Comprobar únicamente los `CREATE ROLE` restantes:

``` bash
sudo grep '^CREATE ROLE ' \
  /ruta/staging/postgresql/roles-restore.sql
```

Aplicar:

``` bash
sudo docker compose exec -T db sh -c \
  'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d postgres' \
  < /ruta/staging/postgresql/roles-restore.sql
```

Comprobar:

``` bash
echo $?
```

Debe devolver `0`.

Verificar los roles y sus atributos sin mostrar contraseñas.

### 11.8.17 Restaurar la base de datos

Antes de restaurar, confirmar que la base destino no contiene tablas de
una restauración anterior.

Después:

``` bash
sudo docker compose exec -T db sh -c \
  'pg_restore \
     --exit-on-error \
     -U "$POSTGRES_USER" \
     -d "$POSTGRES_DB"' \
  < /ruta/staging/postgresql/nextcloud.dump
```

Comprobar:

``` bash
echo $?
```

Debe devolver `0`.

No continuar ante un error de `pg_restore`.

### 11.8.18 Validar PostgreSQL

Comprobar la presencia de tablas esenciales:

``` text
oc_users
oc_filecache
oc_appconfig
oc_migrations
```

Comprobar también que contienen datos razonables.

Verificar propietarios:

``` sql
SELECT tableowner, count(*) AS tablas
FROM pg_tables
WHERE schemaname = 'public'
GROUP BY tableowner
ORDER BY tableowner;
```

Y el schema `public`:

``` sql
SELECT nspname,
       pg_get_userbyid(nspowner) AS owner,
       nspacl
FROM pg_namespace
WHERE nspname = 'public';
```

La restauración debe reproducir los propietarios y ACL del entorno
respaldado.

### 11.8.19 Arrancar Redis y Nextcloud

Con PostgreSQL validado:

``` bash
sudo docker compose up -d redis
sudo docker compose up -d app
```

Comprobar:

``` bash
sudo docker compose ps
```

Después:

``` bash
sudo docker compose exec -u www-data app php occ status
```

Se espera:

``` text
installed: true
needsDbUpgrade: false
```

El estado puede mostrar:

``` text
maintenance: true
```

porque el backup se captura durante el modo mantenimiento.

Una vez comprobada la restauración:

``` bash
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --off
```

Volver a comprobar:

``` bash
sudo docker compose exec -u www-data app php occ status
```

### 11.8.20 Validar DATA desde el contenedor

Comprobar la ruta configurada:

``` bash
sudo docker compose exec -u www-data app \
  php occ config:system:get datadirectory
```

Debe apuntar al directorio montado dentro del contenedor.

Comprobar `.ncdata`:

``` bash
sudo docker compose exec app sh -c \
  'head -n 1 /var/www/html/data/.ncdata && df -h /var/www/html/data'
```

Esta comprobación ayuda a detectar un bind mount apuntando
accidentalmente al filesystem equivocado.

### 11.8.21 Arrancar cron

Una vez validada la aplicación:

``` bash
sudo docker compose up -d cron
```

En este punto deben estar activos:

``` text
db
redis
app
cron
```

y `cloudflared` debe continuar desactivado.

### 11.8.22 Realizar validación funcional local

Acceder a la instancia mediante la dirección local del servidor.

Si la nueva dirección no figura en `trusted_domains`, puede añadirse
temporalmente durante la recuperación.

Comprobar:

1.  inicio de sesión con una cuenta recuperada;
2.  navegación por carpetas existentes;
3.  apertura o descarga de un archivo conocido;
4.  creación de un directorio de prueba;
5.  subida de un archivo;
6.  descarga posterior;
7.  comprobación de su contenido.

No considerar la restauración completa únicamente porque `occ status`
sea correcto.

### 11.8.23 Instalar los scripts recuperados

Desde el recovery bundle, revisar e instalar los scripts necesarios en:

``` text
/usr/local/sbin/
```

Entre ellos:

``` text
check-nextcloud-data.sh
backup-nextcloud.sh
check-borg.sh
check-borg-data.sh
check-smart-disks.sh
send-systemd-alert.sh
generate-hpserver-recovery.sh
```

Antes de instalarlos deben reconciliarse los UUID y cualquier parámetro
específico del host.

Aplicar permisos restrictivos según corresponda.

### 11.8.24 Restaurar el arranque seguro

Instalar el override de Docker:

``` text
/etc/systemd/system/docker.service.d/override.conf
```

con la dependencia de DATA:

``` ini
[Unit]
RequiresMountsFor=/srv/storage
After=srv-storage.mount
```

Instalar también:

``` text
/etc/systemd/system/nextcloud.service
```

El servicio debe conservar:

``` ini
RequiresMountsFor=/srv/storage
ExecStartPre=/usr/local/sbin/check-nextcloud-data.sh
```

Durante un DRT o una recuperación paralela a producción, el `ExecStart`
debe excluir explícitamente `cloudflared`.

Ejemplo temporal:

``` ini
ExecStart=/usr/bin/docker compose up -d db redis app cron
```

En producción, una vez validado el entorno y cuando corresponda
habilitar el túnel, podrá utilizarse la configuración operativa normal.

### 11.8.25 Validar systemd y `fstab`

Ejecutar:

``` bash
sudo findmnt --verify --verbose
```

Comprobar:

``` bash
sudo /usr/local/sbin/check-nextcloud-data.sh
echo $?
```

Debe devolver `0`.

Después:

``` bash
sudo systemctl daemon-reload
sudo systemd-analyze verify docker.service nextcloud.service
```

Habilitar los servicios correspondientes:

``` bash
sudo systemctl enable docker.service
sudo systemctl enable nextcloud.service
```

### 11.8.26 Reboot de validación

Antes del reinicio, crear o identificar un archivo de prueba cuya
persistencia pueda comprobarse.

Reiniciar:

``` bash
sudo reboot
```

Después del arranque no levantar servicios manualmente.

Comprobar:

``` bash
findmnt /srv/storage
findmnt /srv/backup
```

Después:

``` bash
systemctl status nextcloud.service --no-pager
sudo docker ps
```

Verificar:

-   PostgreSQL healthy;
-   Redis healthy;
-   app activa;
-   cron activo;
-   DATA correcto;
-   archivo de prueba persistente;
-   acceso local funcional;
-   `cloudflared` todavía desactivado durante la validación.

### 11.8.27 Restaurar servicios auxiliares

Una vez superado el reboot, instalar y revisar desde el recovery bundle:

-   timers de backup;
-   comprobaciones Borg;
-   comprobaciones SMART;
-   alertas systemd;
-   `msmtp`;
-   configuración de Docker;
-   configuración de energía;
-   resto de unidades administrativas.

No habilitar automáticamente un timer hasta comprobar que sus rutas,
UUID, secretos y dependencias corresponden al nuevo servidor.

### 11.8.28 Habilitar acceso externo

Cloudflare Tunnel es una de las últimas piezas que deben habilitarse.

Antes:

-   confirmar funcionamiento local;
-   confirmar DATA;
-   confirmar PostgreSQL;
-   confirmar reboot;
-   confirmar configuración de `trusted_domains`;
-   revisar el token recuperado;
-   verificar que no existe otra instancia de recuperación compitiendo
    por el mismo túnel.

Sólo entonces iniciar `cloudflared` mediante el mecanismo operativo
elegido.

Después debe comprobarse el acceso HTTPS externo.

### 11.8.29 Estado final de la reconstrucción

La reconstrucción completa puede considerarse técnicamente satisfactoria
cuando se cumple:

``` text
Debian operativo
        ↓
SYSTEM reconstruido
        ↓
DATA montado y protegido
        ↓
BACKUP accesible
        ↓
PostgreSQL + roles restaurados
        ↓
Nextcloud operativo
        ↓
lectura y escritura verificadas
        ↓
reboot superado
        ↓
servicios auxiliares restaurados
        ↓
acceso externo habilitado
```

La recuperación todavía no debe darse por cerrada hasta generar y
validar una nueva copia de seguridad del servidor reconstruido. Este
paso se desarrolla en la sección 11.10.

------------------------------------------------------------------------

## 11.9 Validación posterior a la recuperación

Una restauración no se considera satisfactoria únicamente porque los
contenedores hayan arrancado o porque Nextcloud muestre su interfaz.
Debe comprobarse el sistema completo: almacenamiento, base de datos,
aplicación, lectura, escritura, persistencia y arranque automático.

### 11.9.1 Sistema y almacenamiento

``` bash
lsblk -f
findmnt /
findmnt /srv/storage
findmnt /srv/backup
sudo findmnt --verify --verbose
sudo /usr/local/sbin/check-nextcloud-data.sh
echo $?
```

El guard de DATA debe devolver `0`. También debe confirmarse que
`/srv/storage` corresponde realmente al filesystem DATA esperado y no al
directorio homónimo perteneciente a SYSTEM.

### 11.9.2 Docker y servicios

``` bash
systemctl status docker.service --no-pager
systemctl status nextcloud.service --no-pager
sudo docker ps
```

Deben estar disponibles `db`, `redis`, `app` y `cron`. `cloudflared`
sólo debe activarse cuando la recuperación haya finalizado y se decida
devolver el servidor a producción.

### 11.9.3 PostgreSQL

``` bash
cd /srv/docker/nextcloud

sudo docker compose exec -T db sh -c \
  'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

Verificar tablas esenciales, datos coherentes, roles globales
recuperados, propietarios, ACL del schema `public` y ausencia de errores
durante `pg_restore`.

Un `pg_restore` parcial o con errores no constituye una recuperación
válida aunque Nextcloud llegue a mostrar alguna funcionalidad.

### 11.9.4 Nextcloud

``` bash
sudo docker compose exec -u www-data app php occ status
```

Debe comprobarse:

``` text
installed: true
needsDbUpgrade: false
maintenance: false
```

Si `maintenance` continúa en `true` después de validar la restauración:

``` bash
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --off
```

No debe ejecutarse `maintenance:repair` automáticamente si no existe una
razón concreta.

### 11.9.5 DATA desde Nextcloud

``` bash
sudo docker compose exec -u www-data app \
  php occ config:system:get datadirectory

sudo docker compose exec app sh -c \
  'head -n 1 /var/www/html/data/.ncdata && df -h /var/www/html/data'
```

Esta comprobación permite detectar un bind mount incorrecto aunque el
directorio exista.

### 11.9.6 Prueba funcional

Desde la interfaz:

1.  iniciar sesión con una cuenta recuperada;
2.  navegar por directorios existentes;
3.  abrir o descargar un archivo conocido;
4.  crear un directorio de prueba;
5.  subir un archivo;
6.  descargarlo a otra ubicación;
7.  comprobar su contenido.

Cuando exista una referencia fiable:

``` bash
sha256sum archivo-original
sha256sum archivo-descargado
```

Los hashes deben coincidir.

### 11.9.7 Validación mediante reboot

Una recuperación completa debe superar un reinicio. Antes del reboot
debe existir un archivo o directorio de prueba creado después de la
restauración.

``` bash
sudo reboot
```

Tras el arranque:

``` bash
findmnt /srv/storage
findmnt /srv/backup
systemctl status nextcloud.service --no-pager
sudo docker ps
```

Volver a acceder a Nextcloud y comprobar que el contenido de prueba
persiste.

### 11.9.8 Criterio de aceptación

``` text
✓ SYSTEM arranca correctamente
✓ DATA está montado en el filesystem esperado
✓ guard de DATA devuelve 0
✓ PostgreSQL está operativo
✓ roles y propietarios son correctos
✓ Redis está operativo
✓ Nextcloud está instalado y sin upgrade pendiente
✓ modo mantenimiento desactivado
✓ archivos existentes son accesibles
✓ lectura validada
✓ escritura validada
✓ descarga validada
✓ reboot completo superado
✓ datos nuevos persisten después del reboot
```

------------------------------------------------------------------------

## 11.10 Primera copia de seguridad tras la recuperación

Una recuperación satisfactoria crea un nuevo estado operativo que debe
protegerse inmediatamente. El backup utilizado para restaurar el
servidor no contiene necesariamente los cambios introducidos durante la
reconstrucción.

### 11.10.1 Revisar la configuración

Comprobar mounts, UUID de los scripts, passphrase Borg, repositorio,
espacio libre, recovery bundle, acceso a PostgreSQL y unidades systemd.

Buscar referencias al hardware anterior:

``` bash
sudo grep -R "UUID_ANTIGUO" \
  /etc \
  /usr/local/sbin \
  /srv/docker 2>/dev/null
```

### 11.10.2 Ejecutar una copia manual

``` bash
sudo systemctl start nextcloud-backup.service
systemctl status nextcloud-backup.service --no-pager
journalctl -u nextcloud-backup.service --since today
```

Al ser `oneshot`, tras una ejecución correcta puede quedar como
`inactive (dead)`. Debe comprobarse el resultado de la ejecución.

### 11.10.3 Validar el nuevo archive

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg list /srv/backup/borg
```

Confirmar un archive posterior a la recuperación y la presencia de:

``` text
srv/storage/nextcloud-data/.ncdata
srv/docker/nextcloud/compose.yml
srv/docker/nextcloud/.env
var/backups/nextcloud/nextcloud.dump
var/backups/nextcloud/postgres-roles.sql
var/backups/hpserver-config/
```

Cuando corresponda:

``` bash
sudo env BORG_PASSCOMMAND="cat /root/.config/borg/passphrase" \
  borg check --show-rc /srv/backup/borg
```

Sólo después de validar esta nueva copia se puede considerar cerrado el
proceso de recuperación.

------------------------------------------------------------------------

## 11.11 RPO y RTO

### 11.11.1 RPO

El Recovery Point Objective representa la cantidad máxima de información
que podría perderse al restaurar desde una copia anterior.

Actualmente se realiza una copia Borg diaria. Por tanto, el RPO práctico
depende del intervalo entre backups y de la hora del último archive
válido.

En el peor caso habitual, una pérdida inmediatamente anterior a la
siguiente copia podría implicar perder aproximadamente los cambios
realizados desde el backup diario anterior.

No constituye una garantía contractual y puede aumentar si una tarea
falla o BACKUP no está disponible.

### 11.11.2 RTO

El Recovery Time Objective representa el tiempo esperado para devolver
el servicio a funcionamiento.

No se establece actualmente un RTO contractual. El tiempo real dependerá
de la avería, disponibilidad de hardware, tamaño de DATA, velocidad de
BACKUP, instalación, restauración de Borg y PostgreSQL y validaciones
posteriores.

La prioridad es la integridad de los datos, no reducir el tiempo de
interrupción omitiendo comprobaciones.

------------------------------------------------------------------------

## 11.12 Limitaciones del sistema de recuperación

### 11.12.1 DATA y BACKUP comparten ubicación física

Aunque son dispositivos independientes, normalmente se encuentran en la
misma ubicación. Un incidente físico grave podría afectar
simultáneamente a SYSTEM, DATA y BACKUP.

La principal mejora futura es disponer de una copia **off-site** o
geográficamente independiente.

### 11.12.2 BACKUP no proporciona alta disponibilidad

Borg permite reconstruir el sistema, pero la pérdida de SYSTEM o DATA
requiere restauración y tiempo de indisponibilidad.

### 11.12.3 Dependencia de secretos externos

Deben preservarse fuera del servidor la passphrase Borg, el material de
recuperación de la clave y las credenciales administrativas necesarias.

### 11.12.4 Dependencias externas

El acceso remoto y el correo dependen de servicios externos. Una
restauración local puede ser válida aunque uno de esos proveedores esté
temporalmente indisponible.

### 11.12.5 Los backups pueden conservar errores lógicos

Un backup puede contener archivos ya eliminados, configuraciones
erróneas o corrupción lógica previa. La retención histórica reduce el
riesgo, pero no lo elimina.

### 11.12.6 Las pruebas deben repetirse

Cambios en Nextcloud, PostgreSQL, Docker, Borg, Debian, scripts o
almacenamiento pueden modificar los requisitos de recuperación. Los DRT
deben repetirse periódicamente y después de cambios estructurales
importantes.

------------------------------------------------------------------------

## 11.13 Disaster Recovery Tests

La estrategia se ha validado mediante pruebas progresivamente más
completas. El objetivo no es demostrar sólo que Borg lista archivos,
sino que el contenido respaldado permite reconstruir realmente el
servicio.

### 11.13.1 DRT-02 --- Restauración de DATA

**Fecha:** 17/09/2026\
**Resultado:** SUPERADO

Objetivo: comprobar que DATA podía reconstruirse completamente desde
Borg en una ubicación independiente, sin modificar producción.

El primer intento perdió la salida final al interrumpirse la sesión SSH.
La prueba se repitió con ejecución persistente, log y código de retorno
independiente.

Se validó:

``` text
✓ extracción completa
✓ código de retorno 0
✓ presencia de .ncdata
✓ estructura de directorios
✓ propietarios y permisos
✓ volumen de datos coherente
✓ recuento coherente de archivos y directorios
✓ hash de un archivo seleccionado idéntico al original
```

### 11.13.2 DRT-03 --- Restauración de componentes de SYSTEM

**Fecha:** 18/09/2026\
**Resultado:** SUPERADO

Se recuperaron:

``` text
compose.yml
.env
volumen de aplicación Nextcloud
nextcloud.dump
```

El host de pruebas no disponía inicialmente de `pg_restore`, por lo que
la validación se realizó mediante una imagen PostgreSQL 18 compatible.

Se comprobó:

``` text
✓ dump legible
✓ restauración PostgreSQL aislada
✓ tablas esenciales presentes
✓ datos de usuarios presentes
✓ filecache presente
✓ configuración de aplicaciones presente
✓ migraciones presentes
✓ versión Nextcloud coherente
```

DRT-03 permitió además detectar que la reconstrucción del host requería
conservar más configuración de SYSTEM, lo que llevó a crear el recovery
bundle automático.

### 11.13.3 DRT-04 --- Reconstrucción completa

**Fecha:** 03/10/2026\
**Resultado:** SUPERADO

Objetivo: simular la pérdida simultánea de SYSTEM y DATA y reconstruir
el servicio sobre Debian 13 limpio utilizando BACKUP y la información
externa de recuperación.

Durante la prueba se detectaron varios puntos importantes.

#### Roles globales PostgreSQL

El primer restore falló porque el dump hacía referencia al rol
`oc_admin`. El dump de la base no contenía todos los objetos globales
del clúster.

El backup se amplió para generar:

``` text
postgres-roles.sql
```

mediante `pg_dumpall --roles-only`, se creó un nuevo archive y la fase
final se repitió con él.

#### Réplica Borg inconsistente

Una sincronización incremental del repositorio de laboratorio produjo
una réplica que podía listar archives, pero cuyo `borg check` detectó
inconsistencias.

No se utilizó `borg check --repair`. La réplica se descartó y se realizó
una copia completa desde un repositorio de producción previamente
validado. La nueva copia independiente superó `borg check` con retorno
`0`.

A partir de ese momento producción quedó excluida como fuente de la
recuperación final.

#### Aislamiento de Cloudflare Tunnel

El entorno recuperado contenía el token real. `cloudflared` se excluyó
deliberadamente para impedir que el laboratorio compitiese con
producción.

#### Modo mantenimiento persistente

Después de restaurar Nextcloud apareció `maintenance: true`, coherente
con un archive capturado durante el modo mantenimiento. Se desactivó
explícitamente después de validar la restauración. No fue necesario
ejecutar `maintenance:repair`.

#### Resultado final

``` text
✓ Debian 13 limpio en un entorno diferente
✓ SYSTEM nuevo
✓ DATA nuevo y vacío
✓ BACKUP independiente
✓ Borg accesible y validado
✓ restauración completa de DATA
✓ restauración del volumen de aplicación
✓ compose.yml y configuración recuperados
✓ PostgreSQL 18 reconstruido desde cero
✓ roles globales recuperados
✓ nextcloud y oc_admin reconstruidos
✓ pg_restore final con código 0
✓ propietarios y ACL recuperados
✓ Redis operativo
✓ Nextcloud 34.0.3 operativo
✓ autenticación con credenciales recuperadas
✓ archivos restaurados accesibles
✓ creación de directorio
✓ subida de archivo
✓ descarga y comprobación
✓ fstab reconciliado con el nuevo hardware
✓ DATA protegido mediante UUID y guard
✓ dependencia systemd de /srv/storage
✓ arranque automático reconstruido
✓ reboot completo satisfactorio
✓ persistencia de datos después del reboot
✓ cloudflared excluido deliberadamente del laboratorio
✓ producción no utilizada como fuente durante la recuperación final
```

DRT-04 demostró que la infraestructura puede reconstruirse desde un
sistema limpio y, además, que los propios DRT permiten descubrir
dependencias que una comprobación de integridad del repositorio por sí
sola no detectaría.

------------------------------------------------------------------------

## 11.14 Checklist de emergencia

### Fase 1 --- Preservar

``` text
[ ] Detener cambios innecesarios
[ ] No formatear ningún dispositivo
[ ] No reutilizar el hardware averiado
[ ] No ejecutar reparaciones destructivas
[ ] Preservar la única copia superviviente
```

### Fase 2 --- Identificar

``` bash
lsblk -f
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
findmnt /srv/storage
findmnt /srv/backup
```

``` text
[ ] SYSTEM identificado
[ ] DATA identificado
[ ] BACKUP identificado
[ ] UUID comprobados
[ ] mounts reales comprobados
```

### Fase 3 --- Diagnosticar

``` text
[ ] A — DATA
[ ] B — BACKUP
[ ] C — SYSTEM
[ ] D — SYSTEM + DATA
[ ] E — pérdida lógica
```

### Fase 4 --- Validar la fuente

``` text
[ ] Passphrase disponible
[ ] Borg accesible
[ ] Archive seleccionado
[ ] DATA presente
[ ] Nextcloud presente
[ ] nextcloud.dump presente
[ ] postgres-roles.sql presente
[ ] recovery bundle presente
```

### Fase 5 --- Recuperar

``` text
[ ] Preparar SYSTEM si es necesario
[ ] Preparar DATA si es necesario
[ ] Reconciliar UUID
[ ] Restaurar DATA
[ ] Restaurar Nextcloud
[ ] Reconstruir PostgreSQL
[ ] Restaurar roles globales
[ ] Restaurar systemd y guards
[ ] Mantener cloudflared desactivado
```

### Fase 6 --- Validar

``` text
[ ] guard DATA = 0
[ ] PostgreSQL operativo
[ ] roles correctos
[ ] Nextcloud operativo
[ ] maintenance = false
[ ] login correcto
[ ] archivos existentes accesibles
[ ] creación correcta
[ ] subida correcta
[ ] descarga correcta
[ ] reboot completo correcto
[ ] persistencia confirmada
```

### Fase 7 --- Volver a producción

``` text
[ ] Habilitar acceso externo
[ ] Verificar HTTPS
[ ] Verificar correo y alertas
[ ] Verificar timers
[ ] Verificar monitorización
```

### Fase 8 --- Crear un nuevo punto de recuperación

``` text
[ ] Ejecutar backup manual
[ ] Comprobar resultado
[ ] Confirmar nuevo archive
[ ] Confirmar nextcloud.dump
[ ] Confirmar postgres-roles.sql
[ ] Confirmar recovery bundle
[ ] Validar Borg
```

### Fase 9 --- Documentar

Registrar fecha y hora, síntomas, componente afectado, causa conocida o
probable, acciones realizadas, archive utilizado, resultado,
comprobaciones, cambios introducidos y mejoras pendientes.

### Regla final

``` text
PARAR
  ↓
IDENTIFICAR
  ↓
PRESERVAR
  ↓
VALIDAR BACKUP
  ↓
RECUPERAR
  ↓
COMPROBAR
  ↓
REINICIAR Y VOLVER A COMPROBAR
  ↓
CREAR NUEVO BACKUP
  ↓
DOCUMENTAR
```

Con estas comprobaciones completadas, el capítulo de Disaster Recovery
puede considerarse cerrado.
