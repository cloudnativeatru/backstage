# Ejercicio 11: Customización de la Interfaz de Backstage

## Objetivo

Personalizar la interfaz de Backstage con la identidad visual de **Cloud Native Academy**, incluyendo:

- Tema personalizado con colores de marca (azul `#4B6CF7`, naranja `#F5A623`, navy `#0D1B3E`)
- Logo propio en la barra lateral
- Tipografía moderna con la fuente **Inter**
- Gradientes en los headers de cada sección
- Modo claro y oscuro con la paleta de CNA

---

## Contexto técnico

Este ejercicio usa el **nuevo sistema de frontend de Backstage** (declarativo, basado en extensiones), que se habilita con `createApp` de `@backstage/frontend-defaults`. En este sistema, todo —páginas, temas, APIs— se registra como una **extensión** con un ID único del formato `tipo:pluginId/nombre`.

Para los temas en particular, Backstage registra de fábrica `theme:app/light` y `theme:app/dark`. Para reemplazarlos, la solución no es deshabilitarlos en la configuración, sino registrar nuevas extensiones con los mismos IDs desde el módulo `app`. Eso es exactamente lo que hace este ejercicio.

---

## Paleta de Colores Cloud Native Academy

| Color | Hex | Uso |
|---|---|---|
| Azul primario | `#4B6CF7` | Botones, links, headers |
| Azul oscuro | `#3457E5` | Hover states |
| Azul claro | `#7B9EFF` | Gradientes, modo oscuro |
| Naranja accent | `#F5A623` | Indicador sidebar, CTAs |
| Navy oscuro | `#0D1B3E` | Fondo sidebar, texto principal |
| Navy profundo | `#070E1F` | Fondo sidebar modo oscuro |

---

## Paso 1: Agregar `@backstage/theme` como dependencia

El paquete `@backstage/theme` existe en el monorepo (lo usan otros plugins) pero no siempre está en las dependencias directas del frontend. Hay que declararlo explícitamente para poder importarlo.

**1a.** Edita `packages/app/package.json` y agrega dentro de `"dependencies"`:

```json
{
  "dependencies": {
    "@backstage/theme": "^0.7.3",
    ...
  }
}
```

> Verifica la versión disponible en tu proyecto con:
> ```bash
> cat node_modules/@backstage/theme/package.json | grep '"version"'
> ```

**1b.** Instala las dependencias desde la raíz del proyecto:

```bash
yarn install
```

> **Importante:** Este paso es necesario aunque el paquete ya exista en `node_modules`. Yarn necesita registrar la dependencia correctamente en el workspace.

---

## Paso 2: Agregar la fuente Inter de Google Fonts

Edita el archivo `packages/app/public/index.html`.

**Cambia el `theme-color`** (color de la barra del navegador en móvil) de negro a navy CNA:

```html
<!-- Antes -->
<meta name="theme-color" content="#000000" />

<!-- Después -->
<meta name="theme-color" content="#0D1B3E" />
```

**Agrega los links de Inter** justo antes de `</head>`. Reemplaza la línea de `safari-pinned-tab.svg`:

```html
<link
  rel="mask-icon"
  href="<%= publicPath %>/safari-pinned-tab.svg"
  color="#4B6CF7"
/>
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
<link
  href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap"
  rel="stylesheet"
/>
```

---

## Paso 3: Actualizar el título de la app

Edita `app-config.yaml` en la raíz del proyecto:

```yaml
# Antes
app:
  title: Scaffolded Backstage App

# Después
app:
  title: Cloud Native Academy - Developer Portal
```

También actualiza el nombre de la organización:

```yaml
# Antes
organization:
  name: My Company

# Después
organization:
  name: Cloud Native Academy
```

---

## Paso 4: Crear el archivo del tema personalizado

Crea la carpeta `packages/app/src/theme/` y dentro crea el archivo `cloudNativeTheme.tsx`:

```bash
mkdir -p packages/app/src/theme
```

Crea el archivo `packages/app/src/theme/cloudNativeTheme.tsx` con el siguiente contenido:

> **Nota sobre JSX sin `import React`:** Este proyecto usa el compilador moderno de TypeScript con `"jsx": "react-jsx"`, lo que significa que **no necesitas** escribir `import React from 'react'` en archivos `.tsx`. Si lo añades, TypeScript mostrará el error `'React' is declared but never read (TS6133)`. Basta con escribir JSX directamente.

```tsx
import {
  createUnifiedTheme,
  genPageTheme,
  palettes,
  shapes,
  UnifiedThemeProvider,
} from '@backstage/theme';
import { ThemeBlueprint } from '@backstage/plugin-app-react';

// Paleta de colores Cloud Native Academy
const cna = {
  blue: '#4B6CF7',
  blueDark: '#3457E5',
  blueLight: '#7B9EFF',
  orange: '#F5A623',
  orangeDark: '#E0891A',
  orangeLight: '#FFC55A',
  navy: '#0D1B3E',
  navyMid: '#111936',
  navyDeep: '#070E1F',
};

// ─── TEMA CLARO ───────────────────────────────────────────────────────────────
export const cnaLightTheme = createUnifiedTheme({
  palette: {
    ...palettes.light,
    primary: {
      main: cna.blue,
      dark: cna.blueDark,
      light: cna.blueLight,
      contrastText: '#FFFFFF',
    },
    secondary: {
      main: cna.orange,
      dark: cna.orangeDark,
      light: cna.orangeLight,
      contrastText: cna.navy,
    },
    // Colores de la barra lateral (sidebar)
    navigation: {
      background: cna.navy,       // Fondo del sidebar: navy oscuro
      indicator: cna.orange,      // Indicador del ítem activo: naranja CNA
      color: '#B8C5D6',           // Color del texto de los ítems
      selectedColor: '#FFFFFF',   // Color del ítem activo
      navItem: {
        hoverBackground: 'rgba(75, 108, 247, 0.22)', // Hover con azul semitransparente
      },
      submenu: {
        background: cna.navyMid,
      },
    },
    background: {
      default: '#F4F6FB',  // Fondo de página: azul muy claro
      paper: '#FFFFFF',
    },
    banner: {
      info: cna.blue,
      error: '#FF5252',
      text: '#FFFFFF',
      link: cna.orange,
      warning: '#FFB84D',
    },
    errorBackground: '#FFEBEE',
    warningBackground: '#FFF8E1',
    infoBackground: '#E8EDFB',
    text: {
      primary: '#1A2460',   // Texto principal: navy suave
      secondary: '#5E6E8A', // Texto secundario: azul grisáceo
    },
  },
  defaultPageTheme: 'home',
  // Fuente Inter para look moderno
  fontFamily:
    "'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif",
  // Gradientes de los headers por tipo de entidad
  pageTheme: {
    home:          genPageTheme({ colors: [cna.blue, cna.blueLight],    shape: shapes.wave }),
    documentation: genPageTheme({ colors: [cna.orange, cna.orangeLight], shape: shapes.wave2 }),
    tool:          genPageTheme({ colors: [cna.blueDark, cna.blue],     shape: shapes.round }),
    service:       genPageTheme({ colors: [cna.blue, cna.orange],       shape: shapes.wave }),
    website:       genPageTheme({ colors: [cna.blueDark, cna.blueLight], shape: shapes.wave }),
    library:       genPageTheme({ colors: [cna.orange, cna.orangeLight], shape: shapes.wave2 }),
    other:         genPageTheme({ colors: [cna.blue, cna.blueDark],     shape: shapes.wave }),
    app:           genPageTheme({ colors: [cna.blue, cna.blueLight],    shape: shapes.wave }),
    apis:          genPageTheme({ colors: [cna.blueDark, cna.blue],     shape: shapes.wave2 }),
  },
});

// ─── TEMA OSCURO ──────────────────────────────────────────────────────────────
export const cnaDarkTheme = createUnifiedTheme({
  palette: {
    ...palettes.dark,
    primary: {
      main: cna.blueLight,
      dark: cna.blue,
      light: '#B8C7FF',
      contrastText: cna.navy,
    },
    secondary: {
      main: cna.orangeLight,
      dark: cna.orange,
      light: '#FFD980',
      contrastText: cna.navy,
    },
    navigation: {
      background: cna.navyDeep,
      indicator: cna.orangeLight,
      color: '#8899BB',
      selectedColor: '#FFFFFF',
      navItem: {
        hoverBackground: 'rgba(123, 158, 255, 0.18)',
      },
      submenu: {
        background: cna.navy,
      },
    },
    background: {
      default: '#0B0F2D',
      paper: cna.navyMid,
    },
    banner: {
      info: cna.blueLight,
      error: '#FF6B6B',
      text: '#FFFFFF',
      link: cna.orangeLight,
      warning: '#FFD166',
    },
    errorBackground: '#2D1515',
    warningBackground: '#2D2510',
    infoBackground: '#0F1840',
  },
  defaultPageTheme: 'home',
  fontFamily:
    "'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif",
  pageTheme: {
    home:          genPageTheme({ colors: [cna.blue, cna.blueLight],    shape: shapes.wave }),
    documentation: genPageTheme({ colors: [cna.orange, cna.orangeLight], shape: shapes.wave2 }),
    tool:          genPageTheme({ colors: [cna.blueDark, cna.blue],     shape: shapes.round }),
    service:       genPageTheme({ colors: [cna.blue, cna.orange],       shape: shapes.wave }),
    website:       genPageTheme({ colors: [cna.blueDark, cna.blueLight], shape: shapes.wave }),
    library:       genPageTheme({ colors: [cna.orange, cna.orangeLight], shape: shapes.wave2 }),
    other:         genPageTheme({ colors: [cna.blue, cna.blueDark],     shape: shapes.wave }),
    app:           genPageTheme({ colors: [cna.blue, cna.blueLight],    shape: shapes.wave }),
    apis:          genPageTheme({ colors: [cna.blueDark, cna.blue],     shape: shapes.wave2 }),
  },
});

// ─── EXTENSIONS ───────────────────────────────────────────────────────────────
// ThemeBlueprint.make({ name: 'light' }) registrado en el módulo 'app'
// genera el ID de extensión 'theme:app/light', que sobreescribe el tema
// built-in de Backstage. Sin pluginId:'app' + name:'light'/'dark', los temas
// se registran con IDs distintos y el sistema no los usa como reemplazo.
//
// themeName="cna-light" le dice a UnifiedThemeProvider el nombre MUI del tema.
// Sin este prop, el Provider siempre devuelve el tema "backstage" por defecto,
// haciendo que el cambio de colores no tenga efecto visual aunque esté
// correctamente registrado.

export const cnaLightThemeExtension = ThemeBlueprint.make({
  name: 'light',
  params: {
    theme: {
      id: 'light',
      title: 'Cloud Native Academy',
      variant: 'light',
      Provider: ({ children }) => (
        <UnifiedThemeProvider
          theme={cnaLightTheme}
          themeName="cna-light"
          children={children}
        />
      ),
    },
  },
});

export const cnaDarkThemeExtension = ThemeBlueprint.make({
  name: 'dark',
  params: {
    theme: {
      id: 'dark',
      title: 'Cloud Native Academy Dark',
      variant: 'dark',
      Provider: ({ children }) => (
        <UnifiedThemeProvider
          theme={cnaDarkTheme}
          themeName="cna-dark"
          children={children}
        />
      ),
    },
  },
});
```

---

## Paso 5: Actualizar el Logo de la Barra Lateral

El sidebar tiene tres archivos que debes modificar:
- `LogoFull.tsx` — logo completo cuando el sidebar está **expandido**
- `LogoIcon.tsx` — ícono pequeño cuando el sidebar está **colapsado**
- `SidebarLogo.tsx` — contenedor del logo (controla el ancho dinámico)

### 5a. Actualizar el contenedor SidebarLogo.tsx

El contenedor original usa el ancho del sidebar **colapsado** para ambos estados, lo que hace que el logo se vea pequeño cuando el sidebar está abierto. Edita `packages/app/src/modules/nav/SidebarLogo.tsx`:

```tsx
import {
  Link,
  sidebarConfig,
  useSidebarOpenState,
} from '@backstage/core-components';
import { makeStyles } from '@material-ui/core';
import { LogoFull } from './LogoFull';
import { LogoIcon } from './LogoIcon';

const useSidebarLogoStyles = makeStyles({
  root: {
    // Ancho dinámico: usa el ancho abierto cuando el sidebar está expandido
    width: ({ isOpen }: { isOpen: boolean }) =>
      isOpen ? sidebarConfig.drawerWidthOpen : sidebarConfig.drawerWidthClosed,
    height: 3 * sidebarConfig.logoHeight,
    display: 'flex',
    flexFlow: 'row nowrap',
    alignItems: 'center',
    marginBottom: -14,
    overflow: 'visible',
  },
  link: {
    width: ({ isOpen }: { isOpen: boolean }) =>
      isOpen ? sidebarConfig.drawerWidthOpen : sidebarConfig.drawerWidthClosed,
    marginLeft: 20,
    display: 'flex',
    alignItems: 'center',
  },
});

export const SidebarLogo = () => {
  const { isOpen } = useSidebarOpenState();
  const classes = useSidebarLogoStyles({ isOpen });

  return (
    <div className={classes.root}>
      <Link to="/" underline="none" className={classes.link} aria-label="Home">
        {isOpen ? <LogoFull /> : <LogoIcon />}
      </Link>
    </div>
  );
};
```

> **¿Por qué este cambio?** `sidebarConfig.drawerWidthClosed` es ~72px. Si el link del logo también tiene ese ancho, el SVG se recorta cuando el sidebar se abre (~224px). Pasando `isOpen` como prop a `makeStyles` obtenemos ancho dinámico.

### 5b. Opción A: Usar los PNG de Cloud Native Academy (recomendado)

Necesitas dos archivos PNG que el instructor provee en el material del curso:
- `logo.png` — logo completo con el texto "Cloud Native Academy" (fondo transparente, texto en navy oscuro)
- `logo-icon.png` — solo el símbolo de las dos nubes superpuestas, **sin texto**

> **¿Por qué no usar `logo.png` directamente en el sidebar?**
> El `logo.png` tiene el texto en navy oscuro (`#0D1B3E`). El sidebar también tiene fondo `#0D1B3E`, así que el texto quedaría invisible. La solución es usar solo el símbolo (`logo-icon.png`) y reproducir el texto "Cloud Native / Academy" con CSS en blanco.

1. Crea la carpeta de assets y copia los logos desde donde los tengas guardados:
```bash
mkdir -p packages/app/src/assets
cp /ruta/a/tu/logo.png packages/app/src/assets/logo.png
cp /ruta/a/tu/logo-icon.png packages/app/src/assets/logo-icon.png
```

> **Nota TypeScript:** Backstage incluye declaraciones de tipos para importar PNG como módulos. No necesitas configurar nada extra — `import logoIcon from '../../assets/logo-icon.png'` compila sin errores.

2. Edita `packages/app/src/modules/nav/LogoFull.tsx`:

```tsx
import logoIcon from '../../assets/logo-icon.png';

export const LogoFull = () => {
  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: 10 }}>
      <img
        src={logoIcon}
        alt=""
        style={{ height: 44, width: 'auto', flexShrink: 0 }}
      />
      <div style={{ display: 'flex', flexDirection: 'column', lineHeight: 1.25 }}>
        <span style={{
          fontFamily: "'Inter', -apple-system, BlinkMacSystemFont, sans-serif",
          fontSize: 14, fontWeight: 700,
          color: '#FFFFFF', letterSpacing: 0.2, whiteSpace: 'nowrap',
        }}>
          Cloud Native
        </span>
        <span style={{
          fontFamily: "'Inter', -apple-system, BlinkMacSystemFont, sans-serif",
          fontSize: 14, fontWeight: 400,
          color: '#B8C5D6', letterSpacing: 0.2, whiteSpace: 'nowrap',
        }}>
          Academy
        </span>
      </div>
    </div>
  );
};
```

3. Edita `packages/app/src/modules/nav/LogoIcon.tsx`:

```tsx
import logoIcon from '../../assets/logo-icon.png';

export const LogoIcon = () => {
  return (
    <img
      src={logoIcon}
      alt="Cloud Native Academy"
      style={{ height: 38, width: 'auto', display: 'block' }}
    />
  );
};
```

### 5c. Opción B: Usar SVG con los colores de marca (sin archivos externos)

Edita `packages/app/src/modules/nav/LogoFull.tsx`:

```tsx
export const LogoFull = () => {
  return (
    <svg
      xmlns="http://www.w3.org/2000/svg"
      viewBox="0 0 190 52"
      height="46"
      width="auto"
      aria-label="Cloud Native Academy"
      style={{ display: 'block' }}
    >
      {/* ── Ícono: dos nubes superpuestas estilo CNA ── */}

      {/* Nube naranja — inferior-izquierda */}
      <circle cx="16" cy="34" r="12" fill="#F5A623" />
      <circle cx="26" cy="27" r="12" fill="#F5A623" />
      <circle cx="9"  cy="27" r="8"  fill="#F5A623" />
      <rect   x="4"  y="27" width="34" height="19" rx="4" fill="#F5A623" />

      {/* Nube azul — superior-derecha */}
      <circle cx="27" cy="21" r="12" fill="#4B6CF7" />
      <circle cx="37" cy="13" r="12" fill="#4B6CF7" />
      <circle cx="43" cy="23" r="8"  fill="#4B6CF7" />
      <rect   x="15" y="13" width="36" height="18" rx="4" fill="#4B6CF7" />

      {/* Franja blanca separadora */}
      <line x1="5" y1="48" x2="47" y2="4"
        stroke="white" strokeWidth="7" strokeLinecap="round" />

      {/* ── Texto ── */}
      <text
        x="59" y="23"
        fontFamily="'Inter', -apple-system, BlinkMacSystemFont, sans-serif"
        fontSize="15" fontWeight="700"
        fill="#FFFFFF" letterSpacing="0.2"
      >
        Cloud Native
      </text>

      <text
        x="59" y="41"
        fontFamily="'Inter', -apple-system, BlinkMacSystemFont, sans-serif"
        fontSize="15" fontWeight="400"
        fill="#B8C5D6" letterSpacing="0.2"
      >
        Academy
      </text>
    </svg>
  );
};
```

Edita `packages/app/src/modules/nav/LogoIcon.tsx`:

```tsx
export const LogoIcon = () => {
  return (
    <svg
      xmlns="http://www.w3.org/2000/svg"
      viewBox="0 0 52 52"
      height="38"
      width="38"
      aria-label="CNA"
      style={{ display: 'block' }}
    >
      {/* Nube naranja — inferior-izquierda */}
      <circle cx="15" cy="36" r="13" fill="#F5A623" />
      <circle cx="26" cy="29" r="13" fill="#F5A623" />
      <circle cx="8"  cy="29" r="9"  fill="#F5A623" />
      <rect   x="4"  y="29" width="35" height="20" rx="5" fill="#F5A623" />

      {/* Nube azul — superior-derecha */}
      <circle cx="27" cy="23" r="13" fill="#4B6CF7" />
      <circle cx="38" cy="15" r="13" fill="#4B6CF7" />
      <circle cx="44" cy="26" r="9"  fill="#4B6CF7" />
      <rect   x="14" y="14" width="38" height="19" rx="5" fill="#4B6CF7" />

      {/* Franja blanca separadora */}
      <line x1="4" y1="48" x2="48" y2="4"
        stroke="white" strokeWidth="8" strokeLinecap="round" />
    </svg>
  );
};
```

---

## Paso 6: Registrar el tema en App.tsx

Edita `packages/app/src/App.tsx`.

**Agrega el import** del tema al principio del archivo:

```tsx
// Agrega esta línea junto a los demás imports
import { cnaLightThemeExtension, cnaDarkThemeExtension } from './theme/cloudNativeTheme';
```

**Registra las extensions** en el módulo `app` (IMPORTANTE: deben ir en `pluginId: 'app'` para sobrescribir los temas built-in de Backstage):

```tsx
// Busca el createFrontendModule({ pluginId: 'app', ... }) que ya existe
// y agrega los temas junto al signInPage

createFrontendModule({
  pluginId: 'app',
  extensions: [signInPage, cnaLightThemeExtension, cnaDarkThemeExtension],
  //           ^^^^^^^^^^ ya existía, agrega los dos temas nuevos
}),
```

Tu `App.tsx` debería verse así al final:

```tsx
import { createApp } from '@backstage/frontend-defaults';
import catalogPlugin from '@backstage/plugin-catalog/alpha';
import { navModule } from './modules/nav';
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPageBlueprint } from '@backstage/plugin-app-react';
import { SignInPage } from '@backstage/core-components';
import { createFrontendModule, createFrontendPlugin, PageBlueprint } from '@backstage/frontend-plugin-api';
import githubActionsPlugin from '@backstage-community/plugin-github-actions/alpha';
import apiDocsPlugin from '@backstage/plugin-api-docs/alpha';
import kubernetesPlugin from '@backstage/plugin-kubernetes/alpha';
import { cnaLightThemeExtension, cnaDarkThemeExtension } from './theme/cloudNativeTheme';

// ... (resto del archivo igual) ...

export default createApp({
  features: [
    catalogPlugin,
    navModule,
    githubActionsPlugin,
    apiDocsPlugin,
    kubernetesPlugin,
    // ... otros plugins ...
    createFrontendModule({
      pluginId: 'app',
      extensions: [signInPage, cnaLightThemeExtension, cnaDarkThemeExtension],
    }),
  ],
});
```

---

## Paso 7: Verificar el resultado

Inicia la aplicación:

```bash
yarn start
```

Abre el navegador en `http://localhost:3000`. Deberías ver:

| Elemento | Antes | Después |
|---|---|---|
| Título de la página | "Scaffolded Backstage App" | "Cloud Native Academy - Developer Portal" |
| Header de sign-in | Degradado verde/teal | Degradado azul `#4B6CF7 → #7B9EFF` |
| Botón "SIGN IN" | Azul Material por defecto | Azul CNA `#4B6CF7` |
| Sidebar (tras login) | Negro por defecto | Navy oscuro `#0D1B3E` |
| Indicador activo sidebar | Azul claro | Naranja CNA `#F5A623` |
| Logo en sidebar | Logo Backstage | Logo Cloud Native Academy |
| Fuente | Roboto | Inter (cuando carga Google Fonts) |

---

## Paso 8: Cambiar entre temas

El sistema de temas de Backstage permite múltiples temas. Tras iniciar sesión:

1. Ve a **Settings** (ícono de usuario en la barra lateral)
2. Busca la sección **Appearance**
3. Verás los temas: **"Cloud Native Academy"** (claro) y **"Cloud Native Academy Dark"** (oscuro)
4. Selecciona el que prefieras — la preferencia se guarda en `localStorage`

---

## Paso 9 (Opcional): Personalizar el favicon

Para completar la experiencia de marca, actualiza el favicon con el ícono de CNA:

1. Genera tus favicons desde el logo en: https://realfavicongenerator.net/
2. Copia los archivos generados a `packages/app/public/`
3. Los archivos a reemplazar son:
   - `favicon.ico`
   - `favicon-16x16.png`
   - `favicon-32x32.png`
   - `apple-touch-icon.png`
   - `android-chrome-192x192.png`

---

## Resumen de archivos modificados

```
my-backstage-app/
├── app-config.yaml                              # Título y organización actualizados
└── packages/app/
    ├── package.json                             # @backstage/theme añadido
    ├── public/
    │   └── index.html                           # theme-color + fuente Inter
    └── src/
        ├── App.tsx                              # Import y registro del tema
        ├── theme/
        │   └── cloudNativeTheme.tsx             # ★ NUEVO: tema CNA completo
        └── modules/nav/
            ├── SidebarLogo.tsx                  # Contenedor: ancho dinámico open/closed
            ├── LogoFull.tsx                     # Logo expandido (sidebar abierto)
            └── LogoIcon.tsx                     # Logo colapsado (sidebar cerrado)
```

---

## Conceptos clave aprendidos

### `createUnifiedTheme`
Función de `@backstage/theme` que crea un tema unificado compatible con Material UI v4 y v5. Acepta:
- **`palette`**: colores del sistema (primary, secondary, navigation, background...)
- **`fontFamily`**: fuente tipográfica global
- **`pageTheme`**: gradientes y formas de los headers por tipo de entidad
- **`defaultPageTheme`**: qué gradiente usar por defecto

**¿Por qué `...palettes.light` y `...palettes.dark`?**
MUI define ~50 campos de paleta (colores de errores, warnings, dividers, actions, overlays, etc.). Con el spread `...palettes.light` o `...palettes.dark` heredas todos esos valores por defecto de Backstage. Sin el spread, tendrías que definir todos los campos manualmente o quedarías con `undefined` en muchos colores — botones que no cambian de color al hacer hover, texto de error invisible, etc. Siempre incluye el spread y sobreescribe solo los colores que quieres personalizar.

---

### `genPageTheme`
Genera la configuración del header de cada página. Los parámetros son:
- **`colors`**: array de dos colores para el degradado
- **`shape`**: forma del fondo (`shapes.wave`, `shapes.wave2`, `shapes.round`, `shapes.sphere`, `shapes.none`)

Se define un `pageTheme` por cada tipo de entidad del catálogo de Backstage. Las claves disponibles son: `home`, `documentation`, `tool`, `service`, `website`, `library`, `other`, `app`, `apis`. Si el tipo de entidad no coincide con ninguna clave, se usa el `defaultPageTheme`.

---

### `ThemeBlueprint`
Extension blueprint de Backstage para registrar temas. Cada tema necesita:
- **`name`**: nombre de la extensión — `'light'` y `'dark'` son especiales porque sobreescriben los built-in
- **`id`** y **`title`**: identificador interno y nombre mostrado en la sección **Settings → Appearance**
- **`variant`**: `'light'` o `'dark'` — determina cuál se aplica automáticamente según el sistema operativo
- **`Provider`**: componente React que envuelve la app con el tema

---

### Sistema de IDs de extensiones (por qué `pluginId: 'app'` es crítico)
Backstage identifica cada extensión con el patrón `tipo:pluginId/nombre`. Un `ThemeBlueprint.make({ name: 'light' })` dentro de `createFrontendModule({ pluginId: 'app' })` genera el ID `theme:app/light`.

Backstage incluye de fábrica los temas `theme:app/light` y `theme:app/dark`. Cuando registramos nuestras extensiones con esos mismos IDs, el sistema de extensiones usa las nuestras en lugar de las built-in. Si usaras `pluginId: 'cna-theme'`, el ID sería `theme:cna-theme/light` — un tema nuevo pero los built-in seguirían activos y serían los que se mostrarían por defecto.

---

### `themeName` en `UnifiedThemeProvider`
`UnifiedThemeProvider` es el componente que aplica el tema de MUI a toda la app. Tiene un prop `themeName` que le indica qué nombre de tema interno usar.

Sin `themeName`, el Provider devuelve siempre el tema llamado `"backstage"` (el default hardcodeado en la librería), aunque el objeto `theme` que le pasas tenga tus colores CNA. El resultado: los colores del CSS quedan como Backstage por defecto, aunque el registro de la extensión funcione correctamente. Siempre proporciona un `themeName` único para tu tema.

---

### Palette `navigation`
Propiedad específica de Backstage (no parte del estándar de MUI) que controla la apariencia del sidebar. Backstage la lee internamente y la aplica a los componentes del sidebar:
- `background`: color de fondo del panel lateral
- `indicator`: barra de color a la izquierda del ítem activo
- `color`: color del texto e íconos de los ítems
- `selectedColor`: color del ítem actualmente activo
- `navItem.hoverBackground`: fondo al hacer hover sobre un ítem
- `submenu.background`: fondo de los submenús

---

### `makeStyles` con props dinámicas (MUI v4)
Backstage usa Material UI v4, que proporciona `makeStyles` de `@material-ui/core`. Una característica útil es que los valores de estilo pueden ser funciones que reciben props:

```tsx
const useStyles = makeStyles({
  root: {
    width: ({ isOpen }: { isOpen: boolean }) => isOpen ? 224 : 72,
  },
});

const MyComponent = () => {
  const { isOpen } = useSidebarOpenState();
  const classes = useStyles({ isOpen }); // se pasan las props aquí
  return <div className={classes.root} />;
};
```

Esto permite estilos CSS que reaccionan al estado del componente sin necesidad de `style={{ }}` inline ni clases CSS condicionales. Se usa en `SidebarLogo.tsx` para que el ancho del contenedor del logo cambie con el estado del sidebar.

---

## Troubleshooting

### El tema no se aplica (sigue viendo el tema por defecto de Backstage)

**Causa más común:** las extensiones del tema no están en el módulo `app`.

Verifica que en `App.tsx` tengas:
```tsx
createFrontendModule({
  pluginId: 'app',          // ← debe ser exactamente 'app'
  extensions: [signInPage, cnaLightThemeExtension, cnaDarkThemeExtension],
}),
```
Si los temas están en `createFrontendPlugin({ pluginId: 'cna-theme', ... })` o en otro `pluginId`, generan IDs como `theme:cna-theme/light` en lugar de `theme:app/light` y no sobreescriben los built-in.

**Segunda causa:** el `localStorage` guarda el tema anterior. Limpia el storage del navegador: DevTools → Application → Local Storage → selecciona `http://localhost:3000` → Clear All → recarga.

---

### Error: "Cannot find module '@backstage/theme'"

El paquete no está declarado como dependencia directa del workspace `app`. Sigue el Paso 1:
1. Agrega `"@backstage/theme": "^0.7.3"` en `packages/app/package.json`
2. Ejecuta `yarn install` desde la raíz del proyecto (no desde `packages/app/`)

---

### Los colores cambian pero el sidebar sigue negro

El sidebar usa la paleta `navigation` de Backstage, que es diferente a `palette.primary` y `palette.secondary`. Verifica que tu tema tenga el bloque `navigation` dentro de `palette`:

```tsx
palette: {
  ...palettes.light,
  navigation: {
    background: '#0D1B3E',  // ← este campo controla el sidebar
    indicator: '#F5A623',
    color: '#B8C5D6',
    selectedColor: '#FFFFFF',
  },
  ...
}
```

---

### Error TypeScript: "Cannot find module '../../assets/logo-icon.png'"

Backstage incluye declaraciones de tipos para PNG en su tsconfig base. Verifica que `packages/app/tsconfig.json` extienda del tsconfig de Backstage:
```json
{
  "extends": "@backstage/cli/config/tsconfig.json",
  ...
}
```
Si esa línea existe, TypeScript ya reconoce los imports de `.png`. Si sigue fallando, asegúrate de que el archivo `logo-icon.png` exista físicamente en `packages/app/src/assets/`.

---

### Error: "'React' is declared but never read (TS6133)"

Este proyecto usa `"jsx": "react-jsx"` (nuevo transform de React). No necesitas `import React from 'react'` en archivos `.tsx`. Si lo tienes, elimínalo.

---

### La fuente Inter no cambia

La fuente Google Fonts carga de forma asíncrona desde internet. Si estás en una red lenta o sin conexión, se usará el fallback `-apple-system, BlinkMacSystemFont` (que también se ve bien en Mac/iOS). Para self-hostear la fuente sin depender de Google:

```bash
yarn workspace app add @fontsource/inter
```

Luego agrega en el entry point de la app (por ejemplo al inicio de `App.tsx`):
```tsx
import '@fontsource/inter/300.css';
import '@fontsource/inter/400.css';
import '@fontsource/inter/500.css';
import '@fontsource/inter/600.css';
import '@fontsource/inter/700.css';
```
