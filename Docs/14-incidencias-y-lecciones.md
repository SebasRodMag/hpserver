# 14. Incidencias y lecciones aprendidas

## 14.1 Objetivo

Este capítulo reúne las incidencias técnicas más relevantes ocurridas
durante la construcción, operación, endurecimiento y validación de
`hpserver`.

No pretende ser un simple registro cronológico de errores. Su objetivo
es conservar:

-   el síntoma observado;
-   la causa confirmada o, cuando no pudo demostrarse, la hipótesis
    razonable;
-   el procedimiento de diagnóstico;
-   la solución aplicada;
-   la mejora permanente introducida;
-   la lección reutilizable.

Una incidencia sólo resulta realmente útil para el proyecto cuando
modifica positivamente el diseño, el procedimiento o la documentación.

La regla utilizada en este capítulo es:

> **Distinguir siempre entre lo observado, lo demostrado y lo
> supuesto.**

Cuando una causa no pudo probarse, se documenta como posible o probable
y no como hecho.

------------------------------------------------------------------------

## 14.2 Instalación de Docker: error en el comando de la clave GPG

### Síntoma

Durante la instalación del repositorio oficial de Docker se ejecutó:

``` bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \ -o /etc/apt/keyrings/docker.asc
```

La clave apareció en la salida estándar en lugar de guardarse
correctamente.

### Causa

Existía un espacio después de la barra invertida.

La barra `\` sólo funciona como continuación de línea cuando es el
último carácter antes del salto.

### Solución

``` bash
sudo curl -fsSL \
  https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
```

### Lección

Los errores de shell pueden parecer problemas del repositorio, permisos
o red cuando en realidad proceden del parsing del comando.

En comandos sensibles conviene comprobar inmediatamente el archivo
esperado:

``` bash
ls -l /etc/apt/keyrings/docker.asc
```

------------------------------------------------------------------------

## 14.3 PostgreSQL 18 y cambio de estructura del volumen

### Síntoma

PostgreSQL entraba en un bucle de reinicios durante la primera
instalación y Nextcloud no podía conectarse a la base de datos.

La configuración inicial utilizaba una imagen flotante:

``` yaml
image: postgres:alpine
```

con:

``` yaml
volumes:
  - ./volumes/postgres:/var/lib/postgresql/data
```

### Causa

La etiqueta flotante descargó PostgreSQL 18.

La imagen había cambiado la organización recomendada de los datos para
permitir directorios versionados bajo:

``` text
/var/lib/postgresql
```

La configuración utilizada correspondía a expectativas anteriores.

### Solución

Se fijó explícitamente la major:

``` yaml
image: postgres:18-alpine
```

y el volumen pasó a:

``` yaml
volumes:
  - ./volumes/postgres:/var/lib/postgresql
```

Como la inicialización fallida era todavía nueva y no contenía datos de
producción, se eliminó ese estado incompleto y se repitió la
inicialización.

### Mejora permanente

Las versiones de servicios con estado deben fijarse explícitamente.

### Lección

Una etiqueta aparentemente cómoda como:

``` text
postgres:alpine
```

puede introducir una nueva major sin que `compose.yml` cambie.

Para servicios stateful:

> **actualizar versión es una operación planificada, no una consecuencia
> accidental de `docker pull`.**

------------------------------------------------------------------------

## 14.4 Carrera de arranque al publicar el puerto sobre la IP del host

### Síntoma

Después de reiniciar el servidor, el contenedor `app` podía no arrancar
correctamente.

La publicación era:

``` yaml
ports:
  - "192.168.1.10:8080:80"
```

### Causa

Docker podía intentar crear el binding antes de que `dhcpcd` hubiese
asignado `192.168.1.10` a la interfaz.

El problema no era Nextcloud, sino una dependencia temporal entre red y
Docker.

### Solución

Se cambió a:

``` yaml
ports:
  - "8080:80"
```

El servicio continúa accesible en LAN mediante:

``` text
http://192.168.1.10:8080
```

sin requerir que Docker encuentre esa IP asignada en el instante exacto
en que crea el contenedor.

### Consideración

El puerto queda escuchando en las interfaces del host, por lo que la
ausencia de port forwarding en el router y el diseño de red siguen
siendo parte de la protección.

### Lección

Una configuración que funciona después de ejecutar manualmente:

``` bash
docker compose up -d
```

puede fallar durante un boot real por diferencias en el orden de
inicialización.

Por ello los reinicios forman parte de las pruebas de infraestructura.

------------------------------------------------------------------------

## 14.5 DNS de Docker no disponible después del arranque

### Síntoma

Los contenedores podían resolver servicios internos mediante Docker,
pero no correctamente nombres externos.

Esto afectó especialmente al envío SMTP desde Nextcloud.

Dentro del contenedor aparecía:

``` text
nameserver 127.0.0.11
```

pero el resolver interno no disponía de una salida externa funcional en
ese estado.

### Solución

Se creó:

``` text
/etc/docker/daemon.json
```

con:

``` json
{
  "dns": [
    "192.168.1.1",
    "1.1.1.1"
  ]
}
```

Después se reinició Docker y se validó de nuevo la resolución y el
correo.

### Lección

`127.0.0.11` dentro de un contenedor es el DNS embebido de Docker. Verlo
en `resolv.conf` no demuestra por sí mismo que la resolución externa
funcione.

Debe probarse la función real que depende de DNS.

------------------------------------------------------------------------

## 14.6 HSTS: configuración creada pero cabecera no demostrada inicialmente

### Situación

Nextcloud advertía que faltaba:

``` text
Strict-Transport-Security
```

con un `max-age` suficiente.

Se creó una regla en Cloudflare para el hostname externo.

### Dificultad

Configurar una regla en el panel no equivale a demostrar que la
respuesta HTTP final contiene la cabecera.

### Lección

Las configuraciones de seguridad deben validarse desde el extremo
cliente.

El criterio correcto es inspeccionar la respuesta HTTPS final, por
ejemplo:

``` bash
curl -I https://HOST
```

y comprobar la cabecera esperada.

No debe documentarse como «validado» únicamente porque la regla exista
en el proveedor.

------------------------------------------------------------------------

## 14.7 msmtp y AppArmor

### Síntoma

Durante la configuración de alertas del host, `msmtp` presentó problemas
relacionados con el acceso a un logfile personalizado.

### Diagnóstico

La configuración de AppArmor del paquete limitaba el acceso esperado.

### Solución

Se eliminó la necesidad del logfile personalizado y se adoptó `journald`
como fuente de logs.

### Estado final

La configuración de alertas utiliza:

``` text
/root/.msmtprc
/root/.config/msmtp/
/etc/hpserver-alert.conf
```

con permisos restrictivos.

### Lección

Antes de modificar una política de seguridad del sistema para acomodar
una aplicación, debe comprobarse si la necesidad puede eliminarse.

En este caso, utilizar `journald` simplificó el diseño y evitó mantener
una excepción AppArmor propia.

------------------------------------------------------------------------

## 14.8 Borg, passphrase y entorno de sudo

### Síntoma

Algunas ejecuciones manuales de Borg no encontraban la passphrase
esperada cuando se utilizaba `sudo`.

### Causa

Las variables de entorno del usuario no se conservan necesariamente al
cambiar de contexto mediante `sudo`.

### Solución

La automatización utiliza una fuente explícita protegida:

``` text
/root/.config/borg/passphrase
```

y:

``` text
BORG_PASSCOMMAND
```

### Lección

Una automatización privilegiada no debe depender implícitamente del
entorno interactivo de un usuario.

Los secretos deben tener una ruta y mecanismo de carga explícitos, con
permisos adecuados.

------------------------------------------------------------------------

## 14.9 Timer trimestral deshabilitado accidentalmente

### Incidencia

Durante la configuración de las verificaciones periódicas se detectó que
el timer destinado a una comprobación trimestral no permanecía
habilitado como se esperaba.

### Solución

Se revisó la unidad, se habilitó de nuevo y se comprobó mediante:

``` bash
systemctl list-timers --all
```

### Lección

Crear correctamente un `.timer` no garantiza que esté activo.

Después de modificar timers deben comprobarse:

``` bash
systemctl status NOMBRE.timer
systemctl list-timers --all
```

y verificar la próxima ejecución calculada por systemd.

------------------------------------------------------------------------

## 14.10 Incidente del cliente Android

### Síntoma

El cliente oficial de Nextcloud para Android realizó operaciones
autenticadas que incluyeron la eliminación de una carpeta completa y su
posterior recreación vacía mientras el usuario no estaba interactuando
conscientemente con la aplicación.

Los archivos originales continuaban presentes en el teléfono.

### Observación

Los logs permitieron identificar las operaciones realizadas por el
cliente, pero no demostrar con certeza el mecanismo interno que las
provocó.

### Recuperación

Tras reinstalar el cliente, su estado local se reinicializó y el
contenido que seguía existiendo en el dispositivo volvió a subirse.

### Causa

**No determinada con certeza.**

No debe documentarse como un bug concreto de caché, sincronización o
versión sin evidencia adicional.

### Lección

Un cliente sincronizado y autenticado también puede originar operaciones
destructivas válidas desde el punto de vista del servidor.

Por tanto:

> **sincronización no equivale a backup.**

La recuperación histórica debe depender de mecanismos independientes
como Borg y, cuando sea suficiente, de papelera/versiones de Nextcloud.

------------------------------------------------------------------------

## 14.11 Carrera crítica: Docker arrancó antes de montar DATA

### Síntoma

Después de un boot, Nextcloud podía arrancar aparentemente con la ruta:

``` text
/srv/storage/nextcloud-data
```

pero sin utilizar realmente el filesystem DATA esperado.

### Causa

Docker creó los contenedores antes de que `/srv/storage` estuviese
montado.

En ese instante existía el directorio `/srv/storage` perteneciente al
filesystem SYSTEM.

El bind mount del contenedor quedó asociado a ese directorio subyacente.

Montar posteriormente DATA sobre `/srv/storage` en el host no corregía
automáticamente el mount namespace del contenedor ya creado.

### Riesgo

Esta incidencia era especialmente peligrosa porque las rutas parecían
correctas por nombre.

Nextcloud podía escribir datos en SYSTEM creyendo que utilizaba DATA.

### Reparación

Después de montar correctamente DATA fue necesario recrear los
contenedores afectados para que sus bind mounts se construyeran contra
el filesystem correcto.

### Mejora permanente: guard de DATA

Se creó:

``` text
/usr/local/sbin/check-nextcloud-data.sh
```

que comprueba:

-   que `/srv/storage` es un mountpoint;
-   que corresponde al UUID esperado;
-   que existe el directorio de datos;
-   que `.ncdata` contiene el marcador esperado.

### Mejora permanente: systemd

`nextcloud.service` utiliza:

``` ini
RequiresMountsFor=/srv/storage
After=srv-storage.mount
```

y:

``` ini
ExecStartPre=/usr/local/sbin/check-nextcloud-data.sh
```

Además Docker dispone de un override con dependencia de `/srv/storage`.

### Lección

Esta fue una de las incidencias más importantes del proyecto.

> **La existencia de una ruta no demuestra que el filesystem correcto
> esté montado detrás de ella.**

Los servicios que dependen de almacenamiento externo deben validar
identidad y montaje, no sólo directorios.

------------------------------------------------------------------------

## 14.12 Error de sintaxis en un override systemd

### Incidencia

Durante la configuración de dependencias systemd se introdujo
inicialmente:

``` ini
[unit]
```

en lugar de:

``` ini
[Unit]
```

### Detección

La validación de la unidad permitió detectar el problema antes de
confiar en ella.

### Lección

Después de crear o modificar units:

``` bash
sudo systemctl daemon-reload
sudo systemd-analyze verify /ruta/unidad.service
```

Una configuración visualmente plausible puede no estar siendo
interpretada como se espera.

------------------------------------------------------------------------

## 14.13 Apagado inesperado por evento de botón de encendido

### Síntoma

El journal registró:

``` text
Power key pressed short
```

antes de un apagado inesperado.

### Contexto físico

El cable/mecanismo del botón de encendido del portátil presenta
deterioro conocido.

Esto hace plausible una activación eléctrica falsa.

### Causa

El evento de power key está demostrado por el journal.

La causa física exacta del evento **no pudo demostrarse**.

El cable deteriorado se considera una explicación plausible, no una
conclusión definitiva.

### Mitigación

Se configuró:

``` ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
HandlePowerKey=ignore
```

### Lección

Los logs permiten demostrar qué evento recibió el sistema, pero no
necesariamente qué fenómeno físico lo originó.

Debe conservarse esa distinción en la documentación.

------------------------------------------------------------------------

## 14.14 Estado inconsistente de Docker después del apagado

### Síntoma

Después del apagado inesperado, Docker mostró un error relacionado con
una capa writable del contenedor:

``` text
RWLayer ... unexpectedly nil
```

### Solución

El contenedor `app` fue recreado.

Los datos persistentes de DATA permanecían intactos.

### Lección

Los contenedores deben tratarse como componentes recreables.

Los datos críticos deben residir exclusivamente en almacenamiento
persistente claramente identificado.

Esta incidencia reforzó la separación entre:

``` text
contenedor
configuración persistente
base de datos
DATA
BACKUP
```

------------------------------------------------------------------------

## 14.15 Endurecimiento del backup después de la incidencia de DATA

La carrera de montaje reveló que un backup podía convertirse en un
riesgo adicional si copiaba silenciosamente desde el filesystem
equivocado.

### Mejora

El script de backup pasó a validar antes de copiar:

-   UUID de DATA;
-   mountpoint;
-   `.ncdata`;
-   filesystem observado desde host y contenedor;
-   UUID de BACKUP;
-   espacio libre;
-   disponibilidad de la aplicación;
-   dump PostgreSQL.

### Lección

Un backup que termina con código `0` no es útil si ha respaldado la
fuente equivocada.

> **Primero se valida la identidad de los datos; después se realiza la
> copia.**

------------------------------------------------------------------------

## 14.16 DRT-02: pérdida de la sesión SSH durante una restauración

### Objetivo

Restaurar DATA desde Borg en una ubicación independiente.

### Incidencia

La primera extracción se ejecutó desde una sesión SSH que se
interrumpió.

Como consecuencia, no quedó disponible de forma fiable la salida final
ni el código de retorno del proceso.

### Solución

La prueba se repitió mediante ejecución persistente, guardando:

``` text
log
código de retorno
```

independientemente de la sesión SSH.

La segunda ejecución finalizó correctamente.

### Lección

Las operaciones largas no deben depender de la vida de una terminal
remota.

Para pruebas de recuperación debe conservarse evidencia verificable del
resultado.

------------------------------------------------------------------------

## 14.17 DRT-03: `pg_restore` no instalado en el host

### Síntoma

Durante la validación aislada del dump:

``` text
pg_restore: command not found
```

equivalente a un fallo de ejecución del host.

### Solución

En lugar de modificar innecesariamente el sistema de pruebas, se utilizó
una imagen:

``` text
postgres:18-alpine
```

para ejecutar las herramientas de la misma major que producción.

### Resultado

El dump pudo listarse y restaurarse de forma aislada.

### Lección

Los contenedores también son útiles como herramientas reproducibles de
administración.

Utilizar la misma major reduce diferencias entre la herramienta de
validación y el servicio origen.

------------------------------------------------------------------------

## 14.18 DRT-03: el backup no contenía suficiente configuración del host

### Hallazgo

DRT-03 demostró que podían recuperarse Nextcloud, el volumen de
aplicación y PostgreSQL, pero una reconstrucción completa de Debian
requería más información del host.

### Mejora

Se creó:

``` text
/usr/local/sbin/generate-hpserver-recovery.sh
```

que genera:

``` text
/var/backups/hpserver-config/
```

con una selección curada de:

-   `fstab`;
-   red;
-   Docker;
-   repositorio APT;
-   units y timers systemd;
-   scripts;
-   configuración de energía;
-   alertas;
-   manifiesto de paquetes;
-   información de discos y mounts.

### Diseño

No se incluyeron la passphrase Borg ni la clave exportada dentro del
propio repositorio, evitando una dependencia circular.

### Lección

Respaldar una aplicación no equivale a poder reconstruir el host que la
ejecuta.

------------------------------------------------------------------------

## 14.19 DRT-04: creación incorrecta de la partición BACKUP

### Síntoma

Durante la preparación de los discos virtuales, una orden con
porcentajes:

``` text
0% 100%
```

produjo una partición de aproximadamente 0,09 GiB en lugar de ocupar el
disco esperado.

### Solución

La partición se recreó utilizando límites explícitos en sectores,
reservando correctamente el inicio y final necesarios.

Después se volvió a formatear y se corrigió `fstab`.

### Lección

Después de cualquier operación de particionado debe verificarse el
resultado real:

``` bash
lsblk
lsblk -f
```

antes de continuar.

Nunca asumir que una herramienta interpretó los parámetros como se
pretendía.

------------------------------------------------------------------------

## 14.20 DRT-04: expansión de glob antes de `sudo`

### Síntoma

La instalación de varios scripts mediante un glob no funcionó como se
esperaba al intentar acceder a archivos protegidos bajo `/root`.

### Causa

El shell del usuario expande:

``` text
*.sh
```

antes de ejecutar `sudo`.

Por tanto, elevar privilegios al comando no eleva privilegios al proceso
previo de expansión realizado por el shell actual.

### Solución

Se ejecutó el comando mediante un shell privilegiado controlado:

``` bash
sudo bash -c '... *.sh ...'
```

### Lección

En shell, es importante conocer qué parte del comando interpreta el
shell actual y qué parte ejecuta realmente `sudo`.

------------------------------------------------------------------------

## 14.21 DRT-04: reconciliación manual de UUID incompleta

### Síntoma

Después de adaptar los scripts al nuevo hardware, todavía permanecía una
referencia a un UUID de producción en:

``` text
check-borg.sh
```

### Detección

Una búsqueda global del UUID antiguo permitió localizarla.

### Mejora

Después de reconciliar hardware debe ejecutarse una búsqueda
transversal:

``` bash
sudo grep -R "UUID_ANTIGUO" \
  /etc \
  /usr/local/sbin \
  /srv/docker 2>/dev/null
```

### Lección

Una checklist basada exclusivamente en recordar archivos conocidos es
menos fiable que comprobar globalmente la identidad antigua.

------------------------------------------------------------------------

## 14.22 DRT-04: inestabilidad de VirtualBox en el primer host

### Síntomas

Durante el DRT aparecieron mensajes como:

``` text
watchdog: BUG: soft lockup
rcu_preempt self-detected stall
clocksource: Long readout interval
Perf NMI watchdog permanently disabled
```

### Pruebas realizadas

Se realizaron pruebas A/B con:

-   Docker sin contenedores;
-   Redis;
-   PostgreSQL;
-   stack detenido.

Los stalls aparecieron especialmente al ejecutar contenedores.

El host Windows utilizaba VirtualBox en un entorno donde también estaban
presentes Hyper-V/VBS.

Se actualizó VirtualBox y se probaron cambios gráficos, pero la
plataforma continuó mostrando problemas.

### Resolución operativa

La misma VM se trasladó a otro ordenador.

En el segundo host los contenedores permanecieron funcionando durante un
periodo prolongado de prueba sin reproducir los errores del journal.

### Conclusión

La inestabilidad se consideró específica del primer entorno de
virtualización.

La causa exacta dentro de ese host **no quedó demostrada**.

### Lección

Un fallo del laboratorio no debe confundirse automáticamente con un
fallo del backup o del software que se está validando.

Mover el mismo entorno a otra plataforma permitió aislar mejor la
variable.

------------------------------------------------------------------------

## 14.23 DRT-04: el dump PostgreSQL dependía de un rol global no respaldado

### Síntoma

La restauración del dump falló con:

``` text
pg_restore: ERROR: role "oc_admin" does not exist
```

aunque se había utilizado una estrategia destinada a evitar restaurar
propietarios.

### Investigación

El listado del dump mostró ACL relacionadas con `oc_admin`.

La inspección de producción confirmó que `oc_admin` no era un residuo
irrelevante:

-   existía realmente;
-   poseía objetos de Nextcloud;
-   participaba en ACL del schema `public`.

También se confirmó la existencia del rol `nextcloud` con sus atributos
correspondientes.

### Causa

`pg_dump` de una base individual no incluye todos los objetos globales
del clúster PostgreSQL.

Los roles pertenecen a ese ámbito global.

### Mejora

El backup se amplió para generar:

``` text
postgres-roles.sql
```

mediante:

``` bash
pg_dumpall --roles-only
```

El archivo se protege porque puede contener hashes de contraseñas.

### Restauración final

En un PostgreSQL limpio, Compose ya había creado `nextcloud`.

Por ello se eliminó únicamente la sentencia exacta:

``` text
CREATE ROLE nextcloud;
```

de una copia temporal del SQL.

Después:

1.  se restauraron los roles y atributos;
2.  se restauró la base sin eliminar ownership;
3.  se comprobaron propietarios y ACL.

La restauración final terminó con código `0`.

### Lección

> **Un dump válido no demuestra por sí solo que una base sea
> completamente reconstruible.**

Los objetos globales deben formar parte explícita del plan de
recuperación.

------------------------------------------------------------------------

## 14.24 DRT-04: réplica Borg inconsistente después de una sincronización incremental

### Primer intento

Al sincronizar el repositorio Borg hacia el laboratorio, `rsync` falló
con código `23`.

La raíz del repositorio había cambiado de propietario, pero los objetos
internos seguían perteneciendo a `root`, provocando errores de permisos.

### Segundo estado

Después de corregir permisos y repetir incrementalmente la copia, los
archives podían listarse.

Sin embargo:

``` bash
borg check
```

detectó una diferencia entre el índice comprometido y el reconstruido.

Se observaron 47 objetos adicionales en la reconstrucción.

### Decisión

No se utilizó:

``` text
borg check --repair
```

porque la réplica de laboratorio era descartable y existía un
repositorio origen válido.

### Solución

Se eliminó únicamente la réplica Borg defectuosa del laboratorio y se
realizó una copia completa nueva desde producción, después de validar
primero el repositorio de producción.

La nueva copia superó:

``` bash
borg check --show-rc
```

con código `0`.

### Lección

Que:

``` bash
borg list
```

funcione no demuestra que una réplica del repositorio sea íntegra.

Y cuando existe una fuente sana:

> **es preferible descartar una réplica dudosa y volver a copiar que
> reparar innecesariamente la única evidencia disponible.**

------------------------------------------------------------------------

## 14.25 DRT-04: `rsync` ejecutado accidentalmente hacia el propio host

### Incidencia

Después de limpiar la réplica defectuosa se lanzó accidentalmente una
sincronización desde el host DRT hacia la dirección IP que correspondía
al propio DRT.

El resultado fue:

``` text
xfr#0
```

con código `0`.

### Impacto

No se utilizó `--delete` y producción no resultó modificada.

### Detección

El prompt y la ausencia de transferencia permitieron identificar que
origen y destino no eran los previstos.

### Lección

Antes de una sincronización crítica debe comprobarse explícitamente:

``` text
hostname local
IP local
hostname remoto
IP remota
ruta origen
ruta destino
```

Un código de retorno `0` significa que el comando ejecutado terminó
correctamente; no significa que el operador haya pedido la operación
correcta.

------------------------------------------------------------------------

## 14.26 DRT-04: Nextcloud restaurado en modo mantenimiento

### Síntoma

Después de la restauración:

``` text
maintenance: true
```

### Explicación

El backup se crea mientras el script mantiene Nextcloud en modo
mantenimiento para obtener un estado coherente.

El flag queda persistido dentro de la configuración respaldada.

Por tanto, restaurar ese estado puede reproducir legítimamente:

``` text
maintenance: true
```

### Solución

Después de validar base de datos, DATA y aplicación:

``` bash
sudo docker compose exec -u www-data app \
  php occ maintenance:mode --off
```

### Decisión

No se ejecutó `maintenance:repair` porque no existía una necesidad
demostrada.

### Lección

No todos los estados inesperados después de un restore son corrupción.

Antes de «reparar» debe comprenderse si el estado es una consecuencia
lógica del procedimiento de backup.

------------------------------------------------------------------------

## 14.27 DRT-04: aislamiento del Cloudflare Tunnel

### Riesgo

El `.env` restaurado contenía el token real de Cloudflare Tunnel.

Arrancar el stack completo en el laboratorio podía hacer que
`cloudflared` intentara conectar utilizando la identidad de producción.

### Medida

Durante la fase final se arrancaron únicamente:

``` text
db
redis
app
cron
```

`cloudflared` permaneció fuera del laboratorio.

Para la validación del reboot se adaptó temporalmente el `ExecStart` de
`nextcloud.service` del DRT:

``` text
docker compose up -d db redis app cron
```

### Lección

Una copia de seguridad puede contener credenciales suficientemente
reales como para que un entorno restaurado interactúe con producción.

Los laboratorios de recuperación deben aislar explícitamente las
integraciones externas.

------------------------------------------------------------------------

## 14.28 DRT-04 como validación del diseño

DRT-04 comenzó como una prueba del backup y terminó validando también la
arquitectura operativa.

### Resultado

Se consiguió reconstruir sobre un entorno diferente:

``` text
✓ Debian 13 limpio
✓ SYSTEM nuevo
✓ DATA nuevo
✓ BACKUP independiente
✓ Borg validado
✓ DATA restaurado
✓ Nextcloud restaurado
✓ PostgreSQL 18 reconstruido
✓ roles globales restaurados
✓ ownership y ACL restaurados
✓ Redis operativo
✓ cron operativo
✓ login válido
✓ lectura válida
✓ escritura válida
✓ descarga válida
✓ systemd reconstruido
✓ guard de DATA operativo
✓ reboot satisfactorio
✓ persistencia posterior al reboot
✓ cloudflared aislado
```

### Hallazgo principal

El DRT descubrió un defecto real del backup: los roles globales
PostgreSQL no estaban incluidos.

La prueba no se dio por superada ignorando el fallo.

Se modificó producción, se creó un nuevo backup con
`postgres-roles.sql`, se generó una nueva copia independiente y se
repitió la recuperación.

### Lección

Ésta es la diferencia entre:

``` text
tener backups
```

y:

``` text
tener una estrategia de recuperación probada
```

------------------------------------------------------------------------

## 14.29 Lecciones transversales

Las incidencias anteriores permiten extraer principios que se aplican a
todo el servidor.

### 14.29.1 Validar identidad, no sólo rutas

``` text
/srv/storage
```

puede existir sin que DATA esté montado.

Por ello se utilizan:

``` text
mountpoint
UUID
.ncdata
guards
```

### 14.29.2 Un código `0` sólo valida la operación ejecutada

No demuestra necesariamente:

-   que se eligió el host correcto;
-   que se copió la fuente correcta;
-   que el contenido es recuperable;
-   que el servicio sobrevivirá a un reboot.

### 14.29.3 Los backups deben restaurarse

``` text
borg create → no basta
borg list   → no basta
borg check  → muy útil, pero no basta para demostrar toda la aplicación
restore     → evidencia de recuperabilidad
DRT completo → evidencia del sistema
```

### 14.29.4 Las dependencias ocultas aparecen durante la recuperación

El caso de `oc_admin` mostró una dependencia que no era evidente
observando únicamente `nextcloud.dump`.

Las pruebas reales revelan información que la documentación inicial
puede desconocer.

### 14.29.5 El reboot es una prueba de integración

Un reboot valida simultáneamente:

``` text
fstab
mounts
red
Docker
systemd
guards
orden de arranque
contenedores
persistencia
```

Por ello forma parte de las validaciones importantes.

### 14.29.6 No reparar antes de comprender

Ejemplos del proyecto:

``` text
no usar borg check --repair por reflejo
no ejecutar maintenance:repair sin necesidad
no aplicar chmod/chown recursivos automáticamente
no borrar volúmenes para corregir contenedores
```

La reparación incorrecta puede destruir evidencia o agravar el
incidente.

### 14.29.7 Preservar antes de modificar

Ante una incidencia:

``` text
PARAR
  ↓
OBSERVAR
  ↓
PRESERVAR
  ↓
DIAGNOSTICAR
  ↓
MODIFICAR
```

### 14.29.8 Distinguir hecho de hipótesis

Ejemplos:

-   el journal registró un evento de botón de encendido: **hecho**;
-   el cable deteriorado pudo generarlo: **hipótesis plausible**;
-   el cliente Android emitió operaciones DELETE autenticadas:
    **hecho**;
-   su causa interna exacta: **no demostrada**;
-   el primer host VirtualBox sufría stalls: **hecho**;
-   la interacción exacta entre VirtualBox, Hyper-V/VBS y el host: **no
    demostrada**.

Esta distinción aumenta la calidad de la documentación técnica.

------------------------------------------------------------------------

## 14.30 Mejoras introducidas como consecuencia de incidencias

El diseño actual no fue definido completamente desde el primer día.
Varias protecciones existen porque una prueba o incidencia demostró su
necesidad.

  -----------------------------------------------------------------------
  Incidencia / hallazgo               Mejora permanente
  ----------------------------------- -----------------------------------
  PostgreSQL 18                       Major fijada y layout correcto

  Bind a IP durante boot              Publicación independiente de la
                                      asignación temprana de IP

  DNS Docker                          DNS explícito en `daemon.json`

  DATA no montado                     Guard por UUID + `.ncdata`

  Carrera Docker/DATA                 Dependencias systemd

  Backup sobre filesystem incorrecto  Guards dentro del backup

  Configuración host no respaldada    Recovery bundle

  Borg y secretos                     Passphrase y clave externa
                                      gestionadas explícitamente

  DRT-02/SSH                          Ejecuciones largas con log y RC
                                      persistentes

  Falta de `pg_restore`               Herramienta PG18 reproducible
                                      mediante contenedor

  UUID olvidado                       Búsqueda global después de
                                      reconciliar

  `oc_admin` ausente                  `pg_dumpall --roles-only`

  Réplica Borg inconsistente          Validación independiente después de
                                      copiar repositorio

  Cloudflare en DRT                   Aislamiento explícito de servicios
                                      externos

  Reboot de DRT                       Validación obligatoria de
                                      persistencia y arranque
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 14.31 Registro de futuras incidencias

Las nuevas incidencias deben añadirse siguiendo una plantilla común.

``` markdown
## INC-XX — Título

**Fecha:** YYYY-MM-DD
**Estado:** RESUELTA / ABIERTA / EN OBSERVACIÓN

### Síntoma

Qué se observó.

### Impacto

Qué servicio o datos resultaron afectados.

### Evidencia

Logs, códigos de retorno, estado del sistema y pruebas relevantes.

### Causa

Causa demostrada.

Si no está demostrada:

> Causa no confirmada.

### Resolución

Acciones realizadas.

### Validación

Cómo se comprobó que el sistema volvió a un estado correcto.

### Mejora permanente

Cambio introducido para evitar recurrencia o mejorar su detección.

### Lección

Conclusión reutilizable.
```

No deben incluirse en el repositorio público:

``` text
contraseñas
tokens
hashes de contraseña
claves privadas
passphrase Borg
token Cloudflare Tunnel
credenciales SMTP
UUID u otros identificadores si se decide mantenerlos privados
datos personales de usuarios
```

------------------------------------------------------------------------

## 14.32 Cierre

La evolución de `hpserver` muestra un patrón importante:

``` text
INCIDENCIA
    ↓
DIAGNÓSTICO
    ↓
EVIDENCIA
    ↓
SOLUCIÓN
    ↓
VALIDACIÓN
    ↓
MEJORA DE DISEÑO
    ↓
DOCUMENTACIÓN
    ↓
PRUEBA DE RECUPERACIÓN
```

Las incidencias no se han tratado únicamente como problemas puntuales.

Han servido para introducir:

-   control explícito del almacenamiento;
-   dependencias de arranque;
-   backups endurecidos;
-   recovery bundle;
-   monitorización;
-   alertas;
-   pruebas reales de restauración;
-   aislamiento de servicios externos;
-   procedimientos de migración;
-   documentación reproducible.

El resultado final no debe evaluarse únicamente por que Nextcloud esté
disponible.

El objetivo alcanzado es disponer de una infraestructura doméstica que
pueda:

``` text
funcionar
ser observada
fallar de forma detectable
ser diagnosticada
ser respaldada
ser restaurada
ser migrada
y ser reconstruida
```

sin depender exclusivamente del conocimiento informal de quien la
instaló.
