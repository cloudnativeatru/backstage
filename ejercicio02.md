# 🔐 Configuración de Autenticación en Backstage

> **Audiencia:** Administradores y Desarrolladores

> ℹ️ Esta guía está escrita para el **nuevo sistema de frontend** de Backstage. Si usas el sistema antiguo, consulta [esta guía alternativa](https://backstage.io/docs/getting-started/config/authentication--old).

---

## 📋 Resumen

Esta guía te explica cómo configurar autenticación en tu instancia de Backstage usando **GitHub como proveedor de identidad**. Al terminar tendrás:

- Un flujo de login funcional con GitHub OAuth
- Usuarios en el Catálogo de Backstage que coinciden con los usuarios que inician sesión

> 💡 Backstage soporta múltiples proveedores de autenticación (Google, GitLab, Okta, etc.). Esta guía usa GitHub por ser el más común. Puedes explorar los demás en la [documentación de auth](https://backstage.io/docs/auth/).

> ⚠️ **Nota sobre el modo Guest:** Por defecto, Backstage viene con un resolver de tipo "guest" que hace que todos los usuarios compartan una sola identidad genérica. Es útil para arrancar rápido, pero **no es adecuado para un entorno real**. Esta guía reemplaza ese comportamiento con autenticación real.

---

## Paso 1 — Crear una OAuth App en GitHub

Backstage se autentica con GitHub mediante **OAuth 2.0**. Para eso necesitas registrar una aplicación en tu cuenta de GitHub.

1. Ve a: [https://github.com/settings/applications/new](https://github.com/settings/applications/new)

2. Completa el formulario con estos valores:

   | Campo | Valor |
   |-------|-------|
   | **Application name** | `Backstage (local)` o el nombre que prefieras |
   | **Homepage URL** | `http://localhost:3000` |
   | **Authorization callback URL** | `http://localhost:7007/api/auth/github/handler/frame` |

   > La "Homepage URL" apunta al frontend de Backstage. La "callback URL" es la dirección del backend que procesará la respuesta de GitHub tras el login.

3. Haz clic en **Register application**.

4. En la siguiente pantalla verás el **Client ID**. Haz clic en **"Generate a new client secret"** para obtener el **Client Secret**.

   > 🔑 Guarda ambos valores, los necesitarás en el siguiente paso.

---

## Paso 2 — Configurar las credenciales en `app-config.yaml`

Abre el archivo `app-config.yaml` en la raíz del proyecto y agrega la sección de GitHub dentro de `auth.providers`:

```yaml
auth:
  # Entorno activo (development para local, production para prod)
  environment: development
  providers:
    # El proveedor guest se mantiene por compatibilidad
    guest: {}
    github:
      development:
        clientId: TU_CLIENT_ID       # Reemplaza con el valor de GitHub
        clientSecret: TU_CLIENT_SECRET  # Reemplaza con el valor de GitHub
```

> ⚠️ **Nunca subas el `clientSecret` a un repositorio público.** Para producción, usa variables de entorno o un gestor de secretos.

---

## Paso 3 — Agregar el botón de login al frontend

Ahora hay que modificar el código del frontend para mostrar la pantalla de inicio de sesión con GitHub.

### 3.1 Instalar los paquetes necesarios

Desde la raíz del proyecto, ejecuta:

```bash
yarn --cwd packages/app add @backstage/core-plugin-api @backstage/plugin-app-react
```

Estos paquetes proveen las APIs de autenticación y los componentes de UI para la página de sign-in.

### 3.2 Modificar `App.tsx`

Abre `packages/app/src/App.tsx` y añade estas importaciones **al final de los imports existentes**:

```typescript
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPageBlueprint } from '@backstage/plugin-app-react';
import { SignInPage } from '@backstage/core-components';
import { createFrontendModule } from '@backstage/frontend-plugin-api';
```

> `githubAuthApiRef` es la referencia a la API de autenticación de GitHub.
> `SignInPageBlueprint` es la fábrica que construye la pantalla de login.
> `SignInPage` es el componente visual de la pantalla.

### 3.3 Crear la extensión de sign-in

Justo debajo de los imports, agrega este bloque que define cómo se verá y funcionará la pantalla de login:

```tsx
const signInPage = SignInPageBlueprint.make({
  params: {
    loader: async () => props =>
      (
        <SignInPage
          {...props}
          provider={{
            id: 'github-auth-provider',
            title: 'GitHub',
            message: 'Inicia sesión con tu cuenta de GitHub',
            apiRef: githubAuthApiRef,
          }}
        />
      ),
  },
});
```

> Este código construye una "extensión" de Backstage que le indica al sistema de frontend que renderice la pantalla de login con el botón de GitHub.

### 3.4 Registrar la extensión en `createApp()`

Busca la llamada a `createApp()` en el mismo archivo. Reemplaza:

```tsx
// ❌ Antes
export default createApp({
  features: [catalogPlugin, navModule],
});
```

por:

```tsx
// ✅ Después
export default createApp({
  features: [
    catalogPlugin,
    navModule,
    createFrontendModule({
      pluginId: 'app',
      extensions: [signInPage],  // Registra la pantalla de login
    }),
  ],
});
```

> `createFrontendModule` empaqueta la extensión `signInPage` dentro del sistema de plugins de Backstage para que sea reconocida por el framework.

---

## Paso 4 — Agregar el resolver de sign-in

El **resolver** es la lógica que vincula la identidad de GitHub con un usuario del Catálogo de Backstage. Sin esto, podrías autenticarte con GitHub pero Backstage no sabría qué usuario del Catálogo eres tú.

Actualiza `app-config.yaml` añadiendo la sección `signIn.resolvers`:

```yaml
auth:
  environment: development
  providers:
    guest: {}
    github:
      development:
        clientId: TU_CLIENT_ID
        clientSecret: TU_CLIENT_SECRET
        signIn:
          resolvers:
            # Busca en el Catálogo un User cuyo metadata.name
            # coincida con el username de GitHub
            - resolver: usernameMatchingUserEntityName
```

> El resolver `usernameMatchingUserEntityName` toma tu nombre de usuario de GitHub (ej. `jperez`) y busca en el Catálogo una entidad `User` con `metadata.name: jperez`. Si no la encuentra, el login falla con el mensaje *"unable to resolve user identity"*. Lo solucionamos en el Paso 6.

---

## Paso 5 — Agregar el plugin de auth al backend

El backend también necesita el módulo de GitHub para poder procesar el flujo OAuth.

### 5.1 Instalar el paquete

```bash
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider
```

### 5.2 Registrar el módulo en el backend

Abre `packages/backend/src/index.ts` y agrega estas dos líneas:

```typescript
backend.add(import('@backstage/plugin-auth-backend'));
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```

> La primera línea activa el sistema de autenticación del backend. La segunda carga específicamente el proveedor de GitHub. Ambas son necesarias.

### 5.3 Reiniciar Backstage

```bash
# Detener con Ctrl+C, luego:
yarn start
```

En este punto ya debería aparecer la pantalla de login. Sin embargo, si intentas ingresar obtendrás *"Failed to sign-in, unable to resolve user identity"* — eso es normal, lo resolvemos en el siguiente paso.

> ⚠️ A veces el frontend arranca antes que el backend y muestra errores en la pantalla de login. Espera a que el backend termine de iniciar y recarga la página.

---

## Paso 6 — Agregar tu usuario al Catálogo

Para que el resolver pueda encontrarte, necesitas existir como entidad `User` en el Catálogo de Backstage.

La forma recomendada en producción es usar un [proveedor de org de GitHub](https://backstage.io/docs/integrations/github/org) que sincronice usuarios y grupos automáticamente. Para esta guía, lo hacemos manualmente:

1. Abre el archivo `examples/org.yaml`

2. Al final del archivo, agrega:

```yaml
---
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: TU_USUARIO_DE_GITHUB   # Reemplaza con tu username exacto de GitHub
spec:
  memberOf: [guests]
```

> El campo `metadata.name` debe coincidir **exactamente** con tu nombre de usuario en GitHub (case-sensitive). Si en GitHub eres `jperez`, aquí debe decir `jperez`.

3. Reinicia Backstage:

```bash
# Ctrl+C para detener, luego:
yarn start
```

Ahora deberías poder iniciar sesión y ver el Catálogo con tu usuario.

---

## Paso 7 — Configurar la integración de GitHub

La **integración** (diferente a la autenticación) permite a Backstage leer entidades del Catálogo directamente desde repositorios de GitHub, como archivos `catalog-info.yaml`.

Para habilitarla necesitas un **Personal Access Token (PAT)** de GitHub.

### 7.1 Crear el token

1. Ve a: [https://github.com/settings/tokens/new](https://github.com/settings/tokens/new)
2. Dale un nombre descriptivo y elige una fecha de expiración
3. Selecciona los scopes: `repo` y `workflow`
4. Haz clic en **Generate token** y copia el valor

### 7.2 Agregar el token a la configuración local

Crea (o edita) el archivo `app-config.local.yaml` en la raíz del proyecto:

```yaml
integrations:
  github:
    - host: github.com
      token: ghp_tu_token_aqui   # Reemplaza con tu token
```

> `app-config.local.yaml` es un archivo de configuración local que **no se sube a Git** (ya está en `.gitignore`). Es el lugar correcto para poner secretos durante el desarrollo.

Para entornos más seguros, usa una variable de entorno en lugar del token directo:

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}   # Lee el valor de la variable de entorno
```

> Si modificas la configuración de integración, reinicia el backend para que tome efecto.

---

*Basado en la documentación oficial de [Backstage.io](https://backstage.io) — Copyright © 2026 Backstage Project Authors.*
