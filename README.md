# Parcial práctico COMM: despliegue multi-contenedor

Proyecto de cinco servicios: Nginx, Joomla, PostgreSQL, Jupyter y Grafana.
La única publicación al host es `80:80` en Nginx. Grafana y Jupyter se abren
por rutas del proxy: `/grafana/` y `/jupyter/`.

## Requisitos previos

- Docker Desktop iniciado (en Windows, con contenedores Linux habilitados).
- Git para clonar y publicar el repositorio.
- Puerto 80 libre en el equipo.
- Acceso a Internet durante la primera construcción/descarga de imágenes.

## Arranque limpio

Desde la carpeta del repositorio:

```powershell
Copy-Item .env.example .env
docker compose up -d --build
```

En Linux/macOS, para copiar el archivo usa `cp .env.example .env`.
El comando `docker compose up -d --build` construye las imágenes personalizadas
de Joomla (controlador PHP de PostgreSQL) y Jupyter.

URLs locales:

- Joomla: <http://localhost/>
- Jupyter: <http://localhost/jupyter/> (token definido en `.env`)
- Grafana: <http://localhost/grafana/> (usuario/clave definidos en `.env`)

Grafana aprovisiona automáticamente la fuente PostgreSQL y el dashboard
**Actividad de Joomla y PostgreSQL**. Jupyter abre el cuaderno precargado
`analisis_datos.ipynb`.

## Comprobación

```powershell
docker compose ps
docker compose logs --tail=100
```

En Grafana, abre **Dashboards > Comunicaciones > Actividad de Joomla y
PostgreSQL**. Para detener contenedores sin eliminar datos persistentes:

```powershell
docker compose down
```

Para reiniciar desde cero, ten presente que `docker compose down -v` elimina
los volúmenes de datos.

Consulta [INFORME.md](INFORME.md) para el análisis técnico y el procedimiento
de demostración.
