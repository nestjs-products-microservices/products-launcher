# E-Commerce Microservices - Products Launcher

Repositorio orquestador de microservicios para una aplicación de e-commerce completa, centralizando la construcción y publicación de imágenes Docker en **Google Cloud Platform (GCP)** utilizando **Cloud Build**, **Artifact Registry**, **Secret Manager** e **IAM**.

## Arquitectura General

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USUARIO EXTERNO                                 │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │ HTTPS
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INGRESS (GKE LoadBalancer)                          │
│  ┌─────────────────────────┐    ┌─────────────────────────────────────┐  │
│  │ client-gateway          │    │ payments-webhook                    │  │
│  │ Path: /*                │    │ Path: /*                            │  │
│  └────────────┬────────────┘    └──────────────┬──────────────────────┘  │
└───────────────┼──────────────────────────────────┼─────────────────────────┘
                │                                  │
                ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLIENT GATEWAY (Port 3000)                          │
│                    NestJS API Gateway - HTTP/REST                           │
└─────────┬────────────────┬────────────────┬────────────────┬─────────────┘
          │                │                │                │
          ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NATS MESSAGE BROKER (Port 4222)                    │
│                    (ClusterIP - Solo interno)                                │
└─────────┬────────────────┬────────────────┬────────────────┬─────────────┘
          │                │                │                │
          ▼                ▼                ▼                ▼
┌────────────────┐ ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│   AUTH-MS     │ │  PRODUCTS-MS  │ │   ORDERS-MS    │ │  PAYMENTS-MS   │
│   (Port 3000) │ │  (Port 3000)  │ │  (Port 3000)   │ │  (Port 3000)   │
│               │ │                │ │                │ │                │
│ JWT Auth      │ │ CRUD Products │ │ Order Mgmt     │ │ Stripe Integ   │
│ User Mgmt     │ │                │ │ Status Mgmt    │ │ Webhooks       │
└───────┬────────┘ └────────┬────────┘ └───────┬────────┘ └───────┬────────┘
        │                   │                   │                 │
        ▼                   │                   ▼                 │
┌────────────────┐          │          ┐          │
│ ┌──────────────── MongoDB Atlas  │          │           │   Neon DB      │          │
│ (Auth Data)    │          │           │ (PostgreSQL)   │          │
└────────────────┘          │           └────────────────┘          │
                           ▼                                            │
                    ┌────────────────┐                                  │
                    │ SQLite (dev.db)│                                  │
                    │ (Products)     │                                  │
                    └────────────────┘                                  │
                                                                      ▼
                                                         ┌────────────────────┐
                                                         │    STRIPE          │
                                                         │ (External Payment)  │
                                                         └────────────────────┘
```

## Submódulos (Microservicios)

| Submódulo | Repositorio | Descripción | Puerto |
|-----------|-------------|-------------|--------|
| `client-gateway` | [nestjs-api-gateway](https://github.com/nestjs-products-microservices/nestjs-api-gateway.git) | API Gateway principal | 3000 |
| `auth-ms` | [auth-microservice](https://github.com/nestjs-products-microservices/auth-microservice.git) | Autenticación JWT | 3000 |
| `products-ms` | [nestjs-products-microservice](https://github.com/nestjs-products-microservices/nestjs-products-microservice.git) | Gestión de productos | 3000 |
| `orders-ms` | [nestjs-orders-microservice](https://github.com/nestjs-products-microservices/nestjs-orders-microservice.git) | Gestión de órdenes | 3000 |
| `payments-ms` | [payments-microservice](https://github.com/nestjs-products-microservices/payments-microservice.git) | Integración con Stripe | 3000 |

## Tecnologías

| Componente | Tecnología | Versión |
|------------|------------|---------|
| Framework | NestJS | 10.x / 11.x |
| Lenguaje | TypeScript | 5.x |
| Mensajería | NATS | Latest |
| Base de Datos (Auth) | MongoDB Atlas | - |
| Base de Datos (Orders) | PostgreSQL (Neon) | - |
| Base de Datos (Products) | SQLite | - |
| Contenedores | Docker | Latest |
| Orquestación | Kubernetes (GKE) | 1.28+ |
| Helm | Helm 3 | Latest |
| CI/CD | Cloud Build | Latest |
| Registry | Artifact Registry | Latest |

## Documentación

- [Arquitectura Completa](docs/ARCHITECTURE.md) - Diagrama detallado y flujos
- [Kubernetes + Helm](docs/K8S.md) - Comandos y guías de operación
- [Submódulos](docs/SUBMODULES.md) - Guía operativa de submódulos

## Prerrequisitos

### Herramientas Requeridas

- `gcloud` CLI instalado y autenticado
- `kubectl` configurado
- `helm` 3.x instalado
- Docker instalado (para desarrollo local)
- Node.js 20+ (para desarrollo local)

### Configuración GCP

1. Proyecto GCP configurado
2. APIs habilitadas:
   - Cloud Build API
   - Artifact Registry API
   - Secret Manager API
   - Kubernetes Engine API
3. Artifact Registry creado: `southamerica-east1-docker.pkg.dev`
4. Secret Manager con los secretos necesarios

### Cuenta de Servicio CI/CD

La cuenta de servicio utilizada en Cloud Build debe tener los siguientes roles:

```
202140589714-compute@developer.gserviceaccount.com
```

**Roles IAM mínimos requeridos:**
- `roles/cloudbuild.builds.editor`
- `roles/secretmanager.secretAccessor`
- `roles/artifactregistry.writer`
- `roles/container.developer` (para GKE)

## Variables de Entorno

### .env.template

```bash
# Puertos
CLIENT_GATEWAY_PORT=3000
PAYMENTS_MS_PORT=3003

# Stripe
STRIPE_SECRET=sk_test_...
STRIPE_SUCCESS_URL=http://localhost:3003/payments/success
STRIPE_CANCEL_URL=http://localhost:3003/payments/cancel
STRIPE_ENDPOINT_SECRET=whsec_...

# Auth
AUTH_DATABASE_URL=mongodb+srv://...
JWT_SECRET=your-jwt-secret

# Orders
ORDERS_DATABASE_URL=postgresql://...
```

## Desarrollo Local

### Inicializar Submódulos

```bash
# Clonar con submódulos
git clone https://github.com/nestjs-products-microservices/products-launcher.git
cd products-launcher

# Inicializar submódulos
git submodule update --init --recursive

# Crear archivo .env
cp .env.template .env
# Editar .env con tus valores
```

### Levantar Servicios con Docker Compose

```bash
# Desarrollo
docker compose up --build

# Producción local
docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml up
```

### Servicios Disponibles en Desarrollo

| Servicio | Endpoint |
|----------|----------|
| Client Gateway | http://localhost:3000/api |
| NATS Monitor | http://localhost:8222 |
| Orders DB | localhost:5432 |

## CI/CD - Cloud Build

### Convenciones

**Registro de Imágenes:**
```
southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/<service-name>
```

**Triggers:**
- Push a rama `cloud-build` ejecuta los triggers por submódulo
- Naming: `<submodulo>-trigger`

### Pipeline Base (cloudbuild.yml)

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

### Pipeline con Secretos

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

### Construcción Manual

```bash
# Desde el directorio del submódulo
gcloud builds submit --config cloudbuild.yml .
```

## Kubernetes + Helm

### Estructura del Chart

```
k8s/ecommerce/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── auth-ms/
    │   ├── deployment.yml
    │   └── service.yml
    ├── orders-ms/
    │   ├── deployment.yml
    │   └── service.yml
    ├── products-ms/
    │   ├── deployment.yml
    │   └── service.yml
    ├── payments-ms/
    │   ├── deployment.yml
    │   └── service.yml
    ├── client-gateway/
    │   ├── deployment.yml
    │   └── service.yml
    ├── nats/
    │   ├── deployment.yml
    │   └── service.yml
    └── ingress/
        ├── client-gateway.ingress.yml
        └── payments-webhook.ingress.yml
```

### Secretos Requeridos

| Secret | Claves | Servicio |
|--------|--------|----------|
| `auth-secret` | `jwt_secret`, `database_url` | auth-ms |
| `orders-secret` | `database_url` | orders-ms |
| `payments-secret` | `stripe_secret`, `stripe_endpoint_secret` | payments-ms |

### Despliegue en GKE

```bash
# Ir al directorio de Helm
cd k8s/ecommerce

# Instalar
helm install ecommerce .

# Actualizar
helm upgrade ecommerce .

# Ver status
helm status ecommerce
```

### Ver Endpoints Públicos

```bash
kubectl get ingress

# Output:
# NAME                       HOSTS   ADDRESS         PORTS
# client-gateway-ingress     *       XXX.XXX.XXX.XXX 80
# payments-webhook-ingress   *       XXX.XXX.XXX.XXX 80
```

> Usa el `ADDRESS` asignado por el Load Balancer para consumir el API o configurar el webhook de Stripe.

### Comandos Útiles

```bash
# Ver pods
kubectl get pods

# Ver servicios
kubectl get services

# Ver ingress
kubectl get ingress

# Ver logs
kubectl logs -l app=client-gateway

# Ver secretos
kubectl get secrets

# Eliminar pod (forzar restart)
kubectl delete pod <pod-name>
```

## API Endpoints

### Client Gateway (Puerto 3000)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| **Auth** | | |
| POST | `/api/auth/register` | Registro de usuarios |
| POST | `/api/auth/login` | Inicio de sesión |
| GET | `/api/auth/verify` | Verificar token |
| **Products** | | |
| GET | `/api/products` | Listar productos |
| GET | `/api/products/:id` | Obtener producto |
| POST | `/api/products` | Crear producto |
| PATCH | `/api/products/:id` | Actualizar producto |
| DELETE | `/api/products/:id` | Eliminar producto |
| **Orders** | | |
| POST | `/api/orders` | Crear orden |
| GET | `/api/orders` | Listar órdenes |
| GET | `/api/orders/id/:id` | Obtener orden por ID |
| GET | `/api/orders/:status` | Listar órdenes por estado |
| PATCH | `/api/orders/:id` | Cambiar estado de orden |

### Payments MS (Puerto 3003)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/payments/webhook` | Webhook de Stripe |
| GET | `/payments/success` | Página de éxito |
| GET | `/payments/cancel` | Página de cancelación |

## Mejoras Recomendadas

### Alta Prioridad

- [ ] Implementar SSL/TLS con Google-managed certificates
- [ ] Configurar Cloud Armor para WAF
- [ ] Implementar OAuth 2.0
- [ ] Configurar HPA (Horizontal Pod Autoscaler)
- [ ] Implementar Rolling Updates

### Media Prioridad

- [ ] Configurar NetworkPolicies
- [ ] Implementar observabilidad (Cloud Logging, Prometheus, Grafana)
- [ ] Configurar dead-letter queues en NATS
- [ ] Implementar Canary Deployments

### Baja Prioridad

- [ ] Migrar SQLite a PostgreSQL para products-ms
- [ ] Configurar GitOps con ArgoCD
- [ ] Implementar service mesh (Istio)

## Licencia

UNLICENSED
