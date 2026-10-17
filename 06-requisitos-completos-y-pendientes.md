# ServiLift: requisitos completos y pendientes

17/10/2026 · Esteban

## Resumen

ServiLift cumple por completo 33 de los 37 requisitos del documento (89 %) y tiene 4 a medias; contando los parciales como medio requisito, el avance es del 95 %. La meta del 80 % ya se supera. Ningún requisito falta del todo.

| Tipo | Total | Completos | Parciales | Faltan | % completo | % con parciales |
| --- | --- | --- | --- | --- | --- | --- |
| Funcionales (RF) | 27 | 23 | 4 | 0 | 85 % | 93 % |
| No funcionales (RNF) | 10 | 10 | 0 | 0 | 100 % | 100 % |
| **Total** | **37** | **33** | **4** | **0** | **89 %** | **95 %** |

Se comparó el documento `ServiLift_Documentacion1.docx` (capítulo 6 y casos de uso del capítulo 8) con la rama `main` del repositorio ServiLift\_react\_n y con las 30 pruebas E2E del 14/10/2026. Un requisito es *Parcial* cuando la app hace la parte principal pero falta algo que el documento promete.

## Requisitos funcionales

23 de 27 están completos y 4 son parciales (RF-002, RF-005, RF-015 y RF-020). Tres requisitos completos (RF-007, RF-019 y RF-023) funcionan en la app, pero el texto del documento no coincide y hay que corregirlo.

| Código | Requisito | Estado | Qué falta |
| --- | --- | --- | --- |
| RF-001 | Iniciar sesión | Completo | Nada |
| RF-002 | Gestionar usuarios | Parcial | La pantalla solo ofrece Activo e Inactivo: falta el estado Bloqueado. El botón se llama "Crear usuario", no "Crear acceso al portal" |
| RF-003 | Configurar normas y checklist | Completo | Nada |
| RF-004 | Configurar parámetros | Completo | Nada |
| RF-005 | Gestionar instrumentos de medición | Parcial | Existen la tabla y la API (`GET` y `POST /instrumentos`), pero no hay pantalla Administración > Instrumentos para registrarlos ni asignarlos al inspector. La inspección muestra "No hay instrumentos registrados" |
| RF-006 | Registrar clientes, edificios y equipos | Completo | Nada |
| RF-007 | Crear y enviar cotización | Completo | Solo el documento: el número COT y el estado PENDIENTE\_APROBACION se asignan al crear la cotización, no al enviarla |
| RF-008 | Aprobar o rechazar cotización | Completo | Nada |
| RF-009 | Enviar a programación | Completo | Nada |
| RF-010 | Programar visita | Completo | Nada |
| RF-011 | Reprogramar o cancelar visita | Completo | Nada |
| RF-012 | Consultar agenda | Completo | Nada |
| RF-013 | Realizar inspección | Completo | Nada (el PR #7 mejora el No aplica precargado y sigue abierto) |
| RF-014 | Registrar evidencia fotográfica | Completo | Nada |
| RF-015 | Registrar firmas | Parcial | Las 3 firmas funcionan. Falta lo que promete el CU-15: que el inspector guarde su firma en el perfil y la app la proponga. El backend ya tiene `PUT /usuarios/me/firma`, pero la app no lo usa |
| RF-016 | Trabajar sin señal | Completo | Nada |
| RF-017 | Finalizar inspección | Completo | Nada |
| RF-018 | Revisar y aprobar informe | Completo | Nada |
| RF-019 | Devolver informe al inspector | Completo | Solo el documento: la inspección queda DEVUELTA y es el equipo el que vuelve a EN\_INSPECCION |
| RF-020 | Solicitar segunda visita | Parcial | El backend deja pedirla aunque la fecha límite de corrección ya venció; el CU-20 dice que no se puede |
| RF-021 | Elaborar y enviar certificado | Completo | Nada |
| RF-022 | Anular certificado | Completo | Nada |
| RF-023 | Consultar seguimiento e historial | Completo | Solo el documento: el inspector y Certificados también tienen Seguimiento en su menú |
| RF-024 | Descargar informes y certificados | Completo | Nada |
| RF-025 | Verificar certificado por QR | Completo | Nada |
| RF-026 | Consultar tablero e indicadores | Completo | Nada |
| RF-027 | Recibir notificaciones | Completo | Nada |

## Requisitos no funcionales

Los 10 están completos. Docker y uv quedaron en `main` al fusionarse el PR #2.

| Código | Requisito | Estado | Evidencia en el código |
| --- | --- | --- | --- |
| RNF-001 | Trabajo sin conexión | Completo | Cola persistente `src/offline/cola.ts`; probado en la E2E de segunda visita |
| RNF-002 | Seguridad | Completo | PBKDF2-SHA256 de 600.000 iteraciones, JWT de 12 h, permisos por rol y enlace firmado de 10 min |
| RNF-003 | Trazabilidad | Completo | Tabla `historial_estado` y huella SHA-256 del PDF aprobado |
| RNF-004 | Compatibilidad | Completo | Expo SDK 57 para Android, iOS y web |
| RNF-005 | Usabilidad | Completo | Menú y tablero por rol; las notificaciones llevan a la pantalla que resuelve |
| RNF-006 | Mantenibilidad | Completo | Backend por capas y migraciones Alembic 0001 a 0005 |
| RNF-007 | Calidad verificable | Completo | Umbral de cobertura `fail_under = 44` en `pyproject.toml`, GitHub Actions y 30 pruebas E2E |
| RNF-008 | Instalación repetible | Completo | `docker-compose.yml` y `servilift-backend/uv.lock` |
| RNF-009 | Zona horaria | Completo | `app/tiempo.py` guarda todo en hora de Colombia (UTC−5) |
| RNF-010 | Integridad de datos | Completo | 56 llaves foráneas, `SELECT … FOR UPDATE` en los consecutivos y respuesta 409 en transiciones no permitidas |

## Qué falta, en orden de prioridad

Cerrar estos 4 cambios lleva los requisitos funcionales al 100 %. Cada uno iría en su propia rama y PR, con su prueba.

1. **Pantalla Administración > Instrumentos (RF-005, CU-05).** Crear, editar, inactivar y asignar instrumentos a un inspector. Hace falta también el endpoint para editar, porque hoy solo hay `GET` y `POST`.
2. **Estado Bloqueado en usuarios (RF-002, CU-02).** Agregar la opción en la pantalla y que el inicio de sesión lo rechace con un mensaje claro.
3. **Plazo de la segunda visita (RF-020, CU-20).** Rechazar la solicitud si la fecha límite de corrección ya venció.
4. **Firma guardada del inspector (RF-015, CU-15).** Una pantalla para registrar la firma en el perfil y que la inspección la proponga.

Aparte de estos 4, conviene fusionar el PR #7 (No aplica preseleccionado), que sigue abierto.

## Diferencias que se corrigen solo en el documento

Estas no cambian el porcentaje, porque la app funciona; es el texto el que no coincide con lo que el jurado verá en la demo.

| Dónde | Dice el documento | Hace la app |
| --- | --- | --- |
| RF-007, CU-07 | El número COT se asigna al enviar | Se asigna al crear la cotización en borrador |
| RF-019 | La inspección vuelve a EN\_INSPECCION | La inspección queda DEVUELTA y el equipo vuelve a EN\_INSPECCION |
| RF-023, Tabla 9 | Seguimiento para 4 roles | También el inspector y Certificados |
| Casos de uso | "Mis cotizaciones", "Enviar a programación", "Vista previa", "Por elaborar", "Tomar foto" | "Cotizaciones", "Aceptar y enviar a programación", "Ver borrador PDF", "Certificados por elaborar", "Agregar fotos" |
| Tabla 6, R-001 | Ver la sección 9.6 | El trabajo sin señal está en la 10.5 |
| Cap. 14, Tabla 24 | Integración en `tests/integration`, 5 flujos E2E | `servilift-backend/tests/test_flujo.py`; 9 archivos y 30 pruebas E2E |
| Cap. 13, Tabla 11 | `docs/instalacion.md` | La guía es `GUIA_EJECUCION.md` |
| Referencias | Lista APA sin citas en el texto | Faltan citas como (ICONTEC, 2012) y la Resolución 0675 de 2013 en la lista |
