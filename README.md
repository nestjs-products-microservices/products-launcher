# 🚀 CI/CD - Products Launcher

Repositorio orquestador de microservicios que centraliza la construcción y publicación de imágenes Docker en GCP usando **Cloud Build**, **Artifact Registry**, **Secret Manager** e **IAM**.

## 🧩 Servicios (submódulos)
- `auth-ms`
- `orders-ms`
- `payments-ms`
- `products-ms`
- `client-gateway`

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
