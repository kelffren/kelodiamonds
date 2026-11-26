# KELO ENGINEERING BLUEPRINT
## 0. Control de versión y alcance
Fecha 2026-11-26. Fuente funcional: docs/KELO-BUSINESS-OS-V6.md, blob 06b7c66d5fb19fefd07cf3e816996ddabb7470cb, rama main. Último commit funcional observado 4ce9524bc1958e3f3ba0a2a781d82e6c62d08f2e. El repositorio contiene README.md y docs/; docs contiene solo el maestro antes de esta creación. package.json devuelve 404. No hay aplicación, lockfiles, esquemas ni tests visibles aquí. Ninguna tecnología propuesta se presenta como instalada. No se autoriza implementación ni despliegue. La fuente funcional prevalece; cap.75 y contratos posteriores resuelven precedencia. El capítulo 0 también es requisito, no solo introducción.

## 1. ADR-001 — núcleo modular y auditoría previa
Estado: propuesta condicionada a obtener código real. Mantener una aplicación modular, una identidad y una fuente transaccional; no microservicios iniciales. Alternativas:
| Opción | Costo relativo | Complejidad | Madurez/riesgo | Decisión |
|---|---|---|---|---|
| Reutilizar stack existente | Bajo si compatible | Baja-media | Desconocida hasta auditoría | Preferida |
| React/TypeScript + API Node o Python + PostgreSQL administrado | Medio | Media | Ecosistema maduro; migración potencial | Candidato si no existe equivalente |
| MongoDB existente con transacciones y restricciones equivalentes | Bajo si ya instalado | Media-alta para ledger | Validar atomicidad y topología | No reemplazar automáticamente |
| Microservicios y broker dedicado | Alto | Alta | Fallos distribuidos | Diferir |
PostgreSQL candidato por transacciones, restricciones y bloqueo; documentación consultada versión 19, no implica elegir versión 19 recién publicada. Elegir versión soportada por proveedor y dependencias, fijar versión exacta tras auditoría. Stripe Checkout alojado candidato conforme maestro. Identidad gestionada o existente antes de construir contraseñas propias. Cola DB/outbox inicial; Redis/broker solo con evidencia. No introducir IA para aritmética ni decisiones de permisos.

Gate G0: obtener acceso de lectura al repositorio de aplicación autorizado; inventariar manifests/lockfiles, rutas, auth, DB, migrations, Stripe, Meta, pruebas, entornos, despliegue, dependencias y licencias. Registrar versiones y mapa de reutilización sin copiar secretos. Identificar backup y restauración probada, baseline de tienda y URLs. Si no hay acceso, detener decisiones irreversibles, continuar especificaciones independientes. No cambiar repositorio de trabajo por inferencia.

## 2. Estructura lógica propuesta (adaptar al stack)
- app/public: catálogo, carrito, checkout, propuesta pública.
- app/seller: daily board, workspace, CRM, inbox.
- app/admin: operaciones, finanzas, seguridad.
- domain/identity, commerce, inventory, payments, attribution, ledger, crm, messaging, marketplace, academy.
- infrastructure/db, providers/stripe, providers/meta, jobs, observability.
- contracts: API schemas, event schemas, policy versions.
- tests/unit, integration, e2e, concurrency, fixtures; migrations/ (solo plan).
Dependencias: UI→API→servicio de dominio→repositorio/adaptador; módulos no escriben tablas ajenas directamente. Eventos para proyecciones, no para sustituir transacciones financieras. Un worker puede compartir despliegue lógico sin compartir proceso HTTP.

## 3. Unidad técnica E01 — identidad, autorización y aislamiento
Alcance cap.3,23,45,84 AUTH-01..06 y HEALTH-04; especificación inicial, no implementado.
Datos: users(id UUID, auth_subject UNIQUE, status, created_at); organizations(id,status); memberships(org_id,user_id,status, UNIQUE org/user); role_grants(membership_id,role,scope,valid_from,revoked_at); seller_profiles(org_id,user_id,seller_number UNIQUE por org, eligibility_status); sessions gestionadas por proveedor; consent_records(org_id,subject_id,purpose,policy_version,granted_at,revoked_at); audit_events(id,org_id,actor_id,action,resource_type,resource_id,result,correlation_id,created_at). IDs opacos no son autorización. Seller 000001 asignado mediante bootstrap autorizado y auditado, no contraseña predeterminada ni datos personales en repositorio. Número de vendedor es presentación, no PK.
Estados cuenta pending_verification→active→suspended/closed; elegibilidad seller separate pending→eligible/restricted. Roles customer, seller, specialist, instructor, operations, finance, admin con permisos explícitos; rol no implica acceso a todo. Finance puede preparar lote, aprobador distinto según política; seller solo oportunidades asignadas y clientes autorizados; publicista solo agregados propios; instructor solo alumnos consentidos; admin no acceso indiscriminado a grabaciones. Denegar por defecto y comprobar relación/organización en cada petición. Revocar acceso al cambiar membresía; MFA para privilegios y pagos. Separar opt-out comercial de acceso a soporte.
Contratos internos propuestos (no APIs externas): GET /api/v1/me devuelve roles, permisos y espacios autorizados; POST /api/v1/workspace/select {workspace_id} valida membresía y devuelve contexto; POST /api/v1/sessions/revoke revoca sesión propia; PATCH /api/v1/admin/memberships/:id {role,expected_version,reason} exige permiso y MFA reciente, audita. Registro/verificación/reset mediante proveedor elegido, respuesta genérica para evitar enumeración. Errores 401 sesión inválida, 403 prohibido, 404 recurso ajeno cuando convenga, 409 versión obsoleta, 422 validación, 429 límite; sin PII en error.
Pseudoflujo: autenticar → cargar membresía vigente desde fuente confiable → verificar permiso y relación con recurso → ejecutar transacción → auditar resultado → responder. Nunca confiar en org_id o role enviado por navegador. Para jobs, actor de servicio de mínimo privilegio y contexto de organización explícito. Cookies Secure/HttpOnly/SameSite si sesión web; CSRF para mutaciones con cookies. No tokens en URL o logs.
UI: /auth registro, verificación, recuperación; /seller y /admin protegidos en servidor; selector avatar cambia espacio sin duplicar usuario; loading/error/empty, navegación teclado y móvil. Si sesión expira, conservar solo borrador no sensible y reautenticar. Eventos user_registered, email_verified, session_started, workspace_switched, session_ended; reset_requested sin revelar existencia. Eventos de seguridad no son métricas comerciales de éxito.
Pruebas E01: registro duplicado; token expirado; usuario suspendido con sesión antigua; vendedor A solicita oportunidad B; acceso cruzado por URL y exportación; rol cambiado durante sesión; doble aprobación mismo actor rechazada cuando requerida; revocación de consentimiento; invitado no puede acceder admin; recuperación no enumera usuarios. Aceptación: todas negativas denegadas, ningún dato ajeno en respuesta, audit trail correlacionable, founder opera solo tras autorización real. Estado: especificación inicial; pendiente proveedor, matriz granular por endpoint y ejecución.

## 4. Contrato transversal E02 — eventos y operaciones seguras
Especificación preliminar. event_outbox(id UUID, org_id, type, schema_version, aggregate_id, aggregate_version, occurred_at, correlation_id, causation_id, payload_minimal, available_at, attempts, status); consumer_receipts(consumer,event_id UNIQUE compuesto); idempotency_keys(org_id,actor_id,operation,key,request_hash,result_ref,expires_at UNIQUE compuesto). Escritura de negocio y outbox en una transacción; worker reclama filas con bloqueo y lease, publica/consume al menos una vez; consumidor registra receipt y efecto DB juntos. Efecto externo requiere clave estable del proveedor y conciliación si resultado desconocido. No prometer exactamente una vez de extremo a extremo. Backoff con jitter, límite y cola de revisión; alertar antigüedad y fallos. No guardar chats completos en eventos analíticos. Versionar esquemas de forma aditiva.
Concurrencia: bloqueo de filas en orden consistente para stock/cupo/claim; transacciones serializables donde invariantes abarcan múltiples filas, reintentar transacción completa con límite ante conflicto. Nunca mantener lock mientras se llama API externa. Para misma clave y hash devolver resultado previo; hash distinto 409. Pruebas: caída tras commit antes de ACK, duplicado, orden invertido, worker muerto, lease expirado, proveedor timeout y replay. Aceptación: ningún efecto financiero duplicado en fixtures, evento recuperable, trazabilidad por correlation_id.

## 5. Siguiente unidad E03 — pagos e inventario (borrador de frontera)
Cap.4,23,88 CHECK/OPS. Pedido con snapshot de SKU/precio/impuesto/envío/moneda y atribución; reserva stock separada de reserva cliente y claim. Estado pedido, pago, fulfillment separados. Stripe Checkout creado por servidor, clave idempotente por intento; redirect no confirma pago. Webhook HTTPS verifica firma con cuerpo original y secreto de entorno; receipt único por cuenta/evento, ACK rápido tras persistencia; procesar asincrónicamente, no asumir orden. Confirmar importe, moneda, cuenta y objeto contra pedido; distinguir métodos asíncronos, depósito y pago completo. Reconciliar eventos faltantes con proveedor. No almacenar tarjeta. Siguiente ronda detalla tablas, eventos concretos según API estable fijada, reservas, devoluciones, ledger y tests. No usar API preview solo porque aparece en documentación.

## 6. Fases, costos y gates
F0 auditoría; F1 identidad/eventos; F2 catálogo/pedidos/stock/pagos; F3 atribución/ledger; F4 CRM/inbox/Workspace y piloto Seller Zero; F5 Meta/Acquisition Exchange con gate de cuentas independientes; F6 marketplace/Recovery; F7 llamadas/propuestas; F8 Academy/quality; F9 comunidad/automatización/escala. Reordenar por dependencias y evidencia, no por numeración. Cada gate exige pruebas negativas, migración reversible, observabilidad y flag apagado hasta aprobación. Costos ilustrativos, no cotización: entorno pequeño $30–150/mes infraestructura fija supuesto, excluye Stripe, Meta, IA, multimedia, soporte e impuestos; validar precios y proveedor antes de aprobar. No presupuesto autorizado. Migración expand→backfill idempotente→validar→switch flag→contract solo tras ventana y backup; rollback de aplicación no borra ledger ni pedidos. Entornos local/test/staging/prod separados; credenciales en gestor, datos sintéticos en pruebas. Métricas: errores, p95, edad outbox, conciliación, denegaciones, costo por módulo; sin PII en logs. Baseline real antes de fijar SLO.

## 7. Matriz inicial de trazabilidad
Cobertura de inventario de capítulos, no cobertura de implementación. Estado pendiente hasta especificación detallada por control, contrato y prueba. LOG es histórico, no módulo ejecutable. Las unidades E01/E02 son iniciales; no se declaran capítulos completos. En rondas futuras dividir filas por requisito/control, asignar IDs de pruebas y actualizar estados.
| Capítulo | Requisito / tema | Módulo | Estado | Dependencia | Aceptación |
|---|---|---|---|---|---|
| 0 | Especificación funcional — cómo se usa KELO BUSINESS OS | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 1 | Visión y modelo empresarial | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 2 | Principios y arquitectura | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 3 | Roles y paneles | IDENTITY/WORK | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 4 | Comercio y checkout | COMMERCE | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 5 | Ads Engine y Meta Connect | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 6 | Mini tiendas, afiliados y atribución | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 7 | KELO Chat y agentes IA | CRM/COMM | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 8 | CRM y cualificación | CRM/COMM | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 9 | Closers Network | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 10 | Opportunity Exchange | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 11 | Recovery Engine | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 12 | Seller DNA, Match y Event Engine | QUALITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 13 | Compensation Engine 2.0 | FINANCE | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 14 | Commercial Missions y Earnings Engine | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 15 | Academy y formación | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 16 | Private Club y fidelización | COMMUNITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 17 | Finance y CEO Command Center | FINANCE | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 18 | Operations | OPS/GROWTH | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 19 | Growth Engine | OPS/GROWTH | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 20 | Autopilot y herramientas internas | OPS/GROWTH | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 21 | FTC, privacidad y cumplimiento | SECURITY/SCALE | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 22 | Referentes competitivos y aprendizajes | OPS/GROWTH | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 23 | Modelo de datos e invariantes | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 24 | Implementación en Emergent | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 25 | Pruebas y criterios de aceptación | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 26 | Decisiones abiertas y validación comercial | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 27 | Glosario | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 28 | Outbound Sales Engine y High Ticket Marketplace | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 29 | KELO Phone, Call Intelligence y Live Copilot | CALL/PROPOSAL | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 30 | KELO Proposal Engine y Digital Sales Room | CALL/PROPOSAL | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 31 | Dual Close Engine — filosofía de doble cierre | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 32 | Integración técnica y aceptación de la ampliación | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 33 | Registro de cambios | LOG | Referencia histórica | — | Conservar precedencia |
| 34 | Commercial DNA 2.0 y Talent Discovery | QUALITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 35 | Ronda de mejoras: infraestructura de aprendizaje y rentabilidad | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 36 | Auditoría orientativa de viabilidad | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 37 | Registro de ampliación | LOG | Referencia histórica | — | Conservar precedencia |
| 38 | KELO Groups — colaboración voluntaria por trato | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 39 | Registro de mejora continua | LOG | Referencia histórica | — | Conservar precedencia |
| 40 | KELO Community & Social Engine | COMMUNITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 41 | KELO Legacy, Seller Identity y Digital Tokens | COMMUNITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 42 | KELO Scale & Economic Governance — fórmula maestra | FINANCE | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 43 | KELO Infinity — capacidad, arquitectura y límites | SECURITY/SCALE | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 44 | Registro de continuidad | LOG | Referencia histórica | — | Conservar precedencia |
| 45 | Seller Zero — prioridad de producto y validación diaria | IDENTITY/WORK | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 46 | KELO Seller Workspace, Connect y Window Manager | IDENTITY/WORK | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 47 | WhatsApp First y KELO Smart Calling | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 48 | KELO Direct Call y Live Shopping — dominio oficial | CALL/PROPOSAL | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 49 | Registro de continuidad | LOG | Referencia histórica | — | Conservar precedencia |
| 50 | Recording Center, Multimodal Intelligence y Customer Intelligence | CALL/PROPOSAL | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 51 | Mobile Office — paridad funcional multiplataforma | CALL/PROPOSAL | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 52 | Ghost Feedback — evaluación confidencial posllamada | QUALITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 53 | Performance Firewall y Adaptive Academy | QUALITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 54 | Seller Economics, Rating, Engagement y Capital Allocation | QUALITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 55 | Seller Recovery Engine y Second Chance | QUALITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 56 | Knowledge Economy, Teacher Marketplace y Live Campus | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 57 | Scholarship Program — becas de recuperación | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 58 | Legacy Mentorship — linaje comercial voluntario | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 59 | Integración funcional y criterios transversales de la ampliación | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 60 | Registro de continuidad | LOG | Referencia histórica | — | Conservar precedencia |
| 61 | Mapa maestro de arquitectura y construcción | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 62 | Roadmap económico — construcción autofinanciada | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 63 | Economía de operación, escalabilidad y financiación | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 64 | Mapa organizativo y principios de ejecución | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 65 | Registro de continuidad | LOG | Referencia histórica | — | Conservar precedencia |
| 66 | KELO Competitive Intelligence — registro permanente de competidores | OPS/GROWTH | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 67 | Backlog competitivo ejecutable y criterios de aceptación | OPS/GROWTH | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 68 | Protocolo de mejora competitiva continua | OPS/GROWTH | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 69 | KELO Open Academy — universidad y empresa en la palma de la mano | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 70 | Learn → Earn — marketplace comercial flexible | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 71 | KELO Practice y Mastery Engine — aprendizaje interactivo universal | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 72 | Spaced Repetition, Adaptive Learning y verificación de dominio | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 73 | Sales Gym y English Conversation Simulator | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 74 | Arquitectura educativa, costos y aceptación | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 75 | Consolidación de decisiones y resolución de ambigüedades | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 76 | Contratos funcionales comerciales — detalles recuperados | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 77 | Capital, Ghost Feedback y recuperación — reglas operativas detalladas | QUALITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 78 | Academia — currículo, pantallas y sesiones concretas | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 79 | Profesores, becas, mentoría y legado — detalle recuperado | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 80 | Economía detallada y escenarios de capacidad | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 81 | Matriz de entrega y pruebas integradas | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 82 | Registro de auditoría de continuidad | LOG | Referencia histórica | — | Conservar precedencia |
| 83 | Contrato de interacción y ciclo comercial continuo | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 84 | Identidad, navegación y oficina | IDENTITY/WORK | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 85 | Publicidad, captación y comunicaciones | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 86 | CRM, llamadas y propuestas | CRM/COMM | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 87 | Marketplaces, atribución y ventas colaborativas | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 88 | Comercio, fidelización y operaciones | COMMERCE | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 89 | Inteligencia, calidad y capital | QUALITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 90 | Universidad, profesores y mentoría | ACADEMY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 91 | Comunidad, reputación y remuneración | COMMUNITY | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 92 | Dirección, crecimiento y control técnico | FINANCE | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 93 | Orquestación del ciclo infinito de ventas — reglas y medición | MARKET | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 94 | Aceptación y registro de cobertura | CORE | Pendiente de detalle | Auditoría | Prueba específica del requisito pendiente |
| 95 | Acquisition Exchange — propósito y referencias | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 96 | Implementación paso a paso — prueba de integración antes de marketplace | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 97 | Contratos, modelo de datos y estados | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 98 | Quality Shield, atribución y remuneración | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 99 | Pantallas, botones y eventos | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 100 | Economía, pujas y ciclo de retroalimentación | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 101 | Piloto, pruebas y aceptación | ACQUISITION | Pendiente de detalle | CORE | Prueba específica del requisito pendiente |
| 102 | Registro de continuidad | LOG | Referencia histórica | — | Conservar precedencia |

## 8. Fuentes primarias consultadas 2026-11-26
- PostgreSQL, documentación actual identificada como v19, aislamiento y concurrencia: https://www.postgresql.org/docs/current/transaction-iso.html . Fundamenta locks/serialización y reintentos; versión de despliegue pendiente.
- Stripe, webhooks: https://docs.stripe.com/webhooks . Fundamenta firma, entrega asíncrona y pruebas; API/SDK estable exacta pendiente de auditoría.
- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html . Documento vivo, sin versión fija observada; fundamenta mínimo privilegio y autorización por recurso.
No se verificaron permisos Meta ni cuentas reales; no se afirma viabilidad de Acquisition Exchange.

## 9. Registro
2026-11-26 — Creación del blueprint independiente. Inventario de capítulos 0–102, ADR condicionado, estructura lógica, E01 identidad y E02 eventos, frontera E03. Evidencia: maestro blob indicado, historial reciente, raíz y docs sin código de aplicación; fuentes oficiales arriba. Criterios de aceptación documentados, no ejecutados. Próxima prioridad: E03 checkout/stock/Stripe con ledger y conciliación posterior. No se modificó maestro ni aplicación.
