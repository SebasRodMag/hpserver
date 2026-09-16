# 08 - Seguridad y correo

## 1. Objetivo

Este capítulo documenta las medidas de seguridad aplicadas alrededor de `hpserver` y la infraestructura utilizada para el envío de correo electrónico.

Los objetivos principales son:

- proteger las cuentas administrativas;
- evitar la publicación accidental de credenciales;
- proporcionar correo saliente a Nextcloud;
- disponer de un mecanismo de correo independiente de Nextcloud para alertas del sistema;
- autenticar correctamente el dominio utilizado como remitente;
- proteger las credenciales SMTP;
- generar alertas automáticas cuando fallen tareas críticas;
- mantener separadas las distintas capas de seguridad.

La arquitectura de correo final utiliza dos caminos:

```text
                     Brevo SMTP
                    smtp-relay...
                         ▲
                         │
              STARTTLS / autenticación
                         │
              ┌──────────┴──────────┐
              │                     │
              │                     │
         Nextcloud                msmtp
              │                     ▲
              │                     │
    notificaciones          alertas systemd
                                    ▲
                                    │
                              Debian host
```

Esta separación es deliberada.

Nextcloud puede enviar sus propias notificaciones, mientras que Debian conserva un mecanismo independiente capaz de avisar incluso cuando Nextcloud está detenido.

> [!NOTE]
> Por motivos de seguridad y portabilidad, dominios, direcciones de correo, usuarios, tokens y credenciales específicos de la instalación se sustituyen por valores de ejemplo.

---

## 2. Seguridad por capas

La seguridad de `hpserver` no depende de una única medida.

Se aplican varias capas:

```text
cuentas
   │
   ├── contraseñas
   └── 2FA

servidor
   │
   ├── Debian actualizado
   ├── administración con sudo
   └── acceso SSH controlado

aplicación
   │
   ├── trusted_domains
   ├── trusted_proxies
   └── permisos

Internet
   │
   ├── Cloudflare Tunnel
   ├── HTTPS
   └── HSTS

datos
   │
   ├── permisos
   ├── Borg
   └── recuperación

monitorización
   │
   ├── systemd
   ├── SMART
   └── alertas SMTP
```

Una capa no sustituye a las demás.

Por ejemplo:

```text
2FA
 ≠
backup
```

y:

```text
Cloudflare Tunnel
 ≠
protección frente a borrado autenticado
```

---

## 3. Cuenta administrativa de Nextcloud

La cuenta administrativa dispone de permisos suficientes para modificar la configuración global de Nextcloud.

Por ello se protege especialmente.

Las medidas aplicadas incluyen:

- contraseña independiente;
- autenticación de dos factores;
- códigos de recuperación;
- uso administrativo separado de las cuentas familiares normales.

Los usuarios normales no necesitan privilegios administrativos para almacenar o sincronizar sus archivos.

---

## 4. Autenticación de dos factores

La cuenta administrativa utiliza TOTP como segundo factor.

El proceso de autenticación pasa a ser:

```text
usuario
   │
   ▼
contraseña
   │
   ▼
TOTP
   │
   ▼
Nextcloud
```

La contraseña por sí sola deja de ser suficiente para iniciar sesión.

Después de activar TOTP se realizó una prueba desde una sesión independiente para confirmar que el nuevo mecanismo funcionaba correctamente.

---

## 5. Códigos de recuperación

Durante la configuración de 2FA se generaron códigos de recuperación.

Estos códigos deben almacenarse fuera de Nextcloud.

No tendría sentido mantener la única copia en:

```text
Nextcloud
```

porque precisamente pueden ser necesarios cuando no sea posible acceder normalmente a Nextcloud.

Tampoco deben publicarse en:

```text
Git
documentación pública
capturas
scripts
README
```

### Principio

> **Un mecanismo de recuperación debe permanecer accesible cuando el sistema que pretende recuperar no lo esté.**

---

## 6. Política de secretos

El proyecto utiliza distintos tipos de credenciales:

```text
PostgreSQL
Nextcloud admin
Cloudflare Tunnel
Brevo SMTP
Borg
2FA recovery codes
```

Ninguna de ellas debe aparecer en el repositorio público.

La documentación utiliza únicamente placeholders:

```text
<POSTGRES_PASSWORD>
<NEXTCLOUD_ADMIN_PASSWORD>
<CLOUDFLARE_TUNNEL_TOKEN>
<SMTP_USERNAME>
<SMTP_PASSWORD>
<BORG_PASSPHRASE>
```

---

## 7. `.env`

Las credenciales utilizadas por Docker Compose se almacenan localmente en:

```text
/srv/docker/nextcloud/.env
```

El archivo debe tener permisos restrictivos.

Por ejemplo:

```bash
chmod 600 /srv/docker/nextcloud/.env
```

El repositorio público contiene únicamente:

```text
.env.example
```

con nombres de variables y valores ficticios.

### Ejemplo

```dotenv
POSTGRES_DB=nextcloud
POSTGRES_USER=nextcloud
POSTGRES_PASSWORD=<POSTGRES_PASSWORD>

NEXTCLOUD_ADMIN_USER=<ADMIN_USER>
NEXTCLOUD_ADMIN_PASSWORD=<ADMIN_PASSWORD>

CLOUDFLARE_TUNNEL_TOKEN=<CLOUDFLARE_TUNNEL_TOKEN>
```

El archivo real:

```text
.env
```

debe estar excluido mediante `.gitignore`.

---

## 8. Secretos y backups

Excluir un secreto de Git no significa que deba excluirse necesariamente de todos los mecanismos de recuperación.

Existe una diferencia entre:

```text
repositorio público
```

y:

```text
backup privado cifrado
```

El backup Borg de `hpserver` incluye información necesaria para reconstruir el servicio, incluyendo el `.env` real.

Esto es aceptable porque el repositorio Borg está cifrado.

Sin embargo, el acceso a ese backup depende a su vez de:

```text
Borg passphrase
+
Borg recovery key
```

que deben existir también fuera del propio servidor.

---

## 9. Correo de Nextcloud

Nextcloud necesita correo saliente para funciones como:

- recuperación de contraseña;
- notificaciones;
- comparticiones;
- avisos de actividad;
- mensajes administrativos.

Nextcloud no actúa como servidor de correo completo.

En `hpserver` se conecta a un relay SMTP externo.

La arquitectura es:

```text
Nextcloud
    │
    │ SMTP
    ▼
Brevo
    │
    ▼
Internet
    │
    ▼
destinatario
```

---

## 10. Proveedor SMTP

El proveedor utilizado es Brevo.

La configuración conceptual es:

```text
SMTP server:
smtp-relay.brevo.com

Port:
587

Transport:
SMTP + STARTTLS

Authentication:
enabled
```

El remitente pertenece a un dominio administrado por el proyecto.

En la documentación pública:

```text
nextcloud@example.com
```

puede utilizarse como valor de ejemplo.

---

## 11. STARTTLS

La conexión SMTP utiliza el puerto:

```text
587
```

y STARTTLS.

El flujo conceptual es:

```text
cliente SMTP
      │
      │ conexión
      ▼
servidor SMTP
      │
      │ STARTTLS
      ▼
canal TLS
      │
      ▼
autenticación + mensaje
```

En versiones modernas de Nextcloud, STARTTLS puede negociarse automáticamente cuando el servidor SMTP lo anuncia.

La configuración debe distinguirse de SMTPS implícito, habitualmente asociado al puerto `465`.

---

## 12. Configuración de correo en Nextcloud

La configuración se realiza desde la administración de Nextcloud o mediante sus parámetros de configuración.

Conceptualmente:

```text
Modo:
SMTP

Servidor:
smtp-relay.brevo.com

Puerto:
587

Seguridad:
None / STARTTLS

Autenticación:
Sí

Usuario:
<SMTP_USERNAME>

Contraseña:
<SMTP_PASSWORD>

Remitente:
nextcloud@example.com
```

La contraseña SMTP nunca debe copiarse a la documentación pública.

---

## 13. Prueba de correo desde Nextcloud

Nextcloud dispone de una función para enviar un mensaje de prueba.

La prueba permite validar:

```text
Nextcloud
    │
    ▼
resolución DNS
    │
    ▼
conexión SMTP
    │
    ▼
STARTTLS
    │
    ▼
autenticación
    │
    ▼
relay
```

Sin embargo, incluso una aceptación por parte del relay no demuestra necesariamente que el mensaje haya terminado en la bandeja del destinatario.

La entrega final debe comprobarse independientemente.

---

## 14. Primera incidencia SMTP: DNS dentro de Docker

La primera prueba de correo desde Nextcloud falló con un error equivalente a:

```text
php_network_getaddresses:
getaddrinfo for smtp-relay.brevo.com failed:
Temporary failure in name resolution
```

Este mensaje podría confundirse con un problema del servidor SMTP.

Sin embargo:

```text
host Debian
     │
     └── DNS correcto

contenedor Nextcloud
     │
     └── DNS incorrecto
```

Dentro del contenedor aparecía el resolver interno de Docker:

```text
nameserver 127.0.0.11
```

pero Docker no había configurado correctamente resolvers externos después del arranque.

---

## 15. Diagnóstico por capas

La prueba correcta consistió en separar:

```text
¿Debian resuelve?
       │
       ▼
¿Docker resuelve?
       │
       ▼
¿Nextcloud conecta?
       │
       ▼
¿SMTP autentica?
```

Como Debian resolvía correctamente y el contenedor no, las credenciales SMTP no eran todavía el punto a investigar.

### Lección

> **No se deben cambiar credenciales SMTP para intentar corregir un error de resolución DNS.**

El mensaje de error debe guiar el diagnóstico hacia la capa correspondiente.

---

## 16. DNS persistente para Docker

Después de corregir temporalmente el problema reiniciando Docker una vez disponible la red, se configuraron resolvers explícitos para evitar depender completamente del estado detectado durante el arranque.

Una configuración equivalente es:

```json
{
  "dns": [
    "<LAN_DNS>",
    "<SECONDARY_DNS>"
  ]
}
```

en:

```text
/etc/docker/daemon.json
```

Después del cambio:

```bash
sudo systemctl restart docker
```

y se valida nuevamente la resolución desde los contenedores.

La configuración pública no necesita contener las direcciones reales utilizadas en `hpserver`.

---

## 17. Resultado de la configuración SMTP de Nextcloud

Una vez corregido DNS, Nextcloud pudo:

```text
resolver smtp-relay.brevo.com
          │
          ▼
conectar al relay
          │
          ▼
negociar SMTP
          │
          ▼
autenticar
          │
          ▼
enviar
```

El correo de prueba fue recibido correctamente.

Esto confirmó que la configuración SMTP de Nextcloud era funcional.

---

## 18. Autenticación del dominio de envío

Conseguir autenticarse contra el relay SMTP no es lo mismo que autenticar el dominio utilizado como remitente.

Son dos problemas diferentes:

```text
SMTP authentication
       │
       └── ¿puede este cliente utilizar el relay?

Domain authentication
       │
       └── ¿está autorizado/autenticado este dominio
           para el sistema de envío?
```

Para mejorar la legitimidad y entregabilidad del correo se autenticó el dominio de envío en Brevo.

---

## 19. Registros utilizados por Brevo

En la configuración actual de Brevo con infraestructura compartida, la autenticación del dominio utiliza los registros proporcionados por el propio servicio.

Conceptualmente:

```text
Brevo code
    │
    └── TXT

DKIM
    │
    └── CNAME/TXT según configuración proporcionada

DMARC
    │
    └── TXT
```

Los valores concretos no se documentan públicamente.

### Importante sobre SPF

No debe afirmarse que `hpserver` añadió un registro SPF específico de Brevo como requisito de esta configuración.

La documentación actual de Brevo indica que para la autenticación normal del dominio se utilizan:

```text
Brevo code
DKIM
DMARC
```

y que los registros SPF/MX que proporciona Brevo están asociados a determinados escenarios de IP dedicada.

Si el dominio dispone de otros registros SPF por otros servicios, éstos deben gestionarse según las necesidades reales de esos servicios.

---

## 20. DKIM

DKIM permite firmar criptográficamente el correo enviado en nombre del dominio.

Conceptualmente:

```text
Brevo
   │
   ▼
firma DKIM
   │
   ▼
mensaje
   │
   ▼
destinatario
   │
   ▼
DNS del dominio
   │
   ▼
verificación
```

Esto permite comprobar que el mensaje firmado no ha sido modificado después de la firma y que la firma corresponde al dominio configurado.

Los registros exactos deben obtenerse desde la cuenta Brevo correspondiente y no copiarse de otra instalación.

---

## 21. DMARC

DMARC define una política para mensajes que utilizan el dominio y proporciona mecanismos de reporting.

Un registro conceptual puede contener:

```text
v=DMARC1;
p=<POLICY>;
rua=mailto:<REPORT_ADDRESS>
```

Los valores reales dependen de la política elegida.

DMARC permite indicar cómo deben tratarse determinados mensajes que no superan las comprobaciones de autenticación y alineamiento correspondientes.

---

## 22. Un único registro DMARC

Un dominio no debe publicar múltiples registros DMARC independientes para el mismo hostname.

Durante la configuración ya existía una política DMARC.

Por tanto, no se añadió simplemente un segundo registro para Brevo.

Se mantuvo:

```text
un único _dmarc
```

y se integró en él la información necesaria, incluyendo el destino `rua` correspondiente cuando procedía.

### Principio

> **Modificar una política DNS existente requiere integrar los cambios; no duplicar registros incompatibles.**

Antes de crear un registro nuevo debe comprobarse si ya existe uno para la misma función.

---

## 23. Validación en Brevo

Después de añadir los registros DNS, el dominio se verificó desde Brevo.

El estado final fue:

```text
Authenticated
```

La propagación DNS puede impedir que la validación sea inmediata.

Por tanto:

```text
registro creado
      │
      ≠
validación instantánea garantizada
```

Debe comprobarse el estado final después de la propagación correspondiente.

---

## 24. Segunda incidencia: SMTP `250` pero sin entrega

Para disponer de alertas independientes de Nextcloud se configuró posteriormente `msmtp` en Debian.

Durante una prueba, el servidor SMTP aceptó el mensaje con una respuesta equivalente a:

```text
250
```

indicando que el mensaje había sido aceptado/encolado por el relay.

Sin embargo, el mensaje no llegó al destinatario.

Esto demostró una diferencia fundamental:

```text
SMTP 250
   │
   ▼
relay aceptó el mensaje
   │
   ≠
entrega final confirmada
```

---

## 25. Diagnóstico mediante Brevo

Los logs del proveedor mostraron posteriormente que el problema estaba relacionado con el remitente/dominio.

El dominio todavía no se encontraba correctamente autenticado para ese flujo.

Después de completar la autenticación del dominio, un nuevo mensaje:

```text
msmtp
   │
   ▼
Brevo
   │
   ▼
destinatario
```

fue registrado como:

```text
Delivered
```

y apareció en el buzón del destinatario.

Inicialmente fue clasificado como spam, lo que también demuestra que:

```text
entregado
   │
   ≠
Inbox garantizado
```

---

## 26. Estados diferentes del correo

Para diagnosticar correctamente correo electrónico conviene distinguir al menos:

```text
cliente conecta
       │
       ▼
SMTP autentica
       │
       ▼
relay acepta
       │
       ▼
relay procesa
       │
       ▼
servidor destino acepta
       │
       ▼
Delivered
       │
       ▼
Inbox / Spam
```

Cada paso responde a una pregunta diferente.

Por ello:

> **`250 queued` no debe interpretarse como prueba de entrega final al destinatario.**

---

## 27. `msmtp`

Para las alertas del sistema Debian se instaló:

```text
msmtp
```

`msmtp` es un cliente SMTP ligero.

No se instaló un servidor de correo completo en `hpserver`.

La arquitectura es:

```text
systemd
   │
   ▼
script de alerta
   │
   ▼
msmtp
   │
   ▼
Brevo
   │
   ▼
correo administrador
```

Esto reduce la complejidad respecto a operar un MTA completo únicamente para enviar alertas.

---

## 28. Configuración de `msmtp`

La configuración administrativa se mantiene en:

```text
/root/.msmtprc
```

con permisos:

```bash
chmod 600 /root/.msmtprc
```

Una versión pública equivalente puede ser:

```text
defaults
auth on
tls on
tls_starttls on

account default
host smtp-relay.brevo.com
port 587
from alerts@example.com
user <SMTP_USERNAME>
passwordeval "cat /root/.config/msmtp/brevo-password"
```

Los parámetros concretos deben adaptarse al proveedor y a la instalación.

---

## 29. Separación de la contraseña SMTP

En lugar de escribir directamente la contraseña dentro de `.msmtprc`, se separó en:

```text
/root/.config/msmtp/brevo-password
```

El directorio:

```text
/root/.config/msmtp
```

tiene permisos restrictivos.

Por ejemplo:

```bash
chmod 700 /root/.config/msmtp
chmod 600 /root/.config/msmtp/brevo-password
```

`msmtp` obtiene la contraseña mediante:

```text
passwordeval
```

Conceptualmente:

```text
.msmtprc
    │
    ▼
passwordeval
    │
    ▼
archivo secreto
```

Esto permite publicar una plantilla de configuración sin publicar la credencial.

---

## 30. Configuración pública de `msmtp`

El repositorio puede incluir:

```text
config/msmtp/msmtprc.example
```

pero nunca:

```text
/root/.msmtprc
```

real.

El ejemplo debe contener únicamente:

```text
<SMTP_HOST>
<SMTP_USERNAME>
<SMTP_PASSWORD_FILE>
<ALERT_FROM>
```

o valores pertenecientes a `example.com`.

---

## 31. AppArmor

En Debian, `msmtp` quedó sujeto a las restricciones correspondientes de AppArmor.

Inicialmente se configuró un log específico:

```text
/var/log/msmtp.log
```

El envío SMTP funcionaba, pero `msmtp` no podía escribir correctamente ese archivo debido a la política de AppArmor.

Esto generó una situación importante:

```text
correo
  │
  └── funciona

logfile
  │
  └── bloqueado
```

---

## 32. Incidencia AppArmor

Una posible reacción habría sido modificar o relajar el perfil de AppArmor para permitir el nuevo archivo.

No se consideró necesario.

El objetivo del sistema era enviar alertas, no mantener obligatoriamente un fichero de log separado para `msmtp`.

Por ello se eliminó:

```text
logfile /var/log/msmtp.log
```

de la configuración.

Los eventos relevantes pueden consultarse mediante el journal del sistema.

### Lección

> **No se debe debilitar un mecanismo de seguridad únicamente para mantener una comodidad de logging que no es necesaria.**

Cuando el sistema ya proporciona un mecanismo adecuado como `journald`, añadir permisos adicionales debe estar justificado.

---

## 33. Configuración de destinatarios de alertas

Las direcciones utilizadas por los scripts no se escriben directamente en cada script.

Se utiliza:

```text
/etc/hpserver-alert.conf
```

con permisos restrictivos.

Conceptualmente:

```bash
ALERT_FROM="alerts@example.com"
ALERT_TO="admin@example.net"
```

El archivo real:

```text
/etc/hpserver-alert.conf
```

no debe publicarse.

El repositorio contiene:

```text
config/hpserver-alert.conf.example
```

---

## 34. Script de alertas

El script utilizado es:

```text
/usr/local/sbin/send-systemd-alert.sh
```

con permisos restrictivos para administración.

Su función es recibir el nombre de una unidad systemd que ha fallado y construir un mensaje con información útil para el diagnóstico.

Conceptualmente:

```text
systemd unit falla
       │
       ▼
OnFailure
       │
       ▼
hpserver-alert@.service
       │
       ▼
send-systemd-alert.sh
       │
       ├── systemctl status
       ├── journalctl
       └── msmtp
              │
              ▼
            Brevo
              │
              ▼
        administrador
```

---

## 35. Información incluida en la alerta

El correo puede incluir:

```text
hostname
unidad que falló
fecha/hora
systemctl status
últimas líneas del journal
```

Por ejemplo, el script recoge aproximadamente las últimas:

```text
50
```

líneas relevantes del journal.

Esto permite que el administrador disponga de contexto inicial sin tener que conectarse inmediatamente al servidor.

La alerta no pretende sustituir una investigación completa.

---

## 36. Servicio template de systemd

Se utiliza una unidad template:

```text
hpserver-alert@.service
```

El símbolo:

```text
@
```

permite reutilizar el mismo servicio para diferentes unidades que fallen.

Conceptualmente:

```text
hpserver-alert@nextcloud-backup.service
hpserver-alert@borg-check.service
hpserver-alert@smart-check.service
```

utilizan la misma plantilla.

La instancia permite identificar qué servicio originó la alerta.

---

## 37. `OnFailure`

systemd proporciona la directiva:

```ini
OnFailure=
```

para activar una o varias unidades cuando otra entra en estado `failed`.

En `hpserver` se utiliza conceptualmente:

```ini
[Unit]
OnFailure=hpserver-alert@%n.service
```

`%n` representa el nombre completo de la unidad que ha fallado.

El resultado es:

```text
servicio crítico
      │
      ▼
FAILED
      │
      ▼
OnFailure
      │
      ▼
servicio de alerta
```

---

## 38. Servicios protegidos mediante alertas

El mecanismo se aplica a tareas críticas como:

```text
nextcloud-backup.service
borg-check.service
borg-verify-data.service
smart-check.service
```

Por tanto, pueden generarse avisos ante fallos de:

```text
backup
integridad Borg
verificación de datos Borg
SMART
```

El sistema de alertas es independiente de la interfaz de Nextcloud.

---

## 39. Por qué las alertas no dependen de Nextcloud

Una arquitectura como:

```text
Nextcloud falla
      │
      ▼
Nextcloud intenta enviar alerta
```

no es suficientemente robusta.

Por ello:

```text
Debian/systemd
      │
      ▼
msmtp
      │
      ▼
Brevo
```

funciona independientemente de:

```text
Nextcloud
PostgreSQL
Redis
```

Mientras Debian conserve conectividad y pueda utilizar el relay SMTP, puede informar de fallos de esos servicios.

### Principio

> **El mecanismo que informa de un fallo crítico no debería depender innecesariamente del mismo componente que puede estar fallando.**

---

## 40. Validación manual del sistema de alertas

El mecanismo se probó manualmente antes de confiar en él.

Primero se validó:

```text
msmtp
   │
   ▼
Brevo
   │
   ▼
destinatario
```

Después se comprobó:

```text
script
   │
   ▼
msmtp
```

y finalmente:

```text
systemd failure
      │
      ▼
OnFailure
      │
      ▼
alert service
      │
      ▼
script
      │
      ▼
correo
```

Se provocó deliberadamente un fallo controlado y se confirmó la recepción de la alerta.

---

## 41. Importancia de probar el fallo real

Ejecutar únicamente:

```bash
echo "test" | msmtp admin@example.net
```

demuestra que `msmtp` puede enviar un mensaje.

No demuestra que:

```text
OnFailure
+
template service
+
script
+
journal
+
msmtp
```

funcionen conjuntamente.

Por ello se realizó también una prueba de la cadena completa.

### Principio

```text
probar componente
      │
      ≠
probar sistema
```

Ambos niveles son necesarios.

---

## 42. Seguridad del usuario `docker`

La pertenencia al grupo:

```text
docker
```

permite utilizar el daemon Docker sin `sudo`.

Sin embargo, esta comodidad tiene implicaciones de seguridad importantes.

Un usuario capaz de controlar Docker puede, por ejemplo, crear contenedores con acceso privilegiado o montar partes del filesystem del host.

Por tanto:

> **La pertenencia al grupo `docker` debe considerarse equivalente a disponer de privilegios administrativos sobre el servidor.**

No debe añadirse a usuarios normales sin necesidad.

---

## 43. SSH y administración

La administración habitual se realiza mediante una cuenta no root con:

```text
sudo
```

El acceso root directo no se utiliza como mecanismo administrativo cotidiano.

Esto proporciona una separación entre:

```text
identidad del administrador
        │
        ▼
elevación mediante sudo
        │
        ▼
acción privilegiada
```

La configuración completa de Debian y SSH se documenta en:

```text
docs/03-debian-y-red.md
```

---

## 44. Permisos de archivos sensibles

Los principales archivos privados deben mantener permisos restrictivos.

Ejemplos:

```text
/srv/docker/nextcloud/.env
/root/.msmtprc
/root/.config/msmtp/brevo-password
/etc/hpserver-alert.conf
/root/.config/borg/passphrase
```

El principio general es:

```text
secreto
   │
   ▼
mínimo acceso necesario
```

Los scripts que contienen lógica pero no secretos pueden publicarse después de sanitizarlos.

---

## 45. Borg y seguridad de recuperación

El repositorio Borg utiliza cifrado:

```text
repokey-blake2
```

La capacidad de recuperación depende de conservar:

```text
repositorio
+
passphrase
+
recovery key
```

La recovery key y la passphrase no deben existir únicamente dentro del servidor protegido.

La clave de recuperación fue exportada y transferida fuera de `hpserver`.

Los detalles se documentan en:

```text
docs/09-backup-borg.md
```

---

## 46. Qué no debe publicarse

Antes de hacer público el repositorio deben buscarse al menos:

```text
contraseñas
tokens
API keys
SMTP credentials
Borg passphrase
Borg recovery key
2FA recovery codes
direcciones de correo personales
UUID reales
seriales
dominio real si se desea anonimizar
```

También deben revisarse:

```text
capturas
logs
comandos copiados
.env
config.php
shell history exportada
```

porque pueden contener secretos indirectamente.

---

## 47. Rotación ante exposición

Si una credencial aparece accidentalmente en un repositorio Git, eliminarla del último commit no debe considerarse suficiente.

Debe asumirse que pudo ser copiada.

El procedimiento correcto es:

```text
detectar exposición
       │
       ▼
revocar / rotar secreto
       │
       ▼
actualizar servicio
       │
       ▼
validar
       │
       ▼
sanear repositorio
```

Dependiendo del secreto afectado puede implicar:

```text
SMTP key
Cloudflare token
password
API key
```

### Principio

> **Un secreto publicado deja de considerarse secreto aunque posteriormente se borre del repositorio.**

---

## 48. Correo como dependencia de monitorización

Las alertas por correo dependen de:

```text
Debian
   │
   ▼
DNS
   │
   ▼
Internet
   │
   ▼
Brevo
   │
   ▼
servidor destino
```

Por tanto, no pueden detectar de forma fiable una pérdida total de:

```text
electricidad
Internet
red local
servidor completo
```

si el propio servidor no puede enviar el mensaje.

Esto es una limitación importante.

Para detectar que `hpserver` ha desaparecido completamente sería necesario un sistema externo que comprobara periódicamente su disponibilidad.

---

## 49. Alertas internas frente a monitorización externa

Los dos mecanismos resuelven problemas diferentes.

```text
MONITORIZACIÓN INTERNA

hpserver detecta
su propio fallo
      │
      ▼
envía alerta
```

frente a:

```text
MONITORIZACIÓN EXTERNA

sistema externo
      │
      ▼
comprueba hpserver
      │
      ▼
no responde
      │
      ▼
envía alerta
```

La instalación actual dispone principalmente del primer modelo para tareas programadas.

Una futura ampliación puede incorporar monitorización externa.

---

## 50. Principios aplicados

### 2FA para cuentas privilegiadas

Una contraseña comprometida no debe proporcionar automáticamente acceso administrativo.

### Recovery codes fuera del servicio

La recuperación no debe depender del sistema que se intenta recuperar.

### Secretos fuera de Git

Los repositorios contienen plantillas, no credenciales reales.

### Backups privados pueden contener información necesaria para recuperación

Siempre que exista cifrado adecuado y las claves estén protegidas independientemente.

### SMTP no implica operar un servidor de correo

Se utiliza un relay externo y clientes SMTP.

### `250` no significa entrega final

Debe comprobarse el estado posterior del mensaje.

### La autenticación SMTP y la autenticación del dominio son conceptos distintos

Resolver una no garantiza la otra.

### DMARC no se duplica

Una política existente debe integrarse correctamente.

### No debilitar AppArmor sin necesidad

Se prefirió `journald` frente a ampliar permisos únicamente para un fichero de log adicional.

### Las alertas deben ser independientes de Nextcloud

systemd + `msmtp` permiten avisar aunque la aplicación esté caída.

### El grupo Docker implica privilegios elevados

Su pertenencia debe limitarse a administradores.

---

## 51. Incidencias y lecciones principales

| Incidencia | Causa / diagnóstico | Solución / lección |
| --- | --- | --- |
| Nextcloud no conectaba con Brevo | DNS roto dentro del contenedor | Corregir DNS de Docker |
| Reiniciar Docker solucionaba temporalmente DNS | Docker había arrancado sin resolvers externos adecuados | Configurar DNS persistente |
| SMTP aceptaba el mensaje pero no llegaba | Dominio/remitente no autenticado correctamente | Autenticar dominio y revisar logs del proveedor |
| `250 queued` interpretado inicialmente como éxito completo | Sólo confirmaba aceptación por el relay | Separar aceptación de entrega |
| Mensaje entregado apareció en spam | Entrega no implica Inbox | Revisar autenticación y reputación |
| DMARC ya existía | Riesgo de crear dos registros | Mantener uno e integrar `rua` |
| `msmtp` no escribía `/var/log/msmtp.log` | AppArmor bloqueaba el acceso | Eliminar logfile y utilizar journal |
| Riesgo de contraseña en `.msmtprc` | Credencial almacenada junto a configuración | Separarla y usar `passwordeval` |
| Alertas probadas sólo por componentes inicialmente | Una prueba SMTP no valida `OnFailure` | Provocar fallo controlado y probar cadena completa |

---

## 52. Resumen

La arquitectura final de seguridad y correo queda:

```text
                         INTERNET
                             │
             ┌───────────────┴───────────────┐
             │                               │
             ▼                               ▼
        Cloudflare                         Brevo
             │                               ▲
             │                               │
        Nextcloud                     SMTP / STARTTLS
             │                               │
       ┌─────┴─────┐                 ┌───────┴───────┐
       │           │                 │               │
      2FA       usuarios         Nextcloud         msmtp
                                                   ▲
                                                   │
                                                systemd
                                                   │
                                      ┌────────────┼────────────┐
                                      │            │            │
                                    Borg         SMART        Backup
```

Las medidas no se consideran soluciones aisladas.

La seguridad de `hpserver` depende de la combinación de:

```text
mínimos privilegios
+
2FA
+
protección de secretos
+
HTTPS
+
autenticación del dominio
+
backups cifrados
+
monitorización
+
alertas independientes
```

El correo cumple dos funciones distintas:

```text
Nextcloud
   │
   └── comunicación de la aplicación

Debian
   │
   └── monitorización de infraestructura
```

La separación permite que un fallo de Nextcloud no elimine también el mecanismo utilizado para informar del problema.

---

## 53. Referencias

### Nextcloud — correo

- **Nextcloud Administration Manual — Email**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/email_configuration.html

  - Configuración SMTP.
  - STARTTLS.
  - Autenticación.
  - Puerto SMTP.
  - Correo de prueba.
  - Configuración mediante `config.php`.

- **Nextcloud — Configuration Parameters**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/config_sample_php_parameters.html

  - `mail_domain`.
  - `mail_from_address`.
  - `mail_smtpmode`.
  - `mail_smtphost`.
  - `mail_smtpport`.
  - `mail_smtpauth`.

### Nextcloud — autenticación de dos factores

- **Nextcloud Administration Manual — Two-factor authentication**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_user/two_factor-auth.html

  - Proveedores 2FA.
  - TOTP.
  - Códigos de recuperación.
  - Administración de 2FA.

### Brevo — autenticación del dominio

- **Brevo — Autenticar el dominio con Brevo**  
  https://help.brevo.com/hc/es/articles/12163873383186-Autenticar-el-dominio-con-Brevo-c%C3%B3digo-Brevo-DKIM-DMARC

  - Código Brevo.
  - DKIM.
  - DMARC.
  - Verificación del dominio.
  - Propagación DNS.
  - Consideraciones sobre SPF y MX.

- **Brevo — FAQ sobre autenticación de dominio**  
  https://help.brevo.com/hc/es/articles/17286219877778-FAQ-Acerca-de-la-autenticaci%C3%B3n-de-dominio-c%C3%B3digo-Brevo-DKIM-DMARC

  - Propósito de DKIM.
  - Propósito de DMARC.
  - Entregabilidad.
  - Protección frente a spoofing y phishing.

### msmtp

- **Debian Manpages — msmtp**  
  https://manpages.debian.org/trixie/msmtp/msmtp.1.en.html

  - Cliente SMTP.
  - TLS.
  - Autenticación.
  - Configuración.
  - `passwordeval`.

- **Debian Manpages — msmtprc**  
  https://manpages.debian.org/trixie/msmtp/msmtprc.5.en.html

  - Archivo de configuración.
  - Cuentas.
  - Parámetros SMTP.
  - Gestión de contraseña.

### systemd

- **systemd.unit — OnFailure**  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html

  - `OnFailure=`.
  - Dependencias entre unidades.
  - Especificadores utilizados en unidades.

- **systemd.service**  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html

  - Servicios.
  - `Type=oneshot`.
  - Ejecución de scripts mediante systemd.

- **systemd specifiers**  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Specifiers

  - `%n`.
  - Nombres de unidades.
  - Templates e instancias.

### AppArmor

- **Debian Wiki — AppArmor**  
  https://wiki.debian.org/AppArmor

  - Funcionamiento de AppArmor en Debian.
  - Perfiles.
  - Diagnóstico de denegaciones.

### Docker

- **Docker Engine — Linux post-installation steps**  
  https://docs.docker.com/engine/install/linux-postinstall/

  - Grupo `docker`.
  - Implicaciones de seguridad.
  - Privilegios equivalentes a root.

### BorgBackup

- **BorgBackup — Security FAQ**  
  https://borgbackup.readthedocs.io/en/stable/faq.html#security

- **BorgBackup — Key management**  
  https://borgbackup.readthedocs.io/en/stable/usage/key.html

  - Claves.
  - Exportación.
  - Recuperación.
  - Consideraciones de seguridad.

> Las referencias describen el comportamiento oficial de Nextcloud, Brevo, msmtp, systemd, AppArmor, Docker y BorgBackup. La arquitectura de alertas, las decisiones operativas y las incidencias descritas corresponden a la implementación de `hpserver`.