# Arquitectura del Proyecto E-Commerce Microservices

## 1. Resumen del Proyecto

Este proyecto es una aplicación de e-commerce basada en una arquitectura de microservicios desplegada en **Google Kubernetes Engine (GKE)**. La aplicación permite a los usuarios autenticarse, explorar productos, crear órdenes y procesar pagos a través de Stripe.

### Flujo de la Aplicación

```
Usuario → Client Gateway → Auth MS
                         → Products MS
                         → Orders MS → Payments MS (Stripe)
```

1. El usuario se autentica a través del **Client Gateway**
2. El **Auth MS** valida credenciales y genera tokens JWT
3. El usuario puede listar y buscar productos a través del **Products MS**
4. El usuario crea una orden que es procesada por el **Orders MS**
5. El **Orders MS** crea una sesión de pago en el **Payments MS**
6. El usuario es redirigido a Stripe para completar el pago
7. Stripe envía un webhook al **Payments MS** que notifica al **Orders MS**

---

## 2. Infraestructura de Kubernetes

### Clúster GKE

| Componente | Detalle |
|------------|---------|
| Plataforma | Google Kubernetes Engine (GKE) |
| Región | southamerica-east1 |
| Registro de Imágenes | Artifact Registry (`southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/`) |

### Networking

| Recurso | Tipo | Descripción |
|---------|------|-------------|
| `client-gateway` | Service (NodePort: 3000) | Punto de entrada principal |
| `payments-webhook` | Service (NodePort: 3000) | Webhooks de Stripe |
| `nats` | Service (ClusterIP: 4222) | Mensajería interna |
| `client-gateway-ingress` | Ingress | Enrutamiento HTTP |
| `payments-webhook-ingress` | Ingress | Webhooks externos |

### Secrets

Los secretos se gestionan mediante **Kubernetes Secrets** y **Google Secret Manager**:

| Secret | Contenido | Microservicio |
|--------|-----------|---------------|
| `auth-secret` | `jwt_secret`, `database_url` | auth-ms |
| `orders-secret` | `database_url` | orders-ms |
| `payments-secret` | `stripe_secret`, `stripe_endpoint_secret` | payments-ms |

### Notas de Mejora

> ⚠️ **PUNTOS DE MEJORA IDENTIFICADOS:**
> - No hay SSL/TLS configurado en los Ingress
> - No hay WAF/Cloud Armor para protección
> - No hay NetworkPolicy para segmentación
> - Los servicios usan NodePort en lugar de ClusterIP para exposición externa

---

## 3. Registro de Imágenes y CI/CD

### Artifact Registry

```
southamerica-east1-docker.pkg.dev/ecommerce-microservices-488103/image-registry/
├── auth-ms
├── orders-ms
├── products-ms
├── payments-ms
└── client-gateway
```

### Pipeline de CI/CD (Cloud Build)

El pipeline actual se activa con push a la rama `main`:

1. **Build**: Construye la imagen Docker con el Dockerfile.prod
2. **Push**: Envía la imagen a Artifact Registry
3. **Secrets**: Utiliza Secret Manager para variables sensibles

### Estrategia de Despliegue

| Microservicio | Estrategia Actual | Estado |
|---------------|-------------------|--------|
| Todos | Desconocido (strategy: {}) | ⚠️ No especificado |

### Notas de Mejora

> ⚠️ **PUNTOS DE MEJORA IDENTIFICADOS:**
> - No se especifica estrategia de despliegue (Rolling/Canary/Blue-Green)
> - No hay configuración de Readiness/Liveness Probes
> - No hay HorizontalPodAutoscaler (HPA)
> - Falta automatización de despliegues (GitOps/ArgoCD)

---

## 4. Microservicios

### Tabla de Microservicios

| Microservicio | Puerto Interno | Exposición Externa | Comunicación | Base de Datos |
|---------------|----------------|--------------------|--------------|---------------|
| **client-gateway** | 3000 | ✅ Ingress (/) | HTTP (REST) | N/A |
| **auth-ms** | 3000 | ❌ Interno | NATS (Request/Response) | MongoDB Atlas |
| **products-ms** | 3000 | ❌ Interno | NATS (Request/Response) | SQLite (dev.db) |
| **orders-ms** | 3000 | ❌ Interno | NATS (Request/Response + Pub/Sub) | PostgreSQL (Neon) |
| **payments-ms** | 3000 | ✅ Ingress (/payments/*) | NATS + HTTP (Webhook) | N/A |
| **nats** | 4222 | ❌ Interno (ClusterIP) | TCP | N/A |

### Endpoints Expuestos

#### Client Gateway (Puerto 3000)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/auth/register` | Registro de usuarios |
| POST | `/api/auth/login` | Inicio de sesión |
| GET | `/api/auth/verify` | Verificar token (requiere AuthGuard) |
| GET | `/api/products` | Listar productos |
| GET | `/api/products/:id` | Obtener producto |
| POST | `/api/products` | Crear producto |
| PATCH | `/api/products/:id` | Actualizar producto |
| DELETE | `/api/products/:id` | Eliminar producto |
| POST | `/api/orders` | Crear orden |
| GET | `/api/orders` | Listar órdenes |
| GET | `/api/orders/id/:id` | Obtener orden por ID |
| GET | `/api/orders/:status` | Listar órdenes por estado |
| PATCH | `/api/orders/:id` | Cambiar estado de orden |

#### Payments MS (Puerto 3003)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/payments/webhook` | Webhook de Stripe |
| GET | `/payments/success` | Página de éxito |
| GET | `/payments/cancel` | Página de cancelación |

### Temas NATS (Comunicación)

| Tema | Tipo | Descripción |
|------|------|-------------|
| `auth.register.user` | Request/Response | Registro de usuarios |
| `auth.login.user` | Request/Response | Inicio de sesión |
| `auth.verify.user` | Request/Response | Verificación de token |
| `create_product` | Request/Response | Crear producto |
| `find_all_products` | Request/Response | Listar productos |
| `find_one_product` | Request/Response | Obtener producto |
| `update_product` | Request/Response | Actualizar producto |
| `delete_product` | Request/Response | Eliminar producto |
| `validate_products` | Request/Response | Validar productos |
| `createOrder` | Request/Response | Crear orden |
| `findAllOrders` | Request/Response | Listar órdenes |
| `findOneOrder` | Request/Response | Obtener orden |
| `changeOrderStatus` | Request/Response | Cambiar estado |
| `create.payment.session` | Request/Response | Crear sesión de pago |
| `payment.succeeded` | Event (Pub/Sub) | Notificación de pago exitoso |

---

## 5. Mensajería (NATS)

### Configuración

| Parámetro | Valor |
|-----------|-------|
| Servidor | `nats://nats:4222` |
| Protocolo | NATS (TCP) |
| Tipo de comunicación | Request/Response + Pub/Sub |

### Resiliencia Actual

> ⚠️ **PUNTOS DE MEJORA IDENTIFICADOS:**
> - No hay retry logic configurado
> - No hay dead-letter queues
> - No hay persistencia de mensajes
> - Un solo-replica de NATS (sin cluster)

---

## 6. Bases de Datos

| Base de Datos | Tipo | Proveedor | Replicación | Backups |
|---------------|------|-----------|-------------|---------|
| **Auth DB** | MongoDB | MongoDB Atlas | ⚠️ Desconocido | ⚠️ Desconocido |
| **Products DB** | SQLite | Local (dev.db) | ❌ No | ❌ No |
| **Orders DB** | PostgreSQL | Neon | ✅ Alta disponibilidad | ⚠️ Configuración de Neon |

### Notas de Mejora

> ⚠️ **PUNTOS DE MEJORA IDENTIFICADOS:**
> - Products usa SQLite (no recomendado para producción)
> - Falta estrategia de backups documentada
> - Sin migraciones automatizadas en K8s

---

## 7. Flujo de Arquitectura

### Diagrama ASCII

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USUARIO EXTERNO                                 │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      │ HTTPS (sin SSL actualmente)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INGRESS (GKE)                                        │
│  ┌─────────────────────────┐    ┌─────────────────────────────────────┐  │
│  │ client-gateway-ingress   │    │ payments-webhook-ingress            │  │
│  │ Path: /*                │    │ Path: /*                             │  │
│  └────────────┬────────────┘    └──────────────┬──────────────────────┘  │
└───────────────┼──────────────────────────────────┼─────────────────────────┘
                │                                  │
                ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLIENT GATEWAY (3000)                                │
│                    (API Gateway - HTTP/REST)                                 │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ /auth/*     │  │ /products/* │  │ /orders/*    │  │ /payments/* │     │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘     │
└─────────┼────────────────┼────────────────┼────────────────┼─────────────┘
          │                │                │                │
          │                │                │                │
          ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NATS MESSAGE BROKER (4222)                          │
│                    (ClusterIP - Solo interno)                                │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                                                                      │    │
│  │  Request/Response:  Pub/Sub:                                       │    │
│  │  ─────────────────   ─────────                                       │    │
│  │  • auth.*           • payment.succeeded                             │    │
│  │  • create_product                                                    │    │
│  │  • find_*_products                                                   │    │
│  │  • createOrder                                                       │    │
│  │  • findAllOrders                                                     │    │
│  │  • create.payment.session                                           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────┬────────────────┬────────────────┬────────────────┬─────────────┘
          │                │                │                │
          ▼                ▼                ▼                ▼
┌────────────────┐ ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│   AUTH-MS     │ │  PRODUCTS-MS  │ │   ORDERS-MS    │ │  PAYMENTS-MS   │
│   (3000)      │ │    (3000)      │ │    (3000)      │ │     (3000)     │
│               │ │                │ │                │ │                │
│ JWT Auth      │ │ CRUD Products  │ │ Order Mgmt     │ │ Stripe Integ   │
│ User Mgmt     │ │                 │ │ Status Mgmt    │ │ Webhooks       │
└───────┬────────┘ └────────┬────────┘ └───────┬────────┘ └───────┬────────┘
        │                   │                   │                 │
        │                   │                   │                 │
        ▼                   │                   ▼                 │
┌────────────────┐          │           ┌────────────────┐          │
│ MongoDB Atlas  │          │           │   Neon DB      │          │
│ (Auth Data)   │          │           │ (PostgreSQL)   │          │
└────────────────┘          │           └────────────────┘          │
                           │                                            │
                           ▼                                            │
                    ┌────────────────┐                                  │
                    │ SQLite (dev.db)│                                  │
                    │ (Products)     │                                  │
                    └────────────────┘                                  │
                                                                      │
      ┌───────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          STRIPE (EXTERNO)                                   │
│                    https://dashboard.stripe.com                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Mejoras Sugeridas

### 🔴 Alta Prioridad

#### 8.1 SSL/TLS

| Aspecto | Actual | Sugerido |
|---------|--------|----------|
| Protocolo | HTTP (sin cifrado) | HTTPS con TLS 1.3 |
| Certificados | Ninguno | Google-managed SSL Certificates |
| Redirect | N/A | HTTP → HTTPS automatico |

**Implementación:**
```yaml
# Ejemplo de Ingress con SSL
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    networking.gke.io/pre-shared-certs: "ssl-certificate"
    kubernetes.io/ingress.allow-http: "false"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: ssl-certificate
```

#### 8.2 WAF / Cloud Armor

| Aspecto | Actual | Sugerido |
|---------|--------|----------|
| Protección | Ninguna | Google Cloud Armor |
| Rate Limiting | No | Sí |
| SQL Injection | No | Sí |
| XSS Protection | No | Sí |

**Implementación:**
```yaml
apiVersion: security.google.com/v1
kind: BackendConfig
metadata:
  name: client-gateway-backend
spec:
  securityPolicy:
    name: "ecommerce-waf-policy"
  connectionDraining:
    drainingTimeoutSec: 60
```

#### 8.3 OAuth / Autenticación

| Aspecto | Actual | Sugerido |
|---------|--------|----------|
| Auth | JWT manual | OAuth 2.0 / OIDC |
| Proveedor | N/A | Google Identity Platform |
| Tokens | JWT básico | Refresh tokens |

### 🟡 Media Prioridad

#### 8.4 NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector:
    matchLabels:
      app: auth-ms
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: client-gateway
```

#### 8.5 Autoscaling (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: client-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: client-gateway
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

#### 8.6 Rolling Update / Canary Deployment

```yaml
# Rolling Update
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

# Canary con Ingress
metadata:
  annotations:
    kubernetes.io/canary: "true"
    canary-weight: "20"
```

#### 8.7 Observabilidad

| Componente | Herramienta | Propósito |
|------------|-------------|-----------|
| Logs | Cloud Logging | Centralización de logs |
| Métricas | Prometheus + Grafana | Monitorización |
| Tracing | OpenTelemetry / Jaeger | Distributed tracing |

**Implementación recomendada:**
- Cloud Logging para logs
- Prometheus Operator + Grafana
- OpenTelemetry para tracing

### 🟢 Baja Prioridad

#### 8.8 Resiliencia de Mensajería

| Mejora | Descripción |
|--------|-------------|
| NATS Cluster | 3 réplicas de NATS |
| Retry Logic | Exponential backoff |
| Dead Letter Queue | Persistencia de mensajes fallidos |
| Message Persistence |耐久性 |

#### 8.9 Health Checks

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 20
```

---

## 9. Beneficios de la Arquitectura Mejorada

### Seguridad

| Beneficio | Impacto |
|-----------|---------|
| SSL/TLS | Cifrado de datos en tránsito |
| Cloud Armor | Protección contra DDoS y ataques |
| OAuth 2.0 | Autenticación segura centralizada |
| NetworkPolicy | Segmentación de red |

### Disponibilidad

| Beneficio | Impacto |
|-----------|---------|
| HPA | Escalamiento automático |
| Rolling Updates | Zero-downtime deployments |
| Canary Deployments | Despliegues de bajo riesgo |
| Multi-replica NATS | Alta disponibilidad de mensajería |

### Observabilidad

| Beneficio | Impacto |
|-----------|---------|
| Logs centralizados | Debugging rápido |
| Métricas | Monitorización en tiempo real |
| Distributed Tracing | Troubleshooting de errores |

### Mantenibilidad

| Beneficio | Impacto |
|-----------|---------|
| GitOps | Despliegues reproducibles |
| IAC (Terraform) | Infraestructura como código |
| Documentación | Conocimiento compartido |

---

## 10. Recursos Adicionales

- [Documentación GKE](https://cloud.google.com/kubernetes-engine/docs)
- [NATS Documentation](https://docs.nats.io/)
- [NestJS Microservices](https://docs.nestjs.com/microservices/basics)
- [Google Cloud Armor](https://cloud.google.com/armor/docs)
- [Artifact Registry](https://cloud.google.com/artifact-registry/docs)

---

*Documento generado automáticamente. Última actualización: 2026-02-22*
