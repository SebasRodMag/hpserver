# 07 - Cloudflare Tunnel

## 1. Objetivo

`hpserver` permite acceder a Nextcloud desde Internet mediante **Cloudflare Tunnel**. Esta es una de las decisiones arquitectónicas más importantes, porque Nextcloud no esta publicado desde un puerto abierto en el router, Es cloudflared quien inicia la conexión hacia Nextcloud, y Cloudflare documenta expresamente este modelo como conexiones outbound-only.

El objetivo es proporcionar:

- acceso remoto mediante un dominio público;
- HTTPS para los clientes externos;
- ausencia de port forwarding en el router;
- una separación clara entre el servidor de origen y su exposición pública;
- una conexión iniciada desde el propio servidor hacia Cloudflare;
- integración con la arquitectura Docker existente.

La arquitectura general es:

```text
Internet
    │
    │ HTTPS
    ▼
Cloudflare
    │
    │ Tunnel
    ▼
cloudflared
    │
    │ HTTP / red Docker
    ▼
Nextcloud
```

El servidor no necesita aceptar directamente conexiones entrantes desde Internet para publicar Nextcloud mediante este mecanismo.

> [!NOTE]
> El dominio real, token del túnel, identificadores y otros datos específicos de la instalación se sustituyen por valores de ejemplo.

---

## 2. Por qué utilizar Cloudflare Tunnel

Una forma tradicional de publicar un servicio doméstico consiste en:

```text
Internet
   │
   ▼
IP pública
   │
   ▼
router
   │
port forwarding
   │
   ▼
servidor
```

Esto requiere exponer uno o varios puertos del router hacia el servidor interno.

`hpserver` utiliza un modelo diferente:

```text
hpserver
   │
   │ conexión saliente
   ▼
Cloudflare
   ▲
   │
Internet
```

`cloudflared` inicia desde el servidor las conexiones necesarias hacia la red de Cloudflare.

No es necesario crear una redirección entrante hacia Nextcloud en el router.

---

## 3. Modelo de conexión saliente

Cloudflare Tunnel utiliza un modelo de conexiones **outbound-only**.

El conector:

```text
cloudflared
```

establece conexiones salientes persistentes hacia la red de Cloudflare.

Una vez establecido el túnel, el tráfico puede circular en ambos sentidos sobre esas conexiones.

Conceptualmente:

```text
                    INTERNET
                        │
                        ▼
                  Cloudflare
                        ▲
                        │
              conexión iniciada
               por cloudflared
                        │
                        │
                  ┌─────┴─────┐
                  │ hpserver  │
                  └───────────┘
```

Esto permite publicar el servicio sin que el origen necesite una dirección IP públicamente enrutable.

---

## 4. Ausencia de port forwarding

No se configura en el router una regla equivalente a:

```text
WAN:443
   │
   ▼
SERVER_LAN_IP:443
```

ni:

```text
WAN:8080
   │
   ▼
SERVER_LAN_IP:8080
```

El acceso remoto sigue:

```text
cliente
   │
   ▼
cloud.example.com
   │
   ▼
Cloudflare
   │
   ▼
Tunnel
   │
   ▼
cloudflared
   │
   ▼
Nextcloud
```

Mientras que el acceso local puede seguir:

```text
cliente LAN
    │
    ▼
http://<SERVER_LAN_IP>:8080
    │
    ▼
Nextcloud
```

Los dos caminos son independientes.

---

## 5. Ventajas de esta arquitectura

La elección de Cloudflare Tunnel aporta varias características útiles para el proyecto.

### No requiere IP pública directamente accesible

El origen no necesita recibir conexiones iniciadas directamente desde Internet.

### No requiere abrir puertos de entrada

No es necesario publicar Nextcloud mediante NAT/port forwarding.

### HTTPS público

El cliente accede mediante:

```text
https://cloud.example.com
```

### Ocultación del origen

El servicio público se presenta a través de la infraestructura de Cloudflare en lugar de necesitar publicar directamente el origen.

### Integración con Docker

`cloudflared` puede ejecutarse dentro de la misma red Compose que Nextcloud y acceder a la aplicación mediante su nombre de servicio.

---

## 6. Túnel administrado remotamente

La instalación utiliza un túnel administrado remotamente.

La configuración principal del túnel se gestiona desde Cloudflare, mientras el conector local se autentica mediante un token.

Conceptualmente:

```text
Cloudflare
    │
    │ configuración del túnel
    │
    ▼
Tunnel
    ▲
    │ token
    │
cloudflared
```

El token permite asociar una instancia de `cloudflared` con el túnel correspondiente.

---

## 7. El token del túnel

El token es una credencial sensible.

Una persona que disponga del token de un túnel administrado remotamente puede ejecutar un conector asociado a ese túnel.

Por ello:

```text
CLOUDFLARE_TUNNEL_TOKEN
```

no debe aparecer en:

```text
Git
README
compose.yml público
documentación
capturas públicas
logs compartidos
```

En `hpserver` se proporciona mediante:

```text
.env
```

y el repositorio contiene únicamente un placeholder:

```dotenv
CLOUDFLARE_TUNNEL_TOKEN=<CLOUDFLARE_TUNNEL_TOKEN>
```

### Principio

> **El token de Cloudflare Tunnel debe tratarse como una credencial y no como un simple identificador del túnel.**

Si existe sospecha de exposición, debe rotarse o sustituirse siguiendo los mecanismos proporcionados por Cloudflare.

---

## 8. `cloudflared` en Docker

El conector se ejecuta como parte de la pila Compose.

Una definición equivalente es:

```yaml
cloudflared:
  image: cloudflare/cloudflared:<VERSION>
  restart: unless-stopped
  command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
```

El comando:

```text
tunnel run
```

inicia el conector.

La opción:

```text
--token
```

asocia la instancia con el túnel administrado remotamente.

La opción:

```text
--no-autoupdate
```

es apropiada en el contexto de un contenedor, donde las actualizaciones deben realizarse sustituyendo la imagen y recreando el contenedor.

---

## 9. Política de versión de `cloudflared`

Durante la instalación inicial se utilizó una imagen equivalente a:

```text
cloudflare/cloudflared:latest
```

A diferencia de los componentes con estado como PostgreSQL, `cloudflared` no contiene la base de datos ni los archivos de los usuarios.

Por tanto, su sustitución es considerablemente menos arriesgada.

Sin embargo, para mejorar la reproducibilidad del proyecto se puede fijar también una versión concreta:

```yaml
image: cloudflare/cloudflared:<CLOUDFLARED_VERSION>
```

Esto permite que una reconstrucción posterior utilice exactamente la versión prevista.

La utilización actual o histórica de `latest` debe considerarse una decisión revisable y no una recomendación general para todos los componentes.

---

## 10. Red Docker

`cloudflared` pertenece a la misma red Compose que Nextcloud.

Por ello puede localizar la aplicación mediante:

```text
app
```

sin utilizar:

```text
SERVER_LAN_IP
```

ni la dirección IP interna dinámica del contenedor.

El flujo interno es:

```text
cloudflared
     │
     │ Docker DNS
     ▼
    app
     │
     ▼
  puerto 80
```

Esto evita salir innecesariamente de la red Docker para volver a entrar por el puerto publicado del host.

---

## 11. Ruta pública

La aplicación publicada utiliza una configuración conceptual equivalente a:

```text
Hostname:
cloud.example.com

Service:
http://app:80
```

Por tanto:

```text
https://cloud.example.com
          │
          ▼
      Cloudflare
          │
          ▼
       Tunnel
          │
          ▼
   http://app:80
```

La conexión entre `cloudflared` y Nextcloud utiliza HTTP dentro de la infraestructura local de Docker.

La conexión pública del cliente utiliza HTTPS.

---

## 12. HTTPS y terminación TLS

Desde el punto de vista del usuario:

```text
cliente
   │
   │ HTTPS
   ▼
Cloudflare
```

Cloudflare termina la conexión HTTPS pública.

Posteriormente, el tráfico alcanza el servicio de origen a través del túnel.

En esta arquitectura:

```text
Internet
    │ HTTPS
    ▼
Cloudflare
    │
    │ Tunnel
    ▼
cloudflared
    │ HTTP
    ▼
Nextcloud
```

Esto explica por qué Nextcloud necesita conocer que la petición pública original utiliza HTTPS aunque internamente reciba HTTP.

La configuración correspondiente:

```text
trusted_proxies
overwriteprotocol
overwritecondaddr
overwrite.cli.url
```

se documenta en `06-nextcloud.md`.

---

## 13. Separación entre acceso local y remoto

`hpserver` mantiene dos caminos de acceso.

### Local

```text
http://<SERVER_LAN_IP>:8080
```

### Remoto

```text
https://cloud.example.com
```

Conceptualmente:

```text
                    Nextcloud
                    ▲       ▲
                    │       │
                  HTTP     HTTP
                    │       │
                   LAN   cloudflared
                            ▲
                            │ Tunnel
                            │
                        Cloudflare
                            ▲
                            │ HTTPS
                          Internet
```

Esto permite continuar accediendo localmente aunque el servicio de túnel o la conectividad con Cloudflare no estén disponibles.

---

## 14. Dependencia de Cloudflare

Cloudflare Tunnel añade una dependencia externa para el acceso remoto.

Si ocurre:

```text
cloudflared detenido
```

o:

```text
conectividad Internet ausente
```

o existe una incidencia externa que impide utilizar el túnel:

```text
acceso remoto
     │
     ▼
no disponible
```

pero Nextcloud puede continuar funcionando dentro de la LAN:

```text
LAN
 │
 ▼
SERVER_LAN_IP:8080
 │
 ▼
Nextcloud
```

Esta separación resulta útil tanto para disponibilidad local como para diagnóstico.

---

## 15. Estado del contenedor

El estado puede comprobarse mediante:

```bash
docker compose ps cloudflared
```

Los logs:

```bash
docker compose logs cloudflared
```

o:

```bash
docker compose logs --tail=100 cloudflared
```

permiten diagnosticar:

- establecimiento del túnel;
- conexiones con Cloudflare;
- errores de red;
- problemas de autenticación;
- reconexiones.

---

## 16. Estado del túnel

Además del contenedor local, el estado del túnel puede consultarse desde la administración de Cloudflare.

Debe distinguirse:

```text
contenedor cloudflared running
            │
            ≠
túnel necesariamente operativo
```

Un contenedor puede estar ejecutándose mientras intenta reconectar o presenta problemas de conectividad.

La validación definitiva consiste en comprobar el servicio desde una red externa.

---

## 17. Validación desde Internet

Durante la puesta en marcha se realizó una prueba utilizando una conexión móvil independiente de la LAN.

El objetivo era evitar una falsa validación causada por acceder al servidor desde la misma red doméstica.

La prueba fue:

```text
dispositivo móvil
      │
      │ red móvil
      ▼
Internet
      │
      ▼
https://cloud.example.com
      │
      ▼
Nextcloud
```

Se confirmó:

- resolución del dominio;
- establecimiento de HTTPS;
- carga de Nextcloud;
- autenticación;
- acceso a los archivos.

Esta prueba demuestra el funcionamiento completo del camino externo.

---

## 18. Validación después de reiniciar

La disponibilidad remota también se comprobó después de reiniciar físicamente el servidor.

El flujo esperado es:

```text
hpserver arranca
      │
      ▼
Docker
      │
      ▼
cloudflared
      │
      ▼
Tunnel
      │
      ▼
acceso remoto
```

La política:

```yaml
restart: unless-stopped
```

permite que `cloudflared` vuelva a iniciarse automáticamente con Docker.

Una configuración de acceso remoto no se considera completa hasta comprobar que se recupera sin intervención manual después de un reboot.

---

## 19. Incidencia: 502 después de un reinicio

Durante una fase inicial apareció:

```text
502
```

al acceder mediante el túnel después de reiniciar el servidor.

Cloudflare y `cloudflared` estaban disponibles, pero el servicio Nextcloud `app` no se había iniciado correctamente.

La causa se encontraba en Docker:

```text
failed to bind host port
cannot assign requested address
```

El contenedor `app` intentaba publicar:

```text
<SERVER_LAN_IP>:8080
```

antes de que la interfaz de red hubiera recibido la dirección estática.

La arquitectura real del fallo era:

```text
Internet
   │
   ▼
Cloudflare          OK
   │
   ▼
cloudflared         OK
   │
   ▼
app                 NO DISPONIBLE
```

### Solución

Se cambió la publicación:

```yaml
ports:
  - "<SERVER_LAN_IP>:8080:80"
```

por:

```yaml
ports:
  - "8080:80"
```

Después de un nuevo reinicio:

- `app` inició automáticamente;
- `cloudflared` inició automáticamente;
- el acceso LAN funcionó;
- el acceso remoto funcionó.

### Lección

> **Un error HTTP presentado por la capa externa no demuestra que el problema se encuentre en esa capa.**

El diagnóstico debe seguir todo el recorrido hasta el origen.

---

## 20. HSTS

Nextcloud detectó inicialmente que la respuesta pública no contenía:

```text
Strict-Transport-Security
```

con el tiempo mínimo recomendado.

HSTS indica al navegador que debe utilizar HTTPS para acceder al hostname durante el periodo especificado.

Como HTTPS termina en Cloudflare, se decidió añadir el encabezado en esa capa.

La política configurada utiliza:

```text
Strict-Transport-Security: max-age=15552000
```

sin añadir inicialmente:

```text
includeSubDomains
```

ni:

```text
preload
```

---

## 21. Por qué no utilizar inicialmente `includeSubDomains`

La directiva:

```text
includeSubDomains
```

aplicaría la política HSTS también a los subdominios.

Esto puede tener consecuencias sobre otros servicios existentes o futuros bajo el mismo dominio.

Por tanto, no se activa simplemente para eliminar un warning.

La política inicial se limita al hostname de Nextcloud.

### Principio

> **Las políticas de seguridad con efectos persistentes sobre navegadores y subdominios deben ampliarse únicamente cuando se conocen sus consecuencias.**

---

## 22. Incidencia: Request Header frente a Response Header

Durante la configuración de HSTS se creó inicialmente la regla en la categoría incorrecta.

Se utilizó una:

```text
Request Header Transform Rule
```

cuando el objetivo era modificar la respuesta enviada al navegador.

Esto no producía el encabezado esperado al comprobar la respuesta pública.

### Diferencia

Una Request Header Transform Rule modifica:

```text
cliente
   │
   ▼
Cloudflare
   │
   │ request modificada
   ▼
origen
```

Pero HSTS debe enviarse hacia el cliente:

```text
origen
   │
   ▼
Cloudflare
   │
   │ response modificada
   ▼
cliente
```

Por tanto, la regla correcta es una:

```text
Response Header Transform Rule
```

### Solución

Se creó una regla de transformación de respuesta aplicable al hostname de Nextcloud.

La operación establece:

```text
Strict-Transport-Security
```

con:

```text
max-age=15552000
```

### Lección

> **Antes de modificar una cabecera HTTP debe identificarse en qué dirección viaja.**

```text
Request  → cliente hacia servidor
Response → servidor hacia cliente
```

HSTS pertenece a la respuesta.

---

## 23. Validación de HSTS

La configuración no se consideró correcta simplemente porque la regla apareciera activa en Cloudflare.

Se comprobó la respuesta HTTP pública.

Por ejemplo:

```bash
curl -I https://cloud.example.com
```

El resultado debe contener:

```text
Strict-Transport-Security: max-age=15552000
```

Esto valida el comportamiento desde el punto de vista del cliente.

### Principio

```text
configuración guardada
        │
        ≠
resultado necesariamente correcto
```

Debe comprobarse el efecto real de la configuración.

---

## 24. Transform Rules

Cloudflare distingue diferentes tipos de reglas.

Entre ellas:

```text
Request Header Transform Rules
        │
        └── modifican petición hacia origen

Response Header Transform Rules
        │
        └── modifican respuesta hacia cliente
```

Para HSTS interesa la segunda categoría.

Las reglas de respuesta permiten:

- añadir cabeceras;
- establecer valores;
- sustituir valores existentes;
- eliminar cabeceras.

La regla debe limitarse al hostname correspondiente cuando no se desea afectar otros servicios del dominio.

---

## 25. Seguridad del origen

Cloudflare Tunnel permite evitar la publicación directa del origen en Internet.

En `hpserver` no se necesita una regla de NAT pública hacia Nextcloud.

Sin embargo, esto no significa que el servidor deje de necesitar medidas de seguridad.

Siguen siendo necesarios:

```text
actualizaciones
+
credenciales seguras
+
2FA
+
trusted_domains
+
trusted_proxies
+
backups
+
monitorización
```

Cloudflare Tunnel resuelve una parte concreta del problema:

```text
conectividad remota
+
exposición del origen
```

No sustituye la seguridad de Nextcloud ni del servidor.

---

## 26. Firewall y tráfico saliente

`cloudflared` necesita establecer conexiones salientes hacia Cloudflare.

La documentación oficial identifica el puerto:

```text
7844
```

para las conexiones principales del túnel.

Según el protocolo utilizado puede emplearse:

```text
UDP → QUIC
TCP → HTTP/2
```

En una política de firewall restrictiva debe permitirse el tráfico saliente necesario para `cloudflared`.

La arquitectura no requiere por ello permitir conexiones entrantes equivalentes desde Internet hacia el servidor.

---

## 27. DNS público

El hostname público está asociado al túnel dentro de la infraestructura de Cloudflare.

Conceptualmente:

```text
cloud.example.com
       │
       ▼
Cloudflare
       │
       ▼
Tunnel
```

El cliente no necesita conocer:

```text
SERVER_LAN_IP
```

ni ninguna dirección interna de Docker.

Esto mantiene separadas:

```text
identidad pública
       │
       └── cloud.example.com

identidad LAN
       │
       └── SERVER_LAN_IP

red Docker
       │
       └── app
```

---

## 28. Privacidad de la documentación

En el repositorio público no se incluyen:

```text
dominio real
token del túnel
UUID del túnel
identificadores de cuenta
direcciones internas reales innecesarias
```

Se utilizan valores como:

```text
cloud.example.com
<CLOUDFLARE_TUNNEL_TOKEN>
<TUNNEL_ID>
<DOCKER_NETWORK_CIDR>
```

El dominio real no es necesario para reproducir la arquitectura.

La documentación privada de recuperación puede conservar la correspondencia con los valores reales.

---

## 29. Recuperación del túnel

En caso de reconstruir el servidor, el túnel no depende del contenedor anterior.

El procedimiento conceptual es:

```text
nuevo Debian
     │
     ▼
Docker
     │
     ▼
Compose
     │
     ▼
recuperar token
     │
     ▼
cloudflared
     │
     ▼
Tunnel existente
```

Al tratarse de un túnel administrado remotamente, la configuración pública permanece gestionada desde Cloudflare.

El nuevo conector necesita disponer de las credenciales válidas para asociarse al túnel.

Esto será desarrollado en:

```text
docs/12-recuperacion-desastres.md
```

---

## 30. Dependencias

El acceso remoto depende de varias capas:

```text
DNS público
     │
     ▼
Cloudflare
     │
     ▼
Tunnel
     │
     ▼
cloudflared
     │
     ▼
red Docker
     │
     ▼
Nextcloud
```

Un fallo en cualquiera puede producir una indisponibilidad externa.

Por ello el diagnóstico debe realizarse desde fuera hacia dentro.

---

## 31. Diagnóstico del acceso remoto

Un procedimiento básico es comprobar:

```text
1. ¿Resuelve el dominio?
        │
2. ¿Responde Cloudflare?
        │
3. ¿Tunnel está conectado?
        │
4. ¿cloudflared está running?
        │
5. ¿app está running?
        │
6. ¿cloudflared alcanza app:80?
        │
7. ¿Nextcloud funciona localmente?
```

Comandos locales útiles:

```bash
docker compose ps
```

```bash
docker compose logs cloudflared
```

```bash
docker compose logs app
```

y:

```bash
curl -I http://localhost:8080
```

cuando corresponda.

Una prueba externa puede utilizar:

```bash
curl -I https://cloud.example.com
```

o un dispositivo conectado mediante una red diferente de la LAN.

---

## 32. Interpretación de un 502

Un:

```text
502 Bad Gateway
```

en el dominio público no significa necesariamente que Cloudflare Tunnel esté caído.

Puede significar que Cloudflare alcanza `cloudflared`, pero el conector no puede obtener una respuesta válida del servicio de origen.

Por ello deben comprobarse:

```text
cloudflared
     │
     ▼
DNS Docker
     │
     ▼
app
     │
     ▼
Nextcloud
```

La incidencia ocurrida después del reboot es un ejemplo de este comportamiento.

---

## 33. Monitorización

El contenedor utiliza:

```yaml
restart: unless-stopped
```

para recuperarse automáticamente después de reinicios.

Actualmente la monitorización principal de la infraestructura local se realiza mediante:

- systemd;
- Docker;
- logs;
- alertas por fallo de tareas críticas;
- comprobaciones funcionales.

Cloudflare proporciona adicionalmente información sobre el estado de sus conectores desde su plataforma.

Una futura mejora podría incorporar alertas específicas ante indisponibilidad prolongada del túnel.

---

## 34. Alta disponibilidad

La arquitectura actual utiliza un único servidor físico y un único contenedor `cloudflared`.

Cloudflare Tunnel admite múltiples conectores asociados a un mismo túnel, lo que puede proporcionar redundancia de conectividad.

Sin embargo:

```text
2 cloudflared
      │
      ≠
2 Nextcloud
```

Ejecutar múltiples conectores en el mismo `hpserver` no solucionaría el fallo físico del servidor.

Para obtener redundancia real del servicio sería necesaria también redundancia del origen.

Por ello no se añade complejidad de alta disponibilidad que el hardware actual no puede aprovechar completamente.

---

## 35. Límites de la solución

Cloudflare Tunnel mejora considerablemente la exposición remota, pero no protege frente a todos los escenarios.

No evita por sí mismo:

- fallo del servidor;
- fallo del HDD DATA;
- corrupción de datos;
- eliminación autenticada;
- robo de credenciales de Nextcloud;
- errores de administración;
- pérdida del repositorio Borg;
- pérdida simultánea del servidor y backup local.

Estos riesgos se tratan mediante otras capas del proyecto.

---

## 36. Principios aplicados

### No abrir puertos si no es necesario

El acceso remoto utiliza una conexión iniciada desde el servidor.

### El acceso local debe seguir siendo independiente

Una incidencia externa no debe impedir utilizar Nextcloud desde la LAN.

### El token es una credencial

No se publica ni se almacena en Git.

### El proxy y Nextcloud deben compartir una interpretación coherente del protocolo

Por ello se utilizan `trusted_proxies` y `overwrite*`.

### Las cabeceras se configuran en la dirección correcta

HSTS pertenece a la respuesta hacia el cliente.

### La configuración debe validarse desde fuera

Un túnel aparentemente conectado no demuestra que la aplicación sea utilizable.

### Un 502 se diagnostica hasta el origen

No se atribuye automáticamente a Cloudflare.

---

## 37. Incidencias y lecciones principales

| Incidencia | Causa / diagnóstico | Solución / lección |
| --- | --- | --- |
| 502 después de reboot | Nextcloud `app` no arrancó | Diagnosticar hasta el origen |
| `app` no arrancaba | Bind a IP todavía no asignada | Publicar `8080:80` |
| HSTS no aparecía | Regla aplicada a Request Header | Utilizar Response Header Transform Rule |
| HSTS configurado pero no visible inicialmente | Se comprobó la configuración, no todavía la respuesta efectiva | Validar mediante respuesta HTTP real |
| HTTPS remoto interfería con LAN | Nextcloud forzaba HTTPS globalmente | `overwritecondaddr` |
| Riesgo de exponer token | Token permite ejecutar un conector | Mantenerlo fuera de Git |

---

## 38. Resumen

La arquitectura remota final es:

```text
                        INTERNET
                            │
                            │ HTTPS
                            ▼
                       Cloudflare
                            │
                            │ Tunnel
                            ▼
                       cloudflared
                            │
                            │ app:80
                            ▼
                       Nextcloud
                            ▲
                            │
                            │ :8080
                            │
                           LAN
```

No existe una redirección pública directa desde el router hacia Nextcloud.

La conexión externa se construye mediante:

```text
DNS
+
HTTPS
+
Cloudflare
+
Tunnel
+
cloudflared
+
Docker
+
Nextcloud
```

La solución mantiene al mismo tiempo un camino local independiente.

Esto permite disponer de acceso remoto sin convertir el puerto HTTP local de Nextcloud en un servicio publicado directamente hacia Internet.

---

## 39. Referencias

### Cloudflare Tunnel

- **Cloudflare Tunnel — Documentation**  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/

  - Arquitectura de Cloudflare Tunnel.
  - Conexiones `outbound-only`.
  - Funcionamiento de `cloudflared`.
  - Ausencia de necesidad de una IP públicamente enrutable.

- **Create a remotely-managed tunnel — Dashboard**  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/

  - Creación de túneles.
  - Instalación del conector.
  - Gestión desde Cloudflare.

- **Tunnel run parameters**  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/run-parameters/

  - `tunnel run`.
  - `--token`.
  - Parámetros del conector.
  - Variables de entorno.

### Seguridad del token

- **Tunnel permissions**  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/remote-tunnel-permissions/

  - Permisos.
  - Obtención del token.
  - Seguridad de túneles administrados remotamente.

  La documentación oficial indica que quien disponga del token puede ejecutar el túnel, por lo que debe tratarse como una credencial.

### Firewall y conectividad

- **Tunnel with firewall**  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/

  - Modelo de tráfico saliente.
  - Puerto `7844`.
  - QUIC.
  - HTTP/2.
  - Configuración con firewall restrictivo.

### Configuración del origen

- **Origin parameters**  
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/origin-parameters/

  - Configuración del servicio de origen.
  - Parámetros HTTP.
  - TLS.
  - Conexiones entre `cloudflared` y la aplicación.

### Transform Rules

- **Cloudflare Transform Rules**  
  https://developers.cloudflare.com/rules/transform/

  - Transformaciones de peticiones.
  - Transformaciones de respuestas.
  - Modificación de cabeceras HTTP.

- **Request Header Transform Rules**  
  https://developers.cloudflare.com/rules/transform/request-header-modification/

  - Modificación de cabeceras enviadas hacia el servidor de origen.

- **Response Header Transform Rules**  
  https://developers.cloudflare.com/rules/transform/response-header-modification/

  - Modificación de cabeceras enviadas hacia el cliente.
  - Adición y modificación de cabeceras HTTP.

- **Create a Response Header Transform Rule**  
  https://developers.cloudflare.com/rules/transform/response-header-modification/create-dashboard/

  - Creación desde el dashboard.
  - `Set static`.
  - `Add static`.
  - Expresiones de filtrado por hostname.

### Nextcloud y reverse proxy

- **Nextcloud — Reverse proxy**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/reverse_proxy_configuration.html

  - `trusted_proxies`.
  - `overwriteprotocol`.
  - `overwritecondaddr`.
  - `overwrite.cli.url`.

- **Nextcloud — Security setup warnings**  
  https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/security_setup_warnings.html

  - Advertencias de seguridad.
  - HTTPS.
  - Cabeceras.
  - Configuración detrás de proxy.

> Las referencias describen el comportamiento oficial de Cloudflare Tunnel, Transform Rules y Nextcloud. La elección de la arquitectura, las pruebas realizadas y las incidencias descritas corresponden a la implementación real de `hpserver`.