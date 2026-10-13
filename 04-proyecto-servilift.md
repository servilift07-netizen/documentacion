# 4. ServiLift v2 — Documentación completa del proyecto

Repositorio: https://github.com/servilift07-netizen/ServiLift_react_n

---

## 4.1 Qué es ServiLift

Sistema para una empresa de **certificación de ascensores bajo la norma NTC 5926** (hoy NTC 5926-1:2012, preparado
para ediciones nuevas y para las partes 2 y 3). Cubre todo el proceso: el asesor registra al cliente y cotiza, el
cliente aprueba, programación agenda la visita, el inspector inspecciona en campo (incluso **sin señal**), el
director técnico revisa y firma el informe, certificados emite el certificado con QR y el cliente lo descarga.

### Roles

| Rol | Dónde trabaja | Qué hace |
|---|---|---|
| Administrador | Navegador | Usuarios, normas y checklist, parámetros del sistema |
| Asesor | Navegador | Clientes, edificios, equipos y cotizaciones |
| Cliente | Navegador (portal) | Aprueba cotizaciones, solicita 2ª visita, descarga informes y certificados |
| Programación | Navegador | Agenda visitas (inspector, fecha, hora) |
| Inspector | **Celular** | Agenda, datos del equipo, checklist, fotos y firmas, con o sin señal |
| Director técnico | Navegador | Revisa, devuelve o aprueba y firma el informe (genera el PDF) |
| Certificados | Navegador | Elabora, envía o anula certificados |

### Reglas del negocio

- Cada equipo cotizado tiene su **norma** (por defecto la edición vigente).
- **Solo hay dos visitas** por servicio:
  1. **1ª visita** → informe `INF-AAAA-NNNN-1V`. Conforme: certificado. No conforme: plazo de corrección = el
     menor entre los hallazgos (leve 180 días, grave 30, muy grave inmediato; configurable por norma o por ítem).
  2. **2ª visita** → informe `INF-AAAA-NNNN-2V`. Todo corregido: certificado. Algo sin corregir: el equipo queda
     **cerrado no conforme** y necesita **cotización nueva** (nueva 1ª visita) enlazada al servicio anterior.
- Un informe por visita, con su revisión, firma y PDF. Todo cambio de estado queda en el historial.
- Consecutivos por año: `COT-2026-0001`, `INS-…`, `INF-…`, `CER-…`.

### Estados de cada equipo cotizado

```
PENDIENTE_APROBACION → APROBADO_CLIENTE → POR_PROGRAMAR → PROGRAMADO → EN_INSPECCION → EN_REVISION
EN_REVISION ─ 1ª visita conforme ─────────────→ INFORME_APROBADO → CERTIFICADO_LISTO
EN_REVISION ─ 1ª visita no conforme → NO_CONFORME → SEGUNDA_VISITA_SOLICITADA → PROGRAMADO → … → EN_REVISION
EN_REVISION ─ 2ª visita todo corregido ───────→ INFORME_APROBADO
EN_REVISION ─ 2ª visita algo sin corregir ────→ CERRADO_NO_CONFORME (cotización nueva)
Además: RECHAZADO y ANULADO.
```

La tabla `TRANSICIONES` de `app/services/flujo.py` es la única fuente de verdad: cualquier otro paso responde 409.

---

## 4.2 Arquitectura

```
┌──────────────────────────────┐  HTTP/JSON + JWT   ┌─────────────────────────┐   SQL   ┌──────────────────┐
│ App React Native + Expo      │ ─────────────────▶ │ API FastAPI             │ ──────▶ │ MariaDB / MySQL  │
│ (servilift-app)              │ ◀───────────────── │ (servilift-backend)     │ ◀────── │ servilift_db     │
│ · Android / iOS: inspector   │ fotos, firmas, PDF │ · reglas de negocio     │         └──────────────────┘
│ · Navegador: oficina/cliente │                    │ · PDF (ReportLab) + QR  │
│ · Cola sin conexión          │                    │ · archivos con enlaces  │
└──────────────────────────────┘                    │   firmados que vencen   │
                                                    └─────────────────────────┘
```

Una sola base de código de app sirve para **celular y navegador** (React Native Web). El backend es una API REST;
la base guarda todo en hora de Colombia.

---

## 4.3 Stack tecnológico y para qué sirve cada cosa

### Base de datos

| Tecnología | Para qué |
|---|---|
| **MariaDB 10.4 (XAMPP) / MySQL 8** | Motor relacional donde viven clientes, cotizaciones, inspecciones, informes, etc. |
| **Alembic** | Migraciones versionadas del esquema (`alembic/versions/0001…0004`); la tabla `alembic_version` dice en qué versión está la base |
| Scripts SQL (`database/`) | Crear la base desde cero: esquema, catálogos (norma, 169 ítems de checklist) y datos de prueba |

### Backend (`servilift-backend`)

| Tecnología | Versión | Para qué |
|---|---|---|
| **Python** | 3.13 | Lenguaje del backend |
| **FastAPI** | 0.141 | Framework web: define los endpoints, valida datos y genera la documentación en `/docs` |
| **Uvicorn** | 0.52 | Servidor ASGI que ejecuta la app FastAPI |
| **SQLAlchemy** | 2.0 | ORM: clases Python ↔ tablas (`app/models.py`) |
| **PyMySQL** + cryptography | | Controlador para conectarse a MySQL/MariaDB |
| **Pydantic** / pydantic-settings | 2 | Validación de lo que llega en cada petición y lectura del `.env` |
| **python-jose** | | Tokens JWT de sesión y de enlaces de descarga |
| **bcrypt** | | Hash de contraseñas |
| **python-multipart** | | Recibir fotos y firmas como formulario |
| **ReportLab** | 4.2 | Generar el PDF del informe y del certificado (con QR) |
| **Pillow** | | Reducir fotos y crear miniaturas |
| **httpx** | | Cliente HTTP que usa `TestClient` en las pruebas |
| **pytest** + **pytest-cov** | 9.1 / 7.1 | Pruebas y cobertura (`requirements-dev.txt`) |

### App (`servilift-app`)

| Tecnología | Para qué |
|---|---|
| **Expo SDK 57** | Plataforma sobre React Native: compila para Android, iOS y web, y da módulos nativos (cámara, archivos…) |
| **React 19 + React Native 0.86** | Interfaz con componentes |
| **React Native Web** | La misma app corre en el navegador (oficina y cliente) |
| **TypeScript 6** | Tipos estáticos: errores detectados antes de ejecutar (`npm run typecheck`) |
| **Expo Router** | Navegación por archivos: cada archivo de `app/` es una ruta |
| **React Native Paper** | Componentes Material Design (botones, tarjetas, campos) |
| expo-secure-store / AsyncStorage | Guardar la sesión de forma segura y la cola sin conexión en el teléfono |
| @react-native-community/netinfo | Saber si hay señal para sincronizar |
| expo-image-picker / image-manipulator / file-system | Tomar fotos, reducirlas y guardarlas en el teléfono |
| expo-crypto | Generar uuid para que un reenvío no duplique fotos y firmas |
| react-native-svg | Firmas a mano y gráficas del inicio |
| **Vitest** + @vitest/coverage-v8 | Pruebas unitarias y cobertura de la lógica de la app |

### Pruebas y calidad

| Herramienta | Para qué |
|---|---|
| **Vitest** | Unitarias de la app (`servilift-app/tests/`) |
| **pytest** | Unitarias del backend (`tests/unit/`) e integración (`tests/test_flujo.py`) |
| **Playwright** | E2E en un navegador real (`e2e/`) |
| **pnpm** | Gestor de paquetes del proyecto E2E |
| **GitHub Actions** | CI: corre las pruebas y exige el umbral de cobertura en cada PR |

---

## 4.4 Estructura del repositorio

```
servi lift react/
├── .github/
│   ├── workflows/tests.yml          CI: jobs frontend, backend y evidencia-individual
│   └── pull_request_template.md     Plantilla de cada PR (capas autor/revisor, checklist)
├── database/
│   ├── 01_schema.sql                Tablas, índices, vistas y versión de Alembic
│   ├── 02_seed.sql                  Roles, norma NTC 5926-1:2012, variantes, 169 ítems y parámetros
│   ├── 03_datos_iniciales.sql       Usuarios de prueba (contraseña 123456) y un cliente con 2 edificios
│   └── README.md                    Explicación del modelo, estados y cómo agregar normas
├── docs/
│   └── matriz-rotacion.md           Bootcamp: capas por integrante, cobertura semanal, mutantes, vocero
├── e2e/                             Pruebas de extremo a extremo
│   ├── package.json / pnpm-lock.yaml   Playwright 1.64.0 fijo
│   ├── playwright.config.js         Levanta backend (8010) y app (8091) contra servilift_v2_prueba
│   └── tests/*.spec.js              Ingreso, errores de ingreso, búsqueda de clientes, cierre de sesión
├── servilift-backend/
│   ├── app/                         Código de la API (ver 4.5)
│   ├── alembic/ + alembic.ini       Migraciones del esquema
│   ├── scripts/crear_admin.py       Crear el primer administrador
│   ├── tests/
│   │   ├── unit/                    Pruebas unitarias (sin base de datos)
│   │   └── test_flujo.py            Prueba de integración del flujo completo (necesita MySQL)
│   ├── storage/                     Fotos, firmas y PDF generados (no se sube a git)
│   ├── requirements.txt             Dependencias de producción
│   ├── requirements-dev.txt         + pytest y pytest-cov
│   ├── pyproject.toml               Configuración de pytest y umbral de cobertura
│   └── .env.example                 Variables de entorno de ejemplo
├── servilift-app/
│   ├── app/                         Rutas (Expo Router, ver 4.6)
│   ├── src/                         Lógica, componentes, pantallas, sesión y modo sin conexión
│   ├── tests/                       Pruebas unitarias con Vitest
│   ├── assets/                      Íconos y splash
│   ├── app.json                     Configuración de Expo (nombre, paquete Android, permisos)
│   ├── vitest.config.mts            Configuración de pruebas y umbral de cobertura
│   ├── package.json / package-lock.json
│   └── .env.example                 EXPO_PUBLIC_API_URL (dirección del backend)
├── instalar.ps1                     Instalación en un PC nuevo
├── GUIA_EJECUCION.md                Paso a paso para ejecutar
└── README.md
```

---

## 4.5 Backend: módulos

| Archivo | Para qué sirve |
|---|---|
| `app/main.py` | Crea la app FastAPI, configura CORS, registra los routers y sirve fotos/firmas solo con enlace firmado |
| `app/config.py` | Lee el `.env` (base de datos, zona horaria, JWT, archivos, CORS) con validación |
| `app/database.py` | Motor de SQLAlchemy, sesión por petición (`get_db`) y zona horaria de la conexión |
| `app/models.py` | Modelos ORM de todas las tablas |
| `app/security.py` | bcrypt, JWT, `require_roles` (permiso por rol), enlaces de descarga y enlaces firmados HMAC |
| `app/tiempo.py` | Hora de Colombia: `ahora()`, `hoy()`, conversión de fechas con zona |
| `app/services/flujo.py` | Consecutivos, máquina de estados (`TRANSICIONES`, `puede_pasar`, `cambiar_estado`), notificaciones |
| `app/services/checklist.py` | Ítems que aplican según variantes, resumen de hallazgos, concepto de visita, plazo, pendientes para finalizar |
| `app/services/reglas.py` | Reglas puras: horario de programación y parámetros numéricos |
| `app/services/archivos.py` | Guardar fotos, firmas y PDF en `storage/` |
| `app/services/pdf.py` | PDF del informe (estructura de la plantilla NTC 5926-1) y del certificado con QR |
| `app/routes/*.py` | Endpoints (ver tabla) |
| `app/routes/serializers.py` | Convierte modelos en el JSON que recibe la app |

### Endpoints principales

| Router | Prefijo | Ejemplos |
|---|---|---|
| auth | `/auth` | `POST /login`, `GET /me`, `POST /cambiar-contrasena` |
| usuarios | `/usuarios` | listar, crear, editar, `PUT /me/firma` |
| clientes | `/clientes`, `/edificios`, `/equipos` | CRUD de clientes, edificios y equipos; generar equipos |
| cotizaciones | `/cotizaciones` | crear, editar, enviar al cliente, responder, enviar a programación, anular |
| programacion | `/programacion` | pendientes, programar, reprogramar, cancelar |
| inspecciones | `/inspecciones` | agenda, iniciar, encabezado, equipo, variantes, checklist, resultados, segunda visita, fotos, firmas, validar, finalizar |
| informes | `/informes` | listar, PDF, vista previa, devolver, aprobar |
| certificados | `/certificados`, `/descargas`, `/verificar` | pendientes, emitir, enviar, anular, PDF, verificación pública por QR |
| seguimiento | `/seguimiento` | estado de cada equipo, historial, solicitar 2ª visita |
| catalogos | `/catalogos`, `/instrumentos`, `/notificaciones` | normas, variantes, checklist, parámetros, notificaciones |
| dashboard | `/dashboard` | indicadores y gráficas del inicio por rol |

Documentación interactiva: `http://localhost:8000/docs`.

---

## 4.6 App: organización

| Carpeta / archivo | Para qué sirve |
|---|---|
| `app/_layout.tsx` | Raíz: tema, sesión y sincronización para toda la app |
| `app/login.tsx` | Pantalla de ingreso |
| `app/(app)/_layout.tsx` | Menú lateral y protección de rutas (solo con sesión) |
| `app/(app)/…` | Una ruta por pantalla: inicio, agenda, clientes, cotizaciones, programación, inspección, informes, certificados, seguimiento, notificaciones, sincronización, administración |
| `src/pantallas/` | El contenido de cada pantalla (las rutas de `app/` solo las muestran) |
| `src/componentes/` | Piezas reutilizables: campos, diálogos, calendario, reloj, firma, gráficas, plazo |
| `src/logica/` | **Lógica pura** probada con Vitest: formatos, estados, plazo, recorrido, ingreso, notificaciones, checklist local |
| `src/api/cliente.ts` | Cliente HTTP del backend (token, errores, subida de archivos, PDF) |
| `src/sesion/` | Contexto de sesión y almacenamiento seguro del token |
| `src/offline/` | Modo sin conexión: cola persistente, fotos locales y motor de sincronización |
| `src/hooks/useCarga.ts` | Carga de datos sin pantallas de espera, con caché |
| `src/tema.ts` | Colores, tema de Paper y reexporta los formatos |
| `src/config.ts` | URL del backend (`EXPO_PUBLIC_API_URL`) |

### Trabajo sin señal (inspector)

1. Con señal, al abrir la agenda, cada inspección se guarda en el teléfono.
2. Cada cambio se guarda **primero en el teléfono** en una cola persistente (sobrevive a cerrar la app).
3. La cola se envía sola al haber señal, en orden, con reintentos.
4. Sin duplicados: fotos y firmas llevan uuid; cada resultado lleva la hora del cambio y **el más reciente gana**.
5. Un error del servidor no borra el cambio: queda en *Sincronización* para reintentar o descartar.
6. Abrir la inspección por primera vez y finalizar requieren señal.

---

## 4.7 Seguridad

- Contraseñas con bcrypt; sesión con JWT (12 h); permisos por rol en cada endpoint.
- El cliente solo ve lo suyo: informes aprobados y certificados enviados.
- Fotos y firmas solo con **enlaces firmados (HMAC) que vencen**; PDF con enlaces temporales de 10 minutos.
- Proyecto de demostración: arranca sin clave secreta y todos los usuarios usan `123456`. Para producción se
  define `SECRET_KEY` y `CORS_ORIGINS` en el `.env`.

---

## 4.8 Pruebas automáticas

| Nivel | Herramienta | Dónde | Cantidad | Comando |
|---|---|---|---|---|
| Unitarias app | Vitest | `servilift-app/tests/` | 103 | `npm test` |
| Unitarias backend | pytest | `servilift-backend/tests/unit/` | 70 | `venv\Scripts\python -m pytest` |
| Integración | pytest + MySQL | `servilift-backend/tests/test_flujo.py` | 1 flujo completo | `pytest -m integracion --no-cov` |
| E2E | Playwright | `e2e/tests/` | 6 | `pnpm test` |

Cobertura y umbrales (semana 2): app 41% líneas, backend 44%. El CI los exige en cada pull request.
Detalle en las guías de la [semana 1](02-semana-01-por-que-probar-y-primer-e2e.md) y la
[semana 2](03-semana-02-pruebas-unitarias-y-ci.md).

---

## 4.9 Puertos, bases y variables de entorno

| Uso | Backend | App | Base de datos |
|---|---|---|---|
| Día a día | 8000 | 8081 | `servilift_db` |
| Pruebas E2E e integración | 8010 | 8091 | `servilift_v2_prueba` |

| Variable | Dónde | Para qué |
|---|---|---|
| `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`, `DB_NAME` | backend `.env` | Conexión a la base |
| `ZONA_HORARIA` | backend | Hora de las fechas (`-05:00`) |
| `SECRET_KEY` | backend | Firma de tokens y enlaces (opcional en demo) |
| `STORAGE_DIR`, `MAX_FOTO_MB`, `ARCHIVOS_URL_MINUTOS` | backend | Archivos y vigencia de enlaces |
| `URL_VERIFICACION` | backend | URL del QR del certificado |
| `CORS_ORIGINS` | backend | Orígenes web permitidos en producción |
| `EXPO_PUBLIC_API_URL` | app `.env` | Dirección del backend (localhost, `10.0.2.2` en emulador o IP del PC en el celular) |

---

## 4.10 Instalar y ejecutar en otro PC

Requisitos: Git, XAMPP (MySQL), Python 3.13, Node.js 22+.

```
git clone -b version-2.1 https://github.com/servilift07-netizen/ServiLift_react_n.git "servi lift react"
cd "servi lift react"
powershell -ExecutionPolicy Bypass -File .\instalar.ps1 -ConBaseDeDatos
```

El script (probado en un clon limpio) instala todo, copia los `.env` y crea las dos bases si no existen. Luego:

```
cd servilift-backend ; venv\Scripts\activate ; uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
cd servilift-app ; npx expo start --web            # http://localhost:8081
```

Usuarios (contraseña `123456`): admin@gmail.com, asesor@gmail.com, programacion@gmail.com, inspector@gmail.com,
director@gmail.com, certificados@gmail.com, cliente@gmail.com.

Celular: poner la IP del PC en `servilift-app/.env` (`EXPO_PUBLIC_API_URL=http://192.168.x.x:8000`), correr
`npx expo start` y escanear el QR con Expo Go (misma Wi-Fi; abrir los puertos 8000 y 8081 en el firewall).

---

## 4.11 Versiones

| Rama / etiqueta | Qué es |
|---|---|
| `v2.0` (etiqueta) | Punto seguro: la versión 2.0 antes del bootcamp de pruebas |
| `main` | Versión estable entregada |
| `version-2.1` | Versión en curso: aquí se integran los cambios por pull request |
| `test/semana-01`, `test/semana-02` | Trabajo de cada semana del bootcamp |

Para volver a la versión anterior: `git checkout v2.0`.
