# Propuesto 05 — Plugin: Generador de Diagramas de Arquitectura con IA

## Contexto

Tu empresa está adoptando Backstage como portal interno del desarrollador. Los equipos ya pueden registrar sus servicios en el catálogo, pero cuando alguien quiere entender cómo están conectados los componentes de un sistema nuevo, tiene que dibujarlo manualmente en herramientas externas como Draw.io o Lucidchart.

La idea es simple: ¿y si desde Backstage pudieras describir con palabras una arquitectura y obtener automáticamente un diagrama?

---

## Objetivo

Crear un plugin de Backstage que permita al usuario **describir una arquitectura de software en lenguaje natural** y generar automáticamente un **diagrama visual** usando inteligencia artificial local (Ollama) y el estándar de diagramas **Mermaid**.

---

## ¿Qué debe hacer el plugin?

1. Mostrar una página accesible desde el sidebar de Backstage con el nombre **"AI Diagrams"**
2. Tener un campo de texto donde el usuario escriba la descripción de su arquitectura
3. Al enviar, consultar a Ollama para que genere el código Mermaid correspondiente
4. Renderizar el diagrama de forma visual en la misma pantalla
5. Permitir al usuario copiar el código Mermaid generado

---

## Ejemplos de uso

El usuario escribe:
```
Un frontend React que se comunica con una API Gateway. 
La API Gateway enruta a dos microservicios: uno de usuarios y uno de pagos. 
Ambos microservicios guardan datos en PostgreSQL. 
El servicio de pagos también publica eventos en Kafka.
```

Y el plugin genera y muestra visualmente algo así:
```
graph TD
    A[Frontend React] --> B[API Gateway]
    B --> C[Servicio Usuarios]
    B --> D[Servicio Pagos]
    C --> E[(PostgreSQL)]
    D --> F[(PostgreSQL)]
    D --> G[Kafka]
```

---

## Restricciones técnicas

- Debes crear el plugin dentro del proyecto Backstage existente (no un repositorio separado)
- Debes usar **Ollama** como motor de IA (el mismo que configuraste en el ejercicio anterior)
- El diagrama debe renderizarse visualmente dentro de Backstage usando la librería **Mermaid**
- El plugin debe aparecer en el sidebar de navegación
- No se permite usar servicios externos de IA (OpenAI, Gemini, etc.)

---

## Criterios de evaluación

| Criterio | Puntos |
|---|---|
| El plugin aparece en el sidebar y la ruta funciona | 20 |
| El campo de descripción acepta texto y envía a Ollama | 20 |
| El diagrama Mermaid se renderiza visualmente | 30 |
| El código Mermaid generado se puede copiar | 10 |
| Manejo de errores (Ollama no disponible, respuesta inválida) | 10 |
| Calidad del prompt enviado a Ollama | 10 |

**Total: 100 puntos**

---

## Pistas

- Revisa cómo creaste el componente `ChatBotPage.tsx` en el ejercicio anterior — la estructura es muy similar
- Para renderizar Mermaid en React necesitarás instalar una dependencia: investiga `mermaid` en npm
- El éxito del diagrama depende mucho del **prompt** que le envíes a Ollama — sé específico en pedirle que responda SOLO con código Mermaid válido
- Mermaid soporta varios tipos de diagramas: `graph`, `flowchart`, `sequenceDiagram`, `classDiagram`, `erDiagram` — el modelo de IA debería elegir el más apropiado según la descripción

---

## Entregable

Un pull request o zip con los archivos modificados/creados, más una captura de pantalla mostrando un diagrama generado con al menos una descripción de arquitectura real.
