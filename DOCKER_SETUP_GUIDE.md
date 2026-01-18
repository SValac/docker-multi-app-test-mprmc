# Guía de Implementación Docker - MAPREMEC

## Fecha de inicio: 17 de enero de 2026
## Última actualización: 18 de enero de 2026

---

## Índice
1. [Introducción](#introducción)
2. [Arquitectura del proyecto](#arquitectura-del-proyecto)
3. [Servicios implementados](#servicios-implementados)
4. [Archivos creados](#archivos-creados)
5. [Estado actual](#estado-actual)
6. [Comandos útiles](#comandos-útiles)
7. [Docker Watch (Hot Reload)](#docker-watch-hot-reload)

---

## Introducción

Este documento detalla la implementación de Docker para el proyecto MAPREMEC, que consiste en:
- **Backend**: Laravel 10 con PHP 8.2
- **Frontend**: Nuxt 3 con Node 20
- **Base de datos**: MySQL 9.5

### ¿Por qué Docker?

Docker permite ejecutar ambos proyectos con todas sus dependencias en contenedores aislados, sin contaminar el sistema local.

**Beneficios:**
- No necesitas instalar PHP, Composer, MySQL, Node.js localmente
- Mismo entorno para desarrollo y producción
- Fácil de compartir con el equipo
- Datos persistentes

---

## Arquitectura del proyecto

```
Mapremec/
├── MAPREMEC_BACK/          # Backend Laravel
│   ├── Dockerfile          # Receta para construir imagen de Laravel
│   ├── .dockerignore       # Archivos a excluir del build
│   ├── .env.docker         # Variables de entorno para Docker
│   └── docker-entrypoint.sh # Script de inicialización
│
├── MAPREMEC_FRONT/         # Frontend Nuxt
│   ├── Dockerfile          # Receta para construir imagen de Nuxt
│   └── .dockerignore       # Archivos a excluir del build
│
├── docker-compose.yml      # Orquestador de servicios
└── DOCKER_SETUP_GUIDE.md   # Este documento
```

---

## Servicios implementados

### 1. MySQL (Base de datos)

**Puerto local:** 3307 (mapeado al 3306 del contenedor)
**Imagen:** mysql:9.5
**Propósito:** Base de datos para Laravel

#### Configuración:
```yaml
services:
  mysql:
    image: mysql:9.5
    container_name: mysql_container
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: mapremec
      MYSQL_USER: mapremec_user
      MYSQL_PASSWORD: mapremec_password
    ports:
      - "3307:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - mapremec_network
```

#### Conceptos clave:

**Driver bridge (networks):**
- Crea una red virtual privada donde los contenedores pueden comunicarse
- Laravel puede conectarse a MySQL usando el nombre del servicio: `DB_HOST=mysql`
- Similar a un router WiFi que conecta varios dispositivos

**Driver local (volumes):**
- Guarda los datos de MySQL en el disco duro
- Los datos persisten aunque apagues el contenedor
- Similar a un disco duro externo que no se borra

**Variables de entorno:**
- `MYSQL_ROOT_PASSWORD`: Contraseña del usuario root
- `MYSQL_DATABASE`: Nombre de la base de datos (se crea automáticamente)
- `MYSQL_USER` y `MYSQL_PASSWORD`: Usuario adicional

---

### 2. Backend Laravel

**Puerto:** 8000
**Propósito:** API REST para el frontend

#### Archivos creados:

##### 1. `MAPREMEC_BACK/Dockerfile`

Define cómo construir la imagen de Laravel:

```dockerfile
# Imagen base de PHP 8.2 con FPM
FROM php:8.2-fpm

LABEL maintainer="Tu Nombre"

# Instalar dependencias del sistema + netcat para el script
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    libzip-dev \
    netcat-traditional

# Limpiar cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Instalar extensiones de PHP
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip

# Instalar Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Establecer directorio de trabajo
WORKDIR /var/www

# Copiar archivos de la aplicación
COPY . /var/www

# Copiar script de entrada
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

# Dar permisos
RUN chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache

# Exponer puerto 8000
EXPOSE 8000

# Ejecutar script de inicialización
ENTRYPOINT ["docker-entrypoint.sh"]
```

**Explicación línea por línea:**
- `FROM php:8.2-fpm`: Imagen base de PHP 8.2
- `RUN apt-get install`: Instala herramientas del sistema necesarias
- `docker-php-ext-install`: Instala extensiones de PHP que Laravel necesita
- `COPY --from=composer`: Copia Composer desde su imagen oficial
- `WORKDIR /var/www`: Carpeta de trabajo en el contenedor
- `COPY . /var/www`: Copia el código de Laravel al contenedor
- `chown`: Da permisos correctos para que Laravel pueda escribir archivos
- `EXPOSE 8000`: Indica que el contenedor escucha en puerto 8000
- `ENTRYPOINT`: Ejecuta el script de inicialización al iniciar

##### 2. `MAPREMEC_BACK/.env.docker`

Variables de entorno específicas para Docker:

```env
APP_NAME=MAPREMEC
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8000

LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

# Configuración de MySQL para Docker
DB_CONNECTION=mysql
DB_HOST=mysql              # ← Nombre del servicio en docker-compose
DB_PORT=3306
DB_DATABASE=mapremec
DB_USERNAME=mapremec_user
DB_PASSWORD=mapremec_password

BROADCAST_DRIVER=log
CACHE_DRIVER=file
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120
```

**Lo más importante:**
- `DB_HOST=mysql`: Usa el nombre del servicio (no 127.0.0.1)
- Docker traduce automáticamente "mysql" a la IP del contenedor

##### 3. `MAPREMEC_BACK/docker-entrypoint.sh`

Script que automatiza la configuración de Laravel al iniciar:

```bash
#!/bin/bash

# Esperar a que MySQL esté listo
echo "Esperando a que MySQL esté disponible..."
while ! nc -z mysql 3306; do
  sleep 1
done
echo "MySQL está listo!"

# Copiar archivo de entorno si no existe
if [ ! -f .env ]; then
    echo "Copiando .env.docker a .env..."
    cp .env.docker .env
fi

# Instalar dependencias de Composer
echo "Instalando dependencias de Composer..."
composer install --no-interaction --optimize-autoloader --no-dev

# Generar key de aplicación si no existe
if grep -q "APP_KEY=$" .env; then
    echo "Generando APP_KEY..."
    php artisan key:generate --force
fi

# Limpiar y optimizar
echo "Optimizando configuración..."
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Ejecutar migraciones
echo "Ejecutando migraciones..."
php artisan migrate --force

# Iniciar servidor Laravel
echo "Iniciando servidor Laravel..."
php artisan serve --host=0.0.0.0 --port=8000
```

**Qué hace cada parte:**
- `while ! nc -z mysql 3306`: Espera a que MySQL esté listo (evita errores)
- `cp .env.docker .env`: Copia la configuración de Docker
- `composer install`: Instala dependencias de Laravel
- `php artisan key:generate`: Genera clave de encriptación
- `php artisan config:cache`: Optimiza configuración
- `php artisan migrate`: Crea tablas en la base de datos
- `php artisan serve`: Inicia el servidor en puerto 8000

#### Configuración en docker-compose.yml:

```yaml
  backend:
    build:
      context: ./MAPREMEC_BACK
      dockerfile: Dockerfile
    container_name: mapremec_backend
    restart: unless-stopped
    working_dir: /var/www
    volumes:
      - ./MAPREMEC_BACK:/var/www
      - backend_vendor:/var/www/vendor
    ports:
      - "8000:8000"
    depends_on:
      - mysql
    networks:
      - mapremec_network
```

**Explicación:**
- `build:`: Construye imagen desde Dockerfile
- `volumes:`:
  - `./MAPREMEC_BACK:/var/www`: Cambios en tiempo real
  - `backend_vendor:/var/www/vendor`: Volumen separado para dependencias
- `ports: "8000:8000"`: Mapea puerto 8000 del contenedor al 8000 de tu PC
- `depends_on: - mysql`: Inicia MySQL primero

---

### 3. Frontend Nuxt

**Puerto:** 3000
**Imagen:** Node 20 Alpine
**Propósito:** Aplicación frontend Nuxt 3

#### Dockerfile (`MAPREMEC_FRONT/Dockerfile`):

```dockerfile
FROM node:20-alpine AS base
WORKDIR /usr/src/app

# Instalar dependencias
FROM base AS install
RUN mkdir -p /temp/dev
COPY package.json package-lock.json /temp/dev/
RUN cd /temp/dev && npm install

# Copiar node_modules y código
FROM base AS development
COPY --from=install /temp/dev/node_modules node_modules
COPY . .

# Cambiar permisos al usuario node
RUN chown -R node:node /usr/src/app
USER node
EXPOSE 3000/tcp
ENTRYPOINT [ "npm", "run", "dev" ]
```

#### Configuración en docker-compose.yml:

```yaml
nuxt-app:
    build: ./MAPREMEC_FRONT
    container_name: maprmec-frontend
    ports:
        - '3000:3000'
    env_file:
        - ./MAPREMEC_FRONT/.env
    volumes:
        - /usr/src/app/node_modules
    networks:
        - mapremec_network
    restart: unless-stopped
    develop:
        watch:
            - path: ./MAPREMEC_FRONT/
              target: /usr/src/app
              action: sync
              ignore:
                  - 'node_modules/**'
                  - '.nuxt/**'
```

---

## Archivos creados

### Completados ✅

| Archivo | Ubicación | Propósito |
|---------|-----------|-----------|
| `Dockerfile` | `MAPREMEC_BACK/` | Construcción de imagen Laravel |
| `Dockerfile` | `MAPREMEC_FRONT/` | Construcción de imagen Nuxt |
| `.dockerignore` | `MAPREMEC_BACK/` | Optimización de build backend |
| `.dockerignore` | `MAPREMEC_FRONT/` | Optimización de build frontend |
| `.env.docker` | `MAPREMEC_BACK/` | Variables de entorno para Docker |
| `docker-entrypoint.sh` | `MAPREMEC_BACK/` | Script de inicialización de Laravel |
| `docker-compose.yml` | `Mapremec/` (raíz) | Orquestador de servicios |

### Opcionales ⏳

| Archivo | Ubicación | Propósito |
|---------|-----------|-----------|
| `nginx.conf` | `Mapremec/` (raíz) | Configuración Nginx (para producción) |

---

## Estado actual

### ✅ Completado:
1. Servicio MySQL configurado (puerto 3307)
2. Servicio Backend Laravel configurado (puerto 8000)
3. Servicio Frontend Nuxt configurado (puerto 3000)
4. Archivos .dockerignore para optimización de builds
5. Docker Watch configurado para hot reload
6. Stack completo probado y funcionando

### ⚠️ Correcciones realizadas:
- **Puerto MySQL**: Cambiado a 3307 para evitar conflicto con MySQL local
- **Frontend**: Migrado de Bun a Node/npm por compatibilidad con Tailwind

---

## Comandos útiles

### Iniciar servicios
```bash
# Construir e iniciar todos los contenedores
docker-compose up --build

# Iniciar en modo background (detached)
docker-compose up -d

# Ver logs en tiempo real
docker-compose logs -f

# Ver logs de un servicio específico
docker-compose logs -f backend
```

### Gestión de contenedores
```bash
# Detener todos los servicios
docker-compose down

# Detener y eliminar volúmenes (CUIDADO: borra datos)
docker-compose down -v

# Reiniciar un servicio específico
docker-compose restart backend

# Ver estado de los contenedores
docker-compose ps
```

### Ejecutar comandos dentro de contenedores
```bash
# Entrar al contenedor de Laravel
docker-compose exec backend bash

# Ejecutar comando artisan
docker-compose exec backend php artisan migrate

# Entrar a MySQL
docker-compose exec mysql mysql -u mapremec_user -p mapremec

# Ver logs de Laravel
docker-compose exec backend tail -f storage/logs/laravel.log
```

### Limpieza
```bash
# Eliminar imágenes no usadas
docker image prune

# Eliminar todo (contenedores, imágenes, volúmenes)
docker system prune -a

# Reconstruir un servicio específico
docker-compose build backend
```

---

## Conceptos clave aprendidos

### Docker Networks (driver: bridge)
- Red virtual privada para comunicación entre contenedores
- Permite usar nombres de servicio en lugar de IPs
- Ejemplo: `DB_HOST=mysql` en lugar de `DB_HOST=192.168.1.x`

### Docker Volumes (driver: local)
- Almacenamiento persistente
- Los datos sobreviven aunque se elimine el contenedor
- Ubicación: `C:\ProgramData\docker\volumes\` (Windows)

### Dockerfile vs docker-compose.yml
- **Dockerfile**: "Receta" para construir UNA imagen
- **docker-compose.yml**: Orquesta MÚLTIPLES contenedores

### Mapeo de puertos
- Formato: `"puerto_host:puerto_contenedor"`
- Ejemplo: `"8000:8000"` → accedes desde tu navegador a `localhost:8000`

---

## Notas importantes

1. **Puertos usados:**
   - 3307: MySQL (host) → 3306 (contenedor)
   - 8000: Laravel Backend
   - 3000: Nuxt Frontend

2. **Credenciales de MySQL:**
   - Root password: `root`
   - Database: `mapremec`
   - User: `mapremec_user`
   - Password: `mapremec_password`

3. **Archivos a NO versionar en Git:**
   - `.env` (usar `.env.docker` como referencia)
   - `vendor/`
   - `node_modules/`

---

## Troubleshooting

### Problema: "Port 3306 already in use"
**Solución:** Ya tienes MySQL corriendo localmente
```bash
# Detener MySQL local en Windows
net stop MySQL80

# O cambiar puerto en docker-compose.yml
ports:
  - "3307:3306"  # Usar 3307 en tu PC
```

### Problema: "Connection refused" desde Laravel a MySQL
**Solución:** MySQL no está listo aún
- El script `docker-entrypoint.sh` ya tiene un wait loop
- Verifica que `depends_on: - mysql` esté configurado

### Problema: Cambios en código no se reflejan
**Solución:** Verifica los volúmenes en docker-compose.yml
```yaml
volumes:
  - ./MAPREMEC_BACK:/var/www  # Debe estar presente
```

---

## Referencias

- [Documentación oficial de Docker](https://docs.docker.com/)
- [Docker Compose reference](https://docs.docker.com/compose/compose-file/)
- [Laravel en Docker](https://laravel.com/docs/10.x/sail)

---

## Docker Watch (Hot Reload)

Docker Watch permite sincronizar automáticamente los cambios de código local con el contenedor.

### Iniciar con Watch

```bash
docker-compose watch
```

### Tipos de acciones

| Action | Descripción | Uso |
|--------|-------------|-----|
| `sync` | Copia archivos al contenedor sin reiniciar | Código fuente (pages, components) |
| `restart` | Reinicia el contenedor | Archivos de configuración (nuxt.config.ts) |
| `rebuild` | Reconstruye la imagen completa | Dependencias (package.json) |

### Flujo de trabajo

1. Ejecutas `docker-compose watch`
2. Editas un archivo (ej: `pages/index.vue`)
3. Docker detecta el cambio y lo sincroniza
4. Nuxt HMR actualiza el navegador automáticamente

---

## URLs de acceso

| Servicio | URL |
|----------|-----|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:8000 |
| MySQL | localhost:3307 |

---

**Última actualización:** 18 de enero de 2026
**Estado:** Stack completo funcionando ✅
