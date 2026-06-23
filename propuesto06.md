# Propuesto 06 — Plugin: FinOps Dashboard de Facturación Cloud

## Contexto

Tu empresa usa servicios cloud y los costos están creciendo mes a mes. El equipo de infraestructura quiere visibilidad centralizada de la facturación sin salir de Backstage, y además quiere usar IA para identificar oportunidades de ahorro.

El objetivo es construir un **FinOps Dashboard** que conecte con la API de tu proveedor cloud preferido, muestre el desglose de costos del mes actual y genere recomendaciones de optimización usando Ollama.

---

## Objetivo

Crear un plugin de Backstage que permita a cualquier usuario ingresar las credenciales de su cuenta cloud y visualizar los costos del mes actual con análisis inteligente de optimización.

---

## ¿Qué debe hacer el plugin?

1. Mostrar una página accesible desde el sidebar llamada **"FinOps"**
2. Tener un formulario para ingresar credenciales del proveedor cloud (no hardcodeadas)
3. Al consultar, mostrar:
   - **Total gastado** en el mes actual
   - **Desglose por servicio** (tabla ordenada de mayor a menor costo)
   - **Desglose por grupo de recursos** (o equivalente según el proveedor)
4. Tener un botón **"Analizar con IA"** que envíe los costos a Ollama y devuelva recomendaciones FinOps específicas
5. Manejar errores de conexión y credenciales inválidas

---

## Proveedor cloud (elige uno)

| Proveedor | Credenciales necesarias | API de costos |
|---|---|---|
| **Azure** | Tenant ID, Client ID, Client Secret, Subscription ID | Azure Cost Management API |
| **GCP** | Service Account JSON key | Cloud Billing API |
| **AWS** | Access Key ID, Secret Access Key, Region | Cost Explorer API |

> **Recomendación:** Azure o GCP tienen APIs REST más directas. AWS requiere firma SigV4 que es más compleja de implementar en JavaScript.

---

## Ejemplos de lo que debe mostrar

**Resumen:**
```
Total mes actual: $342.18 USD
Servicios activos: 12
Mayor gasto: Virtual Machines ($198.50)
```

**Tabla por servicio:**
```
Servicio                    | Costo USD | % del total
Virtual Machines            | 198.50    | ████████ 58%
Azure App Service           | 67.20     | ███ 20%
Azure Storage               | 41.30     | ██ 12%
Azure SQL Database          | 23.10     | █ 7%
...
```

**Análisis FinOps con IA (Ollama):**
```
1. Virtual Machines (58% del gasto): Las VMs están sobredimensionadas...
2. Quick win: Activar auto-shutdown en VMs de desarrollo...
3. Estimación de ahorro: 20-35% con estas medidas...
```

---

## Restricciones técnicas

- Las credenciales deben ingresarse en el formulario, **no hardcodeadas** en el código
- Puedes guardar credenciales no sensibles en `sessionStorage` para conveniencia del usuario
- El Client Secret / Service Account key **nunca debe almacenarse en localStorage**
- Usa **Ollama** para el análisis FinOps (mismo modelo del chatbot anterior)
- Los llamados a las APIs cloud deben hacerse desde el **backend de Backstage** (no desde el browser) para evitar problemas de CORS y no exponer credenciales
- No uses SDKs cloud oficiales — usa llamadas REST directas con `fetch`

---

## Criterios de evaluación

| Criterio | Puntos |
|---|---|
| Formulario de credenciales con validación básica | 10 |
| Conexión exitosa al proveedor cloud elegido | 20 |
| Tabla de costos por servicio (mes actual) | 20 |
| Tabla de costos por grupo de recursos / región | 15 |
| Tarjeta resumen (total, servicios activos, mayor gasto) | 10 |
| Análisis FinOps con Ollama (recomendaciones específicas) | 20 |
| Manejo de errores (credenciales inválidas, timeout, sin datos) | 5 |

**Total: 100 puntos**

---

## Arquitectura esperada

```
Browser (BillingPage.tsx)
        ↓ POST /api/azure-billing/costs  {credenciales + groupBy}
Backend Backstage (azureBilling.ts)
        ↓ POST https://login.microsoftonline.com/...  (obtiene token)
        ↓ POST https://management.azure.com/...  (consulta costos)
        ↑ devuelve filas de costos al frontend
Browser
        ↓ POST http://localhost:11434/api/chat  (análisis IA)
Ollama (llama3.2)
        ↑ recomendaciones FinOps en texto
```

---

## Pistas

- Las APIs de autenticación cloud (como `login.microsoftonline.com`) **no permiten llamadas desde el browser** por seguridad (CORS bloqueado para `client_credentials`). Necesitarás un **backend plugin de Backstage** que haga las llamadas a Azure server-side
- El backend plugin se crea con `createBackendPlugin` de `@backstage/backend-plugin-api` — revisa el ejercicio 10 para ver el patrón
- Para el análisis FinOps, el prompt es clave: pídele a Ollama que actúe como un experto FinOps y dale los datos de costos como contexto
- Puedes hacer dos llamadas paralelas al backend (una por servicio, otra por grupo de recursos) usando `Promise.all()` para que sea más rápido
- La respuesta de Azure Cost Management tiene esta estructura: `data.properties.columns` (nombres de columnas) y `data.properties.rows` (arreglo de valores) — los índices de columnas no siempre están en el mismo orden

---

## Entregable

Pull request o zip con los archivos creados/modificados, más una captura del dashboard con datos reales de una suscripción (pueden ser costos de $0 si la cuenta está en free tier).
