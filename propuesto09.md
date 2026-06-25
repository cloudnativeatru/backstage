# Ejercicio Propuesto 09: Pipeline End-to-End para una Función Serverless

**Tiempo estimado:** 60-75 minutos  
**Herramientas permitidas:** Claude, GitHub Copilot, ChatGPT — cualquier IA  
**Dificultad:** Alta  

---

## Prerequisitos

Verifica que tienes esto instalado y configurado antes de empezar:

- Cuenta de AWS con permisos para crear recursos Lambda e IAM
- **Terraform CLI** — `terraform -version`
- **AWS CLI** — `aws configure` (necesitas `AWS_ACCESS_KEY_ID` y `AWS_SECRET_ACCESS_KEY`)
- **GitHub CLI** o cuenta de GitHub para crear repositorios y configurar Secrets

---

## Escenario

Un desarrollador del equipo pide un endpoint HTTP serverless que responda con información del portal. Como **platform engineer**, tu trabajo no es escribir la lógica de negocio — es construir el andamiaje completo para que cualquier desarrollador pueda hacer push de su función y que quede desplegada automáticamente.

El desarrollador solo debería tener que editar `src/handler.js` y hacer push a `main`. El resto — infraestructura, empaquetado, deploy — debe ocurrir solo.

---

## Lo que debes crear

### 1. Repositorio en GitHub

Crea un repositorio llamado `cna-serverless-function` con esta estructura:

```
cna-serverless-function/
├── src/
│   └── handler.js              # La función Lambda
├── terraform/
│   ├── provider.tf             # AWS provider
│   ├── main.tf                 # Lambda + Function URL + IAM Role
│   ├── variables.tf            # Variables configurables
│   └── outputs.tf              # URL del endpoint como output
├── .github/
│   └── workflows/
│       └── deploy.yml          # Pipeline CI/CD
├── .gitignore
└── README.md
```

---

### 2. La función (`src/handler.js`)

Una función Lambda Node.js que responda a cualquier petición HTTP con un JSON que contenga:

- Un mensaje de bienvenida
- El nombre del entorno (`dev`, `staging`, `prod`) — que venga de una variable de entorno
- Un timestamp del momento en que fue invocada

---

### 3. La infraestructura (Terraform)

Crea en AWS los recursos mínimos necesarios para que la función sea invocable vía HTTPS:

- Un **IAM Role** de ejecución para Lambda con permisos mínimos
- La **función Lambda** apuntando al código en `src/`
- Una **Lambda Function URL** con acceso público (sin autenticación)

> **Pista:** Para empaquetar el código fuente antes de subirlo, investiga el data source `archive_file` del provider `hashicorp/archive`. Terraform puede generar el ZIP sin necesidad de un paso de build manual.

> **Pista:** El `terraform.tfstate` nunca debe subirse al repositorio. Investiga cómo usar S3 como backend remoto para el estado — o al menos agrégalo al `.gitignore`.

---

### 4. El pipeline (`.github/workflows/deploy.yml`)

El workflow debe dispararse en cada push a `main` y ejecutar:

1. Checkout del código
2. Configurar credenciales AWS desde **GitHub Secrets** (no hardcodeadas)
3. Setup de Terraform
4. `terraform init`
5. `terraform apply -auto-approve`

Al final del workflow, el output de Terraform con la URL del endpoint debe ser visible en los logs.

---

## Pistas

- Lambda Function URL es más simple que API Gateway para este caso — genera un endpoint HTTPS directo sin recursos adicionales
- Las credenciales AWS para el pipeline van como **GitHub Secrets** (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`). El step de GitHub Actions `aws-actions/configure-aws-credentials` sabe cómo inyectarlas
- Terraform necesita una región AWS. Puedes declararla como variable o hardcodearla en el provider para este ejercicio
- Cuando el pipeline hace `terraform apply` por segunda vez (segundo push), Terraform compara el estado actual con el deseado y solo actualiza lo que cambió — eso es idempotencia
- El runtime de Lambda para Node.js en AWS es `nodejs20.x`

---

## Criterios de evaluación

| Criterio | Puntos |
|---|---|
| El repositorio tiene la estructura completa con todos los archivos | 10 |
| `terraform apply` crea los recursos sin errores desde cero | 25 |
| La Function URL responde con el JSON correcto al hacer `curl` | 25 |
| El pipeline de GitHub Actions completa exitosamente en push a `main` | 25 |
| Las credenciales AWS están en Secrets y no aparecen en ningún archivo del repo | 15 |
| **Total** | **100** |

**Puntos extra:**
- +10 El estado de Terraform usa un backend remoto en S3
- +10 El pipeline incluye `terraform plan` antes del apply y muestra el diff en los logs
- +5 El `README.md` documenta en 5 pasos cómo un desarrollador nuevo usaría este repo

---

## Cómo verificar que funciona

- [ ] `terraform apply` completa sin errores y muestra la Function URL como output
- [ ] `curl <function-url>` responde con el JSON esperado
- [ ] Un push a `main` dispara el workflow en GitHub Actions y completa sin errores
- [ ] Un segundo push (modificando el mensaje de la función) redespliega automáticamente
- [ ] `terraform destroy` limpia todos los recursos sin errores

> **Importante:** Ejecuta `terraform destroy` al terminar el ejercicio para no generar costos en tu cuenta de AWS.

---

## Preguntas de análisis

1. ¿Qué pasa si dos platform engineers ejecutan `terraform apply` al mismo tiempo desde sus máquinas locales? ¿Cómo lo resuelve un backend remoto con state locking?

2. El IAM Role de Lambda tiene permisos mínimos. ¿Qué permisos exactos necesita una función Lambda básica para ejecutarse? ¿Qué pasaría si le dieras `AdministratorAccess`?

3. En un equipo real, ¿el pipeline debería hacer `terraform apply` directamente en cada push a `main`, o habría algún mecanismo de aprobación humana antes? ¿Para qué entorno sí y para cuál no?

4. Lambda Function URL vs. API Gateway: ¿qué te da API Gateway que Function URL no tiene? ¿Cuándo valdría la pena la complejidad adicional?

5. Hoy el pipeline hace deploy cada vez que alguien hace push a `main`. ¿Cómo modificarías el workflow para que solo haga deploy si los archivos en `src/` o `terraform/` cambiaron?

---

## Entrega

Comparte:
1. Link al repositorio de GitHub (puede ser público o privado con acceso al instructor)
2. Output de `terraform apply` mostrando los recursos creados y la Function URL
3. Captura del workflow de GitHub Actions ejecutándose exitosamente
4. El resultado del `curl` a la Function URL con la respuesta JSON
