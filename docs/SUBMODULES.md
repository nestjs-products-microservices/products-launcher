# Submódulos - Guía Operativa

Guía completa para trabajar con submódulos Git en este proyecto de microservicios.

## Estructura de Submódulos

```
products-launcher/                    # Repositorio principal
├── .gitmodules                       # Archivo de configuración
├── client-gateway/                   # Submódulo: API Gateway
├── auth-ms/                          # Submódulo: Autenticación
├── products-ms/                      # Submódulo: Productos
├── orders-ms/                        # Submódulo: Órdenes
└── payments-ms/                      # Submódulo: Pagos
```

## Configuración Actual (.gitmodules)

```ini
[submodule "client-gateway"]
    path = client-gateway
    url = https://github.com/nestjs-products-microservices/nestjs-api-gateway.git

[submodule "products-ms"]
    path = products-ms
    url = https://github.com/nestjs-products-microservices/nestjs-products-microservice.git

[submodule "orders-ms"]
    path = orders-ms
    url = https://github.com/nestjs-products-microservices/nestjs-orders-microservice.git

[submodule "payments-ms"]
    path = payments-ms
    url = https://github.com/nestjs-products-microservices/payments-microservice.git

[submodule "auth-ms"]
    path = auth-ms
    url = https://github.com/nestjs-products-microservices/auth-microservice.git
```

## Inicializar Submódulos

### Clonar Repositorio con Submódulos

```bash
# Clonar con submódulos incluidos
git clone https://github.com/nestjs-products-microservices/products-launcher.git
cd products-launcher

# Inicializar submódulos
git submodule update --init --recursive
```

### Ver Estado de Submódulos

```bash
# Ver estado de todos los submódulos
git submodule status

# Ver estado con colores
git submodule status --recursive
```

## Desarrollo Local

### Configuración Inicial

```bash
# 1. Clonar el repositorio principal
git clone https://github.com/nestjs-products-microservices/products-launcher.git
cd products-launcher

# 2. Inicializar submódulos
git submodule update --init --recursive

# 3. Crear archivo .env basado en template
cp .env.template .env

# 4. Editar .env con tus valores
vim .env

# 5. Levantar servicios con Docker Compose
docker compose up --build

# O en background
docker compose up -d --build
```

### Servicios en Desarrollo

| Servicio | Puerto | Descripción |
|----------|--------|-------------|
| Client Gateway | 3000 | API principal |
| NATS | 4222 | Mensajería |
| NATS Monitor | 8222 | Dashboard |
| Orders DB (PostgreSQL) | 5432 | Base de datos |

### Comandos Docker Compose

```bash
# Levantar todos los servicios
docker compose up

# Con rebuild
docker compose up --build

# En modo detach (background)
docker compose up -d

# Ver logs
docker compose logs -f

# Ver logs de un servicio específico
docker compose logs -f client-gateway

# Detener servicios
docker compose down

# Detener y eliminar volúmenes
docker compose down -v

# Rebuild de un servicio específico
docker compose build <servicio>
docker compose up -d <servicio>
```

## Producción Local

```bash
# 1. Construir imágenes
docker compose -f docker-compose.prod.yml build

# 2. Levantar servicios
docker compose -f docker-compose.prod.yml up -d

# 3. Verificar servicios
docker compose -f docker-compose.prod.yml ps
```

## Trabajar con un Submódulo

### Flujo de Trabajo Recomendado

1. **Cambios en submódulo**: Primero haz commit y push en el submódulo
2. **Actualizar referencia**: Luego actualiza el commit en el repositorio principal

```bash
# 1. Entrar al submódulo
cd auth-ms

# 2. Crear rama de trabajo
git checkout -b feature/nueva-funcionalidad

# 3. Hacer cambios, commit y push
git add .
git commit -m "feat: nueva funcionalidad"
git push -u origin feature/nueva-funcionalidad

# 4. Volver al repositorio principal
cd ..
git add auth-ms
git commit -m "Update auth-ms to latest"
git push
```

### Actualizar Submódulos

#### Después de un clon inicial

```bash
git submodule update --init --recursive
```

#### Obtener últimos cambios del remoto

```bash
# Para un submódulo específico
cd <submodulo>
git fetch origin
git pull origin main

# Para todos los submódulos
git submodule foreach 'git fetch origin && git pull origin main'
```

#### Actualizar referencia del submódulo

```bash
# Después de actualizar el submódulo, actualizar la referencia
git add <submodulo>
git commit -m "Update <submodulo> to latest"
git push
```

## Crear un Nuevo Submódulo

### Pasos para Agregar un Submódulo

```bash
# 1. Crear el repositorio en GitHub primero

# 2. Agregar el submódulo al repositorio principal
git submodule add <repository_url> <directory_name>

# Ejemplo:
git submodule add https://github.com/usuario/nuevo-servicio.git nuevo-servicio

# 3. Verificar
git submodule status

# 4. Confirmar cambios
git add .
git commit -m "Add nuevo-servicio submodule"
git push
```

### Inicializar Submódulo Vacío

```bash
# Crear directorio
mkdir nuevo-servicio

# Inicializar repositorio Git dentro
cd nuevo-servicio
git init

# Crear archivo inicial
echo "# Nuevo Servicio" > README.md
git add .
git commit -m "Initial commit"

# Agregar remoto
git remote add origin https://github.com/usuario/nuevo-servicio.git
git push -u origin main

# Volver al repositorio principal
cd ..
git submodule add https://github.com/usuario/nuevo-servicio.git nuevo-servicio
```

## Eliminar un Submódulo

```bash
# 1. Eliminar del .gitmodules
git config -f .gitmodules --remove-section submodule.<path>
git add .gitmodules

# 2. Eliminar del índice
git rm --cached <path>

# 3. Eliminar archivos
rm -rf <path>

# 4. Eliminar en .git/config
git config --remove-section submodule.<path>

# 5. Commit
git commit -m "Remove submodule <path>"
git push
```

## Buenas Prácticas

### Commits

1. **No mezclar cambios**: Evita hacer cambios en el submódulo y en el repositorio principal en el mismo commit
2. **Commits atómicos**: Haz commits pequeños y enfocados
3. **Mensajes claros**: Usa conventional commits

```bash
# ✅ Correcto: Commits separados
# 1. En auth-ms
git commit -m "fix: corrige bug en login"

# 2. En repositorio principal
git commit -m "chore: update auth-ms to latest fix"
```

### Ramas

1. **Trabaja en ramas**: No trabajes directamente en `main`
2. **Sincroniza frecuentemente**: Mantén tus submódulos actualizados

```bash
# Crear rama de trabajo en submódulo
cd products-ms
git checkout -b fix/bug-description

# Trabajar, commit, push
git push -u origin fix/bug-description

# Merge a main en el submódulo
git checkout main
git merge fix/bug-description
git push

# Actualizar referencia en repositorio principal
cd ..
git add products-ms
git commit -m "fix: update products-ms"
git push
```

### Sincronización

```bash
# Script para actualizar todos los submódulos
#!/bin/bash
echo "Updating submodules..."

for submodule in $(git submodule status | cut -d' ' -f2); do
    echo "Updating $submodule"
    cd $submodule
    git checkout main
    git pull origin main
    cd ..
done

echo "Committing updates..."
git add .
git commit -m "chore: update all submodules"
git push
```

## Troubleshooting

### Submódulo en estado "detached"

```bash
# Solución: cambiar a la rama correcta
cd <submodulo>
git checkout main
git pull origin main
cd ..
git add <submodulo>
git commit -m "fix: sync submodule to main"
```

### Submódulo no encontrado

```bash
# Reinicializar submódulos
git submodule deinit -f --all
git submodule update --init --recursive
```

### Conflictos en submódulo

```bash
# 1. Resolver conflicto en el submódulo
cd <submodulo>
git checkout --theirs <archivo-conflicto>
git add <archivo-conflicto>
git commit -m "fix: resolve conflict"

# 2. Actualizar referencia
cd ..
git add <submodulo>
git commit -m "merge: resolve conflict in <submodulo>"
```

### Error: "is not a valid commit"

```bash
# El remote del submódulo cambió
# Actualizar la URL del submódulo
git config submodule.<path>.url <nueva-url>

# O editar .gitmodules y actualizar
git submodule sync
```

## Referencia Rápida

| Acción | Comando |
|---------|---------|
| Clonar con submódulos | `git clone --recurse-submodules <url>` |
| Inicializar submódulos | `git submodule update --init --recursive` |
| Actualizar submódulos | `git submodule update --remote` |
| Ver estado | `git submodule status` |
| Entrar a submódulo | `cd <submódulo>` |
| Agregar submódulo | `git submodule add <url> <path>` |
| Eliminar submódulo | `git rm --cached <path>` |
| Sincronizar URLs | `git submodule sync` |

## Links Útiles

- [Git Submodules Documentation](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
- [Working with Submodules](https://www.atlassian.com/git/tutorials/git-submodules)
