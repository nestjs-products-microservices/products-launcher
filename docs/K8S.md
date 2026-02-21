# Kubernetes + Helm - comandos utiles

Guia rapida de comandos para operar el cluster y el chart de Helm de este proyecto.

## Helm
- Crear un chart: `helm create <nombre>`
- Instalar un chart local: `helm install <nombre> .`
- Actualizar un release: `helm upgrade <nombre> .`

## Operaciones basicas de Kubernetes
- Listar recursos: `kubectl get <pods | deployments | services>`
- Describir pods: `kubectl describe pods`
- Describir un pod: `kubectl describe pod <nombre>`
- Eliminar un pod: `kubectl delete pod <nombre>`
- Ver logs: `kubectl logs <nombre>`

## Crear manifests rapidamente

### Deployment
```bash
kubectl create deployment <nombre> --image=<registro/url/imagen> --dry-run=client -o yaml > deployment.yml
```

### Service
```bash
kubectl create service clusterip <nombre> --tcp=<8888> --dry-run=client -o yaml > service.yml
kubectl create service nodeport <nombre> --tcp=<3000> --dry-run=client -o yaml > service.yml
```
- **clusterip**: acceso solo dentro del cluster.
- **nodeport**: acceso desde fuera del cluster.

## Secrets

### Crear secrets
```bash
kubectl create secret generic <nombre> --from-literal=key=value
kubectl create secret generic secret1 --from-literal=key1=value1 --from-literal=key2=value2
```

### Inspeccionar secrets
- Listar: `kubectl get secrets`
- Ver contenido: `kubectl get secrets <nombre> -o yaml`

### Editar un secret
La forma mas segura suele ser recrearlo, pero si necesitas editarlo:
1. Ejecuta `kubectl edit secret <nombre>`.
2. Los valores estan en `base64`. Puedes usar un conversor en linea (por ejemplo [rapidtables](https://www.rapidtables.com/web/tools/base64-decode.html)).
3. En el editor, presiona **i** para editar, **esc** para salir del modo edicion y `:wq` para guardar.

## Secrets de Google Cloud para pull de imagenes

1. Crear el secret de registro:
```bash
kubectl create secret docker-registry gcr-json-key --docker-server=SERVIDOR-DE-GOOGLE-docker.pkg.dev --docker-username=_json_key --docker-password="$(cat 'PATH/DE/Tienda Microservices IAM.json')" --docker-email=TU_CORREO@gmail.com
```

2. Asociarlo al service account por defecto:
```bash
kubectl patch serviceaccounts default -p '{ "imagePullSecrets": [{ "name":"gcr-json-key" }] }'
```

## Exportar y aplicar configuraciones
- Exportar a archivo:
```bash
kubectl get secret <nombre> -o yaml > <nombre>.yml
```
- Aplicar desde archivo:
```bash
kubectl create -f <nombre>.yml
```

