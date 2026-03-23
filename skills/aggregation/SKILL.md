---
name: bff:aggregation
description: Guia para implementar patrones de agregacion de datos en el BFF
---

# Aggregation Skill

Guia completa para implementar agregacion de datos de multiples servicios.

## Que es la Agregacion?

La agregacion combina datos de multiples fuentes en una sola respuesta optimizada para el cliente.

```
Client Request: GET /api/dashboard
                     │
                     ▼
              ┌─────────────┐
              │  Aggregator │
              └──────┬──────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
     ▼               ▼               ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│  User   │    │ Orders  │    │  Notif  │
│ Service │    │ Service │    │ Service │
└─────────┘    └─────────┘    └─────────┘
     │               │               │
     └───────────────┼───────────────┘
                     │
                     ▼
              Combined Response
```

## Implementacion Base

### Aggregator Abstracto

```typescript
// src/aggregators/base.aggregator.ts
export interface AggregatorOptions {
  timeout?: number;
  cache?: {
    enabled: boolean;
    ttl: number;
  };
  parallel?: boolean;
}

export abstract class BaseAggregator<TInput, TOutput> {
  protected readonly options: Required<AggregatorOptions>;

  constructor(options: AggregatorOptions = {}) {
    this.options = {
      timeout: options.timeout ?? 5000,
      cache: options.cache ?? { enabled: false, ttl: 0 },
      parallel: options.parallel ?? true
    };
  }

  abstract aggregate(input: TInput): Promise<TOutput>;

  protected async withTimeout<T>(
    promise: Promise<T>,
    ms: number = this.options.timeout
  ): Promise<T> {
    const timeout = new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error('Aggregation timeout')), ms)
    );
    return Promise.race([promise, timeout]);
  }

  protected async parallel<T>(promises: Promise<T>[]): Promise<T[]> {
    return Promise.all(promises);
  }

  protected async parallelSettled<T>(
    promises: Promise<T>[]
  ): Promise<PromiseSettledResult<T>[]> {
    return Promise.allSettled(promises);
  }
}
```

### Dashboard Aggregator

```typescript
// src/aggregators/dashboard.aggregator.ts
import { BaseAggregator } from './base.aggregator';
import { UserAdapter } from '../adapters/user.adapter';
import { OrderAdapter } from '../adapters/order.adapter';
import { NotificationAdapter } from '../adapters/notification.adapter';
import { UserTransformer } from '../transformers/user.transformer';
import { OrderTransformer } from '../transformers/order.transformer';

interface DashboardInput {
  userId: string;
  includeOrders?: boolean;
  includeNotifications?: boolean;
}

interface DashboardOutput {
  user: UserSummary;
  stats: UserStats;
  recentOrders?: OrderCard[];
  notifications?: NotificationSummary;
}

export class DashboardAggregator extends BaseAggregator<DashboardInput, DashboardOutput> {
  constructor(
    private readonly userAdapter: UserAdapter,
    private readonly orderAdapter: OrderAdapter,
    private readonly notificationAdapter: NotificationAdapter,
    private readonly userTransformer: UserTransformer,
    private readonly orderTransformer: OrderTransformer
  ) {
    super({ timeout: 3000, parallel: true });
  }

  async aggregate(input: DashboardInput): Promise<DashboardOutput> {
    const { userId, includeOrders = true, includeNotifications = true } = input;

    // Datos obligatorios en paralelo
    const [user, stats] = await this.parallel([
      this.userAdapter.getById(userId),
      this.userAdapter.getStats(userId)
    ]);

    const result: DashboardOutput = {
      user: this.userTransformer.toSummary(user),
      stats
    };

    // Datos opcionales con fallback
    if (includeOrders || includeNotifications) {
      const optionalPromises: Promise<any>[] = [];

      if (includeOrders) {
        optionalPromises.push(this.orderAdapter.getRecent(userId, 5));
      }
      if (includeNotifications) {
        optionalPromises.push(this.notificationAdapter.getUnread(userId));
      }

      const settled = await this.parallelSettled(optionalPromises);
      let index = 0;

      if (includeOrders) {
        const ordersResult = settled[index++];
        if (ordersResult.status === 'fulfilled') {
          result.recentOrders = ordersResult.value.map(
            (o: Order) => this.orderTransformer.toCard(o)
          );
        }
      }

      if (includeNotifications) {
        const notifResult = settled[index++];
        if (notifResult.status === 'fulfilled') {
          result.notifications = {
            count: notifResult.value.length,
            hasUnread: notifResult.value.length > 0
          };
        }
      }
    }

    return result;
  }
}
```

## Patrones de Agregacion

### 1. Agregacion Paralela (Recomendado)

```typescript
// Todas las llamadas en paralelo - mejor latencia
async aggregateParallel(userId: string): Promise<AggregatedData> {
  const [user, orders, recommendations, notifications] = await Promise.all([
    this.userAdapter.getById(userId),
    this.orderAdapter.getByUser(userId),
    this.recommendationAdapter.getForUser(userId),
    this.notificationAdapter.getUnread(userId)
  ]);

  return { user, orders, recommendations, notifications };
}
```

### 2. Agregacion con Fallback

```typescript
// Continuar aunque algunos servicios fallen
async aggregateWithFallback(userId: string): Promise<AggregatedData> {
  const results = await Promise.allSettled([
    this.userAdapter.getById(userId),           // Critico
    this.orderAdapter.getByUser(userId),        // Critico
    this.recommendationAdapter.getForUser(userId), // Opcional
    this.notificationAdapter.getUnread(userId)    // Opcional
  ]);

  // Validar datos criticos
  if (results[0].status === 'rejected') {
    throw new Error('Failed to fetch user data');
  }
  if (results[1].status === 'rejected') {
    throw new Error('Failed to fetch orders');
  }

  return {
    user: results[0].value,
    orders: results[1].value,
    recommendations: results[2].status === 'fulfilled' ? results[2].value : [],
    notifications: results[3].status === 'fulfilled' ? results[3].value : []
  };
}
```

### 3. Agregacion Secuencial (Dependiente)

```typescript
// Cuando los datos dependen unos de otros
async aggregateSequential(userId: string): Promise<AggregatedData> {
  // 1. Obtener usuario primero
  const user = await this.userAdapter.getById(userId);

  // 2. Usar datos del usuario para siguientes llamadas
  const [orders, preferences] = await Promise.all([
    this.orderAdapter.getByUser(userId),
    this.preferenceAdapter.getByCountry(user.country)
  ]);

  // 3. Usar ordenes para recomendaciones
  const recommendations = await this.recommendationAdapter.getBasedOn(
    orders.map(o => o.productId)
  );

  return { user, orders, preferences, recommendations };
}
```

### 4. Agregacion Condicional

```typescript
// Agregar datos segun parametros
interface AggregateOptions {
  includeOrders?: boolean;
  includeRecommendations?: boolean;
  includeStats?: boolean;
}

async aggregateConditional(
  userId: string,
  options: AggregateOptions = {}
): Promise<Partial<AggregatedData>> {
  const data: Partial<AggregatedData> = {};

  // Siempre incluir usuario
  data.user = await this.userAdapter.getById(userId);

  // Construir promesas condicionales
  const conditionalPromises: { key: string; promise: Promise<any> }[] = [];

  if (options.includeOrders) {
    conditionalPromises.push({
      key: 'orders',
      promise: this.orderAdapter.getByUser(userId)
    });
  }

  if (options.includeRecommendations) {
    conditionalPromises.push({
      key: 'recommendations',
      promise: this.recommendationAdapter.getForUser(userId)
    });
  }

  if (options.includeStats) {
    conditionalPromises.push({
      key: 'stats',
      promise: this.statsAdapter.getForUser(userId)
    });
  }

  // Ejecutar en paralelo
  const results = await Promise.all(
    conditionalPromises.map(p => p.promise)
  );

  // Asignar resultados
  conditionalPromises.forEach((p, index) => {
    (data as any)[p.key] = results[index];
  });

  return data;
}
```

### 5. Agregacion con Cache

```typescript
async aggregateWithCache(userId: string): Promise<AggregatedData> {
  const cacheKey = `dashboard:${userId}`;

  // Intentar cache primero
  const cached = await this.cache.get<AggregatedData>(cacheKey);
  if (cached) {
    return cached;
  }

  // Agregar datos
  const data = await this.aggregateParallel(userId);

  // Guardar en cache
  await this.cache.set(cacheKey, data, 300); // 5 minutos

  return data;
}
```

## Agregacion para Diferentes Clientes

### Web Client

```typescript
// Datos completos para web
async aggregateForWeb(userId: string): Promise<WebDashboard> {
  const data = await this.aggregateParallel(userId);

  return {
    ...data,
    user: this.userTransformer.toWebProfile(data.user),
    orders: data.orders.map(o => this.orderTransformer.toWebCard(o)),
    // Incluir datos adicionales para web
    analytics: await this.analyticsAdapter.getForUser(userId)
  };
}
```

### Mobile Client

```typescript
// Datos optimizados para mobile
async aggregateForMobile(userId: string): Promise<MobileDashboard> {
  const [user, orders] = await Promise.all([
    this.userAdapter.getById(userId),
    this.orderAdapter.getRecent(userId, 3) // Solo ultimas 3
  ]);

  return {
    user: this.userTransformer.toMobileSummary(user), // Menos campos
    orders: orders.map(o => this.orderTransformer.toMobileCard(o)),
    // Omitir datos pesados para mobile
  };
}
```

## CLI para Generar Agregadores

```bash
# Agregador basico
bff g a dashboard

# Con servicios especificos
bff g a user-profile --services "user,orders,preferences"

# Con cache
bff g a product-catalog --cache 300

# Para cliente especifico
bff g a mobile-home --client mobile

# Preview
bff g a my-aggregator --dry-run
```

## Testing de Agregadores

```typescript
// src/aggregators/__tests__/dashboard.aggregator.test.ts
describe('DashboardAggregator', () => {
  let aggregator: DashboardAggregator;
  let userAdapter: jest.Mocked<UserAdapter>;
  let orderAdapter: jest.Mocked<OrderAdapter>;

  beforeEach(() => {
    userAdapter = createMock<UserAdapter>();
    orderAdapter = createMock<OrderAdapter>();

    aggregator = new DashboardAggregator(
      userAdapter,
      orderAdapter,
      // ... other adapters
    );
  });

  it('should aggregate data from all services', async () => {
    userAdapter.getById.mockResolvedValue(mockUser);
    orderAdapter.getRecent.mockResolvedValue(mockOrders);

    const result = await aggregator.aggregate({ userId: '123' });

    expect(result.user).toBeDefined();
    expect(result.recentOrders).toHaveLength(mockOrders.length);
  });

  it('should handle service failures gracefully', async () => {
    userAdapter.getById.mockResolvedValue(mockUser);
    orderAdapter.getRecent.mockRejectedValue(new Error('Service unavailable'));

    const result = await aggregator.aggregate({ userId: '123' });

    expect(result.user).toBeDefined();
    expect(result.recentOrders).toBeUndefined();
  });
});
```

## Mejores Practicas

1. **Paralelizar**: Siempre usar `Promise.all` cuando las llamadas son independientes
2. **Fallback**: Usar `Promise.allSettled` para datos no criticos
3. **Timeout**: Configurar timeouts para evitar bloqueos
4. **Cache**: Cachear agregaciones frecuentes
5. **Transformar**: Transformar datos al formato optimo del cliente
6. **Logging**: Loguear tiempos de agregacion para optimizar
7. **Testing**: Mockear adaptadores para tests unitarios
