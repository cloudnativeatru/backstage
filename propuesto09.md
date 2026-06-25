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

Crea un repositorio en GitHub llamado `cna-templates` con esta estructura:

```
cna-templates/
└── serverless-function/
    ├── template.yaml               # Definición del Software Template
    └── skeleton/                   # Archivos que se generan al usar el template
        ├── src/
        │   └── handler.js
        ├── terraform/
        │   ├── main.tf
        │   ├── variables.tf
        │   ├── outputs.tf
        │   └── provider.tf
        ├── .github/
        │   └── workflows/
        │       └── deploy.yml
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
- Nombre de la función (el nombre del repositorio y de la Lambda)
- Descripción del servicio
- Dueño del componente (owner)
- Entorno destino (`dev` o `staging`)

**Pasos (`spec.steps`):**
El template debe ejecutar tres acciones en orden:
1. `fetch:template` — copia los archivos de `skeleton/` al repo destino, reemplazando las variables del formulario
2. `publish:github` — crea el repositorio en GitHub con el resultado del paso anterior
3. `catalog:register` — registra el nuevo servicio en el catálogo de Backstage

---

### Parte 3 — Los archivos skeleton

Los archivos en `skeleton/` son plantillas — usan la sintaxis `${{ values.nombre }}` para inyectar los valores que ingresó el desarrollador en el formulario.

**`skeleton/src/handler.js`**  
Una función Lambda Node.js que responda con JSON incluyendo un mensaje de bienvenida, el nombre del servicio (inyectado como variable) y un timestamp.

**`skeleton/terraform/main.tf`**  
Terraform que crea:
- IAM Role de ejecución con permisos mínimos
- Función Lambda con nombre derivado del input del formulario
- Lambda Function URL con acceso público

Los valores variables (nombre de la función, entorno) deben venir de `variables.tf` — no hardcodeados.

**`skeleton/.github/workflows/deploy.yml`**  
Pipeline que en push a `main` empaqueta el código y ejecuta `aws lambda update-function-code`. Las credenciales AWS vienen de GitHub Secrets.

**`skeleton/catalog-info.yaml`**  
Entidad de tipo `Component` con el nombre, descripción y dueño inyectados desde el formulario.

> **Pista:** Los archivos skeleton usan `${{ values.name }}`, `${{ values.description }}`, `${{ values.owner }}` — las mismas variables que defines en `spec.parameters` del `template.yaml`. Revisa la documentación de Backstage Scaffolder para ver la sintaxis exacta.

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

- El `template.yaml` referencia los archivos skeleton con una ruta relativa. La estructura de carpetas importa
- En `spec.steps`, cada paso `fetch:template` + `publish:github` trabajan en un workspace temporal — investiga cómo encadenan su output
- El nombre del repositorio en GitHub generalmente se deriva del parámetro `name` del formulario. Investiga el campo `repoUrl` en el paso `publish:github`
- `catalog-info.yaml` dentro del skeleton es lo que permite que `catalog:register` funcione — ese archivo le dice a Backstage qué tipo de entidad es el nuevo servicio
- Si el template falla en Backstage, los logs del Scaffolder (en la propia UI de Backstage) muestran exactamente en qué paso y por qué

---

## Criterios de evaluación

| Criterio | Puntos |
|---|---|
| El template aparece en la pantalla Create de Backstage | 15 |
| El formulario tiene los parámetros correctos y se muestra bien | 10 |
| `publish:github` crea el repositorio con todos los archivos generados | 25 |
| Las variables del formulario se inyectan correctamente en los archivos skeleton | 20 |
| El nuevo servicio aparece registrado en el catálogo de Backstage | 15 |
| El pipeline del repo generado despliega la Lambda correctamente | 15 |
| **Total** | **100** |

**Puntos extra:**
- +10 El formulario tiene validaciones (nombre sin espacios, longitud máxima)
- +10 El template genera un segundo repositorio para la infra Terraform, separado del repo de código
- +5 El `README.md` generado incluye la URL del endpoint como placeholder que el desarrollador debe completar

---

## Cómo verificar que funciona

- [ ] El template aparece en Backstage → Create con su nombre, descripción y tags
- [ ] Completar el formulario y hacer clic en Create no genera errores en el Scaffolder
- [ ] El repositorio en GitHub existe y contiene los archivos con los valores correctos (no aparecen `${{ values.name }}` sin reemplazar)
- [ ] El nuevo componente aparece en el catálogo de Backstage con el dueño y descripción correctos
- [ ] `terraform apply` en el repo generado crea la Lambda sin errores
- [ ] `curl <function-url>` responde con el JSON de la función

---

## Preguntas de análisis

1. El template ejecuta `fetch:template` → `publish:github` → `catalog:register` en ese orden. ¿Qué pasaría si `catalog:register` falla después de que el repo ya fue creado? ¿Es idempotente el proceso?

2. ¿Por qué el Terraform del skeleton tiene variables para el nombre y el entorno en lugar de valores hardcodeados? ¿Qué problema resuelve esto cuando se instancia el template para una segunda función?

3. Hoy el template crea un solo repositorio mezclando código e infra. ¿Cuál sería el diseño ideal para un entorno productivo real? ¿Qué implicaciones tendría tener dos repositorios generados desde un mismo template?

4. Un desarrollador usó el template, creó su función y seis meses después quiere cambiar la región de AWS. ¿Dónde haría ese cambio? ¿Cómo se aseguraría el equipo de plataforma de que todos los servicios creados con el template usen la misma región?

5. ¿Qué ventaja tiene que el `catalog-info.yaml` sea generado por el template en lugar de que el desarrollador lo escriba manualmente?

---

## Entrega

Comparte:
1. Link al repositorio `cna-templates` con el `template.yaml` y el skeleton
2. Captura de pantalla del template en la pantalla **Create** de Backstage
3. Captura del Scaffolder ejecutando los pasos exitosamente
4. Link al repositorio generado en GitHub
5. El nuevo servicio visible en el catálogo de Backstage
