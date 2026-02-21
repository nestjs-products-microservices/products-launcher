# Submódulos - guia operativa

## Dev
1. Clonar el repositorio.
2. Crear un `.env` basado en `.env.template`.
3. Inicializar y reconstruir submodulos:
   ```bash
   git submodule update --init --recursive
   ```
4. Levantar los servicios:
   ```bash
   docker compose up --build
   ```

## Crear un submodulo
1. Crear un nuevo repositorio en GitHub.
2. Clonar el repo principal en local.
3. Agregar el submodulo (la carpeta de destino no debe existir):
   ```bash
   git submodule add <repository_url> <directory_name>
   ```
4. Confirmar los cambios en el repo principal:
   ```bash
   git add .
   git commit -m "Add submodule"
   git push
   ```

## Actualizar submodulos
- Inicializar y actualizar (primer clon):
  ```bash
  git submodule update --init --recursive
  ```
- Actualizar referencias a la ultima version:
  ```bash
  git submodule update --remote
  ```

## Buenas practicas
- Si trabajas en un submodulo, **primero** actualiza y hace push en el submodulo y **despues** en el repo principal.
- Evita mezclar cambios del submodulo y del repo principal en el mismo commit.

## Prod
1. Clonar el repositorio.
2. Crear un `.env` basado en `.env.template`.
3. Ejecutar:
   ```bash
   docker compose -f docker-compose.prod.yml build
   ```
