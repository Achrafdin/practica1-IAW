☁️ NimbusDocs - Práctica de Servidores y Proxies

Bienvenido/a al repositorio de la Práctica 1 del módulo de Implantación de Aplicaciones Web.

Este proyecto consiste en diseñar y desplegar desde cero la infraestructura web para NimbusDocs, una empresa ficticia que aloja documentación técnica de dos marcas distintas compartiendo el mismo hardware.

🎯 Objetivo del Proyecto

El objetivo principal es levantar un entorno completo y seguro utilizando contenedores. La arquitectura se divide en dos capas principales:

Capa Frontal (Nginx): Actúa como proxy inverso. Es el único punto de entrada público, encargado de recibir el tráfico HTTPS, gestionar los certificados de seguridad (TLS) y aplicar reglas de protección contra ataques.

Capa Backend (Apache): Servidor web interno protegido que aloja el contenido real de las dos marcas mediante Virtual Hosts, totalmente aislado del exterior.

🛠️ Tecnologías Utilizadas

Docker & Docker Compose: Para la contenerización y orquestación de los servicios.

Nginx: Proxy inverso, terminación TLS y seguridad perimetral (Rate Limiting).

Apache HTTP Server: Servidor web backend multi-sitio.

OpenSSL: Para la generación de certificados de seguridad.

🚀 Retos Principales a Resolver

Enrutamiento Inteligente: Conseguir que Nginx pase las peticiones al backend de forma que Apache sepa exactamente qué marca debe mostrar.

Seguridad: Implementar certificados ECDSA, forzar conexiones seguras (HSTS) y evitar ataques de denegación de servicio o scraping limitando el número de peticiones por segundo.

Resolución de Problemas (Troubleshooting): Diagnosticar y reparar fallos de configuración provocados intencionadamente en la red, en los certificados o en la comunicación entre contenedores.

[ 👤 Cliente web ]
               │
               │ (Tráfico exterior 443/HTTPS)
               ▼
  +--------------------------+
  |  🛡️ Nginx (Proxy)        |
  |  Bloquea exceso de reqs  |
  +--------------------------+
               :
               : (Tráfico interno Docker)
               : (Línea intercortada)
               ▼
  +--------------------------+
  |  ⚙️ Apache (Backend)     |
  |  Puerto oculto (Ej: 8080)|
  +--------------------------+
          /          \
         /            \
 [ 🌐 Marca 1 ]  [ 🔒 Marca 2 ]

Práctica realizada para el curso 2026/2027.


# Memoria de la Práctica: Infraestructura Web con Proxy Inverso (Nginx + Apache)

El objetivo principal de esta práctica ha sido simular un entorno empresarial real alojando varias páginas web (una marca pública y una intranet privada) en un mismo servidor backend (Apache), pero protegiéndolas a través de un intermediario de seguridad o proxy inverso (Nginx). 

A continuación detallo todo el proceso que he seguido, los comandos utilizados y cómo he solucionado los problemas encontrados.

## Fase 1: Creación del entorno y configuración de Apache (Backend)

Lo primero que he hecho ha sido crear un archivo `docker-compose.yml` para levantar tres contenedores en la misma red: Nginx, Apache y un cliente Ubuntu. Añadí la instrucción `command: sleep infinity` a todos para evitar que arrancaran sus servicios por defecto y poder encenderlos manualmente.

### Comandos utilizados:

*   `docker compose up -d`
    *   *Por qué lo he usado:* Lee el archivo YAML y crea los contenedores dejándolos funcionando en segundo plano (`-d`).
*   `docker ps`
    *   *Por qué lo he usado:* Muestra la lista de contenedores activos. Me sirvió para comprobar que la infraestructura se había levantado correctamente.
*   `docker exec -it nimbus_apache bash`
    *   *Por qué lo he usado:* Abre una terminal interactiva dentro del contenedor de Apache para poder configurarlo por dentro.
*   `a2ensite nimbus.conf`
    *   *Por qué lo he usado:* Activa el archivo de configuración con los VirtualHosts que he creado (la web pública, la interna y el bloqueo por defecto).
*   `service apache2 start`
    *   *Por qué lo he usado:* Como el contenedor estaba "durmiendo", este comando arranca el motor del servidor web Apache para que empiece a escuchar en el puerto 8080.

*Nota sobre errores:* Durante esta fase, Apache devolvía errores 502 y 404 porque, al clonar el repositorio en otro equipo, el archivo `nimbus.conf` y los archivos `.html` estaban vacíos (0 bytes). Lo solucioné rellenando los archivos con el código correspondiente y reiniciando el servicio (`service apache2 restart`).

---

## Fase 2: Configuración de Nginx y Certificados de Seguridad

En esta fase he configurado Nginx para que actúe como proxy inverso: da la cara a Internet en los puertos 80 y 443, recibe la conexión segura HTTPS, la descifra y envía el tráfico al puerto 8080 de Apache.

### Comandos utilizados:

*   `openssl req -new -x509 -key nimbus.key -out nimbus.crt -days 365 -subj "/CN=docs.nimbus-public.local"`
    *   *Por qué lo he usado:* Genera las claves criptográficas y el certificado SSL autofirmado necesarios para habilitar el HTTPS.
*   `docker exec nimbus_nginx nginx`
    *   *Por qué lo he usado:* Arranca el motor web de Nginx, ya que el contenedor también estaba en modo `sleep infinity`.
*   `docker restart nimbus_nginx`
    *   *Por qué lo he usado:* Reinicia el contenedor por completo. Lo utilicaje para limpiar la caché de red cuando la comunicación entre los contenedores fallaba.

### La trampa de la cabecera "Host"

Al probar la conexión, Nginx le pasaba la petición a Apache, pero este me devolvía un error **403 Forbidden**. 

El problema era que Nginx enviaba el tráfico mediante la directiva `proxy_pass http://apache:8080;`, pero omitía decirle a Apache qué dominio específico estaba buscando el usuario. Al recibir una petición "huérfana", Apache aplicaba su regla del VirtualHost por defecto, que estaba configurada para denegar todo el acceso por seguridad (`Require all denied`).

Para solucionarlo, añadí la siguiente directiva en la configuración de Nginx:
```nginx
proxy_set_header Host $host;

### Fase 3: Pruebas de acceso y seguridad

Para comprobar el enrutamiento y las reglas de seguridad, utilicé el contenedor cliente (Ubuntu) simulando ser un usuario externo.

**Comandos utilizados:**

*   `docker exec nimbus_cliente apt-get install -y curl`
    *   *Por qué lo he usado:* El contenedor base de Ubuntu no incluía `curl`. Lo instalé para poder hacer peticiones web desde la terminal.
*   `docker exec nimbus_cliente curl -k -i -H "Host: docs.nimbus-public.local" https://nginx/`
    *   *Por qué lo he usado:* Es el comando de diagnóstico principal.
        *   `-k`: Ignora las advertencias del certificado autofirmado.
        *   `-i`: Muestra las cabeceras HTTP de respuesta para ver exactamente los códigos de estado (200, 403, 502).
        *   `-H "Host: ..."`: Inyecta manualmente el dominio en la petición para decirle a Nginx qué web queremos ver.
        *   `https://nginx/`: Apunta a la IP interna que Docker le ha asignado al contenedor llamado "nginx".

**Resultados de la validación:**

1.  **Acceso público (`docs.nimbus-public.local`):** El servidor devolvió un **HTTP 200 OK** mostrando correctamente el HTML de la web pública.
2.  **Acceso a intranet (`admin.nimbus-internal.local`):** El servidor devolvió un **HTTP 200 OK** mostrando el HTML de la web interna privada.
3.  **Prueba de seguridad (`/db-passwords.ini`):** Al intentar acceder al archivo de contraseñas de la intranet, el servidor bloqueó la petición devolviendo un **HTTP 403 Forbidden**. Esto confirma que las directivas de seguridad configuradas en el VirtualHost de Apache funcionan correctamente y protegen los archivos sensibles.

