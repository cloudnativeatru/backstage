# Ejercicio Propuesto 08: Dockerizar Backstage

**Tiempo estimado:** 45 minutos  
**Herramientas permitidas:** Claude, GitHub Copilot, ChatGPT — cualquier IA  
**Dificultad:** Media  

---

## Objetivo

Levantar Backstage completamente dockerizado con PostgreSQL como base de datos, usando `docker compose`. Al finalizar, el portal debe ser accesible en `http://localhost:7007` corriendo dentro de contenedores — sin depender del servidor de desarrollo local.

---

## Contexto

El proyecto ya tiene todo lo necesario para dockerizarlo. Antes de escribir cualquier comando, explora estos archivos:

- `packages/backend/Dockerfile` — ya existe, generado por Backstage
- `packages/backend/package.json` — tiene un script llamado `build-image`
- `app-config.yaml` — configuración de desarrollo
- `app-config.production.yaml` — configuración de producción (observa la sección `database`)
- `.dockerignore` — ya existe en la raíz

Entiende qué hace cada uno **antes** de ejecutar nada. Usa IA si necesitas que te expliquen instrucciones del Dockerfile que no reconoces.

---

## Lo que debes lograr

1. **Compilar el proyecto** para que el Dockerfile tenga los artefactos que necesita
2. **Construir la imagen Docker** usando el script que ya existe en el proyecto
3. **Crear `docker-compose.yml`** en la raíz del proyecto con dos servicios:
   - `backstage` — usando la imagen que construiste
   - `db` — PostgreSQL con un volumen para persistir datos
4. **Levantar los contenedores** y verificar que el portal carga en el navegador

---

## Pistas

- El `app-config.production.yaml` te dice exactamente qué variables de entorno necesita el contenedor de Backstage para conectarse a la base de datos
- Dentro de Docker Compose, los servicios se comunican por nombre — no por `localhost`
- La imagen base del Dockerfile no incluye `curl`. Tenlo en cuenta si intentas agregar un healthcheck
- `docker compose down` y `docker compose down -v` hacen cosas diferentes. Investiga cuál quieres usar para desarrollo y cuál para producción

---

## Criterios de evaluación

| Criterio | Puntos |
|---|---|
| La imagen Docker se construye sin errores | 20 |
| El portal carga en `http://localhost:7007` | 30 |
| Se puede iniciar sesión como Guest | 20 |
| El volumen de PostgreSQL persiste datos al reiniciar | 15 |
| El `docker-compose.yml` está bien estructurado y sin credenciales hardcodeadas en texto plano | 15 |
| **Total** | **100** |

**Puntos extra:**
- +15 Los servicios tienen healthchecks y el `depends_on` usa `condition: service_healthy`
- +10 Las credenciales están en un archivo `.env` separado (no directamente en el compose)

---

## Cómo verificar que funciona

- [ ] `docker ps` muestra dos contenedores corriendo (`backstage` y `db`)
- [ ] `http://localhost:7007` carga la pantalla de sign-in
- [ ] Login como Guest funciona y el catálogo muestra servicios
- [ ] Tras `docker compose down && docker compose up` el catálogo sigue teniendo datos (volumen persistente)

---

## Preguntas de análisis

Reflexiona sobre estas preguntas y discútelas con tu equipo o en clase:

1. ¿Por qué el Dockerfile no compila el código dentro del contenedor? ¿Qué ventajas y desventajas tiene ese enfoque?
2. ¿Por qué el proceso de Node corre con un usuario no-root dentro del contenedor?
3. ¿Cuándo usarías `docker compose down -v`? ¿En qué situación sería peligroso?
4. Si un compañero cambia el código fuente de la app, ¿qué pasos hay que repetir para que el contenedor refleje los cambios?

---

## Entrega

Comparte:
1. Tu `docker-compose.yml`
2. Captura del portal funcionando en `http://localhost:7007`
3. Salida de `docker ps` con los dos contenedores corriendo
