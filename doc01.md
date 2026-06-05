# Documentación de `catalog-info.yaml` — Backstage Software Catalog
## Referencia completa de entidades y campos

> Versión del formato: `backstage.io/v1alpha1` y `backstage.io/v1beta1`
> Documentación oficial: https://backstage.io/docs/features/software-catalog/descriptor-format

---

## Estructura base de cualquier entidad

Todo `catalog-info.yaml` comparte la misma estructura raíz, igual que un manifiesto de Kubernetes:

```yaml
apiVersion: backstage.io/v1alpha1   # versión del esquema de Backstage
kind: Component                      # tipo de entidad (ver tabla abajo)
metadata:                            # metadatos comunes a todas las entidades
  name: mi-servicio                  # identificador único (OBLIGATORIO)
  namespace: default                 # espacio de nombres (default si se omite)
  title: "Mi Servicio de Pedidos"    # nombre amigable para la UI
  description: "Descripción larga"   # texto libre, aparece en la ficha
  labels:                            # clave-valor libres para filtrar
    squad: backend
    tier: critical
  tags:                              # lista de tags (minúsculas, kebab-case)
    - python
    - fastapi
    - rest
  annotations:                       # clave-valor con semántica definida por plugins
    github.com/project-slug: org/repo
  links:                             # links externos que aparecen en la UI
    - url: https://grafana.mi-empresa.com
      title: "Dashboard"
      icon: dashboard
      type: runbook
spec:                                # campos específicos según el Kind
  type: service
  lifecycle: production
  owner: group:default/backend-team
```

---

## Tipos de entidades (`kind`)

| Kind | Propósito | Cuándo usarlo |
|------|-----------|---------------|
| `Component` | Pieza de software (servicio, librería, web) | Casi siempre — es el tipo principal |
| `API` | Contrato de una API publicada | Cuando un servicio expone una interfaz consumible |
| `Group` | Equipo, squad, tribu | Para modelar la organización |
| `User` | Persona individual | Miembros de equipos |
| `System` | Agrupación de componentes de un producto | Para agrupar microservicios que forman un todo |
| `Domain` | Área de negocio (nivel más alto) | Para agrupar Systems |
| `Resource` | Infraestructura física/cloud | Bases de datos, buckets S3, colas |
| `Location` | Apunta a otras fuentes de entidades | Bootstrapping del catálogo |

---

## 1. `Component` — el tipo más usado

Representa cualquier pieza de software: microservicio, librería, frontend, CLI, etc.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: orders-service
  # ─── Identificación ───────────────────────────────────────────────────────
  namespace: default          # opcional; "default" si se omite
  title: "Servicio de Pedidos"
  description: |
    Microservicio encargado de gestionar el ciclo de vida
    de los pedidos: creación, actualización y cancelación.

  # ─── Clasificación libre ──────────────────────────────────────────────────
  tags:
    - python          # lenguaje
    - fastapi         # framework
    - orders          # dominio de negocio
    - rest            # protocolo

  labels:
    squad: "backend"
    tier: "critical"
    cost-center: "eng-001"

  # ─── Annotations (semántica definida por plugins) ─────────────────────────
  annotations:
    # Plugin de GitHub — habilita la pestaña de Pull Requests y Actions
    github.com/project-slug: my-org/orders-service

    # Plugin de GitLab (alternativa a GitHub)
    # gitlab.com/project-slug: my-group/orders-service

    # TechDocs — habilita la pestaña de documentación generada desde MkDocs
    backstage.io/techdocs-ref: dir:.

    # URL de CI — link directo al pipeline
    backstage.io/ci-url: https://github.com/my-org/orders-service/actions

    # Kubernetes — para el plugin de K8s (muestra pods en la ficha)
    backstage.io/kubernetes-id: orders-service
    backstage.io/kubernetes-namespace: production

    # PagerDuty — para ver incidentes activos en la ficha
    pagerduty.com/service-id: "P123456"

    # Sonarqube — para ver métricas de calidad de código
    sonarqube.org/project-key: my-org_orders-service

    # Runbook
    backstage.io/runbook-url: https://wiki.mi-empresa.com/runbooks/orders

  # ─── Links externos (aparecen en la ficha del componente) ─────────────────
  links:
    - url: https://orders-service.mi-empresa.com
      title: "Producción"
      icon: web          # web | dashboard | docs | email | alert | warning
      type: website
    - url: https://grafana.mi-empresa.com/d/orders
      title: "Dashboard Grafana"
      icon: dashboard
      type: monitoring
    - url: https://wiki.mi-empresa.com/orders
      title: "Runbook"
      icon: docs
      type: runbook

spec:
  # ─── type: categoría del componente ───────────────────────────────────────
  # service    → backend/API (lo más común)
  # website    → aplicación web/frontend
  # library    → librería compartida (npm, pip, Maven)
  # documentation → sitio de docs puro
  type: service

  # ─── lifecycle: madurez del componente ────────────────────────────────────
  # experimental  → en desarrollo o prueba
  # production    → en producción y estable
  # deprecated    → en proceso de baja
  lifecycle: production

  # ─── owner: quién es responsable (OBLIGATORIO) ────────────────────────────
  # Formato: <kind>:<namespace>/<name>
  owner: group:default/backend-team
  # También puede ser un usuario:
  # owner: user:default/juan.perez

  # ─── system: a qué sistema pertenece ─────────────────────────────────────
  system: plataforma-pedidos

  # ─── subcomponentOf: si es parte de un componente mayor ──────────────────
  # subcomponentOf: component:default/mega-monolith

  # ─── Relaciones con otras entidades ──────────────────────────────────────
  dependsOn:
    - component:default/auth-service        # depende de otro componente
    - resource:default/postgres-orders      # depende de una base de datos

  providesApis:
    - orders-api          # APIs que este componente expone

  consumesApis:
    - auth-api            # APIs de terceros que este componente consume
    - notifications-api
```

### Valores válidos para `spec.type` en Component

| Valor | Uso |
|-------|-----|
| `service` | API REST, gRPC, microservicio backend |
| `website` | Aplicación web, SPA, frontend |
| `library` | Paquete npm, pip, Maven, NuGet |
| `documentation` | Sitio de documentación puro |
| `data-pipeline` | ETL, pipeline de datos (convención de equipos) |
| `ml-model` | Modelo de ML en producción (convención) |

> Los valores marcados como "convención" no son validados por Backstage; puedes usar cualquier string. Se recomienda estandarizarlos en la organización.

---

## 2. `API` — contratos de interfaces

Representa la interfaz que un servicio expone al mundo. Se asocia a un `Component` mediante `providesApis`.

```yaml
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: orders-api
  description: "API REST para gestión de pedidos (OpenAPI 3.0)"
  tags:
    - rest
    - openapi
  annotations:
    github.com/project-slug: my-org/orders-service

spec:
  # ─── type: formato del contrato ──────────────────────────────────────────
  # openapi   → OpenAPI / Swagger (REST)
  # asyncapi  → AsyncAPI (eventos, mensajería)
  # graphql   → esquema GraphQL
  # grpc      → Protobuf / gRPC
  # trpc      → tRPC (convención)
  type: openapi

  lifecycle: production
  owner: group:default/backend-team
  system: plataforma-pedidos

  # ─── definition: dónde está la especificación ────────────────────────────
  definition:
    # Opción A — texto externo (referencia a archivo en el repo)
    $text: https://raw.githubusercontent.com/my-org/orders-service/main/openapi.yaml

    # Opción B — inline (para specs cortas o cuando no hay archivo separado)
    # $text: |
    #   openapi: "3.0.0"
    #   info:
    #     title: Orders API
    #     version: "1.0.0"
    #   paths:
    #     /orders:
    #       get:
    #         summary: List orders

    # Opción C — URL a un API gateway o Swagger Hub
    # $text: https://api.swaggerhub.com/apis/my-org/orders/1.0.0
```

---

## 3. `Group` — equipos y estructuras organizacionales

```yaml
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: backend-team
  description: "Squad de desarrollo backend — plataforma de pedidos"
  links:
    - url: https://slack.mi-empresa.com/channels/backend
      title: "Canal de Slack"
      icon: chat

spec:
  # ─── type: nivel en la jerarquía organizacional ──────────────────────────
  # team         → equipo de trabajo directo (nivel más bajo)
  # squad        → sinónimo de team (Spotify model)
  # tribe        → agrupación de squads
  # business-unit → unidad de negocio
  # department   → departamento
  # organization → nivel raíz de la empresa
  type: team

  profile:
    displayName: "Backend Team"
    email: backend-team@mi-empresa.com
    picture: https://avatars.mi-empresa.com/backend-team.png

  # ─── Jerarquía ──────────────────────────────────────────────────────────
  parent: engineering-tribe      # grupo padre (opcional)
  children:                      # subgrupos (opcional)
    - payments-squad
    - orders-squad

  # ─── Miembros ────────────────────────────────────────────────────────────
  members:
    - juan.perez
    - maria.garcia
    - carlos.lopez
```

---

## 4. `User` — personas individuales

```yaml
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: juan.perez          # debe coincidir con el login de GitHub/GitLab si usas SSO
  description: "Tech Lead — Backend Team"

spec:
  profile:
    displayName: "Juan Pérez"
    email: juan.perez@mi-empresa.com
    picture: https://avatars.githubusercontent.com/juanperez

  memberOf:
    - backend-team          # puede pertenecer a varios grupos
    - platform-guild
```

---

## 5. `System` — agrupación de componentes de un producto

```yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: plataforma-pedidos
  description: |
    Plataforma end-to-end de gestión de pedidos, incluyendo
    ingesta, procesamiento, notificaciones y reporting.
  tags:
    - core-product
    - revenue-critical
  links:
    - url: https://wiki.mi-empresa.com/plataforma-pedidos
      title: "Documentación del sistema"
      icon: docs

spec:
  owner: group:default/backend-team
  domain: ecommerce           # Domain al que pertenece este System
```

---

## 6. `Domain` — área de negocio (nivel más alto)

```yaml
apiVersion: backstage.io/v1alpha1
kind: Domain
metadata:
  name: ecommerce
  description: "Dominio de comercio electrónico — pedidos, pagos y logística"
  tags:
    - revenue

spec:
  owner: group:default/engineering-tribe
```

---

## 7. `Resource` — infraestructura y dependencias externas

```yaml
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: postgres-orders
  description: "Base de datos PostgreSQL del servicio de pedidos"
  tags:
    - database
    - postgresql
  annotations:
    # Anotación custom para el equipo de plataforma
    platform.mi-empresa.com/rds-instance: orders-db-prod

spec:
  # ─── type: tipo de recurso ───────────────────────────────────────────────
  # database      → bases de datos (PostgreSQL, MySQL, MongoDB)
  # s3            → storage de objetos
  # message-broker → Kafka, RabbitMQ, SQS
  # cache         → Redis, Memcached
  # cdn           → CloudFront, Fastly
  # cluster       → clúster de Kubernetes
  type: database

  lifecycle: production
  owner: group:default/platform-team
  system: plataforma-pedidos

  dependsOn:
    - resource:default/aws-rds-cluster-prod   # infraestructura subyacente
```

---

## Múltiples entidades en un solo archivo

Un mismo `catalog-info.yaml` puede contener varias entidades separadas por `---`. Útil en monorepos o para mantener juntos un Component y su API:

```yaml
# Primera entidad
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: auth-service
spec:
  type: service
  lifecycle: production
  owner: group:default/backend-team
  providesApis:
    - auth-api

---
# Segunda entidad en el mismo archivo
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: auth-api
  description: "API de autenticación JWT"
spec:
  type: openapi
  lifecycle: production
  owner: group:default/backend-team
  definition:
    $text: https://raw.githubusercontent.com/my-org/auth-service/main/openapi.yaml
```

---

## Annotations más usadas (referencia rápida)

| Annotation | Plugin | Qué activa |
|---|---|---|
| `github.com/project-slug: org/repo` | GitHub | Pestaña de PRs, Actions, releases |
| `gitlab.com/project-slug: group/repo` | GitLab | Pestaña de MRs, pipelines |
| `backstage.io/techdocs-ref: dir:.` | TechDocs | Documentación generada desde MkDocs |
| `backstage.io/kubernetes-id: <name>` | Kubernetes | Vista de pods y deployments |
| `backstage.io/kubernetes-namespace: <ns>` | Kubernetes | Namespace donde vive el servicio |
| `backstage.io/runbook-url: <url>` | Core | Link al runbook en la ficha |
| `backstage.io/ci-url: <url>` | Core | Link directo al pipeline de CI |
| `pagerduty.com/service-id: <id>` | PagerDuty | Incidentes activos en la ficha |
| `sonarqube.org/project-key: <key>` | SonarQube | Métricas de código en la ficha |
| `datadoghq.com/dashboard-url: <url>` | Datadog | Iframe del dashboard |
| `sentry.io/project-slug: org/project` | Sentry | Errores recientes |

---

## Formato de referencias entre entidades

Cuando en un campo como `owner`, `dependsOn`, `system`, o `memberOf` referencias otra entidad, el formato es:

```
<kind>:<namespace>/<name>
```

| Ejemplo | Significado |
|---|---|
| `group:default/backend-team` | Grupo "backend-team" en namespace "default" |
| `user:default/juan.perez` | Usuario "juan.perez" |
| `component:default/auth-service` | Componente "auth-service" |
| `api:default/orders-api` | API "orders-api" |
| `system:default/plataforma-pedidos` | System "plataforma-pedidos" |
| `resource:default/postgres-orders` | Resource "postgres-orders" |

> Si la entidad está en el namespace `default`, puedes omitir el prefijo y poner solo el nombre corto (e.g., `auth-service` en vez de `component:default/auth-service`) — Backstage infiere el namespace.

---

## Reglas de nombrado (`metadata.name`)

- Solo letras minúsculas, números y guiones (`-`)
- Sin espacios, sin underscores, sin puntos (excepto en `User`, donde se admite `juan.perez`)
- Máximo 63 caracteres
- Debe ser único dentro del mismo `kind` y `namespace`

```yaml
# ✅ Válidos
name: orders-service
name: auth-api-v2
name: juan.perez        # solo en User

# ❌ Inválidos
name: Orders Service    # espacios y mayúsculas
name: auth_api          # underscore no permitido
name: AUTH-SERVICE      # mayúsculas no permitidas
```

---

## Diagrama de relaciones entre entidades

```
Domain
  └── System
        ├── Component  ──providesApis──▶  API
        │     └── dependsOn ──────────▶  Component
        │     └── dependsOn ──────────▶  Resource
        └── Resource

Group ──owns──▶ Component / System / API / Resource
User  ──memberOf──▶ Group
```

---

## Checklist antes de hacer push de un `catalog-info.yaml`

- [ ] `name` en kebab-case, sin espacios ni mayúsculas
- [ ] `owner` apunta a un `Group` o `User` que existe en el catálogo
- [ ] `lifecycle` es uno de: `experimental`, `production`, `deprecated`
- [ ] `spec.type` está estandarizado con el resto del equipo
- [ ] Si el componente expone una API, declarada en `providesApis`
- [ ] Si el componente pertenece a un `System`, declarado en `system`
- [ ] Las annotations de GitHub/GitLab apuntan al slug correcto
- [ ] Validado con `backstage-cli catalog validate --file catalog-info.yaml`

---

## Referencias oficiales

- [Descriptor Format Reference](https://backstage.io/docs/features/software-catalog/descriptor-format)
- [Well-known Annotations](https://backstage.io/docs/features/software-catalog/well-known-annotations)
- [Well-known Relations](https://backstage.io/docs/features/software-catalog/well-known-relations)
- [System Model](https://backstage.io/docs/features/software-catalog/system-model)
- [API Reference — Catalog](https://backstage.io/docs/features/software-catalog/software-catalog-api)
