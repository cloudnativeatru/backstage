# Ejercicio Propuesto — Sesión 4: API Catalog & OpenAPI Integration

**Duración estimada:** 50 minutos | **Backstage ya en ejecución**

---

## Contexto

Eres el **Platform Engineer** de **FinTech Peru S.A.**, una empresa con tres unidades de negocio: Banca Digital, Créditos y Notificaciones. Hasta hoy, cada equipo vive en su propio silo — nadie sabe quién es dueño de qué, qué APIs existen ni cómo consumirlas.

Tu misión: modelar toda la organización en Backstage en una sola sesión. Al terminar, cualquier desarrollador de la empresa debe poder entrar al portal y responder en menos de 30 segundos:

> *¿Quién mantiene este servicio? ¿Qué endpoints expone? ¿Qué eventos produce? ¿Dónde está la documentación?*

---

## Estructura del repositorio

Crea un repositorio llamado `fintech-platform-catalog` con esta estructura:

```
fintech-platform-catalog/
├── org/
│   ├── groups.yaml                  ← equipos de la empresa
│   └── users.yaml                   ← miembros de cada equipo
├── systems/
│   ├── banca-digital.yaml
│   ├── creditos.yaml
│   └── notificaciones.yaml
├── services/
│   ├── payments-api/
│   │   ├── catalog-info.yaml        ← Component + API (OpenAPI)
│   │   ├── openapi.yaml
│   │   └── docs/
│   │       └── index.md             ← TechDocs
│   ├── loans-api/
│   │   ├── catalog-info.yaml
│   │   └── openapi.yaml
│   ├── accounts-api/
│   │   ├── catalog-info.yaml
│   │   └── openapi.yaml
│   └── notifications-service/
│       ├── catalog-info.yaml        ← Component + API (AsyncAPI)
│       └── asyncapi.yaml
└── all-locations.yaml               ← registra todo desde un solo archivo
```

---

## Parte 1 — Organización: Groups y Users

**⏱ 10 min**

Modela los tres equipos de la empresa y sus miembros.

### `org/groups.yaml`

Declara los tres equipos con `Kind: Group`:

| Grupo | Tipo | Descripción |
|-------|------|-------------|
| `platform-team` | `team` | Equipo de plataforma, dueño del portal |
| `banca-digital-team` | `team` | Dueño de pagos y cuentas |
| `creditos-team` | `team` | Dueño del servicio de préstamos |

Ejemplo de estructura para un grupo:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: banca-digital-team
  description: "Equipo responsable de pagos y cuentas — FinTech Peru"
spec:
  type: team
  members:
    - usuario1
    - usuario2
```

### `org/users.yaml`

Declara al menos **4 usuarios** distribuidos entre los equipos. Cada usuario debe tener `displayName`, `email` y pertenecer a un `memberOf`.

```yaml
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: ana.torres
spec:
  profile:
    displayName: Ana Torres
    email: ana.torres@fintechperu.com
  memberOf:
    - banca-digital-team
```

### Verificación

En Backstage → **Catalog → Groups**: los tres equipos deben aparecer con sus miembros listados.

---

## Parte 2 — Sistemas

**⏱ 5 min**

Cada sistema agrupa los componentes de una unidad de negocio. Crea los tres sistemas con `Kind: System`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: banca-digital
  description: "Sistema de banca digital — pagos y cuentas"
spec:
  owner: banca-digital-team
```

Crea un archivo por sistema en `systems/`. Asigna el `owner` según esta tabla:

| Sistema | Owner |
|---------|-------|
| `banca-digital` | `banca-digital-team` |
| `creditos` | `creditos-team` |
| `notificaciones` | `platform-team` |

### Verificación

En Backstage → **Catalog → Systems**: tres sistemas visibles. Al hacer clic en uno debe mostrar los componentes que le pertenecen (se poblarán en la Parte 3).

---

## Parte 3 — Componentes y APIs REST

**⏱ 20 min**

Cada servicio necesita un `Kind: Component` y un `Kind: API` en el mismo `catalog-info.yaml`, enlazados con `providesApis`. Usa `---` para separar los dos documentos YAML en el mismo archivo.

Estructura base del `catalog-info.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payments-api
  description: "Servicio de procesamiento de pagos"
  annotations:
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: production
  owner: banca-digital-team
  system: banca-digital
  providesApis:
    - payments-api-spec        # ← referencia al Kind: API de abajo
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payments-api-spec
  description: "Contrato OpenAPI del servicio de pagos"
spec:
  type: openapi
  lifecycle: production
  owner: banca-digital-team
  definition:
    $text: ./openapi.yaml      # ← apunta al archivo del contrato
```

---

### payments-api — sistema: `banca-digital` · owner: `banca-digital-team`

Crea `services/payments-api/openapi.yaml` con los siguientes endpoints:

| Método | Path | Descripción |
|--------|------|-------------|
| `POST` | `/payments` | Iniciar un pago |
| `GET` | `/payments/{id}` | Consultar estado de un pago |
| `DELETE` | `/payments/{id}` | Anular un pago pendiente |
| `GET` | `/health` | Health check |

Schemas requeridos en `components/schemas`:

- `Payment`: id, amount, currency, status, createdAt
- `ErrorResponse`: code, message

---

### accounts-api — sistema: `banca-digital` · owner: `banca-digital-team`

Crea `services/accounts-api/openapi.yaml` con:

| Método | Path | Descripción |
|--------|------|-------------|
| `GET` | `/accounts/{id}` | Obtener datos de una cuenta |
| `GET` | `/accounts/{id}/balance` | Consultar saldo |
| `POST` | `/accounts` | Crear una nueva cuenta |

Schema requerido: `Account` (id, ownerId, type, balance, currency).

---

### loans-api — sistema: `creditos` · owner: `creditos-team`

Crea `services/loans-api/openapi.yaml` con:

| Método | Path | Descripción |
|--------|------|-------------|
| `POST` | `/loans/apply` | Solicitar un préstamo |
| `GET` | `/loans/{id}` | Consultar estado del préstamo |
| `GET` | `/loans/{id}/schedule` | Ver cronograma de pagos |

Schema requerido: `Loan` (id, applicantId, amount, term, interestRate, status).

---

### notifications-service — sistema: `notificaciones` · owner: `platform-team`

Este servicio no expone REST. Solo produce eventos Kafka. Su API usa `type: asyncapi`.

Crea `services/notifications-service/asyncapi.yaml` con dos canales:

| Canal | Operación | Descripción |
|-------|-----------|-------------|
| `payment.completed` | publish | Pago procesado exitosamente |
| `loan.approved` | publish | Préstamo aprobado |

Cada mensaje debe tener schema con al menos: `eventId`, `timestamp`, `payload`.

Estructura base del contrato:

```yaml
asyncapi: '2.6.0'
info:
  title: Notifications Service
  version: '1.0.0'
  description: Servicio de eventos de dominio — FinTech Peru
channels:
  payment.completed:
    subscribe:
      message:
        payload:
          type: object
          properties:
            eventId:
              type: string
            timestamp:
              type: string
              format: date-time
            payload:
              type: object
```

En el `catalog-info.yaml` de este servicio, el `Kind: API` debe declarar:

```yaml
spec:
  type: asyncapi     # ← no openapi
```

### Verificación

En Backstage → **Catalog → APIs**: las cuatro APIs deben aparecer listadas. Al hacer clic en cualquiera de las tres REST, el Swagger UI debe mostrar los endpoints expandibles. Al hacer clic en `notifications-service`, el contrato AsyncAPI debe renderizar con los canales Kafka.

---

## Parte 4 — TechDocs para payments-api

**⏱ 8 min**

En `services/payments-api/docs/index.md` escribe la documentación técnica con estas cinco secciones obligatorias:

**1. Descripción**
Qué hace el servicio y a qué sistema pertenece.

**2. Equipo responsable**
Nombre del grupo y canal de contacto (puede ser inventado, por ejemplo `#banca-digital` en Slack).

**3. Endpoints**
Tabla con método, path y descripción breve de cada uno.

**4. Ejemplo de uso**
Bloque de código con un `curl` completo hacia `POST /payments`, incluyendo headers `Content-Type` y `Authorization`, y un body JSON de ejemplo.

```bash
curl -X POST https://api.fintechperu.com/payments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "amount": 150.00,
    "currency": "PEN",
    "sourceAccountId": "acc-001",
    "destinationAccountId": "acc-002"
  }'
```

**5. Dependencias**
Menciona que este servicio consume `accounts-api` para validar el origen de los fondos antes de procesar el pago.

### Verificación

En Backstage → componente `payments-api` → pestaña **Docs**: el Markdown debe renderizar con todas las secciones, la tabla de endpoints y el bloque de código con syntax highlighting.

---

## Parte 5 — Relaciones entre componentes

**⏱ 5 min**

Las relaciones se declaran en el `catalog-info.yaml` de cada componente con `consumesApis` y `dependsOn` dentro de `spec`. Configura las siguientes:

| Componente | Relación | Destino | Motivo |
|------------|----------|---------|--------|
| `payments-api` | `consumesApis` | `accounts-api-spec` | Valida fondos antes de procesar |
| `payments-api` | `dependsOn` | `component:notifications-service` | Publica evento al completar el pago |
| `loans-api` | `consumesApis` | `accounts-api-spec` | Verifica historial del cliente |

Ejemplo en `catalog-info.yaml`:

```yaml
spec:
  type: service
  owner: banca-digital-team
  system: banca-digital
  providesApis:
    - payments-api-spec
  consumesApis:
    - accounts-api-spec
  dependsOn:
    - component:notifications-service
```

### Verificación

En Backstage → componente `payments-api` → pestaña **Relations** o el grafo de dependencias: deben aparecer flechas hacia `accounts-api` y `notifications-service`.

---

## Registro centralizado — all-locations.yaml

Usa un único `Kind: Location` que apunte a todos los archivos del repositorio. El instructor registrará solo este archivo en Backstage para que aparezca todo el catálogo de golpe.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Location
metadata:
  name: fintech-platform-all
  description: Catálogo completo de FinTech Peru S.A.
spec:
  targets:
    - ./org/groups.yaml
    - ./org/users.yaml
    - ./systems/banca-digital.yaml
    - ./systems/creditos.yaml
    - ./systems/notificaciones.yaml
    - ./services/payments-api/catalog-info.yaml
    - ./services/accounts-api/catalog-info.yaml
    - ./services/loans-api/catalog-info.yaml
    - ./services/notifications-service/catalog-info.yaml
```

---

## Criterios de evaluación

| Criterio | Puntos |
|----------|--------|
| 3 grupos y 4+ usuarios registrados con relaciones correctas | 15 |
| 3 sistemas visibles con owner correcto en Backstage | 10 |
| `payments-api`: OpenAPI con 4 endpoints, schemas y Swagger UI navegable | 20 |
| `accounts-api` y `loans-api`: OpenAPI correcto y visible en API Catalog | 15 |
| `notifications-service`: AsyncAPI con 2 canales Kafka renderizado | 15 |
| TechDocs de `payments-api` con las 5 secciones y bloque de código | 15 |
| Relaciones `consumesApis` / `dependsOn` visibles en el grafo | 10 |
| **Total** | **100** |

---

## Pistas

- Los `Kind: Group` y `Kind: User` se registran igual que los componentes — a través de `all-locations.yaml`
- Para que el grafo de relaciones funcione, los valores en `consumesApis` y `dependsOn` deben coincidir **exactamente** con el `metadata.name` de la entidad referenciada
- `Kind: System` necesita `spec.owner` pero no requiere `spec.type`
- Si el AsyncAPI no renderiza, verifica que tienes instalado `@backstage/plugin-api-docs` y que el `type` en el `Kind: API` dice exactamente `asyncapi`
- Si TechDocs no aparece, verifica que la anotación `backstage.io/techdocs-ref: dir:.` está en el `Kind: Component`, no en el `Kind: API`
