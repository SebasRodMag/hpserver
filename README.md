# HPServer

Servidor doméstico basado en **Debian 13** destinado al almacenamiento privado, sincronización de archivos y copias de seguridad mediante **Nextcloud**.

Este proyecto reutiliza un ordenador portátil y viejos HDD´s para convertirlos en un servidor de bajo consumo y combina contenedores Docker, almacenamiento dedicado, acceso remoto seguro, copias de seguridad cifradas y monitorización automatizada.

> **Estado del proyecto:** infraestructura base funcionando y la configuración validada.

---

## Objetivos

`hpserver` nace de la necesidad de solucionar los problemas de almacenamiento en la nube y la molestia de los avisos de Google. Con esto, conseguimos disponer de una solución de almacenamiento doméstico que permita:

- Centralizar archivos personales y familiares.
- Acceder a los datos tanto desde la red local como desde Internet.
- Sincronizar archivos desde ordenadores y dispositivos móviles.
- Mantener los datos bajo control propio utilizando principalmente software libre.
- Disponer de copias de seguridad independientes del almacenamiento principal.
- Detectar de forma automática problemas en los discos y en los procesos de backup.
- Poder reconstruir o migrar el servidor de forma documentada en caso de avería.

El proyecto busca también reutilizar hardware disponible y mantener una arquitectura suficientemente sencilla para poder administrarla sin depender de servicios cloud de almacenamiento de terceros.

---

## Arquitectura general

El servidor utiliza tres dispositivos de almacenamiento con funciones diferenciadas:

| Dispositivo | Función |
| --- | --- |
| SSD 240 GB | Sistema operativo, Docker, Nextcloud y PostgreSQL |
| HDD 500 GB interno | Datos de los usuarios de Nextcloud |
| HDD 500 GB externo | Repositorio cifrado de copias de seguridad Borg |

Los principales servicios se ejecutan mediante **Docker Compose**:

- **Nextcloud:** plataforma principal que proporciona almacenamiento, sincronización, gestión y acceso a los archivos de los usuarios.
- **PostgreSQL:** base de datos utilizada por Nextcloud para almacenar usuarios, configuración, metadatos, permisos y demás información de la aplicación.
- **Redis:** almacén de datos en memoria utilizado por Nextcloud para caché y bloqueo de archivos, reduciendo accesos innecesarios a la base de datos y evitando conflictos cuando varios procesos acceden simultáneamente a los mismos recursos.
- **Nextcloud Cron:** ejecuta periódicamente las tareas internas en segundo plano de Nextcloud, como trabajos de mantenimiento, limpieza y procesamiento de tareas pendientes.
- **Cloudflare Tunnel:** crea un túnel saliente cifrado entre el servidor y Cloudflare que permite acceder a Nextcloud desde Internet mediante HTTPS sin abrir ni redirigir puertos del router.

Un matiz importatne con **Redis.** En este caso, tiene dos funciones especiales:
```text
Redis
├── Caché
│   └── información temporal de acceso rápido
│
└── File locking
    └── coordinación de accesos concurrentes a archivos
```

El acceso remoto se realiza mediante **Cloudflare Tunnel**, evitando exponer directamente puertos del router a Internet.

```text
Internet
   │
   ▼
Cloudflare
   ▲
   │ conexión iniciada desde dentro
   │
cloudflared
   │
   ▼
Nextcloud
```

Las copias de seguridad se realizan mediante **BorgBackup**, mientras que `systemd` se encarga de programar backups, verificaciones de integridad y comprobaciones SMART.

```text
                         Internet
                            │
                    Cloudflare Tunnel
                            │
                     nube.asrm.dev
                            │
                    ┌───────▼───────┐
                    │   Nextcloud   │
                    │    Docker     │
                    └───────┬───────┘
                            │
                ┌───────────┴───────────┐
                │                       │
          PostgreSQL / Redis          DATA
                SSD                HDD interno
                                    /srv/storage
                                         │
                                         │ BorgBackup
                                         ▼
                                      BACKUP
                                   HDD externo
                                   /srv/backup
```

---

## Tecnologías principales

| Área | Tecnología | Función |
| --- | --- | --- |
| Sistema operativo | Debian 13 (Trixie) | Sistema base del servidor |
| Contenedores | Docker Engine + Docker Compose | Aislamiento y gestión de los servicios |
| Cloud privado | Nextcloud | Almacenamiento, sincronización y acceso a archivos |
| Base de datos | PostgreSQL 18 | Persistencia de datos y metadatos de Nextcloud |
| Caché y locking | Redis | Caché en memoria y coordinación de accesos concurrentes |
| Tareas internas | Nextcloud Cron | Ejecución periódica de trabajos en segundo plano |
| Acceso remoto | Cloudflare Tunnel | Acceso HTTPS sin exposición directa de puertos |
| Copias de seguridad | BorgBackup | Backups cifrados, deduplicados e incrementales |
| Automatización | systemd | Programación y supervisión de tareas del servidor |
| Monitorización | smartmontools | Supervisión del estado de los discos |
| Alertas | msmtp + Brevo | Envío de notificaciones ante fallos |
| Seguridad | AppArmor + 2FA | Restricción de procesos y protección de cuentas |
| Control de versiones | Git / GitHub | Versionado de documentación y configuración |

---

## Copias de seguridad y monitorización

El servidor dispone de varias tareas automáticas:

| Tarea | Frecuencia |
| --- | --- |
| Backup de Nextcloud con Borg | Diaria |
| Comprobación del repositorio Borg | Semanal |
| Comprobación SMART de discos | Mensual |
| Verificación completa de datos Borg | Trimestral |

Los servicios de mantenimiento están integrados con un sistema general de alertas basado en `systemd OnFailure`.

Si una tarea crítica termina con error se llama al servicio de mensajería para la notificación:

```text
Servicio
   │
   ▼
systemd detecta FAILED
   │
   ▼
OnFailure
   │
   ▼
hpserver-alert
   │
   ▼
msmtp
   │
   ▼
Brevo
   │
   ▼
Correo de alerta
```

Las comprobaciones SMART también convierten determinados indicadores de degradación del disco en errores del servicio, permitiendo recibir una alerta aunque el dispositivo continúe funcionando.

---

## Documentación

La documentación completa del proyecto se encuentra en [`docs/`](docs/).

| Capítulo | Contenido |
| --- | --- |
| [01 - Arquitectura](docs/01-arquitectura.md) | Diseño general y decisiones de arquitectura |
| [02 - Hardware](docs/02-hardware.md) | Servidor y dispositivos de almacenamiento |
| [03 - Debian y red](docs/03-debian-y-red.md) | Instalación, administración y configuración de red |
| [04 - Almacenamiento](docs/04-almacenamiento.md) | Particionado, sistemas de archivos y montajes |
| [05 - Docker](docs/05-docker.md) | Instalación y configuración de Docker |
| [06 - Nextcloud](docs/06-nextcloud.md) | Despliegue y configuración de Nextcloud |
| [07 - Cloudflare Tunnel](docs/07-cloudflare-tunnel.md) | Acceso remoto seguro |
| [08 - Seguridad y correo](docs/08-seguridad-y-correo.md) | 2FA, AppArmor, SMTP y seguridad |
| [09 - Backup con Borg](docs/09-backup-borg.md) | Estrategia y automatización de copias |
| [10 - Monitorización y alertas](docs/10-monitorizacion-y-alertas.md) | SMART, systemd y notificaciones |
| [11 - Operación y mantenimiento](docs/11-operacion-y-mantenimiento.md) | Administración habitual del servidor |
| [12 - Recuperación ante desastres](docs/12-recuperacion-desastres.md) | Procedimientos de restauración |
| [13 - Migración de hardware](docs/13-migracion-hardware.md) | Sustitución de discos, SSD o servidor |
| [14 - Incidencias y lecciones](docs/14-incidencias-y-lecciones.md) | Problemas encontrados y soluciones |

---

## Estructura del repositorio

```text
hpserver/
├── README.md
├── docs/
├── docker/
├── scripts/
├── systemd/
├── config/
├── diagrams/
├── .gitignore
├── LICENSE
└── CHANGELOG.md
```

Además de la documentación, el repositorio contiene versiones reproducibles y saneadas de los scripts y archivos de configuración utilizados por el servidor.

---

## Seguridad y secretos

Este repositorio **no contiene credenciales ni secretos utilizados en producción**.

No deben almacenarse en Git:

- Contraseñas de PostgreSQL o Nextcloud.
- Tokens de Cloudflare Tunnel.
- Credenciales SMTP.
- Passphrase de BorgBackup.
- Claves de recuperación de Borg.
- Archivos `.env` de producción.

Cuando sea necesario documentar una configuración que utilice credenciales, se proporcionarán archivos `.example` con valores ficticios.

Hay que tener en cuenta que, los secretos necesarios para una recuperación completa deben conservarse mediante mecanismos independientes del servidor y del propio repositorio.

---

## Filosofía de recuperación

Una copia de seguridad no se considera válida únicamente porque haya sido creada correctamente.

El proyecto contempla:

- Verificación periódica de la integridad del repositorio.
- Verificación criptográfica de los datos almacenados.
- Restauraciones de prueba.
- Validación de los dumps de PostgreSQL.
- Monitorización SMART de los discos.
- Procedimientos documentados de recuperación y migración.

Como principios general:
> **Ningún backup se considera una copia de seguridad fiable hasta que no se comprueba su funcionamiento**

> **El hardware antiguo no debe borrarse, formatearse ni reutilizarse hasta que el nuevo sistema haya sido validado y se haya realizado una restauración real satisfactoria.**

---

## Estado

**Versión inicial de la infraestructura: operativa.**

La instalación base, acceso remoto, almacenamiento, backups, comprobaciones de integridad, monitorización y sistema de alertas han sido configurados y probados.

La documentación detallada del proyecto se encuentra actualmente en desarrollo.