# Ejercicio 09 — Solución: Generador de Diagramas de Arquitectura con IA

## Prerequisitos

- Backstage corriendo con `yarn start`
- Ollama instalado, corriendo (`ollama serve`) y con el modelo `llama3.2` descargado
- El chatbot del ejercicio anterior ya configurado (Ollama accesible desde el browser)

---

## Paso 1 — Instalar la librería Mermaid

Mermaid es la librería que convierte texto en diagramas visuales. Instálala en el paquete `app`:

```bash
yarn workspace app add mermaid
```

Verifica que quedó en `packages/app/package.json` dentro de `dependencies`.

---

## Paso 2 — Crear el componente DiagramPage

Crea el archivo **`packages/app/src/components/DiagramGenerator/DiagramPage.tsx`**:

```tsx
import React, { useState, useEffect, useRef } from 'react';
import {
  Box, TextField, Button, Typography, Container,
  CircularProgress, Paper, Tabs, Tab, IconButton, Tooltip,
} from '@material-ui/core';
import { makeStyles } from '@material-ui/core/styles';
import SendIcon from '@material-ui/icons/Send';
import FileCopyIcon from '@material-ui/icons/FileCopy';
import { Page, Header, Content } from '@backstage/core-components';
import mermaid from 'mermaid';

const useStyles = makeStyles(theme => ({
  layout: {
    display: 'flex',
    flexDirection: 'column',
    gap: theme.spacing(3),
    paddingTop: theme.spacing(2),
  },
  inputSection: {
    display: 'flex',
    flexDirection: 'column',
    gap: theme.spacing(2),
  },
  actions: {
    display: 'flex',
    gap: theme.spacing(2),
    alignItems: 'center',
  },
  resultSection: {
    border: `1px solid ${theme.palette.divider}`,
    borderRadius: theme.shape.borderRadius,
    overflow: 'hidden',
  },
  tabPanel: {
    padding: theme.spacing(3),
    minHeight: 300,
    backgroundColor: theme.palette.background.default,
    position: 'relative',
  },
  diagramContainer: {
    display: 'flex',
    justifyContent: 'center',
    alignItems: 'flex-start',
    '& svg': {
      maxWidth: '100%',
    },
  },
  codeContainer: {
    position: 'relative',
  },
  copyButton: {
    position: 'absolute',
    top: 0,
    right: 0,
  },
  codeBlock: {
    backgroundColor: theme.palette.type === 'dark' ? '#1e1e1e' : '#f5f5f5',
    padding: theme.spacing(2),
    borderRadius: theme.shape.borderRadius,
    fontFamily: 'monospace',
    fontSize: 13,
    overflowX: 'auto',
    whiteSpace: 'pre',
    marginTop: theme.spacing(4),
  },
  emptyState: {
    display: 'flex',
    flexDirection: 'column',
    alignItems: 'center',
    justifyContent: 'center',
    height: 260,
    color: theme.palette.text.secondary,
    gap: theme.spacing(1),
  },
  errorBox: {
    padding: theme.spacing(2),
    color: theme.palette.error.main,
    whiteSpace: 'pre-wrap',
    fontFamily: 'monospace',
    fontSize: 13,
  },
  examples: {
    display: 'flex',
    gap: theme.spacing(1),
    flexWrap: 'wrap',
  },
  exampleChip: {
    cursor: 'pointer',
    padding: theme.spacing(0.5, 1.5),
    border: `1px solid ${theme.palette.divider}`,
    borderRadius: 16,
    fontSize: 12,
    '&:hover': {
      backgroundColor: theme.palette.action.hover,
    },
  },
}));

const OLLAMA_URL = 'http://localhost:11434';

const SYSTEM_PROMPT = `You are an expert software architect specialized in creating Mermaid diagrams.
The user will describe a software architecture or system in natural language.
Your response must contain ONLY valid Mermaid diagram code — no explanation, no markdown fences, no extra text.
Choose the most appropriate diagram type: graph TD, flowchart LR, sequenceDiagram, classDiagram, or erDiagram.
Use clear and descriptive node labels in the same language the user writes in.`;

const EXAMPLES = [
  'API REST con autenticación JWT, base de datos PostgreSQL y caché Redis',
  'Microservicios de e-commerce: catálogo, carrito, pagos y notificaciones con Kafka',
  'App móvil conectada a Firebase con funciones serverless y Firestore',
  'CI/CD pipeline con GitHub Actions, Docker y deploy en Kubernetes',
];

mermaid.initialize({
  startOnLoad: false,
  theme: 'default',
  securityLevel: 'loose',
});

export const DiagramPage = () => {
  const classes = useStyles();
  const [description, setDescription] = useState('');
  const [mermaidCode, setMermaidCode] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  const [tab, setTab] = useState(0);
  const [copied, setCopied] = useState(false);
  const diagramRef = useRef<HTMLDivElement>(null);
  const diagramId = useRef(`mermaid-${Date.now()}`);

  useEffect(() => {
    if (!mermaidCode || tab !== 0) return;

    const renderDiagram = async () => {
      try {
        const { svg } = await mermaid.render(diagramId.current, mermaidCode);
        if (diagramRef.current) {
          diagramRef.current.innerHTML = svg;
        }
        // Regenerate ID to avoid conflicts on re-render
        diagramId.current = `mermaid-${Date.now()}`;
      } catch (e: any) {
        setError(`El diagrama generado tiene errores de sintaxis Mermaid.\n\nDetalle: ${e.message}\n\nIntenta con una descripción más detallada.`);
        if (diagramRef.current) diagramRef.current.innerHTML = '';
      }
    };

    renderDiagram();
  }, [mermaidCode, tab]);

  const generate = async () => {
    if (!description.trim() || loading) return;
    setLoading(true);
    setError('');
    setMermaidCode('');
    if (diagramRef.current) diagramRef.current.innerHTML = '';

    try {
      const res = await fetch(`${OLLAMA_URL}/api/chat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          model: 'llama3.2',
          messages: [
            { role: 'system', content: SYSTEM_PROMPT },
            { role: 'user', content: description },
          ],
          stream: false,
        }),
      });

      if (!res.ok) throw new Error(`Ollama respondió con HTTP ${res.status}. ¿Está corriendo ollama serve?`);

      const data = await res.json();
      const raw: string = data.message?.content ?? '';

      // Strip markdown code fences if the model includes them anyway
      const cleaned = raw
        .replace(/^```mermaid\s*/i, '')
        .replace(/^```\s*/i, '')
        .replace(/```\s*$/i, '')
        .trim();

      if (!cleaned) throw new Error('El modelo no devolvió código Mermaid. Intenta con una descripción más específica.');

      setMermaidCode(cleaned);
      setTab(0);
    } catch (e: any) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };

  const copyCode = () => {
    navigator.clipboard.writeText(mermaidCode);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  const handleKey = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && e.ctrlKey) generate();
  };

  return (
    <Page themeId="tool">
      <Header
        title="AI Diagram Generator"
        subtitle="Describe una arquitectura en lenguaje natural y obtén un diagrama automáticamente"
      />
      <Content>
        <Container maxWidth="lg">
          <Box className={classes.layout}>

            {/* Input */}
            <Box className={classes.inputSection}>
              <Typography variant="subtitle2" color="textSecondary">
                Ejemplos rápidos:
              </Typography>
              <Box className={classes.examples}>
                {EXAMPLES.map(ex => (
                  <span
                    key={ex}
                    className={classes.exampleChip}
                    onClick={() => setDescription(ex)}
                  >
                    {ex}
                  </span>
                ))}
              </Box>
              <TextField
                multiline
                minRows={4}
                maxRows={8}
                fullWidth
                variant="outlined"
                label="Describe tu arquitectura"
                placeholder="Ej: Un frontend React que se comunica con una API Node.js, la cual guarda datos en PostgreSQL y usa Redis como caché..."
                value={description}
                onChange={e => setDescription(e.target.value)}
                onKeyDown={handleKey}
                disabled={loading}
                helperText="Ctrl+Enter para generar"
              />
              <Box className={classes.actions}>
                <Button
                  variant="contained"
                  color="primary"
                  endIcon={loading ? <CircularProgress size={18} color="inherit" /> : <SendIcon />}
                  onClick={generate}
                  disabled={loading || !description.trim()}
                >
                  {loading ? 'Generando…' : 'Generar diagrama'}
                </Button>
                {mermaidCode && (
                  <Typography variant="body2" color="textSecondary">
                    Diagrama generado con llama3.2
                  </Typography>
                )}
              </Box>
            </Box>

            {/* Result */}
            {(mermaidCode || error) && (
              <Paper className={classes.resultSection} elevation={2}>
                <Tabs value={tab} onChange={(_, v) => setTab(v)}>
                  <Tab label="Diagrama" />
                  <Tab label="Código Mermaid" />
                </Tabs>

                <Box className={classes.tabPanel}>
                  {error && (
                    <Typography className={classes.errorBox}>❌ {error}</Typography>
                  )}

                  {!error && tab === 0 && (
                    <Box className={classes.diagramContainer}>
                      <div ref={diagramRef} />
                    </Box>
                  )}

                  {!error && tab === 1 && (
                    <Box className={classes.codeContainer}>
                      <Tooltip title={copied ? '¡Copiado!' : 'Copiar código'}>
                        <IconButton className={classes.copyButton} onClick={copyCode} size="small">
                          <FileCopyIcon fontSize="small" />
                        </IconButton>
                      </Tooltip>
                      <pre className={classes.codeBlock}>{mermaidCode}</pre>
                    </Box>
                  )}
                </Box>
              </Paper>
            )}

            {!mermaidCode && !error && (
              <Box className={classes.emptyState}>
                <Typography variant="h6">Ningún diagrama generado aún</Typography>
                <Typography variant="body2">
                  Escribe una descripción arriba y haz clic en "Generar diagrama"
                </Typography>
              </Box>
            )}

          </Box>
        </Container>
      </Content>
    </Page>
  );
};
```

---

## Paso 3 — Registrar la página en App.tsx

Abre **`packages/app/src/App.tsx`**.

**3a.** En el import de `@backstage/frontend-plugin-api`, ya tienes `createFrontendPlugin` y `PageBlueprint` del ejercicio anterior. No necesitas cambiar los imports.

**3b.** Agrega el plugin del generador de diagramas junto al `chatbotPlugin` existente:

```tsx
const diagramPage = PageBlueprint.make({
  params: {
    path: '/diagrams',
    loader: async () => {
      const { DiagramPage } = await import('./components/DiagramGenerator/DiagramPage');
      return <DiagramPage />;
    },
  },
});

const diagramPlugin = createFrontendPlugin({
  pluginId: 'diagram-generator',
  extensions: [diagramPage],
});
```

**3c.** Agrega `diagramPlugin` a la lista `features`:

```tsx
export default createApp({
  features: [
    catalogPlugin,
    navModule,
    githubActionsPlugin,
    apiDocsPlugin,
    kubernetesPlugin,
    chatbotPlugin,
    diagramPlugin,       // ← agrega esta línea
    createFrontendModule({
      pluginId: 'app',
      extensions: [signInPage],
    }),
  ],
});
```

---

## Paso 4 — Agregar el ícono en el sidebar

Abre **`packages/app/src/modules/nav/Sidebar.tsx`**.

**4a.** Agrega el import del ícono junto a los otros de `@material-ui/icons`:

```tsx
import AccountTreeIcon from '@material-ui/icons/AccountTree';
```

**4b.** Dentro del `<SidebarGroup label="Menu" ...>`, agrega el item debajo del de AI Chat:

```tsx
<SidebarGroup label="Menu" icon={<MenuIcon />}>
  {nav.take('page:catalog')}
  {nav.take('page:scaffolder')}
  <SidebarItem icon={ChatIcon} to="/chatbot" text="AI Chat" />
  <SidebarItem icon={AccountTreeIcon} to="/diagrams" text="AI Diagrams" />  {/* ← nueva línea */}
  <SidebarDivider />
  <SidebarScrollWrapper>
    {nav.rest({ sortBy: 'title' })}
  </SidebarScrollWrapper>
</SidebarGroup>
```

---

## Paso 5 — Probar el plugin

1. Reinicia Backstage: `yarn start`
2. Asegúrate de que Ollama está corriendo: `ollama serve`
3. Abre `http://localhost:3000`
4. Haz clic en **AI Diagrams** en el sidebar
5. Escribe una descripción y haz clic en **Generar diagrama**

**Descripciones para probar:**

```
Un frontend React que se comunica con una API Gateway.
La API Gateway enruta a dos microservicios: uno de usuarios y uno de pagos.
Ambos guardan datos en PostgreSQL. El servicio de pagos publica eventos en Kafka.
```

```
Pipeline CI/CD: el desarrollador hace push a GitHub, se ejecutan tests automáticos
con GitHub Actions, se construye una imagen Docker, se publica en Docker Hub
y se despliega automáticamente en un cluster de Kubernetes.
```

```
App de e-commerce con microservicios: catálogo de productos, carrito de compras,
servicio de pagos con Stripe, notificaciones por email con SendGrid,
y un API Gateway que autentica con JWT.
```

---

## ¿Cómo funciona por dentro?

```
Usuario escribe descripción
        ↓
DiagramPage envía POST a Ollama (llama3.2)
con un system prompt que pide SOLO código Mermaid
        ↓
Ollama devuelve el código Mermaid
        ↓
El componente limpia el texto (quita ```mermaid si viene)
        ↓
mermaid.render() convierte el código en SVG
        ↓
El SVG se inyecta en el DOM → diagrama visual
```

---

## Solución de problemas

| Problema | Causa probable | Solución |
|---|---|---|
| "Ollama respondió con HTTP 404" | Ollama no está corriendo | Ejecuta `ollama serve` |
| El diagrama no se renderiza | Sintaxis Mermaid inválida | Cambia a la pestaña "Código Mermaid" y revisa el código generado |
| El modelo devuelve texto en vez de Mermaid | El prompt no fue respetado | Prueba con una descripción más corta y específica |
| Tarda mucho en generar | Modelo muy grande | Prueba con `llama3.2:1b` — edita `model: 'llama3.2'` en el código |
| Error de compilación TypeScript | Falta el tipo de mermaid | Ejecuta `yarn workspace app add --dev @types/mermaid` |
