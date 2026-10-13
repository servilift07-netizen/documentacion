# 2. Semana 1 — Por qué probar y primer E2E con Playwright

> Capa de la semana: **E2E para todo el grupo** (cada integrante, un flujo distinto).
> Evaluación: Conocimiento 30% (individual, vocero) · Desempeño 40% (tu test E2E) · Producto 30% (grupal).

Esta guía explica **todo lo que el instructor puede preguntar** de la semana 1 y **cómo quedó aplicado en ServiLift**.

---

## 2.1 Qué tienes que saber explicar (criterios de Conocimiento)

1. La diferencia entre **error, defecto y falla**, con un ejemplo de ServiLift.
2. Dónde van las pruebas **unitarias, de integración y E2E** en la pirámide y **por qué hay pocas E2E**.
3. Señalar **Arrange, Act y Assert** en el test de **otro** integrante.
4. Por qué se usan **`getByRole` / `getByLabel`** y por qué se evita **`waitForTimeout`**.
5. Qué **mide** la cobertura, qué **no mide**, y cuál es la **línea base** del proyecto.

---

## 2.2 Por qué probar

Una **prueba automatizada** es código que ejecuta la aplicación y comprueba que se comporta como se espera. Se
repite sola, en segundos, cada vez que se corre. Sin pruebas pasa lo de siempre en un grupo: alguien cambia el
backend, nadie vuelve a probar el login a mano y la falla aparece el día de la sustentación.

### Error, defecto y falla

| Término | Qué es | Ejemplo en ServiLift |
|---|---|---|
| **Error** | La equivocación humana | El programador escribe `fecha_programada <= hoy` en vez de `< hoy` |
| **Defecto** (bug) | El resultado del error, en el código | La regla rechaza las visitas programadas para **hoy** |
| **Falla** | Lo que ve el usuario | Programación no puede agendar una visita para esta tarde: "No se puede programar una visita en un día pasado" |

Las pruebas buscan **defectos** antes de que se vuelvan **fallas** frente al cliente o al jurado.

### El costo de encontrarlo tarde

| Cuándo se detecta | Cuánto cuesta |
|---|---|
| Mientras escribes el código (el test falla) | Segundos |
| En la revisión del PR | Minutos |
| Al integrar con el grupo | Horas: hay que averiguar qué cambio lo rompió |
| En producción o en la sustentación | Muchísimo: datos dañados, reputación, retrabajo |

Por eso la idea es **probar continuamente**, no al final.

### Qué da y qué no da una prueba

Da: confianza para cambiar código, documentación viva (el nombre del test explica la regla), menos trabajo
repetido y **evidencia** de que el sistema cumple.

No da: **no demuestra que no hay defectos** (solo que los casos probados funcionan); no reemplaza pensar (un test
que nunca falla no sirve); no es gratis (es código que se mantiene).

### Manual vs. automatizada

| | Manual | Automatizada |
|---|---|---|
| Quién la ejecuta | Una persona | Una máquina |
| Velocidad | Minutos por caso | Milisegundos a segundos |
| Repetible | Depende de la persona | Siempre igual |
| Para qué sirve | Explorar, descubrir lo inesperado | Verificar lo conocido una y otra vez |

No compiten: se explora a mano y se automatiza lo que ya se sabe que debe funcionar.

---

## 2.3 La pirámide de pruebas

```
            ▲  pocas      E2E (Playwright)              lentas, convincentes, frágiles
           ▲▲▲            Integración (TestClient + BD)  piezas juntas
        ▲▲▲▲▲▲▲  muchas   Unitarias (Vitest, pytest)     rápidas, baratas, precisas
```

| Nivel | Qué prueba | Velocidad | Ejemplo en ServiLift |
|---|---|---|---|
| **Unitaria** | Una función aislada | Milisegundos | `calcular_plazo` devuelve el plazo más exigente entre los hallazgos |
| **Integración** | Varias piezas juntas (API + BD) | Décimas a segundos | `tests/test_flujo.py`: cotización → inspección → informe → certificado contra MySQL |
| **E2E** | La app completa como la usa una persona | Segundos | El asesor ingresa y busca un cliente en el navegador |

**¿Por qué pocas E2E?** Son las más convincentes, pero las más lentas (levantan backend, app y navegador), las más
frágiles (cualquier cambio de pantalla las rompe) y cuando fallan no dicen **dónde** está el problema. Se reservan
para los **flujos críticos**.

**Antipatrón: cono de helado.** Casi todo se prueba a mano o con E2E y casi nada con unitarias: la suite es lenta y
cuando algo falla nadie sabe por qué.

### Dónde vive cada nivel en nuestro stack

| Nivel | App (Expo / React Native) | Backend (FastAPI) |
|---|---|---|
| Unitaria | Vitest (`servilift-app/tests/`) | pytest (`servilift-backend/tests/unit/`) |
| Integración (API) | — | `TestClient` de FastAPI (semana 4) |
| Integración (BD) | — | pytest contra MySQL (`tests/test_flujo.py`, `-m integracion`) |
| E2E | Playwright (`e2e/`), común a todo | |

El bootcamp **empieza por la cima** (E2E) porque es lo más fácil de entender: hace lo mismo que harías a mano.

---

## 2.4 El patrón AAA

Todo test, en cualquier nivel y lenguaje, tiene tres partes:

1. **Arrange** (preparar): datos y estado inicial.
2. **Act** (actuar): la acción que se prueba, idealmente una sola.
3. **Assert** (verificar): comprobar el resultado.

Ejemplo real de ServiLift (`e2e/tests/login-incorrecto.spec.js`):

```javascript
test('should show an error and stay on the login page when the password is wrong', async ({ page }) => {
  // Arrange
  await page.goto('/');

  // Act
  await page.getByLabel('Correo').fill('asesor@gmail.com');
  await page.getByLabel('Contraseña', { exact: true }).fill('clave-incorrecta');
  await page.getByRole('button', { name: 'Ingresar' }).click();

  // Assert: el backend responde 401 y la app muestra el mensaje sin dejar entrar
  await expect(page.getByRole('alert')).toHaveText('Correo o contraseña incorrectos');
  await expect(page.getByRole('button', { name: 'Ingresar' })).toBeVisible();
});
```

### Un test debe poder fallar

Un test que siempre pasa no protege nada. Por cada test: (1) hacerlo pasar, (2) **romperlo a propósito**, (3) ver
que falla, (4) deshacer. Lo hicimos con el test anterior cambiando el texto esperado por `'Otra cosa'`:

```
Error: expect(locator).toHaveText(expected) failed
Expected: "Otra cosa"
Received: "Correo o contraseña incorrectos"
1 failed
```

Fíjate que Playwright **esperó 15 s reintentando** antes de declarar la falla y muestra lo esperado y lo recibido.

### Nombres que documentan

El nombre dice **qué comportamiento** se espera y **en qué condición**:
✅ `should ask for email and password when the form is submitted empty` · ❌ `test1`, `pruebaLogin`.

---

## 2.5 Cobertura: la calidad se mide desde el día 1

La **cobertura** es el porcentaje del código que **se ejecuta** cuando corren los tests. Se mide por **líneas**,
**ramas** (cada `if` tiene dos caminos), **funciones** y **sentencias**.

| Parte | Herramienta | Configuración | Comando |
|---|---|---|---|
| App | Vitest + `@vitest/coverage-v8` | `servilift-app/vitest.config.mts` | `npm test` |
| Backend | pytest + `pytest-cov` | `servilift-backend/pyproject.toml` | `venv\Scripts\python -m pytest` |

Reglas del bootcamp:

- Se mide la **lógica de negocio**. El arranque, la configuración y los adaptadores quedan fuera **con un comentario**.
- La semana 1 registra la **línea base**; desde la semana 2 el CI rechaza los PR que no cumplen el umbral.
- El umbral sube cada semana hasta el **80%** y **nunca baja**.
- **Los E2E no suman cobertura**: prueban desde afuera, en otro proceso.

> ⚠️ La cobertura dice qué código **se ejecutó**, no qué código **se verificó**. Un test sin aserciones sube el
> número y no protege nada.

### Nuestra línea base (y una lección)

| Parte | Línea base S1 | Por qué |
|---|---|---|
| App | **0%** | No había ninguna prueba unitaria |
| Backend | **41%** | ¡Sin ningún test unitario! pytest **importa** la app al cargar `tests/test_flujo.py` y Python ejecuta todas las líneas `def`, `class` y decoradores. Esa cobertura no verifica nada |

Esa línea base quedó como umbral (`fail_under = 41`, `thresholds: 0`). Y comprobamos que **el umbral muerde**:
subirlo a 99 hace fallar el comando (`FAIL Required test coverage of 99% not reached`).

---

## 2.6 Playwright: anatomía de un test E2E

**Playwright** (de Microsoft) controla un navegador real (Chromium, Firefox o WebKit) desde código: abre la página,
escribe, hace clic y verifica lo que aparece. Como **solo habla con el navegador**, no le importa si el backend es
FastAPI, Express o Spring Boot.

| Pieza | Para qué sirve |
|---|---|
| `test('nombre', async ({ page }) => …)` | Declara un test. `page` es una pestaña nueva y limpia |
| `page.goto('/')` | Navega a una ruta relativa a la `baseURL` |
| `page.getByLabel()`, `page.getByRole()` | **Locators**: encuentran elementos como los encuentra una persona |
| `.fill()`, `.click()`, `.press('Enter')` | **Acciones** |
| `await expect(locator).toHaveText()` | **Aserción**: verifica y **espera** a que se cumpla |
| `test.describe` / `test.beforeEach` | Agrupar tests y compartir el Arrange (ver `buscar-cliente.spec.js`) |

### Locators: busca como busca una persona

| Prioridad | Locator | Ejemplo nuestro |
|---|---|---|
| 1 | `getByRole` | `page.getByRole('button', { name: 'Ingresar' })` |
| 2 | `getByLabel` | `page.getByLabel('Correo')` |
| 3 | `getByText`, `getByPlaceholder` | `page.getByText('Sin resultados para la búsqueda')` |
| Último recurso | `getByTestId` | — (no lo necesitamos) |

❌ Se evitan selectores CSS como `.css-175oi2r > div:nth-child(2) input`: React Native Web genera clases
automáticas que cambian; se rompen con cualquier cambio de diseño.

**Accesibilidad (hallazgo real de la semana):** al grabar el login, los campos aparecían como `textbox` **sin nombre**:
`getByLabel('Correo')` no los encontraba. React Native Paper dibuja la etiqueta como texto, pero no la asocia al
`<input>`. Se corrigió en la app agregando `accessibilityLabel="Correo"`, `"Contraseña"`, `"Buscar clientes"`,
`"Mostrar contraseña"` y `role="alert"` al mensaje de error (`servilift-app/app/login.tsx` y
`src/pantallas/clientes/Clientes.tsx`). Ese cambio **mejora los tests y la app** (lectores de pantalla).

### Espera automática: nada de `sleep`

- Antes de hacer clic, Playwright **espera** a que el elemento exista, sea visible y esté habilitado.
- `await expect(...)` **reintenta** hasta que se cumple o se agota el tiempo (en nuestro config, 15 s).

`page.waitForTimeout(3000)` es malo por partida doble: **lento** cuando sobra tiempo y **falla igual** cuando falta
(test *flaky*, inestable).

### Codegen: grabar primero, limpiar después

```
cd e2e
pnpm codegen          # abre la app en http://localhost:8091 y genera código mientras la usas
```

Lo grabado es un **borrador**: le faltan **aserciones**, a veces elige mal los locators y no tiene estructura.
Se limpia: quitar `click()` antes de `fill()`, usar `page.goto('/')`, organizar en AAA, agregar la aserción y un
nombre descriptivo.

### Nuestra configuración (`e2e/playwright.config.js`) explicada

| Opción | Valor | Por qué |
|---|---|---|
| `testDir` | `./tests` | Dónde están los `.spec.js` |
| `timeout` / `expect.timeout` | 60 s / 15 s | Expo compila la app web la primera vez que se abre |
| `use.baseURL` | `http://localhost:8091` | Para escribir `page.goto('/')` |
| `use.trace` | `on-first-retry` | Guarda la traza paso a paso solo si un test falla y se reintenta |
| `reporter` | `list` + `html` | Lista en consola y reporte HTML (`pnpm report`) |
| `retries` | 1 en CI, 0 en local | En local queremos ver la falla de una vez |
| `projects` | Chromium de escritorio | Navegador donde corre la suite |
| `webServer[0]` | `uvicorn` en **8010** con `DB_NAME=servilift_v2_prueba` | Playwright levanta el backend contra la **base de pruebas**, nunca la de desarrollo |
| `webServer[1]` | Expo web en **8091** con `EXPO_PUBLIC_API_URL=http://localhost:8010` | La app de prueba habla con el backend de prueba |
| `reuseExistingServer` | `!CI` | En local reutiliza un servidor ya abierto |

> El bootcamp advierte: **no usar la BD de desarrollo para los tests**. Por eso existe `servilift_v2_prueba` y
> los puertos 8010/8091 no chocan con el backend (8000) y Expo (8081) del día a día.

Comandos:

| Comando (en `e2e/`) | Qué hace |
|---|---|
| `pnpm test` | Corre toda la suite sin ventana (*headless*) |
| `pnpm test:headed` | Muestra el navegador |
| `pnpm test:ui` | Modo interactivo paso a paso con línea de tiempo |
| `pnpm report` | Abre el reporte HTML de la última ejecución |
| `pnpm codegen` | Graba un flujo nuevo |

---

## 2.7 Lo que se implementó en ServiLift

| Archivo | Qué es |
|---|---|
| `e2e/package.json` | Proyecto de pruebas E2E: `@playwright/test` **1.64.0 exacto** y `packageManager: pnpm@10.34.5` |
| `e2e/playwright.config.js` | Configuración explicada arriba |
| `e2e/tests/login-exitoso.spec.js` | El asesor ingresa y ve "Inicio" y "Hola, Asesor" |
| `e2e/tests/login-incorrecto.spec.js` | Contraseña errada → alerta "Correo o contraseña incorrectos" y sigue en el login |
| `e2e/tests/login-campos-vacios.spec.js` | Formulario vacío → "Ingrese correo y contraseña" (sin llamar al backend) |
| `e2e/tests/buscar-cliente.spec.js` | El asesor busca "POR-DEFINIR" y ve el cliente; con un texto único ve "Sin resultados" |
| `e2e/tests/cerrar-sesion.spec.js` | Programación cierra sesión, confirma el diálogo y vuelve al login |
| `docs/matriz-rotacion.md` | Matriz de capas, flujos por integrante, línea base |
| `.github/pull_request_template.md` | Plantilla del PR (capa del autor y del revisor, checklist) |
| `servilift-app/vitest.config.mts` y `servilift-backend/pyproject.toml` | Cobertura con la línea base como umbral |

Resultado: **6 tests E2E en verde** (~25 s), y cada uno se hizo fallar a propósito.

¿Por qué el texto único en la búsqueda (`no-existe-${Date.now()}`)? Porque la base conserva datos entre ejecuciones:
un texto fijo podría coincidir algún día con un cliente creado en otra prueba.

---

## 2.8 Preguntas probables del instructor (con respuesta)

**¿Qué flujo prueba el test de tu compañero y por qué es crítico?**
`login-exitoso`: sin ingreso ningún rol puede trabajar, ni el inspector en campo. `buscar-cliente`: el asesor
empieza todo el proceso (cotización) buscando al cliente.

**Señala el Arrange, Act y Assert de `cerrar-sesion.spec.js`.**
Arrange: abrir `/`, ingresar como programación y esperar "Inicio". Act: clic en "Cerrar sesión" y confirmar en el
diálogo. Assert: vuelve a verse el botón "Ingresar" y el campo "Correo".

**¿Qué cambio en la app haría fallar ese test?**
Cambiar el texto del botón, quitar el diálogo de confirmación, que el backend no responda, o borrar el
`accessibilityLabel` del campo Correo (el locator dejaría de encontrarlo).

**¿Por qué `getByRole`/`getByLabel` y no CSS?** Buscan como una persona (rol y nombre visible), resisten cambios de
diseño y obligan a que la app sea accesible. Las clases CSS de React Native Web son generadas y cambian.

**¿Por qué no `waitForTimeout`?** Porque es una espera fija: lenta cuando sobra y falla cuando falta. Playwright ya
espera solo y las aserciones reintentan.

**¿Por qué hay pocas E2E?** Son lentas, frágiles y no localizan el defecto. Van solo en flujos críticos; la base
de la pirámide son las unitarias.

**¿Qué mide la cobertura y qué no? ¿Cuál es su línea base?** Mide qué líneas/ramas se ejecutaron; no mide si se
verificaron. Línea base: app 0%, backend 41% (que salió **sin tests**, solo por importar módulos).

**¿Los E2E suben la cobertura?** No: corren el sistema desde afuera, en otros procesos.

**Error, defecto, falla con ejemplo del proyecto.** Ver la tabla de 2.2 (la regla de "día pasado").

---

## 2.9 Glosario

- **AAA**: preparar, actuar, verificar.
- **Aserción**: verificación; si no se cumple, el test falla (`expect(...)`, `assert`).
- **Auto-waiting**: Playwright espera a que el elemento esté listo y reintenta las aserciones.
- **baseURL**: dirección base de la app en el config.
- **Codegen**: graba acciones y genera un borrador de test.
- **Cobertura / línea base / umbral**: % ejecutado / medida inicial / mínimo que hace fallar el comando.
- **Cono de helado**: antipatrón con casi todo manual o E2E.
- **E2E**: prueba de la app completa desde el usuario.
- **Flaky**: test que a veces pasa y a veces falla sin cambios.
- **Headless / headed**: navegador sin ventana / con ventana.
- **Locator**: forma de encontrar un elemento.
- **Suite**: conjunto de tests.
- **webServer**: opción que levanta la app antes de los tests.
