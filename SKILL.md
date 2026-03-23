---
name: globalsone-bff-skill
description: Skill para desarrollo del Backend for Frontend (BFF). Guia la creacion de agregadores, orquestadores, adaptadores y patrones de optimizacion para clientes.
argument-hint: "<command> [options]"
---

# BFF (Backend for Frontend) Skill

Genera y gestiona el codigo del Backend for Frontend para optimizar la comunicacion entre clientes y microservicios.

## Recursos Asociados

| Recurso | Descripcion |
|---------|-------------|
| `globalsone-bff-template` | Template base del BFF |
| `globalsone-bff-cli` | CLI para generar codigo |

## Que es un BFF?

El Backend for Frontend es una capa de agregacion que:

- **Agrega** datos de multiples microservicios en una sola respuesta
- **Orquesta** llamadas a servicios de manera eficiente
- **Adapta** respuestas al formato optimo para cada cliente (web, mobile, etc.)
- **Cachea** datos frecuentes para reducir latencia
- **Transforma** datos para reducir payload y mejorar rendimiento

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Web App   │     │ Mobile App  │     │   TV App    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │     BFF     │
                    │  Aggregator │
                    └──────┬──────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
│    User     │     │   Product   │     │    Order    │
│   Service   │     │   Service   │     │   Service   │
└─────────────┘     └─────────────┘     └─────────────┘
```

## Inicio Rapido

```bash
# 1. Clonar template
git clone https://github.com/GlobalS1/globalsone-bff-template.git my-bff
cd my-bff
pnpm install

# 2. Instalar CLI
npm install -g globalsone-bff-cli

# 3. Generar agregadores
bff generate aggregator dashboard
bff generate orchestrator checkout-flow
```

## CLI - Generador de Codigo

El CLI `globalsone-bff-cli` permite generar codigo siguiendo los patrones del BFF.

### Instalacion

```bash
# Global
npm install -g globalsone-bff-cli

# O como devDependency
npm install -D globalsone-bff-cli
```

### Comandos Disponibles

| Comando | Alias | Descripcion |
|---------|-------|-------------|
| `bff generate aggregator <name>` | `bff g a` | Genera agregador de datos |
| `bff generate orchestrator <name>` | `bff g o` | Genera orquestador de flujo |
| `bff generate adapter <name>` | `bff g ad` | Genera adaptador de servicio |
| `bff generate transformer <name>` | `bff g t` | Genera transformador de datos |
| `bff generate cache <name>` | `bff g c` | Genera estrategia de cache |
| `bff generate endpoint <path>` | `bff g e` | Genera endpoint BFF |

### Generar Agregador

```bash
# Agregador basico
bff g a dashboard

# Agregador con servicios especificos
bff g a user-profile --services "user,orders,preferences"

# Agregador con cache
bff g a product-catalog --cache 300

# Preview sin crear archivos
bff g a my-aggregator --dry-run
```

### Generar Orquestador

```bash
# Orquestador de flujo
bff g o checkout-flow

# Con pasos especificos
bff g o onboarding --steps "validate,create-user,send-email,setup-preferences"

# Con rollback
bff g o payment-process --with-rollback

# Preview
bff g o my-flow --dry-run
```

### Generar Adaptador

```bash
# Adaptador de servicio
bff g ad user-service --base-url "http://localhost:3001"

# Con metodos especificos
bff g ad order-service --methods "getAll,getById,create"

# Con retry y circuit breaker
bff g ad payment-service --with-retry --with-circuit-breaker
```

### Generar Transformador

```bash
# Transformador de datos
bff g t user-response

# Para cliente especifico
bff g t product-list --client mobile

# Con campos seleccionados
bff g t order-summary --fields "id,total,status,items.name"
```

### Estructura del BFF

```
src/
├── aggregators/                 # Agregadores de datos
│   ├── dashboard.aggregator.ts
│   └── user-profile.aggregator.ts
├── orchestrators/               # Orquestadores de flujos
│   ├── checkout.orchestrator.ts
│   └── onboarding.orchestrator.ts
├── adapters/                    # Adaptadores de servicios
│   ├── user.adapter.ts
│   ├── order.adapter.ts
│   └── product.adapter.ts
├── transformers/                # Transformadores de datos
│   ├── user.transformer.ts
│   └── product.transformer.ts
├── cache/                       # Estrategias de cache
│   ├── redis.cache.ts
│   └── memory.cache.ts
├── controllers/                 # Controladores HTTP
│   └── bff.controller.ts
├── routes/                      # Definicion de rutas
│   └── bff.routes.ts
├── services/                    # Servicios internos
│   └── http-client.service.ts
└── utils/
    ├── logger.ts
    └── error-handler.ts
```

### Opciones del CLI

| Opcion | Descripcion |
|--------|-------------|
| `--services <list>` | Servicios a agregar |
| `--steps <list>` | Pasos del orquestador |
| `--cache <ttl>` | TTL de cache en segundos |
| `--client <type>` | Tipo de cliente (web, mobile, tv) |
| `--fields <list>` | Campos a incluir en transformacion |
| `--with-retry` | Incluir logica de retry |
| `--with-circuit-breaker` | Incluir circuit breaker |
| `--with-rollback` | Incluir logica de rollback |
| `--dry-run` | Preview sin crear archivos |
| `--force` | Sobrescribir archivos existentes |

## Arquitectura del BFF

### Patron Aggregator

```typescript
// Agrega datos de multiples servicios
class DashboardAggregator {
  async aggregate(userId: string): Promise<DashboardData> {
    const [user, orders, notifications] = await Promise.all([
      this.userAdapter.getById(userId),
      this.orderAdapter.getRecent(userId, 5),
      this.notificationAdapter.getUnread(userId)
    ]);

    return {
      user: this.userTransformer.toSummary(user),
      recentOrders: orders.map(o => this.orderTransformer.toCard(o)),
      notifications: notifications.length,
      lastLogin: user.lastLogin
    };
  }
}
```

### Patron Orchestrator

```typescript
// Orquesta flujos complejos con pasos secuenciales
class CheckoutOrchestrator {
  async execute(data: CheckoutInput): Promise<CheckoutResult> {
    const context = new OrchestrationContext(data);

    try {
      await this.validateCart(context);
      await this.reserveInventory(context);
      await this.processPayment(context);
      await this.createOrder(context);
      await this.sendConfirmation(context);

      return context.getResult();
    } catch (error) {
      await this.rollback(context);
      throw error;
    }
  }
}
```

### Patron Adapter

```typescript
// Abstrae la comunicacion con servicios externos
class UserAdapter {
  constructor(private httpClient: HttpClient) {}

  async getById(id: string): Promise<User> {
    const response = await this.httpClient.get(`/users/${id}`);
    return this.mapToUser(response.data);
  }

  private mapToUser(data: UserDTO): User {
    return {
      id: data.id,
      name: `${data.firstName} ${data.lastName}`,
      email: data.email,
      avatar: data.profileImage
    };
  }
}
```

### Patron Transformer

```typescript
// Transforma datos para clientes especificos
class ProductTransformer {
  toMobile(product: Product): MobileProduct {
    return {
      id: product.id,
      name: product.name,
      price: product.price,
      thumb: product.images[0]?.thumbnail // Solo thumbnail para mobile
    };
  }

  toWeb(product: Product): WebProduct {
    return {
      ...product,
      images: product.images.map(i => i.fullSize),
      relatedProducts: product.related?.slice(0, 4)
    };
  }
}
```

## Sub-Skills Disponibles

| Skill | Descripcion |
|-------|-------------|
| `/bff:aggregation` | Guia de agregacion de datos |
| `/bff:orchestration` | Guia de orquestacion de flujos |
| `/bff:caching` | Estrategias de caching |
| `/bff:adapters` | Patrones de adaptadores |
| `/bff:error-handling` | Manejo de errores y resiliencia |

## Patrones de Agregacion

### Agregacion Paralela

```typescript
// Ejecutar llamadas en paralelo para reducir latencia
const [users, products, orders] = await Promise.all([
  userAdapter.getAll(),
  productAdapter.getPopular(),
  orderAdapter.getRecent()
]);
```

### Agregacion con Fallback

```typescript
// Continuar si un servicio falla
const results = await Promise.allSettled([
  userAdapter.getById(id),
  recommendationAdapter.getForUser(id) // Opcional
]);

const user = results[0].status === 'fulfilled' ? results[0].value : null;
const recommendations = results[1].status === 'fulfilled' ? results[1].value : [];
```

### Agregacion Condicional

```typescript
// Agregar datos segun contexto
async aggregate(userId: string, options: AggregateOptions) {
  const data: DashboardData = {
    user: await this.userAdapter.getById(userId)
  };

  if (options.includeOrders) {
    data.orders = await this.orderAdapter.getByUser(userId);
  }

  if (options.includeRecommendations) {
    data.recommendations = await this.recommendationAdapter.get(userId);
  }

  return data;
}
```

## Estrategias de Caching

### Cache por Entidad

```json
{
  "cache": {
    "user": { "ttl": 300, "key": "user:{id}" },
    "products": { "ttl": 600, "key": "products:list" },
    "categories": { "ttl": 3600, "key": "categories:all" }
  }
}
```

### Cache con Invalidacion

```typescript
// Invalidar cache cuando cambian datos
async updateUser(id: string, data: UpdateUserDTO): Promise<User> {
  const user = await this.userAdapter.update(id, data);
  await this.cache.delete(`user:${id}`);
  await this.cache.delete(`dashboard:${id}`);
  return user;
}
```

## Stack Tecnologico

- **Express 4** - Framework HTTP
- **TypeScript 5** - Tipado estatico
- **Axios/undici** - HTTP client
- **Redis** - Caching distribuido
- **Pino** - Logging estructurado
- **Zod** - Validacion de datos

## Variables de Entorno

```env
# Server
PORT=3000
NODE_ENV=production

# Services
USER_SERVICE_URL=http://user-service:3001
ORDER_SERVICE_URL=http://order-service:3002
PRODUCT_SERVICE_URL=http://product-service:3003

# Cache
REDIS_URL=redis://localhost:6379
CACHE_DEFAULT_TTL=300

# Circuit Breaker
CB_FAILURE_THRESHOLD=5
CB_RESET_TIMEOUT=30000

# Logging
LOG_LEVEL=info
```

## Scripts Disponibles

```bash
pnpm dev           # Servidor de desarrollo
pnpm build         # Build de produccion
pnpm start         # Iniciar servidor
pnpm test          # Ejecutar tests
pnpm test:coverage # Tests con cobertura
pnpm lint          # Linter
```

## Documentacion Adicional

- `docs/aggregation-patterns.md` - Patrones de agregacion
- `docs/caching-strategies.md` - Estrategias de cache
- `skills/orchestration/SKILL.md` - Guia de orquestacion
- `workflows/create-aggregator.md` - Workflow de creacion
