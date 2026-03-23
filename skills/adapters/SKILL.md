---
name: bff:adapters
description: Guia para implementar adaptadores de servicios en el BFF
---

# Adapters Skill

Guia completa para implementar adaptadores de servicios externos.

## Que es un Adapter?

Un adapter abstrae la comunicacion con servicios externos, manejando:

- Transformacion de datos (DTO → Domain Model)
- Manejo de errores
- Retry y circuit breaker
- Logging y metricas

```
BFF Layer                    External Services
┌─────────────┐              ┌─────────────┐
│  Aggregator │              │    User     │
│             │──► Adapter ──│   Service   │
│             │              └─────────────┘
│             │              ┌─────────────┐
│             │──► Adapter ──│   Order     │
│             │              │   Service   │
└─────────────┘              └─────────────┘
```

## Implementacion Base

### Base Adapter

```typescript
// src/adapters/base.adapter.ts
import { HttpClient, HttpClientOptions } from '../services/http-client.service';
import { logger } from '../utils/logger';

export interface AdapterOptions {
  baseUrl: string;
  timeout?: number;
  retries?: number;
  circuitBreaker?: boolean;
}

export abstract class BaseAdapter {
  protected readonly http: HttpClient;
  protected readonly serviceName: string;

  constructor(serviceName: string, options: AdapterOptions) {
    this.serviceName = serviceName;
    this.http = new HttpClient({
      baseURL: options.baseUrl,
      timeout: options.timeout ?? 5000,
      retries: options.retries ?? 3,
      circuitBreaker: options.circuitBreaker ?? true
    });
  }

  protected async get<T>(path: string, params?: Record<string, any>): Promise<T> {
    const startTime = Date.now();

    try {
      const response = await this.http.get<T>(path, { params });

      this.logRequest('GET', path, Date.now() - startTime, 200);

      return response.data;
    } catch (error) {
      this.logRequest('GET', path, Date.now() - startTime, this.getStatusCode(error));
      throw this.handleError(error);
    }
  }

  protected async post<T>(path: string, data: any): Promise<T> {
    const startTime = Date.now();

    try {
      const response = await this.http.post<T>(path, data);

      this.logRequest('POST', path, Date.now() - startTime, 200);

      return response.data;
    } catch (error) {
      this.logRequest('POST', path, Date.now() - startTime, this.getStatusCode(error));
      throw this.handleError(error);
    }
  }

  protected async put<T>(path: string, data: any): Promise<T> {
    const startTime = Date.now();

    try {
      const response = await this.http.put<T>(path, data);

      this.logRequest('PUT', path, Date.now() - startTime, 200);

      return response.data;
    } catch (error) {
      this.logRequest('PUT', path, Date.now() - startTime, this.getStatusCode(error));
      throw this.handleError(error);
    }
  }

  protected async delete(path: string): Promise<void> {
    const startTime = Date.now();

    try {
      await this.http.delete(path);

      this.logRequest('DELETE', path, Date.now() - startTime, 200);
    } catch (error) {
      this.logRequest('DELETE', path, Date.now() - startTime, this.getStatusCode(error));
      throw this.handleError(error);
    }
  }

  private logRequest(method: string, path: string, duration: number, status: number): void {
    logger.info({
      service: this.serviceName,
      method,
      path,
      duration: `${duration}ms`,
      status
    });
  }

  private getStatusCode(error: any): number {
    return error?.response?.status || 500;
  }

  protected handleError(error: any): Error {
    const status = error?.response?.status;
    const message = error?.response?.data?.message || error.message;

    if (status === 404) {
      return new NotFoundError(`${this.serviceName}: ${message}`);
    }

    if (status === 400) {
      return new ValidationError(`${this.serviceName}: ${message}`);
    }

    if (status === 401 || status === 403) {
      return new UnauthorizedError(`${this.serviceName}: ${message}`);
    }

    return new ServiceError(this.serviceName, message, status);
  }
}
```

### User Adapter

```typescript
// src/adapters/user.adapter.ts
import { BaseAdapter } from './base.adapter';

// DTOs del servicio externo
interface UserDTO {
  id: string;
  first_name: string;
  last_name: string;
  email_address: string;
  profile_image_url: string;
  created_at: string;
  preferences: {
    theme: string;
    notifications_enabled: boolean;
  };
}

// Modelo de dominio del BFF
interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
  createdAt: Date;
  preferences: UserPreferences;
}

interface UserPreferences {
  theme: 'light' | 'dark';
  notificationsEnabled: boolean;
}

export class UserAdapter extends BaseAdapter {
  constructor() {
    super('user-service', {
      baseUrl: process.env.USER_SERVICE_URL || 'http://localhost:3001',
      timeout: 5000,
      retries: 3
    });
  }

  async getById(id: string): Promise<User> {
    const dto = await this.get<UserDTO>(`/users/${id}`);
    return this.mapToUser(dto);
  }

  async getAll(params?: { page?: number; limit?: number }): Promise<User[]> {
    const dtos = await this.get<UserDTO[]>('/users', params);
    return dtos.map(dto => this.mapToUser(dto));
  }

  async create(data: CreateUserInput): Promise<User> {
    const dto = await this.post<UserDTO>('/users', this.mapToDTO(data));
    return this.mapToUser(dto);
  }

  async update(id: string, data: UpdateUserInput): Promise<User> {
    const dto = await this.put<UserDTO>(`/users/${id}`, this.mapToDTO(data));
    return this.mapToUser(dto);
  }

  async delete(id: string): Promise<void> {
    await this.delete(`/users/${id}`);
  }

  async getStats(id: string): Promise<UserStats> {
    const stats = await this.get<UserStatsDTO>(`/users/${id}/stats`);
    return this.mapToStats(stats);
  }

  // Mappers
  private mapToUser(dto: UserDTO): User {
    return {
      id: dto.id,
      name: `${dto.first_name} ${dto.last_name}`,
      email: dto.email_address,
      avatar: dto.profile_image_url,
      createdAt: new Date(dto.created_at),
      preferences: {
        theme: dto.preferences.theme as 'light' | 'dark',
        notificationsEnabled: dto.preferences.notifications_enabled
      }
    };
  }

  private mapToDTO(input: CreateUserInput | UpdateUserInput): Partial<UserDTO> {
    const dto: Partial<UserDTO> = {};

    if ('firstName' in input && input.firstName) {
      dto.first_name = input.firstName;
    }
    if ('lastName' in input && input.lastName) {
      dto.last_name = input.lastName;
    }
    if ('email' in input && input.email) {
      dto.email_address = input.email;
    }

    return dto;
  }
}
```

### Order Adapter

```typescript
// src/adapters/order.adapter.ts
import { BaseAdapter } from './base.adapter';

interface OrderDTO {
  id: string;
  user_id: string;
  items: OrderItemDTO[];
  total_amount: number;
  status: string;
  created_at: string;
  shipping_address: AddressDTO;
}

interface Order {
  id: string;
  userId: string;
  items: OrderItem[];
  total: number;
  status: OrderStatus;
  createdAt: Date;
  shippingAddress: Address;
}

export class OrderAdapter extends BaseAdapter {
  constructor() {
    super('order-service', {
      baseUrl: process.env.ORDER_SERVICE_URL || 'http://localhost:3002',
      timeout: 10000,
      retries: 2
    });
  }

  async getById(id: string): Promise<Order> {
    const dto = await this.get<OrderDTO>(`/orders/${id}`);
    return this.mapToOrder(dto);
  }

  async getByUser(userId: string, params?: { limit?: number }): Promise<Order[]> {
    const dtos = await this.get<OrderDTO[]>(`/users/${userId}/orders`, params);
    return dtos.map(dto => this.mapToOrder(dto));
  }

  async getRecent(userId: string, limit: number = 5): Promise<Order[]> {
    return this.getByUser(userId, { limit });
  }

  async create(data: CreateOrderInput): Promise<Order> {
    const dto = await this.post<OrderDTO>('/orders', this.mapToCreateDTO(data));
    return this.mapToOrder(dto);
  }

  async updateStatus(id: string, status: OrderStatus): Promise<Order> {
    const dto = await this.put<OrderDTO>(`/orders/${id}/status`, { status });
    return this.mapToOrder(dto);
  }

  async cancel(id: string): Promise<void> {
    await this.post(`/orders/${id}/cancel`, {});
  }

  // Mappers
  private mapToOrder(dto: OrderDTO): Order {
    return {
      id: dto.id,
      userId: dto.user_id,
      items: dto.items.map(item => this.mapToOrderItem(item)),
      total: dto.total_amount,
      status: dto.status as OrderStatus,
      createdAt: new Date(dto.created_at),
      shippingAddress: this.mapToAddress(dto.shipping_address)
    };
  }

  private mapToOrderItem(dto: OrderItemDTO): OrderItem {
    return {
      productId: dto.product_id,
      name: dto.product_name,
      quantity: dto.quantity,
      price: dto.unit_price,
      subtotal: dto.quantity * dto.unit_price
    };
  }

  private mapToAddress(dto: AddressDTO): Address {
    return {
      street: dto.street_address,
      city: dto.city,
      state: dto.state,
      country: dto.country,
      zipCode: dto.postal_code
    };
  }

  private mapToCreateDTO(input: CreateOrderInput): any {
    return {
      user_id: input.userId,
      items: input.items.map(item => ({
        product_id: item.productId,
        quantity: item.quantity
      })),
      shipping_address: {
        street_address: input.shippingAddress.street,
        city: input.shippingAddress.city,
        state: input.shippingAddress.state,
        country: input.shippingAddress.country,
        postal_code: input.shippingAddress.zipCode
      }
    };
  }
}
```

## Patrones Avanzados

### Adapter con Circuit Breaker

```typescript
export class ResilientAdapter extends BaseAdapter {
  private readonly circuitBreaker: CircuitBreaker;

  constructor() {
    super('resilient-service', {
      baseUrl: process.env.SERVICE_URL,
      circuitBreaker: true
    });

    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,
      resetTimeout: 30000
    });
  }

  protected async get<T>(path: string): Promise<T> {
    if (!this.circuitBreaker.canExecute()) {
      throw new ServiceUnavailableError('Circuit breaker is open');
    }

    try {
      const result = await super.get<T>(path);
      this.circuitBreaker.recordSuccess();
      return result;
    } catch (error) {
      this.circuitBreaker.recordFailure();
      throw error;
    }
  }
}
```

### Adapter con Retry

```typescript
export class RetryableAdapter extends BaseAdapter {
  private readonly maxRetries: number;
  private readonly retryDelay: number;

  constructor() {
    super('retryable-service', {
      baseUrl: process.env.SERVICE_URL
    });

    this.maxRetries = 3;
    this.retryDelay = 1000;
  }

  protected async get<T>(path: string): Promise<T> {
    let lastError: Error;

    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await super.get<T>(path);
      } catch (error) {
        lastError = error as Error;

        // No reintentar errores de cliente (4xx)
        if (this.isClientError(error)) {
          throw error;
        }

        if (attempt < this.maxRetries) {
          await this.delay(this.retryDelay * attempt);
        }
      }
    }

    throw lastError!;
  }

  private isClientError(error: any): boolean {
    const status = error?.response?.status;
    return status >= 400 && status < 500;
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### Adapter Factory

```typescript
// src/adapters/adapter.factory.ts
export class AdapterFactory {
  private static instances: Map<string, BaseAdapter> = new Map();

  static get<T extends BaseAdapter>(
    AdapterClass: new () => T,
    serviceName: string
  ): T {
    if (!this.instances.has(serviceName)) {
      this.instances.set(serviceName, new AdapterClass());
    }
    return this.instances.get(serviceName) as T;
  }

  static getUserAdapter(): UserAdapter {
    return this.get(UserAdapter, 'user');
  }

  static getOrderAdapter(): OrderAdapter {
    return this.get(OrderAdapter, 'order');
  }

  static getProductAdapter(): ProductAdapter {
    return this.get(ProductAdapter, 'product');
  }
}
```

## CLI para Generar Adapters

```bash
# Adapter basico
bff g ad user-service --base-url "http://localhost:3001"

# Con metodos especificos
bff g ad order-service --methods "getById,getByUser,create,updateStatus"

# Con retry y circuit breaker
bff g ad payment-service --with-retry --with-circuit-breaker

# Con timeout custom
bff g ad slow-service --timeout 30000

# Preview
bff g ad my-service --dry-run
```

## Testing de Adapters

```typescript
describe('UserAdapter', () => {
  let adapter: UserAdapter;
  let httpMock: jest.Mocked<HttpClient>;

  beforeEach(() => {
    httpMock = createMock<HttpClient>();
    adapter = new UserAdapter();
    (adapter as any).http = httpMock;
  });

  describe('getById', () => {
    it('should return mapped user', async () => {
      httpMock.get.mockResolvedValue({
        data: {
          id: '123',
          first_name: 'John',
          last_name: 'Doe',
          email_address: 'john@example.com'
        }
      });

      const user = await adapter.getById('123');

      expect(user.name).toBe('John Doe');
      expect(user.email).toBe('john@example.com');
    });

    it('should throw NotFoundError for 404', async () => {
      httpMock.get.mockRejectedValue({
        response: { status: 404, data: { message: 'User not found' } }
      });

      await expect(adapter.getById('123')).rejects.toThrow(NotFoundError);
    });
  });
});
```

## Mejores Practicas

1. **Single Responsibility**: Un adapter por servicio
2. **Mapping**: Siempre mapear DTOs a modelos de dominio
3. **Error Handling**: Transformar errores HTTP a errores de dominio
4. **Logging**: Loguear todas las llamadas con tiempos
5. **Timeout**: Configurar timeouts apropiados
6. **Retry**: Implementar retry para errores transitorios
7. **Circuit Breaker**: Proteger contra servicios caidos
8. **Testing**: Mockear HTTP client para tests unitarios
