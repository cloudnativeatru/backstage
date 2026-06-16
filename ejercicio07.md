# 🧩 Tema 1 – Kubernetes Plugin en Backstage
## Visualización del estado de Pods y Deployments (sin kubectl)

> **Curso:** Backstage para Ingeniería de Plataformas  
> **Nivel:** Postgrado
> **Prerequisito:** Backstage corriendo en local (`http://localhost:3000`)

---

## 🎯 Objetivo del ejercicio

Integrar el **Kubernetes Plugin** de Backstage con un cluster real de **AKS (Azure Kubernetes Service)** para visualizar, directamente desde el portal, el estado de Pods y Deployments de una aplicación registrada en el Software Catalog — sin necesidad de abrir una terminal ni ejecutar `kubectl`.

---

## 🏗️ Arquitectura del ejercicio

```
┌─────────────────────────────────────────────────────────┐
│                    BACKSTAGE (local)                    │
│                                                         │
│   Software Catalog                                      │
│   └── my-app (Component)                               │
│        └── @backstage/plugin-kubernetes  ──────────────┼──► AKS Cluster (Azure)
│            muestra: Pods / Deployments / RS             │      └── namespace: production
└─────────────────────────────────────────────────────────┘
```

---

## 📋 Prerrequisitos

| Herramienta | Versión mínima | Verificación |
|---|---|---|
| Node.js | 18.x | `node -v` |
| Yarn | 1.22.x | `yarn -v` |
| Azure CLI | 2.50+ | `az --version` |
| kubectl | 1.27+ | `kubectl version --client` |
| Backstage App | corriendo en local | `http://localhost:3000` |

> ✅ **Cuenta de Azure**: necesitas acceso con permisos para crear recursos (Contributor o superior en una suscripción).

---

## 📦 Parte 1 – Crear el cluster AKS en Azure

### 1.1 Login y configuración inicial

```bash
# Autenticarse en Azure
az login

# Verificar la suscripción activa
az account show --query "{name:name, id:id}" -o table

# (Opcional) Cambiar de suscripción si es necesario
az account set --subscription "<SUBSCRIPTION_ID>"
```

### 1.2 Crear el Resource Group

```bash
az group create \
  --name rg-backstage-lab \
  --location eastus
```

### 1.3 Crear el cluster AKS

```bash
az aks create \
  --resource-group rg-backstage-lab \
  --name aks-backstage-lab \
  --node-count 1 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --kubernetes-version 1.34.8 \
  --network-plugin kubenet \
  --tags "curso=backstage" "entorno=lab"
```

> ⏳ La creación tarda aproximadamente **5–8 minutos**.

### 1.4 Obtener credenciales del cluster

```bash
az aks get-credentials \
  --resource-group rg-backstage-lab \
  --name aks-backstage-lab \
  --overwrite-existing

# Verificar conexión
kubectl get nodes
```

Deberías ver algo así:

```
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-XXXXXXXX-vmss000000   Ready    agent   2m    v1.29.x
```

---

## 🚀 Parte 2 – Desplegar una aplicación de prueba en AKS

### 2.1 Crear el namespace

```bash
kubectl create namespace production
```

### 2.2 Aplicar el manifiesto de la aplicación

Crea el archivo `k8s/my-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    backstage.io/kubernetes-id: my-app   # ← etiqueta clave para Backstage
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        backstage.io/kubernetes-id: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  namespace: production
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f k8s/my-app.yaml

# Verificar que los pods están corriendo
kubectl get pods -n production
```

---

## 🔌 Parte 3 – Instalar el Kubernetes Plugin en Backstage v1.52

### 3.1 Instalar los paquetes

Desde la raíz de tu proyecto Backstage:

```bash
# Plugin de frontend
yarn --cwd packages/app add @backstage/plugin-kubernetes

# Plugin de backend
yarn --cwd packages/backend add @backstage/plugin-kubernetes-backend
```

### 3.2 Configurar el backend

Edita `packages/backend/src/index.ts`:

```typescript
import { createBackend } from '@backstage/backend-defaults';

const backend = createBackend();

// ... otros plugins existentes ...
backend.add(import('@backstage/plugin-kubernetes-backend'));

backend.start();
```

### 3.3 Configurar el frontend

Tu proyecto usa el **nuevo Frontend System** de Backstage (`@backstage/frontend-plugin-api`). La tab de Kubernetes se agrega como un plugin/feature directamente en `packages/app/src/App.tsx`.

Primero instala el paquete (sin `/alpha` — ese sufijo es solo para los imports):

```bash
yarn --cwd packages/app add @backstage/plugin-kubernetes
```

Luego edita `packages/app/src/App.tsx`:

```tsx
import { createApp } from '@backstage/frontend-defaults';
import catalogPlugin from '@backstage/plugin-catalog/alpha';
import { navModule } from './modules/nav';
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPageBlueprint } from '@backstage/plugin-app-react';
import { SignInPage } from '@backstage/core-components';
import { createFrontendModule } from '@backstage/frontend-plugin-api';
import githubActionsPlugin from '@backstage-community/plugin-github-actions/alpha';
import apiDocsPlugin from '@backstage/plugin-api-docs/alpha';
import kubernetesPlugin from '@backstage/plugin-kubernetes/alpha'; // ← agrega esta línea

const signInPage = SignInPageBlueprint.make({
  params: {
    loader: async () => props =>
      (
        <SignInPage
          {...props}
          provider={{
            id: 'github-auth-provider',
            title: 'GitHub',
            message: 'Inicia sesión con tu cuenta de GitHub',
            apiRef: githubAuthApiRef,
          }}
        />
      ),
  },
});

export default createApp({
  features: [
    catalogPlugin,
    navModule,
    githubActionsPlugin,
    apiDocsPlugin,
    kubernetesPlugin,       // ← agrega esta línea
    createFrontendModule({
      pluginId: 'app',
      extensions: [signInPage],
    }),
  ],
});
```

> 💡 El plugin en modo `/alpha` registra automáticamente la tab **Kubernetes** en las entidades del catálogo que tengan las annotations correctas. No necesitas configurar `EntityLayout` manualmente.

---

## ⚙️ Parte 4 – Configurar `app-config.yaml`

### 4.1 Obtener el Service Account Token para Backstage

```bash
# Crear Service Account con permisos de lectura
kubectl create serviceaccount backstage-sa -n production

# Crear ClusterRoleBinding (permisos de solo lectura)
kubectl create clusterrolebinding backstage-sa-binding \
  --clusterrole=view \
  --serviceaccount=production:backstage-sa

# Obtener el token (Kubernetes 1.24+)
kubectl create token backstage-sa -n production --duration=87600h
```

Copia el token generado.

### 4.2 Obtener el endpoint del cluster

```bash
kubectl cluster-info | grep "Kubernetes control plane"
# Ejemplo: https://aks-backstage-lab-XXXXX.hcp.eastus.azmk8s.io:443

# Obtener el certificado CA (en base64)
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'
```

### 4.3 Editar `app-config.yaml`

```yaml
kubernetes:
  serviceLocatorMethod:
    type: 'multiTenant'
  clusterLocatorMethods:
    - type: 'config'
      clusters:
        - url: https://aks-backstage-lab-XXXXX.hcp.eastus.azmk8s.io:443
          name: aks-backstage-lab
          authProvider: 'serviceAccount'
          skipTLSVerify: false
          skipMetricsLookup: true
          serviceAccountToken: ${KUBE_TOKEN}
          caData: ${KUBE_CA_DATA}
```

Crea un archivo `.env` o exporta las variables:

```bash
export KUBE_TOKEN="<token-copiado-del-paso-4.1>"
export KUBE_CA_DATA="<base64-del-ca-certificado>"
```

---

## 📄 Parte 5 – Registrar el componente en el Software Catalog

Crea el archivo `catalog/my-app.yaml` en tu repositorio:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: my-app
  description: Aplicación de prueba desplegada en AKS
  annotations:
    backstage.io/kubernetes-id: my-app                      # ← debe coincidir con la label del pod
    backstage.io/kubernetes-namespace: production           # ← namespace donde están los pods
    backstage.io/kubernetes-cluster: aks-backstage-lab     # ← nombre del cluster en app-config.yaml
spec:
  type: service
  lifecycle: experimental
  owner: group:default/platform-team
```

Registra el componente desde Backstage:

1. Ve a `http://localhost:3000/catalog-import`
2. Ingresa la URL del archivo YAML (o usa una ruta local durante el lab)
3. Haz clic en **Analyze** → **Import**

---

## ▶️ Parte 6 – Levantar Backstage y verificar

```bash
# Desde la raíz del proyecto
yarn dev
```

1. Abre `http://localhost:3000`
2. Ve a **Catalog** → busca **my-app**
3. Haz clic en la pestaña **Kubernetes**
4. Deberías ver los Pods y Deployments del namespace `production`

### ✅ Resultado esperado

```
Your Clusters
─────────────────────────────────────────────────────────────
aks-backstage-lab                          ✅ 2 pods
Cluster                                    ✅ No pods with errors

  my-app
  Deployment  namespace: production        ✅ 2 pods
                                           ✅ No pods with errors

  NAME                          PHASE    STATUS  CONTAINERS  RESTARTS
  my-app-c58d9696b-chkvz        Running  ✅ OK   1/1         0
  my-app-c58d9696b-hcq5g        Running  ✅ OK   1/1         0
```

> 📍 URL de verificación: `http://localhost:3000/catalog/default/component/my-app/kubernetes`

---

## 🧹 Limpieza de recursos (al finalizar el lab)

```bash
# Eliminar el Resource Group y todos sus recursos
az group delete \
  --name rg-backstage-lab \
  --yes \
  --no-wait
```

---

## 🐛 Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| `ErrCode_InsufficientVCPUQuota` | Cuota agotada para esa familia de VM | Usa `Standard_D2s_v3` (familia Dv3, común en free tier). Si también falla, ejecuta `az vm list-usage --location eastus --query "[?contains(name.value, 'Family') && currentValue < limit]" -o table` para ver qué familias tienen cuota disponible |
| Pestaña Kubernetes no aparece | Frontend no reiniciado | `yarn dev` de nuevo |
| `Unauthorized` en el plugin | Token expirado o incorrecto | Regenerar token con `kubectl create token` |
| No se ven pods | Label `backstage.io/kubernetes-id` incorrecta | Verificar que coincida en el pod y en el catalog YAML |
| `certificate verify failed` | CA incorrecto | Usar `skipTLSVerify: true` solo en lab |
| `ECONNREFUSED` | URL del cluster incorrecta | Revisar `kubectl cluster-info` |

---

## 📚 Referencias

- [Backstage Kubernetes Plugin Docs](https://backstage.io/docs/features/kubernetes/)
- [AKS Quickstart – Azure CLI](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli)
- [Kubernetes Plugin Configuration](https://backstage.io/docs/features/kubernetes/configuration)

---

> 💬 **Nota del instructor:** La etiqueta `backstage.io/kubernetes-id` en el manifiesto de Kubernetes y en el `catalog-info.yaml` debe ser **idéntica**. Es el mecanismo que usa Backstage para asociar un componente del catálogo con sus recursos en el cluster.
