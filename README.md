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

