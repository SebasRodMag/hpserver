# 09 - Backup con BorgBackup

## 1. Objetivo

El sistema de backup de `hpserver` tiene como objetivo permitir la recuperación de Nextcloud ante situaciones como:

- fallo del HDD DATA;
- fallo del SSD SYSTEM;
- corrupción de archivos;
- eliminación accidental;
- errores durante una actualización;
- pérdida de la base de datos;
- necesidad de reconstruir el servidor;
- migración futura a otro hardware.

No copiamos archivos de un disco a otro, sino que empleamos un subsistema de recuperación completo: datos, configuración, base de datos, cifrado, retención, comprobaciones de integridad y pruebas reales de restauración.

La solución utiliza:

```text
BorgBackup
+
HDD BACKUP independiente
+
dump de PostgreSQL
+
modo mantenimiento de Nextcloud
+
verificación automática
+
pruebas reales de restauración
```

La filosofía principal es:

> **Un backup no se considera válido únicamente porque el comando de copia haya terminado correctamente. Debe poder verificarse y restaurarse.**

> [!NOTE]
> UUID, usuarios, contraseñas, claves, nombres de repositorio y otros identificadores específicos de la instalación se sustituyen por valores de ejemplo.

---

## 2. Arquitectura del backup

La arquitectura física es:

```text
                  hpserver
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
      SYSTEM                     DATA
       SSD                        HDD
        │                         │
        ├── Debian                └── archivos usuarios
        ├── Docker
        ├── Nextcloud
        ├── PostgreSQL
        └── configuración
                 │
                 │
                 ▼
            BorgBackup
                 │
                 ▼
              BACKUP
           HDD externo USB
```

El disco BACKUP constituye un dispositivo independiente de SYSTEM y DATA.

Esto permite sobrevivir al fallo individual de uno de los discos principales.

---

## 3. Qué debe protegerse

Una instalación Nextcloud no está formada únicamente por los archivos visibles del usuario.

Para reconstruirla son necesarios varios componentes.

En `hpserver` se protegen:

```text
/srv/storage/nextcloud-data
```

```text
/srv/docker/nextcloud/volumes/nextcloud
```

```text
/srv/docker/nextcloud/compose.yml
```

```text
/srv/docker/nextcloud/.env
```

y un dump de PostgreSQL:

```text
/var/backups/nextcloud/nextcloud.dump
```

Conceptualmente:

```text
Nextcloud recuperable
        │
        ├── DATA
        ├── aplicación/configuración
        ├── custom apps/themes
        ├── base de datos
        ├── Compose
        └── secretos/configuración local
```

---

## 4. Qué no se copia directamente

No se realiza una copia Borg del directorio activo de PostgreSQL:

```text
/srv/docker/nextcloud/volumes/postgres
```

como mecanismo principal de backup de la base de datos.

En su lugar:

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
     Borg
```

Esto evita depender de una copia de archivos internos de PostgreSQL realizada mientras el motor está funcionando.

El directorio persistente de PostgreSQL sigue siendo necesario para la operación normal, pero la estrategia de recuperación utiliza un dump lógico.

---

## 5. Disco BACKUP

El repositorio se almacena en el dispositivo identificado por el rol:

```text
BACKUP
```

y montado en:

```text
/srv/backup
```

El filesystem es:

```text
ext4
```

y se monta mediante UUID desde `/etc/fstab`.

Una entrada pública equivalente es:

```fstab
UUID=<BACKUP_UUID> /srv/backup ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2
```

El repositorio Borg se encuentra en:

```text
/srv/backup/borg
```

---

## 6. Por qué un disco independiente

Guardar el backup en el mismo disco que los datos produciría:

```text
DATA
 │
 ├── archivos
 └── backup
```

Ante un fallo físico:

```text
disco falla
    │
    ├── originales perdidos
    └── backup perdido
```

La arquitectura utilizada es:

```text
DATA HDD
   │
   │ copia
   ▼
BACKUP HDD
```

Esto protege frente al fallo individual del disco DATA.

---

## 7. Limitación: backup local

Aunque BACKUP sea un disco independiente, continúa estando físicamente cerca del servidor.

Por tanto, esta arquitectura no protege completamente frente a:

- incendio;
- robo;
- inundación;
- daño eléctrico grave simultáneo;
- pérdida física conjunta del servidor y disco BACKUP;
- ransomware o compromiso administrativo capaz de acceder también al repositorio.

El diseño actual proporciona:

```text
backup local independiente
```

pero todavía no constituye una estrategia completa:

```text
3-2-1
```

Una futura ampliación puede incorporar una segunda copia offline o fuera de la ubicación física del servidor.

---

## 8. BorgBackup

BorgBackup fue elegido porque proporciona:

- deduplicación;
- compresión;
- cifrado;
- snapshots lógicos mediante archivos;
- retención histórica;
- comprobación de integridad;
- restauración selectiva;
- automatización sencilla mediante scripts.

El repositorio utilizado es:

```text
/srv/backup/borg
```

---

## 9. Inicialización

El repositorio se inicializó mediante:

```bash
sudo borg init \
  --encryption=repokey-blake2 \
  /srv/backup/borg
```

La opción:

```text
repokey-blake2
```

mantiene la clave cifrada asociada al repositorio y protegida mediante una passphrase.

Esto significa que conservar únicamente los archivos del repositorio no es suficiente si se pierde la información necesaria para descifrarlo.

---

## 10. Cifrado

El repositorio Borg contiene información sensible.

Entre otras cosas puede contener:

```text
archivos familiares
configuración Nextcloud
base de datos
.env
credenciales de servicios
```

Por ello se utiliza cifrado.

Conceptualmente:

```text
datos originales
      │
      ▼
Borg
      │
      ├── deduplicación
      ├── compresión
      └── cifrado
              │
              ▼
         HDD BACKUP
```

Una persona que obtenga físicamente el disco BACKUP no debería poder interpretar directamente el contenido del repositorio sin las credenciales correspondientes.

---

## 11. Passphrase de Borg

Las tareas automatizadas necesitan acceder al repositorio sin intervención manual.

La passphrase se almacena localmente en:

```text
/root/.config/borg/passphrase
```

con permisos restrictivos.

Por ejemplo:

```bash
chmod 700 /root/.config/borg
chmod 600 /root/.config/borg/passphrase
```

El script utiliza:

```bash
export BORG_PASSCOMMAND="cat /root/.config/borg/passphrase"
```

Borg ejecuta ese comando cuando necesita obtener la passphrase.

---

## 12. Incidencia: `BORG_PASSCOMMAND`

En una primera versión del script automatizado no se definió:

```text
BORG_PASSCOMMAND
```

La ejecución manual podía funcionar porque la passphrase estaba disponible en el entorno o podía introducirse interactivamente.

Sin embargo, la ejecución automatizada solicitaba la passphrase.

Además:

```text
sudo
```

no debe asumirse que conservará arbitrariamente las variables del entorno del usuario.

### Solución

El propio script define explícitamente:

```bash
export BORG_PASSCOMMAND="cat /root/.config/borg/passphrase"
```

### Lección

> **Un script automatizado debe declarar todas las dependencias de entorno necesarias para ejecutarse sin una sesión interactiva.**

No debe depender accidentalmente del shell desde el que fue probado.

---

## 13. Recovery key

La passphrase no es el único elemento que debe protegerse.

La clave del repositorio se exportó mediante las herramientas de gestión de claves de Borg.

Se almacenó inicialmente en:

```text
/root/borg-recovery/
```

y posteriormente se transfirió una copia fuera del servidor.

La clave de recuperación y la passphrase se conservan fuera de `hpserver`.

### Regla fundamental

```text
repositorio BACKUP
        │
        +
recovery key
        │
        +
passphrase
```

son necesarios para garantizar la recuperación.

Guardar todos estos elementos únicamente en el mismo servidor eliminaría buena parte del valor del backup.

---

## 14. Copia externa de la clave

Después de exportar la recovery key se transfirió una copia a otro equipo.

La operación se verificó antes de eliminar cualquier copia temporal utilizada durante la transferencia.

La clave:

```text
NO debe publicarse en Git
NO debe incluirse en documentación pública
NO debe enviarse a repositorios no confiables
```

La documentación pública sólo indica el procedimiento.

---

## 15. Identificación segura del disco BACKUP

Uno de los riesgos más importantes apareció al automatizar el backup.

El punto de montaje:

```text
/srv/backup
```

es simplemente un directorio del filesystem SYSTEM cuando el disco BACKUP no está montado.

Si un script ejecutara:

```text
borg create /srv/backup/borg::...
```

sin comprobar el montaje, podría terminar escribiendo en el SSD SYSTEM.

El resultado podría ser:

```text
BACKUP no montado
       │
       ▼
/srv/backup = directorio normal
       │
       ▼
Borg escribe en SYSTEM
       │
       ▼
SSD se llena
```

Esto constituye un riesgo crítico.

---

## 16. Protección mediante UUID

Antes de ejecutar Borg, el script comprueba que:

```text
/srv/backup
```

corresponde realmente al filesystem esperado.

No basta con:

```bash
test -d /srv/backup
```

porque el directorio puede existir aunque el disco no esté montado.

La comprobación utiliza información del filesystem montado y el UUID esperado.

Conceptualmente:

```text
/srv/backup existe
       │
       ▼
¿es un mountpoint?
       │
       ▼
¿UUID == BACKUP_UUID?
       │
    ┌──┴──┐
    │     │
   Sí     No
    │     │
    ▼     ▼
backup   abortar
```

### Principio

> **Una ruta de backup no identifica por sí sola el dispositivo de backup.**

La automatización debe verificar el filesystem que existe detrás de esa ruta.

---

## 17. Por qué no utilizar `/dev/sdc`

El nombre:

```text
/dev/sdc
```

no es una identidad permanente.

Puede cambiar dependiendo de:

- orden de detección;
- dispositivos USB conectados;
- hardware añadido;
- reinicios;
- migraciones.

Por ello el script no debe considerar:

```text
/dev/sdc
```

como identidad del disco BACKUP.

Se utilizan:

```text
UUID
+
rol BACKUP
```

La documentación privada puede conservar además modelo y número de serie para identificación física.

---

## 18. Verificación de espacio libre

Antes de poner Nextcloud en modo mantenimiento, el script comprueba el espacio disponible en BACKUP.

Se estableció un umbral de seguridad equivalente a:

```text
20 GiB
```

El orden es importante:

```text
validar BACKUP
      │
      ▼
validar espacio
      │
      ▼
maintenance mode
      │
      ▼
backup
```

No:

```text
maintenance mode
      │
      ▼
descubrir que no hay espacio
```

Esto reduce el tiempo durante el que Nextcloud permanece indisponible innecesariamente.

El umbral es una protección operativa, no una predicción exacta del tamaño que ocupará el siguiente backup.

---

## 19. Script principal

El backup automatizado se realiza mediante:

```text
/usr/local/sbin/backup-nextcloud.sh
```

El script utiliza:

```bash
set -Eeuo pipefail
```

para endurecer el tratamiento de errores.

Conceptualmente ejecuta:

```text
validaciones
     │
     ▼
preparación
     │
     ▼
maintenance
     │
     ▼
PostgreSQL dump
     │
     ▼
validar dump
     │
     ▼
Borg create
     │
     ▼
Borg prune
     │
     ▼
Borg compact
     │
     ▼
restaurar estado
```

---

## 20. Estado inicial de Nextcloud

El script no asume que Nextcloud se encuentra siempre:

```text
maintenance = false
```

Antes de modificarlo, consulta el estado.

Si el administrador ya había activado mantenimiento antes del backup:

```text
maintenance = true
```

el script debe conservar ese estado al terminar.

Por tanto:

```text
estado inicial
      │
      ▼
guardar
      │
      ▼
backup
      │
      ▼
restaurar estado inicial
```

No simplemente:

```text
backup terminado
      │
      ▼
maintenance off
```

---

## 21. Estado del contenedor Cron

La misma filosofía se aplica al contenedor Cron.

Durante las operaciones que requieren consistencia, el script controla su estado.

Antes de modificarlo registra si estaba ejecutándose.

Al terminar, restaura el estado apropiado.

Esto evita:

```text
script
  │
  ▼
cambia estado del sistema
  │
  ▼
no lo restaura correctamente
```

### Principio

> **La automatización debe dejar el sistema en un estado coherente y, cuando sea posible, respetar el estado existente antes de comenzar.**

---

## 22. `trap` de limpieza

El script utiliza un mecanismo de limpieza para que una interrupción o error intermedio no deje innecesariamente Nextcloud en modo mantenimiento.

Conceptualmente:

```text
script inicia
    │
    ▼
registra cleanup
    │
    ▼
operaciones
    │
 ┌──┴──────────┐
 │             │
éxito         error
 │             │
 └──────┬──────┘
        ▼
     cleanup
        │
        ▼
restaurar estado
```

Esto resulta especialmente importante con:

```bash
set -e
```

porque cualquier error puede interrumpir el flujo normal del script.

---

## 23. Dump de PostgreSQL

Antes de ejecutar Borg se crea un dump lógico de la base de datos.

El destino es:

```text
/var/backups/nextcloud/nextcloud.dump
```

Se utiliza el formato custom de PostgreSQL:

```bash
pg_dump -Fc
```

Conceptualmente:

```text
PostgreSQL
    │
    ▼
pg_dump -Fc
    │
    ▼
nextcloud.dump
```

El formato custom está pensado para utilizarse posteriormente mediante:

```text
pg_restore
```

y permite inspeccionar y seleccionar los objetos contenidos durante una restauración.

---

## 24. Ejecución dentro del contenedor PostgreSQL

Como PostgreSQL se ejecuta mediante Docker, el dump se genera utilizando las herramientas PostgreSQL compatibles con la instancia en ejecución.

Una implementación equivalente puede utilizar:

```bash
docker compose exec -T db \
  pg_dump \
  -U <POSTGRES_USER> \
  -d <POSTGRES_DB> \
  -Fc
```

redirigiendo el resultado al archivo de backup correspondiente.

La opción:

```text
-T
```

evita requerir un pseudo-TTY durante una tarea automatizada.

Las credenciales reales proceden de la configuración privada de la instalación.

---

## 25. Validación del dump

Crear un archivo:

```text
nextcloud.dump
```

no demuestra por sí mismo que sea interpretable como un dump PostgreSQL.

Después de generarlo se valida mediante:

```bash
pg_restore -l nextcloud.dump
```

El comando lista la tabla de contenidos del archivo.

Si PostgreSQL no puede interpretar el dump, el backup debe considerarse fallido.

Conceptualmente:

```text
pg_dump
   │
   ▼
archivo existe
   │
   ▼
pg_restore -l
   │
 ┌─┴─┐
 │   │
OK  ERROR
 │   │
 ▼   ▼
Borg abortar
```

---

## 26. Qué incluye Borg

El archivo creado por Borg incluye:

```text
/srv/storage/nextcloud-data
```

```text
/srv/docker/nextcloud/volumes/nextcloud
```

```text
/srv/docker/nextcloud/compose.yml
```

```text
/srv/docker/nextcloud/.env
```

```text
/var/backups/nextcloud/nextcloud.dump
```

Por tanto, un archivo Borg contiene tanto los datos del usuario como la información necesaria para reconstruir la aplicación.

---

## 27. El `.env` dentro del backup

El archivo real:

```text
.env
```

no debe publicarse en Git.

Sin embargo, se incluye deliberadamente en el backup cifrado porque contiene información necesaria para reconstruir los servicios.

La distinción es:

```text
Git público
    │
    └── .env.example

Borg cifrado
    │
    └── .env real
```

Esto permite combinar:

```text
seguridad pública
+
recuperabilidad privada
```

---

## 28. Creación del archivo Borg

La operación conceptual es:

```bash
borg create \
  --stats \
  /srv/backup/borg::nextcloud-<TIMESTAMP> \
  /srv/storage/nextcloud-data \
  /srv/docker/nextcloud/volumes/nextcloud \
  /srv/docker/nextcloud/compose.yml \
  /srv/docker/nextcloud/.env \
  /var/backups/nextcloud/nextcloud.dump
```

El nombre exacto puede incorporar fecha y hora para identificar la ejecución.

La serie automatizada utiliza el prefijo:

```text
nextcloud-
```

---

## 29. Deduplicación

Borg divide la información en chunks y reutiliza aquellos que ya existen en el repositorio.

Por tanto, si entre dos ejecuciones sólo cambia una pequeña parte de los datos:

```text
backup día 1
████████████████

backup día 2
████████████████
       ▲
       cambios
```

Borg no necesita almacenar nuevamente todos los bloques idénticos.

Esto permite mantener múltiples estados históricos sin duplicar necesariamente todo el tamaño lógico de Nextcloud en cada ejecución.

---

## 30. Baseline inicial

Después de configurar Borg se creó manualmente un primer backup completo.

Este archivo se utiliza como referencia inicial.

Posteriormente se renombró a:

```text
baseline-nextcloud-<DATE>
```

El motivo de utilizar un nombre diferente es mantenerlo fuera de la selección utilizada por la política automática de retención.

Conceptualmente:

```text
baseline-nextcloud-...
       │
       └── conservación manual

nextcloud-...
       │
       └── retención automática
```

---

## 31. Incidencia: selección mediante prefijo

Inicialmente se interpretó incorrectamente que una selección por:

```text
nextcloud-
```

no afectaría necesariamente a determinado archivo manual.

Sin embargo, cualquier archivo cuyo nombre comenzara por ese patrón podía entrar en la selección correspondiente.

La solución fue cambiar el nombre del backup de referencia a:

```text
baseline-nextcloud-...
```

### Lección

> **Las políticas de retención deben diseñarse junto con la convención de nombres de los archivos.**

Un nombre no es únicamente descriptivo: puede determinar si un archivo entra en una operación destructiva de `prune`.

---

## 32. Retención

La política automatizada conserva:

```text
7 diarios
4 semanales
6 mensuales
```

Conceptualmente:

```bash
borg prune \
  --glob-archives 'nextcloud-*' \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  /srv/backup/borg
```

El archivo:

```text
baseline-nextcloud-...
```

queda fuera del patrón:

```text
nextcloud-*
```

---

## 33. Significado de `--keep-daily 7`

Una interpretación inicial incorrecta fue pensar que:

```text
--keep-daily 7
```

significaba conservar siete archivos de cada día.

No es así.

La política selecciona archivos representativos de los periodos correspondientes.

Conceptualmente:

```text
--keep-daily 7
      │
      ▼
un backup representativo
para cada uno de los
últimos siete días aplicables
```

Esto es especialmente importante si en el futuro se ejecutaran varios backups en un mismo día.

---

## 34. `--glob-archives`

En Borg 1.4 se utiliza:

```text
--glob-archives
```

para seleccionar la serie automatizada.

Esto evita utilizar la opción histórica:

```text
--prefix
```

que está deprecada en esa rama de Borg.

El patrón:

```text
nextcloud-*
```

hace explícito qué archivos pertenecen a la política automática.

---

## 35. Precaución con `prune`

`borg prune` es una operación potencialmente destructiva.

Antes de introducir o modificar una política debe comprobarse qué archivos seleccionará.

Una buena práctica es utilizar inicialmente:

```bash
borg prune \
  --dry-run \
  --list \
  --glob-archives 'nextcloud-*' \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  /srv/backup/borg
```

y revisar el resultado.

### Principio

> **Toda política automática de eliminación debe probarse primero sin eliminar.**

---

## 36. `borg compact`

Después de `prune`, Borg puede necesitar recuperar físicamente espacio del repositorio.

Por ello el flujo automatizado incluye:

```text
borg prune
    │
    ▼
borg compact
```

La distinción es:

```text
prune
  │
  └── determina qué archivos dejan de conservarse

compact
  │
  └── recupera espacio del repositorio que ya no es necesario
```

Ambas operaciones cumplen funciones diferentes.

---

## 37. Primer backup

El primer backup procesó aproximadamente:

```text
11.6 GB
```

de información original y almacenó aproximadamente:

```text
11.1 GB
```

después de compresión.

Estas cifras pertenecen al estado concreto de la instalación en ese momento y no deben interpretarse como requisitos de capacidad para reproducir el proyecto.

Su utilidad principal fue confirmar que:

```text
DATA
+
Nextcloud
+
DB
+
configuración
```

estaban entrando realmente en el repositorio.

---

## 38. Listado de archivos

Los archivos existentes pueden inspeccionarse mediante las herramientas de Borg.

En Borg 1.x:

```bash
borg list /srv/backup/borg
```

Esto permite comprobar:

- nombres;
- fechas;
- archivos históricos;
- baseline.

La existencia en el listado constituye una primera comprobación, pero no una prueba suficiente de restauración.

---

## 39. Prueba real de restauración

Después del primer backup se realizó una restauración controlada.

No se sustituyó la instalación activa.

Se utilizó un directorio temporal y se extrajeron elementos concretos del archivo Borg.

Se recuperaron al menos:

```text
compose.yml
.env
nextcloud.dump
```

El objetivo era comprobar que los datos podían salir realmente del repositorio.

---

## 40. Comparación mediante SHA-256

Los archivos restaurados:

```text
compose.yml
.env
nextcloud.dump
```

se compararon con los originales mediante hashes SHA-256.

Conceptualmente:

```text
original
   │
   ▼
SHA-256 ─────┐
             │ comparar
restore      │
   │         │
   ▼         │
SHA-256 ─────┘
```

Los hashes coincidieron.

Esto confirmó que los archivos restaurados eran idénticos a los incluidos en el backup.

---

## 41. Validación del dump restaurado

Además de comparar el hash del dump, se ejecutó:

```bash
pg_restore -l nextcloud.dump
```

sobre la copia restaurada.

De esta forma se validaron dos propiedades diferentes:

```text
Borg restauró el mismo archivo
            │
            +
PostgreSQL puede interpretarlo
```

Esto proporciona una comprobación más útil que limitarse a verificar que el fichero existe.

---

## 42. Incidencia: directorio temporal propiedad de root

La restauración de prueba se realizó inicialmente mediante `sudo`.

El directorio temporal quedó:

```text
root:root
```

con permisos restrictivos.

Posteriormente el usuario normal no podía entrar en él.

Se intentó conceptualmente:

```bash
sudo cd <directorio>
```

pero:

```text
cd
```

es una función interna del shell, no un ejecutable externo sobre el que `sudo` pueda cambiar el directorio de la shell actual.

### Solución

Se corrigió el propietario/permisos del directorio temporal cuando fue necesario inspeccionarlo como usuario normal.

### Lección

> **Los archivos restaurados con privilegios administrativos pueden conservar propietarios y permisos que impidan su inspección desde una cuenta normal.**

Y:

> **`sudo cd` no cambia el directorio de la shell actual.**

---

## 43. Backup automatizado con systemd

La ejecución periódica se realiza mediante:

```text
nextcloud-backup.service
```

y:

```text
nextcloud-backup.timer
```

El timer ejecuta el backup diariamente.

La hora configurada es:

```text
03:30
```

una franja prevista de baja utilización.

---

## 44. `Persistent=true`

El timer utiliza:

```ini
Persistent=true
```

Esto permite que, si el servidor estaba apagado cuando correspondía ejecutar el timer, systemd pueda ejecutar posteriormente la tarea pendiente cuando vuelva a estar disponible.

Esto es especialmente útil en un servidor doméstico que puede no permanecer encendido permanentemente.

---

## 45. Dependencia del punto de montaje

El servicio de backup requiere:

```text
/srv/backup
```

antes de ejecutar la tarea.

Además de la dependencia systemd, el propio script realiza la comprobación mediante UUID.

Esto aplica dos capas:

```text
systemd
   │
   └── mount requerido

script
   │
   └── filesystem esperado
```

### Principio

> **En operaciones destructivas o críticas, una segunda validación independiente puede ser preferible a confiar exclusivamente en una única capa.**

---

## 46. Dependencia de Docker

El backup necesita:

```text
Nextcloud
PostgreSQL
Docker Compose
```

Por ello la unidad systemd se ordena respecto al servicio Docker.

Sin embargo:

```text
docker.service activo
       │
       ≠
todos los contenedores necesariamente sanos
```

El script debe fallar claramente si no puede realizar las operaciones requeridas.

---

## 47. `OnFailure`

La unidad incluye:

```ini
OnFailure=hpserver-alert@%n.service
```

Si el backup termina en estado `failed`:

```text
nextcloud-backup.service
          │
          ▼
        failed
          │
          ▼
       OnFailure
          │
          ▼
correo administrador
```

El sistema de alertas se documenta en:

```text
docs/08-seguridad-y-correo.md
```

---

## 48. Logging

El script escribe en:

```text
/var/log/nextcloud-backup/backup.log
```

El log registra:

- inicio;
- validaciones;
- dump;
- Borg;
- retención;
- errores;
- finalización.

Inicialmente se utilizaron nombres de log con fecha.

Posteriormente se cambió a un nombre estable:

```text
backup.log
```

para facilitar la rotación mediante `logrotate`.

---

## 49. Logrotate

La configuración correspondiente se encuentra conceptualmente en:

```text
/etc/logrotate.d/nextcloud-backup
```

y utiliza un patrón equivalente a:

```text
/var/log/nextcloud-backup/*.log
```

La política configurada:

```text
weekly
rotate 12
compress
delaycompress
create 0640 root adm
```

evita que los logs crezcan indefinidamente.

---

## 50. Incidencia: logs fechados y rotación

Crear archivos como:

```text
backup-2026-09-10.log
backup-2026-09-11.log
backup-2026-09-12.log
```

introducía una retención implícita diferente a la gestionada por `logrotate`.

Se simplificó el diseño:

```text
backup.log
     │
     ▼
logrotate
     │
     ├── backup.log.1
     ├── backup.log.2.gz
     └── ...
```

### Lección

> **Si `logrotate` gestiona la retención, el programa no necesita implementar paralelamente su propia nomenclatura diaria de logs salvo que exista una razón específica.**

---

## 51. Comprobación semanal de Borg

Crear backups no demuestra que el repositorio continúe íntegro.

Por ello se añadió:

```text
/usr/local/sbin/check-borg.sh
```

ejecutado mediante:

```text
borg-check.service
borg-check.timer
```

La frecuencia es:

```text
domingo
04:30
```

El script ejecuta una comprobación equivalente a:

```bash
borg check --show-rc /srv/backup/borg
```

Antes vuelve a verificar:

```text
BACKUP montado
+
UUID correcto
+
repositorio existente
```

---

## 52. `borg check`

`borg check` comprueba la consistencia del repositorio y sus archivos.

Esta comprobación es más profunda que:

```text
borg list
```

porque no se limita a leer el listado de archivos.

La primera ejecución manual del check completo terminó correctamente.

El tiempo observado fue de varios minutos, algo esperable para un repositorio de este tamaño y hardware.

---

## 53. Nunca automatizar `--repair`

Borg dispone de opciones de reparación para determinados escenarios.

No se utiliza automáticamente:

```text
borg check --repair
```

Un fallo de integridad debe producir:

```text
detección
   │
   ▼
alerta
   │
   ▼
investigación
```

No:

```text
detección
   │
   ▼
modificación automática
del repositorio
```

### Principio

> **Una herramienta de reparación que modifica el único backup disponible no debe ejecutarse automáticamente sin diagnóstico previo.**

---

## 54. Verificación de datos

Además del check periódico se configuró una comprobación más profunda:

```text
borg check --verify-data
```

Esta operación verifica los datos almacenados y requiere leer una cantidad considerablemente mayor de información.

Por ello no se ejecuta diariamente.

La frecuencia configurada es trimestral:

```text
primer domingo de
enero
abril
julio
octubre
```

a las:

```text
06:00
```

---

## 55. Script de verificación de datos

El script es:

```text
/usr/local/sbin/check-borg-data.sh
```

y se ejecuta mediante:

```text
borg-verify-data.service
borg-verify-data.timer
```

Al igual que el resto de operaciones críticas:

```text
verifica UUID
verifica repositorio
utiliza BORG_PASSCOMMAND
no utiliza --repair
```

y dispone de:

```text
OnFailure
```

para enviar una alerta.

---

## 56. Incidencia: timer trimestral deshabilitado

Durante una revisión de los timers se descubrió que:

```text
borg-verify-data.timer
```

existía, pero se encontraba deshabilitado/inactivo.

La configuración del script era correcta, pero la tarea no se habría ejecutado automáticamente.

Se corrigió mediante la activación correspondiente.

### Lección

> **Crear un timer no significa que esté habilitado.**

Después de crear una tarea programada debe comprobarse:

```bash
systemctl list-timers
```

y no únicamente la existencia de los archivos `.service` y `.timer`.

---

## 57. Auditoría de timers

Las tareas principales son:

```text
03:30 diariamente
    │
    └── backup

domingo 04:30
    │
    └── borg check

primer sábado 05:00
    │
    └── SMART

trimestral 06:00
    │
    └── borg verify-data
```

La revisión conjunta permite detectar:

- timers deshabilitados;
- solapamientos;
- horarios incorrectos;
- tareas que nunca se ejecutarán.

---

## 58. Estado `inactive (dead)` en servicios oneshot

Los servicios utilizados para estas tareas son de tipo puntual.

Después de finalizar correctamente es normal que aparezcan como:

```text
inactive (dead)
```

Esto no significa necesariamente que hayan fallado.

Debe comprobarse:

```text
Result=success
```

o el estado de la última ejecución.

### Distinción

```text
servicio permanente
     │
     └── active mientras funciona

oneshot
     │
     └── inactive después de terminar
```

La interpretación del estado depende del tipo de unidad.

---

## 59. `daemon-reload`

Después de modificar archivos:

```text
.service
.timer
```

systemd debe volver a cargar las definiciones:

```bash
sudo systemctl daemon-reload
```

Modificar el archivo en disco no garantiza que systemd esté utilizando inmediatamente esa nueva versión.

### Lección

> **Después de modificar unidades systemd debe recargarse la configuración antes de validar el nuevo comportamiento.**

---

## 60. Verificación frente a restauración

Existen varios niveles de confianza:

```text
Nivel 1
archivo aparece en borg list

Nivel 2
borg check correcto

Nivel 3
borg check --verify-data correcto

Nivel 4
archivo puede extraerse

Nivel 5
archivo restaurado coincide

Nivel 6
aplicación puede reconstruirse
```

`hpserver` ha validado los niveles intermedios mediante extracción real de archivos y validación del dump.

El capítulo de recuperación ampliará el último nivel hasta una reconstrucción completa.

---

## 61. Backup no equivale a sincronización

Nextcloud sincroniza archivos entre dispositivos.

Esto no debe confundirse con un backup.

Si una eliminación válida se sincroniza:

```text
archivo eliminado
      │
      ▼
Nextcloud
      │
      ▼
clientes sincronizan
      │
      ▼
archivo desaparece
```

Una copia histórica Borg puede conservar un estado anterior independiente de esa sincronización.

Esto quedó especialmente claro durante la incidencia del cliente Android documentada posteriormente.

---

## 62. Protección frente a borrados

Borg conserva archivos históricos según la política de retención.

Si un archivo desaparece hoy:

```text
DATA actual
    │
    └── archivo ausente
```

puede continuar existiendo en:

```text
backup anterior
    │
    └── archivo presente
```

hasta que la política de retención elimine todos los archivos que lo contienen.

Por ello la profundidad histórica es una parte esencial de la protección.

---

## 63. Qué no protege Borg por sí solo

Borg no soluciona automáticamente:

- pérdida física simultánea de servidor y backup;
- pérdida de passphrase y recovery key;
- ausencia prolongada de backups;
- corrupción no detectada;
- scripts mal configurados;
- retención incorrecta;
- repositorio montado en el dispositivo equivocado;
- eliminación de todos los archivos mediante una cuenta administrativa comprometida;
- desastre que afecte a la misma ubicación física.

Por eso la solución combina:

```text
Borg
+
UUID
+
checks
+
restore tests
+
alertas
+
documentación
```

---

## 64. Fallos que deben abortar el backup

El script debe terminar con error ante condiciones como:

```text
BACKUP no montado
UUID incorrecto
repositorio Borg inexistente
espacio libre insuficiente
Docker no disponible
Nextcloud no administrable
pg_dump falla
pg_restore -l falla
borg create falla
borg prune falla
borg compact falla
```

Un backup parcial no debe registrarse silenciosamente como éxito.

---

## 65. Códigos de salida

Los scripts devuelven un código distinto de cero cuando detectan un fallo.

Esto permite que:

```text
script
   │
   ▼
exit != 0
   │
   ▼
systemd
   │
   ▼
FAILED
   │
   ▼
OnFailure
   │
   ▼
alerta
```

La propagación correcta del código de salida es necesaria para que el sistema de monitorización pueda detectar el problema.

---

## 66. Orden de las operaciones

El orden final sigue aproximadamente:

```text
1. validar configuración
        │
2. validar BACKUP y UUID
        │
3. validar espacio libre
        │
4. comprobar estados iniciales
        │
5. detener Cron cuando corresponda
        │
6. activar maintenance
        │
7. generar pg_dump
        │
8. validar dump
        │
9. crear archivo Borg
        │
10. aplicar retención
        │
11. compactar
        │
12. restaurar estados
        │
13. registrar resultado
```

Este orden intenta minimizar tanto el riesgo de inconsistencia como el tiempo de indisponibilidad.

---

## 67. Restauración futura

El backup está diseñado para permitir diferentes niveles de restauración.

### Archivo individual

```text
Borg archive
     │
     ▼
archivo
```

### Directorio de usuario

```text
Borg archive
     │
     ▼
DATA parcial
```

### Base de datos

```text
nextcloud.dump
      │
      ▼
pg_restore
```

### Nextcloud completo

```text
DATA
+
Nextcloud volume
+
PostgreSQL
+
Compose
+
.env
```

### Servidor completo

```text
Debian nuevo
+
Docker
+
discos
+
backup Borg
+
configuración
```

Los procedimientos completos se documentan en:

```text
docs/12-recuperacion-desastres.md
```

---

## 68. Regla para sustituir hardware

Un disco antiguo no debe:

```text
formatearse
reutilizarse
borrarse
descartarse
```

inmediatamente después de copiar sus datos a un dispositivo nuevo.

La regla de `hpserver` es:

> **No destruir, formatear ni reutilizar el hardware anterior hasta que el sistema nuevo haya arrancado, haya sido validado, se haya comprobado su integridad y se haya realizado una prueba real de restauración.**

Esta regla se aplicará a:

- sustitución de DATA;
- sustitución de BACKUP;
- sustitución de SYSTEM;
- migración completa de servidor.

---

## 69. Criterio para considerar un backup válido

Un backup satisfactorio debe cumplir como mínimo:

```text
dispositivo correcto
       │
       ▼
espacio suficiente
       │
       ▼
dump PostgreSQL válido
       │
       ▼
borg create correcto
       │
       ▼
archivo visible
       │
       ▼
checks periódicos
       │
       ▼
restore test
```

La existencia de un fichero o un mensaje:

```text
Backup completed
```

no sustituye estas comprobaciones.

---

## 70. Mejoras futuras

La arquitectura puede ampliarse con:

### Segunda copia offline

Un segundo disco desconectado excepto durante la actualización de la copia.

### Backup off-site

Una copia cifrada almacenada fuera de la ubicación física del servidor.

### Restore test completo automatizado o semiautomatizado

Reconstruir periódicamente una instancia temporal.

### Métricas

Registrar:

```text
duración
tamaño lógico
tamaño deduplicado
espacio libre
último backup válido
último check válido
```

### Alertas externas

Detectar desde otro sistema que `hpserver` ha dejado de ejecutar backups.

Estas mejoras no invalidan la arquitectura actual, sino que aumentan su resistencia frente a escenarios adicionales.

---

## 71. Principios aplicados

### El backup vive en un dispositivo independiente

Un fallo del DATA no debe destruir también la copia.

### Se verifica el UUID

La ruta `/srv/backup` no basta para identificar el disco.

### La base de datos se exporta lógicamente

No se confía en copiar directamente el directorio activo de PostgreSQL.

### El dump se valida antes de Borg

Un archivo existente no implica un dump válido.

### El backup está cifrado

Contiene datos y configuración sensible.

### La clave de recuperación existe fuera del servidor

El backup debe poder abrirse después de perder `hpserver`.

### La retención selecciona únicamente la serie automática

El baseline queda fuera de `nextcloud-*`.

### `prune` debe probarse con `--dry-run`

Las operaciones destructivas requieren validación previa.

### `compact` y `prune` cumplen funciones diferentes

La política lógica y la recuperación física de espacio no son la misma operación.

### No se automatiza `--repair`

Los problemas de integridad deben investigarse antes de modificar el repositorio.

### Se realizan restauraciones reales

Ver un archivo en `borg list` no demuestra que pueda recuperarse correctamente.

### Los timers se auditan

Crear una tarea no demuestra que vaya a ejecutarse.

---

## 72. Incidencias y lecciones principales

| Incidencia | Causa / diagnóstico | Solución / lección |
| --- | --- | --- |
| Borg solicitaba passphrase en automatización | Faltaba `BORG_PASSCOMMAND` | Declararlo dentro del script |
| Riesgo de escribir backup en SYSTEM | `/srv/backup` existe aunque BACKUP no esté montado | Verificar mountpoint y UUID |
| Uso potencial de `/dev/sdc` como identidad | Los nombres `/dev/sdX` no son persistentes | Utilizar UUID y roles |
| Baseline podía entrar en retención | Nombre compatible con selección automática | Renombrar a `baseline-nextcloud-*` |
| Interpretación incorrecta de `--keep-daily 7` | Se confundieron días con número de archivos por día | Documentar semántica real |
| `--prefix` utilizado inicialmente | Opción deprecada en Borg 1.4 | Utilizar `--glob-archives` |
| Restore temporal inaccesible | Directorio creado como root | Ajustar propietario/permisos |
| `sudo cd` no funcionaba | `cd` es builtin del shell | Cambiar permisos o abrir shell privilegiada |
| Timer trimestral no se ejecutaría | Estaba creado pero deshabilitado | Auditar `systemctl list-timers` |
| Logs diarios complicaban rotación | Dos mecanismos de retención | Utilizar `backup.log` + logrotate |
| Riesgo de dejar mantenimiento activo | Error intermedio del script | Cleanup mediante `trap` |
| Riesgo de cambiar estado previo | Script asumía estados iniciales | Registrar y restaurar estado |
| `borg list` podía dar falsa confianza | No prueba extracción | Realizar restore test |
| Dump existente podía ser inválido | Existencia ≠ estructura PostgreSQL válida | Validar con `pg_restore -l` |

---

## 73. Resumen

La arquitectura final es:

```text
                      NEXTCLOUD
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
           DATA                    PostgreSQL
             │                         │
             │                      pg_dump
             │                         │
             │                         ▼
             │                  nextcloud.dump
             │                         │
             └────────────┬────────────┘
                          │
                    configuración
                    Nextcloud volume
                    compose.yml
                    .env
                          │
                          ▼
                     BorgBackup
                          │
               ┌──────────┴──────────┐
               │                     │
               ▼                     ▼
          deduplicación           cifrado
               │                     │
               └──────────┬──────────┘
                          ▼
                     HDD BACKUP
                          │
              ┌───────────┼───────────┐
              │           │           │
              ▼           ▼           ▼
           prune        check     verify-data
              │
              └───────────┬───────────┘
                          ▼
                    restore tests
```

La estrategia no se limita a producir copias.

Se intenta responder a cuatro preguntas diferentes:

```text
¿se creó?
    │
¿es íntegro?
    │
¿puede leerse?
    │
¿puede restaurarse?
```

Sólo cuando estas preguntas tienen una respuesta satisfactoria puede considerarse que existe una estrategia de backup útil.

---

## 74. Referencias

### BorgBackup

- **BorgBackup 1.4 documentation**  
  https://borgbackup.readthedocs.io/en/1.4-maint/

  - Documentación correspondiente a la rama utilizada.
  - Repositorios.
  - Archivos.
  - Cifrado.
  - Backup y restauración.

- **borg create**  
  https://borgbackup.readthedocs.io/en/1.4-maint/usage/create.html

  - Creación de archivos.
  - Compresión.
  - Deduplicación.
  - Estadísticas.
  - Exclusiones.

- **borg prune**  
  https://borgbackup.readthedocs.io/en/1.4-maint/usage/prune.html

  - Políticas de retención.
  - `--keep-daily`.
  - `--keep-weekly`.
  - `--keep-monthly`.
  - Selección de archivos.
  - `--glob-archives`.
  - `--dry-run`.

- **borg compact**  
  https://borgbackup.readthedocs.io/en/1.4-maint/usage/compact.html

  - Recuperación de espacio después de eliminar archivos.
  - Compactación del repositorio.

- **borg check**  
  https://borgbackup.readthedocs.io/en/1.4-maint/usage/check.html

  - Comprobación de consistencia.
  - `--verify-data`.
  - Consideraciones sobre `--repair`.

- **borg extract**  
  https://borgbackup.readthedocs.io/en/1.4-maint/usage/extract.html

  - Restauración.
  - Extracción parcial.
  - Permisos y metadatos.

- **borg key**  
  https://borgbackup.readthedocs.io/en/1.4-maint/usage/key.html

  - Exportación de claves.
  - Importación.
  - Gestión de recovery key.

- **BorgBackup FAQ — Security**  
  https://borgbackup.readthedocs.io/en/1.4-maint/faq.html#security

  - Modelo de seguridad.
  - Cifrado.
  - Consideraciones sobre repositorios.

### Nextcloud

- **Nextcloud Administration Manual — Backup**  
  https://docs.nextcloud.com/server/stable/admin_manual/maintenance/backup.html

  - Directorio de configuración.
  - Custom apps.
  - DATA.
  - Themes.
  - Base de datos.
  - Maintenance mode.
  - PostgreSQL.

- **Nextcloud Administration Manual — Restore**  
  https://docs.nextcloud.com/server/stable/admin_manual/maintenance/restore.html

  - Restauración de archivos.
  - Base de datos.
  - Configuración.
  - Maintenance mode.

### PostgreSQL

- **PostgreSQL — pg_dump**  
  https://www.postgresql.org/docs/current/app-pgdump.html

  - Backup lógico.
  - Formato custom.
  - `-Fc`.
  - Portabilidad.
  - Uso posterior mediante `pg_restore`.

- **PostgreSQL — pg_restore**  
  https://www.postgresql.org/docs/current/app-pgrestore.html

  - Restauración.
  - Formatos de archivo.
  - `--list`.
  - Selección de objetos.

### systemd

- **systemd.timer**  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html

  - Timers.
  - Activación periódica.
  - `Persistent=`.

- **systemd.service**  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html

  - Servicios oneshot.
  - Estados.
  - Ejecución de scripts.

- **systemd.unit**  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html

  - Dependencias.
  - `OnFailure=`.
  - Requisitos de montaje.

### Filesystems

- **findmnt — Debian Manpages**  
  https://manpages.debian.org/trixie/util-linux/findmnt.8.en.html

  - Identificación de filesystems.
  - Mountpoints.
  - UUID.
  - Información de dispositivos.

- **fstab — Debian Manpages**  
  https://manpages.debian.org/trixie/mount/fstab.5.en.html

  - Montaje persistente.
  - UUID.
  - Opciones de montaje.

### logrotate

- **logrotate — Debian Manpages**  
  https://manpages.debian.org/trixie/logrotate/logrotate.8.en.html

  - Rotación.
  - Compresión.
  - Retención de logs.
  - Modo debug.

> Las referencias describen el comportamiento oficial de BorgBackup, Nextcloud, PostgreSQL, systemd y las herramientas del sistema. La estrategia de backup, los scripts, las políticas de retención, las pruebas de restauración y las incidencias descritas corresponden a la implementación de `hpserver`.