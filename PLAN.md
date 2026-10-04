# Plan técnico — Plataforma de chatbots vendedores multi-tenant

**Estado:** borrador inicial · **Versión:** 0.1 · **Última revisión:** 2026-09-25

Este documento es la base del proyecto. Define la estructura, las decisiones
tomadas y las que quedan abiertas. Se actualiza a medida que el proyecto avanza:
si una decisión cambia, se cambia acá primero.

---

## 1. Objetivo y alcance

Construir **un solo producto** que se vende a varios clientes. Cada cliente
(tenant) tiene su marca, su catálogo, sus integraciones y sus modelos de
lenguaje, pero todos corren sobre el mismo código y la misma base de datos.

El caso de uso principal es un **vendedor autónomo**: conversa por un canal de
mensajería, arma un pedido, envía un link de pago de Mercado Pago y, cuando el
pago se confirma, dispara el proceso de venta.

### La regla que ordena todo

> Un repositorio, una imagen Docker, un deploy. El cliente es una **fila en una
> tabla**, no un branch ni un fork.

Si aparece un `if tenant == "acme"` en el código del core, la abstracción está
mal y hay que corregirla, no parchearla.

### La segunda regla

> El LLM **nunca** decide que algo se pagó.

El agente conversa y arma el carrito. El dinero lo maneja una máquina de estados
determinista alimentada por webhooks verificados. Estos dos planos no se mezclan.

### Fuera de alcance por ahora

- Frontend SPA (React/Vue). Solo templates de Django + HTMX.
- Marketplace de plugins de terceros.
- Multi-región.
- WhatsApp Business API (requiere aprobación de Meta; se agrega en fase 3).

---

## 2. Stack

| Capa | Elección | Por qué |
|---|---|---|
| Backend | Django 5.x | ORM maduro, admin gratis, migraciones sólidas |
| API | **Django Ninja** | async nativo, schemas Pydantic, OpenAPI automático |
| Server | Uvicorn (ASGI) + Gunicorn | el agente necesita async y streaming |
| DB | PostgreSQL 16 + pgvector | una sola base para datos relacionales y RAG |
| Cache / locks | Redis | sesiones, rate limit, locks distribuidos |
| Tareas | Celery | broker Redis en local, SQS en AWS |
| Frontend | Templates + HTMX + Alpine.js | sin build step, sin deploy separado |
| Deps | `uv` | resolución e instalación mucho más rápida que pip/poetry |
| Calidad | ruff, mypy, pytest | estándar actual |
| Infra | Docker Compose (local) → Terraform + ECS Fargate (AWS) | misma imagen en los dos lados |

### Por qué Django Ninja y no DRF

DRF es sync-first. El loop del agente hace varias llamadas a APIs externas
(LLM, Mercado Pago, conectores) que tardan segundos; con DRF cada request
bloquea un worker. Ninja permite `async def` en las vistas, streaming con SSE,
y usa Pydantic — que también vamos a usar para los schemas de las tools del
agente y la validación de configuración de conectores. Un solo sistema de
schemas en todo el proyecto.

El admin de Django sigue disponible para operación interna.

---

## 3. Estructura de carpetas

```
chatbot-platform/
├── Makefile
├── README.md
├── PLAN.md                        # este archivo
├── pyproject.toml                 # deps + config de ruff/mypy/pytest
├── uv.lock
├── .env.example
├── .dockerignore
├── docker-compose.yml             # stack local completo
├── docker-compose.override.yml    # ajustes de dev (no versionado)
├── docker/
│   ├── Dockerfile                 # multi-stage: builder → runtime
│   ├── entrypoint.sh
│   └── postgres/
│       └── init/
│           ├── 01-extensions.sql  # CREATE EXTENSION vector, pgcrypto
│           └── 02-app-role.sql    # rol de app sin BYPASSRLS
│
├── src/
│   ├── manage.py
│   ├── config/                    # el "proyecto" Django
│   │   ├── settings/
│   │   │   ├── base.py
│   │   │   ├── local.py
│   │   │   ├── production.py
│   │   │   └── test.py
│   │   ├── urls.py
│   │   ├── api.py                 # NinjaAPI raíz, routers, versionado
│   │   ├── asgi.py
│   │   ├── wsgi.py
│   │   └── celery.py
│   │
│   └── apps/
│       ├── core/                  # base transversal, sin lógica de negocio
│       ├── tenants/               # clientes, API keys, branding
│       ├── messaging/             # canales, conversaciones, mensajes
│       ├── agents/                # configuración del agente, prompts, runs
│       ├── llm/                   # adaptadores y router de modelos
│       ├── knowledge/             # RAG: documentos, chunks, embeddings
│       ├── catalog/               # productos, precios, stock
│       ├── orders/                # pedidos y máquina de estados
│       ├── payments/              # proveedores de pago, webhooks
│       ├── integrations/          # conectores externos (CRM, ERP, etc.)
│       ├── outbox/                # patrón outbox
│       └── console/               # frontend: config + chat de prueba
│
├── tests/
│   ├── conftest.py
│   ├── factories/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│       └── mercadopago/           # payloads reales de webhooks, anonimizados
│
├── terraform/
│   ├── modules/
│   │   ├── network/
│   │   ├── data/
│   │   ├── ecs-cluster/
│   │   ├── ecs-service/
│   │   ├── queues/
│   │   └── observability/
│   └── envs/
│       ├── dev/
│       └── prod/
│
├── scripts/
│   ├── seed_demo.py               # tenant + catálogo de ejemplo
│   ├── simulate_mp_webhook.py     # dispara un webhook firmado contra local
│   └── wait_for_db.sh
│
└── docs/
    ├── adr/                       # Architecture Decision Records
    │   └── 0001-django-ninja.md
    ├── api.md
    └── runbook.md
```

### Notas sobre la estructura

- **`src/` como raíz del código**: evita que el paquete `apps` colisione con
  nombres del sistema y hace obvio qué se copia a la imagen Docker.
- **`apps/` con un paquete por dominio**: cada app es autocontenida (models,
  services, api, tasks, admin, tests). No se importa entre apps de forma
  circular; si dos apps se necesitan mutuamente, la abstracción está mal.
- **`services.py` en cada app**: la lógica de negocio vive ahí, no en las
  vistas ni en los modelos. Las vistas validan y delegan. Esto mantiene la
  lógica testeable sin HTTP y reutilizable desde Celery y el admin.

Estructura interna típica de una app:

```
apps/orders/
├── __init__.py
├── apps.py
├── models.py
├── schemas.py          # Pydantic in/out
├── services.py         # lógica de negocio, transacciones
├── state_machine.py    # transiciones permitidas
├── api.py              # router de Ninja
├── tasks.py            # tareas Celery
├── admin.py
├── migrations/
└── tests/
```

---

## 4. Cómo se gestiona Django

### 4.1 Settings

Settings divididos con `django-environ`. `base.py` no lee secretos con valores
por defecto en producción — si falta una variable obligatoria, el proceso no
arranca. Fallar al inicio es mejor que descubrirlo a las tres horas.

```python
# config/settings/base.py
env = environ.Env()
SECRET_KEY = env("DJANGO_SECRET_KEY")          # sin default: obligatorio
DEBUG = env.bool("DJANGO_DEBUG", default=False)
DATABASE_URL = env.db("DATABASE_URL")
```

`DJANGO_SETTINGS_MODULE` se define por entorno:
`config.settings.local` en Docker Compose, `config.settings.production` en ECS.

### 4.2 Multi-tenancy

Esquema compartido, base compartida, `tenant_id` en toda tabla de negocio.
**No** usamos schemas separados por tenant: complica migraciones y no escala
a cientos de clientes.

Tres capas de defensa, en orden:

**Capa 1 — Contexto de request.** Un middleware resuelve el tenant desde la
API key (o desde la sesión, en la consola) y lo guarda en un `contextvar`.

```python
# apps/core/context.py
_current_tenant: ContextVar[UUID | None] = ContextVar("current_tenant", default=None)
```

Funciona igual en vistas async y en tareas Celery (la tarea recibe el
`tenant_id` como argumento explícito y lo setea al arrancar).

**Capa 2 — Manager por defecto.** Todo modelo de negocio hereda de
`TenantScopedModel`, cuyo manager filtra automáticamente por el tenant del
contexto y **levanta excepción si no hay contexto**. Olvidarse del `filter()`
no es posible; olvidarse del contexto tampoco es posible en silencio.

```python
class TenantScopedModel(models.Model):
    tenant = models.ForeignKey("tenants.Tenant", on_delete=models.CASCADE)
    objects = TenantScopedManager()      # filtra por contextvar
    all_tenants = models.Manager()       # explícito, solo para tareas de sistema

    class Meta:
        abstract = True
```

**Capa 3 — Row Level Security en Postgres.** Red de seguridad para SQL crudo,
reportes y errores humanos.

> **Atención — esto tiene trampas y hay que hacerlo bien:**
>
> 1. El dueño de las tablas **bypasea** RLS por defecto. Las migraciones corren
>    con el rol dueño; la aplicación corre con un rol distinto, sin
>    `BYPASSRLS`. Además hay que declarar `FORCE ROW LEVEL SECURITY` en cada
>    tabla.
> 2. `set_config('app.tenant_id', x, true)` es **local a la transacción**. Si
>    lo seteás en el middleware fuera de una transacción, no sirve.
> 3. **No** envolver todo el request en `transaction.atomic()`: el agente hace
>    llamadas al LLM de 20–40 segundos y mantendrías una transacción y una
>    conexión abiertas todo ese tiempo.
>
> **Decisión:** el GUC se setea dentro de bloques `transaction.atomic()` cortos
> en la capa de servicios, mediante un context manager `tenant_db_scope()`. El
> filtro del manager (capa 2) es el mecanismo primario; RLS es el backstop.
>
> *Decisión abierta:* si más adelante metemos PgBouncer en modo transaction,
> revisar esto — `SET LOCAL` sigue funcionando, pero `SET` de sesión no.

**Escape hatch para clientes grandes.** El connection string se resuelve por
tenant al arranque. Así "instancia dedicada" es una opción de despliegue del
mismo código, no un fork. No se implementa ahora, pero el punto de extensión
queda previsto.

### 4.3 Async

- Vistas del agente y del chat: `async def`.
- Todo acceso al ORM dentro de código async pasa por `sync_to_async` o por los
  métodos `a*` del ORM (`aget`, `acreate`, `afilter`).
- Los adaptadores de LLM y de pagos usan `httpx.AsyncClient` con un cliente
  compartido y reutilizado (crear uno por request tira el pool de conexiones).

### 4.4 Tareas en segundo plano

Celery, con el broker configurable por variable de entorno:

- Local: `redis://redis:6379/0`
- AWS: `sqs://` (colas creadas por Terraform, con redrive policy a una DLQ)

Limitaciones conocidas de SQS como broker de Celery: no hay `countdown` más
allá del visibility timeout, y no hay remote control. Para el scheduling usamos
**EventBridge Scheduler** en AWS y `celery beat` en local, no Celery ETA.

**Los eventos críticos de dinero no van directo a Celery.** Van a la tabla
`outbox` en la misma transacción que el cambio de estado, y un dispatcher los
publica. Ver sección 8.

### 4.5 Frontend (app `console`)

Templates de Django + HTMX + Alpine.js. Dos secciones:

- **Configuración**: alta de tenant, branding, catálogo, bindings de modelos,
  credenciales de Mercado Pago, prompts.
- **Chat de prueba**: la misma API que usan los canales reales, con streaming
  SSE, para probar el agente sin depender de WhatsApp.

Sin build step, sin `node_modules`, sin deploy separado. Cuando el producto
justifique una SPA, la API ya está lista y se agrega sin tocar el backend.

---

## 5. Apps de Django y tablas principales

Convención general: PK `UUID` en todo (los IDs viajan a sistemas externos y
los enteros secuenciales filtran volumen de negocio), `created_at` /
`updated_at` en todo, borrado lógico solo donde haga falta.

### 5.1 `core`

Sin tablas de negocio. Contiene:

- `TimeStampedModel`, `UUIDModel`, `TenantScopedModel` (abstractos)
- `TenantMiddleware`, `RequestIDMiddleware`
- Contexto de tenant (`contextvars`)
- Excepciones de dominio y su mapeo a respuestas HTTP
- Helpers de idempotencia

### 5.2 `tenants`

| Tabla | Campos clave |
|---|---|
| `Tenant` | `id`, `slug`, `name`, `status`, `plan`, `timezone`, `created_at` |
| `ApiKey` | `id`, `tenant`, `kind` (`public`/`secret`), `prefix`, `hashed_key`, `allowed_origins`, `last_used_at`, `revoked_at` |
| `BrandingConfig` | `tenant` (1-1), `display_name`, `logo_url`, `primary_color`, `welcome_message`, `tone_notes` |
| `FeatureFlag` | `tenant`, `key`, `enabled`, `payload` (JSONB) |
| `UsageCounter` | `tenant`, `period`, `metric`, `value` — mensajes, tokens, pedidos |

Las API keys se guardan **hasheadas** (`argon2`). El `prefix` (primeros 8
caracteres) se guarda en claro para poder identificarlas en la UI y en los logs
sin exponer el secreto.

Dos tipos de key: **pública** (widget del navegador, atada a dominios,
rate-limited, solo puede abrir conversaciones) y **secreta**
(server-to-server, administración).

### 5.3 `messaging`

| Tabla | Campos clave |
|---|---|
| `Channel` | `tenant`, `kind` (`web`/`whatsapp`/`telegram`), `config` (JSONB), `credentials_ref`, `is_active` |
| `Contact` | `tenant`, `external_id`, `channel`, `display_name`, `phone`, `email`, `attributes` (JSONB) |
| `Conversation` | `tenant`, `contact`, `channel`, `status`, `agent_config`, `handoff_requested_at`, `assigned_to`, `last_message_at` |
| `Message` | `conversation`, `role` (`user`/`assistant`/`tool`/`system`), `content`, `tool_calls` (JSONB), `attachments`, `provider_message_id`, `created_at` |

**Separar canal de lógica.** Cada canal tiene un adaptador en el borde que
normaliza al formato interno de `Message`. Aunque hoy solo exista el widget
web, la costura se diseña ahora: cuando llegue WhatsApp (y va a llegar en la
segunda reunión con cualquier cliente), es un adaptador nuevo, no una reforma.

`handoff_requested_at` existe **desde la primera versión**. Un vendedor
autónomo necesita poder escalar a un humano —por monto, por producto sensible,
por cliente frustrado, o porque el modelo no entiende. Es barato ahora y caro
después.

### 5.4 `agents`

| Tabla | Campos clave |
|---|---|
| `AgentConfig` | `tenant`, `slug`, `name`, `service_type`, `is_default`, `max_turns`, `handoff_rules` (JSONB) |
| `PromptTemplate` | `agent_config`, `role`, `body`, `version`, `is_active`, `created_by` |
| `ToolBinding` | `agent_config`, `tool_name`, `is_enabled`, `config` (JSONB) |
| `AgentRun` | `id`, `tenant`, `conversation`, `agent_config`, `status`, `turns`, `started_at`, `finished_at`, `error` |
| `AgentStep` | `run`, `index`, `kind` (`llm_call`/`tool_call`), `payload`, `duration_ms` |

El prompt se arma en capas: **base no editable** (reglas de seguridad, formato,
prohibición de inventar precios) + **capa del tenant** (marca, tono, políticas
comerciales). El tenant nunca puede sobrescribir la capa base.

`PromptTemplate` es versionado con rollback. Cambiar un prompt es un deploy
de contenido y tiene que poder revertirse sin tocar código.

`AgentRun` / `AgentStep` son la traza de ejecución. Sin esto, debuggear por qué
el bot dijo algo raro es imposible.

### 5.5 `catalog`

| Tabla | Campos clave |
|---|---|
| `Category` | `tenant`, `name`, `parent`, `position` |
| `Product` | `tenant`, `sku`, `name`, `description`, `category`, `is_active`, `attributes` (JSONB) |
| `ProductVariant` | `product`, `sku`, `name`, `attributes`, `is_active` |
| `Price` | `variant`, `currency`, `amount`, `valid_from`, `valid_to` |
| `StockLevel` | `variant`, `quantity`, `reserved`, `updated_at` |

**El precio y el stock se leen del catálogo en el momento de crear la
preferencia de pago.** El LLM puede haber dicho cualquier cosa durante la
conversación; lo que va al link sale de la base. Si difiere de lo conversado,
el agente avisa del cambio antes de mandar el link.

`Price` con vigencia temporal en vez de un campo `price` en `Product`: permite
listas de precios, promociones y auditoría de qué precio estaba vigente cuando
se cerró un pedido.

### 5.6 `orders`

| Tabla | Campos clave |
|---|---|
| `Order` | `tenant`, `conversation`, `contact`, `number`, `status`, `currency`, `subtotal`, `total`, `expires_at`, `metadata` |
| `OrderItem` | `order`, `variant`, `sku_snapshot`, `name_snapshot`, `unit_price_snapshot`, `quantity`, `line_total` |
| `OrderEvent` | `order`, `from_status`, `to_status`, `actor` (`system`/`webhook`/`user`/`agent`), `reason`, `payload`, `created_at` |

Los campos `*_snapshot` congelan el estado del producto al momento del pedido.
Si mañana cambia el precio o el nombre, el pedido histórico no se altera.

#### Máquina de estados

```
draft          → el agente arma el carrito
pending_payment → preferencia creada, link enviado (TTL configurable)
paid           → webhook verificado + reconsulta a la API del proveedor
fulfilling     → descuento de stock, comprobante
completed
```

Ramas: `expired`, `rejected`, `cancelled`, `refunded`, `charged_back`.

Implementación: un dict `TRANSITIONS` explícito + una función
`transition(order, to_status, actor, reason)` que valida, persiste el cambio y
escribe el `OrderEvent` **en la misma transacción**. Sin librería de FSM: el
conjunto es chico y el control explícito vale más que la magia.

> El agente tiene la tool `crear_link_pago(order_id)`. **No** tiene ninguna
> tool que escriba `paid`. Esa transición solo la pueden hacer el handler de
> webhook y el job de reconciliación.

### 5.7 `payments` — ver sección 7

### 5.8 `knowledge` (RAG)

| Tabla | Campos clave |
|---|---|
| `Document` | `tenant`, `source_type`, `source_ref`, `title`, `checksum`, `status`, `ingested_at` |
| `Chunk` | `tenant`, `document`, `index`, `content`, `embedding` (`vector(N)`), `token_count`, `metadata` |
| `IngestionJob` | `tenant`, `document`, `status`, `error`, `started_at`, `finished_at` |

Índice HNSW sobre `embedding`, con `tenant_id` en el filtro previo. Una sola
base para todo; no hace falta un vector store dedicado hasta bastante más
adelante.

La dimensión del vector depende del modelo de embeddings. **Decisión abierta:**
fijar un modelo de embeddings por tenant y no permitir cambiarlo sin
reindexar, o guardar la dimensión por documento con columnas separadas. Lo
primero es más simple; empezamos por ahí.

### 5.9 `integrations`

| Tabla | Campos clave |
|---|---|
| `ConnectorInstance` | `tenant`, `connector_type`, `name`, `config` (JSONB), `credentials_ref`, `is_active`, `last_health_check` |
| `ConnectorCall` | `tenant`, `instance`, `tool_name`, `request`, `response`, `status`, `duration_ms`, `created_at` |

Los conectores se registran en un registry global y están **apagados por
defecto** para todos. Se prenden por configuración.

Cuando un cliente pide algo, intentar en este orden:

1. **Configuración** (95% de los casos) — colores, prompt, qué conectores
   están activos. No requiere deploy.
2. **Conector nuevo** — módulo que implementa la interfaz común, se mergea al
   core, apagado por defecto.
3. **Plugin por cliente** — lógica que no le sirve a nadie más. Paquete
   separado que implementa interfaces documentadas, cargado dinámicamente,
   dentro del mismo repo.

Lo importante no es la capa 3, es que exista: es lo que evita que el cliente
raro obligue a forkear.

> No construir el marketplace de plugins para siete clientes que todavía no
> existen. Construir las **costuras** correctas y generalizar cuando el tercer
> cliente pida algo parecido. La sobre-abstracción temprana también es una
> forma de quedarse atascado.

### 5.10 `outbox`

| Tabla | Campos clave |
|---|---|
| `OutboxEvent` | `id`, `tenant`, `aggregate_type`, `aggregate_id`, `event_type`, `payload`, `status`, `attempts`, `available_at`, `published_at`, `last_error` |

Índice parcial sobre `status='pending'` ordenado por `available_at`.

---

## 6. Capa de LLM — cómo elegir modelo por servicio

Este es el punto donde más fácil se ata uno a un proveedor. La abstracción
correcta no es "el modelo", es **la tarea**.

### 6.1 El principio

El código nunca dice "dame Claude Sonnet". Dice **"dame el modelo para la tarea
`chat.sales`"**. Quién responde esa pregunta es configuración por tenant.

```python
model = await resolve_model(tenant, task="chat.sales")
```

### 6.2 Catálogo de tareas

Las tareas son un enum cerrado en código. Agregar una es un cambio de código
(deliberadamente: son puntos de extensión del producto, no configuración libre).

| Tarea | Para qué | Perfil típico |
|---|---|---|
| `chat.sales` | conversación de venta con tools | modelo fuerte, caro |
| `chat.support` | preguntas frecuentes sobre el catálogo | modelo medio |
| `classify.intent` | ¿es una consulta, una compra, una queja? | modelo chico, rápido |
| `extract.order` | pasar texto libre a items estructurados | modelo medio con JSON estricto |
| `summarize.conversation` | resumen al cerrar o al escalar a humano | modelo chico |
| `embed.documents` | ingesta de RAG | modelo de embeddings |
| `moderate.input` | filtro de entrada | modelo chico o API de moderación |

Esto también es la palanca de costo: `classify.intent` corriendo en un modelo
barato en vez del modelo grande cambia la economía del producto sin tocar una
línea de la lógica del agente.

### 6.3 Interfaz común

```python
# apps/llm/base.py
@dataclass(frozen=True)
class Usage:
    input_tokens: int
    output_tokens: int
    cached_input_tokens: int = 0

@dataclass(frozen=True)
class ToolCall:
    id: str
    name: str
    arguments: dict

@dataclass(frozen=True)
class LLMResponse:
    text: str
    tool_calls: list[ToolCall]
    usage: Usage
    finish_reason: str
    raw: dict                      # respuesta cruda, para debug

class LLMAdapter(Protocol):
    provider: str

    async def complete(
        self, *, model: str, messages: list[LLMMessage],
        tools: list[ToolSpec] | None = None, params: ModelParams,
    ) -> LLMResponse: ...

    async def stream(
        self, *, model: str, messages: list[LLMMessage],
        tools: list[ToolSpec] | None = None, params: ModelParams,
    ) -> AsyncIterator[StreamChunk]: ...

    async def embed(
        self, *, model: str, texts: list[str],
    ) -> list[list[float]]: ...
```

Adaptadores previstos: `anthropic`, `openai`, `google`, `bedrock`, `fake`.

El `FakeAdapter` devuelve respuestas predefinidas y es lo que se usa en los
tests: rápido, determinista y gratis. Los tests no deben llamar a un proveedor
real nunca.

El formato interno de mensajes y tool calls es **propio**, no el de ningún
proveedor. Cada adaptador traduce en las dos direcciones. Es más trabajo al
principio y es lo único que hace que cambiar de proveedor no sea un rewrite.

### 6.4 Tablas

| Tabla | Campos clave |
|---|---|
| `ModelBinding` | `tenant` (nullable = default global), `task`, `provider`, `model_name`, `params` (JSONB), `fallback` (self-FK), `is_active` |
| `LLMCall` | `tenant`, `agent_run`, `task`, `provider`, `model_name`, `input_tokens`, `output_tokens`, `cost_usd`, `latency_ms`, `status`, `created_at` |
| `ProviderCredential` | `tenant` (nullable = plataforma), `provider`, `credentials_ref`, `is_active` |

Restricción única: `(tenant, task)` — un binding activo por tarea y tenant.

**Orden de resolución:** binding del tenant para la tarea → binding global para
la tarea → error explícito. Sin defaults silenciosos: si no hay binding, hay
que saberlo.

`LLMCall` no es opcional. Sin contabilidad de tokens y costo por tenant no se
puede tarifar, no se puede detectar abuso y no se puede saber si el producto
tiene margen. Es de lo más caro de retrofitear.

### 6.5 Router

```python
# apps/llm/router.py
async def resolve_model(tenant: Tenant, task: Task) -> ResolvedModel:
    """Devuelve provider + modelo + params para una tarea. Cachea en Redis."""

async def call(
    tenant: Tenant, task: Task, messages, tools=None, *, run: AgentRun | None = None
) -> LLMResponse:
    """
    Resuelve, llama, registra LLMCall, aplica fallback ante error del proveedor.
    Todo el resto del código usa esta función, nunca un adaptador directo.
    """
```

El fallback se dispara ante errores de infraestructura del proveedor (5xx,
rate limit, timeout), **no** ante respuestas que no nos gustan. Máximo un salto
de fallback, para no encadenar reintentos caros.

### 6.6 Tools del agente

Cada tool declara su schema con Pydantic y se traduce al formato de cada
proveedor. Las tools de venta:

- `buscar_producto(query, categoria?)` → lee del catálogo
- `ver_producto(sku)` → precio y stock actuales
- `agregar_al_carrito(sku, cantidad)` → valida stock, recalcula total
- `quitar_del_carrito(item_id)`
- `ver_carrito()`
- `crear_link_pago()` → crea la preferencia y devuelve la URL
- `escalar_a_humano(motivo)`

Ninguna tool recibe un precio como argumento. El precio siempre se lee de la
base. Esta es la defensa concreta contra que el modelo invente números.

---

## 7. Pagos

### 7.1 Interfaz común

```python
# apps/payments/base.py
class PaymentProvider(Protocol):
    slug: str

    async def create_checkout(self, order: Order, account: PaymentAccount) -> CheckoutLink: ...
    def verify_webhook(self, headers: Mapping, body: bytes, account: PaymentAccount) -> WebhookNotification: ...
    async def fetch_payment(self, external_id: str, account: PaymentAccount) -> PaymentStatus: ...
    async def refund(self, payment: Payment, amount: Decimal | None = None) -> RefundResult: ...
```

Implementaciones previstas:

| Slug | Estado | Notas |
|---|---|---|
| `mercadopago` | fase 1 | Checkout Pro, link de pago |
| `manual_transfer` | fase 1 | transferencia bancaria con confirmación humana — muy pedido en Argentina |
| `stripe` | fase 3 | para clientes fuera de LATAM |

`manual_transfer` entra en fase 1 a propósito: obliga a que la máquina de
estados no asuma un proveedor específico, y es una funcionalidad real que los
clientes van a pedir.

### 7.2 Tablas

| Tabla | Campos clave |
|---|---|
| `PaymentAccount` | `tenant`, `provider`, `credentials_ref`, `webhook_secret_ref`, `mode` (`sandbox`/`live`), `is_active` |
| `Payment` | `tenant`, `order`, `provider`, `external_id`, `status`, `amount`, `currency`, `checkout_url`, `raw_status`, `paid_at` |
| `PaymentWebhookEvent` | `provider`, `external_event_id` (**unique**), `tenant`, `headers`, `body`, `signature_valid`, `processed_at`, `result` |
| `Refund` | `payment`, `external_id`, `amount`, `status`, `reason` |

El `unique` sobre `external_event_id` es lo que hace la deduplicación: si el
proveedor reenvía la notificación, el insert falla, respondemos 200 y cortamos.

### 7.3 Las cuatro reglas del dinero

**1. Nunca confiar en el payload del webhook.** El webhook avisa "pasó algo con
el pago X". Se toma ese ID y se **vuelve a consultar la API del proveedor** para
ver el estado real. El webhook es un timbre, no una fuente de verdad.

**2. Idempotencia en las dos puntas.** Deduplicación de webhooks por
`external_event_id`; header de idempotencia y `external_reference` con el
`order_id` al crear la preferencia.

**3. Outbox.** Al confirmar el pago, en la **misma transacción** se escribe el
cambio de estado y el evento pendiente. Un worker lo consume y hace el
fulfillment. Así nunca se cobra sin entregar porque falló el envío del mensaje.

**4. Job de reconciliación.** Los webhooks se pierden. Un cron cada 5 minutos
toma los pedidos en `pending_payment` de más de N minutos y le pregunta al
proveedor directamente. Esto es lo que evita el llamado de "pagué y no me llegó
nada".

### 7.4 El endpoint de webhook

```
POST /webhooks/mercadopago/<tenant_slug>/
```

El tenant va en la URL porque la `notification_url` se configura por
preferencia, así que sabemos a qué tenant corresponde antes de parsear nada.

El handler hace **solo tres cosas** y responde en menos de 100 ms:

1. Valida la firma HMAC del header `x-signature` contra el secreto del tenant.
2. Persiste el `PaymentWebhookEvent` crudo (o descarta si ya existe).
3. Encola el procesamiento.

Todo lo demás —reconsulta a la API, transición de estado, fulfillment— pasa en
el worker.

> **Pendiente de verificación:** el formato exacto del manifest que Mercado
> Pago firma (`id`, `request-id`, `ts`) y el header de idempotencia cambian
> entre versiones de la API. Confirmar contra la documentación vigente antes de
> implementar y dejar fixtures reales en `tests/fixtures/mercadopago/`.

Es un servicio ECS **separado** de la API principal: si el chat se satura, los
webhooks de Mercado Pago tienen que seguir entrando.

---

## 8. Comunicación entre servicios

| Origen | Destino | Mecanismo | Por qué |
|---|---|---|---|
| Widget / canal | `api` | HTTPS + SSE | streaming de la respuesta del agente |
| `api` | `worker` | Celery (Redis local / SQS en AWS) | el turno del agente es asíncrono |
| Mercado Pago | `webhooks` | HTTPS firmado | servicio aislado del resto |
| `webhooks` | `worker` | Celery | procesamiento fuera del path del webhook |
| `orders` | `fulfillment` | **outbox en Postgres** → dispatcher | garantía transaccional con el cambio de estado |
| `worker` | canal | adaptador de canal | mensajes salientes al cliente |
| Cron | `worker` | Celery beat (local) / EventBridge (AWS) | reconciliación, limpieza, expiración |

### Por qué el outbox y no Celery directo para el dinero

`transaction.commit()` y `celery_task.delay()` no son atómicos. Si la
transacción commitea y el broker está caído, el evento se pierde: pedido pagado
sin fulfillment. Si se encola antes del commit y la transacción hace rollback,
el evento existe para un pedido que no. El outbox elimina las dos ventanas
porque el evento se escribe en la misma transacción que el cambio de estado.

Para lo no crítico (embeddings, resúmenes, métricas) Celery directo está bien.

---

## 9. Docker — todo corre en local

### 9.1 Servicios en `docker-compose.yml`

| Servicio | Imagen | Notas |
|---|---|---|
| `db` | `pgvector/pgvector:pg16` | scripts de init para extensiones y rol de app |
| `redis` | `redis:7-alpine` | broker Celery + cache |
| `web` | build local | uvicorn con `--reload` |
| `worker` | build local | `celery -A config worker` |
| `beat` | build local | `celery -A config beat` |
| `tunnel` | `cloudflare/cloudflared` | expone `web` para los webhooks de MP |

El servicio `tunnel` está en un profile (`--profile webhooks`) para no
levantarlo siempre.

`depends_on` con `condition: service_healthy`, no con el simple listado: sin
healthcheck, `web` arranca antes que Postgres y falla la primera migración.

### 9.2 Dockerfile

Multi-stage:

- **builder**: instala `uv`, resuelve dependencias, construye el virtualenv
- **runtime**: `python:3.12-slim`, copia el venv, usuario no-root, sin
  compiladores

La **misma imagen** corre `web`, `worker` y `beat`; cambia el comando. Esto es
lo que garantiza que lo que probaste en local es lo que corre en ECS.

### 9.3 Variables de entorno

`.env.example` versionado con todas las claves y valores falsos. `.env` real
nunca se commitea. El entrypoint valida que las variables obligatorias existan
antes de arrancar.

---

## 10. Terraform y AWS

### 10.1 Forma de la infraestructura

```
Internet → CloudFront + WAF → ALB (subredes públicas)
                                ↓
   VPC (2 AZ) — subredes privadas:
     ECS Fargate: api · webhooks · workers
     RDS PostgreSQL (pgvector) · ElastiCache Redis
   NAT Gateway → salida hacia Mercado Pago y los LLM

   Fuera de la VPC: SQS + DLQ · Secrets Manager · EventBridge · S3 · ECR
```

**Por qué ECS Fargate:** corre la misma imagen Docker que se probó en local,
cero drift entre dev y prod. Lambda es tentador por el costo inicial, pero el
streaming es incómodo, el turno del agente puede tardar 20–40 segundos (y se
paga la espera a la API del LLM) y los workers son procesos de larga duración.
EKS es desproporcionado para este tamaño.

### 10.2 Estructura

```
terraform/
├── modules/
│   ├── network/        # vpc, subnets, nat, vpc endpoints, security groups
│   ├── data/           # rds, elasticache, parameter groups, backups
│   ├── ecs-cluster/    # cluster, roles IAM, log groups
│   ├── ecs-service/    # REUTILIZABLE: task def, service, target group, autoscaling
│   ├── queues/         # sqs + dlq + alarmas
│   └── observability/  # alarmas, dashboards, SNS
└── envs/
    ├── dev/
    │   ├── main.tf  backend.tf  variables.tf  terraform.tfvars
    └── prod/
```

`ecs-service` como módulo reutilizable es la pieza clave: los tres servicios
(`api`, `webhooks`, `workers`) son tres invocaciones del mismo módulo con
variables distintas. Un cliente nuevo **no agrega infraestructura** — ya
establecimos que es una fila en una tabla.

### 10.3 Reglas que evitan dolor

**El tag de la imagen no se maneja en Terraform.** Terraform aprovisiona
infraestructura; el CI/CD actualiza task definitions. Si el tag vive en el
`.tf`, cada deploy es un `terraform apply` y vas a terminar haciendo apply
desde la laptop a las once de la noche.

```hcl
resource "aws_ecs_service" "this" {
  # ...
  lifecycle {
    ignore_changes = [task_definition, desired_count]
  }
}
```

**Estado separado por entorno**, backend en S3 con lock. Nunca un solo state
para dev y prod.

**Los secretos se crean vacíos.** El recurso `aws_secretsmanager_secret` va en
Terraform; el `secret_version` con el valor real, no. Se carga por CLI. Si el
token de producción de Mercado Pago queda en el state file, ese archivo pasa a
ser material sensible con todo lo que eso implica.

**`prevent_destroy = true` y `deletion_protection` en RDS.** Con plata de por
medio no es opcional.

### 10.4 Trampas de costo

- **NAT Gateway**: ~USD 32/mes por AZ más tráfico. Todas las llamadas al LLM
  salen por ahí. Mitigación: un solo NAT hasta tener tráfico real, y VPC
  endpoints para S3 (gateway, gratis), ECR, Secrets Manager y CloudWatch Logs.
- **Secrets Manager**: USD 0.40 por secreto por mes. Con 7 clientes es
  irrelevante; pasando los ~50 tenants, migrar a credenciales cifradas en la
  base con envelope encryption de KMS.
- **Redis**: se puede posponer. Sesiones en Postgres y rate limit en el WAF
  alcanzan bastante lejos.

### 10.5 La alarma que importa

De todo lo monitoreable, una sola va al teléfono: **profundidad de la DLQ mayor
a cero**. Si un mensaje llegó a la dead letter queue, hay un pedido pagado que
no se entregó.

Las demás (CPU, memoria, p99, tasa de 5xx) son higiene normal y pueden esperar.

### 10.6 Orden de aplicación

1. `network` + `data` + un servicio ECS que responda `/health`, con dominio y
   certificado. Confirma que el camino de red funciona.
2. El resto de los servicios.
3. Colas y EventBridge.
4. Autoscaling y observabilidad.

> No levantar AWS hasta que el flujo de pago funcione end-to-end en local con
> túnel. Si la nube va primero, vas a estar depurando security groups cuando el
> problema está en la validación de la firma del webhook.

---

## 11. Makefile

```makefile
.DEFAULT_GOAL := help
COMPOSE := docker compose
RUN     := $(COMPOSE) run --rm web
EXEC    := $(COMPOSE) exec web
ENV     ?= dev
TF      := terraform -chdir=terraform/envs/$(ENV)

# ---------- entorno ----------
.PHONY: help init up down restart logs ps
help:            ## Lista los comandos disponibles
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-18s\033[0m %s\n", $$1, $$2}'

init:            ## Primer arranque: .env, build, migrate, seed, superuser
	@test -f .env || cp .env.example .env
	$(COMPOSE) build
	$(MAKE) migrate
	$(MAKE) seed
	$(MAKE) superuser

up:              ## Levanta el stack
	$(COMPOSE) up -d
down:            ## Baja el stack (conserva volúmenes)
	$(COMPOSE) down
restart:         ## Reinicia todo
	$(COMPOSE) restart
logs:            ## Logs en vivo (make logs S=worker)
	$(COMPOSE) logs -f $(or $(S),)
ps:              ## Estado de los servicios
	$(COMPOSE) ps

# ---------- django ----------
.PHONY: sh shell dbsh migrate makemigrations superuser seed
sh:              ## Shell bash en el contenedor web
	$(EXEC) bash
shell:           ## Shell de Django (shell_plus)
	$(EXEC) python manage.py shell_plus
dbsh:            ## psql contra la base local
	$(COMPOSE) exec db psql -U postgres -d chatbot
migrate:         ## Aplica migraciones
	$(RUN) python manage.py migrate
makemigrations:  ## Genera migraciones (make makemigrations A=orders)
	$(RUN) python manage.py makemigrations $(or $(A),)
superuser:       ## Crea un superusuario
	$(RUN) python manage.py createsuperuser
seed:            ## Carga tenant demo + catálogo de ejemplo
	$(RUN) python manage.py seed_demo

# ---------- calidad ----------
.PHONY: test test-fast cov lint fmt typecheck check
test:            ## Corre toda la suite
	$(RUN) pytest
test-fast:       ## Solo tests unitarios, en paralelo
	$(RUN) pytest tests/unit -x -q -n auto
cov:             ## Cobertura con reporte HTML
	$(RUN) pytest --cov=apps --cov-report=html --cov-report=term-missing
lint:            ## ruff check
	$(RUN) ruff check src tests
fmt:             ## ruff format + autofix
	$(RUN) ruff format src tests && $(RUN) ruff check --fix src tests
typecheck:       ## mypy
	$(RUN) mypy src
check: lint typecheck test  ## Todo lo que corre el CI

# ---------- desarrollo de features ----------
.PHONY: tunnel mp-webhook worker-shell reset-db
tunnel:          ## Túnel público para recibir webhooks de Mercado Pago
	$(COMPOSE) --profile webhooks up -d tunnel
	@sleep 3 && $(COMPOSE) logs tunnel | grep -i 'trycloudflare\|https://'
mp-webhook:      ## Simula un webhook firmado (make mp-webhook ORDER=<uuid>)
	$(RUN) python scripts/simulate_mp_webhook.py --order $(ORDER)
worker-shell:    ## Shell en el contenedor del worker
	$(COMPOSE) exec worker bash
reset-db:        ## DESTRUCTIVO: borra la base y rehace todo
	$(COMPOSE) down -v
	$(COMPOSE) up -d db redis
	@sleep 5
	$(MAKE) migrate seed

# ---------- terraform ----------
.PHONY: tf-init tf-plan tf-apply tf-fmt tf-validate
tf-init:         ## terraform init (make tf-init ENV=prod)
	$(TF) init
tf-plan:         ## terraform plan
	$(TF) plan -out=tfplan
tf-apply:        ## terraform apply del plan guardado
	$(TF) apply tfplan
tf-fmt:          ## Formatea los .tf
	terraform fmt -recursive terraform/
tf-validate:     ## Valida la configuración
	$(TF) validate
```

Convenciones: todos los targets tienen `## comentario` para que `make help`
sea la única documentación que hay que recordar. Los targets destructivos
llevan la palabra `DESTRUCTIVO` en la descripción.

---

## 12. Testing

| Nivel | Qué cubre | Herramientas |
|---|---|---|
| Unit | servicios, máquina de estados, router de LLM | pytest, `FakeAdapter` |
| Integración | endpoints de API, webhooks, flujo completo de pedido | pytest-django, fixtures reales |
| Contrato | que cada adaptador de LLM y de pago cumpla la interfaz | suite parametrizada común |

Reglas:

- **Ningún test llama a un proveedor real.** `FakeAdapter` para LLMs, payloads
  grabados para Mercado Pago.
- **El flujo del dinero tiene test end-to-end obligatorio**: crear pedido →
  crear preferencia → recibir webhook (firmado, y también mal firmado) →
  transición → outbox → fulfillment. Incluyendo el caso de webhook duplicado.
- **Test de aislamiento entre tenants**: crear dos tenants con datos y
  verificar que ninguna query cruza. Este test se corre en CI siempre.
- `factory_boy` para construir objetos; nada de fixtures JSON gigantes.

---

## 13. Seguridad y operación

- **API keys** hasheadas con argon2; solo el prefijo en claro.
- **Credenciales de tenants** (Mercado Pago, proveedores) en Secrets Manager;
  en la base solo la referencia (`credentials_ref`).
- **Logs estructurados** en JSON con `request_id`, `tenant_id`, `conversation_id`.
  **Nunca** loguear contenido de mensajes en producción sin política explícita
  de retención — son datos de clientes finales.
- **Rate limit** por API key y por contacto. Las llamadas al LLM cuestan plata;
  el abuso es real y barato de ejecutar.
- **Retención**: definir por tenant cuánto se guardan conversaciones. Pendiente
  de decidir el default.
- **Runbook** (`docs/runbook.md`): qué hacer cuando suena la alarma de la DLQ,
  cómo reprocesar un webhook, cómo revertir un prompt.

---

## 14. Fases

### Fase 1 — El camino del dinero (sin LLM)

Criterio de listo: crear un pedido por API, generar un link de Mercado Pago,
recibir el webhook por túnel, verificar la firma y ver el pedido en `paid`.
Todo con `curl`, sin agente.

- `core`, `tenants`, `catalog`, `orders`, `payments`, `outbox`
- Docker Compose completo
- Suite de tests del flujo de pago
- Job de reconciliación

### Fase 2 — El agente

Criterio de listo: una conversación por la consola web que arme un carrito y
mande un link de pago real.

- `llm` con dos adaptadores + router + `LLMCall`
- `agents`, `messaging`
- Tools de venta
- `console` con chat de prueba y configuración

### Fase 3 — Producto

- `knowledge` (RAG)
- `integrations` con dos conectores reales
- Canal WhatsApp
- Terraform + despliegue en AWS
- Panel de administración self-serve

**No** construir la fase 3 mientras se construye la fase 1. El orden importa:
si se empieza por el chat, vas a estar debuggeando prompts cuando el problema
está en el webhook.

---

## 15. Decisiones abiertas

| Tema | Opciones | Cuándo decidir |
|---|---|---|
| RLS con pooling | `SET LOCAL` por transacción vs. manager-only | antes de meter PgBouncer |
| Embeddings | modelo fijo por tenant vs. dimensión variable | al empezar fase 3 |
| Retención de conversaciones | default de la plataforma | antes del primer cliente real |
| Instancia dedicada | cuándo se ofrece y a qué precio | cuando lo pida un cliente |
| Cola de conversaciones | Celery directo vs. outbox también acá | al medir latencia real |

---

## 16. Registro de decisiones

Las decisiones estructurales se documentan como ADR en `docs/adr/`, numeradas
y con formato: contexto, decisión, consecuencias, alternativas descartadas.

Una ADR se escribe cuando la decisión es difícil de revertir. No hace falta una
para elegir un nombre de variable; sí para elegir Django Ninja sobre DRF, o
esquema compartido sobre schemas por tenant.
