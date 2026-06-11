# Sesión 4: API Catalog & OpenAPI Integration

> **Duración total estimada:** ~4 horas  
> Ejercicios por tema (~30 min c/u) + Lab principal (~2 horas)  
> **Nivel:** Intermedio — Backstage corriendo localmente con Software Catalog activo.

> ⚠️ **Arquitectura:** Este lab usa la **nueva arquitectura declarativa de Backstage**
> (`createApp` + `features` en `App.tsx`). Los pasos son distintos al Backstage clásico
> con `FlatRoutes` y `EntityPage.tsx`. Si tu `App.tsx` usa `createApp` de
> `@backstage/frontend-defaults`, estos pasos son para ti.

---

## Estructura de este repositorio

```
lab-04/
├── README.md
├── all-components.yaml                   ← registro unificado (System + 3 Components + 3 APIs)
│
├── payments-api/
│   ├── catalog-info.yaml
│   ├── package.json
│   ├── mkdocs.yml
│   ├── src/
│   │   ├── index.js
│   │   ├── routes/payments.js
│   │   └── kafka/producer.js
│   ├── openapi/
│   │   └── payments.yaml                 ← OpenAPI 3.0 spec ✅
│   ├── asyncapi/
│   │   └── payments-events.yaml          ← AsyncAPI 2.6 spec ✅
│   └── docs/
│       ├── index.md
│       └── architecture.md
│
├── accounts-api/
│   ├── catalog-info.yaml
│   ├── requirements.txt
│   ├── mkdocs.yml
│   ├── src/
│   │   └── main.py
│   ├── openapi/
│   │   └── accounts.yaml                 ← OpenAPI 3.0 spec ✅
│   └── docs/
│       └── index.md
│
└── notifications-service/
    ├── catalog-info.yaml
    ├── package.json
    ├── mkdocs.yml
    ├── src/
    │   ├── index.js
    │   └── kafka/consumer.js
    └── docs/
        └── index.md
```

---

## Arquitectura del ecosistema FinTech Rápida S.A.

```
┌──────────────────────────────────────────────────────────────────┐
│                     FINTECH RÁPIDA S.A.                          │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │            payments-api  :3000  (Node.js/Express)       │    │
│  │  POST /payments  · GET /payments  · GET /payments/:id   │    │
│  │  PATCH /payments/:id/cancel                             │    │
│  │  GET /api-docs   ← Swagger UI embebido                  │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │ produce eventos Kafka                  │
│                         ▼                                        │
│              ┌──────────────────────┐                            │
│              │   Kafka (opcional)   │  topic: payment.*          │
│              └──────────┬───────────┘                            │
│                         │ consume eventos                        │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │       notifications-service  (Node.js/KafkaJS)          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │            accounts-api  :3001  (Python/FastAPI)        │    │
│  │  POST /accounts  · GET /accounts  · PATCH /balance      │    │
│  │  GET /docs  ← FastAPI Swagger automático                │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

**Relaciones en Backstage:**

```
System: fintech-rapida
  ├── Component: payments-api         → providesApis: [payments-openapi, payments-asyncapi]
  ├── Component: accounts-api         → providesApis: [accounts-openapi]
  ├── Component: notifications-service → consumesApis: [payments-asyncapi]
  ├── API: payments-openapi           (type: openapi)
  ├── API: accounts-openapi           (type: openapi)
  └── API: payments-asyncapi          (type: asyncapi)
```

---

## Tema 1 – Documentación Viva con OpenAPI/Swagger (~30 min)

### Concepto

El plugin `@backstage/plugin-api-docs` renderiza specs OpenAPI/Swagger directamente
en el catálogo. La spec vive en el repo → se versiona con el código → Backstage la
muestra siempre actualizada.

### Cómo funciona en la arquitectura nueva (`createApp` + `features`)

En tu Backstage **no existe** `EntityPage.tsx` ni `FlatRoutes`. Los plugins se registran
como `features` en `App.tsx`. El plugin de API Docs tiene una versión `/alpha` compatible
con esta arquitectura.

### Ejercicio Tema 1: Habilitar el plugin de API Docs (~30 min)

#### Paso 1 — Instalar el paquete (ya lo hiciste)

```bash
yarn --cwd packages/app add @backstage/plugin-api-docs
```

#### Paso 2 — Importar y registrar el plugin en `App.tsx`

Abre `packages/app/src/App.tsx` y agrega el plugin en el array `features`:

```tsx
// ANTES (tu App.tsx actual):
import { createApp } from '@backstage/frontend-defaults';
import catalogPlugin from '@backstage/plugin-catalog/alpha';
import { navModule } from './modules/nav';
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPageBlueprint } from '@backstage/plugin-app-react';
import { SignInPage } from '@backstage/core-components';
import { createFrontendModule } from '@backstage/frontend-plugin-api';
import githubActionsPlugin from '@backstage-community/plugin-github-actions/alpha';

// AGREGAR esta línea de import:
import apiDocsPlugin from '@backstage/plugin-api-docs/alpha';

// DESPUÉS — agregar apiDocsPlugin al array features:
export default createApp({
  features: [
    catalogPlugin,
    navModule,
    githubActionsPlugin,
    apiDocsPlugin,          // ← agrega esta línea
    createFrontendModule({
      pluginId: 'app',
      extensions: [signInPage],
    }),
  ],
});
```

> **¿Por qué `/alpha`?** La nueva arquitectura declarativa de Backstage requiere que
> los plugins exporten sus extensiones desde el entry point `/alpha`. Sin eso,
> el plugin no registra sus vistas en el catálogo.

#### Paso 3 — Reiniciar Backstage

```bash
# Detén el proceso actual (Ctrl+C) y vuelve a arrancar:
yarn dev
```

#### Paso 4 — Verificar que el plugin cargó

1. Abre `http://localhost:3000`
2. En el menú lateral busca **"APIs"** — si aparece, el plugin está activo
3. Si no aparece en el menú: verifica que el import es desde `/alpha` (no desde la raíz)

#### Paso 5 — Registrar la primera API y verla

Arranca el servidor HTTP del lab (desde la carpeta `lab-04`):

```bash
npx serve . -p 8080
```

Luego en Backstage:
1. Ve a **Catalog → + Register Existing Component**
2. Ingresa: `http://localhost:8080/payments-api/catalog-info.yaml`
3. Haz clic en **Analyze** → **Import**
4. Ve a **Catalog → APIs → payments-openapi**
5. Pestaña **Definition** → deberías ver el **Swagger UI interactivo**

✅ **Checkpoint:** Swagger UI renderiza con los endpoints de `payments-api`.

---

## Tema 2 – API Catalog: Centralizar contratos (~30 min)

### Concepto

El API Catalog centraliza todos los contratos de tu organización. Las relaciones
`providesApis` / `consumesApis` en cada `catalog-info.yaml` crean el grafo de
dependencias visible en Backstage.

### Ejercicio Tema 2: Registrar el ecosistema completo

#### Paso 1 — Revisar `all-components.yaml`

Abre `all-components.yaml`. Contiene 7 entidades en un solo archivo:

```yaml
# Fragmento clave — cómo se declaran las relaciones:

# El Component que PROVEE la API:
spec:
  type: service
  providesApis:
    - payments-openapi      # ← nombre de la entidad API

# La entidad API con su spec:
spec:
  type: openapi
  definition:
    $text: ./payments-api/openapi/payments.yaml   # ← ruta al YAML real
```

#### Paso 2 — Registrar todo de un golpe

Con el servidor HTTP corriendo (`npx serve . -p 8080`):

1. Ve a **Catalog → + Register Existing Component**
2. Ingresa: `http://localhost:8080/all-components.yaml`
3. Backstage detectará las 7 entidades — haz clic en **Import All**

#### Paso 3 — Verificar en Backstage

- **Catalog → Systems** → debe aparecer `fintech-rapida`
- **Catalog → APIs** → deben aparecer `payments-openapi`, `accounts-openapi`, `payments-asyncapi`
- **Catalog → Components** → deben aparecer los 3 servicios

#### Paso 4 — Ver el grafo de relaciones

1. Ve a **Catalog → Systems → fintech-rapida**
2. Pestaña **Diagram**
3. El grafo debe mostrar las flechas entre componentes y APIs

✅ **Checkpoint:** Grafo con 7 nodos y flechas de `providesApis`/`consumesApis`.

---

## Tema 3 – TechDocs: Docs-like-code (~30 min)

### Concepto

TechDocs renderiza Markdown del repo como documentación navegable dentro de Backstage.
La documentation vive en Git junto al código. Requiere dos archivos: `mkdocs.yml` y
la carpeta `docs/`.

### Archivos ya listos en este lab

```
payments-api/
├── mkdocs.yml          ← config MkDocs con plugin techdocs-core
└── docs/
    ├── index.md        ← página principal
    └── architecture.md ← diagrama de flujo
```

### Ejercicio Tema 3: Visualizar TechDocs

#### Paso 1 — Verificar la annotation obligatoria

Abre `payments-api/catalog-info.yaml` y confirma que existe:

```yaml
metadata:
  annotations:
    backstage.io/techdocs-ref: dir:.   # ← SIN esta annotation, TechDocs no funciona
```

Todos los `catalog-info.yaml` del lab ya la tienen.

#### Paso 2 — Verificar `mkdocs.yml`

```yaml
# payments-api/mkdocs.yml
site_name: Payments API
nav:
  - Home: index.md
  - Architecture: architecture.md
plugins:
  - techdocs-core    # ← requerido por Backstage
```

#### Paso 3 — Habilitar TechDocs en `app-config.yaml`

Abre el `app-config.yaml` de tu Backstage y verifica (o agrega) esta sección:

```yaml
techdocs:
  builder: 'local'           # genera docs en tu máquina
  generator:
    runIn: 'local'           # sin Docker
  publisher:
    type: 'local'            # guarda en disco local
```

> Si no tienes esta config, TechDocs mostrará "Documentation not found".

#### Paso 4 — Instalar el generador de TechDocs (una sola vez)

```bash
pip install mkdocs-techdocs-core
```

#### Paso 5 — Verificar en Backstage

1. **Catalog → Components → payments-api**
2. Pestaña **Docs**
3. Debes ver las páginas `Home` y `Architecture` renderizadas

#### Actividad adicional

Agrega una nueva página al lab:

```bash
# Crear el archivo
cat > payments-api/docs/errores.md << 'EOF'
# Errores comunes

## 400 — amount inválido
El campo `amount` debe ser mayor a 0.

## 409 — Pago no cancelable
Solo se pueden cancelar pagos en estado `created`.
EOF
```

Actualiza `payments-api/mkdocs.yml`:

```yaml
nav:
  - Home: index.md
  - Architecture: architecture.md
  - Errores: errores.md    # ← agregar esta línea
```

Recarga la pestaña Docs en Backstage → debe aparecer la nueva página.

✅ **Checkpoint:** TechDocs muestra las páginas navegables dentro de Backstage.

---

## Tema 4 – Plugin AsyncAPI para eventos Kafka/RabbitMQ (~30 min)

### Concepto

AsyncAPI es el estándar equivalente a OpenAPI para comunicación asíncrona (Kafka, RabbitMQ,
SQS). Backstage lo renderiza automáticamente cuando `spec.type: asyncapi`.

### Ejercicio Tema 4: Explorar AsyncAPI Studio

#### Paso 1 — Verificar que el plugin api-docs cargó (del Tema 1)

El plugin `@backstage/plugin-api-docs/alpha` maneja **tanto** OpenAPI como AsyncAPI.
No necesitas instalar un plugin adicional.

#### Paso 2 — Ver el archivo AsyncAPI del lab

Abre `payments-api/asyncapi/payments-events.yaml`. Describe 3 topics de Kafka:

```yaml
channels:
  payment.created:      # ← emitido al crear un pago
  payment.processed:    # ← emitido al aprobar/rechazar
  account.updated:      # ← emitido al cambiar saldo
```

#### Paso 3 — Verificar el tipo en `catalog-info.yaml`

```yaml
# En payments-api/catalog-info.yaml
spec:
  type: asyncapi          # ← diferencia clave vs openapi
  definition:
    $text: ./asyncapi/payments-events.yaml
```

#### Paso 4 — Ver AsyncAPI Studio en Backstage

1. **Catalog → APIs → payments-asyncapi**
2. Pestaña **Definition**
3. Explora los channels y sus schemas de payload

#### Paso 5 — Análisis del schema (actividad)

Responde en base al archivo `payments-events.yaml`:

- ¿Qué campos son `required` en `PaymentCreatedPayload`?
- ¿Qué valores acepta `status` en `PaymentProcessedPayload`?
- ¿Qué protocolo usa el server `local`?

#### Actividad adicional: Agregar un nuevo channel

Abre `payments-api/asyncapi/payments-events.yaml` y agrega al final de `channels`:

```yaml
  payment.refunded:
    description: Emitido cuando un pago es reembolsado
    publish:
      operationId: onPaymentRefunded
      summary: Pago reembolsado
      message:
        name: PaymentRefunded
        contentType: application/json
        payload:
          type: object
          required: [paymentId, refundAmount, refundedAt]
          properties:
            paymentId:
              type: string
              format: uuid
            refundAmount:
              type: number
              format: double
            refundedAt:
              type: string
              format: date-time
```

Recarga la entidad en Backstage (o espera el refresh automático) → el nuevo channel
`payment.refunded` debe aparecer en el AsyncAPI Studio.

✅ **Checkpoint:** AsyncAPI Studio muestra 4 channels con sus schemas.

---

## Lab Nº 4 — Integración completa en Backstage (2 horas)

---

### FASE 0 — Prerequisito: Plugin API Docs habilitado (5 min)

Antes de continuar, confirma que completaste el Tema 1:

- [ ] `apiDocsPlugin` importado desde `@backstage/plugin-api-docs/alpha`
- [ ] `apiDocsPlugin` agregado al array `features` en `App.tsx`
- [ ] Backstage reiniciado con `yarn dev`
- [ ] El menú lateral muestra **"APIs"**

Si no ves "APIs" en el menú, **no avances** — revisa el Tema 1 primero.

---

### FASE 1 — Instalar dependencias de los servicios (15 min)

```bash
# Desde la carpeta lab-04/

# payments-api (Node.js)
cd payments-api
npm install
cd ..

# accounts-api (Python)
cd accounts-api
pip install -r requirements.txt
cd ..

# notifications-service (Node.js)
cd notifications-service
npm install
cd ..
```

---

### FASE 2 — Arrancar los tres servicios (10 min)

Abre **3 terminales separados**:

```bash
# Terminal 1 — payments-api (puerto 3000)
cd lab-04/payments-api
npm run dev
# ✅ Esperado:
#   payments-api corriendo en http://localhost:3000
#   Swagger UI:   http://localhost:3000/api-docs
```

```bash
# Terminal 2 — accounts-api (puerto 3001)
cd lab-04/accounts-api
uvicorn src.main:app --port 3001 --reload
# ✅ Esperado: Uvicorn running on http://127.0.0.1:3001
```

```bash
# Terminal 3 — notifications-service (sin puerto HTTP)
cd lab-04/notifications-service
npm run dev
# ✅ Esperado: "modo simulación activo" (sin Kafka es normal)
```

**Verificar health checks:**

```bash
curl http://localhost:3000/health
# {"status":"ok","service":"payments-api","version":"1.0.0"}

curl http://localhost:3001/health
# {"status":"ok","service":"accounts-api","version":"1.0.0"}
```

**Verificar Swagger UI directo (fuera de Backstage):**

```bash
open http://localhost:3000/api-docs
# Abre el Swagger UI de payments-api en el browser
```

---

### FASE 3 — Registrar entidades en Backstage (20 min)

#### Paso 1 — Arrancar servidor HTTP para los archivos del lab

```bash
# En un 4to terminal, desde lab-04/
npx serve . -p 8080
# ✅ Esperado: Serving files from lab-04/ at http://localhost:8080
```

#### Paso 2 — Registrar el sistema completo

1. Abre Backstage: `http://localhost:3000`
2. Ve a **Catalog → + Register Existing Component**
3. Ingresa la URL:
   ```
   http://localhost:8080/all-components.yaml
   ```
4. Clic en **Analyze**
5. Backstage mostrará las 7 entidades detectadas — clic en **Import**

#### Paso 3 — Verificar el registro

| Entidad | Tipo | Dónde verificar |
|---------|------|-----------------|
| `fintech-rapida` | System | Catalog → Systems |
| `payments-api` | Component | Catalog → Components |
| `accounts-api` | Component | Catalog → Components |
| `notifications-service` | Component | Catalog → Components |
| `payments-openapi` | API | Catalog → APIs |
| `accounts-openapi` | API | Catalog → APIs |
| `payments-asyncapi` | API | Catalog → APIs |

> **Si aparece error "Not Found" en alguna spec:**  
> Verifica que el servidor HTTP en puerto 8080 está corriendo y que la ruta en
> `definition.$text` del YAML coincide con la estructura real de carpetas.

---

### FASE 4 — Explorar el API Catalog (20 min)

#### 4.1 Swagger UI de payments-api

1. **Catalog → APIs → payments-openapi**
2. Pestaña **Definition** → Swagger UI
3. Ejecuta directamente desde Backstage:
   - Expande `POST /payments` → clic en **Try it out**
   - Usa este body:
     ```json
     {
       "amount": 350.00,
       "currency": "PEN",
       "sourceAccountId": "acct-origen-01",
       "destinationAccountId": "acct-destino-01",
       "description": "Prueba desde Backstage"
     }
     ```
   - Clic en **Execute**
   - Copia el `paymentId` de la respuesta

4. Expande `GET /payments/{paymentId}` → pega el ID → **Execute**

#### 4.2 Swagger UI de accounts-api

1. **Catalog → APIs → accounts-openapi**
2. Pestaña **Definition** → Swagger UI
3. Explora los endpoints de cuentas

Alternativamente, la FastAPI docs nativa:
```
http://localhost:3001/docs
```

#### 4.3 AsyncAPI Studio de payments-asyncapi

1. **Catalog → APIs → payments-asyncapi**
2. Pestaña **Definition** → AsyncAPI Studio
3. Explora los 3 channels: `payment.created`, `payment.processed`, `account.updated`
4. Pestaña **Relations** → verifica que aparecen:
   - `payments-api` como **Provider**
   - `notifications-service` como **Consumer**

#### 4.4 Grafo del sistema

1. **Catalog → Systems → fintech-rapida**
2. Pestaña **Diagram**
3. El grafo debe mostrar:

```
[payments-api] ──providesApis──► [payments-openapi]
[payments-api] ──providesApis──► [payments-asyncapi]
[accounts-api] ──providesApis──► [accounts-openapi]
[notifications-service] ──consumesApis──► [payments-asyncapi]
```

---

### FASE 5 — TechDocs (15 min)

#### Paso 1 — Verificar configuración en `app-config.yaml`

En tu Backstage, abre `app-config.yaml` y verifica/agrega:

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```

#### Paso 2 — Instalar el generador (si no lo tienes)

```bash
pip install mkdocs-techdocs-core
```

#### Paso 3 — Verificar en Backstage

1. **Catalog → Components → payments-api**
2. Pestaña **Docs**
3. Debes ver navegación con: `Home` y `Architecture`

#### Paso 4 — Actividad: Agregar página de errores

```bash
# Desde lab-04/
cat > payments-api/docs/errores.md << 'EOF'
# Errores comunes

| Código | Mensaje | Causa |
|--------|---------|-------|
| 400 | amount debe ser mayor a 0 | El campo amount es 0 o negativo |
| 400 | currency inválido | Usar solo: PEN, USD, EUR |
| 404 | Pago no encontrado | El paymentId no existe |
| 409 | No se puede cancelar | El pago no está en estado created |
EOF
```

Actualiza `payments-api/mkdocs.yml`:

```yaml
nav:
  - Home: index.md
  - Architecture: architecture.md
  - Errores: errores.md
```

Recarga la pestaña Docs en Backstage para ver el cambio.

---

### FASE 6 — Pruebas end-to-end con curl (20 min)

```bash
# 1. Crear cuenta origen
ACCT_ORIGEN=$(curl -s -X POST http://localhost:3001/accounts \
  -H "Content-Type: application/json" \
  -d '{"ownerId":"user-001","accountType":"savings","currency":"PEN","initialBalance":1000}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['accountId'])")
echo "Cuenta origen:  $ACCT_ORIGEN"

# 2. Crear cuenta destino
ACCT_DESTINO=$(curl -s -X POST http://localhost:3001/accounts \
  -H "Content-Type: application/json" \
  -d '{"ownerId":"user-002","accountType":"checking","currency":"PEN","initialBalance":0}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['accountId'])")
echo "Cuenta destino: $ACCT_DESTINO"

# 3. Crear un pago
PAYMENT_ID=$(curl -s -X POST http://localhost:3000/payments \
  -H "Content-Type: application/json" \
  -d "{
    \"amount\": 250.00,
    \"currency\": \"PEN\",
    \"sourceAccountId\": \"$ACCT_ORIGEN\",
    \"destinationAccountId\": \"$ACCT_DESTINO\",
    \"description\": \"Pago de alquiler Lab 04\"
  }" | python3 -c "import sys,json; print(json.load(sys.stdin)['paymentId'])")
echo "Pago creado:    $PAYMENT_ID"

# 4. Consultar el pago
echo "--- Detalle del pago ---"
curl -s http://localhost:3000/payments/$PAYMENT_ID | python3 -m json.tool

# 5. Listar pagos creados
echo "--- Pagos en estado 'created' ---"
curl -s "http://localhost:3000/payments?status=created" | python3 -m json.tool

# 6. Cancelar el pago
echo "--- Cancelando el pago ---"
curl -s -X PATCH http://localhost:3000/payments/$PAYMENT_ID/cancel | python3 -m json.tool
```

Observa en el terminal de `payments-api`:

```
📋 [KAFKA EVENT SIMULADO]
   Topic   : payment.created
   Payload : { "paymentId": "...", "amount": 250, ... }
```

Observa en el terminal de `notifications-service`:

```
📨 Evento recibido [payment.created]
   → [NOTIFICACIÓN ENVIADA]
     Para    : acct-origen-01
     Mensaje : Tu pago de 250 PEN está siendo procesado...
```

---

### FASE 7 — Validación final (20 min)

Verifica cada checkpoint antes de cerrar el lab:

| # | Checkpoint | Dónde verificar | Estado |
|---|-----------|-----------------|--------|
| 1 | `apiDocsPlugin` en `App.tsx` y menú "APIs" visible | Menú lateral Backstage | ⬜ |
| 2 | System `fintech-rapida` registrado | Catalog → Systems | ⬜ |
| 3 | 3 Components registrados | Catalog → Components | ⬜ |
| 4 | 3 APIs registradas | Catalog → APIs | ⬜ |
| 5 | Swagger UI de `payments-openapi` renderiza | API → payments-openapi → Definition | ⬜ |
| 6 | Swagger UI de `accounts-openapi` renderiza | API → accounts-openapi → Definition | ⬜ |
| 7 | AsyncAPI Studio de `payments-asyncapi` renderiza | API → payments-asyncapi → Definition | ⬜ |
| 8 | Grafo de relaciones completo | System → fintech-rapida → Diagram | ⬜ |
| 9 | `notifications-service` aparece como Consumer | API → payments-asyncapi → Relations | ⬜ |
| 10 | TechDocs de `payments-api` con 2+ páginas | Component → payments-api → Docs | ⬜ |

---

## Troubleshooting

**"APIs" no aparece en el menú de Backstage**
→ El import debe ser exactamente:
```tsx
import apiDocsPlugin from '@backstage/plugin-api-docs/alpha';
```
La barra `/alpha` es obligatoria. Sin ella el plugin se instala pero no registra
sus extensiones en la UI.

**La spec OpenAPI no renderiza — muestra el YAML en texto plano**
→ Mismo problema: el plugin no cargó correctamente. Verifica el import `/alpha`.

**Error al registrar: "Target https://... is not a valid location"**
→ Backstage no puede alcanzar el archivo. Verifica que el servidor HTTP
(`npx serve . -p 8080`) está corriendo y que la URL es accesible:
```bash
curl http://localhost:8080/all-components.yaml
# Debe devolver el contenido YAML, no un error 404
```

**Error: "group:default/guests not found"**
→ Normal en Backstage nuevo. Cambia el `owner` en todos los `catalog-info.yaml`:
```yaml
spec:
  owner: user:default/guest   # en lugar de group:default/guests
```

**accounts-api: "No module named 'fastapi'"**
→ Instala desde la carpeta correcta:
```bash
cd lab-04/accounts-api
pip install -r requirements.txt
```

**TechDocs muestra "Documentation not found"**
→ Verifica en `app-config.yaml`:
```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```
→ Instala el generador: `pip install mkdocs-techdocs-core`

**AsyncAPI Studio muestra error de parsing**
→ Verifica que el archivo empiece con `asyncapi: 2.6.0` (no versión 3.x).

---

## Referencias

- [Backstage New Frontend System](https://backstage.io/docs/frontend-system/)
- [plugin-api-docs en la nueva arquitectura](https://backstage.io/docs/reference/plugin-api-docs/)
- [OpenAPI Specification 3.0](https://swagger.io/specification/)
- [AsyncAPI Specification 2.6](https://www.asyncapi.com/docs/reference/specification/v2.6.0)
- [TechDocs — Backstage](https://backstage.io/docs/features/techdocs/)
- [catalog-info.yaml — Descriptor Format](https://backstage.io/docs/features/software-catalog/descriptor-format)
