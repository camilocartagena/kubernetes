# 🌐 Exponer Kong Gateway localmente usando ngrok

Este documento explica cómo exponer un Kong Gateway (Data Plane) local que corre en Kubernetes para integrarlo con Azure AD a través de Kong Konnect, utilizando `ngrok` como túnel HTTPS público.

---

## 📦 Requisitos

- Kong Gateway instalado en local (por ejemplo, en Minikube).
- Azure AD ya configurado con una aplicación registrada.
- Plugin OIDC habilitado en Kong Konnect.
- Acceso a terminal con permisos administrativos.
- Tener instalado `kubectl`.

---

## 🧰 Instalación de ngrok

### 🖥️ macOS

```bash
brew install ngrok
```

## 🐧 Linux (Debian/Ubuntu)

```bash
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null && \
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list && \
sudo apt update && sudo apt install ngrok
```

---

## 🪟 Windows

1. Descarga desde: https://ngrok.com/download
2. Extrae el archivo `.zip`.
3. Agrega el ejecutable `ngrok.exe` a tu variable de entorno `PATH`.

---

## 🔐 Autenticación en ngrok

1. Regístrate o inicia sesión en: https://dashboard.ngrok.com
2. Copia tu token de autenticación (`authtoken`).
3. Ejecuta:

```bash
ngrok config add-authtoken <tu-token>
```

---

## 🚀 Exponer Kong Gateway local

### Opción 1: Usando `kubectl port-forward`

1. Ejecuta:

```bash
kubectl port-forward -n kong svc/kong-proxy 8000:80
```

2. En otra terminal:

```bash
ngrok http 8000
```

---

### Opción 2: Usando `Ingress` (NGINX/Traefik)

1. Define un recurso `Ingress` en tu cluster para el servicio de Kong.
2. Luego ejecuta:

```bash
ngrok http <puerto-de-tu-ingress>
```

---

## 🌐 Resultado

ngrok te dará una URL como:

```
https://1234abcd.ngrok.io
```

Tu URL de redirección para Azure será:

```bash
https://1234abcd.ngrok.io/oauth2/callback
```

Usa esta URL:

- Como `Redirect URI` en la app registrada en **Azure AD**.
- En la configuración del plugin **OIDC** en **Kong Konnect**.

---

## ✅ Próximo paso

Configura el plugin OIDC en Konnect usando:

- `client_id`
- `client_secret`
- `issuer` → URL del Discovery OIDC de Azure
- `redirect_uri` → URL pública de ngrok (`/oauth2/callback`)
