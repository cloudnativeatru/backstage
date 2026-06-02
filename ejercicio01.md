# 🚀 Backstage — Instalación Standalone

> **Audiencia:** Desarrolladores y Administradores

---

## 📋 Descripción general

Esta guía explica cómo crear tu propia instancia personalizada de Backstage. Es el primer paso para evaluar, desarrollar o hacer demos de la plataforma.

Al finalizar esta guía tendrás una instalación standalone de Backstage corriendo localmente con una base de datos `SQLite` en memoria y contenido de demo.

> ⚠️ **Importante:** Esta instalación **no es apta para producción** y no contiene información específica de tu organización hasta que configures las integraciones con tus fuentes de datos.

> 💡 **Colaboradores:** Si planeas contribuir con una nueva funcionalidad o corrección de bug al proyecto Backstage, sigue la guía de [Contributors](https://github.com/backstage/backstage/blob/master/CONTRIBUTING.md#get-started) para una instalación basada en repositorio.

---

## 📁 Estructura de carpetas

Al crear la aplicación se genera la siguiente estructura de archivos:

```
app
├── app-config.yaml
├── catalog-info.yaml
├── package.json
└── packages
    ├── app
    └── backend
```

| Archivo / Carpeta       | Descripción |
|-------------------------|-------------|
| `app-config.yaml`       | Archivo de configuración principal de la app. Ver [Configuration](https://backstage.io/docs/conf/). |
| `catalog-info.yaml`     | Descriptores de entidades del Catálogo. Ver [Descriptor Format](https://backstage.io/docs/features/software-catalog/descriptor-format). |
| `package.json`          | Package.json raíz del proyecto. **No agregar dependencias npm aquí directamente.** |
| `packages/`             | Workspaces de Yarn — cada subdirectorio es un paquete independiente. |
| `packages/app/`         | Frontend de Backstage — buen punto de partida para explorar la plataforma. |
| `packages/backend/`     | Backend que potencia funcionalidades como [Auth](https://backstage.io/docs/auth/), [Software Catalog](https://backstage.io/docs/features/software-catalog/), [Templates](https://backstage.io/docs/features/software-templates/) y [TechDocs](https://backstage.io/docs/features/techdocs/). |

---

## ✅ Prerrequisitos

Esta guía asume familiaridad básica con entornos Linux y el uso de terminal (`npm`, `yarn`).

### Recursos mínimos

| Recurso | Mínimo requerido |
|---------|-----------------|
| Espacio en disco | 20 GB |
| Memoria RAM | 6 GB |

### Software requerido

- **Sistema operativo:** Unix/Linux, macOS, o [Windows Subsystem for Linux (WSL)](https://docs.microsoft.com/en-us/windows/wsl/)
- **Node.js** — [Active LTS Release](https://backstage.io/docs/overview/versioning-policy#nodejs-releases) (se recomienda Node 24)
- **Yarn 4.4.1**, **Git**, **Docker**

### ⚙️ Instalación de dependencias (Debian/Ubuntu)

Sigue estos pasos en orden para preparar el entorno:

**1. Herramientas de compilación**

```bash
sudo apt update
sudo apt install make build-essential -y
```

**2. Instalar nvm (gestor de versiones de Node.js)**

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
```

**3. Recargar la terminal**

```bash
source ~/.bashrc
```

**4. Instalar Node.js 24**

```bash
nvm install 24
```

**5. Instalar `isolated-vm` y configurar Yarn**

```bash
npm install isolated-vm

corepack enable

yarn set version 4.4.1
```

**6. Instalar Git**

```bash
sudo apt-get install git -y
```

> 🍎 **macOS:** En lugar del paso 1, ejecuta `xcode-select --install`. Para Docker e instrucciones adicionales consulta [docker.com](https://docs.docker.com/engine/install/).

> **Puertos necesarios:** Si el sistema no es accesible directamente en la red, abrir los puertos `3000` y `7007`.

---

## 🛠️ Instalación y ejecución

### 1. Crear la aplicación

Ejecuta el siguiente comando. El asistente te pedirá el nombre de tu app — se creará como un subdirectorio en tu directorio actual.

```bash
npx @backstage/create-app@latest
```

Si es la primera instalación en el dispositivo, se mostrará:

```console
Need to install the following packages:
@backstage/create-app@<version>
ok to proceed? (y)
```

Ingresa `y` y presiona `Enter`.

Luego ingresa el nombre de tu aplicación:

```console
? Enter a name for the app [required] my-backstage-app

Creating the app...
✔ checking      my-backstage-app
✔ moving        my-backstage-app
✔ executing     yarn install
✔ executing     yarn tsc

Successfully created my-backstage-app
```

### 2. Ingresar al directorio

```bash
cd my-backstage-app
```

### 3. Iniciar la aplicación

```bash
yarn start
```

Este comando inicia el **frontend** y el **backend** como procesos separados (`[0]` y `[1]`) en la misma ventana.

Una vez que veas el mensaje `Rspack compiled successfully`, abre tu navegador en:

```
http://localhost:3000
```

> 💡 Si la ventana del navegador no se abrió automáticamente, navega manualmente a la URL anterior.

---

## 🧭 Próximos pasos

### 👨‍💼 Para Administradores

**Configuración:**
- [Configurar autenticación](https://backstage.io/docs/getting-started/config/authentication)
- [Configurar base de datos](https://backstage.io/docs/getting-started/config/database)
- [Agregar plugins](https://backstage.io/docs/getting-started/configure-app-with-plugins)
- [Personalizar la interfaz](https://backstage.io/docs/conf/user-interface/)
- [Configurar la página principal](https://backstage.io/docs/getting-started/homepage)

**Despliegue a producción:**
- [Desplegar con Docker](https://backstage.io/docs/deployment/docker)
- [Desplegar con Kubernetes](https://backstage.io/docs/deployment/k8s)

---

### 👨‍💻 Para Desarrolladores

**Usando tu instancia de Backstage:**
- [Iniciar sesión en Backstage](https://backstage.io/docs/getting-started/logging-in)
- [Ver el Catálogo](https://backstage.io/docs/getting-started/viewing-catalog)
- [Ver lo que tienes registrado](https://backstage.io/docs/getting-started/view-what-you-own)
- [Ver relaciones entre entidades](https://backstage.io/docs/getting-started/viewing-entity-relationships)
- [Filtrar el Catálogo](https://backstage.io/docs/getting-started/filter-catalog)
- [Registrar un componente](https://backstage.io/docs/getting-started/register-a-component)
- [Crear un nuevo componente](https://backstage.io/docs/getting-started/create-a-component)
- [Actualizar un componente](https://backstage.io/docs/getting-started/update-a-component)
- [Desregistrar y eliminar un componente](https://backstage.io/docs/getting-started/unregister-delete-component)

---

## 💬 Comunidad y soporte

¿Tienes comentarios, sugerencias o encontraste un problema?

- 💬 Únete al [Discord de Backstage](https://discord.gg/backstage-687207715902193673)
- 🐛 Reporta [sugerencias de features](https://github.com/backstage/backstage/issues/new?labels=help+wanted&template=feature_template.md)
- 🔌 Propón [nuevos plugins](https://github.com/backstage/community-plugins/issues/new/choose)

---

## 🔗 Referencias

- [Documentación oficial de Backstage](https://backstage.io/docs/getting-started/)
- [Repositorio en GitHub](https://github.com/backstage/backstage)
- [Versión estable actual: v1.51.0](https://backstage.io/docs/releases/v1.51.0)

---

*Basado en la documentación oficial de [Backstage.io](https://backstage.io) — Copyright © 2026 Backstage Project Authors.*
