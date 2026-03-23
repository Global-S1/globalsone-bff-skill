---
name: bff:caching
description: Guia para implementar estrategias de caching en el BFF
---

# Caching Skill

Guia completa para implementar estrategias de caching en el BFF.

## Por que Cachear en el BFF?

```
Sin Cache:
Client ──► BFF ──► Service A (100ms)
                ──► Service B (150ms)
                ──► Service C (80ms)
Total: ~330ms

Con Cache:
Client ──► BFF ──► Cache Hit (5ms)
Total: ~5ms
```

## Implementacion Base

### Cache Interface

```typescript
// src/cache/cache.interface.ts
export interface CacheOptions {
  ttl?: number;  // Time to live in seconds
  tags?: string[]; // Para invalidacion por grupo
}

export interface CacheService {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, options?: CacheOptions): Promise<void>;
  delete(key: string): Promise<void>;
  deleteByTag(tag: string): Promise<void>;
  exists(key: string): Promise<boolean>;
  ttl(key: string): Promise<number>;
}
```

### Redis Cache Implementation

```typescript
// src/cache/redis.cache.ts
import { createClient, RedisClientType } from 'redis';
import { CacheService, CacheOptions } from './cache.interface';
import { logger } from '../utils/logger';

export class RedisCache implements CacheService {
  private client: RedisClientType;
  private readonly defaultTTL: number;

  constructor(url: string, defaultTTL: number = 300) {
    this.client = createClient({ url });
    this.defaultTTL = defaultTTL;
  }

  async connect(): Promise<void> {
    await this.client.connect();
    logger.info('Redis cache connected');
  }

  async disconnect(): Promise<void> {
    await this.client.quit();
  }

  async get<T>(key: string): Promise<T | null> {
    try {
      const data = await this.client.get(key);
      if (!data) return null;

      return JSON.parse(data) as T;
    } catch (error) {
      logger.error({ error: 'Cache get failed', key });
      return null;
    }
  }

  async set<T>(key: string, value: T, options: CacheOptions = {}): Promise<void> {
    const ttl = options.ttl ?? this.defaultTTL;

    try {
      await this.client.setEx(key, ttl, JSON.stringify(value));

      // Guardar tags para invalidacion
      if (options.tags?.length) {
        for (const tag of options.tags) {
          await this.client.sAdd(`tag:${tag}`, key);
        }
      }
    } catch (error) {
      logger.error({ error: 'Cache set failed', key });
    }
  }

  async delete(key: string): Promise<void> {
    await this.client.del(key);
  }

  async deleteByTag(tag: string): Promise<void> {
    const keys = await this.client.sMembers(`tag:${tag}`);
    if (keys.length > 0) {
      await this.client.del(keys);
      await this.client.del(`tag:${tag}`);
    }
  }

  async exists(key: string): Promise<boolean> {
    const result = await this.client.exists(key);
    return result === 1;
  }

  async ttl(key: string): Promise<number> {
    return this.client.ttl(key);
  }
}
```

### Memory Cache (Desarrollo)

```typescript
// src/cache/memory.cache.ts
import { CacheService, CacheOptions } from './cache.interface';

interface CacheEntry<T> {
  value: T;
  expiresAt: number;
  tags: string[];
}

export class MemoryCache implements CacheService {
  private readonly cache = new Map<string, CacheEntry<any>>();
  private readonly defaultTTL: number;

  constructor(defaultTTL: number = 300) {
    this.defaultTTL = defaultTTL;

    // Limpiar entradas expiradas cada minuto
    setInterval(() => this.cleanup(), 60000);
  }

  async get<T>(key: string): Promise<T | null> {
    const entry = this.cache.get(key);

    if (!entry) return null;

    if (Date.now() > entry.expiresAt) {
      this.cache.delete(key);
      return null;
    }

    return entry.value as T;
  }

  async set<T>(key: string, value: T, options: CacheOptions = {}): Promise<void> {
    const ttl = options.ttl ?? this.defaultTTL;

    this.cache.set(key, {
      value,
      expiresAt: Date.now() + ttl * 1000,
      tags: options.tags || []
    });
  }

  async delete(key: string): Promise<void> {
    this.cache.delete(key);
  }

  async deleteByTag(tag: string): Promise<void> {
    for (const [key, entry] of this.cache.entries()) {
      if (entry.tags.includes(tag)) {
        this.cache.delete(key);
      }
    }
  }

  async exists(key: string): Promise<boolean> {
    const entry = this.cache.get(key);
    if (!entry) return false;
    return Date.now() <= entry.expiresAt;
  }

  async ttl(key: string): Promise<number> {
    const entry = this.cache.get(key);
    if (!entry) return -2;
    const remaining = Math.ceil((entry.expiresAt - Date.now()) / 1000);
    return remaining > 0 ? remaining : -1;
  }

  private cleanup(): void {
    const now = Date.now();
    for (const [key, entry] of this.cache.entries()) {
      if (now > entry.expiresAt) {
        this.cache.delete(key);
      }
    }
  }
}
```

## Estrategias de Caching

### 1. Cache-Aside (Lazy Loading)

```typescript
// Cargar en cache solo cuando se necesita
async getUser(userId: string): Promise<User> {
  const cacheKey = `user:${userId}`;

  // Intentar cache primero
  const cached = await this.cache.get<User>(cacheKey);
  if (cached) {
    return cached;
  }

  // Si no esta, obtener del servicio
  const user = await this.userAdapter.getById(userId);

  // Guardar en cache
  await this.cache.set(cacheKey, user, { ttl: 300 });

  return user;
}
```

### 2. Write-Through

```typescript
// Escribir en cache al actualizar
async updateUser(userId: string, data: UpdateUserDTO): Promise<User> {
  // Actualizar en servicio
  const user = await this.userAdapter.update(userId, data);

  // Actualizar cache
  await this.cache.set(`user:${userId}`, user, { ttl: 300 });

  return user;
}
```

### 3. Write-Behind (Async)

```typescript
// Actualizar cache inmediatamente, servicio despues
async updateUserPreferences(userId: string, prefs: Preferences): Promise<void> {
  const cacheKey = `prefs:${userId}`;

  // Actualizar cache inmediatamente
  await this.cache.set(cacheKey, prefs, { ttl: 3600 });

  // Actualizar servicio async (no bloquear respuesta)
  this.queue.add('sync-preferences', { userId, prefs });
}
```

### 4. Cache with Stale-While-Revalidate

```typescript
// Devolver cache aunque este expirado, actualizar en background
async getProducts(): Promise<Product[]> {
  const cacheKey = 'products:all';

  const cached = await this.cache.get<{ data: Product[]; fetchedAt: number }>(cacheKey);

  if (cached) {
    // Si tiene menos de 5 minutos, devolver directamente
    if (Date.now() - cached.fetchedAt < 300000) {
      return cached.data;
    }

    // Si tiene mas, devolver pero refrescar en background
    this.refreshProductsCache(); // No await
    return cached.data;
  }

  // No hay cache, obtener sync
  return this.refreshProductsCache();
}

private async refreshProductsCache(): Promise<Product[]> {
  const products = await this.productAdapter.getAll();
  await this.cache.set('products:all', {
    data: products,
    fetchedAt: Date.now()
  }, { ttl: 3600 }); // Cache por 1 hora pero refrescar cada 5 min
  return products;
}
```

## Cache Decorator

```typescript
// src/cache/cache.decorator.ts
export function Cached(options: CacheOptions & { keyPrefix?: string } = {}) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const cache = (this as any).cache as CacheService;
      const keyPrefix = options.keyPrefix || `${target.constructor.name}:${propertyKey}`;
      const cacheKey = `${keyPrefix}:${JSON.stringify(args)}`;

      // Intentar cache
      const cached = await cache.get(cacheKey);
      if (cached !== null) {
        return cached;
      }

      // Ejecutar metodo original
      const result = await originalMethod.apply(this, args);

      // Guardar en cache
      await cache.set(cacheKey, result, options);

      return result;
    };

    return descriptor;
  };
}

// Uso
class UserService {
  constructor(private cache: CacheService) {}

  @Cached({ ttl: 300, keyPrefix: 'user' })
  async getById(id: string): Promise<User> {
    return this.userAdapter.getById(id);
  }
}
```

## Invalidacion de Cache

### Por Key

```typescript
async updateUser(userId: string, data: UpdateUserDTO): Promise<User> {
  const user = await this.userAdapter.update(userId, data);

  // Invalidar cache especifico
  await this.cache.delete(`user:${userId}`);

  // Invalidar caches relacionados
  await this.cache.delete(`dashboard:${userId}`);

  return user;
}
```

### Por Tags

```typescript
// Al cachear, agregar tags
await this.cache.set(`user:${userId}`, user, {
  ttl: 300,
  tags: ['users', `user:${userId}`]
});

await this.cache.set(`dashboard:${userId}`, dashboard, {
  ttl: 300,
  tags: ['dashboards', `user:${userId}`]
});

// Invalidar todo lo relacionado al usuario
async deleteUser(userId: string): Promise<void> {
  await this.userAdapter.delete(userId);
  await this.cache.deleteByTag(`user:${userId}`);
}

// Invalidar todos los dashboards
async refreshAllDashboards(): Promise<void> {
  await this.cache.deleteByTag('dashboards');
}
```

### Event-Based

```typescript
// Escuchar eventos para invalidar
class CacheInvalidationService {
  constructor(
    private cache: CacheService,
    private eventBus: EventBus
  ) {
    this.setupListeners();
  }

  private setupListeners(): void {
    this.eventBus.on('user.updated', async (event) => {
      await this.cache.deleteByTag(`user:${event.userId}`);
    });

    this.eventBus.on('order.created', async (event) => {
      await this.cache.delete(`orders:${event.userId}:recent`);
    });

    this.eventBus.on('product.priceChanged', async (event) => {
      await this.cache.deleteByTag('products');
    });
  }
}
```

## Configuracion de Cache

```typescript
// src/config/cache.config.ts
export const cacheConfig = {
  // TTLs por tipo de dato
  ttl: {
    user: 300,          // 5 minutos
    userPreferences: 3600, // 1 hora
    products: 600,      // 10 minutos
    categories: 86400,  // 1 dia
    dashboard: 60       // 1 minuto
  },

  // Keys patterns
  keys: {
    user: (id: string) => `user:${id}`,
    userPrefs: (id: string) => `user:${id}:prefs`,
    dashboard: (id: string) => `dashboard:${id}`,
    products: () => 'products:all',
    productById: (id: string) => `product:${id}`
  }
};
```

## CLI para Generar Cache

```bash
# Cache basico
bff g c user-cache

# Con estrategia especifica
bff g c product-cache --strategy stale-while-revalidate

# Con TTL
bff g c session-cache --ttl 3600

# Con tags
bff g c order-cache --tags "orders,user-data"

# Preview
bff g c my-cache --dry-run
```

## Metricas de Cache

```typescript
class CacheMetrics {
  private hits = 0;
  private misses = 0;

  recordHit(): void {
    this.hits++;
  }

  recordMiss(): void {
    this.misses++;
  }

  getStats(): { hits: number; misses: number; hitRate: number } {
    const total = this.hits + this.misses;
    return {
      hits: this.hits,
      misses: this.misses,
      hitRate: total > 0 ? this.hits / total : 0
    };
  }
}
```

## Mejores Practicas

1. **TTL Apropiado**: Configurar TTL segun frecuencia de cambio
2. **Keys Consistentes**: Usar patron consistente para cache keys
3. **Invalidacion**: Siempre invalidar al actualizar datos
4. **Fallback**: Si cache falla, obtener del servicio
5. **Metricas**: Monitorear hit rate y ajustar estrategia
6. **Serializacion**: Usar JSON para compatibilidad
7. **Compresion**: Comprimir datos grandes
8. **Tags**: Usar tags para invalidacion en grupo
