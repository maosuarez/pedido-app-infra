# CLAUDE.md — pedido-app-infra

## Contexto del repositorio

Este repo contiene el **despliegue** de la aplicación de gestión de pedidos.
Su única responsabilidad es describir el estado deseado del cluster Kubernetes.
No contiene código fuente, Dockerfiles ni lógica de aplicación — eso vive en `pedido-app-src`.

ArgoCD monitorea este repositorio. Cualquier cambio en `main` se refleja automáticamente en el cluster.

**Stack:**
- Packaging: Helm 3 (chart raíz con subcharts)
- CD: ArgoCD (GitOps, sync automático)
- DB: PostgreSQL via chart oficial Bitnami
- Entornos: `dev` (namespace `dev`) y `prod` (namespace `prod`)
- Registry: Docker Hub (imágenes producidas por `pedido-app-src`)

---

## Reglas de trabajo

1. **Este repo no sabe nada del código.** Solo sabe de imágenes ya publicadas en Docker Hub. Los tags se actualizan en `values-dev.yaml` o `values-prod.yaml`.

2. **Nada hardcodeado.** Todo valor configurable debe estar en `values.yaml`. Si algo está fijo en un template, es un error de diseño.

3. **Credenciales nunca en texto plano.** Los Secrets usan valores base64 en el chart. En producción real se usaría `helm-secrets` o External Secrets Operator — para este proyecto, base64 en Secret es aceptable pero debe documentarse.

4. **Siempre `resources.requests` y `resources.limits`.** Todo Deployment debe tener ambos definidos y parametrizados en `values.yaml`.

5. **Labels consistentes.** Todos los recursos deben tener al menos: `app`, `component`, `environment`, `managed-by: helm`.

6. **`helm lint` antes de proponer cualquier cambio.** Si el template no pasa lint, no se entrega.

7. **Explica antes de implementar.** Si hay decisiones de diseño (estructura de subcharts, scope del Ingress, estrategia de PVC), descríbelas y espera confirmación.

8. **Un paso a la vez.** Chart raíz → subchart db → subchart backend → subchart frontend → Ingress → ArgoCD. No se salta orden.

---

## Estructura objetivo

```
pedido-app-infra/
├── charts/
│   └── pedido-app/
│       ├── Chart.yaml              # Chart raíz, declara dependencia Bitnami
│       ├── values.yaml             # Todos los defaults parametrizables
│       ├── values-dev.yaml         # Overrides de dev (réplicas bajas, tag dev-*)
│       ├── values-prod.yaml        # Overrides de prod (réplicas altas, tag semver)
│       └── charts/
│           ├── backend/
│           │   ├── Chart.yaml
│           │   └── templates/
│           │       ├── deployment.yaml
│           │       ├── service.yaml     # ClusterIP
│           │       ├── configmap.yaml   # DB_HOST, DB_PORT, DB_NAME, SERVER_PORT
│           │       ├── secret.yaml      # DB_USER, DB_PASSWORD (base64)
│           │       └── hpa.yaml         # HPA por CPU
│           ├── frontend/
│           │   ├── Chart.yaml
│           │   └── templates/
│           │       ├── deployment.yaml
│           │       └── service.yaml     # ClusterIP
│           └── db/
│               └── Chart.yaml          # Wrapper Bitnami: solo declara dependencia + PVC
├── environments/
│   ├── dev/
│   │   └── application.yaml        # ArgoCD Application → namespace dev, sync auto
│   └── prod/
│       └── application.yaml        # ArgoCD Application → namespace prod, sync auto
├── ingress/
│   └── ingress.yaml                # Ingress compartido: / → frontend, /api/* → backend
├── CLAUDE.md
└── README.md
```

---

## Parámetros clave en values.yaml

```yaml
# Imagen y tag de cada componente
backend:
  image:
    repository: <user>/pedido-backend
    tag: "dev-latest"
  replicaCount: 1
  resources:
    requests: { cpu: "100m", memory: "256Mi" }
    limits:   { cpu: "500m", memory: "512Mi" }
  hpa:
    minReplicas: 1
    maxReplicas: 5
    targetCPUUtilizationPercentage: 70

frontend:
  image:
    repository: <user>/pedido-frontend
    tag: "dev-latest"
  replicaCount: 1
  resources:
    requests: { cpu: "50m",  memory: "64Mi" }
    limits:   { cpu: "200m", memory: "128Mi" }

db:
  auth:
    database: pedidos
    username: pedidos_user
    password: ""           # Siempre via Secret, nunca texto plano en prod
  primary:
    persistence:
      size: 1Gi            # Dev: 1Gi | Prod: 10Gi+
```

---

## Criterios de evaluación (referencia constante)

| Criterio | Peso | Target |
|---|---|---|
| Estructura del chart | 20 pts | Subcharts limpios, values por ambiente, sin hardcode |
| Recursos obligatorios | 25 pts | Todos presentes, funcionales, con buenas prácticas |
| ArgoCD | 20 pts | Application por ambiente, sync automático |
| Funcionamiento end-to-end | 20 pts | Ingress correcto, persistencia real |
| Buenas prácticas | 10 pts | resources, labels, credenciales seguras |
| Documentación | 5 pts | README claro y reproducible |

---

## Comandos de validación

```bash
# Siempre antes de hacer PR
helm dependency update charts/pedido-app
helm lint charts/pedido-app
helm template pedido-app charts/pedido-app -f charts/pedido-app/values-dev.yaml | grep -E "kind:|name:"
```

---

## Flujo de trabajo con Claude Code

```
1. Chart.yaml raíz + dependencia Bitnami
2. values.yaml base con todos los parámetros
3. values-dev.yaml y values-prod.yaml
4. Subchart db/ (wrapper Bitnami + PVC)
5. Subchart backend/ (Deployment + Service + ConfigMap + Secret + HPA)
6. Subchart frontend/ (Deployment + Service)
7. Ingress compartido
8. environments/dev/application.yaml (ArgoCD)
9. environments/prod/application.yaml (ArgoCD)
10. helm lint final + helm template review
```

Nunca saltes pasos. Si el usuario pide avanzar más rápido, confirma que entiende lo que se está generando.