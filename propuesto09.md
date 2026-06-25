# Ejercicio Propuesto 09: Pipeline End-to-End para una Función Serverless

**Tiempo estimado:** 75-90 minutos  
**Herramientas permitidas:** Claude, GitHub Copilot, ChatGPT — cualquier IA  
**Dificultad:** Alta  

---

## Prerequisitos

- Cuenta de AWS con permisos para crear recursos Lambda e IAM
- **Terraform CLI** — `terraform -version`
- **AWS CLI** configurado — `aws configure`
- Cuenta de GitHub para crear los dos repositorios y configurar Secrets

---

## Escenario

Un desarrollador del equipo solicita desplegar una función serverless. Como **platform engineer**, tu responsabilidad es:

1. Provisionar la infraestructura necesaria en AWS (el "landing zone")
2. Dejarle al desarrollador un repositorio listo donde solo tenga que editar su código y hacer push

En producción, **infraestructura y código de aplicación no viven en el mismo repositorio**. La infra es propiedad del equipo de plataforma y tiene su propio ciclo de vida, revisión y pipeline. El desarrollador no debería ni saber cómo está construida la infra — solo necesita saber el endpoint que recibe como output.

---

## Dos repositorios, dos responsabilidades

### Repo 1 — `cna-platform-infra` (equipo de plataforma)

Contiene el Terraform que crea la infraestructura. El desarrollador nunca toca este repositorio.

Estructura sugerida:
```
cna-platform-infra/
├── terraform/
│   ├── modules/
│   │   └── lambda-service/     # Módulo reutilizable
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   └── environments/
│       └── dev/                # Instancia del módulo para dev
│           ├── main.tf
│           ├── variables.tf
│           └── backend.tf
├── .github/
│   └── workflows/
│       └── infra-deploy.yml
└── README.md
```

**Qué debe crear el módulo `lambda-service`:**
- IAM Role de ejecución con permisos mínimos para Lambda
- Función Lambda con un nombre predecible basado en variables (ej. `cna-{nombre}-{entorno}`)
- Lambda Function URL con acceso público
- Output: el nombre de la función y la URL del endpoint

**El pipeline de infra** (`infra-deploy.yml`) se dispara manualmente o en push a `main` y ejecuta `terraform apply`. Las credenciales AWS vienen de GitHub Secrets.

---

### Repo 2 — `cna-serverless-function` (equipo de desarrollo)

Contiene únicamente el código de la función y el pipeline que despliega solo el código — **sin tocar la infraestructura**.

Estructura sugerida:
```
cna-serverless-function/
├── src/
│   └── handler.js
├── .github/
│   └── workflows/
│       └── deploy.yml
└── README.md
```

**La función** (`src/handler.js`) debe responder con un JSON que incluya:
- Un mensaje de bienvenida
- El entorno donde corre (variable de entorno)
- Un timestamp de invocación

**El pipeline del desarrollador** (`deploy.yml`) se dispara en push a `main` y hace únicamente esto:
1. Empaqueta el código (`zip`)
2. Actualiza el código de la Lambda existente (`aws lambda update-function-code`)
3. No ejecuta Terraform — la infra ya existe

> **Pregunta clave:** ¿Cómo sabe el pipeline del repo 2 el nombre de la Lambda creada por el repo 1? Investiga las opciones: naming convention acordada, AWS SSM Parameter Store, o variable de entorno en GitHub Secrets. Elige la que te parezca más robusta y justifícala.

---

## Pistas

- El módulo Terraform `lambda-service` debe ser genérico y reutilizable — debería poder instanciarse para crear otra función diferente solo cambiando las variables
- `aws lambda update-function-code` solo actualiza el código, no la configuración de la función. La configuración (memoria, timeout, variables de entorno) la gestiona Terraform
- El pipeline del desarrollador necesita credenciales AWS con permisos **solo** para `lambda:UpdateFunctionCode` — no más. Piensa en el principio de least privilege
- Investiga la diferencia entre `terraform apply` y `aws lambda update-function-code` para entender por qué se usa cada uno en cada repo
- El state de Terraform debe estar en S3, no en el repositorio. El backend debe configurarse en `environments/dev/backend.tf`

---

## Criterios de evaluación

| Criterio | Puntos |
|---|---|
| `cna-platform-infra`: el módulo Terraform crea la infra sin errores | 20 |
| `cna-platform-infra`: el pipeline de infra funciona con `terraform apply` | 15 |
| `cna-serverless-function`: la Function URL responde con el JSON correcto | 20 |
| `cna-serverless-function`: el pipeline actualiza solo el código sin tocar la infra | 20 |
| Separación correcta de responsabilidades entre los dos repos | 15 |
| Credenciales AWS en Secrets con permisos diferenciados por repo | 10 |
| **Total** | **100** |

**Puntos extra:**
- +10 El módulo Terraform es reutilizable y se puede instanciar para un segundo entorno (`staging`) solo cambiando variables
- +10 El pipeline del repo 2 espera confirmación de que el update de Lambda fue exitoso antes de reportar éxito
- +5 El `README.md` del repo 2 describe en menos de 10 líneas todo lo que un nuevo desarrollador necesita saber

---

## Cómo verificar que funciona

- [ ] `terraform apply` en `environments/dev` crea los recursos y muestra la Function URL como output
- [ ] `curl <function-url>` responde con el JSON esperado
- [ ] Un push al repo `cna-serverless-function` dispara el pipeline y actualiza la Lambda **sin ejecutar Terraform**
- [ ] Modificar el mensaje en `handler.js` y hacer push refleja el cambio al invocar la URL
- [ ] `terraform destroy` en el repo de infra limpia todos los recursos sin errores

> **Importante:** Ejecuta `terraform destroy` al terminar para no generar costos en tu cuenta AWS.

---

## Preguntas de análisis

1. El pipeline del desarrollador usa `aws lambda update-function-code` en lugar de `terraform apply`. ¿Qué problema generaría si el desarrollador pudiera ejecutar `terraform apply` en el repo de infra directamente?

2. ¿Cómo decidiste compartir el nombre de la Lambda entre los dos repos? ¿Qué ventajas y riesgos tiene tu enfoque vs. las otras opciones?

3. Los dos pipelines tienen credenciales AWS distintas con permisos distintos. ¿Por qué es importante esto? ¿Qué pasaría si ambos usaran la misma IAM Key con `AdministratorAccess`?

4. Si mañana el equipo de plataforma decide cambiar de Lambda Function URL a API Gateway, ¿qué cambios requeriría el repo de infra y qué cambios (si alguno) requeriría el repo del desarrollador?

5. En un equipo real, ¿quién debería poder aprobar PRs en `cna-platform-infra`? ¿Debería ser el mismo grupo que aprueba PRs en `cna-serverless-function`?

---

## Entrega

Comparte:
1. Links a los dos repositorios de GitHub
2. Output de `terraform apply` con los recursos creados y la Function URL
3. Captura del pipeline del repo de función ejecutándose exitosamente (sin Terraform)
4. Resultado del `curl` a la Function URL antes y después de un cambio en el código
