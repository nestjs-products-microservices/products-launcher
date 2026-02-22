# Kubernetes + Helm - Guía Operativa

Guía completa de comandos para operar el clúster Kubernetes y el chart de Helm de este proyecto.

## Helm

### Comandos Básicos

```bash
# Crear un nuevo chart
helm create <nombre>

# Instalar un chart local
helm install <nombre> .

# Actualizar un release existente
helm upgrade <nombre> .

# Desinstalar
helm uninstall <nombre>

# Ver releases instalados
helm list

# Ver historial de revisiones
helm history <nombre>

# Rollback a revisión anterior
helm rollback <nombre> <revision>
```

### Valores y Templates

```bash
# Renderizar templates sin aplicar
helm template <nombre> .

# Instalar con valores específicos
helm install <nombre> . --set image.tag=v1.0.0

# Instalar con archivo de valores
helm install <nombre> . -f values-prod.yaml

# Obtener valores actuales
helm get values <nombre>
```

## Operaciones Básicas de Kubernetes

### Recursos

```bash
# Listar todos los recursos
kubectl get all

# Listar pods
kubectl get pods

# Listar deployments
kubectl get deployments

# Listar servicios
kubectl get services

# Listar ingress
kubectl get ingress

# Listar secrets
kubectl get secrets

# Listar configmaps
kubectl get configmaps
```

### Descripciones y Detalles

```bash
# Describir todos los pods
kubectl describe pods

# Describir un pod específico
kubectl describe pod <nombre-pod>

# Describir un servicio
kubectl describe service <nombre-servicio>

# Ver detalles de un deployment
kubectl describe deployment <nombre-deployment>

# Ver información del clúster
kubectl cluster-info
```

### Logs y Debugging

```bash
# Ver logs de un pod
kubectl logs <nombre-pod>

# Ver logs en tiempo real
kubectl logs -f <nombre-pod>

# Ver logs de container específico (multi-container)
kubectl logs <nombre-pod> -c <nombre-container>

# Logs de pods con label
kubectl logs -l app=client-gateway

# Ver eventos del namespace
kubectl get events

# Ver eventos sorted por tiempo
kubectl get events --sort-by='.lastTimestamp'
```

### Ejecución en Pods

```bash
# Abrir shell en un pod
kubectl exec -it <nombre-pod> -- /bin/sh

# Ejecutar comando en pod
kubectl exec <nombre-pod> -- <comando>

# Port-forward a un pod
kubectl port-forward <nombre-pod> 8080:3000
```

### Manipulación de Recursos

```bash
# Eliminar un pod (se recreará si hay deployment <nombre-pod)
kubectl delete pod>

# Eliminar todos los pods
kubectl delete pods --all

# Eliminar recursos por label
kubectl delete pods -l app=client-gateway

# Eliminar recursos huérfanos
kubectl delete pods --grace-period=0 --force

# Aplicar configuración
kubectl apply -f <archivo.yml>

# Eliminar configuración
kubectl delete -f <archivo.yml>
```

## Crear Manifests Rápidamente

### Deployment

```bash
# Crear deployment básico
kubectl create deployment <nombre> --image=<imagen> --dry-run=client -o yaml > deployment.yml

# Con replicas
kubectl create deployment <nombre> --image=<imagen> --replicas=3 --dry-run=client -o yaml > deployment.yml

# Con variables de entorno
kubectl create deployment <nombre> --image=<imagen> --env="PORT=3000" --dry-run=client -o yaml > deployment.yml
```

### Service

```bash
# ClusterIP (solo interno)
kubectl create service clusterip <nombre> --tcp=3000:3000 --dry-run=client -o yaml > service.yml

# NodePort (externo)
kubectl create service nodeport <nombre> --tcp=3000:3000 --dry-run=client -o yaml > service.yml

# LoadBalancer
kubectl create service loadbalancer <nombre> --tcp=3000:3000 --dry-run=client -o yaml > service.yml
```

### Ingress

```bash
kubectl create ingress <nombre> --rule="/*=servicio:3000" --dry-run=client -o yaml > ingress.yml
```

### Secret

```bash
# Secret genérico
kubectl create secret generic <nombre> --from-literal=key=value
kubectl create secret generic <nombre> --from-literal=key1=value1 --from-literal=key2=value2

# Secret desde archivo
kubectl create secret generic <nombre> --from-file=cert.pem

# Secret desde archivo .env
kubectl create secret generic <nombre> --from-env-file=.env
```

## Secrets

### Operaciones con Secrets

```bash
# Listar secrets
kubectl get secrets

# Ver contenido de un secret
kubectl get secret <nombre> -o yaml

# Ver valor decodeado
kubectl get secret <nombre> -o jsonpath='{.data.key}' | base64 -d

# Crear secret desde literal
kubectl create secret generic auth-secret \
  --from-literal=jwt_secret=mi-secreto \
  --from-literal=database_url=mongodb://...

# Editar secret (decodea, edita, encodea)
kubectl edit secret <nombre>

# Eliminar secret
kubectl delete secret <nombre>
```

### Secret de Docker para GCR

```bash
# 1. Crear secret de registro
kubectl create secret docker-registry gcr-json-key \
  --docker-server=southamerica-east1-docker.pkg.dev \
  --docker-username=_json_key \
  --docker-password="$(cat 'path/to/service-account.json')" \
  --docker-email=tu-email@email.com

# 2. Asociar al service account por defecto
kubectl patch serviceaccounts default \
  -p '{ "imagePullSecrets": [{ "name":"gcr-json-key" }] }'

# 3. O asociar a un service account específico
kubectl patch serviceaccounts <nombre-sa> \
  -p '{ "imagePullSecrets": [{ "name":"gcr-json-key" }] }'
```

### Editar Secret Manualmente

```bash
# 1. Obtener secret
kubectl get secret <nombre> -o yaml > secret.yml

# 2. Los valores están en base64. Decodificar:
echo "c2VjcmV0LWtleQ==" | base64 -d

# 3. Editar el archivo
vim secret.yml

# 4. Aplicar
kubectl apply -f secret.yml
```

## Ingress y Load Balancing

### Ver Ingress

```bash
kubectl get ingress

# Describe ingress específico
kubectl describe ingress <nombre-ingress>

# Ver IP asignada (ADDRESS)
kubectl get ingress -o wide
```

### Configurar SSL/TLS

```yaml
# Ejemplo de Ingress con SSL
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    kubernetes.io/ingress.allow-http: "false"
    networking.gke.io/pre-shared-certs: "ssl-certificate"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: ssl-certificate
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: client-gateway
                port:
                  number: 3000
```

## Health Checks

### Readiness y Liveness

```yaml
# Agregar a deployment
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 20

readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
```

### Verificar Health

```bash
# Ver estado de probes
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Ver detalles de probe
kubectl describe pod <nombre> | grep -A 10 "Liveness\|Readiness"
```

## Escalado (HPA)

### Crear HPA

```bash
# Crear HPA automáticamente desde deployment
kubectl autoscale deployment <nombre> --cpu-percent=70 --min=2 --max=10

# Ver HPA
kubectl get hpa

# Ver detalles
kubectl describe hpa <nombre>
```

### YAML de HPA

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
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## Recursos Avanzados

### NetworkPolicy

```yaml
# Denegar todo ingreso por defecto
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### Resource Limits

```yaml
# Agregar a container en deployment
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### ConfigMap

```bash
# Crear configmap desde archivo
kubectl create configmap app-config --from-file=config.json

# Crear configmap desde literal
kubectl create configmap app-config \
  --from-literal=ENV=production \
  --from-literal=LOG_LEVEL=info
```

## Troubleshooting

### Pods no inician

```bash
# Ver eventos
kubectl get events --sort-by='.lastTimestamp' | grep <nombre-pod>

# Ver logs
kubectl logs <nombre-pod> --previous

# Describir pod
kubectl describe pod <nombre-pod>
```

### Service no accesible

```bash
# Ver endpoints
kubectl get endpoints <nombre-servicio>

# Ver pods asociados al servicio
kubectl get pods -l app=<nombre-app>

# Probar conectividad entre pods
kubectl exec -it <pod-origen> -- curl <servicio>:3000
```

### Problemas de Red

```bash
# Ver reglas de red
kubectl get networkpolicies

# Veriptables del nodo (requiere acceso SSH)
kubectl debug node/<nombre-nodo> --image=busybox -- chroot /host iptables -L -n -v
```

## Exportar y Aplicar Configuraciones

```bash
# Exportar recurso a YAML
kubectl get secret <nombre> -o yaml > secret.yml

# Aplicar desde archivo
kubectl apply -f manifest.yml

# Aplicar todo un directorio
kubectl apply -f ./directorio/

# Ver diferencias antes de aplicar
kubectl diff -f manifest.yml
```

## Comandos de Emergencia

```bash
# Forzar eliminación de pod
kubectl delete pod <nombre> --grace-period=0 --force

# Restart todos los pods de un deployment
kubectl rollout restart deployment/<nombre>

# Rollback de deployment
kubectl rollout undo deployment/<nombre>

# Ver estado de rollout
kubectl rollout status deployment/<nombre>
```

## Referencia Rápida

| Acción | Comando |
|--------|---------|
| Ver pods | `kubectl get pods` |
| Ver logs | `kubectl logs -f <pod>` |
| Ver eventos | `kubectl get events` |
| Aplicar config | `kubectl apply -f <archivo>` |
| Eliminar recurso | `kubectl delete -f <archivo>` |
| Port forward | `kubectl port-forward <pod> 8080:3000` |
| Exec en pod | `kubectl exec -it <pod> -- /bin/sh` |
| Ver secretos | `kubectl get secrets` |
| Ver ingress | `kubectl get ingress` |
| Escalar deployment | `kubectl scale deployment <nombre> --replicas=3` |
| Restart deployment | `kubectl rollout restart deployment/<nombre>` |

## Links Útiles

- [Documentación Oficial de Kubernetes](https://kubernetes.io/docs/)
- [Documentación de GKE](https://cloud.google.com/kubernetes-engine/docs)
- [Helm Docs](https://helm.sh/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
