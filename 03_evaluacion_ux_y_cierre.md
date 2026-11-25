# ISWZ2202 – Diseño y Arquitectura de Software · Semana 9

## Servicio de Calendario Corporativo para una Empresa Multinacional

**Documento 03 – Evaluación de la experiencia de usuario (Paso 6), verificación final y declaración de uso de IA**

| Campo | Valor |
|---|---|
| Estudiante | *(Nombre y Apellido)* |
| Marco de evaluación | 10 Heurísticas de usabilidad de Jakob Nielsen |
| Flujo evaluado | Crear un evento con alerta recurrente y visualizarlo en distintas zonas horarias |

---

## 1. Paso 6 – Evaluación de UX con las Heurísticas de Nielsen

### 1.1 Justificación del marco

Se eligen las **Heurísticas de Nielsen** porque la guía las recomienda para la *revisión rápida de una pantalla o flujo concreto* y porque el producto aún no está en producción, por lo que no existen métricas de uso que permitan aplicar HEART. La evaluación se realiza como **inspección heurística** sobre el flujo propuesto (prototipo conceptual de la aplicación web y móvil), de acuerdo con el contrato definido en `openapi.yaml`.

### 1.2 Descripción del flujo evaluado

| Paso | Acción del usuario | Respuesta del sistema (API) |
|---|---|---|
| 1 | Pulsa "Nuevo evento" en el calendario "Equipo de Arquitectura LATAM". | Se abre el formulario con la zona horaria por defecto del calendario. |
| 2 | Escribe título, fecha y hora (09:00), duración de 1 h. | — |
| 3 | Activa "Repetir" → semanal, martes, 10 veces. | Se construye la regla `FREQ=WEEKLY;BYDAY=TU;COUNT=10`. |
| 4 | Añade asistentes de Quito, Madrid y São Paulo. | — |
| 5 | Añade alerta "15 minutos antes" por notificación push y web. | — |
| 6 | Pulsa "Guardar". | `POST /v1/calendars/{calendarId}/events` → `201 Created`. |
| 7 | Cambia la vista a la zona `Europe/Madrid` para ver cómo la verán sus colegas. | `GET .../events?displayTimezone=Europe/Madrid`. |
| 8 | Recibe la alerta en el móvil y la marca como leída. | `PATCH /v1/alerts/{alertId}` → se sincroniza en la web. |

### 1.3 Escala de gravedad utilizada (Nielsen)

| Valor | Significado |
|---|---|
| 0 | No es un problema de usabilidad. |
| 1 | Problema estético; se corrige solo si hay tiempo. |
| 2 | Problema menor; baja prioridad. |
| 3 | Problema mayor; alta prioridad de corrección. |
| 4 | Catástrofe de usabilidad; debe corregirse antes del lanzamiento. |

### 1.4 Tabla de hallazgos

| # | Paso | Hallazgo | Principio de Nielsen | Gravedad (0–4) | Cambio propuesto |
|---|---|---|---|---|---|
| H1 | 2, 7 | El selector de hora no indica en qué zona horaria se está introduciendo la hora. Un usuario de Quito que crea una reunión para un equipo de Madrid no sabe si "09:00" es su hora o la de Madrid, lo que provoca reuniones programadas a la hora equivocada. | **H1 – Visibilidad del estado del sistema** | **4** | Mostrar siempre, junto al campo de hora, la zona activa ("09:00 · Hora de Ecuador, GMT-5") con un selector para cambiarla, y una vista previa en vivo: "Para Madrid serán las 15:00; para São Paulo, las 11:00". |
| H2 | 3 | La recurrencia se presenta con términos técnicos ("RRULE", "COUNT", "BYDAY") o en un menú poco claro, y no se explica qué ocurre con la hora cuando cambia el horario de verano en otro país. | **H2 – Coincidencia entre el sistema y el mundo real** | **3** | Usar lenguaje natural ("Cada martes, 10 veces, hasta el 26 de enero de 2027") y un aviso contextual: "Esta reunión se mantiene a las 09:00 de Ecuador; en Madrid cambiará de 15:00 a 16:00 a partir del 29 de marzo". |
| H3 | 6 | Si dos personas editan el mismo evento a la vez, la API responde `409 Conflict` y la interfaz solo muestra "Error al guardar", perdiendo los cambios escritos por el usuario. | **H9 – Ayudar a reconocer, diagnosticar y recuperarse de errores** | **3** | Traducir el error a lenguaje claro ("Carlos modificó este evento hace 1 minuto"), conservar el borrador y ofrecer opciones: "Ver sus cambios", "Combinar" o "Sobrescribir". Incluir el `traceId` en un enlace de "Copiar detalles para soporte". |
| H4 | 3, 6 | Al editar una ocurrencia de un evento recurrente, el sistema aplica el cambio a toda la serie sin preguntar; tampoco permite deshacer. | **H3 – Control y libertad del usuario** | **3** | Mostrar un diálogo obligatorio "¿Aplicar a: solo este evento / este y los siguientes / todos?" (mapeado al parámetro `scope`) y una barra "Cambio guardado · Deshacer" durante 10 segundos. |
| H5 | 2, 5 | El formulario permite guardar una hora de fin anterior a la de inicio o una alerta "15 minutos antes" en un evento de día completo, y solo se valida tras enviar (respuesta `400`). | **H5 – Prevención de errores** | **2** | Validar en el cliente en tiempo real: ajustar automáticamente la hora de fin al cambiar la de inicio; para eventos de día completo ofrecer alertas relativas a la fecha ("el día anterior a las 18:00"); deshabilitar "Guardar" con un mensaje junto al campo erróneo. |
| H6 | 5, 8 | La configuración de alertas usa términos y ubicaciones distintos en web ("Recordatorio") y móvil ("Notificación"), y una alerta descartada en el móvil sigue apareciendo como pendiente en la web. | **H4 – Consistencia y estándares** | **2** | Unificar la terminología ("Alerta") y los iconos en todas las plataformas mediante un sistema de diseño compartido; sincronizar el estado leído/no leído entre dispositivos usando el evento `AlertAcknowledged`. |
| H7 | 4 | Al añadir asistentes, el usuario debe recordar en qué país está cada persona para elegir una hora razonable; no se muestra su disponibilidad ni su hora local. | **H6 – Reconocer antes que recordar** | **3** | Mostrar junto a cada asistente su hora local y una franja de disponibilidad ("Marie · 15:00 en París · fuera de horario laboral ⚠"), y un "buscador de horario" que sugiera franjas compatibles para todas las zonas. |
| H8 | 1–6 | Los usuarios frecuentes (asistentes de dirección) deben recorrer todo el formulario para crear reuniones similares; no hay plantillas ni atajos. | **H7 – Flexibilidad y eficiencia de uso** | **2** | Añadir plantillas ("Reunión semanal de equipo con alerta 15 min"), duplicar evento, atajos de teclado (`N` nuevo evento, `Ctrl+Enter` guardar) y creación rápida en lenguaje natural ("Reunión con Madrid el martes a las 9"). |
| H9 | 1–5 | El formulario muestra a la vez todos los campos avanzados (visibilidad, webhooks, canales, recurrencia, asistentes), lo que satura la pantalla en móvil. | **H8 – Diseño estético y minimalista** | **1** | Aplicar divulgación progresiva: mostrar título, fecha/hora y zona horaria; agrupar "Repetir", "Alertas" y "Opciones avanzadas" en secciones plegables. |
| H10 | 3, 5 | No hay ayuda contextual que explique cómo funcionan las alertas de un evento recurrente (¿se envía una alerta por ocurrencia?, ¿qué pasa si el móvil está sin conexión?). | **H10 – Ayuda y documentación** | **1** | Añadir íconos de ayuda (ⓘ) con explicaciones breves y un enlace a un artículo del centro de ayuda; mostrar el resumen "Recibirás 10 alertas, una antes de cada reunión" antes de guardar. |

### 1.5 Resumen y priorización

| Gravedad | Hallazgos | Acción |
|---|---|---|
| 4 – Catástrofe | H1 | Corregir antes del lanzamiento (bloqueante). |
| 3 – Mayor | H2, H3, H4, H7 | Incluir en la primera versión. |
| 2 – Menor | H5, H6, H8 | Planificar para la siguiente iteración. |
| 1 – Estético | H9, H10 | Atender si hay capacidad disponible. |

**Conclusión:** los problemas más graves están relacionados con las **zonas horarias** y la **concurrencia**, precisamente las restricciones de datos identificadas en el Paso 4 (R4). Esto confirma que las decisiones arquitectónicas (almacenamiento en UTC con `timezone` IANA, parámetro `displayTimezone`, `ETag`/`409`, parámetro `scope` para recurrencias) deben acompañarse de una interfaz que haga visibles esas decisiones al usuario; de lo contrario, una arquitectura correcta producirá igualmente reuniones a la hora equivocada.

---

## 2. Lista de comprobación antes de subir

| # | Criterio de la guía | Dónde se cumple | Estado |
|---|---|---|---|
| 1 | Los dos diagramas se entienden sin explicación y son coherentes entre sí. | `02_diagramas_arquitectura.md` (§1, §2, tabla de coherencia 1 a 1 en §2.3) | ☐ |
| 2 | Toda flecha tiene nombre que indica qué viaja; ninguna caja queda sin flechas. | `02_diagramas_arquitectura.md` (§1.3) | ☐ |
| 3 | Cada requisito del Paso 1 tiene al menos un componente responsable. | `01_arquitectura_y_restricciones.md` (§3.2) y `02_…` (§1.2) | ☐ |
| 4 | El diagrama de infraestructura marca la parte pública y la interna. | `02_diagramas_arquitectura.md` (§2.2) | ☐ |
| 5 | El estilo elegido tiene justificación con un riesgo nombrado. | `01_arquitectura_y_restricciones.md` (§2.2) | ☐ |
| 6 | Las restricciones tienen una decisión cada una. | `01_arquitectura_y_restricciones.md` (§3.1, R1–R5) | ☐ |
| 7 | El archivo OpenAPI abre sin errores en el editor. | `openapi.yaml` (verificar en editor.swagger.io) | ☐ |
| 8 | El OpenAPI declara seguridad y versión en la ruta, con errores estructurados. | `openapi.yaml` (`bearerAuth`, `/v1/...`, esquema `Error`) | ☐ |
| 9 | La evaluación de UX tiene hallazgos con gravedad y cambios. | Este documento (§1.4) | ☐ |
| 10 | Se declaró qué herramienta de IA se usó, para qué y qué se cambió por criterio propio. | Este documento (§3) | ☐ |
| 11 | Los enlaces (draw.io/IcePanel, SwaggerHub) abren sin pedir cuenta (probar en ventana privada). | Enlaces pegados en `02_…` y en la entrega | ☐ |
| 12 | Cada archivo lleva nombre y apellido del estudiante. | Encabezados de los documentos y nombre de archivo | ☐ |
| 13 | Todo se subió a Brightspace antes del martes 24 de noviembre, 23:59 h. | Brightspace | ☐ |
| 14 | Se leyó la rúbrica y el enunciado completo de Brightspace (que prevalece sobre la guía). | — | ☐ |

---

## 3. Declaración de uso de inteligencia artificial (plantilla)

> Completar con sinceridad. La guía exige declarar **qué herramienta se usó, para qué y qué se cambió por criterio propio**.

### 3.1 Herramientas utilizadas

| Herramienta | Versión / modelo | Fecha de uso | Finalidad |
|---|---|---|---|
| Claude Code (Anthropic) | Claude Opus 5.5 | 23/11/2026 | Generación de un primer borrador de los documentos de arquitectura, restricciones, diagramas en Mermaid, especificación OpenAPI y evaluación heurística de UX. |
| *(Otra herramienta, si aplica)* | | | |

### 3.2 Prompts utilizados

| # | Prompt (texto literal o resumen fiel) | Resultado obtenido |
|---|---|---|
| 1 | "Lee el archivo `Semana09_Guia_Ejercicio_Calendario_ESTUDIANTES.pdf` y ayúdame a resolver de forma completa y profesional toda la entrega de la Semana 9 (ISWZ2202) [...] generar `01_arquitectura_y_restricciones.md`, `02_diagramas_arquitectura.md`, `openapi.yaml` y `03_evaluacion_ux_y_cierre.md` [...]" | Borrador de los cuatro archivos, validación sintáctica del OpenAPI y renderizado de prueba de los diagramas. |
| 2 | *(Añadir aquí cualquier prompt adicional de corrección o ampliación)* | |

### 3.3 Verificaciones realizadas por el estudiante

- [ ] Abrí `openapi.yaml` en editor.swagger.io y comprobé que no hay errores ni advertencias.
- [ ] Importé los diagramas en draw.io / IcePanel y revisé que se entienden sin explicación.
- [ ] Contrasté las decisiones con el enunciado completo de Brightspace y con la rúbrica.
- [ ] Revisé que los supuestos (S1–S9) sean razonables para el caso.
- [ ] *(Otras verificaciones)*

### 3.4 Ajustes realizados por criterio propio

| # | Qué propuso la IA | Qué cambié y por qué |
|---|---|---|
| 1 | *(p. ej., proveedor de nube AWS)* | *(p. ej., lo mantuve / lo cambié a Azure porque…)* |
| 2 | *(p. ej., TTL de caché de 5 min)* | *(justificación propia)* |
| 3 | *(p. ej., gravedad asignada a un hallazgo de UX)* | *(justificación propia)* |
| 4 | | |

### 3.5 Declaración

Declaro que he utilizado las herramientas de IA indicadas como apoyo para la elaboración de esta entrega, que he revisado críticamente todo el contenido generado, que he realizado los ajustes descritos por criterio propio y que asumo la responsabilidad sobre el resultado final entregado.

**Nombre y apellido:** ______________________ **Fecha:** ____ / ____ / ______
