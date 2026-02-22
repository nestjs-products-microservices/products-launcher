# 🚀 CI/CD - Products Launcher

Repositorio orquestador de microservicios que centraliza la construcción y publicación de imágenes Docker en GCP usando **Cloud Build**, **Artifact Registry**, **Secret Manager** e **IAM**.

## 🧩 Servicios (submódulos)
- `auth-ms`
- `orders-ms`
- `payments-ms`
- `products-ms`
- `client-gateway`

## 📚 Docs
- [Submódulos](docs/SUBMODULES.md)
- [Kubernetes + Helm](docs/K8S.md)

## ✅ Prerrequisitos
- `gcloud` instalado y autenticado.
- Proyecto GCP configurado.
- Artifact Registry y Secret Manager habilitados.
- Cuenta de servicio usada en CI/CD:
  - `202140589714-compute@developer.gserviceaccount.com`
- Roles IAM mínimos requeridos:
  - `roles/cloudbuild.builds.editor`
  - `roles/secretmanager.secretAccessor`
  - `roles/artifactregistry.writer`

## 🏷️ Convención de imágenes
Artifact Registry:
```
southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/<service-name>
```

## 🏗️ Cloud Build por submódulo
Cada submódulo contiene un `cloudbuild.yml` que construye y publica la imagen correspondiente.

### Ejemplo base (auth-ms)
```yaml
steps:
  - name: "gcr.io/cloud-builders/docker"
    args:
      [
        "build",
        "-t",
        "southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/auth-ms",
        "-f",
        "Dockerfile.prod",
        "--platform=linux/amd64",
        ".",
      ]
  - name: "gcr.io/cloud-builders/docker"
    args:
      [
        "push",
        "southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/auth-ms",
      ]
options:
  logging: CLOUD_LOGGING_ONLY
```

### Ejemplo con secretos (orders-ms)
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
    [
      'push',
      'southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/orders-ms',
    ]

availableSecrets:
  secretManager:
  - versionName: projects/202140589714/secrets/orders_database_url/versions/1
    env: 'DATABASE_URL'

options:
  logging: CLOUD_LOGGING_ONLY
```

## 🔐 Secretos y variables
Algunos servicios requieren secretos durante el build.

### orders-ms
- Secret Manager: `orders_database_url` (versión `1`)
- Se inyecta como `DATABASE_URL`
- `Dockerfile.prod` consume `ORDERS_DATABASE_URL` como build-arg

## 🔄 Integración continua (Cloud Build Triggers)
Al hacer push a la rama `cloud-build`, se ejecutan los triggers por submódulo.
Convención de nombres:
```
<submodulo>-trigger
```

## 🛠️ Builds manuales
Desde el directorio de cada submódulo:
```bash
gcloud builds submit --config cloudbuild.yml .
```

## ☸️ Kubernetes + Helm
Configuración centralizada en `k8s/ecommerce` usando Helm. Incluye deployments y services para los submódulos y NATS.

### 🚀 Despliegue en GKE (nuevo)
El chart ya incluye los Ingress para exponer los servicios en GKE:
- `k8s/ecommerce/templates/ingress/client-gateway.ingress.yml` → `client-gateway` (API principal).
- `k8s/ecommerce/templates/ingress/payments-webhook.ingress.yml` → `payments-webhook` (webhooks de Stripe).

Para ver los endpoints públicos creados por GKE:
```bash
kubectl get ingress
```
> Usa el `ADDRESS` asignado por el Load Balancer para consumir el API o configurar el webhook.

### Estructura del chart
- `k8s/ecommerce/Chart.yaml`: definición del chart.
- `k8s/ecommerce/values.yaml`: valores (vacío por ahora, se usa el YAML directo).
- `k8s/ecommerce/templates/`: manifests por servicio.

### Deployments por submódulo
Cada submódulo tiene un `Deployment` con su imagen de Artifact Registry y variables requeridas:
- `auth-ms`: `JWT_SECRET`, `DATABASE_URL`, `NATS_SERVERS`.
- `orders-ms`: `DATABASE_URL`, `NATS_SERVERS`.
- `products-ms`: `DATABASE_URL` (sqlite local), `NATS_SERVERS`.
- `payments-ms`: `STRIPE_SECRET`, `STRIPE_ENDPOINT_SECRET`, `STRIPE_SUCCESS_URL`, `STRIPE_CANCEL_URL`, `NATS_SERVERS`.
- `client-gateway`: `NATS_SERVERS`.
- `nats`: broker de mensajería.

### Services
- `client-gateway`: `NodePort` en 3000 para exponer el API.
- `payments-webhook`: `NodePort` en 3000 (apunta a `payments-ms`).
- `nats`: `ClusterIP` en 4222.

### Secrets requeridos
Los deployments que consumen secretos esperan secrets de Kubernetes con estas llaves:
- `auth-secret`: `jwt_secret`, `database_url`.
- `orders-secret`: `database_url`.
- `payments-secret`: `stripe_secret`, `stripe_endpoint_secret`.

### Comandos Helm básicos
Desde `k8s/ecommerce`:
```bash
helm install ecommerce .
helm upgrade ecommerce .
```

> Más comandos y tips en `docs/K8S.md`.
