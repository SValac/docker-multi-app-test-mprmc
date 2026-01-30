# Comandos Docker - MAPREMEC

Referencia rápida de comandos para manejar los contenedores.

---

## Iniciar servicios

```bash
# Iniciar todos los contenedores
docker-compose up

# Iniciar en background (sin ver logs)
docker-compose up -d

# Iniciar con rebuild de imágenes
docker-compose up --build

# Iniciar con rebuild sin caché
docker compose build --no-cache

# Iniciar con hot reload (sincroniza cambios automáticamente)
docker-compose watch
```

---

## Detener servicios

```bash
# Detener todos los contenedores
docker-compose down

# Detener todos los contenedores (alternativa)
docker stop $(docker ps -q)

# Eliminar todos los contenedores (detenidos)
docker rm $(docker ps -a -q)

# Detener y eliminar todas las imágenes
docker-compose down --rmi all

# Detener y eliminar imágenes locales
docker-compose down --rmi local

# Detener y eliminar volúmenes (BORRA DATOS DE MySQL)
docker-compose down -v
```

---

## Ver estado y logs

```bash
# Ver contenedores corriendo
docker-compose ps

# Ver todos los contenedores (incluyendo detenidos)
docker ps -a

# Ver logs de todos los servicios
docker-compose logs -f

# Ver logs de un servicio específico
docker-compose logs -f nuxt-app
docker-compose logs -f backend
docker-compose logs -f mysql
```

---

## Comandos Laravel (Backend)

```bash
# Ejecutar migraciones
docker-compose exec backend php artisan migrate

# Ejecutar seeders
docker-compose exec backend php artisan db:seed

# Refrescar migraciones (borra y recrea tablas)
docker-compose exec backend php artisan migrate:fresh --seed

# Limpiar caché
docker-compose exec backend php artisan config:clear
docker-compose exec backend php artisan cache:clear
docker-compose exec backend php artisan route:clear

# Crear un modelo con migración
docker-compose exec backend php artisan make:model NombreModelo -m

# Entrar al contenedor (bash)
docker-compose exec backend bash

# Ejecutar tinker
docker-compose exec backend php artisan tinker
```

---

## Comandos MySQL

```bash
# Conectar a MySQL desde terminal
docker-compose exec mysql mysql -u mapremec_user -pmapremec_password mapremec

# Ejecutar query directamente
docker-compose exec mysql mysql -u mapremec_user -pmapremec_password -e "SHOW TABLES;" mapremec

# Backup de la base de datos
docker-compose exec mysql mysqldump -u mapremec_user -pmapremec_password mapremec > backup.sql

# Restaurar backup
docker-compose exec -T mysql mysql -u mapremec_user -pmapremec_password mapremec < backup.sql
```

---

## Comandos Frontend (Nuxt)

```bash
# Entrar al contenedor
docker-compose exec nuxt-app sh

# Instalar un paquete npm
docker-compose exec nuxt-app npm install nombre-paquete

# Ejecutar linter
docker-compose exec nuxt-app npm run lint

# Ejecutar tests
docker-compose exec nuxt-app npm run test
```

---

## Reiniciar servicios

```bash
# Reiniciar un servicio específico
docker-compose restart backend
docker-compose restart nuxt-app
docker-compose restart mysql

# Reiniciar todos
docker-compose restart
```

---

## Reconstruir imágenes

```bash
# Reconstruir un servicio específico
docker-compose build backend
docker-compose build nuxt-app

# Reconstruir sin usar caché
docker-compose build --no-cache backend

# Reconstruir todos los servicios
docker-compose build
```

---

## Limpieza

```bash
# Eliminar contenedores detenidos
docker container prune

# Eliminar imágenes no usadas
docker image prune

# Eliminar volúmenes no usados
docker volume prune

# Eliminar TODO (contenedores, imágenes, volúmenes, redes)
docker system prune -a

# Eliminar una imagen específica
docker rmi nombre-imagen
docker rmi mapremec-nuxt-app --force
```

---

## Información del sistema

```bash
# Ver información de Docker
docker info

# Ver uso de disco
docker system df

# Ver redes
docker network ls

# Ver volúmenes
docker volume ls
```

---

## URLs de acceso

| Servicio    | URL                   |
| ----------- | --------------------- |
| Frontend    | http://localhost:3000 |
| Backend API | http://localhost:8000 |
| MySQL       | localhost:3307        |

---

## Credenciales MySQL

| Campo    | Valor                                           |
| -------- | ----------------------------------------------- |
| Host     | localhost (desde PC) / mysql (desde contenedor) |
| Puerto   | 3307                                            |
| Database | mapremec                                        |
| Usuario  | mapremec_user                                   |
| Password | mapremec_password                               |
