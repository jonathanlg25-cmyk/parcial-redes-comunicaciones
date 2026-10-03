# Informe técnico: despliegue, redes y modelo OSI

## 1. Topología y flujo de información

```mermaid
flowchart LR
  Browser[Cliente del host] -->|HTTP TCP 80| N[Nginx :80\nfrontend_net]
  N -->|HTTP interno :80| J[Joomla\nfrontend_net + backend_net]
  N -->|HTTP + WebSocket :8888| Y[Jupyter\nfrontend_net + backend_net]
  N -->|HTTP :3000| G[Grafana\nfrontend_net + backend_net]
  J -->|PostgreSQL TCP 5432| DB[(PostgreSQL\nbackend_net interna)]
  G -->|Consultas de lectura TCP 5432| DB
  Y -->|Consultas TCP 5432| DB
  N -. access.log compartido .-> Logs[(Volumen de logs Nginx)]
```

Nginx es el único servicio con un puerto publicado al host. Enruta `/` a Joomla,
`/jupyter/` a Jupyter y `/grafana/` a Grafana. Los servicios usan DNS de Docker
por nombre (`joomla`, `jupyter`, `grafana`, `database`), sin direcciones IP fijas.

`frontend_net` conecta al proxy y a los servicios web. `backend_net` es una red
bridge marcada `internal: true`; conecta Joomla, PostgreSQL, Jupyter y Grafana
para consultas de datos. PostgreSQL solo pertenece a backend_net, no publica el
puerto 5432 y no tiene una ruta directa a la red externa. Los volúmenes nombrados
persisten PostgreSQL, Joomla y Grafana. El cuaderno se monta desde el repositorio
para estar disponible en cada despliegue.

Grafana tiene una fuente PostgreSQL aprovisionada desde YAML y un dashboard JSON
con consultas de solo lectura a `pg_stat_database` y `pg_database_size()`. Así se
muestran conexiones y transacciones acumuladas, más el tamaño de la base. Los
logs de Nginx y Apache de Joomla se conservan en volúmenes compartidos; el
dashboard entregado usa las métricas PostgreSQL y no interpreta esos archivos
como una serie de logs.

## 2. Análisis del modelo OSI

### Capa 7: Aplicación

El navegador solicita HTTP a Nginx. El proxy conserva `Host` para que la
aplicación conozca el nombre solicitado. `X-Forwarded-For` comunica la dirección
del cliente original (Nginx añade el salto observado), y `X-Forwarded-Proto`
indica el esquema externo. En un despliegue HTTPS, esto permite que Joomla y
Grafana generen enlaces correctos detrás del proxy.

Jupyter usa HTTP para la interfaz y WebSocket para el canal persistente entre el
navegador y el kernel. El cliente envía `Upgrade: websocket`; Nginx opera con
HTTP/1.1 y retransmite `Upgrade` y `Connection` para aceptar la transición del
protocolo. Sin esos encabezados, la página podría cargar mientras fallan las
celdas del notebook.

PostgreSQL implementa un protocolo cliente-servidor propio sobre TCP. El cliente
negocia conexión/autenticación y luego intercambia mensajes de consulta y
respuesta (por ejemplo, SQL y filas tipadas). Joomla usa el controlador PHP
`pgsql`/`pdo_pgsql`; Jupyter usa `psycopg2`; Grafana usa su datasource PostgreSQL
nativo. Apache produce logs de acceso en formato combinado, con IP, marca de
tiempo, línea de solicitud, código HTTP, bytes, referer y agente de usuario.
Nginx registra campos equivalentes y añade la dirección reenviada.

### Capa 4: Transporte

HTTP del host termina en TCP/80 de Nginx. Entre Nginx y Joomla se usa TCP/80;
Nginx llega a Jupyter por TCP/8888 y a Grafana por TCP/3000. Joomla, Grafana y
Jupyter usan TCP/5432 para PostgreSQL. Esos puertos internos no están publicados
al host. Cada conexión TCP mantiene estado, números de secuencia y
retransmisiones; keep-alive permite reutilizar conexiones HTTP, mientras las
conexiones PostgreSQL persistentes/pooling reducen el costo de crear una sesión
por operación. El límite de conexiones de PostgreSQL debe considerar los
procesos concurrentes de Joomla y las herramientas analíticas.

### Capa 3: Red

Docker crea subredes IP para cada bridge y conecta cada contenedor con una
interfaz virtual. Nginx solo publica el puerto 80. El aislamiento de
`backend_net` evita que PostgreSQL se conecte directamente a redes externas. El
DNS embebido del daemon Docker, normalmente accesible desde contenedores en
`127.0.0.11`, resuelve los nombres de servicio a IPs internas, que pueden
cambiar al recrear contenedores.

El motor de Docker configura reglas de reenvío y NAT en el host para dirigir
conexiones entrantes del puerto 80 al contenedor Nginx. El tráfico interno entre
servicios se enruta por sus bridges; publicar PostgreSQL rompería el requisito
de mantenerlo privado y no se hace en este compose.

### Capa 2: Enlace de datos

En Linux, cada extremo de una interfaz virtual `veth` vive en un espacio de red
distinto: uno dentro del contenedor y el otro conectado al bridge `br-*` del
host. El bridge conmuta tramas Ethernet entre miembros de la misma red Docker.
Cuando un contenedor debe enviar tráfico a otro cuya MAC desconoce, usa ARP para
resolver la IP local a una dirección MAC; el bridge reenvía la trama al puerto
correspondiente. Docker abstrae estos detalles, pero son la base del enlace entre
contenedores en una misma red.

## 3. Guía de verificación y demostración

1. Copiar `.env.example` a `.env` y ejecutar `docker compose up -d --build`.
2. Esperar a que PostgreSQL pase el healthcheck y comprobar `docker compose ps`.
3. Abrir `http://localhost/`, completar la instalación inicial de Joomla si la
   imagen no pudo automatizarla, y navegar varias páginas para generar tráfico.
4. Abrir `http://localhost/grafana/`, iniciar sesión con las credenciales de
   `.env` y revisar el dashboard aprovisionado **Actividad de Joomla y
   PostgreSQL**. Los paneles presentan transacciones, conexiones y tamaño de BD.
5. Abrir `http://localhost/jupyter/`, autenticarse con `JUPYTER_TOKEN` de `.env`,
   y ejecutar todas las celdas de `analisis_datos.ipynb`. El cuaderno consulta
   PostgreSQL, muestra estadísticas de actividad y grafica transacciones.
6. Ver los registros de los servicios con `docker compose logs --tail=100`.

## 4. Decisiones y limitaciones de la demostración

El volumen `nginx_logs` hace que el access log esté disponible en Nginx y se
monte de solo lectura en Grafana para una futura integración de logs. Los
dashboards actuales consultan directamente PostgreSQL, por lo que no requieren
un sexto contenedor de recolección. Las contraseñas del `.env.example` son
valores didácticos; reemplazarlas antes de usar el sistema fuera del ejercicio.
