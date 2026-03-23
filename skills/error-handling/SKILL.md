---
name: bff:error-handling
description: Guia para manejo de errores y resiliencia en el BFF
---

# Error Handling Skill

Guia completa para implementar manejo de errores robusto en el BFF.

## Arquitectura de Errores

```
Service Error ──► Adapter ──► Transform ──► BFF Error ──► Client Response
                     │
                     ▼
              Log + Metrics
```

## Jerarquia de Errores

```typescript
// src/errors/base.error.ts
export abstract class BFFError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;

  constructor(
    message: string,
    public readonly details?: Record<string, any>
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }

  toJSON(): object {
    return {
      error: this.name,
      code: this.code,
      message: this.message,
      details: this.details
    };
  }
}

// 4xx Errores de cliente
export class BadRequestError extends BFFError {
  readonly statusCode = 400;
  readonly code = 'BAD_REQUEST';
}

export class UnauthorizedError extends BFFError {
  readonly statusCode = 401;
  readonly code = 'UNAUTHORIZED';
}

export class ForbiddenError extends BFFError {
  readonly statusCode = 403;
  readonly code = 'FORBIDDEN';
}

export class NotFoundError extends BFFError {
  readonly statusCode = 404;
  readonly code = 'NOT_FOUND';
}

export class ConflictError extends BFFError {
  readonly statusCode = 409;
  readonly code = 'CONFLICT';
}

export class ValidationError extends BFFError {
  readonly statusCode = 422;
  readonly code = 'VALIDATION_ERROR';

  constructor(
    message: string,
    public readonly validationErrors: Array<{ field: string; message: string }>
  ) {
    super(message, { validationErrors });
  }
}

// 5xx Errores de servidor
export class ServiceError extends BFFError {
  readonly statusCode = 502;
  readonly code = 'SERVICE_ERROR';

  constructor(
    public readonly serviceName: string,
    message: string,
    public readonly originalStatus?: number
  ) {
    super(message, { serviceName, originalStatus });
  }
}

export class ServiceUnavailableError extends BFFError {
  readonly statusCode = 503;
  readonly code = 'SERVICE_UNAVAILABLE';
}

export class TimeoutError extends BFFError {
  readonly statusCode = 504;
  readonly code = 'TIMEOUT';
}

export class CircuitBreakerOpenError extends BFFError {
  readonly statusCode = 503;
  readonly code = 'CIRCUIT_BREAKER_OPEN';

  constructor(serviceName: string) {
    super(`Service ${serviceName} is temporarily unavailable`, { serviceName });
  }
}
```

## Error Handler Middleware

```typescript
// src/middleware/error-handler.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { BFFError } from '../errors/base.error';
import { logger } from '../utils/logger';

export const errorHandlerMiddleware = () => {
  return (
    error: Error,
    req: Request,
    res: Response,
    next: NextFunction
  ): void => {
    const correlationId = req.headers['x-correlation-id'] as string;

    // Log del error
    logger.error({
      correlationId,
      error: error.name,
      message: error.message,
      stack: error.stack,
      path: req.path,
      method: req.method
    });

    // Si es un error conocido del BFF
    if (error instanceof BFFError) {
      res.status(error.statusCode).json({
        ...error.toJSON(),
        correlationId
      });
      return;
    }

    // Error generico
    res.status(500).json({
      error: 'InternalServerError',
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
      correlationId
    });
  };
};
```

## Patrones de Resiliencia

### 1. Circuit Breaker

```typescript
// src/resilience/circuit-breaker.ts
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

export class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failures = 0;
  private successes = 0;
  private lastFailureTime = 0;

  constructor(
    private readonly options: {
      failureThreshold: number;
      successThreshold: number;
      resetTimeout: number;
    }
  ) {}

  async execute<T>(action: () => Promise<T>): Promise<T> {
    if (!this.canExecute()) {
      throw new CircuitBreakerOpenError('service');
    }

    try {
      const result = await action();
      this.recordSuccess();
      return result;
    } catch (error) {
      this.recordFailure();
      throw error;
    }
  }

  private canExecute(): boolean {
    if (this.state === 'CLOSED') return true;

    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime >= this.options.resetTimeout) {
        this.state = 'HALF_OPEN';
        return true;
      }
      return false;
    }

    return true;
  }

  private recordSuccess(): void {
    if (this.state === 'HALF_OPEN') {
      this.successes++;
      if (this.successes >= this.options.successThreshold) {
        this.reset();
      }
    }
    this.failures = 0;
  }

  private recordFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();

    if (this.failures >= this.options.failureThreshold) {
      this.state = 'OPEN';
    }
  }

  private reset(): void {
    this.state = 'CLOSED';
    this.failures = 0;
    this.successes = 0;
  }
}
```

### 2. Retry con Backoff

```typescript
// src/resilience/retry.ts
export interface RetryOptions {
  maxRetries: number;
  baseDelay: number;
  maxDelay: number;
  backoffFactor: number;
  retryOn?: (error: Error) => boolean;
}

export async function withRetry<T>(
  action: () => Promise<T>,
  options: RetryOptions
): Promise<T> {
  const {
    maxRetries,
    baseDelay,
    maxDelay,
    backoffFactor,
    retryOn = () => true
  } = options;

  let lastError: Error;
  let delay = baseDelay;

  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      return await action();
    } catch (error) {
      lastError = error as Error;

      if (attempt > maxRetries || !retryOn(lastError)) {
        throw lastError;
      }

      logger.warn({
        message: 'Retrying after error',
        attempt,
        maxRetries,
        delay,
        error: lastError.message
      });

      await sleep(delay);
      delay = Math.min(delay * backoffFactor, maxDelay);
    }
  }

  throw lastError!;
}

const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));
```

### 3. Timeout

```typescript
// src/resilience/timeout.ts
export async function withTimeout<T>(
  promise: Promise<T>,
  ms: number,
  message?: string
): Promise<T> {
  const timeout = new Promise<never>((_, reject) => {
    setTimeout(() => {
      reject(new TimeoutError(message || `Operation timed out after ${ms}ms`));
    }, ms);
  });

  return Promise.race([promise, timeout]);
}

// Uso
const user = await withTimeout(
  userAdapter.getById(userId),
  5000,
  'User service timeout'
);
```

### 4. Fallback

```typescript
// src/resilience/fallback.ts
export async function withFallback<T>(
  primary: () => Promise<T>,
  fallback: () => Promise<T> | T,
  onFallback?: (error: Error) => void
): Promise<T> {
  try {
    return await primary();
  } catch (error) {
    onFallback?.(error as Error);
    return fallback();
  }
}

// Uso
const recommendations = await withFallback(
  () => recommendationAdapter.getForUser(userId),
  () => [], // Fallback a array vacio
  (error) => logger.warn({ message: 'Using fallback', error: error.message })
);
```

### 5. Bulkhead (Isolation)

```typescript
// src/resilience/bulkhead.ts
export class Bulkhead {
  private activeCount = 0;
  private readonly queue: Array<{
    resolve: () => void;
    reject: (error: Error) => void;
  }> = [];

  constructor(
    private readonly maxConcurrent: number,
    private readonly maxQueue: number
  ) {}

  async execute<T>(action: () => Promise<T>): Promise<T> {
    if (this.activeCount >= this.maxConcurrent) {
      if (this.queue.length >= this.maxQueue) {
        throw new ServiceUnavailableError('Bulkhead queue full');
      }

      await new Promise<void>((resolve, reject) => {
        this.queue.push({ resolve, reject });
      });
    }

    this.activeCount++;

    try {
      return await action();
    } finally {
      this.activeCount--;

      if (this.queue.length > 0) {
        const next = this.queue.shift()!;
        next.resolve();
      }
    }
  }
}
```

## Manejo de Errores en Agregadores

```typescript
// src/aggregators/dashboard.aggregator.ts
async aggregate(userId: string): Promise<DashboardData> {
  // Datos criticos - fallan el request
  const [user, orders] = await Promise.all([
    this.userAdapter.getById(userId),
    this.orderAdapter.getByUser(userId)
  ]);

  // Datos opcionales - continuar con fallback
  const optionalResults = await Promise.allSettled([
    this.recommendationAdapter.getForUser(userId),
    this.notificationAdapter.getUnread(userId)
  ]);

  const recommendations = this.extractOrDefault(optionalResults[0], []);
  const notifications = this.extractOrDefault(optionalResults[1], []);

  return {
    user,
    orders,
    recommendations,
    notifications: {
      count: notifications.length,
      hasUnread: notifications.length > 0
    }
  };
}

private extractOrDefault<T>(
  result: PromiseSettledResult<T>,
  defaultValue: T
): T {
  if (result.status === 'fulfilled') {
    return result.value;
  }

  logger.warn({
    message: 'Optional service failed',
    error: result.reason?.message
  });

  return defaultValue;
}
```

## Graceful Degradation

```typescript
// Degradar funcionalidad cuando servicios fallan
async getDashboard(userId: string): Promise<DashboardData> {
  const dashboard: Partial<DashboardData> = {};
  const errors: string[] = [];

  // User - critico
  try {
    dashboard.user = await this.userAdapter.getById(userId);
  } catch (error) {
    throw new ServiceError('user-service', 'Failed to fetch user');
  }

  // Orders - importante pero no critico
  try {
    dashboard.orders = await this.orderAdapter.getRecent(userId, 5);
  } catch (error) {
    errors.push('orders');
    dashboard.orders = [];
  }

  // Recommendations - opcional
  try {
    dashboard.recommendations = await this.recommendationAdapter.get(userId);
  } catch (error) {
    errors.push('recommendations');
    dashboard.recommendations = [];
  }

  // Indicar que hay degradacion
  if (errors.length > 0) {
    dashboard.degraded = true;
    dashboard.unavailableFeatures = errors;
  }

  return dashboard as DashboardData;
}
```

## Logging de Errores

```typescript
// src/utils/error-logger.ts
export function logError(
  error: Error,
  context: {
    correlationId?: string;
    userId?: string;
    operation?: string;
    [key: string]: any;
  }
): void {
  const logData = {
    ...context,
    error: {
      name: error.name,
      message: error.message,
      stack: error.stack
    }
  };

  if (error instanceof BFFError) {
    logData.error = {
      ...logData.error,
      code: error.code,
      statusCode: error.statusCode,
      details: error.details
    };
  }

  logger.error(logData);
}
```

## Testing de Errores

```typescript
describe('Error Handling', () => {
  it('should return 404 for not found user', async () => {
    userAdapter.getById.mockRejectedValue(
      new NotFoundError('User not found')
    );

    const response = await request(app)
      .get('/api/users/123')
      .expect(404);

    expect(response.body.code).toBe('NOT_FOUND');
  });

  it('should handle circuit breaker open', async () => {
    userAdapter.getById.mockRejectedValue(
      new CircuitBreakerOpenError('user-service')
    );

    const response = await request(app)
      .get('/api/users/123')
      .expect(503);

    expect(response.body.code).toBe('CIRCUIT_BREAKER_OPEN');
  });

  it('should retry on transient errors', async () => {
    userAdapter.getById
      .mockRejectedValueOnce(new Error('Connection reset'))
      .mockRejectedValueOnce(new Error('Connection reset'))
      .mockResolvedValue(mockUser);

    const result = await getUserWithRetry('123');

    expect(result).toEqual(mockUser);
    expect(userAdapter.getById).toHaveBeenCalledTimes(3);
  });
});
```

## Mejores Practicas

1. **Errores Tipados**: Usar clases de error especificas
2. **Logging**: Siempre loguear errores con contexto
3. **No Exponer**: No exponer detalles internos al cliente
4. **Graceful Degradation**: Continuar con funcionalidad reducida
5. **Circuit Breaker**: Proteger contra cascading failures
6. **Retry**: Solo para errores transitorios
7. **Timeout**: Siempre configurar timeouts
8. **Monitoring**: Alertar en errores criticos
