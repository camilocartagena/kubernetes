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
2. Seleccionar: Gateway > Kubernetes > Hybrid Mode
3. Copiar el DP JWT Token

## 4. 📁 Crear archivo kong-values.yaml con configuración avanzada

```yaml
# kong-values.yaml

env:
  role: data_plane
  cluster_control_plane: "your-control-plane-id.konghq.com:443"
  cluster_server_name: "your-control-plane-id.konghq.com"
  cluster_telemetry_endpoint: "telemetry.konghq.com:443"
  cluster_cert: |
    -----BEGIN CERTIFICATE-----
    (tu certificado aquí)
    -----END CERTIFICATE-----
  cluster_cert_key: |
    -----BEGIN PRIVATE KEY-----
    (tu clave aquí)
    -----END PRIVATE KEY-----
  lua_ssl_trusted_certificate: /etc/secrets/kong-cluster-cert/tls.crt
  lua_ssl_verify_depth: 2
  database: "off"

secretVolumes:
  - kong-cluster-cert

volumes:
  - name: kong-cluster-cert
    secret:
      secretName: kong-cluster-cert

resources:
  limits:
    cpu: 500m
    memory: 1Gi
  requests:
    cpu: 200m
    memory: 512Mi

livenessProbe:
  httpGet:
    path: /status
    port: 8001
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /status
    port: 8001
  initialDelaySeconds: 5
  periodSeconds: 10

ingressController:
  enabled: false

proxy:
  type: LoadBalancer
```

## 5. 🔐 Crear el secreto kong-cluster-cert en Kubernetes

```bash
   kubectl create secret generic kong-cluster-cert \
    --from-file=tls.crt=cluster.crt \
    --from-file=tls.key=cluster.key \
    -n kong
```

> 📝 Los archivos cluster.crt y cluster.key vienen en el bundle que descargas desde Konnect al agregar un nuevo runtime.

## 6. 🚀 Instalar Kong con Helm

```bash
   helm install kong kong/kong -n kong -f kong-values.yaml
```

## 7. ✅ Verificar que Kong esté funcionando

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
