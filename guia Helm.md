# 🧭 Guía Rápida de Helm para Kubernetes

## 🚀 ¿Qué es Helm?

**Helm** es el **gestor de paquetes para Kubernetes**, similar a apt o yum en Linux. Permite instalar, actualizar, configurar y gestionar aplicaciones como paquetes llamados **charts**.

---

## 🧠 Conceptos Clave

| Concepto       | Descripción                                                              |
| -------------- | ------------------------------------------------------------------------ |
| **Chart**      | Un paquete Helm. Contiene toda la configuración para desplegar una app.  |
| **Release**    | Una instancia desplegada de un chart en un clúster de Kubernetes.        |
| **Repository** | Fuente de charts. Ej: Bitnami, Artifact Hub, etc.                        |
| **Values**     | Archivo `values.yaml` con variables de configuración para personalizar.  |
| **Template**   | Plantillas que se convierten en manifiestos YAML estándar de Kubernetes. |

---

## 🛠 Instalación de Helm

```bash
# En Linux/macOS
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

# O con Homebrew en macOS
brew install helm
```

## 🔧 Comandos Básicos

### 1. 📦 Agregar un repositorio

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 2. 🚀 Instalar un chart

```bash
helm install mi-release bitnami/nginx
```

Esto instala nginx usando el chart de Bitnami, creando un release llamado mi-release.

### 3. 📁 Ver los charts disponibles en un repo

```bash
helm search repo nginx
```

### 4. 📝 Ver y modificar configuración (values.yaml)

```bash
# Ver valores por defecto
helm show values bitnami/nginx > values.yaml

# Instalar usando archivo modificado
helm install mi-release bitnami/nginx -f values.yaml
```

### 5. 🔄 Actualizar un release

```bash
helm upgrade mi-release bitnami/nginx -f values.yaml
```

### 6. ❌ Eliminar un release

```bash
helm uninstall mi-release
```

### 7. 📋 Ver releases instalados

```bash
helm list
```

### 8. 🔍 Ver historial de upgrades

```bash
helm history mi-release
```

## 🧪 Crear tu propio chart

```bash
helm create mi-chart
```

Esto genera una estructura como:

```bash
mi-chart/
├── Chart.yaml         # Metadatos del chart
├── values.yaml        # Variables de configuración
└── templates/         # Manifiestos de Kubernetes con Go templates
```

Puedes modificar templates/deployment.yaml, etc.

## 🌐 Recursos útiles

- Artifact Hub → Buscar charts Helm
- Documentación oficial de Helm
