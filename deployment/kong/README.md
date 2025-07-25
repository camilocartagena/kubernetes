# 🐙 Deploy Avanzado de Kong en Minikube (Modo Hybrid + Kong Konnect)

### 📌 Requisitos previos

- [Minikube](https://minikube.sigs.k8s.io/docs/)
- [Helm](https://helm.sh/)
- Cuenta en [Kong Konnect](https://cloud.konghq.com/)
- Token de Data Plane desde Kong Konnect
- Docker y kubectl instalados
- Namespace dedicado para Kong: `kong`

---

### 1. 🔧 Iniciar Minikube con suficientes recursos

```bash
minikube start --cpus=2 --memory=2200 --disk-size=10g
```

> 🛠️ Lo recomendado es minikube start --cpus=4 --memory=8192 --disk-size=30g pero depende de la capacidad de maquina. Docker Desktop has only 3914MB memory but you specified 8192MB.

## 🧩 Desglose del comando

| Opción          | Explicación                                                                                                                                         |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| --cpus=4        | Asigna 4 CPUs virtuales al entorno de Minikube. Esto asegura que haya suficiente capacidad de procesamiento para Kong y sus componentes auxiliares. |
| --memory=8192   | Asigna 8192 MB (8 GB de RAM) a Minikube. Esto es necesario para ejecutar Kong en modo data_plane, más sus sidecars y posibles servicios conectados. |
| --disk-size=30g | Asigna 30 GB de espacio en disco virtual. Garantiza almacenamiento suficiente para logs, certificados, bundles y otros recursos que usa Kong.       |

## ✅ ¿Por qué estos valores son importantes?

| Recurso | Motivo                                                                                                                                                  |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CPU     | Kong puede manejar múltiples peticiones, plugins, TLS y sincronización con Konnect. Mínimo 2 cores, pero 4 garantiza fluidez al hacer pruebas y cargas. |
| Memoria | Kong + NGINX + LuaJIT + proxy + TLS + Prometheus metrics requieren buena memoria. 4GB es el mínimo recomendado, 8GB es lo ideal para pruebas avanzadas. |
| Disco   | Permite manejar certificados, logs, imágenes del contenedor, bundles de Konnect y configuraciones persistentes sin error de espacio insuficiente.       |

## 💡 Analogía simple para sustentarlo

Piensa en Minikube como un simulador de Kubernetes en tu PC, y Kong como una aplicación de alto rendimiento que necesita una buena computadora para correr. Si le das pocos recursos (como 1 CPU y 2GB de RAM), se va a “congelar” o tirar errores. Al darle estos valores, evitas cuellos de botella y simulas mejor un entorno productivo.

## 📌 ¿Se puede cambiar esto después?

Sí. Puedes eliminar el clúster y reiniciarlo con nuevos valores:

```bash
minikube delete
minikube start --cpus=6 --memory=12288 --disk-size=50g
```

### 📌 Requisitos Mínimos Recomendados para una Prueba Básica

Ejecutar el siguiente comando:

```bash
minikube start --cpus=2 --memory=4096 --disk-size=10g
```

| Recurso           | Valor mínimo sugerido | Por qué                                                                                                                  |
| ----------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `--cpus=2`        | 2 núcleos             | Kong consume CPU al manejar TLS, proxy y sincronización con Konnect. 1 CPU puede generar errores por timeout o lentitud. |
| `--memory=4096`   | 4 GB de RAM           | Kong + NGINX + Lua + plugins usan bastante memoria. 2 GB puede ser muy justo y generar OOMKills (Out of Memory Kills).   |
| `--disk-size=10g` | 10 GB de disco        | Suficiente para imágenes, certificados, logs y carga básica sin errores de espacio.                                      |

## 2. 📦 Crear namespace y agregar el repo de Kong

```bash
kubectl create namespace kong


helm repo add kong https://charts.konghq.com
helm repo update
```

## 3. 🔐 Obtener el token del Data Plane en Kong Konnect

1. Ir a Konnect > Runtime Manager > Add Runtime
2. Seleccionar: new Gateway > Kubernetes > Selft-Manager Hybrid Mode
3. Copiar el DP JWT Token

## 4. 📁 Crear archivo kong-values.yaml con configuración avanzada

```yaml
# kong-values.yaml

image:
  repository: kong/kong-gateway
  tag: "3.11"

secretVolumes:
  - your-certificated

admin:
  enabled: false

env:
  role: data_plane
  database: "off"
  cluster_mtls: pki
  cluster_control_plane: your-control-plane-id.konghq.com::443
  cluster_dp_labels: "type:docker-kubernetesOS"
  cluster_server_name: your-control-plane-id.konghq.com:
  cluster_telemetry_endpoint: telemetry.konghq.com:443:443
  cluster_telemetry_server_name: telemetry.konghq.com
  cluster_cert: /etc/secrets/your-certificated/tls.crt
  cluster_cert_key: /etc/secrets/your-certificated/tls.key
  lua_ssl_trusted_certificate: system
  konnect_mode: "on"
  vitals: "off"
  nginx_worker_processes: "1"
  upstream_keepalive_max_requests: "100000"
  nginx_http_keepalive_requests: "100000"
  proxy_access_log: "off"
  dns_stale_ttl: "3600"
  router_flavor: expressions

ingressController:
  enabled: false
  installCRDs: false

resources:
  requests:
    cpu: 1
    memory: "2Gi"

manager:
  enabled: false

```

## 5. 🔐 Crear el secreto kong-cluster-cert en Kubernetes

```bash
    kubectl create secret tls kong-cluster-cert -n kong --cert=tls.crt --key=tls.key
```

Sí, en Kubernetes existen varios tipos de secretos, y los dos más comunes son:

-generic
-tls

A continuación te explico qué son, en qué se diferencian, y cuándo usar cada uno.

### 🔑 1. generic Secret

Es un tipo genérico que tú defines manualmente con pares clave-valor (por ejemplo, contraseñas, certificados, tokens, etc.).

➕ Se usa para:
API keys

Archivos .crt, .key, .pem

Passwords, config files, etc.

🛠 Ejemplo:

```bash
kubectl create secret generic mi-secreto --from-file=tls.crt  --from-file=tls.key -n kong
```

Esto crea un secreto con:

```yaml
data:
  cluster.crt: <base64-encoded>
  cluster.key: <base64-encoded>
```

✅ Flexible y se adapta a cualquier necesidad.

### 🔒 2. tls Secret

Es un tipo especializado para certificados TLS/SSL. Kubernetes espera dos claves exactas:

tls.crt → el certificado

tls.key → la clave privada

➕ Se usa para:
Ingress TLS

Kong TLS communication (cuando se necesita un secreto de tipo tls)

Servicios que requieren tls.key y tls.crt explícitamente

🛠 Ejemplo:

```bash
kubectl create secret tls kong-cluster-cert --cert=tls.crt --key=tls.key -n kong
```

Esto crea automáticamente:

```yaml
type: kubernetes.io/tls
data:
tls.crt: <base64-encoded>
tls.key: <base64-encoded>
```

✅ Es requerido por controladores como NGINX Ingress o Kong si el type debe ser tls.

> 📝 Los archivos cluster.crt y cluster.key vienen en el bundle que descargas desde Konnect al agregar un nuevo runtime.

Si estás trabajando en Kubernetes, aquí te muestro cómo ver, detallar y eliminar secretos actuales en un namespace específico (o todos).

### 🔍 1. Ver todos los secretos en un namespace

```bash
kubectl get secrets -n <namespace>
```

Ejemplo:

```bash
kubectl get secrets -n default
```

Para todos los namespaces:

```bash
kubectl get secrets --all-namespaces
```

### 📄 2. Ver el contenido de un secreto (codificado en base64)

```bash
kubectl get secret <nombre-del-secreto> -n <namespace> -o yaml
```

Si quieres ver los valores decodificados (por ejemplo, un tls.crt o password), puedes usar este comando:

```bash
kubectl get secret <nombre> -n <namespace> -o jsonpath="{.data.<clave>}" | base64 -d
```

Ejemplo para ver el certificado:

```bash
kubectl get secret kong-cluster-cert -n kong -o jsonpath="{.data.tls.crt}" | base64 -d
```

### 🧹 3. Eliminar un secreto

```bash
kubectl delete secret <nombre-del-secreto> -n <namespace>
```

Ejemplo:

```bash
kubectl delete secret kong-cluster-cert -n kong
```

## 6. 🚀 Instalar Kong con Helm

```bash
   helm install kong kong/kong -n kong -f kong-values.yaml
```

## 7. 🚀 Luego haz port-forward:

```bash
   kubectl port-forward svc/kong-admin -n kong 8001:8001 curl http://localhost:8001/status
```

## 8. 🚀 Actualizar Kong con Helm

Si ya existe el release y solo quieres aplicar cambios:

```bash
   helm upgrade --install kong kong/kong -n kong --create-namespace -f kong-values.yaml
```

## 9. 🚀 Eliminar el release existente

Si ya existe el release y solo quieres eliminarlo:

```bash
   helm uninstall kong -n kong
```

## 9. ✅ Verificar que Kong esté funcionando

```bash
   kubectl get pods -n kong
   kubectl get svc -n kong
```

### Para probar la conexión:

```bash
curl -i $(minikube service kong-proxy -n kong --url)
```

## 🧠 Tips avanzados

### 🔄 Auto-reload y configuración dinámica

- Konnect envía configuración dinámica al data plane.

- El data_plane sincroniza automáticamente cada pocos segundos.

### 📈 Telemetría

- Las métricas se envían al control plane de Konnect (si habilitado).

### 🔐 Seguridad

- No uses el control plane público en entornos productivos.

- Protege tu JWT token y certificados en secretos seguros.
