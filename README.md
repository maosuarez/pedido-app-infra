# pedido-app-infra

> Helm charts + ArgoCD GitOps pipeline para el sistema de gestión de pedidos.
> Este repositorio es el **source of truth** del estado del cluster Kubernetes.
> El código fuente y las imágenes Docker viven en [`pedido-app-src`](https://github.com/maosuarez/pedido-app-src).

---

## 📂 Repositorios del proyecto

| Repo | Responsabilidad |
|---|---|
| `pedido-app-src` | Código fuente + Dockerfiles + CI |
| `pedido-app-infra` *(este repo)* | Helm charts + ArgoCD + despliegue en K8s |

---

## 🏗️ Arquitectura de despliegue

```
  pedido-app-src (CI)
        │
        │  docker push maosuarez/pedido-backend:<tag>
        │  docker push maosuarez/pedido-frontend:<tag>
        ▼
   Docker Hub
        │
        │  actualizar image.tag en values-dev.yaml → git push
        ▼
  pedido-app-infra (este repo)
        │
        │  ArgoCD detecta cambio en main
        ▼
┌───────────────────────────────────────┐
│          Kubernetes Cluster           │
│                                       │
│  namespace: dev       namespace: prod │
│  ┌────────────────┐  ┌─────────────┐ │
│  │ Ingress(nginx) │  │   Ingress   │ │
│  │ / → frontend   │  │ / → front.  │ │
│  │ /api/ → back.  │  │ /api/ → b.  │ │
│  └───────┬────────┘  └──────┬──────┘ │
│   ┌──────▼──────┐    ┌──────▼──────┐ │
│   │  Frontend   │    │  Frontend   │ │
│   └──────┬──────┘    └──────┬──────┘ │
│   ┌──────▼──────┐◄HPA ┌─────▼──────┐ │
│   │   Backend   │    │   Backend  │ │
│   └──────┬──────┘    └──────┬─────┘ │
│   ┌──────▼──────┐    ┌──────▼─────┐ │
│   │ PostgreSQL  │    │ PostgreSQL │ │
│   │   + PVC     │    │   + PVC    │ │
│   └─────────────┘    └────────────┘ │
└───────────────────────────────────────┘
```

---

## 📁 Estructura del repositorio

```
pedido-app-infra/
├── charts/
│   └── pedido-app/
│       ├── Chart.yaml              # Chart raíz + dependencia Bitnami PostgreSQL
│       ├── values.yaml             # Defaults base (todos los parámetros)
│       ├── values-dev.yaml         # Overrides dev: réplicas bajas, tag Major.Minor.Fix-dev
│       ├── values-prod.yaml        # Overrides prod: réplicas altas, tag Major.Minor.Fix
│       └── charts/
│           ├── backend/
│           │   ├── Chart.yaml
│           │   └── templates/
│           │       ├── deployment.yaml   # Deployment con resources y probes
│           │       ├── service.yaml      # ClusterIP
│           │       ├── configmap.yaml    # DB_HOST, DB_PORT, DB_NAME
│           │       ├── secret.yaml       # DB_USER, DB_PASSWORD (base64)
│           │       └── hpa.yaml          # HPA por CPU
│           ├── frontend/
│           │   ├── Chart.yaml
│           │   └── templates/
│           │       ├── deployment.yaml
│           │       └── service.yaml      # ClusterIP
│           └── db/
│               └── Chart.yaml            # Wrapper Bitnami con PVC
├── environments/
│   ├── dev/
│   │   └── application.yaml        # ArgoCD Application → namespace dev
│   └── prod/
│       └── application.yaml        # ArgoCD Application → namespace prod
├── CLAUDE.md
└── README.md
```

---

## ⚙️ Prerrequisitos del cluster

| Herramienta | Instalación rápida |
|---|---|
| ingress-nginx | `helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace --set controller.service.externalTrafficPolicy=Local` |
| metrics-server | `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` |
| ArgoCD | ver sección ArgoCD abajo |

---

## 🚀 Instalación manual con Helm

```bash
# 1. Clonar
git clone https://github.com/maosuarez/pedido-app-infra
cd pedido-app-infra

# 2. Descargar dependencias (Bitnami PostgreSQL)
helm dependency update charts/pedido-app

# 3. Validar
helm lint charts/pedido-app
helm template pedido-app charts/pedido-app -f charts/pedido-app/values-dev.yaml

# 4a. Instalar en dev
helm install pedido-app charts/pedido-app \
  -f charts/pedido-app/values-dev.yaml \
  --namespace dev --create-namespace

# 4b. Instalar en prod
helm install pedido-app charts/pedido-app \
  -f charts/pedido-app/values-prod.yaml \
  --namespace prod --create-namespace

# 5. Verificar
kubectl get all -n dev
kubectl get ingress -n dev
```

### Actualizar imagen tras nuevo CI

```bash
# Editar values-dev.yaml con el nuevo tag
# backend.image.tag: "1.0.0-dev"
# Luego:
helm upgrade pedido-app charts/pedido-app \
  -f charts/pedido-app/values-dev.yaml --namespace dev

# O simplemente hacer git push → ArgoCD lo aplica solo
```

---

## 🔄 Configuración de ArgoCD

```bash
# Instalar ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que esté listo
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s

# Obtener password inicial
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Acceder a la UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Abrir: https://localhost:8080  |  user: admin

# Registrar el repo (si es privado)
argocd repo add https://github.com/maosuarez/pedido-app-infra \
  --username <user> --password <token>

# Aplicar Applications
kubectl apply -f environments/dev/application.yaml
kubectl apply -f environments/prod/application.yaml
```

### Cómo funciona la sincronización automática

Cuando se hace `git push` a `main` en este repo, ArgoCD (con polling cada 3 min o via webhook) detecta el diff entre el estado del repo y el estado del cluster y aplica los cambios sin intervención manual. Configurado con `syncPolicy.automated.selfHeal: true` y `prune: true`.

---

## 🌍 Configuración de hosts local (DNS local)

Para acceder por nombre de dominio desde tu máquina, el sistema operativo debe resolver
`pedido-app.dev.local` y `pedido-app.prod.local` a la IP del Ingress (`20.185.8.196`).

### Windows (como Administrador)

Abre PowerShell como administrador y ejecuta:

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "20.185.8.196  pedido-app.dev.local"
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "20.185.8.196  pedido-app.prod.local"
```

Verifica:
```powershell
Get-Content "C:\Windows\System32\drivers\etc\hosts" | Select-String "pedido-app"
```

### Linux / WSL (Parrot OS, Ubuntu, etc.)

```bash
echo "20.185.8.196  pedido-app.dev.local"  | sudo tee -a /etc/hosts
echo "20.185.8.196  pedido-app.prod.local" | sudo tee -a /etc/hosts

# Verifica
grep "pedido-app" /etc/hosts
```

### Probar con curl y navegador

```bash
# Dev
curl http://pedido-app.dev.local/api/pedidos        # Espera: []
curl http://pedido-app.dev.local/actuator/health    # Espera: {"status":"UP"}

# Prod
curl http://pedido-app.prod.local/api/pedidos
curl http://pedido-app.prod.local/actuator/health
```

En el navegador simplemente abre `http://pedido-app.dev.local/api/pedidos`.

> **Nota AKS**: El health probe de ingress-nginx en AKS requiere `externalTrafficPolicy: Local`.
> Si el curl da TCP timeout, ejecuta:
> `helm upgrade ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --set controller.service.externalTrafficPolicy=Local`

---

## 🌐 Endpoints de acceso

```bash
# Obtener IP del Ingress
kubectl get ingress -n dev
kubectl get ingress -n prod
```

## Comando de puente port-forwarding
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

IP pública del Ingress (AKS): `20.185.8.196`

| Entorno | Host local | Backend API |
|---|---|---|
| dev  | `pedido-app.dev.local`  | `http://pedido-app.dev.local/api/pedidos`  |
| prod | `pedido-app.prod.local` | `http://pedido-app.prod.local/api/pedidos` |

> El Ingress enruta por header `Host`, no por IP directa. Siempre usar el hostname.

---

## 🐳 Imágenes utilizadas

| Componente | Imagen | Dev tag | Prod tag |
|---|---|---|---|
| Backend | `maosuarez/pedido-backend` | `Major.Minor.Fix-dev` (ej. `1.0.0-dev`) | `Major.Minor.Fix` (ej. `1.0.0`) |
| Frontend | `maosuarez/pedido-frontend` | `Major.Minor.Fix-dev` (ej. `1.0.0-dev`) | `Major.Minor.Fix` (ej. `1.0.0`) |

> Los tags se actualizan en `values-dev.yaml` y `values-prod.yaml` respectivamente.
> Formato: `Major.Minor.Fix-dev` para desarrollo, `Major.Minor.Fix` para producción (SemVer).

---

## 🔀 Flujo de trabajo Git

```
main  (protegido, requiere PR)
 ├── feature/pa-<tarea>   → PR → revisión → merge → ArgoCD sync
 └── feature/pb-<tarea>   → PR → revisión → merge → ArgoCD sync
```

Todo merge a `main` en este repo desencadena una sincronización automática en el cluster.

---

## 📋 Backlog — Repo infra

Leyenda: `⬜ Pendiente` · `🔄 En progreso` · `✅ Completado` · `🔴 Bloqueado`

### 🏗️ Setup

| # | Tarea | Responsable | Estado | Rama |
|---|---|---|---|---|
| 1 | Crear estructura base de carpetas | Persona A | ✅ | `feature/pa-helm-root` |
| 2 | Agregar `CLAUDE.md` y `README.md` | Persona A | ✅ | `feature/pa-repo-setup` |
| 3 | Verificar cluster con ingress-nginx y metrics-server | Persona B | ✅ | — |
| 4 | Instalar y configurar ArgoCD en el cluster | Persona B | ✅ | — |

### 📦 Helm Chart

| # | Tarea | Responsable | Estado | Rama |
|---|---|---|---|---|
| 5 | `Chart.yaml` raíz con dependencia Bitnami PostgreSQL | Persona A | ✅ | `feature/pa-helm-root` |
| 6 | `values.yaml` base — imagen, tag, réplicas, resources, credenciales | Persona A | ✅ | `feature/pa-helm-root` |
| 7 | `values-dev.yaml` — overrides de desarrollo | Persona A | ✅ | `feature/pa-helm-root` |
| 8 | `values-prod.yaml` — overrides de producción | Persona B | ✅ | `feature/pa-helm-root` |
| 9 | Subchart `db/` — wrapper Bitnami + PVC | Persona A | ✅ | `feature/pa-subchart-db` |
| 10 | Subchart `backend/` — Deployment + Service | Persona A | ✅ | `feature/pb-subchart-backend` |
| 11 | Subchart `backend/` — ConfigMap + Secret | Persona A | ✅ | `feature/pb-subchart-backend` |
| 12 | Subchart `backend/` — HPA | Persona A | ✅ | `feature/pb-subchart-backend` |
| 13 | Subchart `frontend/` — Deployment + Service | Persona B | ✅ | `feature/pa-subchart-frontend` |
| 14 | Ingress — `/api/*` → backend (frontend pendiente) | Persona A | ✅ | `feature/pa-ingress` |

### 🔄 ArgoCD

| # | Tarea | Responsable | Estado | Rama |
|---|---|---|---|---|
| 15 | `environments/dev/application.yaml` con sync automático | Persona B | ✅ | `feature/pb-argocd-apps` |
| 16 | `environments/prod/application.yaml` con sync automático | Persona B | ✅ | `feature/pb-argocd-apps` |
| 17 | Validar sync automático al cambiar `values-dev.yaml` | Ambos | ✅ | — |

### ✅ Validación

| # | Tarea | Responsable | Estado | Rama |
|---|---|---|---|---|
| 18 | `helm lint` y `helm template` sin errores | Persona A | ✅ | — |
| 19 | Despliegue en dev: todos los pods Running | Persona B | ✅ | — |
| 20 | Verificar persistencia: reiniciar PostgreSQL, datos intactos | Persona B | ✅ | — |
| 21 | Verificar Ingress: frontend en `/`, API en `/api/` | Ambos | ✅ | — |
| 22 | Verificar HPA: carga → escalado automático del backend | Ambos | ✅ | — |

---

## 📊 Progreso — Repo infra

```
Setup          [██████████]  4/4   ← ingress-nginx, metrics-server y ArgoCD instalados
Helm Chart     [██████████]  10/10 ← todos los subcharts completos
ArgoCD         [██████████]  3/3   ← Applications dev y prod aplicadas, sync automático activo
Validación     [██████████]  5/5   ← helm lint OK + dev Running + Ingress funcional
──────────────────────────────
Total          [██████████]  22/22
```

> Actualiza el estado: `⬜ → 🔄 → ✅`
