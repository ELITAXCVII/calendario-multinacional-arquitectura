# ISWZ2202 – Diseño y Arquitectura de Software · Semana 9

## Servicio de Calendario Corporativo para una Empresa Multinacional

**Documento 01 – Requisitos, estilos arquitectónicos y restricciones**

| Campo | Valor |
|---|---|
| Estudiante | *(Nombre y Apellido)* |
| Asignatura | ISWZ2202 – Diseño y Arquitectura de Software |
| Entrega | Martes 24 de noviembre, 23:59 h (Brightspace) |
| Partes cubiertas | a) Arquitectura (estilos) · b) Restricciones |
| Documentos relacionados | `02_diagramas_arquitectura.md`, `openapi.yaml`, `03_evaluacion_ux_y_cierre.md` |

---

## 1. Paso 1 – Conversión del enunciado en requisitos

### 1.1 Tabla de análisis de requisitos

| Pregunta de la guía | Respuesta (requisito derivado) | Impacto en el diseño | Identificador |
|---|---|---|---|
| **¿Quiénes usan el servicio y desde qué plataformas?** | (1) Empleados de la multinacional, que gestionan sus calendarios y eventos; (2) asistentes y responsables de equipo, que administran calendarios compartidos; (3) administradores de TI, que gestionan usuarios, roles y políticas; (4) sistemas externos (ERP, RR. HH., salas de reuniones, clientes de correo) que se integran por API. Plataformas: **aplicación web** (navegadores modernos), **aplicación móvil** nativa (iOS y Android) y **servicios externos** vía API REST. | Se requiere un contrato de API único y versionado consumido por varios clientes heterogéneos (estilo cliente-servidor) y un punto de entrada común que aplique políticas homogéneas (API Gateway). | RQ-01 |
| **¿Quién se identifica y con qué datos? ¿Qué se verifica después?** | **Autenticación:** el usuario se identifica con su correo corporativo y contraseña, preferentemente federado mediante SSO corporativo (OpenID Connect / SAML con el proveedor de identidad de la empresa) y con segundo factor (MFA) obligatorio para roles administrativos. Los sistemas externos se autentican con *client credentials* de OAuth 2.0. Tras autenticarse, se emite un **JWT de acceso de vida corta (15 min)** firmado (RS256) y un *refresh token* rotativo. **Autorización (RBAC):** en cada petición se verifica la firma, expiración y audiencia del token (en el API Gateway) y, en cada microservicio, el **rol** y el **alcance** sobre el recurso: `ADMIN` (gestión global), `OWNER` (propietario del calendario), `EDITOR` (crea/edita eventos), `VIEWER` (solo lectura), `INTEGRATION` (cuenta de servicio con *scopes* limitados). También se verifica la **pertenencia al inquilino/filial** (*tenant*) para impedir accesos cruzados entre países. | Microservicio dedicado de **Autenticación e IAM**; validación en dos niveles (gateway + servicio) siguiendo el principio de defensa en profundidad; registro de auditoría de accesos. | RQ-02 |
| **¿Qué información es sensible y cómo debe guardarse?** | Son sensibles: título, descripción, ubicación y notas de los eventos (pueden revelar fusiones, reuniones con clientes, temas de RR. HH. o salud); la lista de asistentes y sus correos (datos personales sujetos a GDPR, LOPDP de Ecuador y normativas locales); los tokens de dispositivos para notificaciones push; las credenciales. **Almacenamiento:** cifrado en reposo a nivel de campo con **cifrado de sobre** (*envelope encryption*): cada calendario tiene una llave de datos (DEK, AES-256-GCM) que, a su vez, está cifrada por una llave maestra (KEK) custodiada en **AWS KMS / HashiCorp Vault**; los datos nunca se guardan en claro. Cifrado en tránsito con **TLS 1.3** externamente y **mTLS** entre microservicios. Las contraseñas no se almacenan en el sistema (SSO) o, en su defecto, con hash Argon2id. | Microservicio de **Cifrado (KMS)** como única pieza que manipula llaves; base de datos con cifrado de disco adicional; minimización de datos en los registros (*logs*) para no exponer información sensible. | RQ-03 |
| **¿Qué eventos generan alertas y dónde deben verse?** | Generan alertas: (a) recordatorios programados antes del inicio de un evento (p. ej., 10 min, 1 h, 1 día), incluidos los de **eventos recurrentes**; (b) invitación recibida; (c) modificación o cancelación de un evento en el que se participa; (d) respuesta de un asistente (acepta/rechaza); (e) conflicto de horario detectado; (f) calendario compartido con el usuario. Deben verse **en todos los dispositivos del usuario**: notificación push en el móvil (APNs / FCM), notificación en el navegador (Web Push y WebSocket en tiempo real), correo electrónico y, opcionalmente, *webhooks* hacia sistemas externos. Una alerta descartada en un dispositivo debe marcarse como leída en los demás. | **Motor de Alertas y Notificaciones** desacoplado mediante **publicación/suscripción**: el servicio de calendario publica eventos de dominio y el motor decide canal y momento. Necesidad de idempotencia para no enviar duplicados. | RQ-04 |
| **¿Qué significa multinacional para el diseño (zonas horarias, idiomas, regiones, volumen)?** | **Zonas horarias:** todas las fechas se almacenan en **UTC** y se intercambian en formato **ISO-8601** con desplazamiento (`2026-11-24T15:00:00Z`); cada evento guarda además la zona horaria IANA de origen (`America/Guayaquil`, `Europe/Madrid`) para expandir correctamente las recurrencias ante cambios de horario de verano (DST). La conversión a la hora local se hace en el cliente según la preferencia del usuario. **Idiomas:** interfaz y plantillas de notificación internacionalizadas (i18n) con al menos español, inglés y portugués; formatos de fecha según *locale*. **Regiones:** despliegue **multi-región** (p. ej., `us-east-1` para América, `eu-west-1` para Europa) con **residencia de datos** por filial cuando la ley lo exija (GDPR). **Volumen:** se asumen ~50 000 empleados, ~2 millones de eventos activos y picos de alertas a las horas en punto (p. ej., las 09:00 locales de cada zona). | Datos con `timezone` explícito; base de datos multi-AZ replicada entre regiones; enrutamiento por latencia (DNS geográfico); escalado horizontal del motor de alertas en los picos. | RQ-05 |
| **¿Qué no dice el enunciado y debo decidir yo? (supuestos)** | S1. La empresa dispone de un proveedor de identidad corporativo (Azure AD / Okta) compatible con OIDC. S2. El proveedor de nube es AWS (con alternativa Vault si se requiere independencia de proveedor). S3. Objetivo de disponibilidad del 99,9 % mensual; RPO ≤ 5 min y RTO ≤ 15 min. S4. Latencia objetivo de lectura de calendario p95 < 300 ms. S5. Las alertas toleran un retraso máximo de 60 s respecto a la hora programada. S6. Las recurrencias se modelan con la norma **RFC 5545 (RRULE)** para interoperar con iCalendar. S7. Se conserva un registro de auditoría de 1 año. S8. La aplicación móvil puede funcionar sin conexión en modo lectura y sincroniza al reconectarse. S9. La entrega de notificaciones a terceros (APNs, FCM, SMTP) es *at-least-once*, por lo que el consumidor debe ser idempotente. | Los supuestos fijan los objetivos medibles con los que se toman las decisiones del Paso 4 y deben validarse con el cliente. | RQ-06 |

### 1.2 Requisitos de calidad derivados (resumen)

| Atributo | Escenario medible |
|---|---|
| Seguridad | El 100 % de los datos sensibles se almacena cifrado; ninguna llave maestra sale del KMS; rotación automática anual de KEK y re-cifrado de DEK bajo demanda. |
| Rendimiento | p95 < 300 ms en `GET /v1/calendars/{calendarId}/events` para un rango de 1 semana. |
| Disponibilidad | 99,9 % mensual; la caída de una región completa no detiene el servicio (conmutación en < 15 min). |
| Escalabilidad | El motor de alertas procesa 10 000 notificaciones/minuto en pico sin degradar la API de calendario. |
| Interoperabilidad | Exportación/importación iCalendar (RFC 5545) y API REST documentada en OpenAPI 3.0.3. |
| Mantenibilidad | Cada microservicio se despliega de forma independiente sin interrupción del servicio. |

---

## 2. Paso 2 – Elección de la mezcla de estilos arquitectónicos

### 2.1 Visión general

Se adopta una **mezcla de tres estilos principales**, apoyada en dos estilos de base presentes de forma implícita:

```
Clientes (Web, Móvil, Externos)
        │  HTTPS / REST  ← estilo Cliente-servidor
        ▼
   [ API Gateway ]  ← Puerta de enlace de API
        │  REST interno (mTLS)
        ▼
 ┌──────────────┬──────────────────┬───────────────┬──────────────────────┐
 │ Auth / IAM   │ Gestión de       │ Cifrado / KMS │ Motor de Alertas y   │ ← Microservicios
 │              │ Calendarios      │               │ Notificaciones       │   (cada uno en Capas)
 └──────────────┴────────┬─────────┴───────────────┴──────────▲───────────┘
                         │ publica eventos de dominio          │ consume
                         ▼                                     │
                 [ Message Broker Pub/Sub ] ───────────────────┘  ← Dirigida por eventos
```

| Parte del servicio | Estilo que la organiza |
|---|---|
| Relación entre clientes y backend | Cliente-servidor (base) |
| Entrada, seguridad perimetral y enrutamiento | **Puerta de enlace de API (API Gateway)** |
| Descomposición del backend por capacidades de negocio | **Microservicios** |
| Interior de cada microservicio (controlador → dominio → repositorio) | Capas (base) |
| Generación y distribución de alertas y sincronización entre servicios | **Dirigida por eventos (publicación/suscripción)** |

### 2.2 Justificación por estilo (qué aporta, qué riesgo real conlleva y por qué compensa)

#### A) Puerta de enlace de API (API Gateway)

| Dimensión | Análisis |
|---|---|
| **Qué aporta** | Punto de entrada único para Web, Móvil y servicios externos. Centraliza la validación del JWT, la limitación de tasa (*rate limiting*) por cliente y por inquilino, el enrutamiento por versión (`/v1`, `/v2`), la terminación TLS, la correlación de trazas (`traceId`) y el registro de acceso. Los microservicios quedan ocultos en la red privada. |
| **Qué riesgo real conlleva** | **Punto único de falla y de congestión**: si el gateway cae o se satura, todo el servicio queda inaccesible aunque los microservicios estén sanos. Además, existe el riesgo de convertirlo en un "monolito de lógica" si se le añaden reglas de negocio, y añade un salto de red (~5–15 ms) a cada petición. |
| **Por qué compensa** | Sin gateway, cada microservicio tendría que reimplementar autenticación, límites y CORS, multiplicando la superficie de ataque, algo inaceptable con datos sensibles. El riesgo se mitiga desplegando el gateway como servicio gestionado **sin estado**, con varias réplicas en al menos 3 zonas de disponibilidad por región, detrás de un balanceador y con escalado automático; la regla de diseño es "el gateway solo hace preocupaciones transversales, nunca lógica de negocio". |

#### B) Microservicios

| Dimensión | Análisis |
|---|---|
| **Qué aporta** | Despliegue y escalado independientes por capacidad: el **Motor de Alertas** tiene picos a las horas en punto y debe escalar sin escalar la gestión de calendarios; el **Servicio de Cifrado** concentra el manejo de llaves en un perímetro pequeño y auditable; **Auth/IAM** evoluciona con las políticas corporativas. Cada equipo puede desplegar sin coordinar a toda la organización y aislar fallos (un fallo en notificaciones no impide crear eventos). |
| **Qué riesgo real conlleva** | **Complejidad operativa y consistencia distribuida**: más piezas que desplegar, monitorear y versionar; las llamadas en red pueden fallar o encadenar latencias (p. ej., Calendario → Cifrado en cada lectura); no existen transacciones ACID entre servicios, de modo que crear un evento y programar su alerta no es atómico. |
| **Por qué compensa** | Los requisitos (multi-región, picos de alertas, aislamiento de llaves, zero-downtime) exigen escalado y despliegue independientes que un monolito no ofrece con facilidad. Se acota el riesgo con una **granularidad gruesa** (solo 4 servicios, no decenas), base de datos propia por servicio, **patrón Outbox transaccional** para garantizar que todo cambio confirmado en la BD se publica como evento, *circuit breakers* y *timeouts* en llamadas síncronas, caché de llaves descifradas (ver Paso 4) y observabilidad centralizada (OpenTelemetry). |

#### C) Arquitectura dirigida por eventos (publicación/suscripción)

| Dimensión | Análisis |
|---|---|
| **Qué aporta** | El servicio de calendario publica eventos de dominio (`EventCreated`, `EventUpdated`, `EventCancelled`, `InvitationSent`, `AlertScheduled`) sin conocer a sus consumidores. El Motor de Alertas, la auditoría o futuras integraciones (analítica, sincronización con correo) se suscriben sin modificar al productor. Permite **absorber picos** (el broker actúa como amortiguador) y entregar alertas a múltiples dispositivos de forma asíncrona. |
| **Qué riesgo real conlleva** | **Orden y duplicados de eventos, y depuración más difícil**: un `EventUpdated` puede llegar antes que su `EventCreated`, o un mensaje puede entregarse dos veces (semántica *at-least-once*), lo que produciría alertas duplicadas o con hora incorrecta. La consistencia es eventual: durante unos segundos un dispositivo puede mostrar datos desactualizados. El broker se convierte también en infraestructura crítica. |
| **Por qué compensa** | El envío síncrono de notificaciones a APNs, FCM y SMTP dentro de la petición de creación haría la API lenta y frágil ante caídas de terceros. El riesgo se controla con: **clave de partición por `calendarId`** (garantiza orden por calendario), **versión monotónica** en cada evento y **consumidores idempotentes** (tabla de `eventId` procesados), **cola de mensajes muertos (DLQ)** con reintentos exponenciales, y trazas distribuidas con `traceId` propagado en las cabeceras del mensaje. La consistencia eventual es aceptable para alertas (tolerancia de 60 s, supuesto S5). |

### 2.3 Estilos considerados de forma implícita

- **Cliente-servidor:** base de la relación entre los tres tipos de cliente y el backend. Su riesgo (servidor como punto central de carga) se resuelve con la redundancia multi-región descrita en el Paso 4.
- **Capas:** cada microservicio se estructura internamente en *API/Controlador → Dominio → Repositorio/Infraestructura*. Su riesgo (latencia y código de paso) se limita a tres capas por servicio.

### 2.4 Alternativa descartada

Un **monolito modular en capas** sería más simple de operar y suficiente para una empresa pequeña, pero se descarta porque no permite escalar de forma independiente el motor de alertas en los picos horarios globales ni aislar la gestión de llaves en un perímetro mínimo auditable, ambos requisitos críticos (RQ-03, RQ-04, RQ-05).

---

## 3. Paso 4 – Restricciones y decisiones técnicas

### 3.1 Tabla de restricciones

| # | Categoría | Pregunta guía | Restricción identificada | Decisión técnica concreta | Riesgo residual / verificación |
|---|---|---|---|---|---|
| R1 | **Seguridad y privacidad** | ¿Qué datos protege la ley o la empresa? ¿Quién custodia las claves y cada cuánto se renuevan? | Los eventos y asistentes contienen datos personales y confidenciales protegidos por GDPR, la LOPDP (Ecuador) y las políticas internas; un acceso indebido o una fuga de llaves expondría información estratégica. | 1) **Cifrado de sobre**: DEK por calendario (AES-256-GCM) cifrada con una KEK por región/inquilino alojada en **AWS KMS (HSM FIPS 140-2 nivel 3)** o **HashiCorp Vault Transit**. 2) **Custodia**: solo el *Servicio de Cifrado* tiene permiso IAM `kms:Decrypt`; los demás servicios jamás ven la KEK. Separación de funciones: el equipo de seguridad administra las llaves; los desarrolladores no tienen acceso. 3) **Rotación**: KEK con rotación automática **cada 365 días** (y rotación inmediata ante incidente); DEK rotadas cada 90 días con re-cifrado perezoso (*lazy re-encryption*) al escribir. 4) RBAC + verificación de *tenant* en cada servicio; JWT de 15 min; TLS 1.3 y mTLS internos (malla de servicios). 5) Auditoría inmutable de cada uso de llave (AWS CloudTrail / Vault audit log). 6) Derecho al olvido: borrado criptográfico (*crypto-shredding*) destruyendo la DEK del usuario. | Verificación con pruebas de penetración semestrales y alarmas ante usos anómalos de `Decrypt`. |
| R2 | **Rendimiento** | ¿Qué operación añade tiempo (p. ej., descifrar) y cómo se mitiga? | Descifrar cada evento implica una llamada al KMS (~10–30 ms por llamada de red) y operaciones criptográficas; leer una semana con 50 eventos podría superar el objetivo p95 < 300 ms. Además, el KMS tiene límites de cuota de peticiones por segundo. | 1) **Caché en memoria de DEK descifradas con TTL corto (5 min)** dentro del Servicio de Cifrado, almacenada en **Redis cifrado (TLS + cifrado en reposo)** de la red privada, más una caché L1 en proceso de 60 s; así se llama al KMS una vez por calendario cada 5 min, no por evento. 2) Descifrado de campos en lote (*batch*) por petición. 3) Uso de AES-GCM con aceleración por hardware (AES-NI). 4) Paginación obligatoria y filtros por rango de fechas en la API (`from`, `to`, `limit`). 5) Caché de lectura de vistas de calendario (eventos ya cifrados) invalidada por los eventos `EventUpdated`. 6) Las notificaciones se envían de forma asíncrona (no bloquean la petición). | Compromiso consciente: una DEK permanece en memoria hasta 5 min; se acepta porque la memoria está aislada en la red privada, Redis está cifrado y el TTL limita la ventana de exposición. Se mide con métricas de *cache hit ratio* (> 95 %) y latencia p95. |
| R3 | **Disponibilidad** | ¿Qué pasa si cae una región o un servicio? | La empresa opera en varios continentes las 24 horas; la caída de una región no puede dejar sin calendario a todos los empleados. Objetivo 99,9 %, RTO ≤ 15 min, RPO ≤ 5 min. | 1) **Despliegue multi-región activo-pasivo con lectura activa**: región primaria por inquilino (residencia de datos) y región secundaria con réplica asíncrona de la base de datos (p. ej., Aurora Global Database, retraso < 1 s). 2) **Multi-AZ** dentro de cada región (3 zonas) para Kubernetes, Redis y BD. 3) DNS con verificación de salud y conmutación por error (Route 53 *failover/latency routing*). 4) **Llaves KMS multi-región** (réplica de KEK) para poder descifrar tras la conmutación. 5) Si cae un servicio: *circuit breaker* en el gateway; si cae el Motor de Alertas, los eventos se acumulan en el broker y se procesan al recuperarse (no se pierden); si cae el broker, el **Outbox** retiene los eventos en la BD. 6) Degradación elegante: si el servicio de cifrado no responde, se sirven datos desde la caché o se devuelve `503` con `Retry-After`, nunca datos en claro. | Simulacros trimestrales de conmutación de región (*game days*) e ingeniería del caos controlada. |
| R4 | **Datos (concurrencia y zonas horarias)** | ¿Cómo se manejan zonas horarias, eventos repetidos y cambios simultáneos? | Usuarios de distintas zonas editan el mismo evento; las recurrencias cruzan cambios de horario de verano; dos asistentes pueden modificar un evento al mismo tiempo. | 1) **Zonas horarias**: almacenamiento en **UTC**, intercambio en **ISO-8601** (`2026-11-24T15:00:00Z`), y campo `timezone` IANA en cada evento; la conversión a hora local la realiza el cliente; la base de datos de zonas (tzdata) se actualiza automáticamente. 2) **Eventos repetidos**: se guarda la regla **RRULE (RFC 5545)** y no cada ocurrencia; las ocurrencias se expanden en la zona horaria de origen (así una reunión semanal a las 09:00 de Madrid sigue a las 09:00 tras el cambio de horario); las excepciones se guardan como `EXDATE` o instancias modificadas (`recurrenceId`). 3) **Cambios simultáneos**: **control de concurrencia optimista** con número de versión y cabeceras `ETag` / `If-Match`; si la versión no coincide, la API responde **409 Conflict** con la versión actual para que el cliente resuelva. 4) **Idempotencia**: cabecera `Idempotency-Key` en las creaciones para evitar eventos duplicados por reintentos del móvil. | Pruebas automatizadas con fechas límite (cambio de DST, 29 de febrero, zonas con desplazamiento de 30/45 min como `Asia/Kolkata`). |
| R5 | **Operación (CI/CD sin interrupción)** | ¿Cómo se despliega, se monitorea y se actualiza sin cortar el servicio? | Al existir usuarios en todas las zonas horarias no hay "ventana de mantenimiento" nocturna; cada actualización debe ser *zero-downtime*. | 1) **Pipeline CI/CD** (GitHub Actions / GitLab CI + Argo CD con GitOps): compilación, pruebas unitarias y de contrato contra `openapi.yaml`, análisis estático (SAST), escaneo de dependencias e imágenes, y firma de imágenes. 2) **Despliegue progresivo**: *rolling update* en Kubernetes con *readiness/liveness probes* y **canary** (5 % → 25 % → 100 %) con reversión automática si suben los errores 5xx o la latencia. 3) **Migraciones de BD compatibles hacia atrás** (patrón *expand-and-contract*). 4) **Versionado de API en ruta** (`/v1`); los cambios incompatibles van a `/v2` con período de convivencia y aviso de obsolescencia (`Deprecation`, `Sunset`). 5) **Observabilidad**: métricas (Prometheus/Grafana), registros centralizados sin datos sensibles, trazas distribuidas (OpenTelemetry) con `traceId`, alertas operativas basadas en SLO. 6) Infraestructura como código (Terraform) y gestión de secretos en Vault (nunca en el repositorio). | Indicadores DORA (frecuencia de despliegue, tasa de fallos de cambio, MTTR) revisados mensualmente. |

### 3.2 Trazabilidad requisito → componente → restricción

| Requisito | Componente responsable | Restricciones relacionadas |
|---|---|---|
| RQ-01 Plataformas | Clientes Web/Móvil/Externos, CDN, API Gateway | R3, R5 |
| RQ-02 Autenticación y RBAC | API Gateway, Servicio Auth/IAM, BD IAM | R1 |
| RQ-03 Datos sensibles | Servicio de Cifrado/KMS, AWS KMS / Vault, Redis cifrado | R1, R2 |
| RQ-04 Alertas multi-dispositivo | Message Broker Pub/Sub, Motor de Alertas, proveedores APNs/FCM/SMTP | R3, R4 |
| RQ-05 Multinacional | Gestión de Calendarios (UTC/RRULE), despliegue multi-región | R3, R4 |
| RQ-06 Supuestos | Todos (objetivos medibles) | R1–R5 |
