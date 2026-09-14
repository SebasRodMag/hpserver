# 02 - Hardware

## 1. Objetivo

`hpserver` reutiliza un ordenador portátil como servidor doméstico.

El objetivo no es proporcionar una plataforma de alta disponibilidad ni competir con un NAS comercial, sino aprovechar hardware existente para construir un servidor con capacidad suficiente para:

- ejecutar Nextcloud y sus servicios auxiliares;
- almacenar los archivos de varios usuarios;
- realizar copias de seguridad automáticas;
- ejecutar verificaciones periódicas de integridad;
- monitorizar el estado de los dispositivos de almacenamiento;
- proporcionar acceso local y remoto de forma continuada.

La reutilización del portátil permite reducir el coste inicial y el consumo eléctrico, a cambio de aceptar algunas limitaciones propias de un equipo que no fue diseñado originalmente como servidor.

> [!NOTE]
> Por motivos de privacidad y seguridad, este documento no incluye números de serie, UUID ni otros identificadores únicos de los dispositivos.

---

## 2. Servidor

El sistema utiliza un ordenador portátil HP equipado con:

| Componente | Características |
| --- | --- |
| CPU | Intel Core i7-5500U |
| Arquitectura | x86-64 |
| Núcleos / hilos | 2 / 4 |
| Frecuencia | 2,40 GHz base / hasta 3,00 GHz |
| RAM | 12 GB |
| Almacenamiento de sistema | SSD SATA 240 GB |
| Almacenamiento DATA | HDD SATA 500 GB |
| Almacenamiento BACKUP | HDD SATA 500 GB mediante USB |
| Sistema operativo | Debian 13 (Trixie) |

El Intel Core i7-5500U pertenece a la familia Broadwell y tiene un TDP de 15 W.

Se trata de un procesador móvil y relativamente antiguo, pero la carga prevista para `hpserver` no requiere una elevada capacidad de cálculo.

La mayor parte de las operaciones habituales corresponden a almacenamiento, transferencia de archivos, consultas a PostgreSQL y tareas periódicas de mantenimiento.

Los 12 GB de memoria permiten ejecutar Debian, Docker, Nextcloud, PostgreSQL y Redis con margen suficiente para el número reducido de usuarios previsto.

---

## 3. Reutilización del portátil

La utilización de un portátil presenta algunas ventajas interesantes para un servidor doméstico.

### Consumo

El hardware móvil está diseñado originalmente para trabajar con restricciones energéticas, por lo que puede ofrecer un consumo inferior al de determinados equipos de escritorio reutilizados.

### Tamaño

El equipo integra en un único chasis:

- placa base;
- procesador;
- memoria;
- pantalla;
- teclado;
- interfaz de red;
- alimentación;
- batería.

No necesita periféricos externos para realizar tareas administrativas de emergencia directamente sobre el servidor.

### Batería

La batería proporciona cierta continuidad durante interrupciones breves del suministro eléctrico.

Esto puede permitir que el servidor permanezca funcionando durante pequeños cortes o fluctuaciones.

Sin embargo:

> **La batería del portátil no debe considerarse equivalente a un UPS administrado.**

No existe actualmente un mecanismo específico que garantice un apagado controlado del servidor cuando la batería alcanza un determinado nivel crítico como parte de la arquitectura de `hpserver`.

Por tanto, la batería se considera una protección adicional frente a interrupciones breves y no un sistema completo de alimentación ininterrumpida.

### Tapa del portátil

Un comportamiento especialmente importante en este tipo de reutilización es la suspensión automática al cerrar la tapa.

En `hpserver`, `systemd-logind` está configurado para ignorar este evento.

Esto permite mantener el portátil cerrado físicamente sin detener los servicios.

La configuración concreta se documenta en el capítulo dedicado a Debian y la administración del sistema.

---

## 4. Distribución del almacenamiento

El servidor utiliza tres dispositivos físicos con funciones diferentes:

```text
                 hpserver
                    │
       ┌────────────┼────────────┐
       │            │            │
      SSD       HDD DATA     HDD BACKUP
     240 GB       500 GB       500 GB
       │            │            │
    Sistema      Archivos       Borg
    Docker       Nextcloud     Backup
    PostgreSQL
```

La separación física es deliberada.

No se pretende únicamente aumentar la capacidad disponible, sino asignar a cada dispositivo una responsabilidad claramente definida.

---

## 5. SSD de sistema

### Características

| Propiedad | Valor |
| --- | --- |
| Modelo | Kingston A400 |
| Capacidad nominal | 240 GB |
| Interfaz | SATA |
| Función | SYSTEM |

El SSD contiene principalmente:

```text
Debian
Docker
Nextcloud
PostgreSQL
Redis
configuración
scripts
systemd
logs
```

Los datos de usuario de Nextcloud se almacenan fuera de este dispositivo.

### Motivo de la elección

El SSD es el dispositivo más apropiado para las operaciones frecuentes del sistema operativo y de PostgreSQL debido a su menor latencia respecto a los HDD.

También permite que el almacenamiento principal de archivos pueda administrarse independientemente del sistema.

Una avería del SSD supondría la pérdida temporal del sistema operativo y de los servicios, pero no necesariamente la pérdida física de los archivos almacenados en DATA.

La recuperación completa seguiría necesitando la configuración y la base de datos correspondientes, por lo que esta separación **no elimina la necesidad de backup**.

---

## 6. HDD DATA

### Características

| Propiedad | Valor |
| --- | --- |
| Modelo | HGST HTS545050A7E680 |
| Familia | Travelstar Z5K500 |
| Capacidad nominal | 500 GB |
| Velocidad | 5400 rpm |
| Formato | 2,5 pulgadas |
| Interfaz | SATA |
| Función | DATA |

Este dispositivo contiene los archivos de los usuarios de Nextcloud.

El HDD se instaló utilizando la bahía destinada originalmente a la unidad óptica del portátil mediante un adaptador SATA.

De esta manera se dispone simultáneamente de:

```text
Bahía principal → SSD SYSTEM
Bahía óptica    → HDD DATA
```

### Motivo de la elección

El HDD proporciona una capacidad considerablemente superior a la necesaria actualmente para el sistema operativo y permite reservar el SSD para las cargas que se benefician más de su velocidad.

Para un servidor doméstico con pocos usuarios, la velocidad de un HDD de 5400 rpm es suficiente para el almacenamiento principal previsto.

El rendimiento máximo del disco no constituye el principal objetivo del proyecto.

### Estado inicial

Antes de utilizar el dispositivo como almacenamiento DATA se realizaron comprobaciones SMART.

En el momento de su incorporación al proyecto:

```text
SMART overall-health: PASSED

Reallocated_Sector_Ct:    0
Current_Pending_Sector:   0
Offline_Uncorrectable:    0
UDMA_CRC_Error_Count:     0
```

Los tests SMART realizados finalizaron correctamente.

El disco tenía más de 10.000 horas de funcionamiento acumuladas, por lo que no se considera un dispositivo nuevo.

También presenta un `Load_Cycle_Count` elevado, consecuencia de su historial previo de funcionamiento.

Por este motivo, su utilización se acompaña de:

- monitorización SMART periódica;
- backup independiente;
- verificaciones de integridad;
- procedimiento documentado de sustitución.

El estado `PASSED` de SMART no se interpreta como garantía de que el disco no pueda fallar.

---

## 7. HDD BACKUP

### Características

| Propiedad | Valor |
| --- | --- |
| Modelo | Seagate ST500LT012 |
| Capacidad nominal | 500 GB |
| Velocidad | 5400 rpm |
| Formato | 2,5 pulgadas |
| Conexión | SATA mediante adaptador USB |
| Función | BACKUP |

El dispositivo está conectado externamente mediante un adaptador USB-SATA basado en un controlador ASMedia.

Su única función dentro de la arquitectura es almacenar el repositorio BorgBackup.

No contiene datos necesarios para la ejecución normal de Nextcloud.

### Estado inicial

Las comprobaciones SMART mostraron:

```text
SMART overall-health: PASSED

Reallocated_Sector_Ct:     0
Reallocated_Event_Count:   0
Current_Pending_Sector:    0
Offline_Uncorrectable:     0
Reported_Uncorrect:        0
UDMA_CRC_Error_Count:      0
```

También se realizó una lectura secuencial completa del dispositivo sin errores de E/S ni incrementos en los contadores SMART críticos.

### Particularidad del adaptador USB

Durante las pruebas iniciales se intentaron ejecutar tests SMART extendidos.

En más de una ocasión estos tests fueron abortados por el host cuando habían alcanzado aproximadamente el 90 %.

Sin embargo:

- los tests cortos finalizaron correctamente;
- una lectura secuencial completa no produjo errores;
- los atributos SMART críticos permanecieron en cero;
- no aparecieron errores de E/S, desconexiones ni resets relevantes durante la lectura.

El comportamiento se considera probablemente relacionado con la combinación del puente USB-SATA, UAS o el host, aunque no se ha identificado con certeza la causa exacta.

Por este motivo no se automatizan tests SMART extendidos sobre este dispositivo.

En su lugar se utilizan:

```text
SMART short tests
        +
atributos SMART
        +
Borg check
        +
Borg --verify-data
        +
restauraciones de prueba
```

---

## 8. Identificación de los discos

Linux puede asignar nombres como:

```text
/dev/sda
/dev/sdb
/dev/sdc
```

pero estos nombres **no se consideran identificadores permanentes**.

El orden puede cambiar al modificar hardware, conectar dispositivos USB o durante determinados procesos de arranque.

Por ello, la documentación distingue los dispositivos mediante roles:

```text
SYSTEM
DATA
BACKUP
```

y los montajes persistentes utilizan UUID.

Por ejemplo:

```text
UUID=<DATA_UUID>    /srv/storage
UUID=<BACKUP_UUID>  /srv/backup
```

Los UUID reales y los números de serie se conservan únicamente en la documentación privada de recuperación.

Antes de cualquier operación destructiva debe comprobarse la identidad del dispositivo mediante varios datos, por ejemplo:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,MOUNTPOINTS
```

y, cuando corresponda:

```bash
smartctl -i /dev/sdX
```

> [!WARNING]
> Nunca debe asumirse que `/dev/sdb` o `/dev/sdc` representan siempre DATA o BACKUP.

---

## 9. Por qué no se utiliza RAID

`hpserver` no utiliza actualmente RAID.

Los dos HDD de 500 GB cumplen funciones distintas:

```text
HDD DATA
   │
   │ copia mediante Borg
   ▼
HDD BACKUP
```

No forman un RAID1.

Esto es intencionado.

RAID y backup solucionan problemas diferentes.

### RAID

Una configuración RAID1 podría mantener el servicio disponible ante el fallo físico de uno de los discos que forman el espejo.

Pero una eliminación accidental:

```text
usuario elimina archivo
        │
        ▼
      RAID1
       ├── disco A: eliminado
       └── disco B: eliminado
```

se replica inmediatamente.

Lo mismo puede ocurrir con determinadas corrupciones lógicas o modificaciones no deseadas.

### Backup

Borg mantiene archivos históricos independientes:

```text
DATA actual
    │
    ├── backup día 1
    ├── backup día 2
    ├── backup día 3
    └── ...
```

Esto permite recuperar estados anteriores.

Por tanto:

```text
RAID   → disponibilidad / tolerancia a determinados fallos físicos

BACKUP → recuperación de datos
```

Para las necesidades actuales del proyecto se ha priorizado disponer de un backup independiente antes que proporcionar redundancia del almacenamiento DATA.

La incorporación futura de RAID no eliminaría la necesidad de BorgBackup.

---

## 10. Limitaciones de la configuración física

La arquitectura actual tiene varias limitaciones conocidas.

### Hardware reutilizado

Los dispositivos no son nuevos y algunos acumulan un número significativo de horas de funcionamiento.

La estrategia no consiste en confiar en que nunca fallen, sino en asumir que **cualquier disco terminará fallando en algún momento** y disponer de mecanismos para detectarlo y recuperarse.

### Ausencia de RAID

El fallo del HDD DATA provoca indisponibilidad hasta sustituirlo y restaurar los datos.

### Backup conectado permanentemente

El HDD BACKUP protege frente a numerosos fallos lógicos y físicos de DATA, pero permanece conectado al mismo servidor.

Por tanto, no protege completamente frente a:

- robo;
- incendio;
- daños eléctricos graves;
- pérdida física simultánea del servidor y sus discos;
- determinados ataques capaces de alcanzar también el repositorio.

Una futura copia *off-site* constituiría una mejora adicional.

### Adaptador USB

El disco BACKUP depende también del puente USB-SATA y de la conexión USB.

Un fallo del adaptador puede producir síntomas similares a un fallo del propio disco, por lo que los errores deben diagnosticarse antes de concluir que el medio está dañado.

### Sin fuentes redundantes

El servidor dispone únicamente de su cargador y batería interna.

No existe redundancia de alimentación.

---

## 11. Monitorización del hardware

Los discos DATA y BACKUP se comprueban periódicamente mediante `smartctl`.

SMART permite consultar información de fiabilidad mantenida por los propios dispositivos y ejecutar diferentes tipos de self-test.

La monitorización de `hpserver` presta especial atención a:

```text
Reallocated_Sector_Ct
Reallocated_Event_Count
Current_Pending_Sector
Offline_Uncorrectable
Reported_Uncorrect
UDMA_CRC_Error_Count
```

También se comprueba:

```text
SMART overall-health
resultado del último self-test
```

Las comprobaciones se ejecutan automáticamente mediante systemd.

Si aparece una condición considerada crítica, el servicio termina con un código de error y activa el sistema general de alertas.

La monitorización SMART debe entenderse como un mecanismo de detección y diagnóstico, no como sustituto de las copias de seguridad.

---

## 12. Criterio para sustituir hardware

La sustitución de un disco no depende exclusivamente de que SMART indique `FAILED`.

También puede considerarse necesaria ante:

- aparición o incremento continuado de sectores reasignados;
- sectores pendientes;
- sectores no corregibles;
- errores de lectura/escritura;
- fallos de self-test;
- desconexiones o resets persistentes después de descartar cableado/adaptadores;
- degradación progresiva;
- comportamiento anómalo repetido.

Cuando se sustituya DATA o BACKUP se seguirá el procedimiento específico descrito en el capítulo de migración de hardware.

Se aplica siempre el siguiente principio:

> **El dispositivo antiguo no debe borrarse, formatearse ni reutilizarse hasta que el nuevo almacenamiento haya sido validado, las comprobaciones de integridad hayan finalizado correctamente y se haya realizado una restauración real de prueba.**

---

## 13. Resumen

La arquitectura física de `hpserver` utiliza hardware modesto y reutilizado, pero evita depender de un único dispositivo para todas las funciones.

```text
SSD SYSTEM
    │
    ├── Debian
    ├── Docker
    ├── Nextcloud
    └── PostgreSQL

HDD DATA
    │
    └── archivos de usuarios

HDD BACKUP
    │
    └── repositorio Borg
```

La fiabilidad del sistema no se basa en asumir que el hardware es infalible.

Se basa en combinar:

```text
separación física
      +
monitorización
      +
backup
      +
verificación
      +
restauraciones de prueba
      +
procedimientos de recuperación
```

Esta estrategia permite utilizar hardware doméstico y reutilizado manteniendo un nivel de protección adecuado para los objetivos del proyecto.

---

## 14. Referencias

La información técnica y las decisiones descritas en este capítulo se apoyan principalmente en documentación oficial de los componentes utilizados.

### Intel

- **Intel Core i7-5500U Processor — Specifications**
  - Características del procesador.
  - Número de núcleos e hilos.
  - Frecuencias de funcionamiento.
  - TDP.

### Debian / smartmontools

- **Debian — smartmontools**
  - Herramientas `smartctl` y `smartd`.
  - Monitorización de dispositivos mediante SMART.

- **smartctl(8) — Debian Manpages**
  - Consulta de información SMART.
  - Estado de salud.
  - Atributos.
  - Logs de errores.
  - Ejecución y consulta de self-tests.

### BorgBackup

- **BorgBackup Documentation**
  - Copias de seguridad deduplicadas.
  - Cifrado autenticado.
  - Creación y restauración de archivos.
  - Verificación de repositorios.

> Las referencias se mantienen preferentemente hacia documentación oficial o documentación mantenida por los propios proyectos. La configuración específica de `hpserver`, las decisiones arquitectónicas y las incidencias descritas corresponden a la implementación real del proyecto.