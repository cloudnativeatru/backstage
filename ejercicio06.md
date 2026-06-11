# Sesión 4: API Catalog & OpenAPI Integration

> **Duración total estimada:** ~4 horas  
> Ejercicios por tema (~30 min c/u) + Lab principal (~2 horas)  
> **Nivel:** Intermedio — Backstage corriendo localmente con Software Catalog activo.

---

## Estructura real de este repositorio

```
lab-04/
├── README.md                              ← este archivo
├── all-components.yaml                   ← registro unificado (System + 3 Components + 3 APIs)
│
├── payments-api/
│   ├── catalog-info.yaml                 ← Component + 2 APIs (openapi + asyncapi)
│   ├── package.json
│   ├── mkdocs.yml                        ← TechDocs config
│   ├── src/
│   │   ├── index.js                      ← Entry point Express
│   │   ├── routes/payments.js            ← CRUD de pagos
│   │   └── kafka/producer.js             ← Kafka producer (con fallback a logs)
│   ├── openapi/
│   │   └── payments.yaml                 ← OpenAPI 3.0 spec ✅
│   ├── asyncapi/
│   │   └── payments-events.yaml          ← AsyncAPI 2.6 spec ✅
│   └── docs/
│       ├── index.md
│       └── architecture.md
│
├── accounts-api/
│   ├── catalog-info.yaml                 ← Component + 1 API (openapi)
│   ├── requirements.txt
│   ├── mkdocs.yml
│   ├── src/
│   │   └── main.py                       ← FastAPI app completa
│   ├── openapi/
│   │   └── accounts.yaml                 ← OpenAPI 3.0 spec ✅
│   └── docs/
│       └── index.md
│
└── notifications-service/
    ├── catalog-info.yaml                 ← Component (consumesApis)
    ├── package.json
    ├── mkdocs.yml
    ├── src/
    │   ├── index.js
    │   └── kafka/consumer.js             ← Kafka consumer (con fallback simulado)
    └── docs/
        └── index.md
```

---

## Arquitectura del ecosistema

```
┌──────────────────────────────────────────────────────────────────┐
│                     FINTECH RÁPIDA S.A.                          │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │            payments-api  :3000  (Node.js/Express)       │    │
│  │  GET|POST  /payments                                     │    │
│  │  GET       /payments/:id                                 │    │
│  │  PATCH     /payments/:id/cancel                         │    │
│  │  GET       /api-docs   ← Swagger UI                     │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ produce eventos                        │
│                         ▼                                        │
│              ┌──────────────────────┐                            │
│              │   Kafka (opcional)   │                            │
│              │  topic: payment.*    │                            │
│              └──────────┬───────────┘                            │
│                         │ consume eventos                        │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │       notifications-service  (Node.js/KafkaJS)          │    │
│  │  → Envía notificación al usuario                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │            accounts-api  :3001  (Python/FastAPI)        │    │
│  │  GET|POST  /accounts                                     │    │
│  │  GET       /accounts/:id                                 │    │
│  │  PATCH     /accounts/:id/balance                        │    │
│  │  GET       /docs   ← FastAPI Swagger automático         │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

**Relaciones en Backstage:**

```
System: fintech-rapida
  ├── Component: payments-api        → providesApis: [payments-openapi, payments-asyncapi]
  ├── Component: accounts-api        → providesApis: [accounts-openapi]
  ├── Component: notifications-service → consumesApis: [payments-asyncapi]
  ├── API: payments-openapi          (type: openapi  → payments-api/openapi/payments.yaml)
  ├── API: accounts-openapi          (type: openapi  → accounts-api/openapi/accounts.yaml)
  └── API: payments-asyncapi         (type: asyncapi → payments-api/asyncapi/payments-events.yaml)
```

---

## Tema 1 – Documentación Viva con OpenAPI/Swagger (~30 min)

### Concepto

El plugin `@backstage/plugin-api-docs` renderiza specs OpenAPI/Swagger directamente en el catálogo.
La spec vive en el repo → se versiona con el código → Backstage la muestra siempre actualizada.

### Ejercicio: Visualizar Swagger UI en Backstage

#### Paso 1 — Verificar el plugin en `packages/app/src/App.tsx`

```tsx
import { ApiExplorerPage } from '@backstage/plugin-api-docs';

// Dentro de <FlatRoutes>:
<Route path="/api-docs" element={<ApiExplorerPage />} />
```

Si no existe, instalar:

```bash
yarn --cwd packages/app add @backstage/plugin-api-docs
```

#### Paso 2 — Agregar `ApiDefinitionCard` al EntityPage

En `packages/app/src/components/catalog/EntityPage.tsx`:

```tsx
import {
  ApiDefinitionCard,
  ProvidedApisCard,
  ConsumedApisCard,
} from '@backstage/plugin-api-docs';

// En el componentPage, agregar ruta /api:
<EntityLayout.Route path="/api" title="API">
  <Grid container spacing={3}>
    <Grid item xs={12} md={6}>
      <ProvidedApisCard />
    </Grid>
    <Grid item xs={12} md={6}>
      <ConsumedApisCard />
    </Grid>
  </Grid>
</EntityLayout.Route>

// En el apiPage, agregar ApiDefinitionCard:
<EntityLayout.Route path="/" title="Overview">
  <Grid container spacing={3}>
    <Grid item xs={12}>
      <EntityAboutCard />
    </Grid>
    <Grid item xs={12}>
      <ApiDefinitionCard />    {/* ← Esta card renderiza Swagger/AsyncAPI */}
    </Grid>
  </Grid>
</EntityLayout.Route>
```

#### Paso 3 — Registrar la API en Backstage

La spec ya existe en: `payments-api/openapi/payments.yaml`

Registra el `catalog-info.yaml` de payments-api en tu Backstage:

```
http://localhost:3000/catalog-import
```

Ingresa la ruta del archivo (ver opciones en la Fase 3 del Lab).

✅ **Checkpoint:** Entidad `payments-openapi` → pestaña **Definition** → Swagger UI interactivo.

---

## Tema 2 – API Catalog: Centralizar contratos (~30 min)

### Concepto

El API Catalog centraliza todos los contratos. Las relaciones `providesApis` / `consumesApis`
en el `catalog-info.yaml` de cada Component crean el grafo de dependencias visible en Backstage.

### Ejercicio: Registrar el ecosistema completo

#### Paso 1 — Revisar `all-components.yaml`

Abre el archivo `all-components.yaml` de este lab. Contiene en un solo archivo:

- 1 System (`fintech-rapida`)
- 3 Components (`payments-api`, `accounts-api`, `notifications-service`)
- 3 APIs (`payments-openapi`, `accounts-openapi`, `payments-asyncapi`)

Nota cómo las rutas en `definition.$text` apuntan a los archivos YAML reales:

```yaml
spec:
  type: openapi
  definition:
    $text: ./payments-api/openapi/payments.yaml   # ← archivo real en este repo
```

#### Paso 2 — Registrar en Backstage

Registra `all-components.yaml` en Backstage. Una sola URL registra las 7 entidades.

#### Paso 3 — Explorar el grafo

1. Ve a **Catalog → Systems → fintech-rapida**
2. Pestaña **Diagram**
3. Verifica las flechas: `payments-api → payments-openapi`, `notifications-service → payments-asyncapi`

#### Paso 4 — Verificar relaciones desde la API

1. Ve a **Catalog → APIs → payments-asyncapi**
2. Pestaña **Relations**: debe mostrar que `payments-api` la provee y `notifications-service` la consume

✅ **Checkpoint:** Grafo completo con 6 nodos y flechas de relación.

---

## Tema 3 – TechDocs: Docs-like-code (~30 min)

### Concepto

TechDocs renderiza Markdown del repo como documentación navegable dentro de Backstage.
Los archivos necesarios son: `mkdocs.yml` en la raíz y carpeta `docs/`.

### Archivos ya listos en este lab

```
payments-api/
├── mkdocs.yml          ← configuración MkDocs
└── docs/
    ├── index.md        ← página principal
    └── architecture.md ← diagrama de flujo y decisiones
```

### Ejercicio: Verificar TechDocs en Backstage

#### Paso 1 — Confirmar la annotation en `catalog-info.yaml`

```yaml
metadata:
  annotations:
    backstage.io/techdocs-ref: dir:.   # ← OBLIGATORIA para TechDocs
```

Todos los `catalog-info.yaml` de este lab ya la tienen incluida.

#### Paso 2 — Revisar `mkdocs.yml`

```yaml
# payments-api/mkdocs.yml
site_name: Payments API
nav:
  - Home: index.md
  - Architecture: architecture.md
plugins:
  - techdocs-core    # ← plugin requerido por Backstage
```

#### Paso 3 — Generar preview local (opcional)

```bash
npm install -g @techdocs/cli
cd payments-api
techdocs-cli serve
# Preview en http://localhost:3000
```

#### Paso 4 — Ver en Backstage

Navega a: **Catalog → Components → payments-api → pestaña Docs**

✅ **Checkpoint:** Documentación Markdown renderizada con navegación lateral.

**Actividad adicional:** Agrega una página `docs/changelog.md` a `payments-api`, actualiza `mkdocs.yml` con la nueva entrada, y verifica que aparece en Backstage sin re-registrar la entidad.

---

## Tema 4 – Plugin AsyncAPI para eventos Kafka/RabbitMQ (~30 min)

### Concepto

AsyncAPI es el estándar equivalente a OpenAPI para comunicación asíncrona.
Backstage lo renderiza nativamente cuando `spec.type: asyncapi`.

### Spec ya lista en este lab

```
payments-api/asyncapi/payments-events.yaml
```

Describe 3 topics de Kafka:
- `payment.created` — cuando se crea un pago
- `payment.processed` — cuando se aprueba o rechaza
- `account.updated` — cuando cambia el saldo

### Ejercicio: Explorar AsyncAPI Studio en Backstage

#### Paso 1 — Verificar el tipo en `catalog-info.yaml`

```yaml
# En payments-api/catalog-info.yaml
spec:
  type: asyncapi          # ← diferencia clave vs openapi
  definition:
    $text: ./asyncapi/payments-events.yaml
```

#### Paso 2 — Navegar en Backstage

1. Ve a **Catalog → APIs → payments-asyncapi**
2. Pestaña **Definition** → AsyncAPI Studio
3. Explora los channels: `payment.created`, `payment.processed`, `account.updated`
4. Revisa los schemas de payload en cada mensaje

#### Paso 3 — Análisis del schema

Abre `payments-api/asyncapi/payments-events.yaml` y responde:

- ¿Qué campos son `required` en `PaymentCreatedPayload`?
- ¿Qué valores acepta el campo `status` en `PaymentProcessedPayload`?
- ¿Qué protocolo usa el server `local`?

**Actividad adicional:** Agrega un nuevo channel `payment.refunded` con un payload que incluya `paymentId`, `refundAmount` y `refundedAt`. Verifica que aparece en el AsyncAPI Studio de Backstage.

✅ **Checkpoint:** AsyncAPI Studio muestra los 3 topics con sus schemas interactivos.

---

## Lab Nº 4 — Registrar APIs en Backstage (2 horas)

**Objetivo:** Integrar el ecosistema completo de FinTech Rápida S.A. en Backstage local, con código funcional, specs reales, TechDocs y grafo de relaciones completo.

---

### FASE 1 — Instalar dependencias (15 min)

```bash
# payments-api
cd payments-api
npm install
cd ..

# accounts-api
cd accounts-api
pip install -r requirements.txt
cd ..

# notifications-service
cd notifications-service
npm install
cd ..
```

---

### FASE 2 — Arrancar los tres servicios (10 min)

Abre **3 terminales separados**:

```bash
# Terminal 1 — payments-api
cd payments-api
npm run dev
# ✅ Esperado: "payments-api corriendo en http://localhost:3000"
```

```bash
# Terminal 2 — accounts-api
cd accounts-api
uvicorn src.main:app --port 3001 --reload
# ✅ Esperado: "Uvicorn running on http://127.0.0.1:3001"
```

```bash
# Terminal 3 — notifications-service
cd notifications-service
npm run dev
# ✅ Esperado: "modo simulación activo" (si no hay Kafka)
```

**Verificar que los servicios responden:**

```bash
curl http://localhost:3000/health
# {"status":"ok","service":"payments-api","version":"1.0.0"}

curl http://localhost:3001/health
# {"status":"ok","service":"accounts-api","version":"1.0.0"}
```

---

### FASE 3 — Registrar entidades en Backstage (20 min)

Tienes dos opciones para registrar los archivos locales:

#### Opción A — Servidor HTTP local (recomendado)

```bash
# Desde la raíz del lab-04
npx serve . -p 8080
```

Luego registra en Backstage (`http://localhost:3000/catalog-import`):

```
http://localhost:8080/all-components.yaml
```

Esto registra de un golpe: 1 System + 3 Components + 3 APIs.

#### Opción B — Registrar cada archivo por separado

```
http://localhost:8080/payments-api/catalog-info.yaml
http://localhost:8080/accounts-api/catalog-info.yaml
http://localhost:8080/notifications-service/catalog-info.yaml
```

**Verificar en Backstage:**
- Catalog → Systems → `fintech-rapida` ✓
- Catalog → Components → `payments-api`, `accounts-api`, `notifications-service` ✓
- Catalog → APIs → `payments-openapi`, `accounts-openapi`, `payments-asyncapi` ✓

---

### FASE 4 — Explorar el API Catalog (20 min)

#### 4.1 Swagger UI de payments-api

1. **Catalog → APIs → payments-openapi → Definition**
2. Ejecuta desde Swagger UI:
   - `POST /payments` con body de ejemplo
   - `GET /payments` para listar
   - `GET /payments/{id}` con el ID del pago creado

#### 4.2 FastAPI Docs de accounts-api

1. Abre directamente: `http://localhost:3001/docs`
2. Crea una cuenta con `POST /accounts`
3. Consulta con `GET /accounts/{accountId}`

> **Nota:** accounts-api tiene documentación automática de FastAPI.
> La spec manual en `openapi/accounts.yaml` se usa para Backstage.

#### 4.3 AsyncAPI Studio de payments-asyncapi

1. **Catalog → APIs → payments-asyncapi → Definition**
2. Explora los channels y los schemas de cada mensaje
3. Verifica que `notifications-service` aparece como consumer en la pestaña **Relations**

#### 4.4 Grafo de relaciones

1. **Catalog → Systems → fintech-rapida → Diagram**
2. El grafo debe mostrar:
   - `payments-api` conectado a `payments-openapi` y `payments-asyncapi`
   - `accounts-api` conectado a `accounts-openapi`
   - `notifications-service` conectado (consume) a `payments-asyncapi`

---

### FASE 5 — TechDocs (15 min)

1. **Catalog → Components → payments-api → Docs**
2. Deberías ver las páginas: `Home` y `Architecture`

Si TechDocs no renderiza (requiere configuración de builder en Backstage local):

```bash
# Generar preview directamente con techdocs-cli
npm install -g @techdocs/cli
cd payments-api
techdocs-cli serve
```

**Actividad:** Abre `payments-api/docs/index.md`, agrega una sección nueva (ej: `## Errores comunes`), y verifica el cambio en Backstage o en el preview local.

---

### FASE 6 — Pruebas end-to-end (20 min)

Ejecuta este flujo completo para ver los tres servicios interactuar:

```bash
# 1. Crear cuenta origen en accounts-api
ACCT_ORIGEN=$(curl -s -X POST http://localhost:3001/accounts \
  -H "Content-Type: application/json" \
  -d '{"ownerId":"user-001","accountType":"savings","currency":"PEN","initialBalance":1000}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['accountId'])")

echo "Cuenta origen: $ACCT_ORIGEN"

# 2. Crear cuenta destino
ACCT_DESTINO=$(curl -s -X POST http://localhost:3001/accounts \
  -H "Content-Type: application/json" \
  -d '{"ownerId":"user-002","accountType":"checking","currency":"PEN","initialBalance":0}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['accountId'])")

echo "Cuenta destino: $ACCT_DESTINO"

# 3. Crear un pago en payments-api
PAYMENT_ID=$(curl -s -X POST http://localhost:3000/payments \
  -H "Content-Type: application/json" \
  -d "{
    \"amount\": 250.00,
    \"currency\": \"PEN\",
    \"sourceAccountId\": \"$ACCT_ORIGEN\",
    \"destinationAccountId\": \"$ACCT_DESTINO\",
    \"description\": \"Pago de alquiler Lab 04\"
  }" | python3 -c "import sys,json; print(json.load(sys.stdin)['paymentId'])")

echo "Pago creado: $PAYMENT_ID"

# 4. Consultar el pago
echo "\n--- Detalle del pago ---"
curl -s http://localhost:3000/payments/$PAYMENT_ID | python3 -m json.tool

# 5. Listar todos los pagos
echo "\n--- Lista de pagos ---"
curl -s "http://localhost:3000/payments?status=created" | python3 -m json.tool

# 6. Cancelar el pago
echo "\n--- Cancelando el pago ---"
curl -s -X PATCH http://localhost:3000/payments/$PAYMENT_ID/cancel | python3 -m json.tool
```

**Observar en el terminal de `payments-api`:** los eventos Kafka simulados (`[KAFKA EVENT SIMULADO]`).
**Observar en el terminal de `notifications-service`:** las notificaciones recibidas.

---

### FASE 7 — Validación final (20 min)

Verifica cada checkpoint en Backstage:

| # | Checkpoint | Dónde verificar |
|---|-----------|-----------------|
| ✅ 1 | System `fintech-rapida` registrado | Catalog → Systems |
| ✅ 2 | 3 Components visibles | Catalog → Components |
| ✅ 3 | 3 APIs visibles | Catalog → APIs |
| ✅ 4 | Swagger UI de `payments-openapi` renderiza | API → payments-openapi → Definition |
| ✅ 5 | Swagger UI de `accounts-openapi` renderiza | API → accounts-openapi → Definition |
| ✅ 6 | AsyncAPI Studio de `payments-asyncapi` renderiza | API → payments-asyncapi → Definition |
| ✅ 7 | Grafo de relaciones completo | System → fintech-rapida → Diagram |
| ✅ 8 | `notifications-service` aparece como consumer | API → payments-asyncapi → Relations |
| ✅ 9 | TechDocs de `payments-api` | Component → payments-api → Docs |
| ✅ 10 | Swagger UI de payments funciona vía `http://localhost:3000/api-docs` | Browser directo |

---

## Troubleshooting

**La spec OpenAPI no se carga en Backstage**  
→ Verifica que la ruta en `definition.$text` es relativa al `catalog-info.yaml` y que el archivo YAML existe físicamente.  
→ Para evitar este problema, también puedes apuntar a la URL directa: `http://localhost:3000/openapi.yaml`

**Error: "group:default/guests not found"**  
→ Esto es normal en Backstage recién instalado. El grupo `guests` se crea automáticamente; si no existe, cambia el owner a `user:default/guest` en los `catalog-info.yaml`.

**accounts-api: ModuleNotFoundError**  
→ Asegúrate de ejecutar `uvicorn` desde la carpeta `accounts-api`:
```bash
cd accounts-api && uvicorn src.main:app --port 3001 --reload
```

**TechDocs muestra "Documentation not found"**  
→ Verifica que `backstage.io/techdocs-ref: dir:.` está en las annotations.  
→ En desarrollo local, Backstage puede requerir configurar `techdocs.builder: 'local'` en `app-config.yaml`.

**El plugin asyncapi no muestra el Studio**  
→ Verifica que la versión del archivo es `asyncapi: 2.6.0` (no 3.x, que aún no es soportada por el plugin de Backstage estable).

---

## Referencias

- [Backstage API Docs Plugin](https://backstage.io/docs/features/software-catalog/descriptor-format#kind-api)
- [OpenAPI Specification 3.0](https://swagger.io/specification/)
- [AsyncAPI Specification 2.6](https://www.asyncapi.com/docs/reference/specification/v2.6.0)
- [TechDocs — Backstage](https://backstage.io/docs/features/techdocs/)
- [catalog-info.yaml — Descriptor Format](https://backstage.io/docs/features/software-catalog/descriptor-format)
