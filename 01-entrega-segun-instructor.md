# 1. Cómo se entrega el trabajo (según el bootcamp del instructor)

Fuente: repositorio del instructor `ergrato-dev/bc-testing-adso` (README, `docs/guia-instructor.md`,
`docs/plan-estudios.md`, `plantillas/` y la rúbrica de cada semana).

El instructor **no recibe archivos sueltos**: revisa el **repositorio del grupo en GitHub**. El enlace se le envía
por correo, pero lo que se califica es lo que ve en el repo: los archivos, el historial de commits, los pull
requests y la pestaña **Actions**.

---

## 1.1 Qué revisa el instructor cada semana (y cómo)

| Qué mira | Dónde lo mira | Qué debe encontrar |
|---|---|---|
| CI en verde | Pestaña **Actions** → último run de `main` | ✅ en los jobs `frontend` y `backend` |
| Umbral de cobertura | `servilift-app/vitest.config.mts` y `servilift-backend/pyproject.toml` | Un número ≥ al piso de la semana (S1: línea base, S2: 30%, S3: 40%… S8: 80%) |
| Evidencia individual | Resumen del job `evidencia-individual` en Actions (o `git shortlog`) | **Cada integrante** con al menos 1 commit de test en la semana, **con su propio usuario de Git** |
| Matriz de rotación | `docs/matriz-rotacion.md` | Capa de cada integrante por semana, cobertura semanal, vocero |
| Plantilla de PR | `.github/pull_request_template.md` | Que exista: GitHub la usa al abrir cada PR |
| Pull requests | Pestaña **Pull requests** (abre uno al azar) | Autor y revisor de **capas distintas**, checklist marcado, mutante documentado (S2) |
| Conocimiento | Vocero aleatorio: escribe su respuesta en la matriz y el instructor hace **una** pregunta | Que cualquiera explique el test de otro y haga una mutación en vivo |

> ⚠️ El instructor revisa **`main`**. Por eso, al cierre de cada semana, lo aprobado en la rama de versión
> debe llegar a `main` por pull request (ver 1.4), o hay que decirle en el correo qué rama revisar.

## 1.2 Reglas que el bootcamp exige en el repositorio

1. **Todo cambio de tests entra por pull request.** Nada de subir directo.
2. **Revisión cruzada**: quien revisa el PR trabajó en **otra capa** esa semana y **corre el test en su equipo**.
3. **Mínimo un commit de test por aprendiz por semana**, con su usuario de Git (`git config user.name` y
   `user.email` propios). Un commit hecho por otro se descubre en el vocero.
4. **Ningún PR en rojo se fusiona.** Con ruleset en GitHub (repo público) o como regla del grupo.
5. **El umbral de cobertura nunca baja.** Sube con un PR que solo cambia el número (regla de trinquete:
   el mayor entre el piso de la semana y la cobertura real, redondeada hacia abajo).
6. **Exclusiones justificadas**: todo archivo fuera de la cobertura lleva un comentario del porqué y nunca es
   lógica de negocio.
7. **Tests que pueden fallar**: cada test se rompe a propósito una vez para comprobar que detecta el error.

## 1.3 Estado del repositorio ServiLift para las semanas 1 y 2

| Requisito | Semana | Estado | Archivo |
|---|---|---|---|
| Carpeta `e2e/` con Playwright en versión exacta | 1 | ✅ | `e2e/package.json` (`@playwright/test` 1.64.0) |
| `playwright.config.js` que levanta el frontend | 1 | ✅ | levanta backend (8010) y app (8091) |
| Un E2E por integrante, flujos distintos, AAA y aserción | 1 | ✅ 6 tests | `e2e/tests/*.spec.js` (repartirlos entre integrantes) |
| Matriz de rotación | 1–2 | ✅ (falta poner los usuarios reales) | `docs/matriz-rotacion.md` |
| Plantilla de PR | 1 | ✅ | `.github/pull_request_template.md` |
| Cobertura configurada en front y back + línea base | 1 | ✅ Front 0% · Back 41% | `vitest.config.mts`, `pyproject.toml` |
| ≥ 3 tests unitarios por integrante con valor límite | 2 | ✅ 103 (app) + 70 (backend) | `servilift-app/tests/`, `servilift-backend/tests/unit/` |
| Mutante documentado en el PR | 2 | ✅ 4 mutantes | matriz y descripción del PR |
| Workflow de CI con umbral | 2 | ✅ | `.github/workflows/tests.yml` |
| Umbral ≥ 30% en front y back | 2 | ✅ Front 41% líneas · Back 44% | mismos archivos de configuración |
| Job `evidencia-individual` | 2 | ✅ | en el mismo workflow |
| Regla de rama | 2 | ⏳ la activa el dueño del repo | ver 1.5 |
| PR revisado por otro integrante | 1–2 | ⏳ abrir y pedir revisión | ver 1.4 |

## 1.4 Ramas, versiones y pull requests

```
main ──────────────●  v2.0 (estable, intacta)
                    \
version-2.1          ●────────────────●───────────────●  ← aquí se integran los cambios
                      \              ↑ PR 1           ↑ PR 2
test/semana-01         ●──●──●──●──●─┘                │
                                    \                 │
test/semana-02                       ●──●──●──●──●──●─┘
```

- **`main`** queda como la versión 2.0 estable, con la etiqueta **`v2.0`**. No se trabaja sobre ella.
- **`version-2.1`** es la rama de la versión en curso. El CI también corre en cada push a ella.
- **`test/semana-01`** y **`test/semana-02`** traen el trabajo de cada semana y entran a `version-2.1` por **pull
  request**, en ese orden (la de la semana 2 parte de la 1: al fusionar la primera, la segunda solo muestra lo suyo).
- **Al entregar**: un PR `version-2.1 → main` cuando todo esté en verde, y una etiqueta `v2.1`. Así el instructor ve
  `main` en verde, y `v2.0` sigue disponible para volver atrás.

Pasos para cada PR (los hace cada integrante desde GitHub):

1. GitHub → **Pull requests** → **New pull request** → *base*: `version-2.1`, *compare*: `test/semana-01`.
2. La descripción sale de la plantilla: llenar capa del autor, capa del revisor, checklist y el comando para correrlo.
   En el de la semana 2 anotar el **mutante y el test que lo detecta**.
3. **Reviewers**: un integrante de otra capa. Debe correr los tests en su PC y aprobar (*Approve*).
4. Esperar los checks ✅ y **Merge**. Nunca fusionar en rojo.

## 1.5 Activar la regla de rama (dueño del repositorio)

Settings → Rules → Rulesets → **New branch ruleset**:

1. Nombre `ramas protegidas`, *Enforcement status*: **Active**.
2. *Target branches*: `main` y `version-2.1`.
3. ✅ **Require a pull request before merging** (opcional: 1 aprobación).
4. ✅ **Require status checks to pass** → agregar `frontend` y `backend`.
5. Guardar. Desde ese momento el botón de merge queda bloqueado si un check está en rojo.

El repositorio es público, así que la regla sí se aplica en una cuenta gratuita.

## 1.6 Checklist antes de enviar el correo

- [ ] `docs/matriz-rotacion.md` con el usuario de GitHub real de cada integrante, sus capas S1 y S2 y la cobertura.
- [ ] Cada integrante tiene al menos un commit de test por semana con **su** usuario.
- [ ] PR de la semana 1 y de la semana 2 abiertos, revisados por alguien de otra capa y fusionados.
- [ ] Pestaña Actions: último run en verde (en `version-2.1` y, al entregar, en `main`).
- [ ] Umbral en la configuración: front y back ≥ 30%.
- [ ] Regla de rama activa.
- [ ] El vocero de cada semana anotado en la matriz con su respuesta escrita.
- [ ] Todos ensayaron explicar un test de otra capa y hacer una mutación en vivo (ver las guías de cada semana).

## 1.7 Modelo de correo

> **Asunto**: Bootcamp Testing ADSO – Ficha ____ – Grupo ServiLift – Semanas 1 y 2
>
> Instructor, reciba un cordial saludo. Compartimos el repositorio del proyecto formativo ServiLift con las
> actividades de las semanas 1 y 2:
>
> - Repositorio: https://github.com/servilift07-netizen/ServiLift_react_n
> - Rama revisada: `main` (la versión anterior queda en la etiqueta `v2.0`)
> - CI: pestaña Actions (workflow `tests`) · Matriz: `docs/matriz-rotacion.md`
> - Pull requests: semana 1 y semana 2 (revisión cruzada)
> - Documentación: https://github.com/servilift07-netizen/documentacion
>
> Integrantes: ____ (usuario GitHub) · ____ · ____
>
> Quedamos atentos a sus observaciones.
