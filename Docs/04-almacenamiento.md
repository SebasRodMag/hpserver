# 04 - Almacenamiento

## 1. Objetivo

El almacenamiento de `hpserver` está diseñado siguiendo una separación clara entre tres funciones:

```text
SYSTEM
   │
   └── sistema operativo y servicios

DATA
   │
   └── archivos de los usuarios

BACKUP
   │
   └── copias de seguridad
```

Cada función utiliza un dispositivo físico diferente.

El objetivo de esta organización es facilitar:

- la administración;
- la sustitución independiente de dispositivos;
- la recuperación ante fallos;
- la separación entre datos originales y copias de seguridad;
- la identificación inequívoca de cada almacenamiento;
- la prevención de escrituras accidentales sobre dispositivos incorrectos.

> [!NOTE]
> Los UUID, números de serie y otros identificadores únicos de la instalación real no se incluyen en este repositorio público. Se utilizan identificadores como `<SYSTEM_UUID>`, `<DATA_UUID>` y `<BACKUP_UUID>`.

---

## 2. Distribución general

La arquitectura de almacenamiento es:

```text
                           hpserver
                              │
              ┌───────────────┼───────────────┐
              │               │               │
          SSD SYSTEM       HDD DATA       HDD BACKUP
           240 GB           500 GB          500 GB
              │               │               │
              │               │               │
              ▼               ▼               ▼
              /         /srv/storage     /srv/backup
                              │               │
                              ▼               ▼
                     nextcloud-data       borg/
```

Los tres almacenamientos cumplen responsabilidades diferentes.

| Rol | Punto de montaje | Contenido principal |
| --- | --- | --- |
| SYSTEM | `/` | Debian, Docker, configuración y servicios |
| DATA | `/srv/storage` | Datos de usuarios de Nextcloud |
| BACKUP | `/srv/backup` | Repositorio BorgBackup |

Esta separación física no proporciona alta disponibilidad, pero evita utilizar un único dispositivo para sistema, datos y copias.

---

## 3. Identificación de dispositivos

Linux asigna nombres de dispositivo como:

```text
/dev/sda
/dev/sdb
/dev/sdc
```

Estos nombres describen cómo ha detectado el kernel los dispositivos durante una ejecución concreta.

No deben considerarse identificadores permanentes.

Un cambio de hardware, un adaptador USB diferente o incluso el orden de detección durante el arranque puede provocar que un dispositivo anteriormente identificado como `/dev/sdc` aparezca posteriormente con otro nombre.

Por este motivo, `hpserver` utiliza tres niveles de identificación:

```text
Rol lógico
   │
   ├── SYSTEM
   ├── DATA
   └── BACKUP

Identidad persistente
   │
   └── UUID

Estado actual
   │
   └── /dev/sdX
```

Los nombres `/dev/sdX` siguen siendo útiles para operaciones administrativas y herramientas como `smartctl`, `parted` o `lsblk`, pero antes de realizar cualquier operación destructiva debe verificarse la identidad real del dispositivo.

Una comprobación habitual es:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,UUID,MOUNTPOINTS
```

También pueden utilizarse:

```bash
sudo blkid
```

y:

```bash
sudo smartctl -i /dev/sdX
```

> [!WARNING]
> Nunca debe ejecutarse `mkfs`, `parted`, `wipefs` u otra operación destructiva basándose únicamente en que un disco aparezca como `/dev/sdb` o `/dev/sdc`.

---

## 4. SSD SYSTEM

El SSD contiene el sistema operativo y los componentes necesarios para ejecutar los servicios.

Su esquema general es:

```text
SSD SYSTEM
│
├── EFI
│
├── /
│   ├── Debian
│   ├── Docker
│   ├── Nextcloud
│   ├── PostgreSQL
│   ├── scripts
│   ├── configuración
│   └── logs
│
└── swap
```

La instalación actual utiliza una estructura equivalente a:

| Partición | Sistema | Función |
| --- | --- | --- |
| EFI | FAT | Arranque UEFI |
| SYSTEM | ext4 | Sistema raíz `/` |
| SWAP | swap | Área de intercambio |

Las capacidades exactas pueden variar en una futura reconstrucción del servidor y no forman parte de los requisitos lógicos del proyecto.

Lo importante para la arquitectura es que:

```text
SSD SYSTEM ≠ DATA ≠ BACKUP
```

Los archivos personales de los usuarios no dependen directamente del SSD como almacenamiento principal.

---

## 5. HDD DATA

El HDD DATA está dedicado al almacenamiento de los archivos gestionados por Nextcloud.

Su punto de montaje es:

```text
/srv/storage
```

y el directorio de datos:

```text
/srv/storage/nextcloud-data
```

La estructura simplificada es:

```text
/srv/storage/
└── nextcloud-data/
    ├── <usuario>/
    │   └── files/
    ├── appdata_*/
    └── ...
```

> [!IMPORTANT]
> El directorio `nextcloud-data` es administrado por Nextcloud. Los archivos no deben modificarse manualmente desde el sistema de archivos durante el funcionamiento normal de la aplicación.

---

## 6. Preparación del HDD DATA

El HDD utilizado para DATA contenía inicialmente un sistema de archivos NTFS procedente de su uso anterior.

Antes de incorporarlo al servidor se decidió reutilizarlo exclusivamente para Nextcloud.

El proceso conceptual fue:

```text
disco anterior
     │
     ▼
identificación
     │
     ▼
comprobación SMART
     │
     ▼
eliminación de estructura anterior
     │
     ▼
particionado
     │
     ▼
ext4
     │
     ▼
montaje persistente
     │
     ▼
DATA
```

### Elección de ext4

Para DATA se utiliza `ext4`.

Al tratarse de un servidor Linux, no existe necesidad de conservar compatibilidad directa con Windows mediante NTFS.

`ext4` proporciona una solución nativa, ampliamente soportada y adecuada para almacenamiento general sobre Debian.

La utilización de un sistema de archivos Linux también permite gestionar directamente propietarios y permisos POSIX requeridos por Nextcloud.

---

## 7. Punto de montaje DATA

El punto de montaje se creó bajo `/srv`:

```bash
sudo mkdir -p /srv/storage
```

`/srv` se utiliza para datos proporcionados por servicios del sistema, lo que permite mantener el almacenamiento de Nextcloud separado de los directorios personales y de la estructura interna de Docker.

El dispositivo DATA se monta en:

```text
/srv/storage
```

y Nextcloud utiliza:

```text
/srv/storage/nextcloud-data
```

---

## 8. Permisos de Nextcloud DATA

El contenedor de Nextcloud Apache ejecuta las operaciones sobre los datos utilizando el usuario correspondiente a `www-data`.

Por ello, el directorio de datos debe tener un propietario y permisos compatibles con el proceso de Nextcloud.

La configuración utilizada es equivalente a:

```bash
sudo chown -R www-data:www-data /srv/storage/nextcloud-data
sudo chmod 770 /srv/storage/nextcloud-data
```

El resultado esperado es similar a:

```text
drwxrwx--- www-data www-data nextcloud-data
```

Esto permite acceso al propietario y grupo correspondiente, evitando permisos globales innecesarios.

La identidad numérica del usuario debe comprobarse cuando sea necesario, especialmente al utilizar bind mounts entre host y contenedor.

Por ejemplo:

```bash
id www-data
```

### Validación

Los permisos pueden inspeccionarse mediante:

```bash
ls -ld /srv/storage/nextcloud-data
```

La validación definitiva no consiste únicamente en comprobar permisos Unix.

También debe verificarse funcionalmente que Nextcloud puede:

```text
crear
leer
modificar
eliminar
```

archivos dentro de DATA.

---

## 9. Montaje persistente mediante UUID

Los dispositivos DATA y BACKUP se montan mediante UUID utilizando `/etc/fstab`.

El UUID puede obtenerse mediante:

```bash
sudo blkid
```

o:

```bash
lsblk -f
```

Una entrada pública equivalente para DATA sería:

```fstab
UUID=<DATA_UUID> /srv/storage ext4 defaults 0 2
```

Esto establece:

```text
<DATA_UUID>
     │
     ▼
/srv/storage
```

independientemente del nombre `/dev/sdX` asignado durante el arranque.

### Validación de `fstab`

Después de modificar `/etc/fstab`, debe comprobarse la configuración antes de considerar terminada la operación.

Por ejemplo:

```bash
sudo mount -a
```

y posteriormente:

```bash
findmnt /srv/storage
```

```bash
df -h /srv/storage
```

```bash
lsblk -f
```

Finalmente se realiza una prueba después de un reinicio completo.

---

## 10. HDD BACKUP

El HDD BACKUP está dedicado exclusivamente al repositorio Borg.

Su estructura lógica es:

```text
HDD BACKUP
     │
     ▼
/srv/backup
     │
     └── borg/
```

A diferencia de DATA, este disco no contiene archivos necesarios para ejecutar Nextcloud durante el funcionamiento normal.

Si BACKUP no está disponible:

```text
Nextcloud → puede continuar funcionando

Backup    → no debe ejecutarse
```

Esta distinción será importante para los mecanismos de protección del proceso de copia.

---

## 11. Preparación del HDD BACKUP

El disco externo había tenido usos anteriores y presentaba una estructura de particiones que no era necesaria para `hpserver`.

Antes de reutilizarlo se realizaron:

- identificación del dispositivo;
- comprobaciones SMART;
- pruebas de lectura;
- revisión de errores del kernel;
- confirmación explícita antes de cualquier operación destructiva.

Una vez considerado adecuado para su nueva función, se eliminó la estructura anterior y se creó una tabla GPT con una única partición alineada.

Un procedimiento equivalente es:

```bash
sudo parted /dev/sdX --script mklabel gpt
sudo parted /dev/sdX --script mkpart primary ext4 1MiB 100%
sudo partprobe /dev/sdX
```

La alineación puede comprobarse mediante:

```bash
sudo parted /dev/sdX align-check optimal 1
```

Posteriormente se crea el sistema de archivos:

```bash
sudo mkfs.ext4 -L BACKUP /dev/sdX1
```

> [!WARNING]
> Los comandos anteriores destruyen la estructura previa del dispositivo. `/dev/sdX` es deliberadamente un placeholder y debe sustituirse únicamente después de identificar inequívocamente el disco.

---

## 12. Incidencia: `/dev/sdc/`

Durante la preparación del disco BACKUP se introdujo accidentalmente:

```bash
sudo parted /dev/sdc/ print
```

en lugar de:

```bash
sudo parted /dev/sdc print
```

La barra final hace que la ruta parezca un directorio y produjo un error equivalente a:

```text
No es un directorio
```

No se modificaron datos.

### Lección

Los nombres de dispositivos de bloque representan archivos especiales:

```text
/dev/sdc
```

no directorios:

```text
/dev/sdc/
```

Aunque fue un error menor, se conserva en la documentación porque las operaciones sobre almacenamiento requieren prestar especial atención a las rutas utilizadas.

---

## 13. Punto de montaje BACKUP

El punto de montaje se creó mediante:

```bash
sudo mkdir -p /srv/backup
```

Una entrada pública equivalente en `/etc/fstab` es:

```fstab
UUID=<BACKUP_UUID> /srv/backup ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2
```

### `nofail`

El disco BACKUP es externo.

La opción:

```text
nofail
```

permite que la ausencia del dispositivo no impida por sí sola completar el arranque del servidor.

Esto es apropiado porque Nextcloud puede funcionar sin el disco de backup.

### `x-systemd.device-timeout`

Se establece un tiempo limitado para esperar la aparición del dispositivo durante el arranque.

Esto evita mantener innecesariamente el proceso de arranque esperando un disco externo ausente.

La ausencia de BACKUP debe afectar al proceso de backup, no convertir necesariamente el servidor Nextcloud completo en indisponible.

---

## 14. Riesgo del punto de montaje vacío

Existe un riesgo importante al utilizar un disco externo montado sobre un directorio.

Supongamos:

```text
/srv/backup
```

Cuando el HDD está montado:

```text
/srv/backup
     │
     ▼
HDD BACKUP
```

Pero si el disco no está montado, el directorio sigue existiendo:

```text
/srv/backup
     │
     ▼
directorio del SSD SYSTEM
```

Un script que sólo comprobase:

```bash
test -d /srv/backup
```

podría considerar erróneamente que el backup está disponible.

El resultado podría ser:

```text
HDD BACKUP ausente
        │
        ▼
script escribe /srv/backup
        │
        ▼
datos escritos en SSD SYSTEM
        │
        ▼
consumo inesperado de espacio
        │
        ▼
posible llenado de /
```

Por este motivo, la existencia del directorio **no demuestra que BACKUP esté montado**.

---

## 15. Verificación del dispositivo BACKUP

Antes de ejecutar Borg, el script de backup verifica que `/srv/backup` corresponde realmente al sistema de archivos esperado.

Conceptualmente:

```text
¿existe /srv/backup?
        │
        ▼
¿está montado?
        │
        ▼
¿su UUID es BACKUP_UUID?
        │
     ┌──┴──┐
     │     │
    Sí     No
     │     │
     ▼     ▼
   Borg   ABORTAR
```

Puede obtenerse información del sistema montado mediante `findmnt`.

Por ejemplo:

```bash
findmnt -no UUID /srv/backup
```

El script compara el resultado con el UUID esperado.

Una implementación simplificada sería:

```bash
EXPECTED_UUID="<BACKUP_UUID>"
MOUNTED_UUID="$(findmnt -no UUID /srv/backup)"

if [[ "$MOUNTED_UUID" != "$EXPECTED_UUID" ]]; then
    echo "ERROR: el dispositivo BACKUP correcto no está montado"
    exit 1
fi
```

La implementación completa se documenta en el capítulo dedicado a BorgBackup.

### Principio

> **Un punto de montaje identifica dónde debe aparecer un almacenamiento; no demuestra por sí mismo que dicho almacenamiento esté presente.**

---

## 16. Validación del almacenamiento

Después de preparar un dispositivo se realizan varias comprobaciones.

### Estructura de bloques

```bash
lsblk -f
```

### Identificadores

```bash
sudo blkid
```

### Montaje

```bash
findmnt /srv/storage
```

o:

```bash
findmnt /srv/backup
```

### Capacidad

```bash
df -h /srv/storage
df -h /srv/backup
```

### Escritura

Cuando sea apropiado, puede crearse temporalmente un archivo:

```bash
sudo touch /srv/backup/.write-test
sudo rm /srv/backup/.write-test
```

### Persistencia

Finalmente:

```bash
sudo reboot
```

y se repiten las comprobaciones.

La validación después del reinicio es especialmente importante para comprobar `/etc/fstab`.

---

## 17. Verificación funcional de DATA

Después de configurar Nextcloud se realizó una comprobación adicional para confirmar que los archivos cargados realmente terminaban en el HDD DATA.

El procedimiento conceptual fue:

```text
Cliente Nextcloud
       │
       │ upload
       ▼
Nextcloud
       │
       ▼
/srv/storage/nextcloud-data/
       │
       ▼
HDD DATA
```

Se cargó un archivo de prueba desde Nextcloud y posteriormente se comprobó su existencia física dentro del directorio de archivos del usuario.

También se verificó mediante `df` y `findmnt` que `/srv/storage` correspondía al HDD DATA.

Esta prueba permite descartar un error especialmente peligroso: creer que Nextcloud utiliza el HDD dedicado cuando en realidad está escribiendo sobre el SSD SYSTEM.

---

## 18. Espacio disponible y backups

El script de Borg comprueba también el espacio libre disponible en BACKUP antes de poner Nextcloud en modo mantenimiento.

Esto evita interrumpir temporalmente el servicio para descubrir posteriormente que no existe capacidad suficiente para iniciar la copia.

El flujo es:

```text
verificar BACKUP
       │
       ▼
comprobar espacio
       │
       ▼
¿espacio suficiente?
    ┌──┴──┐
    │     │
   Sí     No
    │     │
    ▼     ▼
backup   abortar
```

Actualmente se exige un margen mínimo antes de comenzar.

La comprobación utiliza la capacidad disponible del sistema de archivos y no una estimación basada únicamente en el tamaño del último backup.

Borg utiliza deduplicación, por lo que el espacio adicional necesario para cada archivo puede variar según los cambios realizados desde las copias anteriores.

---

## 19. Qué se almacena en cada dispositivo

La distribución puede resumirse así:

| Elemento | SYSTEM | DATA | BACKUP |
| --- | :---: | :---: | :---: |
| Debian | ✓ | | |
| Docker Engine | ✓ | | |
| Compose | ✓ | | copia |
| Nextcloud aplicación/configuración | ✓ | | copia |
| PostgreSQL en ejecución | ✓ | | |
| Dump PostgreSQL | ✓ temporal | | copia |
| Archivos de usuarios | | ✓ | copia |
| Repositorio Borg | | | ✓ |
| Scripts systemd/backup | ✓ | | copia según estrategia |

La existencia de información en varios dispositivos no implica necesariamente redundancia inmediata.

Por ejemplo, PostgreSQL se ejecuta desde SYSTEM, pero su estado recuperable se incorpora al backup mediante un dump.

---

## 20. Separación entre almacenamiento y backup

Es importante diferenciar:

```text
DATA
  │
  └── estado operativo actual

BACKUP
  │
  └── estados históricos recuperables
```

El HDD DATA puede contener la papelera y mecanismos propios de Nextcloud, pero éstos forman parte de la misma aplicación y almacenamiento operativo.

Borg añade una capa diferente.

Esta distinción quedó especialmente demostrada durante una incidencia en la que un cliente de Nextcloud eliminó y recreó automáticamente una carpeta.

Los archivos pudieron recuperarse mediante mecanismos de Nextcloud, pero el incidente confirmó que una acción autenticada de un cliente puede modificar DATA correctamente desde el punto de vista de la aplicación.

Por ello, mantener una copia histórica independiente sigue siendo necesario incluso cuando el almacenamiento físico funciona perfectamente.

Los detalles de esa incidencia se documentan en el capítulo dedicado a incidencias y lecciones.

---

## 21. Sustitución futura de discos

La utilización de UUID facilita los montajes persistentes, pero un disco nuevo tendrá un UUID diferente.

Por tanto, al sustituir DATA o BACKUP deben revisarse explícitamente:

```text
nuevo dispositivo
      │
      ▼
SMART / pruebas
      │
      ▼
particionado
      │
      ▼
sistema de archivos
      │
      ▼
nuevo UUID
      │
      ▼
/etc/fstab
      │
      ▼
montaje
      │
      ▼
datos / repositorio
      │
      ▼
validación
```

No debe intentarse hacer que el nuevo dispositivo aparezca simplemente bajo el mismo `/dev/sdX`.

La identidad persistente se actualiza de forma controlada.

Los procedimientos completos se describen en:

```text
docs/12-recuperacion-desastres.md
docs/13-migracion-hardware.md
```

---

## 22. Precauciones antes de operaciones destructivas

Las operaciones de almacenamiento constituyen una de las áreas con mayor riesgo administrativo del proyecto.

Antes de ejecutar comandos como:

```text
mkfs
parted
wipefs
dd
```

debe comprobarse como mínimo:

```text
modelo
+
capacidad
+
serial
+
particiones
+
sistema de archivos
+
punto de montaje
+
función esperada
```

Es preferible detenerse ante cualquier discrepancia antes que asumir la identidad del dispositivo.

Para migraciones se aplica además la regla general:

> **El dispositivo antiguo no debe borrarse, formatearse ni reutilizarse hasta que el nuevo almacenamiento haya sido validado, se hayan completado las comprobaciones de integridad y se haya realizado una restauración real satisfactoria.**

---

## 23. Incidencias y lecciones principales

| Incidencia / riesgo | Diagnóstico | Solución / criterio |
| --- | --- | --- |
| DATA procedía de NTFS | Sistema de archivos de uso anterior | Reutilización mediante sistema de archivos Linux |
| BACKUP contenía particiones antiguas | Estructura innecesaria para su nueva función | GPT + una partición ext4 |
| Uso accidental de `/dev/sdc/` | Dispositivo tratado sintácticamente como directorio | Utilizar `/dev/sdX` sin `/` final |
| `/dev/sdX` puede cambiar | Nombre asignado dinámicamente por Linux | Montaje mediante UUID |
| `/srv/backup` existe sin HDD | El directorio pertenece entonces a SYSTEM | Verificar montaje y UUID antes de Borg |
| Disco USB con tests SMART largos abortados | Causa exacta no demostrada | Short SMART + lectura completa + Borg verify-data |
| Riesgo de formatear disco incorrecto | Identificación insuficiente | Verificación múltiple antes de operaciones destructivas |

---

## 24. Principios aplicados

### Los nombres `/dev/sdX` no son identidades

Se utilizan para interactuar con el dispositivo actual, no como referencia persistente.

### El punto de montaje tampoco identifica el disco

`/srv/backup` puede existir aunque BACKUP esté ausente.

### UUID identifica el sistema de archivos esperado

Los montajes persistentes y las comprobaciones de seguridad utilizan UUID.

### DATA no es BACKUP

Separar físicamente los archivos del sistema no crea una copia de seguridad.

### Un backup debe estar físicamente separado del origen

El repositorio Borg se almacena en otro dispositivo.

### La presencia del backup debe verificarse antes de escribir

El script no debe confiar simplemente en que `/srv/backup` exista.

### Las operaciones destructivas requieren identificación múltiple

Nunca se formatea un dispositivo únicamente porque actualmente aparezca como `/dev/sdc`.

---

## 25. Resumen

La arquitectura de almacenamiento puede representarse finalmente como:

```text
                      hpserver
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
     SYSTEM             DATA            BACKUP
       SSD               HDD              HDD
        │                │                │
        ▼                ▼                ▼
        /          /srv/storage      /srv/backup
        │                │                │
     Debian        nextcloud-data        borg
     Docker              │                ▲
     PostgreSQL          └──── backup ────┘
```

La protección no depende únicamente de disponer de varios discos.

Depende de conocer exactamente:

```text
qué dispositivo es
       +
dónde está montado
       +
qué función cumple
       +
qué datos contiene
       +
cómo se recupera
```

La utilización de UUID, puntos de montaje separados, validaciones previas y backups independientes reduce el riesgo de errores administrativos y facilita futuras migraciones.

---

## 26. Referencias

La configuración descrita se apoya principalmente en documentación oficial de Debian, systemd, util-linux, GNU Parted, ext4 y BorgBackup.

### Debian

- **Debian Reference**
  - Sistemas de archivos.
  - Montaje de dispositivos.
  - Organización del sistema.

- **Debian Administrator's Handbook — Disks**
  - Particiones.
  - Sistemas de archivos.
  - Montajes persistentes.

### util-linux

- **mount(8) / fstab(5)**
  - Montaje de sistemas de archivos.
  - Configuración persistente.
  - Utilización de UUID.

- **findmnt(8)**
  - Consulta de sistemas de archivos montados.

- **lsblk(8)**
  - Identificación de dispositivos de bloque.

- **blkid(8)**
  - Identificación de sistemas de archivos y UUID.

### GNU Parted

- **GNU Parted Manual**
  - Creación de tablas GPT.
  - Particionado.
  - Alineación de particiones.

### ext4

- **ext4 documentation**
  - Sistema de archivos utilizado por DATA y BACKUP.

### systemd

- **systemd.mount**
- **systemd-fstab-generator**
  - Integración de `/etc/fstab` con systemd.
  - Dependencias y comportamiento durante el arranque.

### BorgBackup

- **BorgBackup Documentation 1.4**
  - Repositorios.
  - Deduplicación.
  - Integridad.
  - Verificación y restauración.

> Las referencias explican el funcionamiento de las tecnologías utilizadas. Los puntos de montaje, estrategia de separación, validaciones e incidencias corresponden al diseño e implementación de `hpserver`.