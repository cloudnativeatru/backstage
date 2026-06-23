# Ejercicio: Integrar un Chatbot de IA en Backstage con Ollama

En este ejercicio vas a agregar un chatbot de inteligencia artificial a tu portal de Backstage, usando **Ollama** para correr un modelo de lenguaje de forma local (sin enviar datos a la nube).

Al finalizar tendrás una nueva sección "AI Chat" en el sidebar de Backstage donde podrás hacerle preguntas a un modelo como Llama 3.2.

---

## Requisitos previos

- Backstage instalado y funcionando (`yarn start`)
- Node.js >= 18 y Yarn instalados
- GitHub ya configurado en Backstage

---

## Paso 1 — Instalar Ollama

Ollama es un motor que permite correr modelos de lenguaje localmente en tu máquina.

**macOS:**
```bash
brew install ollama
```
> Si no tienes Homebrew: descarga el instalador desde https://ollama.com/download

**Linux:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**Windows:**
Descarga e instala desde https://ollama.com/download

---

## Paso 2 — Descargar un modelo de lenguaje

```bash
ollama pull llama3.2
```

> Esto descarga ~2 GB. Alternativas más livianas:
> - `ollama pull llama3.2:1b` (~1 GB)
> - `ollama pull phi3` (~2 GB)

Verifica que el modelo quedó instalado:
```bash
ollama list
```

---

## Paso 3 — Iniciar Ollama

```bash
ollama serve
```

Confirma que está funcionando:
```bash
curl http://localhost:11434/api/tags
```

Deberías ver un JSON con los modelos instalados. Ollama escucha en `http://localhost:11434`.

> En macOS, si instalaste la app de escritorio, Ollama se inicia automáticamente.

---

## Paso 4 — Crear el componente ChatBot

Crea la carpeta y el archivo:

**`packages/app/src/components/ChatBot/ChatBotPage.tsx`**

```tsx
import React, { useState, useRef, useEffect } from 'react';
import {
  Box,
  TextField,
  Button,
  Paper,
  Typography,
  CircularProgress,
  Container,
  Chip,
} from '@material-ui/core';
import { makeStyles } from '@material-ui/core/styles';
import SendIcon from '@material-ui/icons/Send';
import { Page, Header, Content } from '@backstage/core-components';

const useStyles = makeStyles(theme => ({
  chatContainer: {
    display: 'flex',
    flexDirection: 'column',
    height: 'calc(100vh - 220px)',
    gap: theme.spacing(2),
  },
  modelBar: {
    display: 'flex',
    alignItems: 'center',
    gap: theme.spacing(1),
  },
  modelInput: {
    width: 200,
  },
  messageList: {
    flex: 1,
    overflowY: 'auto',
    padding: theme.spacing(2),
    backgroundColor: theme.palette.background.default,
    borderRadius: theme.shape.borderRadius,
    border: `1px solid ${theme.palette.divider}`,
    display: 'flex',
    flexDirection: 'column',
    gap: theme.spacing(1.5),
  },
  messageBubble: {
    maxWidth: '80%',
    padding: theme.spacing(1.5, 2),
    borderRadius: 12,
    wordBreak: 'break-word',
    whiteSpace: 'pre-wrap',
  },
  userBubble: {
    alignSelf: 'flex-end',
    backgroundColor: theme.palette.primary.main,
    color: theme.palette.primary.contrastText,
  },
  assistantBubble: {
    alignSelf: 'flex-start',
    backgroundColor: theme.palette.type === 'dark'
      ? theme.palette.grey[800]
      : theme.palette.grey[100],
    color: theme.palette.text.primary,
  },
  inputRow: {
    display: 'flex',
    gap: theme.spacing(1),
    alignItems: 'flex-end',
  },
  inputField: {
    flex: 1,
  },
  sendButton: {
    height: 56,
    minWidth: 56,
  },
  emptyState: {
    flex: 1,
    display: 'flex',
    flexDirection: 'column',
    alignItems: 'center',
    justifyContent: 'center',
    color: theme.palette.text.secondary,
    gap: theme.spacing(1),
  },
}));

interface Message {
  role: 'user' | 'assistant';
  content: string;
}

const OLLAMA_URL = 'http://localhost:11434';

export const ChatBotPage = () => {
  const classes = useStyles();
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);
  const [model, setModel] = useState('llama3.2');
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages, loading]);

  const sendMessage = async () => {
    const text = input.trim();
    if (!text || loading) return;

    const userMsg: Message = { role: 'user', content: text };
    const history = [...messages, userMsg];
    setMessages(history);
    setInput('');
    setLoading(true);

    try {
      const res = await fetch(`${OLLAMA_URL}/api/chat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ model, messages: history, stream: false }),
      });

      if (!res.ok) {
        throw new Error(`HTTP ${res.status} — ¿Ollama está corriendo?`);
      }

      const data = await res.json();
      setMessages(prev => [
        ...prev,
        { role: 'assistant', content: data.message?.content ?? '(sin respuesta)' },
      ]);
    } catch (err: any) {
      setMessages(prev => [
        ...prev,
        {
          role: 'assistant',
          content: `❌ ${err.message}\n\nAsegúrate de que Ollama está activo en ${OLLAMA_URL}:\n  ollama serve\n  ollama pull ${model}`,
        },
      ]);
    } finally {
      setLoading(false);
    }
  };

  const handleKey = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      sendMessage();
    }
  };

  return (
    <Page themeId="tool">
      <Header
        title="AI Chatbot"
        subtitle="Conversa con un modelo de lenguaje local vía Ollama"
      />
      <Content>
        <Container maxWidth="md">
          <Box className={classes.chatContainer}>
            <Box className={classes.modelBar}>
              <Typography variant="body2" color="textSecondary">Modelo:</Typography>
              <TextField
                className={classes.modelInput}
                size="small"
                variant="outlined"
                value={model}
                onChange={e => setModel(e.target.value)}
                placeholder="llama3.2"
              />
              <Chip
                label="Ollama local"
                size="small"
                color="primary"
                variant="outlined"
              />
            </Box>

            <Box className={classes.messageList}>
              {messages.length === 0 && (
                <Box className={classes.emptyState}>
                  <Typography variant="h6">¿En qué puedo ayudarte?</Typography>
                  <Typography variant="body2">
                    Escribe una pregunta y presiona Enter o el botón de enviar.
                  </Typography>
                </Box>
              )}

              {messages.map((msg, i) => (
                <Paper
                  key={i}
                  elevation={1}
                  className={`${classes.messageBubble} ${
                    msg.role === 'user' ? classes.userBubble : classes.assistantBubble
                  }`}
                >
                  <Typography variant="body2">{msg.content}</Typography>
                </Paper>
              ))}

              {loading && (
                <Box display="flex" alignItems="center" gap={1} pl={1}>
                  <CircularProgress size={16} />
                  <Typography variant="body2" color="textSecondary">
                    {model} está pensando…
                  </Typography>
                </Box>
              )}
              <div ref={bottomRef} />
            </Box>

            <Box className={classes.inputRow}>
              <TextField
                className={classes.inputField}
                multiline
                maxRows={4}
                variant="outlined"
                placeholder="Escribe tu pregunta… (Enter para enviar, Shift+Enter para nueva línea)"
                value={input}
                onChange={e => setInput(e.target.value)}
                onKeyDown={handleKey}
                disabled={loading}
              />
              <Button
                className={classes.sendButton}
                variant="contained"
                color="primary"
                onClick={sendMessage}
                disabled={loading || !input.trim()}
              >
                <SendIcon />
              </Button>
            </Box>
          </Box>
        </Container>
      </Content>
    </Page>
  );
};
```

---

## Paso 5 — Registrar la página en App.tsx

Abre **`packages/app/src/App.tsx`**.

**5a.** Busca la línea que importa `createFrontendModule` y agrégale `createFrontendPlugin` y `PageBlueprint`:

```tsx
// Antes:
import { createFrontendModule } from '@backstage/frontend-plugin-api';

// Después:
import { createFrontendModule, createFrontendPlugin, PageBlueprint } from '@backstage/frontend-plugin-api';
```

**5b.** Antes del bloque `const signInPage = ...`, agrega el plugin del chatbot:

```tsx
const chatbotPage = PageBlueprint.make({
  params: {
    path: '/chatbot',
    loader: async () => {
      const { ChatBotPage } = await import('./components/ChatBot/ChatBotPage');
      return <ChatBotPage />;
    },
  },
});

const chatbotPlugin = createFrontendPlugin({
  pluginId: 'chatbot',
  extensions: [chatbotPage],
});
```

**5c.** Dentro de `createApp({ features: [...] })`, agrega `chatbotPlugin`:

```tsx
export default createApp({
  features: [
    catalogPlugin,
    navModule,
    githubActionsPlugin,
    apiDocsPlugin,
    kubernetesPlugin,
    chatbotPlugin,       // ← agrega esta línea
    createFrontendModule({
      pluginId: 'app',
      extensions: [signInPage],
    }),
  ],
});
```

---

## Paso 6 — Agregar el ícono en el sidebar

Abre **`packages/app/src/modules/nav/Sidebar.tsx`**.

**6a.** Agrega el import del ícono junto a los otros imports de `@material-ui/icons`:

```tsx
import ChatIcon from '@material-ui/icons/Chat';
```

**6b.** Dentro del `<SidebarGroup label="Menu" ...>`, agrega el item después de los existentes:

```tsx
<SidebarGroup label="Menu" icon={<MenuIcon />}>
  {nav.take('page:catalog')}
  {nav.take('page:scaffolder')}
  <SidebarItem icon={ChatIcon} to="/chatbot" text="AI Chat" />  {/* ← agrega esta línea */}
  <SidebarDivider />
  <SidebarScrollWrapper>
    {nav.rest({ sortBy: 'title' })}
  </SidebarScrollWrapper>
</SidebarGroup>
```

---

## Paso 7 — Probar el chatbot

1. Asegúrate de que Ollama está corriendo: `ollama serve`
2. Reinicia Backstage: `yarn start`
3. Abre `http://localhost:3000`
4. Haz clic en **AI Chat** en el sidebar izquierdo
5. Escribe una pregunta y presiona Enter

**Ejemplos de preguntas para probar:**
```
¿Qué es un Internal Developer Portal?
¿Para qué sirve Backstage de Spotify?
Explícame qué es GitOps en términos simples
¿Cuál es la diferencia entre Docker y Kubernetes?
```

---

## Solución de problemas

| Problema | Solución |
|---|---|
| "Error: HTTP 404" en el chat | Verifica que Ollama está corriendo: `ollama serve` |
| El modelo no responde | Verifica que está descargado: `ollama list` |
| La página `/chatbot` da 404 | Reinicia Backstage: `yarn start` |
| Tarda mucho en responder | Prueba un modelo más pequeño: `ollama pull llama3.2:1b` |
