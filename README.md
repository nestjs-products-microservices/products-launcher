# E-Commerce Microservices - Products Launcher

Aplicación de **e-commerce** construida con una arquitectura de **microservicios NestJS** que se comunican de forma asíncrona a través de un bus de mensajería **NATS**. Este repositorio es el **orquestador**: agrupa los 5 microservicios como submódulos de Git y centraliza el desarrollo local (Docker Compose), la construcción/publicación de imágenes en **Google Cloud Platform** (Cloud Build + Artifact Registry) y el despliegue en **Kubernetes (GKE)** con Helm.

La app permite registrar usuarios, gestionar un catálogo de productos, crear órdenes de compra y cobrarlas mediante **Stripe**.

## Índice

1. [Arquitectura general](#arquitectura-general)
2. [Stack tecnológico](#stack-tecnológico)
3. [Microservicios y lógica de negocio](#microservicios-y-lógica-de-negocio)
4. [Flujo de la aplicación](#flujo-de-la-aplicación)
5. [Estructura del repositorio (submódulos)](#estructura-del-repositorio-submódulos)
6. [Prerrequisitos](#prerrequisitos)
7. [Variables de entorno](#variables-de-entorno)
8. [Levantar en local](#levantar-en-local)
9. [Levantar en producción](#levantar-en-producción)
10. [Cómo probar / usar la API](#cómo-probar--usar-la-api)
11. [Documentación adicional](#documentación-adicional)
12. [Mejoras recomendadas](#mejoras-recomendadas)

## Arquitectura general

```
┌───────────────────────────────────────────────────────────────────────────┐
│                              USUARIO EXTERNO                                │
└──────────────────────────────────────┬────────────────────────────────────┘
                                        │ HTTPS
                                        ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         INGRESS (GKE LoadBalancer)                          │
│   ┌─────────────────────────┐        ┌─────────────────────────────────┐   │
│   │ client-gateway   Path /* │        │ payments-webhook   Path /*      │   │
│   └────────────┬────────────┘        └──────────────┬──────────────────┘   │
└───────────────┼─────────────────────────────────────┼──────────────────────┘
                │                                       │
                ▼                                       │
┌───────────────────────────────────────────────────┐  │
│              CLIENT GATEWAY (HTTP/REST)            │  │
│                NestJS API Gateway · /api           │  │
└──────┬──────────────┬──────────────┬───────────────┘  │
       │              │              │                   │
       ▼              ▼              ▼                   ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                      NATS MESSAGE BROKER (Port 4222)                        │
│                       (ClusterIP - solo interno)                            │
└──────┬──────────────┬──────────────┬──────────────────────┬────────────────┘
       │              │              │                       │
       ▼              ▼              ▼                       ▼
┌────────────┐ ┌────────────┐ ┌────────────┐         ┌────────────┐
│  AUTH-MS   │ │ PRODUCTS-MS│ │  ORDERS-MS │         │ PAYMENTS-MS│
│ JWT / User │ │ CRUD prod. │ │ Órdenes y  │◀───────▶│ Stripe +   │
│            │ │ + validar  │ │ estados    │  evento │ webhooks   │
└─────┬──────┘ └─────┬──────┘ └─────┬──────┘         └─────┬──────┘
      │              │              │                      │
      ▼              ▼              ▼                      ▼
┌───────────┐  ┌───────────┐  ┌───────────┐         ┌───────────┐
│  MongoDB  │  │  SQLite   │  │ PostgreSQL│         │  STRIPE   │
│  (Atlas)  │  │  (dev.db) │  │  (Neon)   │         │ (externo) │
└───────────┘  └───────────┘  └───────────┘         └───────────┘
```

Puntos clave de la arquitectura:

- **`client-gateway` es la única entrada HTTP**. Traduce las peticiones REST a mensajes NATS y reenvía la respuesta al cliente. Ningún microservicio de negocio se expone directamente a internet (salvo el webhook de pagos).
- **NATS es el bus interno** de comunicación entre servicios (request/response y eventos). En Kubernetes corre como `ClusterIP`, sin acceso externo.
- **Cada microservicio tiene su propia base de datos** (patrón *database-per-service*): MongoDB para auth, PostgreSQL para órdenes, SQLite para productos. `payments-ms` no persiste datos (es stateless).
- **`payments-ms` expone un webhook HTTP** (`/payments/webhook`) por un Ingress aparte, porque Stripe necesita notificar los cobros directamente por HTTP.

## Stack tecnológico

| Componente | Tecnología | Versión |
|------------|------------|---------|
| Framework | NestJS | 10.x / 11.x |
| Lenguaje | TypeScript | 5.x |
| Mensajería | NATS | Latest |
| ORM | Prisma | - |
| Base de Datos (Auth) | MongoDB Atlas | - |
| Base de Datos (Orders) | PostgreSQL (Neon) | - |
| Base de Datos (Products) | SQLite | - |
| Pagos | Stripe | - |
| Contenedores | Docker | Latest |
| Orquestación | Kubernetes (GKE) | 1.28+ |
| Helm | Helm 3 | Latest |
| CI/CD | Cloud Build | Latest |
| Registry | Artifact Registry | Latest |

## Microservicios y lógica de negocio

La aplicación se compone de 5 servicios. Salvo el gateway y el webhook de pagos, todos se comunican exclusivamente por NATS mediante *message patterns* (request/response) y *event patterns* (eventos).

### client-gateway

API Gateway HTTP/REST. Es el único punto de entrada para los clientes. Expone las rutas bajo el prefijo `/api` y reenvía cada petición al microservicio correspondiente vía NATS.

- **Base de datos:** ninguna (stateless).
- **Autenticación:** un `AuthGuard` extrae el token `Bearer` de la cabecera `Authorization` y lo valida contra `auth-ms` (patrón `auth.verify.user`) antes de permitir el acceso a rutas protegidas.
- **Responsabilidad:** enrutar, validar DTOs de entrada y propagar errores.

### auth-ms

Gestión de usuarios y autenticación.

- **Base de datos:** MongoDB (Prisma). Modelo `User { id, email (único), name, password (hash bcrypt) }`.
- **JWT:** los tokens se firman con `JWT_SECRET` y expiran en **2 horas**. Cada verificación re-emite un token con expiración renovada.
- **Patrones NATS:**

| Patrón | Acción |
|--------|--------|
| `auth.register.user` | Registra usuario (valida email único, hashea password) y devuelve `{ user, token }` |
| `auth.login.user` | Verifica credenciales y devuelve `{ user, token }` |
| `auth.verify.user` | Valida un JWT y devuelve el usuario + token renovado |

### products-ms

Catálogo de productos.

- **Base de datos:** SQLite (Prisma). Modelo `Product { id, name, price, available, createdAt, updatedAt }`.
- **Responsabilidad:** CRUD de productos (el borrado es *soft-delete*: marca `available = false`) y validación de existencia de productos cuando `orders-ms` crea una orden.
- **Patrones NATS:**

| Patrón | Acción |
|--------|--------|
| `create_product` | Crea producto |
| `find_all_products` | Lista paginada de productos disponibles |
| `find_one_product` | Obtiene un producto por ID |
| `update_product` | Actualiza producto |
| `delete_product` | Soft-delete (`available = false`) |
| `validate_products` | Valida que un conjunto de IDs exista (lo usa `orders-ms`) |

### orders-ms

Gestión de órdenes y su ciclo de estados.

- **Base de datos:** PostgreSQL (Prisma). Modelos `Order`, `OrderItem`, `OrderReceipt`. Enum `OrderStatus { PENDING, PAID, DELIVERED, CANCELLED }`.
- **Responsabilidad:** crear órdenes validando productos y calculando totales, consultar/listar órdenes por estado, cambiar el estado y reaccionar a la confirmación de pago.
- **Patrones NATS:**

| Patrón | Tipo | Acción |
|--------|------|--------|
| `createOrder` | request | Valida productos, calcula `totalAmount`/`totalItems`, persiste la orden y solicita la sesión de pago |
| `findAllOrders` | request | Lista paginada (con filtro opcional por estado) |
| `findOneOrder` | request | Obtiene una orden con el detalle de sus productos |
| `changeOrderStatus` | request | Cambia el estado de la orden |
| `payment.succeeded` | **event** | Escucha el evento de pago: marca la orden como `PAID`, guarda `stripeChargeId` y crea el `OrderReceipt` |

### payments-ms

Integración con Stripe. Servicio **híbrido HTTP + NATS** y **sin base de datos**.

- **Responsabilidad:** crear sesiones de Stripe Checkout y procesar los webhooks de Stripe.
- **Patrón NATS:** `create.payment.session` → recibe los ítems de la orden y devuelve la URL de Stripe Checkout.
- **Endpoints HTTP:** `POST /payments/webhook` (recibe eventos de Stripe; verifica la firma con `STRIPE_ENDPOINT_SECRET`), `GET /payments/success`, `GET /payments/cancel`.
- **Evento emitido:** al recibir `charge.succeeded` de Stripe, emite el evento NATS `payment.succeeded` que consume `orders-ms`.

### Resumen de persistencia

| Servicio | Base de datos | ORM |
|----------|---------------|-----|
| auth-ms | MongoDB (Atlas) | Prisma |
| products-ms | SQLite | Prisma |
| orders-ms | PostgreSQL (Neon) | Prisma |
| payments-ms | — (stateless) | — |
| client-gateway | — (stateless) | — |

## Flujo de la aplicación

### 1. Autenticación

```
Cliente ── POST /api/auth/register|login ──▶ client-gateway
                                                  │  NATS: auth.register.user / auth.login.user
                                                  ▼
                                               auth-ms ── MongoDB (valida, bcrypt, firma JWT)
                                                  │
                          { user, token } ◀───────┘
```

En rutas protegidas, el `AuthGuard` del gateway intercepta la petición y llama a `auth.verify.user`; si el token es válido, inyecta el usuario en el contexto de la petición.

### 2. Creación de una orden

```
Cliente ── POST /api/orders ──▶ client-gateway
                                     │ NATS: createOrder
                                     ▼
                                  orders-ms
                                     │ 1) NATS: validate_products ─▶ products-ms (verifica IDs y precios)
                                     │ 2) calcula totalAmount y totalItems
                                     │ 3) persiste Order + OrderItems (PostgreSQL)
                                     │ 4) NATS: create.payment.session ─▶ payments-ms (crea Stripe Checkout)
                                     ▼
        { order, paymentSession } ◀──┘   ← incluye la URL de pago de Stripe
```

### 3. Confirmación de pago

```
Stripe ── POST /payments/webhook ──▶ payments-ms
                                         │ verifica firma del webhook
                                         │ en evento "charge.succeeded":
                                         │ NATS event: payment.succeeded
                                         ▼
                                      orders-ms
                                         │ marca la orden como PAID
                                         │ guarda stripeChargeId
                                         └ crea OrderReceipt (URL del recibo)
```

## Estructura del repositorio (submódulos)

Los microservicios viven en repositorios independientes incluidos como **submódulos de Git**.

| Submódulo | Repositorio | Descripción | Puerto (interno) |
|-----------|-------------|-------------|------------------|
| `client-gateway` | [nestjs-api-gateway](https://github.com/nestjs-products-microservices/nestjs-api-gateway.git) | API Gateway HTTP | 3000 |
| `auth-ms` | [auth-microservice](https://github.com/nestjs-products-microservices/auth-microservice.git) | Autenticación JWT | 3004 |
| `products-ms` | [nestjs-products-microservice](https://github.com/nestjs-products-microservices/nestjs-products-microservice.git) | Gestión de productos | 3001 |
| `orders-ms` | [nestjs-orders-microservice](https://github.com/nestjs-products-microservices/nestjs-orders-microservice.git) | Gestión de órdenes | 3002 |
| `payments-ms` | [payments-microservice](https://github.com/nestjs-products-microservices/payments-microservice.git) | Integración con Stripe | 3003 |

```bash
# Clonar incluyendo submódulos
git clone --recurse-submodules https://github.com/nestjs-products-microservices/products-launcher.git

# O, si ya clonaste sin submódulos:
git submodule update --init --recursive
```

> Para el flujo de trabajo con submódulos (actualizar, hacer commit por submódulo, etc.) consulta [docs/SUBMODULES.md](docs/SUBMODULES.md).

## Prerrequisitos

### Para desarrollo local

- **Docker** y Docker Compose
- **Node.js 20+** (solo si vas a ejecutar un servicio fuera de Docker)
- **Git** (con soporte de submódulos)
- Una cuenta de **Stripe** en modo test (claves `sk_test_...` y `whsec_...`)
- Cadenas de conexión para **MongoDB Atlas** (auth) y **PostgreSQL** (orders) — o usa el PostgreSQL local que levanta Docker Compose

### Para producción (GCP / GKE)

- `gcloud` CLI instalado y autenticado
- `kubectl` configurado contra el clúster de GKE
- `helm` 3.x
- APIs de GCP habilitadas: Cloud Build, Artifact Registry, Secret Manager, Kubernetes Engine
- Artifact Registry creado en `southamerica-east1-docker.pkg.dev`
- Cuenta de servicio de Cloud Build con los roles IAM mínimos:
  - `roles/cloudbuild.builds.editor`
  - `roles/secretmanager.secretAccessor`
  - `roles/artifactregistry.writer`
  - `roles/container.developer`

## Variables de entorno

Copia la plantilla y completa los valores:

```bash
cp .env.template .env
```

```bash
# Puertos
CLIENT_GATEWAY_PORT=3000
PAYMENTS_MS_PORT=3003

# Stripe (payments-ms)
STRIPE_SECRET=sk_test_...
STRIPE_SUCCESS_URL=http://localhost:3003/payments/success
STRIPE_CANCEL_URL=http://localhost:3003/payments/cancel
STRIPE_ENDPOINT_SECRET=whsec_...

# Auth (auth-ms)
AUTH_DATABASE_URL=mongodb+srv://...
JWT_SECRET=your-jwt-secret

# Orders (orders-ms)
ORDERS_DATABASE_URL=postgresql://...
```

| Variable | Servicio | Descripción |
|----------|----------|-------------|
| `CLIENT_GATEWAY_PORT` | client-gateway | Puerto HTTP del gateway publicado en local |
| `PAYMENTS_MS_PORT` | payments-ms | Puerto HTTP para el webhook de Stripe |
| `STRIPE_SECRET` | payments-ms | Clave secreta de la API de Stripe |
| `STRIPE_SUCCESS_URL` | payments-ms | URL de redirección tras pago exitoso |
| `STRIPE_CANCEL_URL` | payments-ms | URL de redirección si se cancela el pago |
| `STRIPE_ENDPOINT_SECRET` | payments-ms | Secreto de firma del webhook de Stripe |
| `AUTH_DATABASE_URL` | auth-ms | Cadena de conexión a MongoDB |
| `JWT_SECRET` | auth-ms | Secreto para firmar/verificar los JWT |
| `ORDERS_DATABASE_URL` | orders-ms | Cadena de conexión a PostgreSQL (usado en build de producción) |

## Levantar en local

```bash
# 1. Clonar con submódulos (ver sección de submódulos)
git clone --recurse-submodules <repo>
cd products-launcher

# 2. Configurar variables de entorno
cp .env.template .env   # edita .env con tus valores

# 3. Construir y levantar todos los servicios
docker compose up --build
```

Docker Compose levanta NATS, los 5 microservicios y una base de datos PostgreSQL para órdenes. `products-ms` usa SQLite embebido (sin contenedor) y `orders-ms` depende del contenedor `orders-db`.

### Servicios disponibles en local

| Servicio | Endpoint local |
|----------|----------------|
| Client Gateway (API) | http://localhost:3000/api |
| Payments (webhook/success/cancel) | http://localhost:3003/payments |
| NATS Monitor | http://localhost:8222 |
| Orders DB (PostgreSQL) | localhost:5432 (`postgres` / `123456`, db `ordersdb`) |

### Recibir webhooks de Stripe en local

Stripe necesita una URL pública para notificar los cobros. Usa la **Stripe CLI** para reenviar los eventos a tu `payments-ms` local:

```bash
stripe listen --forward-to localhost:3003/payments/webhook
```

La CLI imprime un `whsec_...` que debes poner en `STRIPE_ENDPOINT_SECRET`.

## Levantar en producción

El despliegue tiene dos fases: **construir/publicar imágenes** con Cloud Build y **desplegar** en GKE con Helm.

### CI/CD — Cloud Build

Las imágenes se publican en Artifact Registry siguiendo la convención:

```
southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/<service-name>
```

- **Trigger:** un push a la rama `cloud-build` dispara los triggers por submódulo (`<submodulo>-trigger`).
- Cada submódulo tiene su propio `cloudbuild.yml` y `Dockerfile.prod`.

Pipeline base (`cloudbuild.yml`):

```yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/auth-ms'
      - '-f'
      - 'Dockerfile.prod'
      - '--platform=linux/amd64'
      - '.'

  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/auth-ms'

options:
  logging: CLOUD_LOGGING_ONLY
```

Pipeline con secretos (ej. `orders-ms`, que necesita la URL de la DB en build-time):

```yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    entrypoint: 'bash'
    args:
      - -c
      - |
        docker build -t southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/orders-ms -f Dockerfile.prod --platform=linux/amd64 --build-arg ORDERS_DATABASE_URL=$$DATABASE_URL .
    secretEnv: ['DATABASE_URL']

  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/orders-ms'

availableSecrets:
  secretManager:
    - versionName: projects/202140589714/secrets/orders_database_url/versions/1
      env: 'DATABASE_URL'

options:
  logging: CLOUD_LOGGING_ONLY
```

Construcción manual desde el directorio de un submódulo:

```bash
gcloud builds submit --config cloudbuild.yml .
```

### Kubernetes + Helm

Chart en `k8s/ecommerce/`:

```
k8s/ecommerce/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── auth-ms/        (deployment.yml)
    ├── products-ms/    (deployment.yml)
    ├── orders-ms/      (deployment.yml)
    ├── payments-ms/    (deployment.yml, service.yml)
    ├── client-gateway/ (deployment.yml, service.yml)
    ├── nats/           (deployment.yml, service.yml)
    └── ingress/        (client-gateway.ingress.yml, payments-webhook.ingress.yml)
```

Secretos requeridos en el clúster:

| Secret | Claves | Servicio |
|--------|--------|----------|
| `auth-secret` | `jwt_secret`, `database_url` | auth-ms |
| `orders-secret` | `database_url` | orders-ms |
| `payments-secret` | `stripe_secret`, `stripe_endpoint_secret` | payments-ms |

Despliegue:

```bash
cd k8s/ecommerce

helm install ecommerce .     # instalar
helm upgrade ecommerce .     # actualizar
helm status ecommerce        # ver estado
```

Obtener los endpoints públicos (Load Balancer):

```bash
kubectl get ingress
# NAME                       HOSTS   ADDRESS         PORTS
# client-gateway-ingress     *       XXX.XXX.XXX.XXX 80
# payments-webhook-ingress   *       XXX.XXX.XXX.XXX 80
```

> Usa el `ADDRESS` del `client-gateway-ingress` para consumir la API y el del `payments-webhook-ingress` para configurar el webhook de Stripe.

Comandos útiles:

```bash
kubectl get pods
kubectl get services
kubectl logs -l app=client-gateway
kubectl get secrets
kubectl delete pod <pod-name>   # forzar reinicio
```

> Más detalle operativo en [docs/K8S.md](docs/K8S.md).

## Cómo probar / usar la API

Los ejemplos asumen el gateway en local (`http://localhost:3000/api`). En producción reemplaza el host por el `ADDRESS` del Ingress del gateway.

### 1. Registrar un usuario

`password` debe ser **fuerte** (mayúsculas, minúsculas, número y símbolo).

```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Leo",
    "email": "leo@example.com",
    "password": "Password123!"
  }'
```

Respuesta (resumida): `{ "user": { "id": "...", "email": "...", "name": "..." }, "token": "eyJ..." }`. Guarda el `token`.

### 2. Iniciar sesión

```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "leo@example.com", "password": "Password123!" }'
```

### 3. Verificar el token (ruta protegida)

```bash
curl http://localhost:3000/api/auth/verify \
  -H "Authorization: Bearer <TOKEN>"
```

### 4. Crear un producto

```bash
curl -X POST http://localhost:3000/api/products \
  -H "Content-Type: application/json" \
  -d '{ "name": "Teclado mecánico", "price": 49.99 }'
```

### 5. Listar / obtener productos

```bash
curl http://localhost:3000/api/products
curl http://localhost:3000/api/products/1
```

### 6. Crear una orden

Cada ítem requiere `productId`, `quantity` y `price`:

```bash
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "productId": 1, "quantity": 2, "price": 49.99 }
    ]
  }'
```

La respuesta incluye la orden y `paymentSession` con la **URL de Stripe Checkout**. Abre esa URL en el navegador.

### 7. Pagar (Stripe en modo test)

En la página de Stripe Checkout usa una tarjeta de prueba:

- Número: `4242 4242 4242 4242`
- Fecha: cualquier fecha futura · CVC: cualquier 3 dígitos

Tras el pago, Stripe llama al webhook → `payments-ms` emite `payment.succeeded` → `orders-ms` marca la orden como `PAID` (recuerda tener corriendo `stripe listen` en local).

### 8. Consultar la orden y cambiar su estado

```bash
# Obtener orden por ID (verifica que pasó a PAID)
curl http://localhost:3000/api/orders/id/<ORDER_ID>

# Listar por estado (PENDING | PAID | DELIVERED | CANCELLED)
curl http://localhost:3000/api/orders/PAID

# Cambiar estado (estados válidos vía gateway: PENDING, DELIVERED, CANCELLED)
curl -X PATCH http://localhost:3000/api/orders/<ORDER_ID> \
  -H "Content-Type: application/json" \
  -d '{ "status": "DELIVERED" }'
```

### Referencia de endpoints

**Client Gateway** (`/api`)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/auth/register` | Registro de usuarios |
| POST | `/api/auth/login` | Inicio de sesión |
| GET | `/api/auth/verify` | Verificar token (protegido) |
| GET | `/api/products` | Listar productos |
| GET | `/api/products/:id` | Obtener producto |
| POST | `/api/products` | Crear producto |
| PATCH | `/api/products/:id` | Actualizar producto |
| DELETE | `/api/products/:id` | Eliminar producto (soft-delete) |
| POST | `/api/orders` | Crear orden |
| GET | `/api/orders` | Listar órdenes |
| GET | `/api/orders/id/:id` | Obtener orden por ID |
| GET | `/api/orders/:status` | Listar órdenes por estado |
| PATCH | `/api/orders/:id` | Cambiar estado de orden |

**Payments MS** (puerto 3003)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/payments/webhook` | Webhook de Stripe |
| GET | `/payments/success` | Página de éxito |
| GET | `/payments/cancel` | Página de cancelación |

## Documentación adicional

- [Arquitectura completa](docs/ARCHITECTURE.md) — diagrama detallado y flujos
- [Kubernetes + Helm](docs/K8S.md) — comandos y guías de operación
- [Submódulos](docs/SUBMODULES.md) — guía operativa de submódulos

## Mejoras recomendadas

### Alta prioridad

- [ ] Implementar SSL/TLS con Google-managed certificates
- [ ] Configurar Cloud Armor para WAF
- [ ] Implementar OAuth 2.0
- [ ] Configurar HPA (Horizontal Pod Autoscaler)
- [ ] Implementar Rolling Updates

### Media prioridad

- [ ] Configurar NetworkPolicies
- [ ] Implementar observabilidad (Cloud Logging, Prometheus, Grafana)
- [ ] Configurar dead-letter queues en NATS
- [ ] Implementar Canary Deployments

### Baja prioridad

- [ ] Migrar SQLite a PostgreSQL para products-ms
- [ ] Configurar GitOps con ArgoCD
- [ ] Implementar service mesh (Istio)

## Licencia

UNLICENSED
