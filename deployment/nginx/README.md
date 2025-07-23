# 🐳 Desplegar NGINX en Minikube con Buenas Prácticas

Esta guía muestra cómo desplegar NGINX en Minikube usando un `Deployment` y un `Service`, aplicando configuraciones recomendadas como `resources.requests`, `resources.limits`, `livenessProbe` y `readinessProbe`.

---

## 📦 Paso 1: Manifiesto de `Deployment`

Guarda este archivo como `nginx-deployment.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
```

## 🌐 Paso 2: Manifiesto del Service

Guarda este archivo como nginx-service.yaml.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

## 🚀 Paso 3: Aplicar los manifiestos

Ejecuta los siguientes comandos:

```yaml
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

## 🌍 Paso 4: Acceder a NGINX desde el navegador

### Opción A: Usar minikube service

```yaml
minikube service nginx-service
```

Este comando abrirá automáticamente tu navegador con la URL del servicio.

### Opción B: Acceder manualmente

Si usaste nodePort: 30080, puedes acceder directamente en:

```
http://localhost:30080
```

## 🚮 Eliminar los recursos:

### Eliminar el Service:

Para eliminar el Service:

```bash
kubectl delete -f nginx-service.yaml
```

### Eliminar el Deployment:

Para eliminar el Deployment que creaste:

```bash
kubectl delete -f nginx-deployment.yaml
```

O si prefieres eliminar ambos recursos de una vez, puedes hacerlo con un solo comando si tienes ambos archivos:

### 🧹 Limpiar Minikube (opcional):

```bash
kubectl delete -f .
```

Esto eliminará todos los archivos de manifiestos dentro del directorio donde te encuentres.

### 🧹 Limpiar Minikube (opcional):

Si deseas asegurarte de que no quedan residuos en Minikube después de eliminar los recursos de Kubernetes, puedes detener Minikube con:

```bash
minikube stop
```

Y si quieres eliminar todo el clúster de Minikube (esto eliminará todos los recursos creados en Minikube), puedes usar:

```bash
minikube delete
```

⚠️ Recuerda:
Al eliminar el servicio y el deployment, cualquier cambio o dato persistente que hayas generado también se eliminará a menos que hayas configurado un PersistentVolume para ello.
