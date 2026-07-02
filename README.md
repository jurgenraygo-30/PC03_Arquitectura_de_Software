# Evaluación PC03 - Arquitectura de Software
**Nombre de Alumno:** Jürgen Rodolfo Raymundo Gonzales

**Código de Alumno:** U22234261

---

## 1. Declaración sobre la Ejecución del Entorno Local

En cumplimiento con los lineamientos de la evaluación, declaro que no fue posible ejecutar el ecosistema completo mediante Docker Compose para la recolección de capturas de ejecución. A continuación, detallo el análisis del incidente:

**A. Qué falló:**
No fue posible aprovisionar el entorno local de desarrollo ni levantar la red de contenedores (`docker compose up --build`). La estación de trabajo carecía de los prerrequisitos base (Docker Desktop, Node.js 18+, y clientes de PostgreSQL), y el intento de despliegue falló durante la fase de descarga y construcción.

**B. Por qué falló (Análisis Técnico):**
Se presentaron dos bloqueantes críticos de infraestructura:
* **Degradación de red (Timeouts):** Existió una limitación severa de ancho de banda en la red local que impidió la descarga exitosa de los instaladores y de las imágenes de los contenedores requeridas por el archivo `docker-compose.yml`. La baja velocidad generó cortes por *timeout* en el *pulling* de las imágenes.
* **Falla de Virtualización:** Durante los intentos de configuración de la capa base de Docker, el host local arrojó un error de soporte de virtualización (*"Virtualization support not detected"*). Esto indica un conflicto en el subsistema de Windows (WSL 2 / Hypervisor) que impide abstraer el hardware.
<img width="1586" height="900" alt="Captura de pantalla 2026-07-01 185838" src="https://github.com/user-attachments/assets/1d44dbeb-a198-4fc9-ab28-b0dbeef58bd7" />


**C. Qué evidencia alternativa se usó:**
Para fundamentar y diseñar el rediseño propuesto en este RFC, me apoyé estrictamente en el análisis estático de los artefactos documentales y de código provistos en el repositorio:
* **Contratos API:** Análisis de los archivos `ms-*/docs/swagger.yaml` para entender las firmas HTTP síncronas y los límites entre servicios.
* **Asincronía y Mensajería:** Revisión del documento `docs/event-contracts.md` para comprender los *payloads*, el enrutamiento y el comportamiento de RabbitMQ.
* **Modelo de datos:** Revisión de los scripts SQL en `docs/setup-and-usage.md` para entender el esquema de bloqueos de `ms-agenda` y el estado de la tabla de tutorías.

**D. Procedimiento para validarlo (cuando el entorno esté disponible):**
1. Descargar las herramientas base y levantar la infraestructura con `docker compose up --build`.
2. Ejecutar los *scripts* de inserción de datos (*seed data*) en los puertos locales de las bases de datos (5432, 5433, 5434).
3. Obtener un token JWT y emitir una solicitud de reserva pagada desde el simulador del cliente web (`localhost:8080`).
4. Monitorear el panel de RabbitMQ (`localhost:15672`) para observar el tránsito del evento de pago y verificar en el Dashboard de Tracking (`localhost:9000`) la trazabilidad a través del `X-Correlation-ID`.

---

## 2. Mini RFC: Arquitectura para Pagos de Tutorías

### 2.1. Contexto
El sistema "Tutorías Universitarias" gestiona la reserva de sesiones integrando autenticación (`ms-auth`), validación (`ms-usuarios`), bloqueo de horarios (`ms-agenda`) y notificaciones asíncronas vía RabbitMQ (`ms-notificaciones`). El nuevo requerimiento exige incorporar un servicio `ms-pagos` que interactúe con proveedores externos (Visa/Mastercard). Estos proveedores introducen variables incontrolables en la red: alta latencia, respuestas tardías, caídas temporales y *callbacks* duplicados, imponiendo la necesidad de un flujo tolerante a fallos parciales y asincronía.

### 2.2. Problema
El diseño propuesto (abrir una transacción de base de datos en `ms-tutorias`, encadenar llamadas HTTP síncronas hacia `ms-pagos` y luego al proveedor externo) es un antipatrón crítico en sistemas distribuidos:
* **Acoplamiento temporal y transacciones largas:** Mantener una transacción en PostgreSQL abierta mientras se espera I/O de red de un tercero agota los hilos y el *connection pool*, degradando el sistema ante carga concurrente.
* **Estado ambiguo por Timeout:** Un *timeout* HTTP (ej. el gateway corta a los 30s) no garantiza que el proveedor haya cancelado el cobro. Si `ms-tutorias` hace *rollback* ciego ante un *timeout*, la tutoría se pierde internamente, pero el estudiante igual recibe el cargo en su tarjeta.
* **Bloqueo de recursos:** `ms-agenda` retiene horarios bloqueados basados en una transacción inestable, reduciendo la disponibilidad real de los tutores de forma injustificada.

### 2.3. Riesgos
* **Riesgos de Negocio:** Cobros duplicados (*double billing*) si el cliente refresca la pantalla o reintenta tras un error; tutorías cobradas pero no registradas; quejas formales por fraude o inconsistencia.
* **Riesgos Técnicos:** Efecto cascada por degradación. Si el proveedor externo responde lento, `ms-pagos` se encola, `ms-tutorias` colapsa, y todo el flujo de reservas se cae.
* **Riesgos de Seguridad:** Exposición accidental de datos sensibles (PCI) si el tráfico de tarjetas se enruta a través de servicios no autorizados como `ms-tutorias`.

### 2.4. Diseño propuesto
Se propone una arquitectura de **Saga Orquestada**, donde `ms-tutorias` actúa como coordinador central, combinando interacciones síncronas para la intención y asíncronas para la confirmación:
1. **Síncrono (Creación):** El cliente solicita la tutoría. `ms-tutorias` valida datos, bloquea `ms-agenda` y guarda la tutoría como `PENDIENTE_PAGO`. Llama a `ms-pagos` para generar una "Intención de Pago". `ms-pagos` devuelve un ID/URL de pago. `ms-tutorias` responde al cliente, cerrando la conexión HTTP.
2. **Asíncrono (Interacción):** El cliente interactúa con la pasarela. El proveedor procesa y envía un *Webhook* asíncrono a `ms-pagos`.
3. **Asíncrono (Eventos):** `ms-pagos` recibe el webhook, actualiza su estado y publica el evento `pago.procesado` en RabbitMQ.
4. **Síncrono (Resolución interna):** `ms-tutorias` consume el evento, cambia la tutoría a `CONFIRMADA` y publica el evento para que `ms-notificaciones` avise al usuario.

### 2.5. Alternativas consideradas
* **Alternativa 1: Orquestación Síncrona HTTP (Diseño deficiente propuesto).** Descartada. Obliga a que la disponibilidad de nuestro sistema dependa 100% de la latencia y disponibilidad de Visa/Mastercard.
* **Alternativa 2: Coreografía basada en Eventos (Sin orquestador).** Descartada. Aunque es altamente desacoplada, obligaría a `ms-agenda` a escuchar eventos de `ms-pagos` para saber si debe liberar un horario. Al haber un flujo de negocio claro (la reserva), es mejor que un orquestador (`ms-tutorias`) controle las transacciones compensatorias para facilitar el diagnóstico.
* **Alternativa 3: Saga Orquestada (Elegida).** Permite respuestas rápidas al cliente (UX), aísla el riesgo de red del proveedor en `ms-pagos` y asegura la consistencia eventual de la agenda.

### 2.6. Contratos
* **API Síncrona (`ms-pagos`):** `POST /api/v1/payments/intents`.
    * *Headers obligatorios:* `Idempotency-Key` y `X-Correlation-ID`.
    * *Errores modelados:* `400 Bad Request`, `409 Conflict` (intención ya existe), `502 Bad Gateway` (falla de red con proveedor).
* **Evento Asíncrono (RabbitMQ):** Exchange `tutorias.topic`, Routing Key `pago.procesado.v1`.
    * *Payload mínimo:* `tutoria_id`, `intent_id`, `status` (APPROVED/REJECTED).
* **Evolución y compatibilidad:** Se versionan las URLs (`/v1/`) y las *routing keys*. Todo campo nuevo en los contratos debe ser opcional para no romper a los consumidores existentes.

### 2.7. Manejo de fallas
* **Idempotencia y Deduplicación:** El cliente envía un `Idempotency-Key` a `ms-tutorias` y este a `ms-pagos`. Si el cliente hace "doble clic", la API devuelve el mismo `intent_id` sin duplicar cobros.
* **Consistencia (Outbox / Inbox):**
    * *Outbox en ms-pagos:* Al recibir el webhook del proveedor, guarda el estado del pago y el evento a emitir en una misma transacción de DB. Un *worker* lee la tabla y lo envía a RabbitMQ. Evita perder eventos si el broker cae tras guardar en DB.
    * *Inbox en ms-tutorias:* Guarda los IDs de los eventos de RabbitMQ procesados. Si RabbitMQ entrega el mismo mensaje de pago exitoso dos veces, se ignora, evitando dobles confirmaciones.
* **Compensación de Saga (Timeouts lógicos):** Si `ms-tutorias` no recibe el evento de pago en 15 minutos, un proceso interno (*cron* o *Delayed Message TTL*) cambia el estado a `CANCELADA` e invoca a `ms-agenda` para liberar el horario. Si el webhook llega *después* de cancelado, el patrón Inbox detecta la anomalía y levanta una alerta para reembolso manual.
* **DLQ y Reintentos:** Si `ms-tutorias` no puede procesar el evento (ej. DB caída), RabbitMQ aplica reintentos con *backoff* exponencial (max 3). Luego, el mensaje va a una *Dead Letter Queue (DLQ)* para intervención operativa.

### 2.8. Observabilidad
No basta con guardar logs de texto. Se requiere:
* **Trazabilidad:** El cliente genera un `X-Correlation-ID` que viaja en las cabeceras HTTP hacia `ms-tutorias` y `ms-pagos`, y se inyecta en los metadatos de los mensajes de RabbitMQ. Esto permite reconstruir la historia completa de una tutoría en el Dashboard.
* **Métricas (Prometheus):**
    * *Histogramas:* Para medir la latencia de respuesta de los webhooks de Visa/Mastercard.
    * *Gauges (Medidores):* Cantidad de tutorías atascadas en estado `PENDIENTE_PAGO`, disparando alertas en Grafana si el número supera un umbral esperado.

### 2.9. Seguridad
* **Límites de confianza (PCI):** `ms-tutorias` y `ms-agenda` nunca reciben, procesan ni almacenan números de tarjetas de crédito (PAN) o CVV. La tokenización ocurre en el frontend y se gestiona exclusivamente entre el proveedor y `ms-pagos`.
* **Autenticación de Servicios:** Las llamadas entre `ms-tutorias` y `ms-pagos` usan tokens JWT de máquina a máquina (M2M), asegurando que el cliente no pueda invocar a `ms-pagos` directamente para auto-aprobarse.
* **Protección de Webhooks:** `ms-pagos` debe validar la firma criptográfica (ej. HMAC-SHA256) enviada en el *header* del webhook del proveedor de pagos para evitar que atacantes falsifiquen eventos de "pagos aprobados".

### 2.10. Despliegue y costo
* **Comparación:**
    * *Serverless (Funciones Lambda):* Escala a cero y reduce costos, pero introduce *cold starts* (latencia inicial) que perjudican la experiencia síncrona del cliente móvil. Además, requiere gestionar el *connection pooling* hacia PostgreSQL a través de proxies adicionales.
    * *Contenedores (PaaS - ej. ECS, Cloud Run):* Mantiene paridad con el entorno local (Docker Compose). Permite ejecutar *workers* asíncronos en segundo plano escuchando a RabbitMQ sin límite de tiempo.
* **Conclusión:** Se opta por Contenedores en modelo PaaS. Ofrece un equilibrio ideal: escala predeciblemente ante picos de demanda (época de exámenes universitarios), menor complejidad operativa y costos controlados mediante auto-escalado horizontal.

### 2.11. Decisión
Se descarta formalmente el diseño transaccional síncrono y se recomienda adoptar la **Saga Orquestada con integración asíncrona mediante webhooks y RabbitMQ**, implementando fuertemente los patrones de Idempotencia y Outbox/Inbox.
**Riesgos residuales:** Si el proveedor externo experimenta una caída masiva y sus webhooks se encolan por horas, el temporizador de `ms-tutorias` cancelará masivamente reservas legítimas. Esto requerirá revisión futura para evaluar la implementación de un proceso automatizado de conciliación financiera nocturna.
