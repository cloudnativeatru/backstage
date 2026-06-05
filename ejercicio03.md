# Sesión 2 — Software Catalog & Service Discovery en Backstage
## Guía de Ejercicios Prácticos (10 ejercicios)

> **Prerrequisitos:** Backstage instalado y corriendo en `http://localhost:3000`. Acceso a al menos un repositorio existente en GitHub o GitLab (propio o del equipo). Git instalado localmente.

---

## 🧭 Contexto de la sesión

En esta sesión aprenderemos a registrar microservicios reales en el catálogo de Backstage usando `catalog-info.yaml`, a visualizar dependencias entre servicios y a gestionar ownership de equipos. Todos los ejercicios parten de repositorios que ya existen; no crearemos aplicaciones desde cero, sino que añadiremos la capa de metadatos que Backstage necesita para entenderlos.

---

## Ejercicio 1 — Explorar el Catálogo vacío y entender su estructura

**Objetivo:** Familiarizarse con la UI del catálogo antes de registrar nada.

**Pasos:**

1. Abre Backstage en `http://localhost:3000` y navega al menú **Catalog** en la barra lateral.
2. Observa que el catálogo puede estar vacío o contener solo los ejemplos por defecto.
3. Haz clic en **"Create"** (o **"Register existing component"**) para ver el flujo sin completarlo aún.
4. Vuelve al catálogo y explora los filtros disponibles: `Kind`, `Type`, `Owner`, `Lifecycle`.
5. En tu terminal, abre el archivo de configuración principal de Backstage:
   ```bash
   cat app-config.yaml | grep -A 10 "catalog:"
   ```
6. Identifica la sección `locations` — ahí es donde se declaran las fuentes de entidades.

**Reflexión:** ¿Qué tipos de entidades conoces? (`Component`, `API`, `Group`, `User`, `System`, `Domain`, `Resource`). El catálogo no es solo para microservicios.

---

## Ejercicio 2 — Crear tu primer `catalog-info.yaml` en un repositorio existente

**Objetivo:** Registrar un servicio real añadiendo el archivo de metadatos mínimo.

**Pasos:**

1. Elige un repositorio existente tuyo (puede ser un API REST, un frontend, un script). Clónalo si no lo tienes en local:
   ```bash
   git clone https://github.com/<tu-usuario>/<tu-repo>.git
   cd <tu-repo>
   ```
2. Crea el archivo en la raíz del repositorio:
   ```bash
   touch catalog-info.yaml
   ```
3. Pega el siguiente contenido y ajusta los valores a tu servicio:
   ```yaml
   apiVersion: backstage.io/v1alpha1
   kind: Component
   metadata:
     name: mi-primer-servicio          # sin espacios, en kebab-case
     description: "Descripción breve del servicio"
     tags:
       - python                        # lenguaje o tecnología principal
       - api
     annotations:
       github.com/project-slug: <tu-usuario>/<tu-repo>
   spec:
     type: service                     # service | website | library
     lifecycle: production             # experimental | production | deprecated
     owner: user:default/<tu-usuario>  # o group:default/<nombre-equipo>
   ```
4. Commitea y pushea:
   ```bash
   git add catalog-info.yaml
   git commit -m "chore: add Backstage catalog-info.yaml"
   git push origin main
   ```
5. En Backstage, ve a **Catalog → "Register Existing Component"** y pega la URL raw del archivo:
   ```
   https://raw.githubusercontent.com/<usuario>/<repo>/main/catalog-info.yaml
   ```
6. Confirma el registro y busca tu componente en el catálogo.

**Punto de validación:** El componente debe aparecer en el catálogo con su nombre, tipo y owner.

---

## Ejercicio 3 — Enriquecer los metadatos con links, labels y anotaciones

**Objetivo:** Hacer que la ficha del componente en Backstage sea realmente útil.

**Pasos:**

1. Abre el `catalog-info.yaml` del ejercicio anterior y añade las siguientes secciones:
   ```yaml
   metadata:
     name: mi-primer-servicio
     description: "API de gestión de pedidos"
     tags:
       - python
       - fastapi
       - orders
     labels:
       squad: "backend"
       tier: "crítico"
     annotations:
       github.com/project-slug: <usuario>/<repo>
       backstage.io/techdocs-ref: dir:.     # activa TechDocs si tienes docs/
       # Agrega el tablero de CI si usas GitHub Actions:
       backstage.io/ci-url: "https://github.com/<usuario>/<repo>/actions"
     links:
       - url: "https://docs.mi-empresa.com/pedidos"
         title: "Documentación"
         icon: "docs"
       - url: "https://grafana.mi-empresa.com/pedidos"
         title: "Dashboard Grafana"
         icon: "dashboard"
   ```
2. Commitea y pushea el cambio.
3. En Backstage, fuerza un refresh del componente desde su ficha (botón **"Refresh"** en la esquina superior derecha de la entidad) o espera el ciclo de polling (por defecto cada 60s).
4. Verifica que aparezcan los links en la pestaña **"Overview"** del componente.

**Tip:** Los labels son clave-valor libres, útiles para filtrar en el catálogo. Las annotations tienen semántica definida por plugins.

---

## Ejercicio 4 — Registrar múltiples componentes de un monorepo

**Objetivo:** Entender cómo manejar repositorios con varios servicios dentro.

**Pasos:**

1. Supón que tu repositorio tiene esta estructura (o créala de forma ficticia con carpetas vacías):
   ```
   mi-monorepo/
   ├── services/
   │   ├── auth-service/
   │   └── orders-service/
   └── frontend/
       └── web-app/
   ```
2. Crea un `catalog-info.yaml` dentro de **cada** subcarpeta:

   `services/auth-service/catalog-info.yaml`:
   ```yaml
   apiVersion: backstage.io/v1alpha1
   kind: Component
   metadata:
     name: auth-service
     description: "Servicio de autenticación"
     tags: [nodejs, jwt]
     annotations:
       github.com/project-slug: <usuario>/mi-monorepo
   spec:
     type: service
     lifecycle: production
     owner: group:default/backend-team
   ```

   `frontend/web-app/catalog-info.yaml`:
   ```yaml
   apiVersion: backstage.io/v1alpha1
   kind: Component
   metadata:
     name: web-app
     description: "Frontend principal"
     tags: [react, typescript]
     annotations:
       github.com/project-slug: <usuario>/mi-monorepo
   spec:
     type: website
     lifecycle: experimental
     owner: group:default/frontend-team
   ```

3. Registra cada uno por separado en Backstage usando sus URLs raw individuales, **o** añade una entrada en `app-config.yaml` que use un glob:
   ```yaml
   catalog:
     locations:
       - type: url
         target: https://github.com/<usuario>/mi-monorepo/blob/main/services/auth-service/catalog-info.yaml
       - type: url
         target: https://github.com/<usuario>/mi-monorepo/blob/main/frontend/web-app/catalog-info.yaml
   ```
4. Reinicia Backstage y verifica que aparezcan los tres componentes en el catálogo.

---

## Ejercicio 5 — Modelar dependencias con `dependsOn` y `providesApis`

**Objetivo:** Visualizar el grafo de dependencias entre servicios en Backstage.

**Pasos:**

1. Primero, define una entidad `API` en el repositorio del servicio que la expone:
   ```yaml
   # En auth-service/catalog-info.yaml — agrega una segunda entidad al mismo archivo
   ---
   apiVersion: backstage.io/v1alpha1
   kind: API
   metadata:
     name: auth-api
     description: "API REST de autenticación (JWT)"
   spec:
     type: openapi
     lifecycle: production
     owner: group:default/backend-team
     definition:
       $text: https://raw.githubusercontent.com/<usuario>/mi-monorepo/main/services/auth-service/openapi.yaml
   ```
   > Si no tienes un `openapi.yaml`, crea uno mínimo o usa `$text: ""` por ahora.

2. Enlaza el componente `auth-service` a la API que provee editando su `spec`:
   ```yaml
   spec:
     type: service
     lifecycle: production
     owner: group:default/backend-team
     providesApis:
       - auth-api
   ```

3. En el componente `orders-service`, declara que consume esa API:
   ```yaml
   spec:
     type: service
     lifecycle: production
     owner: group:default/backend-team
     dependsOn:
       - component:default/auth-service
     consumesApis:
       - auth-api
   ```

4. Pushea todos los cambios y refresca Backstage.
5. Abre la ficha de `orders-service` y ve a la pestaña **"Dependencies"** (o **"Relations"**). Deberías ver el grafo: `orders-service → auth-service → auth-api`.

**Punto de validación:** El grafo de dependencias muestra correctamente quién depende de qué.

---

## Ejercicio 6 — Crear entidades `Group` y `User` para modelar equipos

**Objetivo:** Gestionar ownership real de equipos y usuarios desde el catálogo.

**Pasos:**

1. Crea un archivo `org.yaml` en un repositorio de configuración central (o en el mismo monorepo bajo `backstage/`):
   ```yaml
   apiVersion: backstage.io/v1alpha1
   kind: Group
   metadata:
     name: backend-team
     description: "Equipo de desarrollo Backend"
   spec:
     type: team
     profile:
       displayName: "Backend Team"
       email: backend@mi-empresa.com
     children: []
     members:
       - juan.perez
       - maria.garcia
   ---
   apiVersion: backstage.io/v1alpha1
   kind: User
   metadata:
     name: juan.perez
     description: "Juan Pérez — Tech Lead Backend"
   spec:
     profile:
       displayName: "Juan Pérez"
       email: juan.perez@mi-empresa.com
     memberOf:
       - backend-team
   ```

2. Registra el archivo en Backstage igual que los anteriores.
3. Navega al catálogo y filtra por `Kind: Group`. Deberías ver `backend-team` con sus miembros.
4. Abre cualquier componente que tenga `owner: group:default/backend-team` y verifica que el link de owner lleva al perfil del grupo.
5. Desde el perfil del grupo, haz clic en **"Owned components"** para ver todos los servicios que le pertenecen.

**Reflexión:** Centralizar la organización en Backstage facilita el "who owns what" sin preguntar en Slack.

---

## Ejercicio 7 — Modelar un `System` que agrupe componentes relacionados

**Objetivo:** Usar entidades `System` para agrupar microservicios que forman un producto.

**Pasos:**

1. Agrega una entidad `System` al archivo `org.yaml` (o a un nuevo `systems.yaml`):
   ```yaml
   apiVersion: backstage.io/v1alpha1
   kind: System
   metadata:
     name: plataforma-pedidos
     description: "Plataforma completa de gestión de pedidos"
     tags: [core-product]
   spec:
     owner: group:default/backend-team
   ```

2. Actualiza los componentes `auth-service` y `orders-service` para que declaren a qué sistema pertenecen:
   ```yaml
   spec:
     type: service
     lifecycle: production
     owner: group:default/backend-team
     system: plataforma-pedidos   # ← añadir esta línea
   ```

3. Pushea y refresca Backstage.
4. En el catálogo, filtra por `Kind: System` y abre `plataforma-pedidos`.
5. La pestaña **"Diagram"** mostrará todos los componentes, APIs y recursos agrupados bajo ese sistema.

**Tip avanzado:** Los `System` son el nivel intermedio entre un `Component` individual y un `Domain` (área de negocio completa). Un `Domain: ecommerce` puede contener varios `System`: `plataforma-pedidos`, `plataforma-pagos`, etc.

---

## Ejercicio 8 — Integración con GitHub para sincronización automática

**Objetivo:** Configurar Backstage para que descubra automáticamente `catalog-info.yaml` en toda la organización GitHub, sin registro manual.

**Pasos:**

1. Asegúrate de tener configurado el token de GitHub en `app-config.yaml`:
   ```yaml
   integrations:
     github:
       - host: github.com
         token: ${GITHUB_TOKEN}
   ```

2. Añade la configuración del GitHub discovery provider en `app-config.yaml`:
   ```yaml
   catalog:
     providers:
       github:
         my-org:
           organization: "<nombre-de-tu-org-o-usuario>"
           catalogPath: "/catalog-info.yaml"   # ruta dentro de cada repo
           filters:
             branch: "main"
             repository: ".*"                  # todos los repos; usa regex para filtrar
           schedule:
             frequency: { minutes: 30 }
             timeout: { minutes: 3 }
   ```

3. En el backend de Backstage, instala el plugin de discovery si no lo tienes:
   ```bash
   cd packages/backend
   yarn add @backstage/plugin-catalog-backend-module-github
   ```

4. Registra el módulo en `packages/backend/src/index.ts`:
   ```typescript
   backend.add(import('@backstage/plugin-catalog-backend-module-github'));
   ```

5. Reinicia Backstage. En los logs deberías ver mensajes como:
   ```
   [github-discovery] Discovered N entities in org/<tu-org>
   ```

6. Ve al catálogo y verifica que aparecen automáticamente todos los repositorios que tienen `catalog-info.yaml` en la rama `main`.

**Punto de validación:** Crear un nuevo `catalog-info.yaml` en cualquier repo de la org y esperar el ciclo de 30 minutos — debe aparecer en Backstage sin registro manual.

---

## Ejercicio 9 — Validar y hacer lint de un `catalog-info.yaml` antes de hacer push

**Objetivo:** Detectar errores en el YAML de catálogo antes de que rompan Backstage.

**Pasos:**

1. Instala la CLI de Backstage globalmente:
   ```bash
   npm install -g @backstage/cli
   ```

2. Valida un archivo individual:
   ```bash
   backstage-cli catalog validate --file catalog-info.yaml
   ```
   La salida mostrará errores de esquema si el YAML no es válido.

3. Para validar múltiples archivos en un monorepo:
   ```bash
   find . -name "catalog-info.yaml" | xargs -I {} backstage-cli catalog validate --file {}
   ```

4. Integra esta validación en un GitHub Actions workflow. Crea `.github/workflows/validate-catalog.yaml`:
   ```yaml
   name: Validate Backstage Catalog

   on:
     pull_request:
       paths:
         - "**/catalog-info.yaml"

   jobs:
     validate:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: "20"
         - run: npm install -g @backstage/cli
         - run: |
             find . -name "catalog-info.yaml" \
               | xargs -I {} backstage-cli catalog validate --file {}
   ```

5. Abre un PR que contenga un `catalog-info.yaml` intencionalmente roto (por ejemplo, con `lifecycle: invalid-value`) y verifica que el workflow falla con un mensaje claro.

**Reflexión:** Este workflow evita que PRs rompan el catálogo en producción. Es la "calidad de datos" del Software Catalog.

---

## Ejercicio 10 — Explorar el grafo completo y generar un inventario del catálogo

**Objetivo:** Usar la API de Backstage para consultar el estado del catálogo programáticamente.

**Pasos:**

1. Backstage expone una API REST del catálogo en `http://localhost:7007/api/catalog`. Consulta todas las entidades:
   ```bash
   curl -s http://localhost:7007/api/catalog/entities \
     | python3 -m json.tool | head -80
   ```

2. Filtra solo los componentes de tipo `service`:
   ```bash
   curl -s "http://localhost:7007/api/catalog/entities?filter=kind=Component,spec.type=service" \
     | python3 -c "
   import json, sys
   data = json.load(sys.stdin)
   for e in data:
       print(f\"{e['metadata']['name']:30s} owner={e['spec'].get('owner','N/A'):30s} lifecycle={e['spec'].get('lifecycle','N/A')}\")
   "
   ```

3. Genera un inventario CSV de todos los componentes:
   ```bash
   curl -s "http://localhost:7007/api/catalog/entities?filter=kind=Component" \
     | python3 -c "
   import json, sys, csv
   data = json.load(sys.stdin)
   writer = csv.writer(sys.stdout)
   writer.writerow(['name', 'type', 'owner', 'lifecycle', 'system', 'tags'])
   for e in data:
       m = e['metadata']
       s = e['spec']
       writer.writerow([
           m.get('name',''),
           s.get('type',''),
           s.get('owner',''),
           s.get('lifecycle',''),
           s.get('system',''),
           ','.join(m.get('tags', []))
       ])
   " > inventario_catalogo.csv
   cat inventario_catalogo.csv
   ```

4. Consulta las relaciones de un componente específico para ver su grafo de dependencias via API:
   ```bash
   curl -s "http://localhost:7007/api/catalog/entities/by-name/component/default/orders-service" \
     | python3 -c "
   import json, sys
   data = json.load(sys.stdin)
   print('Relations:')
   for r in data.get('relations', []):
       print(f\"  {r['type']:20s} → {r['targetRef']}\")
   "
   ```

5. Identifica componentes sin sistema asignado (potenciales "huérfanos"):
   ```bash
   curl -s "http://localhost:7007/api/catalog/entities?filter=kind=Component" \
     | python3 -c "
   import json, sys
   data = json.load(sys.stdin)
   huerfanos = [e['metadata']['name'] for e in data if not e['spec'].get('system')]
   print(f'Componentes sin sistema ({len(huerfanos)}):')
   for n in huerfanos: print(f'  - {n}')
   "
   ```

**Punto de validación:** Tienes un inventario CSV del catálogo y puedes identificar gaps de ownership o componentes sin sistema asignado — base para una reunión de "catálogo health check".

---

## 📋 Resumen de entidades usadas

| Entidad | Para qué sirve | Campo clave |
|---|---|---|
| `Component` | Microservicio, librería, frontend | `spec.type`, `spec.lifecycle` |
| `API` | Contrato de la API (OpenAPI, gRPC) | `spec.type: openapi` |
| `Group` | Equipo o squad | `spec.members` |
| `User` | Persona individual | `spec.memberOf` |
| `System` | Agrupación de componentes de un producto | `spec.owner` |
| `Domain` | Área de negocio (nivel más alto) | `spec.owner` |

## 🔗 Referencias

- [Backstage Descriptor Format](https://backstage.io/docs/features/software-catalog/descriptor-format)
- [Well-known Annotations](https://backstage.io/docs/features/software-catalog/well-known-annotations)
- [GitHub Discovery Provider](https://backstage.io/docs/integrations/github/discovery)
- [Catalog API Reference](https://backstage.io/docs/features/software-catalog/software-catalog-api)
