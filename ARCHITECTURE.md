# Architecture Overview

## 1) Estructura de carpetas principal

novus-back/
alembic/
env.py
script.py.mako
versions/
app/
auth/
almacen/
caja/
core/
documentos/
facturacion/
finanzas/
identidad/
infraestructura/
integracion_sunat/
maestro/
modules/
presentation/
proveedores/
public/
shared/
ventas/
workers/
main.py
scripts/
storage/
certificados/
templates/
alembic.ini
Dockerfile
readme.md
requirements.txt

novus-front/
.angular/
.claude/
.gemini/
.vscode/
dist/
node_modules/
public/
src/
angular.json
Dockerfile
package.json
tsconfig.app.json
tsconfig.json
tsconfig.spec.json
README.md

novus-portal-clientes/
.angular/
.claude/
.gemini/
.vscode/
dist/
node_modules/
public/
src/
angular.json
Dockerfile
package.json
postcss.config.mjs
README.md
tsconfig.app.json
tsconfig.json

novus-devops/
.claude/
.env
.git/
docker-compose.arm.yml
docker-compose.yml
readme.md

---

## 2) Resumen de tecnologías clave

### Backend (`novus-back`)

- Python + FastAPI
- Uvicorn como servidor ASGI
- Pydantic v2 / Pydantic Settings
- SQLAlchemy 2.x en modo async
- asyncpg para PostgreSQL
- Alembic para migraciones
- Redis + Celery para procesamiento asíncrono
- Redis y WebSockets / ASGI
- XML / SUNAT: lxml, signxml, pyOpenSSL, cryptography
- HTTP async: httpx
- PDF/QR: weasyprint, reportlab, qrcode[pil], Pillow
- Seguridad: python-jose, passlib[bcrypt], bcrypt
- Form-data multipart: python-multipart

### Frontend (`novus-front`)

- Angular 21
- TypeScript 5.9
- RxJS 7.8
- qrcode para generación de códigos QR
- Vitest para pruebas
- npm@11.6.2 como package manager

### Portal de clientes (`novus-portal-clientes`)

- Angular 21
- TypeScript 5.9
- Tailwind CSS 4 + PostCSS
- Autoprefixer
- Vitest para pruebas
- npm@11.6.2 como package manager

### DevOps / Contenedores (`novus-devops`)

- Docker Compose definido en `docker-compose.yml`
- PostgreSQL 16-alpine
- Redis 7-alpine
- Volúmenes Docker para datos y certificados
- Uso de env_file para variables de configuración

---

## 3) Endpoints principales expuestos en FastAPI

### Entrada principal

- `GET /health`
- API raíz: todas las rutas principales se exponen bajo `/api/v1`

### Grupos de rutas principales

- `/api/v1/auth` — autenticación y token
- `/api/v1/usuarios` — gestión de usuarios
- `/api/v1/maestro/empresas` — CRUD de empresas
- `/api/v1/maestro/sedes` — CRUD de sedes
- `/api/v1/identidad/ruc/{ruc}` — consulta de identidad por RUC
- `/api/v1/identidad/dni/{dni}` — consulta de identidad por DNI
- `/api/v1/proveedores` — CRUD de proveedores
- `/api/v1/caja` — operaciones de caja y salud de caja
- `/api/v1/ventas` — registro, consulta, edición y anulación de ventas
- `/api/v1/facturacion/{venta_id}/estado` — estado de comprobante electrónico
- `/api/v1/finanzas/jornadas/abrir` — abrir jornada financiera
- `/api/v1/finanzas/jornadas/cerrar` — cerrar jornada financiera
- `/api/v1/finanzas/jornadas/movimientos` — movimientos de jornada
- `/api/v1/almacen` — búsqueda y gestión de productos en almacén
- `/api/v1/almacen/presentaciones` — listado de presentaciones
- `/api/v1/almacenes` — listado y creación de almacenes
- `/api/v1/almacenes/movimientos` — registro de movimientos de inventario
- `/api/v1/public` — consultas públicas / endpoints abiertos
- `/api/v1/ventas/health` — salud del módulo de ventas

> Nota: `app/main.py` incluye routers de `auth`, `maestro`, `almacen`, `caja`, `ventas`, `facturacion`, `finanzas`, `identidad`, `productos_pos`, `realtime`, `proveedores` y `public`.

---

## 4) Configuración de red de `docker-compose.yml`

### Servicios principales

- `novus-back`
  - Build: `../backend`
  - Comando: `uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload`
  - Puerto expuesto: `8001:8000`
  - Env file: `.env`
  - Volúmenes:
    - `../backend:/app`
    - `novus_certificados:/app/storage/certificados`
  - Depende de `novus-db` y `redis` con healthcheck

- `celery_worker`
  - Build: `../backend`
  - Comando: `celery -A app.core.celery_app worker --loglevel=info`
  - Volúmenes:
    - `../backend:/app`
    - `novus_certificados:/app/storage/certificados`
  - Depende de `novus-db` y `redis`

- `celery_beat`
  - Build: `../backend`
  - Comando: `celery -A app.core.celery_app beat --loglevel=info`
  - Volúmenes:
    - `../backend:/app`
  - Depende de `novus-db` y `redis`

- `novus-db`
  - Imagen: `postgres:16-alpine`
  - Puerto expuesto: `${POSTGRES_EXTERNAL_PORT:-5433}:5432`
  - Volumen: `novus_pg_data:/var/lib/postgresql/data`
  - Healthcheck: `pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-empower}`

- `redis`
  - Imagen: `redis:7-alpine`
  - Puerto expuesto: `${REDIS_PORT:-6379}:6379`
  - Comando: `redis-server --appendonly yes`
  - Volumen: `novus_redis_data:/data`
  - Healthcheck: `redis-cli ping`

### Volúmenes Docker

- `novus_pg_data`
- `novus_redis_data`
- `novus_certificados`

### Comentarios importantes

- Las secciones `novus-front` y `novus-portal-clientes` están presentes en el archivo como servicios comentados.
- El backend se monta en modo hot-reload sobre el contenedor para desarrollo.
- Redis se usa como broker y backend para Celery, y el `novus-back` depende de su disponibilidad.
