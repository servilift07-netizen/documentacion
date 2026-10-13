# 3. Semana 2 — Pruebas unitarias con AAA y umbral de cobertura en CI

> Primera semana con **rotación de capas** (Front, API, BD, E2E). Piso de cobertura al cerrar: **30%**.
> Evaluación: Conocimiento 30% (vocero) · Desempeño 40% (≥ 3 tests unitarios propios + mutante) · Producto 30%
> (CI en verde, merge bloqueado en rojo, umbral ≥ 30%).

---

## 3.1 Qué tienes que saber explicar (criterios de Conocimiento)

1. Por qué una unidad que depende de la **fecha actual** rompe FIRST y cómo se resuelve.
2. Las **particiones** y **valores límite** de una regla del proyecto que **no** escribiste.
3. Por qué una **cobertura alta no garantiza tests buenos** (el mutante `>=`).
4. Qué hace el **workflow de CI** y qué pasa si la cobertura baja del umbral.

En la revisión el vocero puede tener que hacer **una mutación en vivo** sobre una capa que no trabajó.

---

## 3.2 Qué es una unidad

Una **unidad** es la pieza más pequeña de lógica que tiene sentido probar sola: una función, un método o una clase.
La prueba unitaria la ejecuta **sin servidor HTTP, sin base de datos, sin navegador y sin teléfono**.

| Capa | Unidades con valor en ServiLift |
|---|---|
| App | Validación del ingreso, recorrido estimado, plazo de corrección, formatos de fecha y moneda, estados, destino de una notificación, checklist con cambios pendientes del teléfono |
| Backend | Resumen de hallazgos, concepto de visita, plazo de corrección, máquina de estados, reglas de programación, parámetros, contraseñas, enlaces firmados, hora de Colombia, configuración |

No todo merece prueba unitaria: un endpoint que solo recibe la petición y llama al servicio se prueba como API
(semana 4); una consulta SQL se prueba contra la base real (semana 6).

### Separar la lógica para poder probarla

Si la regla vive dentro del endpoint o del componente, no se puede probar sola. **Hay que extraerla a una función.**
Eso hicimos esta semana:

| Antes | Después (unidad) |
|---|---|
| Validación dentro de `ingresar()` en `app/login.tsx` | `validarIngreso()` en `src/logica/login.ts` |
| `recorridoEstimado` dentro de `DatosTab.tsx` | `src/logica/recorrido.ts` |
| Colores y textos del plazo dentro del componente `Plazo.tsx` | `nivelPlazo()` y `textoPlazo()` en `src/logica/plazo.ts` |
| Formatos y estados dentro de `tema.ts` (que importa React Native Paper) | `src/logica/formatos.ts` y `src/logica/estados.ts` |
| A dónde lleva una notificación, mezclado con `router.push` | `destinoNotificacion()` en `src/logica/notificaciones.ts` |
| `aplicarPendientes` / `resumenLocal` junto a un hook de React | `src/logica/checklistLocal.ts` |
| `_validar_horario` dentro de `routes/programacion.py` | `validar_horario()` en `app/services/reglas.py` |
| Validación del parámetro numérico dentro del endpoint | `numero_parametro()` en `app/services/reglas.py` |
| Transición permitida, escondida en `cambiar_estado` | `puede_pasar()` en `app/services/flujo.py` |

Los archivos originales importan las funciones nuevas: la app hace exactamente lo mismo (lo confirmamos con los 6
E2E en verde después del cambio).

---

## 3.3 Los principios FIRST

| Principio | Significa | Cómo se rompe | Cómo lo cumplimos |
|---|---|---|---|
| **F**ast | Milisegundos | Conectarse a la BD o a la red | 70 tests del backend en ~6 s, 103 de la app en ~1 s |
| **I**solated | No depende de otros tests ni del orden | Un test deja datos que otro necesita | Cada test arma sus propios datos en memoria |
| **R**epeatable | Mismo resultado siempre, en cualquier PC | Usar la fecha real, azar o la zona horaria | El reloj se recibe por parámetro (ver 3.4) |
| **S**elf-validating | Pasa o falla solo | Imprimir con `print`/`console.log` y mirar | Solo aserciones |
| **T**imely | Se escribe junto con el código | Dejarlo "para el final" | Cada regla nueva llega con su test |

### 3.4 El reloj: el enemigo silencioso de *Repeatable*

La regla "no se programa una visita en un día pasado" depende de **hoy**. Si el test usa la fecha real, hoy pasa y
mañana falla. Solución: **recibir "hoy" por parámetro**, con la fecha real como valor por defecto en producción.

```python
# app/services/reglas.py
def validar_horario(fecha_programada, hora_inicio, hora_fin, hoy: date | None = None):
    hoy = hoy or hoy_colombia()          # producción: la fecha real de Colombia
    if fecha_programada and fecha_programada < hoy:
        raise HTTPException(400, "No se puede programar una visita en un día pasado")
    ...

# tests/unit/test_reglas.py — el test fija el día: siempre da lo mismo
HOY = date(2026, 10, 12)
validar_horario(HOY, time(8, 0), time(10, 0), hoy=HOY)
```

Lo mismo hicimos con:

- `url_archivo(ruta, ahora_s=None)` y `validar_url_archivo(..., ahora_s=None)` (enlaces de fotos que vencen).
- `esHoy(v, hoy = new Date())` en la app.
- El enlace de PDF vencido se prueba creándolo **ya vencido** (`minutos=-1`), sin esperar.

---

## 3.5 Diseño de casos

### El bug que la cobertura no ve

Si una regla tiene un `<` y alguien lo cambia a `<=`, **la cobertura no cambia**: la línea se sigue ejecutando.
Solo un test en el **borde** lo detecta. Los casos no se eligen al azar: se usan particiones y valores límite.

### Particiones de equivalencia

Grupos de valores que el código trata igual; basta **un representante por grupo**.

Regla "programar visita" (hoy = 12/10/2026):

| Partición | Representante | Esperado |
|---|---|---|
| Días pasados | 11/10/2026 | 400 "día pasado" |
| Hoy y futuro | 12/10 y 13/10 | Acepta |
| Sin fecha (reprogramar sin cambiarla) | `None` | Acepta |

### Valores límite

Los defectos viven en los **bordes**. Se prueba justo en el borde y a cada lado.

| Regla de ServiLift | Valores límite que probamos | Archivo |
|---|---|---|
| Visita no en día pasado | ayer ❌ · **hoy ✅** · mañana ✅ | `test_reglas.py` |
| Fin después del inicio | 08:00–08:00 ❌ · 08:00–07:59 ❌ · **08:00–08:01 ✅** | `test_reglas.py` |
| Parámetro ≥ 0 | **0 ✅** · −0,1 ❌ | `test_reglas.py`, `recorrido.test.ts` |
| Paradas > 0 para estimar el recorrido | **0 → vacío** · 1 → 4,7 m · 2 → 6,9 m | `recorrido.test.ts` |
| Urgencia del plazo | −1 vencido · **0** urgente · **3** urgente · 4 próximo · **10** próximo · 11 normal | `plazo.test.ts` |
| Concepto 1ª visita | 0 hallazgos conforme · **1** no conforme | `test_checklist.py` |
| Concepto 2ª visita | 3 de 3 corregidos conforme · **2 de 3** no conforme | `test_checklist.py` |
| Plazo propio del ítem | **0** días (no confundir con "sin plazo") | `test_checklist.py` |
| Enlace firmado | válido en el segundo del vencimiento · inválido **un segundo después** | `test_security.py` |
| Separador de miles | 999 → `$ 999` · **1000 → `$ 1.000`** | `formatos.test.ts` |
| Paleta de gráficas (8 colores) | índice **8** vuelve al 0 | `tema.test.ts` |

### Casos que casi siempre se olvidan

Vacío y solo espacios (`""`, `"   "`, `"\t"`), ausente (`null`, `None`, `undefined`), tipo incorrecto (`"doce"`,
`"abc"`), colecciones vacías (inspección sin resultados). Están en `login.test.ts`, `recorrido.test.ts`,
`test_reglas.py` y `test_checklist.py`.

### Tests parametrizados

Varios casos con la misma estructura en un solo test; cada fila se reporta como un test aparte.

```typescript
// Vitest (app): it.each
it.each([
  [-1, 'vencido'], [0, 'urgente'], [3, 'urgente'], [4, 'proximo'], [10, 'proximo'], [11, 'normal'],
])('should classify %d remaining days as %s', (dias, nivel) => {
  expect(nivelPlazo(dias)).toBe(nivel);
});
```

```python
# pytest (backend): @pytest.mark.parametrize
@pytest.mark.parametrize(("total_no_cumple", "corregidos", "visita", "concepto"), [
    (0, 0, 1, "CONFORME"), (1, 0, 1, "NO_CONFORME"), (3, 3, 2, "CONFORME"), (3, 2, 2, "NO_CONFORME"),
])
def test_concepto_de_visita(total_no_cumple, corregidos, visita, concepto):
    assert concepto_de_visita({"total_no_cumple": total_no_cumple, "corregidos": corregidos}, visita) == concepto
```

Se parametrizan casos del **mismo** comportamiento; si un caso necesita otro Arrange o Assert, va en un test aparte.

---

## 3.6 Mutación manual: el detector de tests débiles

1. Cambiar un operador (`>` por `>=`, `min` por `max`) o borrar una validación.
2. Correr los tests.
3. Si **ninguno** falla, el mutante **sobrevivió**: falta un caso.
4. Escribir el caso que lo mata y deshacer la mutación.

Mutantes que probamos (todos detectados):

| Mutante | Test que lo detecta |
|---|---|
| `dias <= 3` → `dias < 3` en `nivelPlazo` (app) | `should classify 3 remaining days as urgente` |
| `n <= 0` → `n < 0` en `recorridoEstimado` (app) | `should return "0" -> "" at the boundary of a positive number of stops` |
| `fecha_programada < hoy` → `<=` en `validar_horario` (backend) | `test_validar_horario_accepts_today_and_future_days[fecha0]` |
| `min(plazos)` → `max(plazos)` en `calcular_plazo` (backend) | `test_calcular_plazo_uses_the_strictest_deadline_among_the_findings` (y 2 más) |

Cómo hacerlo en vivo (ejemplo backend):

```
cd servilift-backend
# 1. editar app/services/reglas.py: cambiar  < hoy  por  <= hoy
venv\Scripts\python -m pytest tests/unit/test_reglas.py --no-cov -q
# 2. se ve: 2 failed  → el mutante murió
# 3. deshacer el cambio y volver a correr: todo en verde
```

`--no-cov` sirve para correr solo una parte de la suite sin que falle el umbral.

### Por qué cobertura alta ≠ tests buenos

La cobertura dice que la línea **se ejecutó**; no dice que su lógica **se verificó en el borde**. Nuestra lección
real: el backend tenía **41% de cobertura con cero tests** (solo por importar los módulos). Y un test que llama a
`validar_horario` con una fecha de 1990 y otra de 2999 cubre la línea, pero deja vivo el mutante `<=`.

---

## 3.7 Las pruebas que escribimos

### App — Vitest (`servilift-app/tests/`, 103 tests)

| Archivo | Unidad | Qué verifica |
|---|---|---|
| `login.test.ts` | `validarIngreso` | Correo vacío, solo espacios o tabulador → mensaje; la contraseña no se recorta |
| `recorrido.test.ts` | `recorridoEstimado`, `numeroParametro` | 12 × 2,2 + 2,5 = 28,9 m; borde de 0 paradas; coma decimal; respaldo si el parámetro es inválido |
| `plazo.test.ts` | `nivelPlazo`, `textoPlazo` | Bordes 0/3/10 de urgencia; plazo inmediato; vencido hace N días |
| `formatos.test.ts` | `fecha`, `fechaHora`, `hora`, `moneda`, `fechaLarga`, `esHoy`… | Fechas del backend sin depender de la zona del celular; miles con punto; `esHoy` con reloj fijo |
| `estados.test.ts` | `etiqueta`, `grupoEstado` | Texto y color de cada estado |
| `notificaciones.test.ts` | `destinoNotificacion` | El director abre la revisión; el cliente abre el PDF; enlaces inválidos no navegan |
| `checklistLocal.test.ts` | `aplicarPendientes`, `resumenLocal` | Lo guardado en el teléfono se muestra encima de lo del servidor y queda marcado como pendiente |
| `tema.test.ts` | `colorEstado`, `colorNombre`, `aclarar` | Paleta; usa `vi.mock` para reemplazar React Native Paper, que no corre en Node |

### Backend — pytest (`servilift-backend/tests/unit/`, 70 tests)

| Archivo | Qué verifica |
|---|---|
| `test_checklist.py` | Resumen de hallazgos; concepto de 1ª y 2ª visita; plazo más exigente; plazo propio del ítem; pendientes de la 2ª visita |
| `test_flujo_estados.py` | Transiciones permitidas y prohibidas (no existe 3ª visita); historial; 409 en transición inválida; cotización finalizada |
| `test_reglas.py` | Programar visita (hoy/ayer, horas) y parámetros numéricos |
| `test_security.py` | bcrypt, JWT con rol y cliente, enlace de PDF vencido o con otro uso, enlaces firmados de fotos |
| `test_tiempo_config.py` | Conversión a hora de Colombia; validación de `ZONA_HORARIA`; URL de la base; orígenes CORS |

Sin base de datos: las inspecciones se arman con `SimpleNamespace` y la sesión de SQLAlchemy se reemplaza por una
clase `SesionFalsa` que solo guarda lo que se agrega (adelanto de la semana 5, dobles de prueba).

La prueba de flujo completo existente (`tests/test_flujo.py`) quedó marcada `@pytest.mark.integracion`: necesita
MySQL, así que **no corre por defecto** (`-m "not integracion"` en `pyproject.toml`). Se corre con
`pytest -m integracion --no-cov` contra `servilift_v2_prueba`.

---

## 3.8 Cobertura y exclusiones

| Parte | Cobertura real S2 | Umbral fijado | Dónde |
|---|---|---|---|
| App | líneas 41,94% · ramas 61,11% · funciones 32,89% · sentencias 43,41% | 41 · 61 · 32 · 43 | `vitest.config.mts` → `thresholds` |
| Backend | 44,49% | 44 | `pyproject.toml` → `fail_under` |

Regla de trinquete: el umbral es el **mayor entre 30 y la cobertura real**, redondeado hacia abajo.

Exclusiones de la app (cada una con su comentario en `vitest.config.mts`):

| Excluido | Por qué |
|---|---|
| `src/pantallas/**/*.tsx`, `src/componentes/**` | Pantallas y componentes visuales: se prueban como componentes desde la **semana 3** |
| `src/**/*Context.tsx`, `src/hooks/**` | Estado de React: se prueba junto con los componentes (semana 3) |
| `src/api/**` | Adaptador HTTP (`fetch`), sin reglas propias |
| `src/offline/almacenamiento.ts`, `fotosLocales*.ts`, `src/sesion/almacen*.ts` | Adaptadores del almacenamiento del teléfono |
| `src/config.ts`, `src/navegacion.ts` | Configuración y arranque |

**No se excluyó** `src/offline/cola.ts` (la cola sin conexión): es lógica de negocio. Hoy está en 0% y se probará
con dobles en la semana 5. Eso es honestidad: el instructor revisa que no se infle el número.

Backend: se mide todo `app/` sin exclusiones.

---

## 3.9 Integración continua con GitHub Actions

**CI** = un servidor corre los tests **solo**, en cada push y en cada pull request. Si fallan o la cobertura baja,
el PR queda en rojo y no se fusiona.

| Concepto | Qué es |
|---|---|
| **Workflow** | Archivo YAML en `.github/workflows/` (el nuestro: `tests.yml`) |
| **Evento** (`on`) | Qué lo dispara: `push` a `main` o `version-2.1`, y todo `pull_request` |
| **Job** | Pasos en una máquina virtual limpia (`ubuntu-latest`). Los jobs corren **en paralelo** |
| **Step** | Una acción (`uses`) o un comando (`run`) |
| **Acción** | Paso reutilizable: `actions/checkout`, `actions/setup-node`… con **versión exacta** (`@v7.0.1`) |

### Nuestro workflow, job por job

| Job | Pasos | Qué demuestra |
|---|---|---|
| `frontend` | checkout → Node 22 (con caché de npm) → `npm ci` → **`npm test`** → sube `coverage/` como artefacto | Tests de la app + umbral |
| `backend` | checkout → Python 3.13 (con caché de pip) → `pip install -r requirements-dev.txt` → **`pytest`** | Tests del backend + umbral |
| `evidencia-individual` | checkout con historial completo → `git shortlog` de commits de test desde el lunes | Quién hizo commits de test (lo mira el instructor en el resumen del run) |

Clave: el CI corre **el mismo comando que en local**. Como el umbral vive en `vitest.config.mts` y `pyproject.toml`,
el YAML no sabe nada de cobertura: **si el comando falla, el job falla**.

¿Qué pasa si la cobertura baja del umbral? El comando termina con error
(`FAIL Required test coverage of 44% not reached` o `ERROR: Coverage for lines (40%) does not meet global threshold (41%)`),
el job queda ❌, el PR muestra el check en rojo y, con la regla de rama, el botón de merge se bloquea.

### Leer rojo o verde

- En el PR, abajo, la lista de **checks** con ✅/❌; **Details** abre el log (el mismo error de la terminal).
- Pestaña **Actions**: historial de todas las ejecuciones; arriba de cada run, el resumen con la evidencia individual.

### Que el rojo bloquee

**Settings → Rules → Rulesets → New branch ruleset** sobre `main` y `version-2.1`: *Require a pull request* y
*Require status checks to pass* con `frontend` y `backend`. Como el repo es **público**, aplica en cuenta gratuita.

### Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Pasa en local, falla en CI | Depende de `.env`, de la hora local o de datos de tu BD | Revisar FIRST: la unitaria no depende de nada de eso |
| Error de conexión al importar la app | El engine se conecta al importar | En ServiLift no pasa: `create_engine` no conecta hasta la primera consulta |
| Paquete distinto en CI | Versiones transitivas diferentes | Versionar el lockfile (`package-lock.json`, `pnpm-lock.yaml`) — lo hacemos |
| No encuentra `package.json` | `working-directory` mal | En nuestro YAML: `servilift-app` y `servilift-backend` |

---

## 3.10 Preguntas probables del instructor (con respuesta)

**¿Por qué una unidad que usa la fecha actual rompe FIRST? ¿Cómo lo resolvieron?**
Rompe *Repeatable*: el resultado cambia según el día en que se corra. Se resolvió recibiendo `hoy` por parámetro
en `validar_horario` (y `ahora_s` en los enlaces firmados, `hoy` en `esHoy`); producción usa el valor por defecto.

**Dame las particiones y valores límite de `nivelPlazo`.**
Particiones: vencido (< 0), urgente (0–3), próximo (4–10), normal (> 10). Límites: −1/0, 3/4, 10/11.

**¿Por qué 44% de cobertura no garantiza buenos tests?**
Porque cobertura es ejecución, no verificación. Ejemplo: el mutante `<` → `<=` en `validar_horario` no cambia la
cobertura; solo lo mata el test con `fecha = hoy`. Y el backend tenía 41% sin un solo test.

**Haz una mutación en vivo.** Cambiar `min(plazos)` por `max(plazos)` en `app/services/checklist.py`, correr
`pytest tests/unit/test_checklist.py --no-cov -q` → 3 fallan. Deshacer → verde.

**¿Qué hace el workflow de CI?** Tres jobs en paralelo: `frontend` (`npm test`), `backend` (`pytest`) y
`evidencia-individual`. Corre en cada PR y en cada push a `main`/`version-2.1`. Si la cobertura baja del umbral, el
comando falla, el job queda en rojo y el PR no se puede fusionar.

**¿Por qué excluyeron las pantallas de la cobertura?** Son UI: se prueban como componentes en la semana 3. La
lógica que tenían adentro se **extrajo** a `src/logica/` para medirla y probarla ahora. La cola sin conexión,
que es lógica de negocio, no se excluyó.

**¿Qué es `SesionFalsa` / `vi.mock`?** Dobles de prueba mínimos: reemplazan la sesión de base de datos y la
librería visual para que el test sea rápido y aislado (FIRST). El tema completo es la semana 5.

**¿Qué es la regla de trinquete?** El umbral solo sube: cada semana es el mayor entre el piso y la cobertura real
anterior, redondeada hacia abajo.

---

## 3.11 Glosario

- **CI**: integración continua; verificar automáticamente cada push/PR.
- **Check**: resultado de un job en el PR (✅/❌).
- **Determinista**: mismo resultado con las mismas entradas.
- **FIRST**: Fast, Isolated, Repeatable, Self-validating, Timely.
- **Función pura**: su resultado depende solo de sus parámetros y no modifica nada externo.
- **Job / step / acción / workflow**: piezas de GitHub Actions.
- **Mutante / prueba de mutación**: defecto introducido a propósito para ver si los tests lo detectan.
- **Partición de equivalencia**: grupo de entradas que el código trata igual.
- **Regla de rama (ruleset)**: impide fusionar sin cumplir condiciones.
- **Test parametrizado**: `it.each` (Vitest), `@pytest.mark.parametrize` (pytest).
- **Trinquete**: el umbral nunca baja.
- **Unidad bajo prueba**: la pieza que el test ejecuta aislada.
- **Valor límite**: el borde entre dos particiones, donde viven los `>` vs `>=`.
