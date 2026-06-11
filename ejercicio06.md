# Sesión 4: API Catalog & OpenAPI Integration

> **Duración total estimada:** ~4 horas (ejercicios por tema ~30 min c/u + laboratorio principal ~2 horas)  
> **Nivel:** Intermedio – se asume Backstage corriendo localmente con el Software Catalog activo.

---

## Tabla de Contenidos

- [Arquitectura del ecosistema de APIs](#arquitectura-del-ecosistema-de-apis)
- [Tema 1 – Documentación Viva con OpenAPI/Swagger](#tema-1--documentación-viva-con-openapiswagger)
- [Tema 2 – API Catalog: Centralizar contratos de APIs](#tema-2--api-catalog-centralizar-contratos-de-apis)
- [Tema 3 – TechDocs: Docs-like-code](#tema-3--techdocs-docs-like-code)
- [Tema 4 – Plugin AsyncAPI para eventos Kafka/RabbitMQ](#tema-4--plugin-asyncapi-para-eventos-kafkarabbitmq)
- [Lab Nº 4 – Registrar APIs en Backstage (2 horas)](#lab-nº-4--registrar-apis-en-backstage-2-horas)
- [Código fuente de la aplicación de ejemplo](#código-fuente-de-la-aplicación-de-ejemplo)

---

## Arquitectura del ecosistema de APIs

El sistema de ejemplo que usaremos durante toda la sesión es **"FinTech Rápida S.A."**, una plataforma de pagos digital con los siguientes servicios:

```
┌─────────────────────────────────────────────────────────────────┐
│                    FINTECH RÁPIDA S.A.                          │
│                                                                 │
│  ┌──────────────┐    REST/OpenAPI    ┌──────────────────────┐   │
│  │  API Gateway │◄──────────────────►│  payments-api (3000) │   │
│  │  (Kong/NGINX)│                    │  (Node.js + Express) │   │
│  └──────┬───────┘                    └─────────┬────────────┘   │
│         │                                      │                │
│         │ REST/OpenAPI              ┌──────────▼────────────┐   │
│         └──────────────────────────►|  accounts-api (3001)  │   │
│                                     │  (Python + FastAPI)   │   │
│                                     └─────────┬─────────────┘   │
│                                               │                 │
│                               Kafka Topics    │                 │
│                     ┌─────────────────────────▼──────────────┐  │
│                     │        Event Bus (Kafka)               │  │
│                     │  ● payment.created                     │  │
│                     │   ● payment.processed                  │  │
│                     │  ● account.updated                     │  │
│                     └──────────────┬─────────────────────────┘  │
│                                    │ AsyncAPI                   │
│                     ┌──────────────▼─────────────────────────┐  │
│                     │    notifications-service (3002)        │  │
│                     │    (Node.js – Consumer Kafka)          │  │
│                     └────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Relación entre entidades en Backstage:**

```
System: fintech-rapida
  ├── Component: payments-api        (providesApis: [payments-openapi])
  ├── Component: accounts-api        (providesApis: [accounts-openapi])
  ├── Component: notifications-service (consumesApis: [payments-asyncapi])
  ├── API: payments-openapi          (type: openapi)
  ├── API: accounts-openapi          (type: openapi)
  └── API: payments-asyncapi         (type: asyncapi)
```

---

## Tema 1 – Documentación Viva con OpenAPI/Swagger

### ¿Qué es "Documentación Viva"?

La integración del plugin `@backstage/plugin-api-docs` permite que Backstage renderice specs de OpenAPI/Swagger directamente desde el catálogo. Al vincular la spec al componente, la documentación **vive junto al código** y se actualiza automáticamente.

### Requisitos previos

Verifica que tienes instalado el plugin en tu Backstage:

```bash
# En el directorio de tu Backstage
yarn --cwd packages/app add @backstage/plugin-api-docs
```

### Ejercicio Tema 1 – Visualizar Swagger UI en Backstage (~30 min)

**Objetivo:** Conectar la spec OpenAPI de `payments-api` al catálogo y visualizarla en Backstage.

#### Paso 1: Registrar el plugin en `packages/app/src/App.tsx`

Verifica (o agrega) estas líneas en tu `App.tsx`:

```tsx
import { ApiExplorerPage } from '@backstage/plugin-api-docs';

// Dentro del elemento <FlatRoutes>:
<Route path="/api-docs" element={<ApiExplorerPage />} />
```

#### Paso 2: Agregar ApiDefinitionCard al EntityPage

En `packages/app/src/components/catalog/EntityPage.tsx`, busca la sección `apiPage` y asegúrate de que incluya:

```tsx
import {
  ApiDefinitionCard,
  ConsumedApisCard,
  ProvidedApisCard,
} from '@backstage/plugin-api-docs';

// Dentro del componente para APIs:
const apiPage = (
  <EntityLayout>
    <EntityLayout.Route path="/" title="Overview">
      <Grid container spacing={3}>
        <Grid item xs={12} md={6}>
          <EntityAboutCard />
        </Grid>
        <Grid item xs={12} md={6}>
          <EntityCatalogGraphCard variant="gridItem" height={400} />
        </Grid>
        <Grid item xs={12}>
          <ApiDefinitionCard />
        </Grid>
      </Grid>
    </EntityLayout.Route>
  </EntityLayout>
);

// En el componentPage, agrega las tarjetas de APIs:
const componentPage = (
  <EntityLayout>
    {/* ...rutas existentes... */}
    <EntityLayout.Route path="/api" title="API">
      <Grid container spacing={3} alignItems="stretch">
        <Grid item xs={12} md={6}>
          <ProvidedApisCard />
        </Grid>
        <Grid item xs={12} md={6}>
          <ConsumedApisCard />
        </Grid>
      </Grid>
    </EntityLayout.Route>
  </EntityLayout>
);
```

#### Paso 3: Crear el archivo `catalog-info.yaml` para payments-api

Crea la siguiente estructura en tu repo local (o usa el que está en el Lab):

```yaml
# catalog-info.yaml  ← en la raíz del repo payments-api
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payments-openapi
  title: Payments API
  description: API REST para procesamiento de pagos de FinTech Rápida S.A.
  tags:
    - payments
    - rest
    - openapi
  links:
    - url: http://localhost:3000/api-docs
      title: Swagger UI (local)
spec:
  type: openapi
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  definition:
    $text: ./openapi/payments.yaml
```

#### Paso 4: Registrar en Backstage

1. Abre tu Backstage local: `http://localhost:3000`
2. Ve a **Catalog → + Register Existing Component**
3. Ingresa la URL de tu `catalog-info.yaml` (puede ser una ruta local con file:// si usas un location estático, o la URL de GitHub)
4. Verifica que aparezca la entidad tipo **API** con la spec Swagger renderizada

✅ **Checkpoint:** Deberías ver el Swagger UI interactivo dentro de la entidad `payments-openapi` en Backstage.

---

## Tema 2 – API Catalog: Centralizar contratos de APIs

### Concepto

El **API Catalog** de Backstage centraliza todos los contratos de APIs de tu organización. Permite:

- Descubrir qué APIs existen y quién las provee
- Ver qué componentes consumen cada API
- Hacer las specs testeables directamente desde el portal

### Relaciones `providesApis` / `consumesApis`

La clave es declarar las relaciones en el `catalog-info.yaml` de cada **Component**:

```yaml
# Component que PROVEE una API
spec:
  providesApis:
    - payments-openapi

# Component que CONSUME una API
spec:
  consumesApis:
    - payments-openapi
    - accounts-openapi
```

### Ejercicio Tema 2 – Mapear el ecosistema completo (~30 min)

**Objetivo:** Registrar el System `fintech-rapida` con sus tres componentes y sus relaciones de APIs.

#### Paso 1: Crear `all-components.yaml` (catalog unificado)

Crea un archivo con múltiples entidades separadas por `---`:

```yaml
# all-components.yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: fintech-rapida
  title: FinTech Rápida S.A.
  description: Plataforma de pagos digitales
  tags:
    - fintech
    - payments
spec:
  owner: group:platform-team
---
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payments-api
  title: Payments API Service
  description: Servicio REST para procesamiento de pagos
  annotations:
    backstage.io/techdocs-ref: dir:.
  tags:
    - node
    - rest
    - payments
spec:
  type: service
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  providesApis:
    - payments-openapi
---
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: accounts-api
  title: Accounts API Service
  description: Servicio REST para gestión de cuentas
  annotations:
    backstage.io/techdocs-ref: dir:.
  tags:
    - python
    - fastapi
    - accounts
spec:
  type: service
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  providesApis:
    - accounts-openapi
---
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: notifications-service
  title: Notifications Service
  description: Servicio de notificaciones vía Kafka
  tags:
    - node
    - kafka
    - consumer
spec:
  type: service
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  consumesApis:
    - payments-asyncapi
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payments-openapi
  title: Payments OpenAPI
  description: Contrato REST del servicio de pagos
  tags:
    - openapi
    - payments
spec:
  type: openapi
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  definition:
    $text: ./openapi/payments.yaml
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: accounts-openapi
  title: Accounts OpenAPI
  description: Contrato REST del servicio de cuentas
  tags:
    - openapi
    - accounts
spec:
  type: openapi
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  definition:
    $text: ./openapi/accounts.yaml
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payments-asyncapi
  title: Payments AsyncAPI (Kafka Events)
  description: Contrato de eventos Kafka del servicio de pagos
  tags:
    - asyncapi
    - kafka
    - events
spec:
  type: asyncapi
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  definition:
    $text: ./asyncapi/payments-events.yaml
```

#### Paso 2: Registrar y verificar

1. Registra el `all-components.yaml` en Backstage
2. Ve a **Catalog → Systems → fintech-rapida**
3. Haz clic en la pestaña **Diagram** para ver el grafo de relaciones

✅ **Checkpoint:** El grafo debe mostrar los 3 componentes, los 3 APIs, y las flechas de `providesApis`/`consumesApis`.

---

## Tema 3 – TechDocs: Docs-like-code

### ¿Qué es TechDocs?

TechDocs es el sistema de documentación técnica de Backstage basado en el concepto **"docs like code"**:

- La documentación se escribe en **Markdown** dentro del repositorio
- Se versiona junto al código fuente en Git
- Backstage la renderiza automáticamente con **MkDocs**

### Estructura requerida

```
mi-servicio/
├── catalog-info.yaml          ← annotation: backstage.io/techdocs-ref: dir:.
├── mkdocs.yml                 ← configuración de MkDocs
└── docs/
    ├── index.md               ← página principal
    ├── architecture.md
    └── api-reference.md
```

### Ejercicio Tema 3 – Publicar docs de payments-api (~30 min)

#### Paso 1: Crear `mkdocs.yml`

```yaml
# mkdocs.yml
site_name: Payments API
site_description: Documentación técnica del servicio de pagos
nav:
  - Home: index.md
  - Architecture: architecture.md
  - API Reference: api-reference.md
  - Changelog: changelog.md
plugins:
  - techdocs-core
```

#### Paso 2: Crear `docs/index.md`

```markdown
# Payments API

Servicio REST encargado del procesamiento de pagos en FinTech Rápida S.A.

## Stack tecnológico

| Tecnología | Versión | Propósito        |
|------------|---------|------------------|
| Node.js    | 20 LTS  | Runtime          |
| Express    | 4.x     | Framework HTTP   |
| Kafka      | 3.x     | Mensajería       |
| PostgreSQL | 15      | Base de datos    |

## Endpoints principales

- `POST /payments` – Crear un nuevo pago
- `GET /payments/{id}` – Obtener detalle de pago
- `GET /payments` – Listar pagos con filtros
- `PATCH /payments/{id}/cancel` – Cancelar un pago

## Quickstart local

```bash
npm install
npm run dev
# Servidor en http://localhost:3000
```
```

#### Paso 3: Crear `docs/architecture.md`

```markdown
# Arquitectura

## Diagrama de flujo de un pago

```
Cliente → API Gateway → payments-api → PostgreSQL
                             ↓
                        Kafka Producer
                             ↓
                    payment.created (topic)
                             ↓
                   notifications-service
```

## Decisiones de diseño

### ¿Por qué Kafka?
Los pagos son eventos irreversibles. Kafka nos permite:
- Replay de eventos en caso de falla del consumidor
- Múltiples consumidores del mismo topic
- Retención configurable de mensajes

### ¿Por qué PostgreSQL?
- ACID compliance para operaciones financieras
- Soporte nativo para UUIDs
- JSON columns para metadata flexible
```

#### Paso 4: Verificar la annotation en `catalog-info.yaml`

```yaml
metadata:
  annotations:
    backstage.io/techdocs-ref: dir:.   # ← Esta annotation es OBLIGATORIA
```

#### Paso 5: Generar y visualizar

```bash
# Instalar techdocs-cli si no lo tienes
npm install -g @techdocs/cli

# Generar la documentación localmente para preview
techdocs-cli serve
```

Luego en Backstage, navega al componente `payments-api` → pestaña **Docs**.

✅ **Checkpoint:** La documentación Markdown se renderiza como un site navegable dentro de Backstage.

---

## Tema 4 – Plugin AsyncAPI para eventos Kafka/RabbitMQ

### ¿Qué es AsyncAPI?

AsyncAPI es el estándar equivalente a OpenAPI pero para **comunicación asíncrona** (eventos, mensajes). Es ideal para describir:

- Topics de Kafka
- Exchanges de RabbitMQ
- Colas de AWS SQS/SNS

### Instalar el plugin AsyncAPI

```bash
# En packages/app
yarn --cwd packages/app add @asyncapi/react-component
```

Backstage soporta AsyncAPI de forma nativa a través del plugin `@backstage/plugin-api-docs` cuando la entidad API tiene `spec.type: asyncapi`.

### Ejercicio Tema 4 – Registrar eventos Kafka con AsyncAPI (~30 min)

#### Paso 1: Crear `asyncapi/payments-events.yaml`

```yaml
asyncapi: 2.6.0
info:
  title: Payments Events API
  version: 1.0.0
  description: |
    Eventos de dominio emitidos por el servicio de pagos de FinTech Rápida S.A.
  contact:
    name: Backend Team
    email: backend@fintechrapida.pe
  license:
    name: Internal

servers:
  production:
    url: kafka.fintechrapida.pe:9092
    protocol: kafka
    description: Broker Kafka de producción
  local:
    url: localhost:9092
    protocol: kafka
    description: Broker Kafka local (Docker)

channels:
  payment.created:
    description: Emitido cuando un nuevo pago es creado exitosamente
    publish:
      operationId: onPaymentCreated
      summary: Nuevo pago registrado
      message:
        $ref: '#/components/messages/PaymentCreated'
    subscribe:
      operationId: consumePaymentCreated
      summary: Consumir evento de pago creado

  payment.processed:
    description: Emitido cuando un pago ha sido procesado (aprobado o rechazado)
    publish:
      operationId: onPaymentProcessed
      summary: Pago procesado
      message:
        $ref: '#/components/messages/PaymentProcessed'

  account.updated:
    description: Emitido cuando el saldo de una cuenta es actualizado
    publish:
      operationId: onAccountUpdated
      summary: Cuenta actualizada
      message:
        $ref: '#/components/messages/AccountUpdated'

components:
  messages:
    PaymentCreated:
      name: PaymentCreated
      title: Pago Creado
      summary: Un nuevo pago fue registrado en el sistema
      contentType: application/json
      payload:
        $ref: '#/components/schemas/PaymentCreatedPayload'

    PaymentProcessed:
      name: PaymentProcessed
      title: Pago Procesado
      summary: Un pago fue aprobado o rechazado
      contentType: application/json
      payload:
        $ref: '#/components/schemas/PaymentProcessedPayload'

    AccountUpdated:
      name: AccountUpdated
      title: Cuenta Actualizada
      contentType: application/json
      payload:
        $ref: '#/components/schemas/AccountUpdatedPayload'

  schemas:
    PaymentCreatedPayload:
      type: object
      required:
        - paymentId
        - amount
        - currency
        - sourceAccountId
        - destinationAccountId
        - createdAt
      properties:
        paymentId:
          type: string
          format: uuid
          example: "550e8400-e29b-41d4-a716-446655440000"
        amount:
          type: number
          format: double
          minimum: 0.01
          example: 250.00
        currency:
          type: string
          enum: [PEN, USD, EUR]
          example: PEN
        sourceAccountId:
          type: string
          format: uuid
        destinationAccountId:
          type: string
          format: uuid
        description:
          type: string
          maxLength: 255
          example: "Pago de servicios"
        createdAt:
          type: string
          format: date-time

    PaymentProcessedPayload:
      type: object
      required:
        - paymentId
        - status
        - processedAt
      properties:
        paymentId:
          type: string
          format: uuid
        status:
          type: string
          enum: [approved, rejected, pending_review]
        rejectionReason:
          type: string
          nullable: true
          example: "Saldo insuficiente"
        processedAt:
          type: string
          format: date-time

    AccountUpdatedPayload:
      type: object
      required:
        - accountId
        - previousBalance
        - newBalance
        - updatedAt
      properties:
        accountId:
          type: string
          format: uuid
        previousBalance:
          type: number
          format: double
        newBalance:
          type: number
          format: double
        currency:
          type: string
          enum: [PEN, USD, EUR]
        updatedAt:
          type: string
          format: date-time
```

#### Paso 2: Registrar en Backstage

La entidad API ya está declarada en `all-components.yaml` del Tema 2 con `spec.type: asyncapi`. Solo asegúrate que el path `./asyncapi/payments-events.yaml` sea accesible.

✅ **Checkpoint:** En Backstage, la entidad `payments-asyncapi` debe mostrar el AsyncAPI Studio con los topics `payment.created`, `payment.processed` y `account.updated` interactivos.

---

## Lab Nº 4 – Registrar APIs en Backstage (2 horas)

### Objetivo del Lab

Construir e integrar el ecosistema completo de **FinTech Rápida S.A.** en tu Backstage local, incluyendo:

1. El código fuente funcional de los tres servicios
2. Sus specs OpenAPI y AsyncAPI completas
3. TechDocs de cada servicio
4. Todo el grafo de relaciones visible en Backstage

### Estructura de directorios del Lab

```
lab-04/
├── README.md                          ← (este archivo)
├── all-components.yaml               ← registro unificado
│
├── payments-api/
│   ├── catalog-info.yaml
│   ├── mkdocs.yml
│   ├── src/
│   │   ├── index.js
│   │   ├── routes/payments.js
│   │   └── kafka/producer.js
│   ├── openapi/
│   │   └── payments.yaml
│   ├── asyncapi/
│   │   └── payments-events.yaml
│   └── docs/
│       ├── index.md
│       └── architecture.md
│
├── accounts-api/
│   ├── catalog-info.yaml
│   ├── mkdocs.yml
│   ├── src/
│   │   └── main.py
│   ├── openapi/
│   │   └── accounts.yaml
│   └── docs/
│       └── index.md
│
└── notifications-service/
    ├── catalog-info.yaml
    ├── src/
    │   ├── index.js
    │   └── kafka/consumer.js
    └── docs/
        └── index.md
```

---

## Código fuente de la aplicación de ejemplo

### payments-api (Node.js + Express)

#### `payments-api/src/index.js`

```javascript
const express = require('express');
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');
const paymentsRouter = require('./routes/payments');

const app = express();
app.use(express.json());

// Swagger UI
const swaggerDoc = YAML.load('./openapi/payments.yaml');
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDoc));

// Routes
app.use('/payments', paymentsRouter);

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', service: 'payments-api', version: '1.0.0' });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`payments-api running on http://localhost:${PORT}`);
  console.log(`Swagger UI: http://localhost:${PORT}/api-docs`);
});
```

#### `payments-api/src/routes/payments.js`

```javascript
const express = require('express');
const { v4: uuidv4 } = require('uuid');
const { producePaymentEvent } = require('../kafka/producer');

const router = express.Router();

// In-memory store para el lab
const payments = new Map();

/**
 * POST /payments
 * Crear un nuevo pago
 */
router.post('/', async (req, res) => {
  const { amount, currency, sourceAccountId, destinationAccountId, description } = req.body;

  // Validaciones básicas
  if (!amount || amount <= 0) {
    return res.status(400).json({ error: 'amount debe ser mayor a 0' });
  }
  if (!['PEN', 'USD', 'EUR'].includes(currency)) {
    return res.status(400).json({ error: 'currency inválido. Use: PEN, USD, EUR' });
  }
  if (!sourceAccountId || !destinationAccountId) {
    return res.status(400).json({ error: 'sourceAccountId y destinationAccountId son requeridos' });
  }

  const payment = {
    paymentId: uuidv4(),
    amount,
    currency,
    sourceAccountId,
    destinationAccountId,
    description: description || '',
    status: 'created',
    createdAt: new Date().toISOString(),
  };

  payments.set(payment.paymentId, payment);

  // Emitir evento a Kafka (o loguear si no hay Kafka)
  await producePaymentEvent('payment.created', payment);

  return res.status(201).json(payment);
});

/**
 * GET /payments/:id
 * Obtener detalle de un pago
 */
router.get('/:id', (req, res) => {
  const payment = payments.get(req.params.id);
  if (!payment) {
    return res.status(404).json({ error: `Pago ${req.params.id} no encontrado` });
  }
  return res.json(payment);
});

/**
 * GET /payments
 * Listar pagos con filtros opcionales
 */
router.get('/', (req, res) => {
  const { status, currency, limit = 20, offset = 0 } = req.query;
  let results = Array.from(payments.values());

  if (status) results = results.filter(p => p.status === status);
  if (currency) results = results.filter(p => p.currency === currency);

  const paginated = results.slice(Number(offset), Number(offset) + Number(limit));

  return res.json({
    total: results.length,
    limit: Number(limit),
    offset: Number(offset),
    data: paginated,
  });
});

/**
 * PATCH /payments/:id/cancel
 * Cancelar un pago pendiente
 */
router.patch('/:id/cancel', async (req, res) => {
  const payment = payments.get(req.params.id);
  if (!payment) {
    return res.status(404).json({ error: `Pago ${req.params.id} no encontrado` });
  }
  if (payment.status !== 'created') {
    return res.status(409).json({ error: `No se puede cancelar un pago en estado: ${payment.status}` });
  }

  payment.status = 'cancelled';
  payment.cancelledAt = new Date().toISOString();
  payments.set(payment.paymentId, payment);

  await producePaymentEvent('payment.processed', {
    paymentId: payment.paymentId,
    status: 'rejected',
    rejectionReason: 'Cancelado por el usuario',
    processedAt: payment.cancelledAt,
  });

  return res.json(payment);
});

module.exports = router;
```

#### `payments-api/src/kafka/producer.js`

```javascript
/**
 * Kafka Producer para payments-api
 * Si no hay Kafka disponible (lab local), loguea el evento en consola.
 */

let kafka = null;
let producer = null;

async function initKafka() {
  try {
    const { Kafka } = require('kafkajs');
    kafka = new Kafka({
      clientId: 'payments-api',
      brokers: [process.env.KAFKA_BROKER || 'localhost:9092'],
    });
    producer = kafka.producer();
    await producer.connect();
    console.log('✅ Kafka producer conectado');
  } catch (err) {
    console.warn('⚠️  Kafka no disponible, los eventos se loguearán en consola');
    producer = null;
  }
}

async function producePaymentEvent(topic, payload) {
  const message = JSON.stringify(payload);

  if (producer) {
    try {
      await producer.send({
        topic,
        messages: [{ key: payload.paymentId, value: message }],
      });
      console.log(`📤 Evento enviado a Kafka [${topic}]:`, payload.paymentId);
    } catch (err) {
      console.error(`❌ Error enviando a Kafka [${topic}]:`, err.message);
    }
  } else {
    // Modo fallback: solo log
    console.log(`📋 [KAFKA EVENT - ${topic}]`, JSON.stringify(payload, null, 2));
  }
}

// Inicializar al cargar el módulo
initKafka();

module.exports = { producePaymentEvent };
```

#### `payments-api/openapi/payments.yaml`

```yaml
openapi: 3.0.3
info:
  title: Payments API
  description: |
    API REST para procesamiento de pagos de **FinTech Rápida S.A.**
    
    Permite crear, consultar y cancelar pagos entre cuentas registradas.
    Los pagos exitosos generan eventos en Kafka (ver AsyncAPI spec).
  version: 1.0.0
  contact:
    name: Backend Team
    email: backend@fintechrapida.pe
  license:
    name: Internal Use Only

servers:
  - url: http://localhost:3000
    description: Servidor local de desarrollo

tags:
  - name: payments
    description: Operaciones de pago

paths:
  /payments:
    post:
      tags: [payments]
      summary: Crear un nuevo pago
      operationId: createPayment
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreatePaymentRequest'
            example:
              amount: 250.00
              currency: PEN
              sourceAccountId: "acct-001"
              destinationAccountId: "acct-002"
              description: "Pago de servicios mensuales"
      responses:
        '201':
          description: Pago creado exitosamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Payment'
        '400':
          description: Datos de entrada inválidos
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '500':
          description: Error interno del servidor

    get:
      tags: [payments]
      summary: Listar pagos
      operationId: listPayments
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [created, approved, rejected, cancelled]
        - name: currency
          in: query
          schema:
            type: string
            enum: [PEN, USD, EUR]
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            minimum: 1
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: Lista de pagos
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentList'

  /payments/{paymentId}:
    get:
      tags: [payments]
      summary: Obtener detalle de un pago
      operationId: getPayment
      parameters:
        - name: paymentId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Detalle del pago
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Payment'
        '404':
          description: Pago no encontrado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

  /payments/{paymentId}/cancel:
    patch:
      tags: [payments]
      summary: Cancelar un pago
      operationId: cancelPayment
      parameters:
        - name: paymentId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Pago cancelado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Payment'
        '404':
          description: Pago no encontrado
        '409':
          description: El pago no puede ser cancelado en su estado actual

  /health:
    get:
      tags: [payments]
      summary: Health check
      operationId: healthCheck
      responses:
        '200':
          description: Servicio saludable
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: ok
                  service:
                    type: string
                    example: payments-api
                  version:
                    type: string
                    example: 1.0.0

components:
  schemas:
    CreatePaymentRequest:
      type: object
      required:
        - amount
        - currency
        - sourceAccountId
        - destinationAccountId
      properties:
        amount:
          type: number
          format: double
          minimum: 0.01
          description: Monto del pago
          example: 250.00
        currency:
          type: string
          enum: [PEN, USD, EUR]
          description: Moneda del pago
          example: PEN
        sourceAccountId:
          type: string
          description: ID de la cuenta origen
          example: "acct-001"
        destinationAccountId:
          type: string
          description: ID de la cuenta destino
          example: "acct-002"
        description:
          type: string
          maxLength: 255
          description: Descripción opcional del pago
          example: "Pago de servicios mensuales"

    Payment:
      type: object
      properties:
        paymentId:
          type: string
          format: uuid
        amount:
          type: number
          format: double
        currency:
          type: string
          enum: [PEN, USD, EUR]
        sourceAccountId:
          type: string
        destinationAccountId:
          type: string
        description:
          type: string
        status:
          type: string
          enum: [created, approved, rejected, cancelled]
        createdAt:
          type: string
          format: date-time
        cancelledAt:
          type: string
          format: date-time
          nullable: true

    PaymentList:
      type: object
      properties:
        total:
          type: integer
        limit:
          type: integer
        offset:
          type: integer
        data:
          type: array
          items:
            $ref: '#/components/schemas/Payment'

    Error:
      type: object
      properties:
        error:
          type: string
          example: "Descripción del error"
```

---

### accounts-api (Python + FastAPI)

#### `accounts-api/src/main.py`

```python
"""
accounts-api — FastAPI
Gestión de cuentas para FinTech Rápida S.A.
"""
from fastapi import FastAPI, HTTPException
from fastapi.openapi.utils import get_openapi
from pydantic import BaseModel, Field
from typing import Optional, List
from uuid import uuid4, UUID
from datetime import datetime
from enum import Enum

app = FastAPI(
    title="Accounts API",
    description="API REST para gestión de cuentas bancarias de FinTech Rápida S.A.",
    version="1.0.0",
    contact={"name": "Backend Team", "email": "backend@fintechrapida.pe"},
)

# ── Models ──────────────────────────────────────────────────────────────────

class Currency(str, Enum):
    PEN = "PEN"
    USD = "USD"
    EUR = "EUR"

class AccountStatus(str, Enum):
    active = "active"
    suspended = "suspended"
    closed = "closed"

class AccountType(str, Enum):
    savings = "savings"
    checking = "checking"
    investment = "investment"

class CreateAccountRequest(BaseModel):
    ownerId: str = Field(..., description="ID del propietario de la cuenta")
    accountType: AccountType = Field(..., description="Tipo de cuenta")
    currency: Currency = Field(default=Currency.PEN, description="Moneda principal")
    initialBalance: float = Field(default=0.0, ge=0, description="Saldo inicial")

class Account(BaseModel):
    accountId: str
    ownerId: str
    accountType: AccountType
    currency: Currency
    balance: float
    status: AccountStatus
    createdAt: str
    updatedAt: str

class AccountList(BaseModel):
    total: int
    data: List[Account]

# ── In-memory store ──────────────────────────────────────────────────────────

accounts: dict = {}

# ── Endpoints ────────────────────────────────────────────────────────────────

@app.post("/accounts", response_model=Account, status_code=201, tags=["accounts"])
def create_account(body: CreateAccountRequest):
    """Crear una nueva cuenta bancaria."""
    account = Account(
        accountId=str(uuid4()),
        ownerId=body.ownerId,
        accountType=body.accountType,
        currency=body.currency,
        balance=body.initialBalance,
        status=AccountStatus.active,
        createdAt=datetime.utcnow().isoformat(),
        updatedAt=datetime.utcnow().isoformat(),
    )
    accounts[account.accountId] = account.dict()
    return account

@app.get("/accounts/{account_id}", response_model=Account, tags=["accounts"])
def get_account(account_id: str):
    """Obtener detalle de una cuenta por ID."""
    acct = accounts.get(account_id)
    if not acct:
        raise HTTPException(status_code=404, detail=f"Cuenta {account_id} no encontrada")
    return acct

@app.get("/accounts", response_model=AccountList, tags=["accounts"])
def list_accounts(
    owner_id: Optional[str] = None,
    status: Optional[AccountStatus] = None,
    currency: Optional[Currency] = None,
    limit: int = 20,
    offset: int = 0,
):
    """Listar cuentas con filtros opcionales."""
    results = list(accounts.values())
    if owner_id:
        results = [a for a in results if a["ownerId"] == owner_id]
    if status:
        results = [a for a in results if a["status"] == status]
    if currency:
        results = [a for a in results if a["currency"] == currency]
    return {"total": len(results), "data": results[offset: offset + limit]}

@app.patch("/accounts/{account_id}/balance", response_model=Account, tags=["accounts"])
def update_balance(account_id: str, amount: float):
    """
    Actualizar el saldo de una cuenta (positivo = crédito, negativo = débito).
    Este endpoint es llamado internamente por payments-api.
    """
    acct = accounts.get(account_id)
    if not acct:
        raise HTTPException(status_code=404, detail=f"Cuenta {account_id} no encontrada")
    if acct["status"] != "active":
        raise HTTPException(status_code=409, detail="La cuenta no está activa")
    new_balance = acct["balance"] + amount
    if new_balance < 0:
        raise HTTPException(status_code=422, detail="Saldo insuficiente")
    acct["balance"] = new_balance
    acct["updatedAt"] = datetime.utcnow().isoformat()
    accounts[account_id] = acct
    return acct

@app.get("/health", tags=["system"])
def health():
    """Health check."""
    return {"status": "ok", "service": "accounts-api", "version": "1.0.0"}
```

#### `accounts-api/openapi/accounts.yaml`

```yaml
openapi: 3.0.3
info:
  title: Accounts API
  description: |
    API REST para gestión de cuentas bancarias de **FinTech Rápida S.A.**
    
    Permite crear cuentas, consultar saldos y actualizar balances.
    Esta API es consumida internamente por `payments-api`.
  version: 1.0.0
  contact:
    name: Backend Team
    email: backend@fintechrapida.pe

servers:
  - url: http://localhost:3001
    description: Servidor local de desarrollo

tags:
  - name: accounts
    description: Operaciones de cuenta
  - name: system
    description: Endpoints de sistema

paths:
  /accounts:
    post:
      tags: [accounts]
      summary: Crear cuenta
      operationId: createAccount
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateAccountRequest'
            example:
              ownerId: "user-123"
              accountType: "savings"
              currency: "PEN"
              initialBalance: 1000.00
      responses:
        '201':
          description: Cuenta creada
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Account'
        '422':
          description: Error de validación

    get:
      tags: [accounts]
      summary: Listar cuentas
      operationId: listAccounts
      parameters:
        - name: owner_id
          in: query
          schema:
            type: string
        - name: status
          in: query
          schema:
            type: string
            enum: [active, suspended, closed]
        - name: currency
          in: query
          schema:
            type: string
            enum: [PEN, USD, EUR]
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: Lista de cuentas
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AccountList'

  /accounts/{accountId}:
    get:
      tags: [accounts]
      summary: Obtener cuenta
      operationId: getAccount
      parameters:
        - name: accountId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Detalle de la cuenta
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Account'
        '404':
          description: Cuenta no encontrada

  /accounts/{accountId}/balance:
    patch:
      tags: [accounts]
      summary: Actualizar saldo
      operationId: updateBalance
      description: |
        Actualiza el saldo de una cuenta.
        - amount positivo → crédito
        - amount negativo → débito
      parameters:
        - name: accountId
          in: path
          required: true
          schema:
            type: string
        - name: amount
          in: query
          required: true
          schema:
            type: number
            format: double
      responses:
        '200':
          description: Saldo actualizado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Account'
        '404':
          description: Cuenta no encontrada
        '409':
          description: Cuenta no activa
        '422':
          description: Saldo insuficiente

  /health:
    get:
      tags: [system]
      summary: Health check
      operationId: healthCheck
      responses:
        '200':
          description: Servicio saludable

components:
  schemas:
    CreateAccountRequest:
      type: object
      required: [ownerId, accountType]
      properties:
        ownerId:
          type: string
          example: "user-123"
        accountType:
          type: string
          enum: [savings, checking, investment]
        currency:
          type: string
          enum: [PEN, USD, EUR]
          default: PEN
        initialBalance:
          type: number
          format: double
          minimum: 0
          default: 0.0

    Account:
      type: object
      properties:
        accountId:
          type: string
          format: uuid
        ownerId:
          type: string
        accountType:
          type: string
          enum: [savings, checking, investment]
        currency:
          type: string
          enum: [PEN, USD, EUR]
        balance:
          type: number
          format: double
        status:
          type: string
          enum: [active, suspended, closed]
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time

    AccountList:
      type: object
      properties:
        total:
          type: integer
        data:
          type: array
          items:
            $ref: '#/components/schemas/Account'
```

---

### notifications-service (Node.js – Consumer Kafka)

#### `notifications-service/src/index.js`

```javascript
const { consumePaymentEvents } = require('./kafka/consumer');

console.log('🔔 notifications-service iniciando...');
consumePaymentEvents();
```

#### `notifications-service/src/kafka/consumer.js`

```javascript
/**
 * Kafka Consumer para notifications-service
 * Consume eventos de payment.created y payment.processed
 * y simula el envío de notificaciones.
 */

async function consumePaymentEvents() {
  try {
    const { Kafka } = require('kafkajs');
    const kafka = new Kafka({
      clientId: 'notifications-service',
      brokers: [process.env.KAFKA_BROKER || 'localhost:9092'],
    });

    const consumer = kafka.consumer({ groupId: 'notifications-group' });
    await consumer.connect();
    await consumer.subscribe({ topics: ['payment.created', 'payment.processed'], fromBeginning: false });

    console.log('✅ Kafka consumer suscrito a: payment.created, payment.processed');

    await consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const payload = JSON.parse(message.value.toString());
        handleEvent(topic, payload);
      },
    });
  } catch (err) {
    console.warn('⚠️  Kafka no disponible. Ejecutando en modo simulación...');
    simulateEvents();
  }
}

function handleEvent(topic, payload) {
  switch (topic) {
    case 'payment.created':
      console.log(`📧 [NOTIFICATION] Nuevo pago recibido: ${payload.paymentId}`);
      console.log(`   Monto: ${payload.amount} ${payload.currency}`);
      sendNotification(payload.sourceAccountId, `Tu pago de ${payload.amount} ${payload.currency} está siendo procesado.`);
      break;

    case 'payment.processed':
      if (payload.status === 'approved') {
        console.log(`✅ [NOTIFICATION] Pago aprobado: ${payload.paymentId}`);
        sendNotification(payload.paymentId, `Tu pago fue aprobado exitosamente.`);
      } else if (payload.status === 'rejected') {
        console.log(`❌ [NOTIFICATION] Pago rechazado: ${payload.paymentId}`);
        sendNotification(payload.paymentId, `Tu pago fue rechazado: ${payload.rejectionReason}`);
      }
      break;

    default:
      console.log(`📨 Evento recibido [${topic}]:`, payload);
  }
}

function sendNotification(recipientId, message) {
  // Simulación de envío — en producción llamaría a un servicio de SMS/email
  console.log(`   → Notificación enviada a [${recipientId}]: "${message}"`);
}

function simulateEvents() {
  // Modo demo sin Kafka
  setTimeout(() => {
    handleEvent('payment.created', {
      paymentId: 'demo-001',
      amount: 150.00,
      currency: 'PEN',
      sourceAccountId: 'acct-demo',
      destinationAccountId: 'acct-dest',
    });
  }, 2000);

  setTimeout(() => {
    handleEvent('payment.processed', {
      paymentId: 'demo-001',
      status: 'approved',
      processedAt: new Date().toISOString(),
    });
  }, 4000);
}

module.exports = { consumePaymentEvents };
```

---

### catalog-info.yaml de cada servicio

#### `payments-api/catalog-info.yaml`

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payments-api
  title: Payments API Service
  description: Servicio REST para procesamiento de pagos de FinTech Rápida S.A.
  annotations:
    backstage.io/techdocs-ref: dir:.
    github.com/project-slug: fintechrapida/payments-api
  tags:
    - node
    - rest
    - payments
    - kafka
  links:
    - url: http://localhost:3000/api-docs
      title: Swagger UI
      icon: docs
    - url: http://localhost:3000/health
      title: Health Check
      icon: search
spec:
  type: service
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  providesApis:
    - payments-openapi
    - payments-asyncapi
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payments-openapi
  title: Payments REST API
  description: Contrato OpenAPI del servicio de pagos
  tags:
    - openapi
    - payments
spec:
  type: openapi
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  definition:
    $text: ./openapi/payments.yaml
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payments-asyncapi
  title: Payments Events (AsyncAPI)
  description: Eventos Kafka emitidos por el servicio de pagos
  tags:
    - asyncapi
    - kafka
    - events
spec:
  type: asyncapi
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  definition:
    $text: ./asyncapi/payments-events.yaml
```

#### `accounts-api/catalog-info.yaml`

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: accounts-api
  title: Accounts API Service
  description: Servicio REST para gestión de cuentas bancarias
  annotations:
    backstage.io/techdocs-ref: dir:.
  tags:
    - python
    - fastapi
    - accounts
  links:
    - url: http://localhost:3001/docs
      title: FastAPI Docs (local)
      icon: docs
spec:
  type: service
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  providesApis:
    - accounts-openapi
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: accounts-openapi
  title: Accounts REST API
  description: Contrato OpenAPI del servicio de cuentas
  tags:
    - openapi
    - accounts
spec:
  type: openapi
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  definition:
    $text: ./openapi/accounts.yaml
```

#### `notifications-service/catalog-info.yaml`

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: notifications-service
  title: Notifications Service
  description: Consumidor Kafka que envía notificaciones basadas en eventos de pago
  annotations:
    backstage.io/techdocs-ref: dir:.
  tags:
    - node
    - kafka
    - consumer
    - notifications
spec:
  type: service
  lifecycle: production
  owner: group:backend-team
  system: fintech-rapida
  consumesApis:
    - payments-asyncapi
```

---

## Pasos del Lab (guía paso a paso — 2 horas)

### Fase 1: Setup del entorno (15 min)

```bash
# 1. Clonar/crear la estructura del lab
mkdir lab-04 && cd lab-04

# 2. Instalar dependencias de payments-api
mkdir -p payments-api/src/routes payments-api/src/kafka \
         payments-api/openapi payments-api/asyncapi payments-api/docs
cd payments-api
npm init -y
npm install express swagger-ui-express yamljs uuid kafkajs
cd ..

# 3. Instalar dependencias de accounts-api
mkdir -p accounts-api/src accounts-api/openapi accounts-api/docs
cd accounts-api
pip install fastapi uvicorn pydantic --quiet
cd ..

# 4. Instalar dependencias de notifications-service
mkdir -p notifications-service/src/kafka notifications-service/docs
cd notifications-service
npm init -y
npm install kafkajs
cd ..
```

### Fase 2: Arrancar los servicios (10 min)

```bash
# Terminal 1: payments-api
cd payments-api && node src/index.js

# Terminal 2: accounts-api
cd accounts-api && uvicorn src.main:app --port 3001 --reload

# Terminal 3: notifications-service
cd notifications-service && node src/index.js
```

Verificar que los tres servicios responden:

```bash
curl http://localhost:3000/health
curl http://localhost:3001/health
```

### Fase 3: Registrar entidades en Backstage (20 min)

```bash
# Asegúrate de que tu Backstage está corriendo
cd tu-backstage && yarn dev
```

1. Abre `http://localhost:3000`
2. Ve a **Catalog → + Register Existing Component**
3. Registra (uno por uno o con el `all-components.yaml` unificado):
   - `file:../lab-04/payments-api/catalog-info.yaml`
   - `file:../lab-04/accounts-api/catalog-info.yaml`
   - `file:../lab-04/notifications-service/catalog-info.yaml`

> **Nota para registro local:** Si tu Backstage no acepta paths `file://`, usa un servidor HTTP simple:
> ```bash
> cd lab-04 && npx serve -p 8080
> # Luego registra: http://localhost:8080/payments-api/catalog-info.yaml
> ```

### Fase 4: Explorar el API Catalog (20 min)

1. Ve a **Catalog → APIs**
2. Verifica que aparecen: `payments-openapi`, `accounts-openapi`, `payments-asyncapi`
3. Abre `payments-openapi` → pestaña **Definition** → deberías ver el Swagger UI
4. Abre `payments-asyncapi` → deberías ver el AsyncAPI Studio
5. Ve a **Catalog → Systems → fintech-rapida** → pestaña **Diagram**

### Fase 5: TechDocs (15 min)

Crea los archivos `mkdocs.yml` y la carpeta `docs/` para `payments-api`:

```bash
# payments-api/mkdocs.yml
cat > payments-api/mkdocs.yml << 'EOF'
site_name: Payments API
nav:
  - Home: index.md
  - Architecture: architecture.md
plugins:
  - techdocs-core
EOF

# payments-api/docs/index.md
mkdir -p payments-api/docs
cat > payments-api/docs/index.md << 'EOF'
# Payments API

Servicio de pagos de FinTech Rápida S.A.

## Endpoints
- `POST /payments` — Crear pago
- `GET /payments/:id` — Consultar pago
- `GET /payments` — Listar pagos
- `PATCH /payments/:id/cancel` — Cancelar pago
EOF
```

Verifica en Backstage: componente `payments-api` → pestaña **Docs**.

### Fase 6: Pruebas end-to-end (20 min)

Ejecuta estos comandos para probar el flujo completo:

```bash
# 1. Crear una cuenta origen
curl -s -X POST http://localhost:3001/accounts \
  -H "Content-Type: application/json" \
  -d '{"ownerId":"user-001","accountType":"savings","currency":"PEN","initialBalance":1000}' \
  | python3 -m json.tool

# 2. Crear una cuenta destino
curl -s -X POST http://localhost:3001/accounts \
  -H "Content-Type: application/json" \
  -d '{"ownerId":"user-002","accountType":"checking","currency":"PEN","initialBalance":0}' \
  | python3 -m json.tool

# 3. Crear un pago (reemplaza los IDs con los generados arriba)
curl -s -X POST http://localhost:3000/payments \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 250.00,
    "currency": "PEN",
    "sourceAccountId": "ACCOUNT_ID_1",
    "destinationAccountId": "ACCOUNT_ID_2",
    "description": "Pago de alquiler"
  }' | python3 -m json.tool

# 4. Consultar el pago (reemplaza PAYMENT_ID)
curl -s http://localhost:3000/payments/PAYMENT_ID | python3 -m json.tool

# 5. Cancelar el pago
curl -s -X PATCH http://localhost:3000/payments/PAYMENT_ID/cancel | python3 -m json.tool

# 6. Verificar en el terminal del notifications-service los eventos recibidos
```

### Fase 7: Validación final en Backstage (20 min)

Verifica los siguientes checkpoints:

| # | Verificación | Dónde verlo |
|---|---|---|
| ✅ 1 | System `fintech-rapida` visible | Catalog → Systems |
| ✅ 2 | 3 Components registrados | Catalog → Components |
| ✅ 3 | Swagger UI de `payments-openapi` | API → payments-openapi → Definition |
| ✅ 4 | Swagger UI de `accounts-openapi` | API → accounts-openapi → Definition |
| ✅ 5 | AsyncAPI Studio de `payments-asyncapi` | API → payments-asyncapi → Definition |
| ✅ 6 | Grafo de relaciones completo | System → fintech-rapida → Diagram |
| ✅ 7 | TechDocs de payments-api | Component → payments-api → Docs |
| ✅ 8 | `notifications-service` consume `payments-asyncapi` | API → payments-asyncapi → Consumers |

---

## Troubleshooting frecuente

**La spec OpenAPI no renderiza en Backstage**  
→ Verifica que la ruta en `definition.$text` es relativa al `catalog-info.yaml` y que el archivo existe.

**Error: "Could not find entity"**  
→ Asegúrate de haber registrado primero el `System` y los `Group` antes de los `Component`.

**El plugin asyncapi no muestra el schema**  
→ Verifica la versión: `asyncapi: 2.6.0` (Backstage soporta AsyncAPI 2.x por defecto).

**TechDocs muestra página en blanco**  
→ Verifica que `backstage.io/techdocs-ref: dir:.` está en las annotations y que `mkdocs.yml` existe.

**Los servicios no arrancan**  
→ Verifica puertos: payments-api (3000), accounts-api (3001), notifications-service (no expone puerto HTTP).

---

## Referencias

- [Backstage API Docs Plugin](https://backstage.io/docs/features/software-catalog/descriptor-format#kind-api)
- [OpenAPI Specification 3.0](https://swagger.io/specification/)
- [AsyncAPI Specification 2.6](https://www.asyncapi.com/docs/reference/specification/v2.6.0)
- [TechDocs — Backstage](https://backstage.io/docs/features/techdocs/)
- [Backstage Software Catalog](https://backstage.io/docs/features/software-catalog/)
