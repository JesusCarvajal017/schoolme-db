# SchoolMe DB por ambientes

## Levantar local (sin Jenkins)
cd main
docker compose -f compose.yml --env-file .env-main up -d

cd ../staging
docker compose -f compose.yml --env-file .env-staging up -d

cd ../qa
docker compose -f compose.yml --env-file .env-qa up -d

cd ../dev
docker compose -f compose.yml --env-file .env-dev up -d

## Conexión (desde tu máquina)
# main:  localhost:10000 / DB=schoolme_main / user=postgres / pass=SchoolMe.28102025
# staging: localhost:10001 / DB=schoolme_staging / ...
# qa:  localhost:10002 / DB=schoolme_qa
# dev: localhost:10003 / DB=schoolme_dev

## Bajar y resetear
docker compose -f compose.yml --env-file .env-<env> down -v
