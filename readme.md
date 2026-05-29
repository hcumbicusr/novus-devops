-- Detener y borrar volumenes
docker compose down -v

-- compilar nuevamente las imagenes del proyecto
docker compose build

-- iniciar los containers
docker compose up -d

-- logs de un container específico: db
docker compose logs -f novus-db

-- Crear una migración
docker compose exec novus-back alembic revision --autogenerate -m "initial_schema"

-- Migrar el archivo previamente creado hacia la bd
docker compose exec novus-back alembic upgrade head

-- Poblar la BD con data inicial
docker compose exec novus-back python -m scripts.seed_erp

-- logs de un container específico: novus-back
docker compose logs -f novus-back

--- para subir a Docker Hub y desplegar en rasbperry pi (ARM):

docker buildx build --platform linux/arm64 -t hcumbicusr/novus-back:latest --push ../backend

docker buildx build --platform linux/arm64 -t hcumbicusr/novus-front:latest --push ../novus-front

docker buildx build --platform linux/arm64 -t hcumbicusr/novus-portal-clientes:latest --push ../portal-clientes

--- desde raspberry pi:
-> copiar el archivo docker-compose.arm.yml

docker compose pull

docker compose up -d
