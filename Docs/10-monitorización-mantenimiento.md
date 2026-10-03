# 10 — Monitorización y mantenimiento

## 1. Objetivo

Un servidor doméstico puede permanecer aparentemente operativo durante meses mientras acumula problemas que todavía no afectan al usuario:

- degradación de un disco;
- backups que dejan de ejecutarse;
- errores de integridad en el repositorio;
- falta de espacio;
- contenedores detenidos;
- actualizaciones pendientes;
- errores recurrentes en Nextcloud;
- servicios que no arrancan correctamente después de un reinicio.

Por este motivo, `hpserver` dispone de varias comprobaciones automáticas y de una rutina de mantenimiento periódico.

El objetivo no es disponer de una plataforma de monitorización empresarial, sino conseguir un sistema sencillo:

```text
               hpserver
                  │
       ┌──────────┼───────────┐
       │          │           │
      SMART      Borg       systemd
       │          │           │
       └──────────┼───────────┘
                  │
               logging
                  │
          detección de fallo
                  │
             alerta email
                  │
                  ▼
            administrador
```

La filosofía utilizada es:

> Automatizar las comprobaciones repetitivas y avisar cuando requieren intervención, manteniendo manuales las operaciones que puedan modificar o reparar datos.

---

## 2. Niveles de monitorización

La supervisión se divide en varios niveles independientes.

| Nivel | Qué se comprueba |
|---|---|
| Hardware | Estado SMART de los discos |
| Filesystems | Montaje correcto de DATA y BACKUP |
| Docker | Estado de los contenedores |
| Nextcloud | Aplicación, cron y logs |
| PostgreSQL | Disponibilidad y dump durante backup |
| Borg | Creación e integridad de backups |
| systemd | Ejecución de servicios y timers |
| Capacidad | Espacio disponible |
| Alertas | Notificación de fallos relevantes |

No todas estas comprobaciones necesitan ejecutarse con la misma frecuencia.

Un disco puede revisarse mensualmente mediante un test SMART, mientras que el backup debe verificarse diariamente.

---

## 3. Timers de mantenimiento

Las tareas automáticas principales se ejecutan mediante timers de systemd.

La planificación actual es:

```text
DIARIO
03:30 ─── Backup Nextcloud/Borg

SEMANAL
Domingo 04:30 ─── Borg check

MENSUAL
Primer sábado 05:00 ─── SMART

TRIMESTRAL
Primer domingo de enero/abril/julio/octubre 06:00
└── Verificación profunda Borg
```

Puede consultarse la programación mediante:

```bash
systemctl list-timers --all
```

o filtrar las tareas relacionadas:

```bash
systemctl list-timers --all | grep -E \
'nextcloud|borg|smart'
```

Esta orden es especialmente útil después de modificar unidades systemd, actualizar el servidor o realizar cambios de mantenimiento.

---

## 4. Backup diario

La comprobación más frecuente es el propio backup.

El timer:

```text
nextcloud-backup.timer
```

ejecuta diariamente:

```text
nextcloud-backup.service
```

El script ya realiza varias comprobaciones antes de comenzar:

```text
DATA montado
    │
    ├── UUID correcto
    ├── .ncdata correcta
    └── app ve el mismo filesystem
              │
              ▼
BACKUP montado
    │
    ├── UUID correcto
    └── espacio suficiente
              │
              ▼
PostgreSQL dump
    │
    └── validación pg_restore
              │
              ▼
Borg
```

Por tanto, el backup diario funciona también como una comprobación básica del estado de buena parte de la infraestructura.

Los detalles completos del sistema de backup se documentan en el capítulo 09.

---

## 5. Comprobación semanal de Borg

La existencia de nuevos archivos Borg no garantiza por sí sola que el repositorio siga siendo íntegro.

Por ello se ejecuta semanalmente una comprobación mediante:

```bash
borg check
```

El script utilizado es:

```text
/usr/local/sbin/check-borg.sh
```

y se ejecuta:

```text
Domingo — 04:30
```

La comprobación valida la estructura del repositorio sin intentar repararla.

El principio utilizado es:

```text
detectar automáticamente
          │
          ▼
diagnosticar
          │
          ▼
intervenir manualmente
```

y no:

```text
detectar
   │
   ▼
reparar automáticamente
```

Por ello **no se automatiza**:

```bash
borg check --repair
```

Una reparación puede modificar información del repositorio y debe realizarse únicamente después de investigar el problema.

---

## 6. Verificación profunda trimestral

Además del `borg check` semanal existe una comprobación más exhaustiva de los datos almacenados.

Se ejecuta trimestralmente:

```text
Primer domingo de:
- enero
- abril
- julio
- octubre

Hora: 06:00
```

mediante:

```text
/usr/local/sbin/check-borg-data.sh
```

Esta comprobación es deliberadamente menos frecuente porque implica leer una cantidad significativamente mayor de información.

La separación permite mantener:

```text
comprobación ligera
       │
       └── frecuente

verificación profunda
       │
       └── periódica
```

sin someter innecesariamente al disco BACKUP a lecturas completas constantes.

No se automatiza ninguna operación de reparación sobre el repositorio.

---

## 7. Monitorización SMART

Los discos constituyen uno de los componentes con mayor probabilidad de sufrir degradación física con el paso del tiempo.

Se utiliza `smartmontools` para consultar SMART.

El script:

```text
/usr/local/sbin/check-smart-disks.sh
```

realiza las comprobaciones periódicas.

Entre otros aspectos se vigilan indicadores críticos relacionados con:

```text
Reallocated sectors
Pending sectors
Offline uncorrectable sectors
CRC errors
SMART overall health
```

También se ejecuta un test corto SMART y se comprueba que finalice correctamente.

La tarea se programa:

```text
Primer sábado de cada mes
05:00
```

Un resultado SMART correcto no sustituye a una copia de seguridad. Un disco puede fallar sin proporcionar necesariamente una advertencia suficientemente temprana.

---

## 8. No confiar en `/dev/sdX`

Una de las lecciones importantes obtenidas durante la instalación es que nombres como:

```text
/dev/sda
/dev/sdb
/dev/sdc
```

no constituyen identificadores permanentes de un disco.

Durante la vida del servidor estos nombres pueden cambiar debido a:

- orden de detección;
- conexión USB;
- controladores;
- reinicios;
- sustitución de hardware.

Por ello la identificación conceptual debe realizarse por roles:

```text
SYSTEM
DATA
BACKUP
```

y técnicamente mediante identificadores persistentes cuando corresponda:

```text
UUID
LABEL
modelo
número de serie
```

Los números de serie y UUID reales se mantienen fuera de la documentación pública.

> Nunca debe ejecutarse una operación destructiva sobre `/dev/sdX` basándose únicamente en que ese nombre coincidía con un disco en una ejecución anterior.

---

## 9. Particularidades del disco DATA

El disco DATA es un HDD reutilizado y dispone de un número considerable de horas de funcionamiento.

Las comprobaciones realizadas hasta el momento no han mostrado:

- sectores reasignados;
- sectores pendientes;
- sectores no corregibles;
- errores CRC relevantes;
- fallo del estado general SMART.

Sin embargo, su antigüedad hace especialmente importante mantener vigilancia SMART.

Un atributo que merece seguimiento es el número de ciclos de carga y descarga de cabezales (`Load_Cycle_Count`), que presenta históricamente un valor elevado.

Esto no implica por sí solo un fallo inmediato del disco, pero constituye una razón adicional para:

- mantener backups independientes;
- observar su evolución;
- realizar pruebas periódicas;
- no utilizar SMART como sustituto del backup.

---

## 10. Particularidades del disco BACKUP

El HDD utilizado para BACKUP también fue sometido a pruebas antes de confiarle el repositorio Borg.

Se realizaron:

- consulta SMART;
- test SMART corto;
- intentos de test extendido;
- lectura secuencial completa;
- revisión posterior de los atributos SMART;
- revisión de mensajes del kernel relacionados con I/O.

Los tests extendidos SMART llegaron a abortarse cerca de su finalización por el host/dispositivo, por lo que no se interpretaron como una prueba completa satisfactoria.

Sin embargo, la lectura secuencial completa terminó correctamente y no produjo:

```text
sectores pendientes
sectores no corregibles
errores I/O
resets
desconexiones
```

Por ello el disco se consideró adecuado para su función actual, manteniendo monitorización periódica.

Esta distinción es importante: un test incompleto no debe documentarse como un resultado satisfactorio que realmente no se obtuvo.

---

## 11. Logs

Los logs son el principal mecanismo para reconstruir qué ocurrió cuando aparece una incidencia.

Las fuentes principales son:

```text
journalctl
Docker / Compose
Nextcloud
Borg
scripts administrativos
```

Para una unidad systemd concreta:

```bash
journalctl -u NOMBRE.service
```

Últimos eventos:

```bash
journalctl -u NOMBRE.service -n 100
```

Desde el último arranque:

```bash
journalctl -b
```

Arranque anterior:

```bash
journalctl -b -1
```

Este último comando resulta especialmente útil durante la investigación de apagados y problemas ocurridos durante el proceso de arranque.

---

## 12. Logs de Docker

El estado general del stack puede consultarse desde:

```bash
cd /srv/docker/nextcloud
docker compose ps
```

Los logs completos:

```bash
docker compose logs
```

o para un servicio concreto:

```bash
docker compose logs app
docker compose logs db
docker compose logs redis
docker compose logs cloudflared
docker compose logs cron
```

Para evitar una salida excesiva:

```bash
docker compose logs --tail=100 app
```

y para observarlos en tiempo real:

```bash
docker compose logs -f app
```

---

## 13. Estado de Nextcloud

Una comprobación rápida del estado de la aplicación puede realizarse mediante:

```bash
cd /srv/docker/nextcloud

docker compose exec -u www-data app \
  php occ status
```

Un estado normal debe mostrar, entre otros aspectos:

```text
installed: true
maintenance: false
needsDbUpgrade: false
```

Tras una actualización es especialmente importante revisar:

```text
needsDbUpgrade
```

y confirmar que Nextcloud no haya quedado accidentalmente en modo mantenimiento.

---

## 14. Comprobación de DATA

El guard creado durante la resolución de la incidencia de montaje también puede utilizarse manualmente:

```bash
sudo /usr/local/sbin/check-nextcloud-data.sh
```

Comprueba:

```text
/srv/storage montado
        │
        ▼
UUID correcto
        │
        ▼
nextcloud-data existe
        │
        ▼
.ncdata válida
```

Para comprobar además qué filesystem está viendo el contenedor:

```bash
stat -c '%d' \
  /srv/storage/nextcloud-data
```

y:

```bash
cd /srv/docker/nextcloud

docker compose exec -T app \
  stat -c '%d' /var/www/html/data
```

Los valores deben corresponder al mismo filesystem.

No se documenta un valor numérico fijo porque no se utiliza como identificador persistente.

---

## 15. Espacio disponible

La falta de espacio puede provocar problemas tanto en Nextcloud como en Borg.

Una comprobación rápida:

```bash
df -h
```

y específicamente:

```bash
df -h / /srv/storage /srv/backup
```

permite revisar los tres roles principales:

```text
SYSTEM
DATA
BACKUP
```

El script de backup incorpora además su propio margen mínimo para BACKUP, actualmente establecido en 20 GiB.

La capacidad debe observarse como una tendencia. No es necesario esperar a que un filesystem alcance el 100 % para ampliar o limpiar almacenamiento.

---

## 16. Alertas automáticas

Las tareas críticas utilizan:

```ini
OnFailure=hpserver-alert@%n.service
```

Actualmente se aplica a servicios relevantes como:

```text
nextcloud.service
nextcloud-backup.service
borg-check.service
borg-verify-data.service
smart-check.service
```

Cuando una unidad falla, se ejecuta el servicio de alerta.

El script:

```text
/usr/local/sbin/send-systemd-alert.sh
```

recopila información sobre:

- unidad que ha fallado;
- estado de systemd;
- eventos recientes del journal.

Después envía el informe mediante SMTP.

El funcionamiento puede representarse como:

```text
servicio programado
       │
       ├── SUCCESS
       │      └── no requiere intervención
       │
       └── FAILURE
              │
              ▼
          OnFailure
              │
              ▼
      hpserver-alert
              │
              ▼
          correo SMTP
```

---

## 17. Correo administrativo

El envío desde Debian utiliza `msmtp`.

La contraseña SMTP no se almacena directamente en el script ni en la documentación.

La configuración utiliza un archivo protegido y un mecanismo basado en:

```text
passwordeval
```

Los permisos se restringen a `root`.

Durante la implementación se comprobó una diferencia importante:

```text
SMTP 250 Accepted
       ≠
mensaje entregado al destinatario
```

La aceptación por el relay únicamente confirma que éste ha aceptado el mensaje para procesarlo.

La entrega final debe verificarse mediante el proveedor de correo y, cuando sea necesario, mediante sus registros de entrega.

---

## 18. Fallo del propio sistema de alertas

Una alerta por correo tampoco debe considerarse infalible.

Puede fallar por:

- pérdida de Internet;
- fallo DNS;
- credenciales SMTP;
- caída del proveedor;
- problemas de autenticación del dominio.

Por ello:

```text
email = aviso
logs  = fuente de diagnóstico
```

y no:

```text
si no llegó email → todo funciona
```

La revisión periódica de timers y logs sigue siendo necesaria.

---

## 19. Actualizaciones de Debian

Las actualizaciones del sistema deben realizarse periódicamente:

```bash
sudo apt update
sudo apt upgrade
```

Antes de aceptar cambios importantes conviene revisar qué paquetes van a modificarse.

Después:

```bash
sudo apt autoremove
```

puede utilizarse cuando corresponda, revisando siempre la lista antes de confirmar.

Si se actualizan componentes como:

```text
kernel
Docker
systemd
```

es recomendable realizar posteriormente un reinicio controlado y comprobar el arranque completo del servidor.

---

## 20. Actualizaciones Docker

Las imágenes de servicios con estado se mantienen con versiones controladas siempre que resulte apropiado.

Por ejemplo, PostgreSQL utiliza una versión principal explícita en lugar de depender de una etiqueta completamente flotante.

Esto surgió de una incidencia real en la que una actualización de PostgreSQL introdujo un cambio en la estructura esperada del volumen.

La regla general es:

```text
backup
   │
   ▼
revisar release
   │
   ▼
actualizar imagen
   │
   ▼
recrear
   │
   ▼
comprobar logs
   │
   ▼
comprobar aplicación
```

No se actualizan automáticamente servicios críticos sin conocer previamente los cambios relevantes.

---

## 21. Actualización de Nextcloud

Antes de actualizar Nextcloud debe existir un backup reciente y válido.

Procedimiento general:

```text
1. Comprobar backup reciente
2. Revisar versión actual
3. Revisar versión objetivo
4. Consultar notas de actualización
5. Actualizar imagen
6. Recrear contenedores
7. Ejecutar migraciones si son necesarias
8. Comprobar occ status
9. Revisar logs
10. Probar acceso LAN
11. Probar acceso remoto
12. Probar operaciones básicas
```

No debe utilizarse este esquema para saltar arbitrariamente varias versiones principales.

Las restricciones y procedimientos de actualización deben comprobarse para la versión concreta de Nextcloud.

---

## 22. Actualización de PostgreSQL

PostgreSQL merece un tratamiento especialmente conservador.

Cambiar:

```text
18.x → otra 18.x
```

no equivale a cambiar:

```text
18 → 19
```

Una actualización de versión principal puede requerir migración explícita de los datos.

Por ello nunca debe modificarse simplemente:

```yaml
image: postgres:18-alpine
```

a una nueva versión principal y asumir que el volumen existente será compatible.

Antes de una migración principal debe existir:

- backup Borg válido;
- dump PostgreSQL válido;
- procedimiento de migración conocido;
- posibilidad de volver al estado anterior.

---

## 23. Reinicio periódico de validación

Un servidor que nunca se reinicia puede ocultar problemas de arranque.

La incidencia de DATA es un buen ejemplo: el servicio podía funcionar después de una reparación manual, pero eso no demostraba que el siguiente boot fuese correcto.

Después de cambios importantes en:

```text
fstab
systemd
Docker
red
mounts
```

debe realizarse un reinicio controlado cuando sea apropiado.

Después del reinicio:

```bash
findmnt /srv/storage
```

```bash
systemctl status docker --no-pager
```

```bash
systemctl status nextcloud.service --no-pager
```

```bash
cd /srv/docker/nextcloud
docker compose ps
```

```bash
docker compose exec -T app \
  findmnt -T /var/www/html/data
```

```bash
docker compose exec -u www-data app \
  php occ status
```

El objetivo es comprobar la **capacidad de recuperación automática del servicio**, no sólo su funcionamiento después de intervención manual.

---

## 24. Validación de unidades systemd

Los archivos systemd deben comprobarse antes de reiniciar.

Después de editar una unidad:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/NOMBRE.service
```

y posteriormente:

```bash
sudo systemctl daemon-reload
```

Esta práctica se incorporó después de detectar un error aparentemente trivial:

```ini
[unit]
```

en lugar de:

```ini
[Unit]
```

Los nombres de sección son sensibles a mayúsculas/minúsculas y systemd ignoró la sección incorrecta.

`systemd-analyze verify` permitió detectar el problema antes de utilizar la configuración en un reinicio real.

---

## 25. Rutina de mantenimiento

Con la automatización actual, la intervención manual puede mantenerse relativamente pequeña.

### Semanal

Comprobar que el servidor continúa accesible y revisar si se ha recibido alguna alerta.

Opcionalmente:

```bash
systemctl list-timers --all
```

No es necesario estudiar todos los logs semanalmente si no existen síntomas o alertas.

### Mensual

Realizar una revisión algo más amplia:

```text
□ timers activos
□ último backup correcto
□ último borg check correcto
□ SMART correcto
□ espacio SYSTEM
□ espacio DATA
□ espacio BACKUP
□ docker compose ps
□ occ status
□ actualizaciones Debian
□ actualizaciones relevantes de Nextcloud/Docker
```

### Trimestral

Además:

```text
□ verificar borg-verify-data
□ revisar evolución SMART
□ revisar crecimiento de DATA
□ revisar crecimiento de BACKUP
□ comprobar recovery key externa
□ revisar documentación de recuperación
```

### Periódicamente

Realizar una **restauración de prueba**.

No es necesario esperar a que ocurra una emergencia para descubrir si el procedimiento de recuperación sigue funcionando.

---

## 26. Checklist mensual rápido

Una revisión normal puede realizarse con pocas órdenes.

### Timers

```bash
systemctl list-timers --all
```

### Espacio

```bash
df -h / /srv/storage /srv/backup
```

### Docker

```bash
cd /srv/docker/nextcloud
docker compose ps
```

### Nextcloud

```bash
docker compose exec -u www-data app \
  php occ status
```

### DATA

```bash
sudo /usr/local/sbin/check-nextcloud-data.sh
```

### Último backup

```bash
sudo tail -n 50 \
  /var/log/nextcloud-backup/backup.log
```

Además deben revisarse los últimos resultados de SMART y Borg mediante sus correspondientes unidades o logs.

---

## 27. Principio de mantenimiento conservador

Una característica deliberada de `hpserver` es separar:

```text
COMPROBACIÓN AUTOMÁTICA
          │
          ▼
       detectar
          │
          ▼
       alertar
```

de:

```text
REPARACIÓN
    │
    ▼
diagnosticar
    │
    ▼
backup / precauciones
    │
    ▼
intervención manual
```

No se automatizan acciones destructivas o difíciles de revertir simplemente porque una comprobación detecte un problema.

Ejemplos:

```text
SMART detecta fallo
→ no formatea ni sustituye automáticamente nada

Borg detecta corrupción
→ no ejecuta --repair automáticamente

Nextcloud falla
→ no elimina ni recrea volúmenes automáticamente

Filesystem incorrecto
→ el backup aborta en lugar de intentar corregir mounts
```

Este comportamiento prioriza la conservación de los datos frente a la disponibilidad inmediata.

---

## 28. Estado actual de monitorización

El sistema dispone actualmente de:

```text
✓ backup Borg diario
✓ validación DATA antes del backup
✓ validación BACKUP antes del backup
✓ Borg check semanal
✓ verificación profunda trimestral
✓ SMART mensual
✓ systemd timers
✓ logs persistentes de tareas
✓ logrotate
✓ alertas SMTP
✓ comprobación de Nextcloud mediante OCC
✓ healthchecks Docker para PostgreSQL y Redis
✓ dependencias systemd de montaje
✓ validación de unidades con systemd-analyze
✓ procedimientos de revisión manual
```

Para un servidor doméstico de estas características, esto proporciona un equilibrio razonable entre supervisión, complejidad y mantenimiento.

No se pretende sustituir una plataforma completa de observabilidad.

Si en el futuro aumentan el número de servicios o la criticidad del servidor, podría incorporarse monitorización centralizada de métricas, disponibilidad y logs.

---

## 29. Resultado

La monitorización de `hpserver` se ha diseñado siguiendo tres principios:

### Prevenir

Evitar que tareas críticas operen sobre estados incorrectos.

### Detectar

Utilizar SMART, Borg, systemd, Docker, Nextcloud y logs para identificar anomalías.

### Recuperar

Mantener backups y procedimientos independientes de los sistemas monitorizados.

```text
PREVENIR
   │
   ▼
DETECTAR
   │
   ▼
ALERTAR
   │
   ▼
DIAGNOSTICAR
   │
   ▼
RECUPERAR
```

Esto convierte el mantenimiento en un proceso periódico y verificable en lugar de depender de descubrir los problemas cuando un usuario intenta acceder al servicio.

---

## Referencias

- Debian Administrator's Handbook:  
  https://www.debian.org/doc/manuals/debian-handbook/

- systemd — documentación:  
  https://systemd.io/

- systemd.timer:  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html

- systemd.service:  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html

- systemd-analyze:  
  https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html

- Docker Engine:  
  https://docs.docker.com/engine/

- Docker Compose:  
  https://docs.docker.com/compose/

- Nextcloud Administration Manual:  
  https://docs.nextcloud.com/server/latest/admin_manual/

- Nextcloud `occ`:  
  https://docs.nextcloud.com/server/latest/admin_manual/occ_command.html

- BorgBackup:  
  https://www.borgbackup.org/

- BorgBackup documentation:  
  https://borgbackup.readthedocs.io/

- smartmontools:  
  https://www.smartmontools.org/

- PostgreSQL Backup and Restore:  
  https://www.postgresql.org/docs/current/backup.html

- msmtp:  
  https://marlam.de/msmtp/