# Ejercicio Propuesto 08: Dockerizar Backstage

**Tiempo estimado:** 45 minutos  
**Herramientas permitidas:** Claude, GitHub Copilot, ChatGPT — cualquier IA  
**Dificultad:** Media  

---

## Objetivo

Levantar Backstage en un contenedor Docker con PostgreSQL como base de datos, usando `docker compose`. Al finalizar, el portal debe ser accesible en `http://localhost:7007` corriendo completamente dentro de contenedores — sin depender del servidor de desarrollo local.

---

## Contexto

En desarrollo usamos `yarn start`, que levanta el frontend y el backend por separado con hot-reload. En producción eso no sirve: necesitamos una imagen Docker con el build compilado, una base de datos persistente (no SQLite in-memory), y todo orquestado con `docker compose`.

El proyecto ya tiene la infraestructura preparada:

| Archivo | Qué hace |
|---|---|
| `packages/backend/Dockerfile` | Imagen del backend compilado |
| `.dockerignore` | Excluye `node_modules`, cache, código fuente |
| `app-config.yaml` | Config de desarrollo (SQLite en memoria) |
| `app-config.production.yaml` | Config de producción (PostgreSQL via env vars) |

Tu tarea: entender cómo encajan estas piezas, compilar el proyecto, construir la imagen y crear el `docker-compose.yml`.

---

## Paso 1: Leer y entender el Dockerfile existente

Antes de ejecutar nada, abre `packages/backend/Dockerfile` y responde estas preguntas:

- ¿Qué imagen base usa?
- ¿Por qué instala `python3`, `g++` y `build-essential`?
- ¿Por qué instala `libsqlite3-dev` si en producción usamos PostgreSQL?
- ¿Por qué copia primero `skeleton.tar.gz` y luego `bundle.tar.gz` en lugar de copiar todo de una vez?
- ¿Con qué usuario corre el proceso de Node? ¿Por qué no usa `root`?
- ¿Qué archivos de configuración carga al iniciar el contenedor?

> **Tip:** Usa una IA para que te explique las instrucciones que no entiendas. Entender el Dockerfile antes de ejecutarlo es parte del ejercicio.

---

## Paso 2: Compilar el backend

El Dockerfile **no compila el código dentro de Docker** — espera encontrar los artefactos ya compilados antes de hacer el `docker build`. Esto mantiene la imagen final más liviana y el build más predecible.

Desde la raíz del proyecto ejecuta en orden:

```bash
# 1. Instalar dependencias (si no lo hiciste antes)
yarn install --immutable

# 2. Compilar los tipos TypeScript compartidos
yarn tsc

# 3. Compilar el backend (genera packages/backend/dist/)
yarn build:backend
```

Verifica que el build fue exitoso:

```bash
ls packages/backend/dist/
```

Deberías ver `skeleton.tar.gz`, `bundle.tar.gz` y otros artefactos.

---

## Paso 3: Construir la imagen Docker

El proyecto tiene un script que construye la imagen con el contexto correcto:

```bash
yarn build-image
```

Esto ejecuta internamente:
```bash
docker build ../.. -f Dockerfile --tag backstage
```

> **¿Por qué `../..` como contexto?** Docker necesita acceso al directorio raíz del monorepo para copiar el `yarn.lock`, el `.yarnrc.yml`, los `package.json` de todos los workspaces y los artefactos compilados.

Verifica que la imagen fue creada:

```bash
docker images | grep backstage
```

---

## Paso 4: Entender app-config.production.yaml

Abre `app-config.production.yaml`. Observa que:

- La base de datos usa `pg` (PostgreSQL) en lugar de SQLite
- Las credenciales vienen de **variables de entorno**: `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
- El autenticador usa `guest: {}` — en producción no requiere GitHub OAuth
- Las locations del catálogo apuntan a archivos locales dentro del contenedor

Este archivo se carga **además** de `app-config.yaml` cuando el contenedor arranca, sobreescribiendo las configuraciones de desarrollo.

---

## Paso 5: Crear docker-compose.yml

Crea el archivo `docker-compose.yml` en la **raíz del proyecto** (`my-backstage-app/docker-compose.yml`).

Debe definir **dos servicios**:

### Servicio `backstage`

- Usa la imagen `backstage` (la que construiste en el Paso 3)
- Expone el puerto `7007`
- Define las variables de entorno que `app-config.production.yaml` espera:
  - `POSTGRES_HOST` — debe apuntar al nombre del servicio de base de datos
  - `POSTGRES_PORT` — puerto estándar de PostgreSQL
  - `POSTGRES_USER` — el usuario que configuraste en el servicio de DB
  - `POSTGRES_PASSWORD` — la contraseña que configuraste en el servicio de DB
- Debe esperar a que la base de datos esté lista antes de arrancar (`depends_on`)

### Servicio `db`

- Usa la imagen oficial `postgres:15`
- Define las variables de entorno de PostgreSQL:
  - `POSTGRES_USER`
  - `POSTGRES_PASSWORD`
  - `POSTGRES_DB`
- Usa un **volumen** para persistir los datos entre reinicios

> **Pista:** el valor de `POSTGRES_HOST` en el servicio `backstage` debe ser exactamente el nombre del servicio de base de datos que definas en el compose. Docker Compose crea una red interna donde los servicios se resuelven por nombre.

---

## Paso 6: Levantar los contenedores

```bash
docker compose up
```

Observa los logs. Deberías ver:
1. PostgreSQL inicializando y aceptando conexiones
2. Backstage conectándose a la base de datos
3. El mensaje de que el backend está escuchando en el puerto 7007

Abre el navegador en:
```
http://localhost:7007
```

---

## Paso 7: Verificar que funciona correctamente

Lista de verificación:

- [ ] La página de sign-in carga en `http://localhost:7007`
- [ ] Puedes iniciar sesión con **"Enter as Guest"**
- [ ] El catálogo muestra los servicios de ejemplo
- [ ] El sidebar muestra el logo y los colores de Cloud Native Academy (el tema del ejercicio 11)
- [ ] Al reiniciar los contenedores (`docker compose down && docker compose up`) los datos persisten gracias al volumen

---

## Preguntas de análisis

Responde brevemente estas preguntas (puedes usar IA para investigar):

1. **¿Qué pasa si cambias el código fuente de la app?** ¿El contenedor se actualiza automáticamente o hay que hacer algo?

2. **¿Por qué el Dockerfile copia primero `skeleton.tar.gz` y luego `bundle.tar.gz` en pasos separados?** ¿Qué ventaja tiene para el cache de Docker?

3. **¿Cuál es la diferencia entre `docker compose down` y `docker compose down -v`?** ¿Cuándo usarías cada uno?

4. **¿Por qué es una mala práctica correr el proceso de Node como `root` dentro del contenedor?**

---

## Requerimiento opcional

Agrega un **healthcheck** al servicio `backstage` en el `docker-compose.yml` que verifique que el backend responde correctamente antes de considerarlo "healthy":

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:7007/healthcheck"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 60s
```

Modifica el `depends_on` del servicio backstage para que espere a que la base de datos esté healthy:

```yaml
depends_on:
  db:
    condition: service_healthy
```

Y agrega también un healthcheck al servicio `db` usando `pg_isready`.

---

## Troubleshooting

### El contenedor de Backstage arranca pero no puede conectarse a PostgreSQL
Verifica que el valor de `POSTGRES_HOST` en el servicio `backstage` sea exactamente el nombre del servicio de base de datos en el compose (no `localhost`). Dentro de Docker Compose, los servicios se comunican por nombre de servicio.

### Error "Cannot find module" al levantar el contenedor
El build del paso 3 no se completó correctamente. Vuelve a ejecutar `yarn build:backend` y luego `yarn build-image`.

### La base de datos se reinicia vacía cada vez
Verifica que el volumen esté declarado correctamente en el `docker-compose.yml` y que no estés usando `docker compose down -v` (que elimina los volúmenes).

### Puerto 5432 ya en uso
Tienes PostgreSQL corriendo localmente. O detén el servicio local (`brew services stop postgresql`) o mapea el puerto del contenedor a uno diferente (ej. `5433:5432`) en el compose.

---

## Entrega

Comparte:
1. El contenido de tu `docker-compose.yml`
2. Una captura de pantalla del portal funcionando en `http://localhost:7007` dentro de Docker
3. La salida del comando `docker ps` mostrando los dos contenedores corriendo
