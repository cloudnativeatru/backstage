# Ejercicio Propuesto 07: Panel de Bienvenida del Developer Portal

**Tiempo estimado:** 45 minutos  
**Herramientas permitidas:** Claude, GitHub Copilot, ChatGPT — cualquier IA  
**Dificultad:** Media  

---

## Objetivo

Crear una página de bienvenida personalizada en la ruta `/welcome` para el Developer Portal de Cloud Native Academy. La página debe funcionar como punto de entrada visual al portal, mostrar cards de acceso rápido a las secciones principales, y respetar la identidad visual CNA establecida en el ejercicio anterior.

Al finalizar, el resultado debe verse como un dashboard de bienvenida real: no una página en blanco con texto, sino algo que un equipo de ingeniería usaría en producción.

---

## Contexto del proyecto

El proyecto ya tiene páginas personalizadas registradas. Puedes usarlas como referencia directa:

- El **patrón para registrar una nueva página** está en `packages/app/src/App.tsx` — busca `chatbotPage` o `diagramPage`
- El **patrón para agregar un ítem al sidebar** está en `packages/app/src/modules/nav/Sidebar.tsx` — busca los `SidebarItem` con `ChatIcon`, `AccountTreeIcon`, etc.
- Los **colores de la paleta CNA** ya están definidos en `packages/app/src/theme/cloudNativeTheme.tsx`
- Los **logos** están en `packages/app/src/assets/`

---

## Lo que debes construir

### Requerimientos obligatorios

**1. Componente de la página**  
Crea el archivo `packages/app/src/components/WelcomePage/WelcomePage.tsx`.

La página debe contener:

- **Hero section**: logo de CNA, título "Bienvenido al Developer Portal" (o similar), subtítulo descriptivo
- **Grid de cards de acceso rápido**: mínimo **4 cards**, cada una con:
  - Un ícono representativo (usa `@material-ui/icons`)
  - Un título de sección (ej. "Catálogo de Servicios", "APIs", "Documentación", "CI/CD")
  - Una descripción corta (1-2 oraciones)
  - Un botón o link que navegue a esa sección del portal
- **Usa los colores CNA**: al menos 3 de los colores de la paleta (`#4B6CF7`, `#F5A623`, `#0D1B3E`, `#7B9EFF`, etc.)

**2. Registro de la página**  
En `packages/app/src/App.tsx`:
- Registra la página en la ruta `/welcome`
- Agrégala a `features` de `createApp`

**3. Link en el sidebar**  
En `packages/app/src/modules/nav/Sidebar.tsx`:
- Agrega un `SidebarItem` con un ícono apropiado (sugerencia: `HomeIcon` o `DashboardIcon`) que navegue a `/welcome`
- Ponlo como el **primer ítem** del menú, antes del catálogo

---

## Requerimientos opcionales (para subir la nota)

- La página se ve bien tanto en **tema claro como oscuro** (usa `useTheme()` de MUI para adaptar colores al tema activo)
- Agrega una **sección de stats** debajo de las cards: número de servicios en el catálogo, APIs registradas, etc. (pueden ser valores hardcodeados por ahora)
- Las cards tienen un efecto **hover** visible (cambio de sombra o borde)
- La página tiene un **footer** con el nombre del equipo y la fecha

---

## Pistas técnicas

### Estructura de una card de acceso rápido

Usa los componentes de MUI v4 (`@material-ui/core`):

```tsx
import { Card, CardContent, Grid, Typography, Button } from '@material-ui/core';
```

Un grid de 4 cards en 2 columnas:
```tsx
<Grid container spacing={3}>
  <Grid item xs={12} sm={6} md={3}>
    <Card>
      <CardContent>
        {/* tu contenido */}
      </CardContent>
    </Card>
  </Grid>
  {/* ... más cards */}
</Grid>
```

### Navegar entre páginas

Usa el componente `Link` de `@backstage/core-components` (el mismo que se usa en `SidebarLogo.tsx`):

```tsx
import { Link } from '@backstage/core-components';
// ...
<Link to="/catalog">Ver catálogo</Link>
```

O el `Button` de MUI con el `component` prop:
```tsx
<Button component={Link} to="/catalog" variant="contained">
  Ver catálogo
</Button>
```

### Adaptar estilos al tema activo

Para que el componente funcione bien en tema claro y oscuro:

```tsx
import { useTheme } from '@material-ui/core';

const MyComponent = () => {
  const theme = useTheme();
  // theme.palette.type === 'light' | 'dark'
  // theme.palette.background.default
  // theme.palette.primary.main  → en CNA: #4B6CF7
  // theme.palette.secondary.main → en CNA: #F5A623
};
```

### Ícono para el sidebar

```tsx
import DashboardIcon from '@material-ui/icons/Dashboard';
// ...
<SidebarItem icon={DashboardIcon} to="/welcome" text="Inicio" />
```

---

## Archivos que debes crear o modificar

```
packages/app/src/
├── components/
│   └── WelcomePage/
│       └── WelcomePage.tsx          ← CREAR (nuevo)
├── App.tsx                          ← MODIFICAR (registrar página)
└── modules/nav/
    └── Sidebar.tsx                  ← MODIFICAR (agregar SidebarItem)
```

---

## Criterios de evaluación

| Criterio | Puntos |
|---|---|
| La página es accesible en `/welcome` y no rompe el resto del portal | 20 |
| Tiene hero section con logo y texto de bienvenida | 20 |
| Tiene al menos 4 cards de acceso rápido con ícono, título y descripción | 25 |
| Los botones/links de las cards navegan correctamente a sus secciones | 15 |
| Usa los colores de la paleta CNA (mínimo 3) | 10 |
| Hay un SidebarItem que lleva a `/welcome` | 10 |
| **Total** | **100** |

**Puntos extra:**
- +10 La página adapta colores al tema claro/oscuro con `useTheme()`
- +5 Las cards tienen efecto hover visible
- +5 Hay una sección de stats o información adicional

---

## Cómo probar que funciona

```bash
yarn start
```

Lista de verificación:
- [ ] Abrir `http://localhost:3000/welcome` directamente en el navegador — debe cargar la página sin errores
- [ ] El sidebar tiene un ítem "Inicio" (o similar) y al hacer clic va a `/welcome`
- [ ] Cada card tiene su botón funcional — al hacer clic navega a la sección correcta
- [ ] Cambiar el tema en Settings → Appearance y recargar `/welcome` — la página no debe quedar con colores hardcodeados ilegibles

---

## Recordatorio: lo que NO tienes que tocar

- `cloudNativeTheme.tsx` — el tema ya está configurado; la página hereda los colores automáticamente
- `packages/app/public/index.html` — ya tiene Inter y el theme-color correcto
- `app-config.yaml` — el título ya está configurado

---

## Entrega

Comparte una captura de pantalla de la página `/welcome` funcionando en el portal, con el sidebar visible mostrando el nuevo ítem de navegación.
