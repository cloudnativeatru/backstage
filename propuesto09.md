# Ejercicio Propuesto 09: Golden Path Serverless con Backstage Software Templates

**Tiempo estimado:** 75-90 minutos  
**Herramientas permitidas:** Claude, GitHub Copilot, ChatGPT — cualquier IA  
**Dificultad:** Alta  

---

## Prerequisitos

- Backstage corriendo localmente (`yarn start`)
- Cuenta de GitHub con un token de acceso personal configurado en Backstage (`GITHUB_TOKEN`)
- Cuenta de AWS con permisos para crear recursos Lambda e IAM
- **Terraform CLI** instalado
- **AWS CLI** configurado

---

## Escenario

El equipo de plataforma recibe constantemente el mismo pedido: "necesito un endpoint serverless". Hoy ese proceso es manual — el desarrollador crea un repo, configura el pipeline, escribe el Terraform, registra el servicio en el catálogo. Todo a mano, todo diferente entre equipos.

Tu trabajo como **platform engineer** es construir el **golden path**: un Software Template en Backstage que, cuando un desarrollador complete un formulario y haga clic en "Create", provisione automáticamente todo lo necesario y lo deje listo para hacer push de código.

**El desarrollador experimenta esto:**
1. Abre Backstage → hace clic en **Create**
2. Completa el formulario: nombre de la función, descripción, dueño
3. Hace clic en **Create Component**
4. Backstage crea el repositorio en GitHub, genera los archivos scaffolded y registra el servicio en el catálogo
5. El desarrollador clona el repo, edita `src/handler.js` y hace push — el pipeline despliega automáticamente

---

## Lo que debes construir

### Parte 1 — Repositorio de templates (`cna-templates`)

Crea un repositorio en GitHub llamado `cna-templates` con esta estructura. Observa que hay **dos skeletons separados** — uno por cada repositorio que se creará:

```
cna-templates/
└── serverless-function/
    ├── template.yaml               # Definición del Software Template
    ├── skeleton-app/               # Archivos del repo de código
    │   ├── src/
    │   │   └── handler.js
    │   ├── .github/
    │   │   └── workflows/
    │   │       └── deploy.yml
    │   ├── catalog-info.yaml
    │   └── README.md
    └── skeleton-infra/             # Archivos del repo de infraestructura
        ├── terraform/
        │   ├── main.tf
        │   ├── variables.tf
        │   ├── outputs.tf
        │   └── provider.tf
        ├── catalog-info.yaml
        └── README.md
```

---

### Parte 2 — El Software Template (`template.yaml`)

El `template.yaml` es el corazón del ejercicio. Debe definir:

**Metadata:**
- `apiVersion: scaffolder.backstage.io/v1beta3`
- `kind: Template`
- Nombre, descripción, tags (`serverless`, `aws`, `terraform`)

**Parámetros del formulario (`spec.parameters`):**
El formulario que ve el desarrollador en Backstage debe pedir:
- Nombre de la función (base para el nombre de ambos repositorios y de la Lambda)
- Descripción del servicio
- Dueño del componente (owner)
- Entorno destino (`dev` o `staging`)

**Pasos (`spec.steps`):**
El template debe ejecutar **cinco pasos** en orden:

1. `fetch:template` desde `skeleton-app/` → directorio temporal `app/`
2. `publish:github` publicando desde `app/` → crea el repo `cna-{name}-app`
3. `fetch:template` desde `skeleton-infra/` → directorio temporal `infra/`
4. `publish:github` publicando desde `infra/` → crea el repo `cna-{name}-infra`
5. `catalog:register` → registra el componente de aplicación en el catálogo

> **Pista clave:** El Scaffolder trabaja en un workspace temporal. Cada paso `fetch:template` usa el parámetro `targetPath` para escribir en un subdirectorio distinto (`app/` e `infra/`). Cada paso `publish:github` usa `sourcePath` para publicar solo ese subdirectorio. Investiga esta combinación de parámetros en la documentación del Scaffolder.

---

### Parte 3 — Los archivos skeleton

Los archivos en ambos skeletons usan la sintaxis `${{ values.nombre }}` para inyectar los valores del formulario.

**`skeleton-app/src/handler.js`**  
Función Lambda Node.js que responda con JSON incluyendo un mensaje de bienvenida, el nombre del servicio (inyectado) y un timestamp.

**`skeleton-app/.github/workflows/deploy.yml`**  
Pipeline que en push a `main` empaqueta el código y ejecuta `aws lambda update-function-code`. **No ejecuta Terraform** — la infra es responsabilidad del otro repo. Las credenciales AWS vienen de GitHub Secrets.

**`skeleton-app/catalog-info.yaml`**  
Entidad `Component` con nombre, descripción y dueño inyectados. Debe tener una anotación que lo vincule al repo de infra como dependencia.

**`skeleton-infra/terraform/main.tf`**  
Terraform que crea:
- IAM Role de ejecución con permisos mínimos
- Función Lambda con nombre derivado del input del formulario
- Lambda Function URL con acceso público

El nombre de la función y el entorno deben venir de `variables.tf` — no hardcodeados.

**`skeleton-infra/catalog-info.yaml`**  
Entidad `Resource` (no `Component`) que representa la infraestructura de la Lambda.

---

### Parte 4 — Registrar el template en Backstage

Para que el template aparezca en la pantalla **Create** de Backstage, debe estar registrado en el catálogo. Investiga cómo hacer esto — hay al menos dos formas:
- Agregarlo a `catalog.locations` en `app-config.yaml`
- Usar la pantalla de **Register Existing Component** en Backstage

---

### Parte 5 — Usarlo como desarrollador

Una vez registrado el template:
1. Ve a Backstage → **Create**
2. Busca tu template y completa el formulario
3. Verifica que GitHub tiene el nuevo repositorio con todos los archivos generados correctamente
4. Verifica que el nuevo servicio aparece en el **catálogo** de Backstage

---

## Pistas

- Cada par `fetch:template` + `publish:github` debe usar `targetPath` / `sourcePath` para no mezclarse en el workspace temporal. Si no los usas, el segundo `fetch:template` sobreescribe los archivos del primero
- Los nombres de los dos repos deben ser predecibles y relacionados: `cna-{name}-app` y `cna-{name}-infra`. El `name` viene del formulario. Investiga cómo construir strings dinámicos en el `repoUrl` del paso `publish:github`
- El pipeline del repo `app` necesita saber el nombre de la Lambda creada por el repo `infra`. Con naming convention basado en el mismo `name` del formulario, ambos repos pueden derivarlo sin comunicarse directamente
- Si el template falla en Backstage, los logs del Scaffolder muestran exactamente en qué paso y por qué — es la primera herramienta de debugging
- `catalog:register` al final solo necesita registrar el `catalog-info.yaml` del repo app — pero puedes registrar también el del repo infra si quieres que ambos aparezcan en el catálogo

---

## Criterios de evaluación

| Criterio | Puntos |
|---|---|
| El template aparece en la pantalla Create de Backstage | 10 |
| El formulario tiene los parámetros correctos y se muestra bien | 10 |
| Se crean **dos repositorios** en GitHub: `cna-{name}-app` y `cna-{name}-infra` | 25 |
| Las variables del formulario se inyectan correctamente en ambos skeletons | 15 |
| El nuevo servicio aparece registrado en el catálogo de Backstage | 10 |
| `terraform apply` en el repo infra crea la Lambda sin errores | 15 |
| El pipeline del repo app despliega el código sin tocar Terraform | 15 |
| **Total** | **100** |

**Puntos extra:**
- +10 El formulario tiene validaciones (nombre sin espacios, longitud máxima)
- +10 El repo infra también aparece como entidad `Resource` en el catálogo de Backstage
- +5 Los `README.md` de ambos repos generados explican claramente su responsabilidad

---

## Cómo verificar que funciona

- [ ] El template aparece en Backstage → Create con su nombre, descripción y tags
- [ ] Al completar el formulario, el Scaffolder crea **dos repos** en GitHub sin errores
- [ ] Ningún archivo generado contiene `${{ values.name }}` sin reemplazar
- [ ] El nuevo componente aparece en el catálogo con dueño y descripción correctos
- [ ] `terraform apply` en `cna-{name}-infra` crea la Lambda y muestra la Function URL como output
- [ ] Un push al repo `cna-{name}-app` despliega el código y la URL responde con el JSON esperado

---

## Preguntas de análisis

1. El template crea dos repos con un único clic. ¿Qué pasaría si el segundo `publish:github` falla después de que el primero ya creó el repo app? ¿Queda el catálogo y GitHub en un estado inconsistente? ¿Cómo lo resolverías?

2. El pipeline del repo app necesita saber el nombre de la Lambda creada por el repo infra. ¿Cómo lo resuelves con naming convention? ¿Qué pasa si en el futuro el equipo de plataforma decide cambiar el naming? ¿Quién se ve afectado?

3. El repo infra tiene su propio ciclo de vida: lo provisiona el platform engineer y no debería ser tocado por el desarrollador. ¿Cómo evitarías que el desarrollador haga push accidentalmente al repo infra? ¿Qué configuraciones de GitHub usarías?

4. Un desarrollador usó el template hace seis meses. El equipo de plataforma actualiza el template para usar una versión más nueva de Node.js. ¿Los repos creados antes se actualizan automáticamente? ¿Qué implicaciones tiene esto a escala (100 funciones creadas con el template)?

5. ¿Por qué el `catalog-info.yaml` del repo infra usa `kind: Resource` y el del repo app usa `kind: Component`? ¿Qué diferencia semántica tiene en el catálogo de Backstage?

---

## Entrega

Comparte:
1. Link al repositorio `cna-templates` con el `template.yaml` y los dos skeletons
2. Captura del template en la pantalla **Create** de Backstage
3. Captura del Scaffolder ejecutando los cinco pasos exitosamente
4. Links a los **dos repositorios** generados en GitHub (`cna-{name}-app` y `cna-{name}-infra`)
5. El nuevo servicio visible en el catálogo de Backstage
6. Resultado del `curl` a la Function URL con la respuesta JSON
