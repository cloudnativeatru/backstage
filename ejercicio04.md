# Ejercicio Intermedio — CI/CD Visible en Backstage
## "El pipeline que nadie podía encontrar"

> **Nivel:** Intermedio
> **Duración estimada:** 20 minutos
> **Prerrequisitos:** Haber completado los ejercicios 1–3 de la Sesión 2 (tener al menos un componente registrado en Backstage con `catalog-info.yaml`)

---

## 🎬 Escenario

Eres parte del equipo de plataforma de **Fintech Rápida S.A.**, una startup que procesa pagos para pequeños comercios. Tienen 4 microservicios corriendo en producción:

```
api-gateway        →  punto de entrada público
payments-service   →  procesa cobros con Stripe/Culqi
notifications-service →  envía emails y SMS
audit-logger       →  registra cada transacción para compliance
```

El problema: cada vez que algo falla en producción, el equipo pasa **20 minutos buscando** en qué estado está el pipeline del servicio afectado. Nadie sabe de memoria la URL de GitHub Actions de cada repo. El CTO está harto y pide que **el estado del CI sea visible directamente desde Backstage** sin tener que abrir GitHub.

Tu misión: registrar los 4 servicios en el catálogo y conectar sus pipelines de GitHub Actions de forma que cualquier persona pueda ver el estado del build desde la ficha del componente.

---

## Parte 1 — Preparar los repositorios (15 min)

### 1.1 Crear la estructura de repos

Para este ejercicio trabajarás con **un solo repositorio** que simula el monorepo de Fintech Rápida. Crea la siguiente estructura en un repo tuyo existente (o uno nuevo):

```bash
mkdir fintech-rapida && cd fintech-rapida
git init
mkdir -p services/api-gateway
mkdir -p services/payments-service
mkdir -p services/notifications-service
mkdir -p services/audit-logger
mkdir -p .github/workflows
```

### 1.2 Crear un workflow de GitHub Actions por servicio

Cada servicio necesita su propio workflow. Crea los siguientes archivos:

**`.github/workflows/api-gateway.yaml`**
```yaml
name: CI — api-gateway

on:
  push:
    branches: [main]
    paths:
      - 'services/api-gateway/**'
  pull_request:
    paths:
      - 'services/api-gateway/**'
  workflow_dispatch:    # permite ejecutarlo manualmente desde la UI

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: echo "npm ci — simulado"

      - name: Run tests
        run: |
          echo "✅ Tests del api-gateway pasaron"
          echo "Coverage: 87%"

      - name: Build
        run: echo "🏗️ Build completado"
```

**`.github/workflows/payments-service.yaml`**
```yaml
name: CI — payments-service

on:
  push:
    branches: [main]
    paths:
      - 'services/payments-service/**'
  pull_request:
    paths:
      - 'services/payments-service/**'
  workflow_dispatch:

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: echo "pip install -r requirements.txt — simulado"

      - name: Run tests
        run: |
          echo "✅ Tests del payments-service pasaron"
          echo "Coverage: 92%"

      - name: Security scan
        run: echo "🔒 Bandit security scan — sin hallazgos críticos"
```

**`.github/workflows/notifications-service.yaml`**
```yaml
name: CI — notifications-service

on:
  push:
    branches: [main]
    paths:
      - 'services/notifications-service/**'
  pull_request:
    paths:
      - 'services/notifications-service/**'
  workflow_dispatch:

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: echo "✅ Tests del notifications-service pasaron"
```

**`.github/workflows/audit-logger.yaml`**
```yaml
name: CI — audit-logger

on:
  push:
    branches: [main]
    paths:
      - 'services/audit-logger/**'
  pull_request:
    paths:
      - 'services/audit-logger/**'
  workflow_dispatch:

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: echo "✅ Tests del audit-logger pasaron"

      - name: Compliance check
        run: echo "📋 Compliance checks — OK"
```

### 1.3 Hacer el primer push y verificar que los workflows existen

```bash
git add .
git commit -m "chore: add GitHub Actions workflows for all services"
git push origin main
```

Ve a `https://github.com/<tu-usuario>/fintech-rapida/actions` y confirma que ves los 4 workflows listados. Ejecuta cada uno manualmente con **"Run workflow"** para tener al menos una ejecución en el historial.

---

## Parte 2 — Registrar los servicios en Backstage (20 min)

### 2.1 Crear los `catalog-info.yaml` de cada servicio

Crea `services/api-gateway/catalog-info.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: api-gateway
  title: "API Gateway"
  description: |
    Punto de entrada público de la plataforma Fintech Rápida.
    Maneja autenticación, rate limiting y routing hacia servicios internos.
  tags:
    - nodejs
    - gateway
    - public-facing
  labels:
    squad: platform
    tier: critical
  annotations:
    # ── GitHub integration ────────────────────────────────────────────────
    github.com/project-slug: <tu-usuario>/fintech-rapida

    # ── CI/CD: apunta al workflow específico de este servicio ─────────────
    # Formato: <org>/<repo>/actions/workflows/<nombre-del-archivo.yaml>
    backstage.io/ci-url: https://github.com/<tu-usuario>/fintech-rapida/actions/workflows/api-gateway.yaml

    # ── Runbook ───────────────────────────────────────────────────────────
    backstage.io/runbook-url: https://wiki.fintech-rapida.com/runbooks/api-gateway

  links:
    - url: https://github.com/<tu-usuario>/fintech-rapida/actions/workflows/api-gateway.yaml
      title: "GitHub Actions — CI Pipeline"
      icon: dashboard
      type: ci-cd
    - url: https://github.com/<tu-usuario>/fintech-rapida/actions
      title: "Todos los workflows"
      icon: web

spec:
  type: service
  lifecycle: production
  owner: group:default/platform-team
  system: fintech-rapida-platform
  dependsOn:
    - component:default/payments-service
    - component:default/notifications-service
    - component:default/audit-logger
```

Crea `services/payments-service/catalog-info.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payments-service
  title: "Payments Service"
  description: |
    Microservicio de procesamiento de pagos. Integra con Culqi y Stripe.
    Maneja cobros, reembolsos y conciliación.
  tags:
    - python
    - fastapi
    - payments
    - stripe
    - culqi
  labels:
    squad: payments
    tier: critical
    pci-scope: "true"       # label custom para compliance
  annotations:
    github.com/project-slug: <tu-usuario>/fintech-rapida
    backstage.io/ci-url: https://github.com/<tu-usuario>/fintech-rapida/actions/workflows/payments-service.yaml
    backstage.io/runbook-url: https://wiki.fintech-rapida.com/runbooks/payments

  links:
    - url: https://github.com/<tu-usuario>/fintech-rapida/actions/workflows/payments-service.yaml
      title: "GitHub Actions — CI Pipeline"
      icon: dashboard
      type: ci-cd

spec:
  type: service
  lifecycle: production
  owner: group:default/payments-team
  system: fintech-rapida-platform
  providesApis:
    - payments-api
  dependsOn:
    - component:default/audit-logger
    - resource:default/postgres-payments
```

Crea `services/notifications-service/catalog-info.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: notifications-service
  title: "Notifications Service"
  description: |
    Servicio de notificaciones multicanal: email (SendGrid),
    SMS (Twilio) y push notifications.
  tags:
    - nodejs
    - typescript
    - notifications
    - sendgrid
    - twilio
  labels:
    squad: platform
    tier: high
  annotations:
    github.com/project-slug: <tu-usuario>/fintech-rapida
    backstage.io/ci-url: https://github.com/<tu-usuario>/fintech-rapida/actions/workflows/notifications-service.yaml

  links:
    - url: https://github.com/<tu-usuario>/fintech-rapida/actions/workflows/notifications-service.yaml
      title: "GitHub Actions — CI Pipeline"
      icon: dashboard
      type: ci-cd

spec:
  type: service
  lifecycle: production
  owner: group:default/platform-team
  system: fintech-rapida-platform
  dependsOn:
    - component:default/audit-logger
```

Crea `services/audit-logger/catalog-info.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: audit-logger
  title: "Audit Logger"
  description: |
    Servicio de auditoría. Registra cada transacción y evento relevante
    para cumplimiento regulatorio (SBS, PCI-DSS).
  tags:
    - python
    - compliance
    - audit
    - kafka
  labels:
    squad: compliance
    tier: critical
    regulatory: "true"
  annotations:
    github.com/project-slug: <tu-usuario>/fintech-rapida
    backstage.io/ci-url: https://github.com/<tu-usuario>/fintech-rapida/actions/workflows/audit-logger.yaml

  links:
    - url: https://github.com/<tu-usuario>/fintech-rapida/actions/workflows/audit-logger.yaml
      title: "GitHub Actions — CI Pipeline"
      icon: dashboard
      type: ci-cd

spec:
  type: service
  lifecycle: production
  owner: group:default/compliance-team
  system: fintech-rapida-platform
  dependsOn:
    - resource:default/kafka-audit-topic
    - resource:default/postgres-audit
```

### 2.2 Crear los grupos y el sistema

Crea `backstage/org.yaml` en la raíz del repo:

```yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: fintech-rapida-platform
  description: "Plataforma completa de pagos de Fintech Rápida S.A."
  tags: [core, revenue-critical]
spec:
  owner: group:default/platform-team
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: platform-team
  description: "Equipo de plataforma e infraestructura"
spec:
  type: team
  profile:
    displayName: "Platform Team"
    email: platform@fintech-rapida.com
  children: []
  members: []
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: payments-team
  description: "Equipo de procesamiento de pagos"
spec:
  type: team
  profile:
    displayName: "Payments Team"
    email: payments@fintech-rapida.com
  children: []
  members: []
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: compliance-team
  description: "Equipo de compliance y auditoría"
spec:
  type: team
  profile:
    displayName: "Compliance Team"
    email: compliance@fintech-rapida.com
  children: []
  members: []
```

### 2.3 Commitear y registrar todo en Backstage

```bash
git add .
git commit -m "chore: add catalog-info.yaml for all services + org definition"
git push origin main
```

Registra en Backstage en este orden (para que las dependencias resuelvan bien):

1. `backstage/org.yaml` → grupos y sistema
2. `services/audit-logger/catalog-info.yaml`
3. `services/payments-service/catalog-info.yaml`
4. `services/notifications-service/catalog-info.yaml`
5. `services/api-gateway/catalog-info.yaml`

---

## Parte 3 — Conectar GitHub Actions con el plugin de Backstage (20 min)

La annotation `backstage.io/ci-url` añade un link, pero para tener el estado del build **inline en la ficha** necesitas el plugin de GitHub Actions.

### 3.1 Instalar el plugin en el frontend

```bash
cd packages/app
yarn add @backstage/plugin-github-actions
```

Edita `packages/app/src/components/catalog/EntityPage.tsx` y añade la pestaña de CI:

```typescript
// Importar el componente del plugin
import {
  EntityGithubActionsContent,
  isGithubActionsAvailable,
} from '@backstage/plugin-github-actions';

// Dentro del serviceEntityPage, añadir la pestaña:
const serviceEntityPage = (
  <EntityLayout>
    {/* ... pestañas existentes ... */}

    <EntityLayout.Route
      path="/ci-cd"
      title="CI/CD"
      if={isGithubActionsAvailable}   // solo muestra si tiene la annotation
    >
      <EntityGithubActionsContent />
    </EntityLayout.Route>

  </EntityLayout>
);
```

### 3.2 Configurar el token de GitHub en `app-config.yaml`

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}    # variable de entorno; nunca hardcodear
```

Exporta el token antes de iniciar Backstage:

```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
yarn dev
```

> **Cómo obtener el token:** GitHub → Settings → Developer settings → Personal access tokens → Fine-grained token con permisos `Actions: Read` y `Metadata: Read` sobre el repositorio.

### 3.3 Verificar la integración

1. Ve al catálogo y abre la ficha de `payments-service`.
2. Deberías ver la pestaña **"CI/CD"** (solo aparece si la annotation `github.com/project-slug` está presente).
3. Dentro verás el historial de ejecuciones del workflow con estado, duración y committer.
4. Haz clic en cualquier ejecución para ver los logs **sin salir de Backstage**.

---

## Parte 4 — Simular un fallo y el flujo de diagnóstico (10 min)

Esta parte simula el escenario real que motivó el ejercicio.

### 4.1 Romper intencionalmente un pipeline

Edita `.github/workflows/payments-service.yaml` y añade un paso que falle:

```yaml
      - name: Run tests
        run: |
          echo "💥 Error: no se puede conectar a la base de datos de test"
          exit 1    # ← fuerza el fallo
```

Commitea y pushea:

```bash
git add .
git commit -m "test: simulate payments-service CI failure"
git push origin main
```

### 4.2 Detectar el fallo desde Backstage (sin abrir GitHub)

1. Abre Backstage → Catalog → busca `payments-service`.
2. Ve a la pestaña **"CI/CD"**.
3. Observa que el último run aparece en rojo con estado `failure`.
4. Haz clic en el run fallido y expande el step `Run tests` para ver el mensaje de error.
5. Identifica el committer responsable del fallo.

**Esto es exactamente lo que el CTO pidió:** diagnóstico en < 30 segundos sin salir de Backstage.

### 4.3 Restaurar el pipeline

Revierte el cambio:

```bash
git revert HEAD --no-edit
git push origin main
```

Espera que el nuevo run complete y verifica en Backstage que el estado vuelve a verde.

---

## Parte 5 — Vista del sistema completo (5 min)

1. En Backstage, filtra el catálogo por `Kind: System`.
2. Abre `fintech-rapida-platform`.
3. Ve a la pestaña **"Diagram"** — deberías ver el grafo completo:

```
fintech-rapida-platform
├── api-gateway ──dependsOn──▶ payments-service
│                ──dependsOn──▶ notifications-service
│                ──dependsOn──▶ audit-logger
├── payments-service ──dependsOn──▶ audit-logger
│                    ──dependsOn──▶ postgres-payments (resource)
├── notifications-service ──dependsOn──▶ audit-logger
└── audit-logger ──dependsOn──▶ kafka-audit-topic (resource)
                 ──dependsOn──▶ postgres-audit (resource)
```

4. Haz clic en cada nodo del grafo para navegar a la ficha del componente.

---

## ✅ Criterios de aceptación

Al finalizar el ejercicio debes poder demostrar:

| # | Criterio | Cómo verificarlo |
|---|---|---|
| 1 | Los 4 componentes aparecen en el catálogo con owner y sistema correctos | Catalog → filtrar por System: `fintech-rapida-platform` |
| 2 | Cada componente tiene el link de CI apuntando al workflow correcto | Ficha del componente → pestaña Overview → sección Links |
| 3 | La pestaña CI/CD muestra el historial de ejecuciones de GitHub Actions | Ficha del componente → pestaña CI/CD |
| 4 | Un fallo en el pipeline es visible en Backstage sin abrir GitHub | Simular fallo → verificar en pestaña CI/CD |
| 5 | El grafo de dependencias entre los 4 servicios es correcto | System diagram → todos los nodos y relaciones presentes |
| 6 | Los 3 grupos de equipos (platform, payments, compliance) existen | Catalog → filtrar Kind: Group |

---

## 🧩 Reto adicional (para quien termine antes)

Añade un workflow de **deployment** que se active solo cuando el CI en `main` pasa, y hazlo visible también en Backstage:

```yaml
# .github/workflows/payments-service-deploy.yaml
name: Deploy — payments-service

on:
  workflow_run:
    workflows: ["CI — payments-service"]
    branches: [main]
    types: [completed]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: echo "🚀 Desplegando payments-service a producción..."
      - name: Notify Backstage
        run: echo "📡 En un caso real, aquí notificarías vía webhook"
```

Actualiza la annotation en `catalog-info.yaml` para apuntar ahora al workflow de deploy:

```yaml
annotations:
  backstage.io/ci-url: https://github.com/<usuario>/fintech-rapida/actions/workflows/payments-service-deploy.yaml
```

**Pregunta de reflexión:** ¿Qué pasaría si quisieras mostrar tanto el CI como el deploy en la misma ficha? ¿Cómo lo modelarías con las herramientas que viste hoy?

---

## 📝 Puntos de aprendizaje clave

- La annotation `github.com/project-slug` es el **puente** entre Backstage y GitHub — sin ella, el plugin no funciona.
- La annotation `backstage.io/ci-url` apunta al workflow **específico** del servicio, no a todos los workflows del repo.
- El plugin de GitHub Actions convierte Backstage en el **panel de control centralizado** de CI/CD — el equipo no necesita recordar URLs.
- Modelar `dependsOn` correctamente permite que, cuando falla el `audit-logger`, sepas inmediatamente cuántos servicios están en riesgo (`api-gateway`, `payments-service`, `notifications-service`).
