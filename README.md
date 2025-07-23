# 🧭 Guía Completa para Usar Kubernetes en AWS

## 1. Introducción

**Kubernetes** es una plataforma de orquestación de contenedores. En AWS, las formas más comunes de usarlo son:

- **Amazon EKS (Elastic Kubernetes Service)** → Servicio administrado por AWS.
- **Kubernetes autogestionado en EC2** → Control total, pero mayor complejidad.

<img src="k8.png" width="500" />

> ✅ **Recomendación**: Usar **Amazon EKS** por su integración nativa con servicios AWS y gestión automatizada del plano de control.

---

## 2. Términos Clave y Recomendaciones para tener en cuenta

### 🧱 Pod

- **Qué es:** Unidad básica de ejecución en Kubernetes que contiene uno o más contenedores compartiendo red y almacenamiento.
- **Recomendaciones:**
  - Un contenedor por pod, salvo que necesites sidecars (ej. logs, proxy).
  - Define `resources.requests` y `resources.limits` para CPU y memoria.
  - Usa `livenessProbe` y `readinessProbe` para asegurar salud y disponibilidad.
  - Evita uso de `hostNetwork` y `privileged: true` a menos que sea estrictamente necesario.

---

### 🖥️ Node

- **Qué es:** Una instancia EC2 (cuando usas EKS o Kubernetes en EC2) que ejecuta pods.
- **Recomendaciones:**
  - Usa **autoscaling groups** con mínimo 2 zonas de disponibilidad.
  - Usa etiquetas (`nodeSelector`, `taints`, `affinity`) para controlar dónde se ejecutan los pods.
  - Usa tipos EC2 según el perfil de carga: `t3`, `m5`, `c5`, `r5`, etc.
  - Aplica políticas de seguridad estrictas en los `Security Groups`.

---

### 🧩 Deployment

- **Qué es:** Recurso que permite desplegar y administrar múltiples réplicas de un pod.
- **Recomendaciones:**
  - Usa `replicas: 2+` en producción para alta disponibilidad.
  - Activa `rollingUpdate` para despliegues sin downtime.
  - Define `revisionHistoryLimit` para controlar el rollback.
  - Usa `strategy.type: RollingUpdate` y configura `maxUnavailable` / `maxSurge`.

---

### 🌐 Service

- **Qué es:** Abstracción que define cómo acceder a los pods. Tipos: `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`.
- **Recomendaciones:**
  - Usa `ClusterIP` para comunicación interna.
  - Usa `LoadBalancer` solo cuando necesitas acceso externo (en EKS se crea un ELB).
  - Evita `NodePort` a menos que estés fuera de la nube.
  - Acompaña de un `HealthCheckNodePort` en servicios críticos.

---

### 🚪 Ingress

- **Qué es:** Gestiona el acceso HTTP/HTTPS a los servicios del clúster.
- **Recomendaciones:**
  - Usa **AWS Load Balancer Controller** para usar ALB como ingress.
  - Configura certificados TLS con **cert-manager** y ACM.
  - Organiza rutas por host/path (`example.com/api`, `example.com/app`).
  - Aplica políticas de WAF y reglas de seguridad en el ALB.

---

### 🗂️ Namespace

# ¿Qué es un namespace en Kubernetes?

Un **namespace** es como un "contenedor lógico" dentro del clúster que agrupa y aísla recursos como:

- **Pods**
- **Services**
- **Secrets**
- **ConfigMaps**
- **Deployments**
- **Ingress**, etc.

---

## ✅ ¿Por qué creamos el namespace kong?

| Razón                 | Descripción                                                                            |
| --------------------- | -------------------------------------------------------------------------------------- |
| 🎯 **Organización**   | Separamos los recursos de Kong de otros del clúster, evitando mezclar configuraciones. |
| 🔐 **Seguridad**      | Podemos aplicar políticas específicas (RBAC, NetworkPolicy) solo al namespace kong.    |
| 🧹 **Mantenimiento**  | Facilita la administración, monitoreo y limpieza (ej. `kubectl delete ns kong`).       |
| 🤖 **Automatización** | Algunos charts como el de Kong esperan ser instalados en un namespace definido.        |

- **Recomendaciones:**
  - Crea namespaces separados para `dev`, `staging`, `prod`.
  - Usa `ResourceQuota` y `LimitRange` para evitar consumo excesivo.
  - Aplica RBAC por namespace para mejorar la seguridad.
  - Establece políticas de red (`NetworkPolicy`) entre namespaces.

---

### ⚙️ ConfigMap y Secret

- **ConfigMap:** Almacena configuración no sensible (ej. variables de entorno, archivos de texto).
- **Secret:** Almacena datos sensibles (tokens, contraseñas, claves).
- **Recomendaciones:**
  - Monta ConfigMaps y Secrets como archivos o variables env.
  - Usa `kubectl create secret` o YAML codificado en base64 para Secrets.
  - Evita exponer Secrets en logs o volúmenes.
  - Para producción: integra con **AWS Secrets Manager** o **HashiCorp Vault**.

---

### 🧰 Helm

- **Qué es:** Gestor de paquetes de Kubernetes (como `apt` o `yum`).
- **Recomendaciones:**
  - Usa charts oficiales (bitnami, prometheus-community, etc.).
  - Personaliza `values.yaml` para cada entorno.
  - Integra en pipelines CI/CD para despliegues automatizados.
  - Usa `helm diff` y `helm upgrade --atomic` para mayor control.

---

### 🔐 IRSA (IAM Roles for Service Accounts)

- **Qué es:** Permite a los pods en EKS asumir roles de IAM mediante su ServiceAccount, sin necesidad de usar claves.
- **Recomendaciones:**
  - Crea roles IAM con políticas mínimas necesarias.
  - Asocia el rol al ServiceAccount con anotaciones:
    ```yaml
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
    ```
  - Habilita OIDC en el clúster EKS para usar IRSA.
  - Usa herramientas como `eksctl` para configurar IRSA fácilmente.

### ⚙️ Manifiesto en Kubernetes

En **Kubernetes (K8s)**, un **manifiesto** es un archivo de configuración en formato YAML o JSON que describe los recursos y su configuración en un clúster. Este archivo especifica el **estado deseado** de los recursos que Kubernetes debe gestionar.

## ¿Qué incluye un manifiesto?

El manifiesto puede describir muchos tipos de recursos en Kubernetes, como:

- **Pods**: Un conjunto de contenedores que comparten el mismo almacenamiento y red.
- **Deployments**: Define cómo debe ser gestionado un conjunto de pods.
- **Services**: Define cómo los pods se comunican entre sí o con el mundo exterior.
- **ConfigMaps**: Proporciona configuraciones a los contenedores.
- Y más...

## ¿Cómo funciona?

El manifiesto describe **cómo debe ser el estado final** del recurso, por ejemplo, cuántas réplicas de un pod deben ejecutarse, qué contenedor utilizar, qué puertos exponer, entre otros. Kubernetes asegura que el clúster siempre esté en ese estado deseado, creando, eliminando o actualizando los recursos conforme sea necesario.

## Ejemplo de un Manifiesto para un Pod

Aquí tienes un ejemplo básico de un manifiesto YAML para crear un **Pod** en Kubernetes que ejecuta un contenedor de **nginx**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mi-pod
spec:
  containers:
    - name: mi-contenedor
      image: nginx:latest
      ports:
        - containerPort: 80
```

### 🧱 Sidecar Containers en Kubernetes ¿Qué es un Sidecar?

Un **Sidecar** es un contenedor que se ejecuta junto a tu contenedor principal dentro del mismo **Pod**, compartiendo:

- Red (IP, puerto)
- Volúmenes (almacenamiento)
- Ciclo de vida del pod

El propósito del sidecar es **extender o complementar** la funcionalidad del contenedor principal sin modificar su lógica.

---

## 🧰 Casos de uso comunes

| Caso de uso                    | Descripción                                                                    |
| ------------------------------ | ------------------------------------------------------------------------------ |
| **Proxy inverso**              | Envoy, Istio o Linkerd manejando tráfico de entrada/salida del pod.            |
| **Log forwarder**              | Sidecars como FluentBit o Filebeat recogen y envían logs a servicios externos. |
| **Agente de monitoreo**        | Prometheus exporter o Datadog Agent para métricas a nivel de aplicación.       |
| **Sincronización de archivos** | Contenedor que descarga archivos o certificados que el principal necesita.     |
| **Inyección de secretos**      | Sidecars que montan secretos desde Vault o AWS Secrets Manager.                |
| **Actualización dinámica**     | Servicio que monitorea cambios y recarga la app (hot reload).                  |

---

### 🛠️ Ejemplo de configuración: FluentBit como sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logs
spec:
  containers:
    - name: main-app
      image: my-app:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    - name: fluentbit
      image: fluent/fluent-bit
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

## 🔁 Contenedores Sidecar y Logs Compartidos

Ambos contenedores comparten `/var/log/app`, permitiendo que **FluentBit** recoja los logs que genera la app principal.

---

### 🧪 Ejemplo de Sidecar: Istio (malla de servicios)

Con **Istio** instalado, puedes inyectar sidecars automáticamente en tu namespace:

```bash
kubectl label namespace my-app istio-injection=enabled
```

Istio añadirá un sidecar istio-proxy (Envoy) al pod:

```yaml
containers:
  - name: my-app
    image: my-app:latest
  - name: istio-proxy
    image: istio/proxyv2
    args: [...configuración de mTLS, tracing, routing...]
```

## 3. Arquitectura Recomendada en AWS

### ⚙️ Componentes Principales

- **Amazon EKS** (plano de control)
- **Node Groups** con:
  - EC2 Auto Scaling Groups
  - AWS Fargate (opcional)
- **ALB/NLB** como Ingress Controller (AWS Load Balancer Controller)
- **VPC** con subredes públicas y privadas
- **IAM + IRSA**
- **Observabilidad** con CloudWatch, Prometheus, Grafana
- **Servicios externos**: S3, RDS, DynamoDB, etc.

### 🌐 Diagrama Conceptual

```plaintext
[ ALB (Ingress Controller) ]
          ↓
   [ Kubernetes Services ]
          ↓
     [ Pods & Deployments ]
          ↓
     [ Node Groups (EC2) ]
          ↓
      [ EKS Control Plane ]
```

# Buenas Prácticas

## Seguridad

- Usa **Namespaces** para separar ambientes.
- Aplica **RBAC** para acceso granular.
- Emplea **IRSA** en lugar de claves embebidas.
- Restringe:
  - `hostNetwork`
  - `hostPath`
  - `privileged=true`

## Alta Disponibilidad

- Despliega en múltiples **Zonas de Disponibilidad**.
- Usa **PodDisruptionBudgets** y **Affinity/AntiAffinity**.
- Implementa:
  - `livenessProbe`
  - `readinessProbe`

## Escalabilidad

- Activa **Cluster Autoscaler** para nodos EC2.
- Configura **Horizontal Pod Autoscaler (HPA)**.

## Observabilidad

- **Prometheus + Grafana** para métricas.
- **CloudWatch Logs** para registros.
- **FluentBit/Fluentd** para recolección de logs.
- **X-Ray** o **Jaeger** para trazabilidad.

## CI/CD

- Usa **GitOps** con **ArgoCD** o **FluxCD**.
- Administra despliegues con **Helm** o **Kustomize**.

# Notas Adicionales

| Componente         | Detalle                                                                  |
| ------------------ | ------------------------------------------------------------------------ |
| **resources**      | Limita el uso de CPU/Memoria para evitar saturar el clúster              |
| **livenessProbe**  | Detecta si NGINX está vivo o debe reiniciarse                            |
| **readinessProbe** | Asegura que NGINX esté listo antes de recibir tráfico                    |
| **NodePort**       | Permite exponer el contenedor al exterior a través de un puerto del nodo |
