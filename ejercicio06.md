# Sesión 4: API Catalog & OpenAPI Integration

> **Duración total estimada:** ~4 horas  
> Ejercicios por tema (~30 min c/u) + Lab principal (~2 horas)  
> **Nivel:** Intermedio — Backstage corriendo localmente con Software Catalog activo.

> ⚠️ **Arquitectura:** Este lab usa la **nueva arquitectura declarativa de Backstage**
> (`createApp` + `features` en `App.tsx`). Los pasos son distintos al Backstage clásico
> con `FlatRoutes` y `EntityPage.tsx`. Si tu `App.tsx` usa `createApp` de
> `@backstage/frontend-defaults`, estos pasos son para ti.

---

## Paso previo — Subir este lab a GitHub

Backstage lee los `catalog-info.yaml` directamente desde GitHub. Es la forma más
limpia: no necesitas servidor HTTP local ni configurar `backend.reading.allow`.

```bash
# Desde la carpeta lab-04/
git init
git add .
git commit -m "feat: lab-04 API Catalog & OpenAPI Integration"

# Opción A — con GitHub CLI
gh repo create lab-04-backstage --public --source=. --push

# Opción B — manual
# 1. Crea el repo en https://github.com/new  (nombre: lab-04-backstage, público)
# 2. git remote add origin https://github.com/TU_USUARIO/lab-04-backstage.git
# 3. git push -u origin main
```

Una vez subido, tus URLs de registro quedan así (reemplaza `TU_USUARIO`):

```
# Registro unificado — 7 entidades de un golpe:
https://github.com/TU_USUARIO/lab-04-backstage/blob/main/all-components.yaml

# O por separado:
https://github.com/TU_USUARIO/lab-04-backstage/blob/main/payments-api/catalog-info.yaml
https://github.com/TU_USUARIO/lab-04-backstage/blob/main/accounts-api/catalog-info.yaml
https://github.com/TU_USUARIO/lab-04-backstage/blob/main/notifications-service/catalog-info.yaml
```

> **Repo privado:** necesitas un token de GitHub en `app-config.yaml`:
> ```yaml
> integrations:
>   github:
>     - host: github.com
>       token: ${GITHUB_TOKEN}
> ```
> **Repo público:** no necesitas token, Backstage lo lee directamente.

> **Rutas `$text` relativas:** Backstage resuelve automáticamente las rutas como
> `./openapi/payments.yaml` desde el mismo repo y rama. No necesitas cambiar nada.

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
  ├── Component: payments-api          → providesApis: [payments-openapi, payments-asyncapi]
  ├── Component: accounts-api          → providesApis: [accounts-openapi]
  ├── Component: notifications-service → consumesApis: [payments-asyncapi]
  ├── API: payments-openapi            (type: openapi)
  ├── API: accounts-openapi            (type: openapi)
  └── API: payments-asyncapi           (type: asyncapi)
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

### Ejercicio Tema 1: Habilitar el plugin de API Docs

#### Paso 1 — Instalar el paquete

```bash
yarn --cwd packages/app add @backstage/plugin-api-docs
```

#### Paso 2 — Importar y registrar en `App.tsx`

Abre `packages/app/src/App.tsx` y aplica estos dos cambios:

```tsx
// 1. Agrega este import junto a los demás imports:
import apiDocsPlugin from '@backstage/plugin-api-docs/alpha';

// 2. Agrega apiDocsPlugin al array features:
export default createApp({
  features: [
    catalogPlugin,
    navModule,
    githubActionsPlugin,
    apiDocsPlugin,          // ← nueva línea
    createFrontendModule({
      pluginId: 'app',
      extensions: [signInPage],
    }),
  ],
});
```

> **¿Por qué `/alpha`?** La nueva arquitectura declarativa requiere que los plugins
> exporten sus extensiones desde `/alpha`. Sin esa barra, el paquete se instala pero
> no registra nada en la UI — por eso no aparecía nada.

#### Paso 3 — Reiniciar Backstage

```bash
# Ctrl+C para detener, luego:
yarn dev
```

#### Paso 4 — Verificar que el plugin cargó

1. Abre `http://localhost:3000`
2. En el menú lateral debe aparecer **"APIs"**
3. Si no aparece: verifica que el import incluye `/alpha`

#### Paso 5 — Registrar la primera API desde GitHub

Con el repo ya subido a GitHub:

1. Ve a **Catalog → + Register Existing Component**
2. Ingresa la URL de GitHub:
   ```
   https://github.com/TU_USUARIO/lab-04-backstage/blob/main/payments-api/catalog-info.yaml
   ```
3. Clic en **Analyze** → **Import**
4. Ve a **Catalog → APIs → payments-openapi**
5. Pestaña **Definition** → Swagger UI interactivo

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

# La entidad API con su spec (ruta relativa dentro del mismo repo):
spec:
  type: openapi
  definition:
    $text: ./payments-api/openapi/payments.yaml
```

#### Paso 2 — Registrar todo de un golpe desde GitHub

1. Ve a **Catalog → + Register Existing Component**
2. Ingresa:
   ```
   https://github.com/TU_USUARIO/lab-04-backstage/blob/main/all-components.yaml
   ```
3. Clic en **Analyze** — Backstage detectará las 7 entidades
4. Clic en **Import**

#### Paso 3 — Verificar en Backstage

- **Catalog → Systems** → `fintech-rapida` ✓
- **Catalog → APIs** → `payments-openapi`, `accounts-openapi`, `payments-asyncapi` ✓
- **Catalog → Components** → `payments-api`, `accounts-api`, `notifications-service` ✓

#### Paso 4 — Ver el grafo de relaciones

1. **Catalog → Systems → fintech-rapida**
2. Pestaña **Diagram**
3. Verifica las flechas entre componentes y APIs

✅ **Checkpoint:** Grafo con 7 nodos y flechas de `providesApis`/`consumesApis`.

---

## Tema 3 – TechDocs: Docs-like-code (~30 min)

### Concepto

TechDocs renderiza Markdown del repo como documentación navegable dentro de Backstage.
La documentación vive en Git junto al código. Requiere `mkdocs.yml` y carpeta `docs/`.

### Archivos ya listos en este repo

```
payments-api/
├── mkdocs.yml          ← config MkDocs con plugin techdocs-core
└── docs/
    ├── index.md        ← página principal
    └── architecture.md ← diagrama de flujo y decisiones de diseño
```

### Ejercicio Tema 3: Visualizar TechDocs

#### Paso 1 — Verificar la annotation obligatoria

Abre `payments-api/catalog-info.yaml` y confirma:

```yaml
metadata:
  annotations:
    backstage.io/techdocs-ref: dir:.   # ← SIN esto, TechDocs no funciona
```

Todos los `catalog-info.yaml` del lab ya la tienen incluida.

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

En tu Backstage, abre `app-config.yaml` y verifica (o agrega):

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```

#### Paso 4 — Instalar el generador (una sola vez)

```bash
pip install mkdocs-techdocs-core
```

#### Paso 5 — Verificar en Backstage

1. **Catalog → Components → payments-api → Docs**
2. Debes ver las páginas `Home` y `Architecture` renderizadas

#### Actividad adicional — Agregar una nueva página

```bash
# En tu repo local, crear el archivo:
cat > payments-api/docs/errores.md << 'EOF'
# Errores comunes

| Código | Mensaje                        | Causa                          |
|--------|--------------------------------|--------------------------------|
| 400    | amount debe ser mayor a 0      | El campo amount es 0 o negativo|
| 400    | currency inválido               | Usar solo: PEN, USD, EUR       |
| 404    | Pago no encontrado             | El paymentId no existe         |
| 409    | No se puede cancelar           | El pago no está en estado created |
EOF
```

Actualiza `payments-api/mkdocs.yml`:

```yaml
nav:
  - Home: index.md
  - Architecture: architecture.md
  - Errores: errores.md    # ← agregar esta línea
```

Haz commit y push:

```bash
git add payments-api/docs/errores.md payments-api/mkdocs.yml
git commit -m "docs: agregar página de errores"
git push
```

Recarga la pestaña Docs en Backstage → nueva página visible.

✅ **Checkpoint:** TechDocs muestra 3 páginas navegables.

---

## Tema 4 – Plugin AsyncAPI para eventos Kafka/RabbitMQ (~30 min)

### Concepto

AsyncAPI es el estándar equivalente a OpenAPI para comunicación asíncrona (Kafka, RabbitMQ,
SQS). Backstage lo renderiza automáticamente cuando `spec.type: asyncapi`.
El plugin `@backstage/plugin-api-docs/alpha` maneja tanto OpenAPI como AsyncAPI — no
necesitas instalar nada adicional.

### Ejercicio Tema 4: Explorar AsyncAPI Studio

#### Paso 1 — Ver el archivo AsyncAPI del lab

Abre `payments-api/asyncapi/payments-events.yaml`. Describe 3 topics de Kafka:

```yaml
channels:
  payment.created:      # emitido al crear un pago
  payment.processed:    # emitido al aprobar o rechazar
  account.updated:      # emitido al cambiar saldo de cuenta
```

#### Paso 2 — Verificar el tipo en `catalog-info.yaml`

```yaml
# payments-api/catalog-info.yaml
spec:
  type: asyncapi          # ← diferencia clave vs openapi
  definition:
    $text: ./asyncapi/payments-events.yaml
```

#### Paso 3 — Ver AsyncAPI Studio en Backstage

1. **Catalog → APIs → payments-asyncapi**
2. Pestaña **Definition** → AsyncAPI Studio
3. Explora los channels y sus schemas de payload
4. Pestaña **Relations** → verifica Provider y Consumer

#### Paso 4 — Actividad: Agregar un nuevo channel

En tu repo local, abre `payments-api/asyncapi/payments-events.yaml` y agrega
al final de la sección `channels`:

```yaml
  payment.refunded:
    description: Emitido cuando un pago es reembolsado al origen
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

Haz commit y push → el nuevo channel aparece en AsyncAPI Studio sin re-registrar.

✅ **Checkpoint:** AsyncAPI Studio muestra 4 channels con sus schemas.

---

## Lab Nº 4 — Integración completa en Backstage (2 horas)

---

### FASE 0 — Prerequisitos (10 min)

Antes de continuar, confirma:

- [ ] Repo `lab-04-backstage` creado en GitHub y con push hecho
- [ ] `apiDocsPlugin` importado desde `@backstage/plugin-api-docs/alpha` en `App.tsx`
- [ ] `apiDocsPlugin` agregado al array `features` en `App.tsx`
- [ ] Backstage reiniciado con `yarn dev`
- [ ] El menú lateral muestra **"APIs"**

Si no ves "APIs" en el menú, **no avances** — revisa el Tema 1 primero.

---

### FASE 1 — Instalar dependencias de los servicios (15 min)

```bash
# Desde la carpeta lab-04/

# payments-api (Node.js)
cd payments-api && npm install && cd ..

# accounts-api (Python)
cd accounts-api && pip install -r requirements.txt && cd ..

# notifications-service (Node.js)
cd notifications-service && npm install && cd ..
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

---

### FASE 3 — Registrar entidades en Backstage desde GitHub (20 min)

#### Paso 1 — Registrar el sistema completo de un golpe

1. Abre Backstage: `http://localhost:3000`
2. Ve a **Catalog → + Register Existing Component**
3. Ingresa la URL de GitHub (reemplaza `TU_USUARIO`):
   ```
   https://github.com/TU_USUARIO/lab-04-backstage/blob/main/all-components.yaml
   ```
4. Clic en **Analyze**
5. Backstage detectará las 7 entidades — clic en **Import**

#### Paso 2 — Verificar el registro

| Entidad | Tipo | Dónde verificar |
|---------|------|-----------------|
| `fintech-rapida` | System | Catalog → Systems |
| `payments-api` | Component | Catalog → Components |
| `accounts-api` | Component | Catalog → Components |
| `notifications-service` | Component | Catalog → Components |
| `payments-openapi` | API | Catalog → APIs |
| `accounts-openapi` | API | Catalog → APIs |
| `payments-asyncapi` | API | Catalog → APIs |

---

### FASE 4 — Explorar el API Catalog (20 min)

#### 4.1 Swagger UI de payments-api

1. **Catalog → APIs → payments-openapi → Definition**
2. Ejecuta directamente desde Backstage:
   - Expande `POST /payments` → **Try it out**
   - Body de ejemplo:
     ```json
     {
       "amount": 350.00,
       "currency": "PEN",
       "sourceAccountId": "acct-origen-01",
       "destinationAccountId": "acct-destino-01",
       "description": "Prueba desde Backstage"
     }
     ```
   - **Execute** → copia el `paymentId` de la respuesta
3. Expande `GET /payments/{paymentId}` → pega el ID → **Execute**

#### 4.2 Swagger UI de accounts-api

1. **Catalog → APIs → accounts-openapi → Definition**
2. Explora los endpoints de cuentas desde Backstage

Alternativa nativa de FastAPI:
```
http://localhost:3001/docs
```

#### 4.3 AsyncAPI Studio de payments-asyncapi

1. **Catalog → APIs → payments-asyncapi → Definition**
2. Explora los 3 channels: `payment.created`, `payment.processed`, `account.updated`
3. Pestaña **Relations** → verifica:
   - `payments-api` como **Provider**
   - `notifications-service` como **Consumer**

#### 4.4 Grafo del sistema

1. **Catalog → Systems → fintech-rapida → Diagram**
2. El grafo debe mostrar:

```
[payments-api] ──providesApis──► [payments-openapi]
[payments-api] ──providesApis──► [payments-asyncapi]
[accounts-api] ──providesApis──► [accounts-openapi]
[notifications-service] ──consumesApis──► [payments-asyncapi]
```

---

### FASE 5 — TechDocs (15 min)

#### Paso 1 — Verificar configuración en `app-config.yaml`

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

1. **Catalog → Components → payments-api → Docs**
2. Debes ver: `Home` y `Architecture`

#### Paso 4 — Actividad: Agregar página y hacer push

```bash
# Crear nueva página
cat > payments-api/docs/errores.md << 'EOF'
# Errores comunes

| Código | Mensaje                   | Causa                             |
|--------|---------------------------|-----------------------------------|
| 400    | amount debe ser mayor a 0 | El campo amount es 0 o negativo   |
| 400    | currency inválido          | Usar solo: PEN, USD, EUR          |
| 404    | Pago no encontrado        | El paymentId no existe            |
| 409    | No se puede cancelar      | El pago no está en estado created |
EOF

# Actualizar mkdocs.yml para incluirla
# (edita el archivo y agrega "  - Errores: errores.md" en el nav)

# Subir cambios
git add .
git commit -m "docs: agregar página de errores comunes"
git push
```

Backstage hace refresh automático del catálogo cada cierto tiempo.
Para forzarlo: abre la entidad → menú de 3 puntos → **Schedule entity refresh**.

---

### FASE 6 — Pruebas end-to-end con curl (20 min)

```bash
# 1. Crear cuenta origen en accounts-api
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
echo "Pago creado:    $PAYMENT_ID"

# 4. Consultar el pago
echo "--- Detalle del pago ---"
curl -s http://localhost:3000/payments/$PAYMENT_ID | python3 -m json.tool

# 5. Listar pagos en estado created
echo "--- Pagos pendientes ---"
curl -s "http://localhost:3000/payments?status=created" | python3 -m json.tool

# 6. Cancelar el pago
echo "--- Cancelando el pago ---"
curl -s -X PATCH http://localhost:3000/payments/$PAYMENT_ID/cancel | python3 -m json.tool
```

**Observa en el terminal de `payments-api`:**

```
📋 [KAFKA EVENT SIMULADO]
   Topic   : payment.created
   Payload : { "paymentId": "...", "amount": 250, ... }
```

**Observa en el terminal de `notifications-service`:**

```
📨 Evento recibido [payment.created]
   → [NOTIFICACIÓN ENVIADA]
     Mensaje : Tu pago de 250 PEN está siendo procesado...
```

---

### FASE 7 — Validación final (20 min)

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
nada en la UI.

**La spec OpenAPI muestra el YAML en texto plano en vez de Swagger UI**
→ Mismo problema: el plugin no cargó. Verifica el import `/alpha` y reinicia.

**Error al registrar: "Unable to read url... not allowed"**
→ Estás usando una URL de `localhost`. Usa la URL de GitHub en su lugar:
```
https://github.com/TU_USUARIO/lab-04-backstage/blob/main/all-components.yaml
```
Si necesitas usar localhost de todas formas, agrega en `app-config.yaml`:
```yaml
backend:
  reading:
    allow:
      - host: localhost:8080
```

**Error: "group:default/guests not found"**
→ Cambia el `owner` en todos los `catalog-info.yaml`:
```yaml
spec:
  owner: user:default/guest   # en lugar de group:default/guests
```

**GitHub devuelve 404 al hacer Analyze**
→ Verifica que el repo es público, o que tienes el token configurado:
```yaml
# app-config.yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

**accounts-api: "No module named 'fastapi'"**
→ Instala desde la carpeta correcta:
```bash
cd lab-04/accounts-api && pip install -r requirements.txt
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

**Los cambios en el repo no se reflejan en Backstage**
→ Backstage refresca el catálogo periódicamente. Para forzar el refresh:
abre la entidad → menú de 3 puntos (⋮) → **Schedule entity refresh**.

---

## Referencias

- [Backstage New Frontend System](https://backstage.io/docs/frontend-system/)
- [plugin-api-docs](https://backstage.io/docs/reference/plugin-api-docs/)
- [OpenAPI Specification 3.0](https://swagger.io/specification/)
- [AsyncAPI Specification 2.6](https://www.asyncapi.com/docs/reference/specification/v2.6.0)
- [TechDocs — Backstage](https://backstage.io/docs/features/techdocs/)
- [catalog-info.yaml — Descriptor Format](https://backstage.io/docs/features/software-catalog/descriptor-format)
