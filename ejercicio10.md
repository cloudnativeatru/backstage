# Ejercicio 10 — Solución: FinOps Dashboard de Facturación Azure

## Prerequisitos

- Backstage corriendo con `yarn start`
- Ollama instalado, corriendo (`ollama serve`) y con el modelo `llama3.2` descargado
- Una cuenta Azure con al menos una suscripción activa
- Azure CLI instalado (`az --version`) — solo para crear el Service Principal

---

## Paso 0 — Crear el Service Principal de Azure

El plugin necesita credenciales de un Service Principal con permisos de lectura de costos.

```bash
# 1. Login a Azure
az login

# 2. Ver tus suscripciones disponibles
az account list --output table

# 3. Guardar el Subscription ID que quieres monitorear
SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# 4. Crear el Service Principal con rol de lectura de costos
az ad sp create-for-rbac \
  --name "backstage-finops" \
  --role "Cost Management Reader" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID"
```

La salida te dará los valores que necesitarás en el formulario:

```json
{
  "appId": "...",       ← este es el Client ID
  "displayName": "backstage-finops",
  "password": "...",    ← este es el Client Secret
  "tenant": "..."       ← este es el Tenant ID
}
```

> **Nota:** El Subscription ID es el que usaste al crear el SP (`$SUBSCRIPTION_ID`).

---

## Paso 1 — Crear el backend plugin de Azure Billing

Las APIs de autenticación de Azure (`login.microsoftonline.com`) no permiten llamadas directas desde el browser (CORS bloqueado para el flujo `client_credentials`). El backend de Backstage actúa como intermediario: recibe las credenciales, llama a Azure server-side y devuelve los datos al frontend.

> **Nota:** El paquete `@azure/identity` ya viene instalado en el backend de Backstage. No necesitas ejecutar `yarn add`.

Crea el archivo **`packages/backend/src/plugins/azureBilling.ts`**:

```typescript
import { createBackendPlugin, coreServices } from '@backstage/backend-plugin-api';
import { ClientSecretCredential } from '@azure/identity';
import express, { Request, Response } from 'express';

export const azureBillingPlugin = createBackendPlugin({
  pluginId: 'azure-billing',
  register(env) {
    env.registerInit({
      deps: { httpRouter: coreServices.httpRouter },
      async init({ httpRouter }) {
        const router = express.Router();
        router.use(express.json());

        router.post('/costs', async (req: Request, res: Response) => {
          const {
            tenantId, clientId, clientSecret, subscriptionId,
            groupBy = 'ServiceName',
          } = req.body;

          if (!tenantId || !clientId || !clientSecret || !subscriptionId) {
            return res.status(400).json({ error: 'Credenciales incompletas' });
          }

          try {
            // @azure/identity maneja el token OAuth automáticamente
            const credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
            const tokenResponse = await credential.getToken('https://management.azure.com/.default');

            const costRes = await fetch(
              `https://management.azure.com/subscriptions/${subscriptionId}/providers/Microsoft.CostManagement/query?api-version=2023-11-01`,
              {
                method: 'POST',
                headers: {
                  'Content-Type': 'application/json',
                  Authorization: `Bearer ${tokenResponse!.token}`,
                },
                body: JSON.stringify({
                  type: 'ActualCost',
                  dataSet: {
                    granularity: 'None',
                    aggregation: { totalCost: { name: 'Cost', function: 'Sum' } },
                    grouping: [{ type: 'Dimension', name: groupBy }],
                  },
                  timeframe: 'MonthToDate',
                }),
              },
            );

            const costData = await costRes.json();
            if (costData.error) {
              return res.status(400).json({ error: costData.error.message });
            }
            return res.json(costData);
          } catch (err: any) {
            return res.status(500).json({ error: err.message });
          }
        });

        httpRouter.use(router);
        httpRouter.addAuthPolicy({ path: '/costs', allow: 'unauthenticated' });
      },
    });
  },
});
```

---

## Paso 2 — Registrar el backend plugin

Abre **`packages/backend/src/index.ts`** y agrega al final, antes de `backend.start()`:

```typescript
import { azureBillingPlugin } from './plugins/azureBilling';

// ... resto del archivo sin cambios ...

backend.add(azureBillingPlugin);   // ← agrega esta línea

backend.start();
```

El backend quedará disponible en `/api/azure-billing/costs`.

---

## Paso 3 — Crear el componente BillingPage

Crea el archivo **`packages/app/src/components/CloudBilling/BillingPage.tsx`**:

```tsx
import React, { useState } from 'react';
import {
  Box, TextField, Button, Typography, Container,
  CircularProgress, Paper, Card, CardContent,
  Table, TableBody, TableCell, TableHead, TableRow,
  Tabs, Tab, LinearProgress, InputAdornment, IconButton,
} from '@material-ui/core';
import { makeStyles } from '@material-ui/core/styles';
import AttachMoneyIcon from '@material-ui/icons/AttachMoney';
import CloudIcon from '@material-ui/icons/Cloud';
import TrendingDownIcon from '@material-ui/icons/TrendingDown';
import VisibilityIcon from '@material-ui/icons/Visibility';
import VisibilityOffIcon from '@material-ui/icons/VisibilityOff';
import { Page, Header, Content } from '@backstage/core-components';

const useStyles = makeStyles(theme => ({
  layout: { display: 'flex', flexDirection: 'column', gap: theme.spacing(3), paddingTop: theme.spacing(2) },
  credForm: { padding: theme.spacing(3) },
  formGrid: {
    display: 'grid',
    gridTemplateColumns: '1fr 1fr',
    gap: theme.spacing(2),
    marginBottom: theme.spacing(2),
    [theme.breakpoints.down('sm')]: { gridTemplateColumns: '1fr' },
  },
  summaryGrid: {
    display: 'grid',
    gridTemplateColumns: 'repeat(3, 1fr)',
    gap: theme.spacing(2),
    [theme.breakpoints.down('sm')]: { gridTemplateColumns: '1fr' },
  },
  summaryCard: { textAlign: 'center' },
  totalAmt: { fontSize: 32, fontWeight: 700, color: theme.palette.primary.main },
  topService: { fontWeight: 700, fontSize: 18 },
  progressBar: { height: 6, borderRadius: 3, flex: 1 },
  percentCell: { display: 'flex', alignItems: 'center', gap: theme.spacing(1) },
  aiBox: { padding: theme.spacing(3) },
  aiText: { whiteSpace: 'pre-wrap', lineHeight: 1.7, fontFamily: 'inherit' },
  actions: { display: 'flex', gap: theme.spacing(2), alignItems: 'center' },
}));

interface CostRow { name: string; cost: number; currency: string; }
interface Creds { tenantId: string; clientId: string; clientSecret: string; subscriptionId: string; }

const BACKEND = 'http://localhost:7007/api/azure-billing/costs';
const OLLAMA = 'http://localhost:11434';

const parseCosts = (data: any): CostRow[] => {
  const cols: string[] = (data.properties?.columns ?? []).map((c: any) => c.name);
  const ci = cols.indexOf('Cost');
  const ri = cols.indexOf('Currency');
  const ni = cols.findIndex((c: string) => c !== 'Cost' && c !== 'Currency');
  return (data.properties?.rows ?? [])
    .map((row: any[]) => ({
      name: String(row[ni] ?? 'Sin nombre'),
      cost: Number(Number(row[ci] ?? 0).toFixed(2)),
      currency: String(row[ri] ?? 'USD'),
    }))
    .filter((r: CostRow) => r.cost > 0)
    .sort((a: CostRow, b: CostRow) => b.cost - a.cost);
};

export const BillingPage = () => {
  const classes = useStyles();
  const [creds, setCreds] = useState<Creds>({
    tenantId: sessionStorage.getItem('az_tenant') ?? '',
    clientId: sessionStorage.getItem('az_client') ?? '',
    clientSecret: '',
    subscriptionId: sessionStorage.getItem('az_sub') ?? '',
  });
  const [showSecret, setShowSecret] = useState(false);
  const [loading, setLoading] = useState(false);
  const [byService, setByService] = useState<CostRow[]>([]);
  const [byRg, setByRg] = useState<CostRow[]>([]);
  const [currency, setCurrency] = useState('USD');
  const [error, setError] = useState('');
  const [tab, setTab] = useState(0);
  const [aiText, setAiText] = useState('');
  const [aiLoading, setAiLoading] = useState(false);

  const total = byService.reduce((s, r) => s + r.cost, 0);
  const rgTotal = byRg.reduce((s, r) => s + r.cost, 0);

  const query = async () => {
    const { tenantId, clientId, clientSecret, subscriptionId } = creds;
    if (!tenantId || !clientId || !clientSecret || !subscriptionId) {
      setError('Completa todas las credenciales');
      return;
    }
    setLoading(true);
    setError('');
    setByService([]);
    setByRg([]);
    setAiText('');
    setTab(0);

    sessionStorage.setItem('az_tenant', tenantId);
    sessionStorage.setItem('az_client', clientId);
    sessionStorage.setItem('az_sub', subscriptionId);

    try {
      const [svcRes, rgRes] = await Promise.all([
        fetch(BACKEND, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ ...creds, groupBy: 'ServiceName' }),
        }),
        fetch(BACKEND, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ ...creds, groupBy: 'ResourceGroupName' }),
        }),
      ]);

      const [svcData, rgData] = await Promise.all([svcRes.json(), rgRes.json()]);
      if (svcData.error) throw new Error(svcData.error);
      if (rgData.error) throw new Error(rgData.error);

      const svcRows = parseCosts(svcData);
      const rgRows = parseCosts(rgData);
      setByService(svcRows);
      setByRg(rgRows);
      if (svcRows[0]) setCurrency(svcRows[0].currency);
    } catch (e: any) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };

  const analyzeWithAI = async () => {
    setAiLoading(true);
    setAiText('');
    const prompt = `Soy administrador de Azure. Aquí están mis costos del mes actual:

Costos por servicio:
${byService.slice(0, 10).map(r => `- ${r.name}: ${r.cost.toFixed(2)} ${r.currency}`).join('\n')}

Total del mes: ${total.toFixed(2)} ${currency}

Por favor proporciona:
1. Los 3 servicios con mayor oportunidad de ahorro y cómo reducirlos específicamente
2. Quick wins que puedo implementar esta semana sin riesgo
3. Estimación de % de ahorro potencial aplicando buenas prácticas FinOps

Responde en español, sé específico y práctico.`;

    try {
      const res = await fetch(`${OLLAMA}/api/chat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          model: 'llama3.2',
          messages: [
            { role: 'system', content: 'Eres un experto FinOps en Azure con 10 años de experiencia. Das recomendaciones específicas, accionables y priorizadas por impacto económico.' },
            { role: 'user', content: prompt },
          ],
          stream: false,
        }),
      });
      const data = await res.json();
      setAiText(data.message?.content ?? '');
      setTab(2);
    } catch (e: any) {
      setError(`Error al consultar Ollama: ${e.message}`);
    } finally {
      setAiLoading(false);
    }
  };

  const CostTable = ({ rows, base }: { rows: CostRow[]; base: number }) => (
    <Table size="small">
      <TableHead>
        <TableRow>
          <TableCell><strong>Nombre</strong></TableCell>
          <TableCell align="right"><strong>Costo ({currency})</strong></TableCell>
          <TableCell><strong>% del total</strong></TableCell>
        </TableRow>
      </TableHead>
      <TableBody>
        {rows.map(row => (
          <TableRow key={row.name} hover>
            <TableCell>{row.name}</TableCell>
            <TableCell align="right">{row.cost.toFixed(2)}</TableCell>
            <TableCell>
              <Box className={classes.percentCell}>
                <LinearProgress
                  variant="determinate"
                  value={Math.min((row.cost / base) * 100, 100)}
                  className={classes.progressBar}
                />
                <Typography variant="caption">
                  {((row.cost / base) * 100).toFixed(1)}%
                </Typography>
              </Box>
            </TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );

  return (
    <Page themeId="tool">
      <Header
        title="Azure FinOps Dashboard"
        subtitle="Visualiza y optimiza los costos de tu suscripción Azure con IA"
      />
      <Content>
        <Container maxWidth="lg">
          <Box className={classes.layout}>

            {/* Formulario de credenciales */}
            <Paper className={classes.credForm} elevation={2}>
              <Typography variant="h6" gutterBottom>Credenciales del Service Principal</Typography>
              <Box className={classes.formGrid}>
                <TextField
                  label="Tenant ID"
                  variant="outlined"
                  size="small"
                  value={creds.tenantId}
                  onChange={e => setCreds(p => ({ ...p, tenantId: e.target.value }))}
                  fullWidth
                />
                <TextField
                  label="Client ID (App ID)"
                  variant="outlined"
                  size="small"
                  value={creds.clientId}
                  onChange={e => setCreds(p => ({ ...p, clientId: e.target.value }))}
                  fullWidth
                />
                <TextField
                  label="Client Secret"
                  variant="outlined"
                  size="small"
                  type={showSecret ? 'text' : 'password'}
                  value={creds.clientSecret}
                  onChange={e => setCreds(p => ({ ...p, clientSecret: e.target.value }))}
                  fullWidth
                  InputProps={{
                    endAdornment: (
                      <InputAdornment position="end">
                        <IconButton size="small" onClick={() => setShowSecret(s => !s)}>
                          {showSecret ? <VisibilityOffIcon /> : <VisibilityIcon />}
                        </IconButton>
                      </InputAdornment>
                    ),
                  }}
                />
                <TextField
                  label="Subscription ID"
                  variant="outlined"
                  size="small"
                  value={creds.subscriptionId}
                  onChange={e => setCreds(p => ({ ...p, subscriptionId: e.target.value }))}
                  fullWidth
                />
              </Box>
              <Box className={classes.actions}>
                <Button
                  variant="contained"
                  color="primary"
                  startIcon={loading ? <CircularProgress size={18} color="inherit" /> : <CloudIcon />}
                  onClick={query}
                  disabled={loading}
                >
                  {loading ? 'Consultando…' : 'Consultar costos'}
                </Button>
                {byService.length > 0 && (
                  <Button
                    variant="outlined"
                    color="secondary"
                    startIcon={aiLoading ? <CircularProgress size={18} /> : <TrendingDownIcon />}
                    onClick={analyzeWithAI}
                    disabled={aiLoading}
                  >
                    {aiLoading ? 'Analizando con IA…' : 'Analizar con IA'}
                  </Button>
                )}
              </Box>
              {error && (
                <Typography color="error" style={{ marginTop: 8 }}>
                  {error}
                </Typography>
              )}
            </Paper>

            {/* Tarjetas de resumen */}
            {byService.length > 0 && (
              <Box className={classes.summaryGrid}>
                <Card>
                  <CardContent className={classes.summaryCard}>
                    <AttachMoneyIcon color="primary" />
                    <Typography variant="subtitle2" color="textSecondary">Total mes actual</Typography>
                    <Typography className={classes.totalAmt}>{total.toFixed(2)}</Typography>
                    <Typography variant="body2" color="textSecondary">{currency}</Typography>
                  </CardContent>
                </Card>
                <Card>
                  <CardContent className={classes.summaryCard}>
                    <Typography variant="subtitle2" color="textSecondary">Servicios activos</Typography>
                    <Typography className={classes.totalAmt}>{byService.length}</Typography>
                    <Typography variant="body2" color="textSecondary">con costo este mes</Typography>
                  </CardContent>
                </Card>
                <Card>
                  <CardContent className={classes.summaryCard}>
                    <Typography variant="subtitle2" color="textSecondary">Mayor gasto</Typography>
                    <Typography className={classes.topService}>{byService[0]?.name}</Typography>
                    <Typography variant="body2" color="textSecondary">
                      {byService[0]?.cost.toFixed(2)} {currency} ({((byService[0]?.cost / total) * 100).toFixed(1)}%)
                    </Typography>
                  </CardContent>
                </Card>
              </Box>
            )}

            {/* Tablas de costos */}
            {byService.length > 0 && (
              <Paper elevation={2}>
                <Tabs value={tab} onChange={(_, v) => setTab(v)}>
                  <Tab label="Por Servicio" />
                  <Tab label="Por Grupo de Recursos" />
                  {aiText && <Tab label="Análisis FinOps IA" />}
                </Tabs>

                {tab === 0 && <CostTable rows={byService} base={total} />}

                {tab === 1 && <CostTable rows={byRg} base={rgTotal} />}

                {tab === 2 && aiText && (
                  <Box className={classes.aiBox}>
                    <Typography variant="h6" gutterBottom>
                      Recomendaciones FinOps — llama3.2
                    </Typography>
                    <Typography className={classes.aiText}>{aiText}</Typography>
                  </Box>
                )}
              </Paper>
            )}

          </Box>
        </Container>
      </Content>
    </Page>
  );
};
```

---

## Paso 4 — Registrar la página en App.tsx

Abre **`packages/app/src/App.tsx`** y agrega:

**4a.** Define el plugin del dashboard (junto a los otros):

```tsx
const billingPage = PageBlueprint.make({
  params: {
    path: '/billing',
    loader: async () => {
      const { BillingPage } = await import('./components/CloudBilling/BillingPage');
      return <BillingPage />;
    },
  },
});

const billingPlugin = createFrontendPlugin({
  pluginId: 'azure-billing-ui',
  extensions: [billingPage],
});
```

**4b.** Agrégalo a la lista `features`:

```tsx
export default createApp({
  features: [
    catalogPlugin,
    navModule,
    githubActionsPlugin,
    apiDocsPlugin,
    kubernetesPlugin,
    chatbotPlugin,
    diagramPlugin,
    billingPlugin,      // ← agrega esta línea
    createFrontendModule({ pluginId: 'app', extensions: [signInPage] }),
  ],
});
```

---

## Paso 5 — Agregar el ícono en el sidebar

Abre **`packages/app/src/modules/nav/Sidebar.tsx`**.

**5a.** Agrega el import del ícono:

```tsx
import MonetizationOnIcon from '@material-ui/icons/MonetizationOn';
```

**5b.** Agrega el item en el `<SidebarGroup label="Menu">`:

```tsx
<SidebarItem icon={ChatIcon} to="/chatbot" text="AI Chat" />
<SidebarItem icon={AccountTreeIcon} to="/diagrams" text="AI Diagrams" />
<SidebarItem icon={MonetizationOnIcon} to="/billing" text="FinOps" />  {/* ← nueva línea */}
```

---

## Paso 6 — Reiniciar y probar

```bash
# Reinicia backend y frontend (Ctrl+C y luego:)
yarn start
```

1. Abre `http://localhost:3000`
2. Haz clic en **FinOps** en el sidebar
3. Ingresa las credenciales del Service Principal creado en el Paso 0
4. Haz clic en **Consultar costos**
5. Revisa las tablas "Por Servicio" y "Por Grupo de Recursos"
6. Haz clic en **Analizar con IA** para obtener recomendaciones FinOps

---

## ¿Cómo funciona por dentro?

```
Usuario ingresa credenciales en BillingPage
        ↓
fetch POST /api/azure-billing/costs  (×2 en paralelo: ServiceName + ResourceGroupName)
        ↓
Backend plugin (azureBilling.ts) recibe las credenciales
        ↓
Backend usa @azure/identity (ClientSecretCredential) para obtener el token OAuth
          (client_credentials grant contra login.microsoftonline.com, automático)
        ↓
Backend → POST https://management.azure.com/subscriptions/.../query
          (Azure Cost Management API con Bearer token → devuelve filas de costos)
        ↓
Backend devuelve los datos al frontend
        ↓
Frontend muestra tablas + tarjetas resumen
        ↓ (opcional)
fetch POST http://localhost:11434/api/chat  (Ollama directo, sin proxy)
        ↓
llama3.2 analiza los costos y devuelve recomendaciones FinOps
```

---

## Permisos requeridos en Azure

El Service Principal necesita el rol **"Cost Management Reader"** a nivel de suscripción.

```bash
# Verificar los permisos asignados
az role assignment list \
  --assignee <clientId> \
  --subscription <subscriptionId> \
  --output table
```

Si los costos aparecen vacíos (0 filas), puede ser que:
- La suscripción es nueva y no tiene datos de facturación aún
- El registro del resource provider `Microsoft.CostManagement` no está habilitado

```bash
# Habilitar el resource provider si falta
az provider register --namespace Microsoft.CostManagement
```

---

## Solución de problemas

| Problema | Causa probable | Solución |
|---|---|---|
| "Auth Azure fallida: AADSTS..." | Credenciales incorrectas o SP sin permisos | Verifica tenant ID, client ID, client secret; revisa el rol asignado al SP |
| "Credenciales incompletas" | Algún campo vacío en el formulario | Completa los 4 campos antes de consultar |
| Tabla vacía (sin filas) | La suscripción no tiene gastos este mes | Prueba con una suscripción con actividad; o el SP no tiene rol Cost Management Reader |
| Error de compilación TypeScript en el backend | `fetch` no reconocido en Node.js < 18 | Verifica que usas Node.js 18+: `node --version` |
| El backend no arranca | Error de importación en `azureBilling.ts` | Revisa que el archivo está en `packages/backend/src/plugins/` y el import en `index.ts` es correcto |
| "Error al consultar Ollama" en el análisis IA | Ollama no está corriendo | Ejecuta `ollama serve` en otra terminal |
| Costos del mes en $0 | Suscripción free tier sin recursos | Es normal; el dashboard funciona pero no habrá datos que analizar |
