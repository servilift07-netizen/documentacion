# Documentación ServiLift v2 — Proyecto y Bootcamp de Testing ADSO

Documentación del proyecto formativo **ServiLift** (certificación de ascensores NTC 5926) y de las actividades del
**Bootcamp Testing ADSO** (instructor: repositorio `ergrato-dev/bc-testing-adso`) aplicadas al proyecto.

- Código del proyecto: https://github.com/servilift07-netizen/ServiLift_react_n (rama `version-2.1`)
- Para descargar toda la documentación: botón verde **Code → Download ZIP** de este repositorio.

## Contenido

| # | Documento | Para qué leerlo |
|---|---|---|
| 1 | [Cómo se entrega el trabajo](01-entrega-segun-instructor.md) | Qué revisa el instructor, condiciones del repo, ramas y PR, checklist y modelo de correo |
| 2 | [Semana 1 — Por qué probar y primer E2E](02-semana-01-por-que-probar-y-primer-e2e.md) | Teoría, Playwright, cobertura, lo implementado y preguntas del vocero |
| 3 | [Semana 2 — Pruebas unitarias y CI](03-semana-02-pruebas-unitarias-y-ci.md) | FIRST, valores límite, mutación, GitHub Actions, lo implementado y preguntas del vocero |
| 4 | [Proyecto ServiLift completo](04-proyecto-servilift.md) | Negocio, arquitectura, stack y para qué sirve cada cosa, estructura, instalación y versiones |

## Resumen de lo implementado (semanas 1 y 2)

| Qué | Resultado |
|---|---|
| E2E con Playwright | 6 flujos en verde: ingreso válido, contraseña errada, formulario vacío, búsqueda de clientes (2), cierre de sesión |
| Unitarias de la app (Vitest) | 103 tests; lógica extraída a `src/logica/` |
| Unitarias del backend (pytest) | 70 tests; reglas en `app/services/reglas.py`, reloj inyectable |
| Integración | Flujo completo cotización → certificado, marcado `-m integracion` |
| Cobertura | Línea base S1: app 0% · backend 41%. Umbral S2: app 41% (líneas) · backend 44% |
| Mutación manual | 4 mutantes documentados, todos detectados |
| CI | `.github/workflows/tests.yml`: frontend, backend y evidencia individual en cada PR |
| Plantillas | `docs/matriz-rotacion.md` y `.github/pull_request_template.md` |
| Otro PC | `instalar.ps1` probado en un clon limpio: instalación, 173 unitarias y 6 E2E en verde |

## Cómo estudiar para el vocero aleatorio

1. Leer la sección "Qué tienes que saber explicar" de cada semana.
2. Abrir un test que **no** escribiste y señalar Arrange, Act y Assert.
3. Practicar una mutación en vivo (sección 3.6) y mostrar qué test falla.
4. Abrir la pestaña **Actions** del repositorio y explicar cada job.
5. Repasar las "Preguntas probables del instructor" al final de cada guía.
