# SESIÓN 3: El Golden Path con Software Templates

> **Taller práctico guiado — 180 minutos**
> Posgrado en Platform Engineering | Backstage Workshop Series

---

## Índice

1. [Objetivos](#1-objetivos)
2. [Prerrequisitos](#2-prerrequisitos)
3. [Cronograma](#3-cronograma)
4. [Paso 1: Diseñando el Skeleton](#4-paso-1-diseñando-el-skeleton)
5. [Paso 2: Construcción del template.yaml](#5-paso-2-construcción-del-templateyaml)
6. [Paso 3: Registro y Troubleshooting](#6-paso-3-registro-y-troubleshooting)
7. [Reto Avanzado](#7-reto-avanzado)

---

## 1. Objetivos

**Duración estimada: 15 minutos**

Al finalizar esta sesión, los participantes serán capaces de:

- Explicar el concepto de **Golden Path** y su rol en reducir la carga cognitiva de los equipos de desarrollo.
- Identificar los componentes de un **Software Template** en Backstage (skeleton, template.yaml, pasos de acción).
- Diseñar la estructura de archivos de un **Starter Kit** para dos tipos de componentes: Backend (Node.js/Express) y Frontend (SPA estática).
- Construir un `template.yaml` completo con parámetros dinámicos y pasos reales: `fetch:template`, `publish:github` y `catalog:register`.
- Registrar el template en el catálogo de Backstage y depurar errores comunes del Scaffolder.

### ¿Qué es el Golden Path?

El **Golden Path** (Camino Dorado) es el conjunto de herramientas, flujos de trabajo y convenciones **preaprobadas y mantenidas por el equipo de plataforma** que cualquier equipo de desarrollo puede usar para construir, probar y desplegar software de forma segura y estandarizada.

```
Sin Golden Path                         Con Golden Path
─────────────────────────────────────   ───────────────────────────────────────
Cada equipo elige su stack            → Stack predefinido y auditado
Configuración de CI/CD manual         → GitHub Actions listo para usar
Dockerfile escrito desde cero         → Dockerfile optimizado incluido
Onboarding tarda días                 → Onboarding en minutos
Deuda técnica acumulada               → Convenciones aplicadas desde el día 0
```

> **Principio clave:** El Golden Path no prohíbe otras rutas; las hace tan fáciles que el camino correcto es también el camino más rápido.

---

## 2. Prerrequisitos

**Duración estimada: 15 minutos**

Antes de iniciar el laboratorio, verifica que tienes lo siguiente instalado y configurado.

### 2.1 Software requerido

| Herramienta | Versión mínima | Verificación |
|---|---|---|
| Node.js | 18.x LTS | `node --version` |
| npm | 9.x | `npm --version` |
| Git | 2.40+ | `git --version` |
| Docker | 24.x | `docker --version` |
| GitHub CLI | 2.x | `gh --version` |

```bash
# Verificación rápida de todo el entorno
node --version && npm --version && git --version && docker --version && gh --version
```

### 2.2 Cuentas y accesos

- **GitHub**: Cuenta personal con permisos para crear repositorios públicos o en una organización de práctica.
- **GitHub Token**: Token con scopes `repo`, `workflow`, `admin:org` (o `public_repo` si usas cuenta personal).
- **Backstage local**: Instancia de Backstage corriendo en `http://localhost:3000` (configurada en la Sesión 1).

### 2.3 Configurar el token de GitHub en Backstage

Agrega el token a tu `app-config.local.yaml` (nunca al `app-config.yaml` principal):

```yaml
# app-config.local.yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

scaffolder:
  defaultAuthor:
    name: Platform Engineering Bot
    email: platform@tuempresa.com
  defaultCommitMessage: "feat: scaffold from golden-path template"
```

```bash
# Exporta el token en tu shell antes de iniciar Backstage
export GITHUB_TOKEN=ghp_TuTokenAquí
yarn dev
```

### 2.4 Estructura de trabajo esperada

Al terminar este laboratorio, tu repositorio de templates tendrá esta estructura:

```
golden-path-templates/
├── backend-nodejs/
│   ├── template.yaml
│   └── skeleton/
│       ├── package.json
│       ├── src/
│       │   ├── index.js
│       │   └── routes/
│       │       └── health.js
│       ├── Dockerfile
│       ├── .dockerignore
│       ├── .github/
│       │   └── workflows/
│       │       └── ci.yaml
│       └── catalog-info.yaml
└── frontend-spa/
    ├── template.yaml
    └── skeleton/
        ├── package.json
        ├── index.html
        ├── src/
        │   └── main.js
        ├── Dockerfile
        ├── nginx.conf
        ├── .dockerignore
        ├── .github/
        │   └── workflows/
        │       └── ci.yaml
        └── catalog-info.yaml
```

---

## 3. Cronograma

**Duración total: 180 minutos**

```
┌──────────────────────────────────────────────────────────────┐
│  SESIÓN 3: El Golden Path con Software Templates             │
│  Duración total: 3 horas / 180 minutos                       │
├────────┬──────────────────────────────────────────┬──────────┤
│ Tiempo │ Actividad                                │ Tipo     │
├────────┼──────────────────────────────────────────┼──────────┤
│ 00:00  │ § 1. Objetivos y contexto teórico        │ Teoría   │
│        │   - Qué es el Golden Path                │          │
│        │   - Anatomía de un Software Template     │          │
│ 00:15  ├──────────────────────────────────────────┼──────────┤
│        │ § 2. Verificación de prerrequisitos       │ Setup    │
│        │   - Entorno, tokens, Backstage local      │          │
│ 00:30  ├──────────────────────────────────────────┼──────────┤
│        │ § 4. PASO 1: Diseñando el Skeleton        │ Lab      │
│        │   - Skeleton Backend Node.js/Express      │          │
│        │   - Dockerfile + GitHub Actions backend   │          │
│        │   - Skeleton Frontend SPA estática        │          │
│        │   - Dockerfile + nginx + GitHub Actions   │          │
│ 01:15  ├──────────────────────────────────────────┼──────────┤
│        │ § 5. PASO 2: Construcción template.yaml   │ Lab      │
│        │   - Parámetros de entrada del wizard      │          │
│        │   - Pasos: fetch, publish, catalog        │          │
│        │   - Variables dinámicas con Nunjucks      │          │
│ 02:00  ├──────────────────────────────────────────┼──────────┤
│        │ § 6. PASO 3: Registro y Troubleshooting   │ Lab      │
│        │   - Registrar template en Backstage       │          │
│        │   - Ejecutar el Scaffolder                │          │
│        │   - Depurar errores comunes               │          │
│ 02:45  ├──────────────────────────────────────────┼──────────┤
│        │ § 7. Reto Avanzado                        │ Reto     │
│        │   - Agregar step condicional              │          │
│        │   - Template para tercer componente       │          │
│ 03:00  └──────────────────────────────────────────┴──────────┘
```

---

## 4. Paso 1: Diseñando el Skeleton

**Duración estimada: 45 minutos**

El **skeleton** es la plantilla de archivos que el Scaffolder copiará y procesará. Usa la sintaxis de **Nunjucks** para inyectar valores dinámicos del formulario del wizard.

> **Convención de naming:** Usa `${{ values.variableName }}` dentro de los archivos del skeleton para referencias a variables.

### 4.1 Crear la estructura base

```bash
# Crea el directorio raíz para tus templates
mkdir -p golden-path-templates/backend-nodejs/skeleton
mkdir -p golden-path-templates/frontend-spa/skeleton
cd golden-path-templates
```

---

### 4.2 Skeleton: Backend Node.js / Express

#### `backend-nodejs/skeleton/package.json`

```json
{
  "name": "${{ values.component_id }}",
  "version": "1.0.0",
  "description": "${{ values.description }}",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "test": "jest --coverage",
    "lint": "eslint src/"
  },
  "dependencies": {
    "express": "^4.18.2",
    "morgan": "^1.10.0",
    "helmet": "^7.1.0",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "nodemon": "^3.0.2",
    "eslint": "^8.57.0",
    "supertest": "^6.3.4"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

#### `backend-nodejs/skeleton/src/index.js`

```javascript
const express = require('express');
const morgan = require('morgan');
const helmet = require('helmet');
const cors = require('cors');
const healthRouter = require('./routes/health');

const app = express();
const PORT = process.env.PORT || 3000;

app.use(helmet());
app.use(cors());
app.use(morgan('combined'));
app.use(express.json());

app.use('/health', healthRouter);

app.get('/', (req, res) => {
  res.json({
    service: '${{ values.component_id }}',
    owner: '${{ values.owner }}',
    version: '1.0.0',
    status: 'running',
  });
});

app.listen(PORT, () => {
  console.log(`[INFO] ${{ values.component_id }} listening on port ${PORT}`);
});

module.exports = app;
```

#### `backend-nodejs/skeleton/src/routes/health.js`

```javascript
const express = require('express');
const router = express.Router();

router.get('/', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  });
});

module.exports = router;
```

#### `backend-nodejs/skeleton/Dockerfile`

```dockerfile
# ── Stage 1: dependencies ──────────────────────────────────────
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# ── Stage 2: runtime ───────────────────────────────────────────
FROM node:18-alpine AS runtime
WORKDIR /app

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=deps /app/node_modules ./node_modules
COPY src/ ./src/
COPY package.json ./

USER appuser

EXPOSE 3000
ENV NODE_ENV=production

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "src/index.js"]
```

#### `backend-nodejs/skeleton/.dockerignore`

```
node_modules
npm-debug.log
.env
.env.*
coverage/
.git
*.md
```

#### `backend-nodejs/skeleton/.github/workflows/ci.yaml`

```yaml
name: CI — ${{ values.component_id }}

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: ${{ values.component_id }}
  REGISTRY: ghcr.io
  IMAGE_FULL: ghcr.io/${{ "${{ github.repository_owner }}" }}/${{ values.component_id }}

jobs:
  test:
    name: Test & Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ "${{ env.REGISTRY }}" }}
          username: ${{ "${{ github.actor }}" }}
          password: ${{ "${{ secrets.GITHUB_TOKEN }}" }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ "${{ env.IMAGE_FULL }}" }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}
            latest

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ "${{ steps.meta.outputs.tags }}" }}
          labels: ${{ "${{ steps.meta.outputs.labels }}" }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

#### `backend-nodejs/skeleton/catalog-info.yaml`

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.component_id }}
  description: ${{ values.description }}
  annotations:
    github.com/project-slug: ${{ values.destination.owner }}/${{ values.destination.repo }}
    backstage.io/techdocs-ref: dir:.
  tags:
    - nodejs
    - express
    - backend
    - ${{ values.environment }}
spec:
  type: service
  lifecycle: ${{ values.lifecycle }}
  owner: ${{ values.owner }}
  system: ${{ values.system | default("default") }}
```

---

### 4.3 Skeleton: Frontend SPA Estática

#### `frontend-spa/skeleton/package.json`

```json
{
  "name": "${{ values.component_id }}",
  "version": "1.0.0",
  "description": "${{ values.description }}",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "lint": "eslint src/"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.1",
    "vite": "^5.0.10",
    "vitest": "^1.2.2",
    "eslint": "^8.57.0",
    "eslint-plugin-react": "^7.33.2"
  }
}
```

#### `frontend-spa/skeleton/index.html`

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>${{ values.component_id }}</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

#### `frontend-spa/skeleton/src/main.js`

```javascript
import React from 'react';
import ReactDOM from 'react-dom/client';

function App() {
  return (
    React.createElement('div', { style: { fontFamily: 'sans-serif', padding: '2rem' } },
      React.createElement('h1', null, '${{ values.component_id }}'),
      React.createElement('p', null, '${{ values.description }}'),
      React.createElement('p', null, `Owner: ${{ values.owner }}`),
    )
  );
}

ReactDOM.createRoot(document.getElementById('root')).render(
  React.createElement(React.StrictMode, null, React.createElement(App))
);
```

#### `frontend-spa/skeleton/nginx.conf`

```nginx
server {
    listen       80;
    server_name  localhost;
    root         /usr/share/nginx/html;
    index        index.html;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy "strict-origin-when-cross-origin";

    # SPA fallback — todas las rutas sirven index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache de assets estáticos
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Health check endpoint
    location /health {
        return 200 '{"status":"healthy"}';
        add_header Content-Type application/json;
    }

    error_page 404 /index.html;
}
```

#### `frontend-spa/skeleton/Dockerfile`

```dockerfile
# ── Stage 1: build ─────────────────────────────────────────────
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ── Stage 2: serve with nginx ──────────────────────────────────
FROM nginx:1.25-alpine AS runtime

# Remove default nginx config
RUN rm /etc/nginx/conf.d/default.conf

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/app.conf

# Non-root nginx
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    chown -R nginx:nginx /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid

USER nginx

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:80/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

#### `frontend-spa/skeleton/.dockerignore`

```
node_modules
dist
.env
.env.*
.git
*.md
coverage/
```

#### `frontend-spa/skeleton/.github/workflows/ci.yaml`

```yaml
name: CI — ${{ values.component_id }}

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_FULL: ghcr.io/${{ "${{ github.repository_owner }}" }}/${{ values.component_id }}

jobs:
  test:
    name: Test & Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ "${{ env.REGISTRY }}" }}
          username: ${{ "${{ github.actor }}" }}
          password: ${{ "${{ secrets.GITHUB_TOKEN }}" }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ "${{ env.IMAGE_FULL }}" }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            latest

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ "${{ steps.meta.outputs.tags }}" }}
          labels: ${{ "${{ steps.meta.outputs.labels }}" }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  lighthouse:
    name: Lighthouse CI
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v11
        with:
          uploadArtifacts: true
          temporaryPublicStorage: true
```

#### `frontend-spa/skeleton/catalog-info.yaml`

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.component_id }}
  description: ${{ values.description }}
  annotations:
    github.com/project-slug: ${{ values.destination.owner }}/${{ values.destination.repo }}
    backstage.io/techdocs-ref: dir:.
  tags:
    - react
    - frontend
    - spa
    - ${{ values.environment }}
spec:
  type: website
  lifecycle: ${{ values.lifecycle }}
  owner: ${{ values.owner }}
  system: ${{ values.system | default("default") }}
```

---

## 5. Paso 2: Construcción del template.yaml

**Duración estimada: 45 minutos**

El `template.yaml` es el corazón del Software Template. Define el wizard de parámetros y la secuencia de pasos que ejecuta el Scaffolder.

### 5.1 Anatomía de un template.yaml

```
template.yaml
│
├── metadata          → Nombre, descripción, tags del template en el catálogo
├── spec.parameters   → Secciones del wizard (formulario de entrada)
└── spec.steps        → Acciones que ejecuta el Scaffolder:
    ├── fetch:template    → Copia y procesa el skeleton
    ├── publish:github    → Crea el repositorio en GitHub
    └── catalog:register  → Registra el componente en el catálogo
```

### 5.2 template.yaml — Backend Node.js/Express

Crea el archivo `golden-path-templates/backend-nodejs/template.yaml`:

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: golden-path-backend-nodejs
  title: "Golden Path: Backend Node.js / Express"
  description: >
    Crea un microservicio Node.js con Express, Dockerfile multi-stage,
    GitHub Actions (CI/CD) y registro automático en el catálogo de Backstage.
    Incluye helmet, cors, morgan y endpoint /health listo para producción.
  tags:
    - nodejs
    - express
    - backend
    - golden-path
    - recommended
  annotations:
    backstage.io/managed-by-location: "url:https://github.com/TU_ORG/golden-path-templates/blob/main/backend-nodejs/template.yaml"

spec:
  owner: platform-engineering
  type: service

  # ─────────────────────────────────────────────────────────────
  # PARÁMETROS — Wizard de entrada en la UI de Backstage
  # ─────────────────────────────────────────────────────────────
  parameters:
    # ── Sección 1: Identidad del componente ───────────────────
    - title: "1 / 3 — Identidad del servicio"
      required:
        - component_id
        - description
        - owner
      properties:
        component_id:
          title: Nombre del servicio
          type: string
          description: >
            Identificador único (slug). Solo minúsculas, números y guiones.
            Ejemplo: payments-api, user-service, order-processor.
          pattern: "^[a-z0-9-]+$"
          maxLength: 63
          ui:autofocus: true
          ui:field: EntityNamePicker

        description:
          title: Descripción
          type: string
          description: Describe brevemente qué hace este servicio.
          maxLength: 200

        owner:
          title: Equipo propietario
          type: string
          description: Equipo responsable del servicio en Backstage.
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group

        system:
          title: Sistema (opcional)
          type: string
          description: Sistema al que pertenece este servicio.
          ui:field: EntityPicker
          ui:options:
            allowedKinds:
              - System

    # ── Sección 2: Configuración técnica ─────────────────────
    - title: "2 / 3 — Configuración técnica"
      required:
        - lifecycle
        - environment
        - port
      properties:
        lifecycle:
          title: Ciclo de vida
          type: string
          description: Estado actual del componente en producción.
          default: experimental
          enum:
            - experimental
            - production
            - deprecated
          enumNames:
            - "Experimental (en desarrollo)"
            - "Producción (estable)"
            - "Deprecado (en retiro)"

        environment:
          title: Ambiente principal
          type: string
          default: dev
          enum:
            - dev
            - staging
            - production
          enumNames:
            - "Desarrollo"
            - "Staging"
            - "Producción"

        port:
          title: Puerto de la aplicación
          type: integer
          default: 3000
          minimum: 1024
          maximum: 65535
          description: Puerto en el que escuchará el servidor Express.

        enable_db:
          title: ¿Incluir integración con base de datos?
          type: boolean
          default: false
          description: Agrega dependencias de pg (PostgreSQL) y variables de entorno de conexión.

    # ── Sección 3: Repositorio destino ────────────────────────
    - title: "3 / 3 — Repositorio en GitHub"
      required:
        - destination
      properties:
        destination:
          title: Ubicación del repositorio
          type: object
          description: >
            Organización y nombre del repositorio que se creará en GitHub.
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com

        repo_visibility:
          title: Visibilidad del repositorio
          type: string
          default: private
          enum:
            - private
            - public
            - internal
          enumNames:
            - "Privado"
            - "Público"
            - "Interno (solo organización)"

        default_branch:
          title: Rama principal
          type: string
          default: main
          enum:
            - main
            - master
            - develop

  # ─────────────────────────────────────────────────────────────
  # PASOS — Secuencia de acciones del Scaffolder
  # ─────────────────────────────────────────────────────────────
  steps:
    # ── Paso 1: Procesar el skeleton ──────────────────────────
    - id: fetch-template
      name: Generar archivos desde el skeleton
      action: fetch:template
      input:
        url: ./skeleton
        values:
          component_id: ${{ parameters.component_id }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          system: ${{ parameters.system }}
          lifecycle: ${{ parameters.lifecycle }}
          environment: ${{ parameters.environment }}
          port: ${{ parameters.port }}
          enable_db: ${{ parameters.enable_db }}
          destination: ${{ parameters.destination | parseRepoUrl }}

    # ── Paso 2 (condicional): Agregar dependencia de DB ───────
    - id: add-db-deps
      name: Agregar dependencias de PostgreSQL
      action: fetch:plain:file
      if: ${{ parameters.enable_db }}
      input:
        url: ./extras/db-config.js
        targetPath: src/config/db.js

    # ── Paso 3: Crear repositorio en GitHub ───────────────────
    - id: publish-github
      name: Publicar en GitHub
      action: publish:github
      input:
        allowedHosts:
          - github.com
        description: ${{ parameters.description }}
        repoUrl: ${{ parameters.destination }}
        repoVisibility: ${{ parameters.repo_visibility }}
        defaultBranch: ${{ parameters.default_branch }}
        gitCommitMessage: "feat: initial scaffold from golden-path-backend-nodejs"
        gitAuthorName: "Platform Engineering Bot"
        gitAuthorEmail: "platform@tuempresa.com"
        topics:
          - nodejs
          - express
          - golden-path
          - ${{ parameters.environment }}
        requireCodeOwnerReviews: true
        deleteBranchOnMerge: true
        allowMergeCommit: false
        allowSquashMerge: true
        allowRebaseMerge: false
        hasProjects: false
        hasWiki: false

    # ── Paso 4: Registrar en el catálogo de Backstage ─────────
    - id: catalog-register
      name: Registrar en el catálogo
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish-github'].output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  # ─────────────────────────────────────────────────────────────
  # OUTPUT — Links que se muestran al finalizar
  # ─────────────────────────────────────────────────────────────
  output:
    links:
      - title: Repositorio en GitHub
        url: ${{ steps['publish-github'].output.remoteUrl }}
        icon: github

      - title: Ver en el catálogo de Backstage
        url: ${{ steps['catalog-register'].output.entityRef }}
        icon: catalog

      - title: Pipeline de CI/CD
        url: ${{ steps['publish-github'].output.remoteUrl }}/actions
        icon: dashboard
```

---

### 5.3 template.yaml — Frontend SPA Estática

Crea el archivo `golden-path-templates/frontend-spa/template.yaml`:

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: golden-path-frontend-spa
  title: "Golden Path: Frontend SPA (React + Vite)"
  description: >
    Crea una Single Page Application con React + Vite, Dockerfile multi-stage
    con nginx, GitHub Actions (CI/CD + Lighthouse) y registro automático
    en el catálogo de Backstage.
  tags:
    - react
    - vite
    - frontend
    - spa
    - golden-path
    - recommended
  annotations:
    backstage.io/managed-by-location: "url:https://github.com/TU_ORG/golden-path-templates/blob/main/frontend-spa/template.yaml"

spec:
  owner: platform-engineering
  type: website

  parameters:
    # ── Sección 1: Identidad ──────────────────────────────────
    - title: "1 / 3 — Identidad de la aplicación"
      required:
        - component_id
        - description
        - owner
      properties:
        component_id:
          title: Nombre de la aplicación
          type: string
          description: >
            Identificador único (slug). Solo minúsculas, números y guiones.
            Ejemplo: admin-portal, checkout-ui, dashboard-app.
          pattern: "^[a-z0-9-]+$"
          maxLength: 63
          ui:autofocus: true
          ui:field: EntityNamePicker

        description:
          title: Descripción
          type: string
          description: Describe brevemente qué hace esta aplicación frontend.
          maxLength: 200

        owner:
          title: Equipo propietario
          type: string
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group

        system:
          title: Sistema (opcional)
          type: string
          ui:field: EntityPicker
          ui:options:
            allowedKinds:
              - System

        backend_service:
          title: Servicio backend relacionado (opcional)
          type: string
          description: Backstage entity ref del API que consume esta SPA.
          ui:field: EntityPicker
          ui:options:
            allowedKinds:
              - Component

    # ── Sección 2: Configuración técnica ─────────────────────
    - title: "2 / 3 — Configuración técnica"
      required:
        - lifecycle
        - environment
      properties:
        lifecycle:
          title: Ciclo de vida
          type: string
          default: experimental
          enum:
            - experimental
            - production
            - deprecated
          enumNames:
            - "Experimental"
            - "Producción"
            - "Deprecado"

        environment:
          title: Ambiente principal
          type: string
          default: dev
          enum:
            - dev
            - staging
            - production

        enable_lighthouse:
          title: ¿Habilitar Lighthouse CI?
          type: boolean
          default: true
          description: Ejecuta auditoría de performance, accesibilidad y SEO en cada merge a main.

        base_api_url:
          title: URL base del API (variable de entorno)
          type: string
          description: >
            Valor por defecto para VITE_API_BASE_URL.
            Ejemplo: https://api.tuempresa.com/v1
          default: "http://localhost:3000"

    # ── Sección 3: Repositorio destino ────────────────────────
    - title: "3 / 3 — Repositorio en GitHub"
      required:
        - destination
      properties:
        destination:
          title: Ubicación del repositorio
          type: object
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com

        repo_visibility:
          title: Visibilidad del repositorio
          type: string
          default: private
          enum:
            - private
            - public
            - internal

        default_branch:
          title: Rama principal
          type: string
          default: main
          enum:
            - main
            - master
            - develop

  steps:
    # ── Paso 1: Procesar skeleton ─────────────────────────────
    - id: fetch-template
      name: Generar archivos desde el skeleton
      action: fetch:template
      input:
        url: ./skeleton
        values:
          component_id: ${{ parameters.component_id }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          system: ${{ parameters.system }}
          lifecycle: ${{ parameters.lifecycle }}
          environment: ${{ parameters.environment }}
          base_api_url: ${{ parameters.base_api_url }}
          enable_lighthouse: ${{ parameters.enable_lighthouse }}
          backend_service: ${{ parameters.backend_service }}
          destination: ${{ parameters.destination | parseRepoUrl }}

    # ── Paso 2: Publicar en GitHub ────────────────────────────
    - id: publish-github
      name: Publicar en GitHub
      action: publish:github
      input:
        allowedHosts:
          - github.com
        description: ${{ parameters.description }}
        repoUrl: ${{ parameters.destination }}
        repoVisibility: ${{ parameters.repo_visibility }}
        defaultBranch: ${{ parameters.default_branch }}
        gitCommitMessage: "feat: initial scaffold from golden-path-frontend-spa"
        gitAuthorName: "Platform Engineering Bot"
        gitAuthorEmail: "platform@tuempresa.com"
        topics:
          - react
          - vite
          - frontend
          - golden-path

    # ── Paso 3: Registrar en el catálogo ─────────────────────
    - id: catalog-register
      name: Registrar en el catálogo
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish-github'].output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - title: Repositorio en GitHub
        url: ${{ steps['publish-github'].output.remoteUrl }}
        icon: github

      - title: Ver en el catálogo de Backstage
        url: ${{ steps['catalog-register'].output.entityRef }}
        icon: catalog

      - title: Pipeline de CI/CD
        url: ${{ steps['publish-github'].output.remoteUrl }}/actions
        icon: dashboard
```

### 5.4 Referencia rápida: Variables dinámicas en Nunjucks

| Sintaxis | Descripción | Ejemplo |
|---|---|---|
| `${{ values.nombre }}` | Valor simple de parámetro | `${{ values.component_id }}` |
| `${{ values.nombre \| upper }}` | Filtro: mayúsculas | `${{ values.component_id \| upper }}` |
| `${{ values.nombre \| default("val") }}` | Valor por defecto | `${{ values.system \| default("default") }}` |
| `${{ values.nombre \| replace("-","_") }}` | Reemplazar caracteres | `${{ values.component_id \| replace("-","_") }}` |
| `${{ values.flag \| string }}` | Convertir a string | `${{ values.enable_db \| string }}` |
| `{% if values.flag %}...{% endif %}` | Bloque condicional | Ver ejemplo abajo |

**Ejemplo de bloque condicional en un archivo del skeleton:**

```javascript
// src/config/index.js
const config = {
  port: ${{ values.port }},
  env: '${{ values.environment }}',
{% if values.enable_db %}
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '5432'),
    name: process.env.DB_NAME || '${{ values.component_id | replace("-", "_") }}_db',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD,
  },
{% endif %}
};

module.exports = config;
```

---

## 6. Paso 3: Registro y Troubleshooting

**Duración estimada: 45 minutos**

### 6.1 Publicar el repositorio de templates en GitHub

```bash
cd golden-path-templates

# Inicializa git si aún no lo has hecho
git init
git add .
git commit -m "feat: initial golden path templates for backend and frontend"

# Crea el repositorio en GitHub y sube el código
gh repo create TU_ORG/golden-path-templates \
  --private \
  --description "Golden Path Software Templates para el catálogo de Backstage" \
  --push \
  --source .
```

### 6.2 Registrar el template en Backstage — Método A: app-config.yaml

Agrega la ubicación directamente en `app-config.yaml` para que cargue al inicio:

```yaml
# app-config.yaml
catalog:
  locations:
    # Templates del Golden Path
    - type: url
      target: https://github.com/TU_ORG/golden-path-templates/blob/main/backend-nodejs/template.yaml
      rules:
        - allow: [Template]

    - type: url
      target: https://github.com/TU_ORG/golden-path-templates/blob/main/frontend-spa/template.yaml
      rules:
        - allow: [Template]
```

### 6.3 Registrar el template en Backstage — Método B: UI manual

1. Abre `http://localhost:3000`
2. Ve al menú lateral → **Catalog** → botón **Register Existing Component**
3. Pega la URL raw del template:
   ```
   https://github.com/TU_ORG/golden-path-templates/blob/main/backend-nodejs/template.yaml
   ```
4. Haz clic en **Analyze** → **Import**
5. Repite para `frontend-spa/template.yaml`

### 6.4 Ejecutar el Scaffolder

1. Ve a **Create** (icono de varita mágica) en el menú lateral
2. Busca "Golden Path" en el buscador de templates
3. Selecciona **"Golden Path: Backend Node.js / Express"**
4. Completa el wizard (3 pasos)
5. Haz clic en **Review** y luego **Create**
6. Observa la ejecución en tiempo real de los 3 pasos

### 6.5 Troubleshooting: Errores más comunes

#### Error 1: Template no aparece en la UI

```
Síntoma: El template no aparece en /create
```

```bash
# Diagnóstico: revisa los logs del backend de Backstage
# En la terminal donde corre yarn dev, busca:
# [catalog] Failed to read location

# Posibles causas y soluciones:
# 1. URL incorrecta — verifica que apunta al archivo RAW o al blob de GitHub
# 2. Repositorio privado sin token — verifica GITHUB_TOKEN en app-config.local.yaml
# 3. Sintaxis YAML inválida — valida con:
npx js-yaml backend-nodejs/template.yaml
```

#### Error 2: `fetch:template` falla con "not found"

```
Error: Failed to resolve template content: ENOENT: no such file or directory './skeleton'
```

```yaml
# Causa: la ruta en el template.yaml es relativa al template.yaml mismo
# Solución: verifica que la carpeta skeleton esté al mismo nivel que template.yaml

golden-path-templates/
└── backend-nodejs/
    ├── template.yaml   ← aquí
    └── skeleton/       ← ./skeleton es correcto
```

#### Error 3: `publish:github` falla — 403 Forbidden

```
Error: HttpError: Resource not accessible by integration
```

```bash
# Solución: el token no tiene los permisos necesarios
# Genera un nuevo token con estos scopes:
#   - repo (completo)
#   - workflow
#   - admin:org (si publicas en una org)
#   - delete_repo (opcional, para limpiar tests)

# Verifica el token actual:
gh auth status

# Re-autentícate si es necesario:
gh auth login
```

#### Error 4: `catalog:register` falla — Entity already exists

```
Error: Conflict: Entity component:default/mi-servicio already exists
```

```bash
# El componente ya está registrado en el catálogo.
# Opciones:
# 1. Usa un nombre diferente en el wizard
# 2. Elimina la entidad existente desde la UI:
#    Catalog → selecciona la entidad → ⋮ menú → Unregister entity
# 3. Si estás re-scaffolding en desarrollo, elimina el repo de GitHub y vuelve a intentarlo
gh repo delete TU_ORG/nombre-del-repo --yes
```

#### Error 5: Variables no se sustituyen en el skeleton

```
Síntoma: El archivo generado contiene literalmente "${{ values.component_id }}" en lugar del valor
```

```yaml
# Causa: el archivo no fue procesado por Nunjucks
# Solución: verifica que el archivo está DENTRO de la carpeta /skeleton
# y que no tiene la extensión .plain o configuración de exclusión

# En el step fetch:template puedes excluir archivos del procesado:
- id: fetch-template
  action: fetch:template
  input:
    url: ./skeleton
    # Excluir archivos binarios o que no deben procesarse:
    copyWithoutTemplating:
      - "**/*.png"
      - "**/*.ico"
      - "**/vendor/**"
    values:
      component_id: ${{ parameters.component_id }}
```

#### Error 6: Caracteres especiales en Nunjucks

```
Error: Template render error: unexpected token
Síntoma: GitHub Actions CI falla porque ${{ ... }} es ambiguo entre Nunjucks y GHA
```

```yaml
# Problema: GitHub Actions también usa ${{ }} para sus expresiones
# Solución aplicada en este lab: escapar las expresiones de GHA

# En el skeleton de .github/workflows/ci.yaml, escribe:
${{ "${{ github.actor }}" }}
# En vez de:
${{ github.actor }}

# Esto le dice a Nunjucks: "no toques esto, es una expresión de GHA"
```

### 6.6 Validación final del laboratorio

Ejecuta este checklist antes de cerrar el Paso 3:

```bash
# ✅ 1. Ambos templates aparecen en http://localhost:3000/create
# ✅ 2. El wizard tiene 3 pasos y valida los campos correctamente
# ✅ 3. El repositorio fue creado en GitHub con todos los archivos
# ✅ 4. El catalog-info.yaml tiene los valores correctos (no variables sin sustituir)
# ✅ 5. El componente aparece en el catálogo de Backstage
# ✅ 6. La GitHub Action se ejecutó correctamente en el nuevo repo

# Verifica el catálogo por CLI:
curl -s http://localhost:7007/api/catalog/entities?filter=kind=Component \
  | jq '.[].metadata.name'
```

---

## 7. Reto Avanzado

**Duración estimada: 15 minutos**

Estos retos están pensados para quienes terminaron los pasos principales y quieren profundizar.

### Reto A: Agregar un step condicional real

Modifica el template de backend para que, si el usuario activa `enable_db`, se agregue un archivo de migración base con `fetch:plain:file`:

```yaml
# En template.yaml, agrega después del step fetch-template:
- id: add-db-migration
  name: Agregar migración inicial de base de datos
  action: fetch:plain
  if: ${{ parameters.enable_db }}
  input:
    url: ./extras/migrations/
    targetPath: migrations/

# Crea la carpeta extras/migrations/ en tu template con:
# 001_initial_schema.sql
# knexfile.js (o el ORM de tu preferencia)
```

### Reto B: Tercer template — Worker / Job en background

Diseña la estructura de un tercer tipo de componente: un **worker de procesamiento en background** (BullMQ + Redis). Debe incluir:

- `skeleton/` con estructura base de BullMQ
- `template.yaml` con parámetros para `queue_name` y `concurrency`
- `Dockerfile` optimizado
- GitHub Action con step de smoke test

**Estructura sugerida:**

```
background-worker/
├── template.yaml
└── skeleton/
    ├── package.json          # bullmq, ioredis
    ├── src/
    │   ├── worker.js         # Worker principal con ${{ values.queue_name }}
    │   ├── processor.js      # Lógica del job
    │   └── config.js         # Redis config desde env vars
    ├── Dockerfile
    └── .github/
        └── workflows/
            └── ci.yaml
```

### Reto C: Validación avanzada de parámetros

Agrega validación en el `template.yaml` para que el nombre del componente no colisione con nombres reservados:

```yaml
component_id:
  title: Nombre del servicio
  type: string
  pattern: "^(?!backstage|platform|infra|system)[a-z0-9-]+$"
  description: >
    El nombre no puede comenzar con: backstage, platform, infra, system.
    Solo minúsculas, números y guiones.
  ui:help: >
    Nombres reservados no permitidos: backstage-*, platform-*, infra-*, system-*
```

### Reto D: Output personalizado con resumen del scaffold

Agrega un output de tipo `text` que muestre un resumen de lo que se creó:

```yaml
output:
  text:
    - title: Resumen del scaffold
      content: |
        ## ¡Componente creado exitosamente!

        | Campo | Valor |
        |-------|-------|
        | Nombre | `${{ parameters.component_id }}` |
        | Equipo | `${{ parameters.owner }}` |
        | Lifecycle | `${{ parameters.lifecycle }}` |
        | Repositorio | [${{ parameters.destination }}](${{ steps['publish-github'].output.remoteUrl }}) |

        ### Próximos pasos
        1. Clona el repositorio: `git clone ${{ steps['publish-github'].output.remoteUrl }}`
        2. Instala dependencias: `npm install`
        3. Inicia en desarrollo: `npm run dev`
        4. El pipeline de CI/CD se activará automáticamente en cada push a `main`.

  links:
    - title: Repositorio en GitHub
      url: ${{ steps['publish-github'].output.remoteUrl }}
      icon: github
    - title: Ver en el catálogo
      url: ${{ steps['catalog-register'].output.entityRef }}
      icon: catalog
```

---

## Referencias y recursos adicionales

| Recurso | URL |
|---|---|
| Backstage Software Templates Docs | https://backstage.io/docs/features/software-templates/ |
| Referencia de acciones del Scaffolder | https://backstage.io/docs/features/software-templates/builtin-actions |
| Nunjucks templating | https://mozilla.github.io/nunjucks/templating.html |
| GitHub Actions: docker/build-push-action | https://github.com/docker/build-push-action |
| CNCF Platform Engineering Whitepaper | https://tag-app-delivery.cncf.io/whitepapers/platforms/ |

---

> **Fin de la Sesión 3** — El Golden Path con Software Templates
>
> En la próxima sesión exploraremos **TechDocs**: cómo generar documentación técnica automática directamente desde el código usando MkDocs integrado con Backstage.
