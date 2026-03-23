---
name: bff:orchestration
description: Guia para implementar orquestacion de flujos en el BFF
---

# Orchestration Skill

Guia completa para implementar orquestacion de flujos complejos.

## Que es la Orquestacion?

La orquestacion coordina la ejecucion de multiples pasos en un flujo de negocio, manejando dependencias, errores y rollbacks.

```
Start
  │
  ▼
┌─────────────────┐
│  Validate Cart  │ ──► Fail ──► Error Response
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│Reserve Inventory│ ──► Fail ──► Rollback
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Process Payment │ ──► Fail ──► Rollback Inventory
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Create Order   │ ──► Fail ──► Refund + Rollback
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│Send Confirmation│
└────────┬────────┘
         │
         ▼
       Success
```

## Implementacion Base

### Orchestrator Context

```typescript
// src/orchestrators/context.ts
export interface OrchestrationStep {
  name: string;
  status: 'pending' | 'running' | 'completed' | 'failed' | 'rolledback';
  startTime?: Date;
  endTime?: Date;
  error?: Error;
  data?: any;
}

export class OrchestrationContext<T = any> {
  private readonly steps: Map<string, OrchestrationStep> = new Map();
  private readonly data: Map<string, any> = new Map();
  private readonly rollbackStack: Array<() => Promise<void>> = [];

  constructor(public readonly input: T) {}

  // Registrar paso
  registerStep(name: string): void {
    this.steps.set(name, { name, status: 'pending' });
  }

  // Iniciar paso
  startStep(name: string): void {
    const step = this.steps.get(name);
    if (step) {
      step.status = 'running';
      step.startTime = new Date();
    }
  }

  // Completar paso
  completeStep(name: string, data?: any): void {
    const step = this.steps.get(name);
    if (step) {
      step.status = 'completed';
      step.endTime = new Date();
      step.data = data;
    }
  }

  // Fallar paso
  failStep(name: string, error: Error): void {
    const step = this.steps.get(name);
    if (step) {
      step.status = 'failed';
      step.endTime = new Date();
      step.error = error;
    }
  }

  // Guardar datos entre pasos
  set<V>(key: string, value: V): void {
    this.data.set(key, value);
  }

  get<V>(key: string): V | undefined {
    return this.data.get(key) as V;
  }

  // Registrar accion de rollback
  addRollback(action: () => Promise<void>): void {
    this.rollbackStack.push(action);
  }

  // Ejecutar rollbacks en orden inverso
  async executeRollbacks(): Promise<void> {
    while (this.rollbackStack.length > 0) {
      const action = this.rollbackStack.pop()!;
      try {
        await action();
      } catch (error) {
        console.error('Rollback failed:', error);
      }
    }
  }

  // Obtener resumen
  getSummary(): { steps: OrchestrationStep[]; duration: number } {
    const steps = Array.from(this.steps.values());
    const firstStart = steps.find(s => s.startTime)?.startTime;
    const lastEnd = steps.reverse().find(s => s.endTime)?.endTime;

    return {
      steps,
      duration: firstStart && lastEnd
        ? lastEnd.getTime() - firstStart.getTime()
        : 0
    };
  }
}
```

### Base Orchestrator

```typescript
// src/orchestrators/base.orchestrator.ts
import { OrchestrationContext } from './context';
import { logger } from '../utils/logger';

export interface OrchestratorOptions {
  timeout?: number;
  retryOnFailure?: boolean;
  maxRetries?: number;
}

export abstract class BaseOrchestrator<TInput, TOutput> {
  protected readonly options: Required<OrchestratorOptions>;

  constructor(options: OrchestratorOptions = {}) {
    this.options = {
      timeout: options.timeout ?? 30000,
      retryOnFailure: options.retryOnFailure ?? false,
      maxRetries: options.maxRetries ?? 3
    };
  }

  abstract execute(input: TInput): Promise<TOutput>;

  protected async executeStep<T>(
    context: OrchestrationContext,
    stepName: string,
    action: () => Promise<T>,
    rollbackAction?: () => Promise<void>
  ): Promise<T> {
    context.startStep(stepName);

    try {
      const result = await action();
      context.completeStep(stepName, result);

      if (rollbackAction) {
        context.addRollback(rollbackAction);
      }

      logger.info({
        step: stepName,
        status: 'completed',
        duration: this.getStepDuration(context, stepName)
      });

      return result;
    } catch (error) {
      context.failStep(stepName, error as Error);

      logger.error({
        step: stepName,
        status: 'failed',
        error: (error as Error).message
      });

      throw error;
    }
  }

  private getStepDuration(context: OrchestrationContext, stepName: string): string {
    const summary = context.getSummary();
    const step = summary.steps.find(s => s.name === stepName);
    if (step?.startTime && step?.endTime) {
      return `${step.endTime.getTime() - step.startTime.getTime()}ms`;
    }
    return 'unknown';
  }
}
```

### Checkout Orchestrator

```typescript
// src/orchestrators/checkout.orchestrator.ts
import { BaseOrchestrator } from './base.orchestrator';
import { OrchestrationContext } from './context';
import { CartAdapter } from '../adapters/cart.adapter';
import { InventoryAdapter } from '../adapters/inventory.adapter';
import { PaymentAdapter } from '../adapters/payment.adapter';
import { OrderAdapter } from '../adapters/order.adapter';
import { NotificationAdapter } from '../adapters/notification.adapter';

interface CheckoutInput {
  userId: string;
  cartId: string;
  paymentMethod: {
    type: 'card' | 'paypal';
    token: string;
  };
  shippingAddress: Address;
}

interface CheckoutOutput {
  orderId: string;
  total: number;
  estimatedDelivery: Date;
  confirmationSent: boolean;
}

export class CheckoutOrchestrator extends BaseOrchestrator<CheckoutInput, CheckoutOutput> {
  constructor(
    private readonly cartAdapter: CartAdapter,
    private readonly inventoryAdapter: InventoryAdapter,
    private readonly paymentAdapter: PaymentAdapter,
    private readonly orderAdapter: OrderAdapter,
    private readonly notificationAdapter: NotificationAdapter
  ) {
    super({ timeout: 60000 });
  }

  async execute(input: CheckoutInput): Promise<CheckoutOutput> {
    const context = new OrchestrationContext(input);

    // Registrar pasos
    context.registerStep('validate-cart');
    context.registerStep('reserve-inventory');
    context.registerStep('process-payment');
    context.registerStep('create-order');
    context.registerStep('send-confirmation');

    try {
      // Paso 1: Validar carrito
      const cart = await this.executeStep(
        context,
        'validate-cart',
        () => this.validateCart(input.cartId)
      );
      context.set('cart', cart);

      // Paso 2: Reservar inventario
      const reservationId = await this.executeStep(
        context,
        'reserve-inventory',
        () => this.reserveInventory(cart.items),
        () => this.releaseInventory(context.get('reservationId')!) // Rollback
      );
      context.set('reservationId', reservationId);

      // Paso 3: Procesar pago
      const paymentResult = await this.executeStep(
        context,
        'process-payment',
        () => this.processPayment(cart.total, input.paymentMethod),
        () => this.refundPayment(context.get('paymentId')!) // Rollback
      );
      context.set('paymentId', paymentResult.paymentId);

      // Paso 4: Crear orden
      const order = await this.executeStep(
        context,
        'create-order',
        () => this.createOrder(input, cart, paymentResult)
      );
      context.set('orderId', order.id);

      // Paso 5: Enviar confirmacion (no critico)
      let confirmationSent = false;
      try {
        await this.executeStep(
          context,
          'send-confirmation',
          () => this.sendConfirmation(input.userId, order)
        );
        confirmationSent = true;
      } catch (error) {
        // Log pero no fallar el checkout
        logger.warn({ message: 'Failed to send confirmation', error });
      }

      return {
        orderId: order.id,
        total: order.total,
        estimatedDelivery: order.estimatedDelivery,
        confirmationSent
      };

    } catch (error) {
      // Ejecutar rollbacks en orden inverso
      await context.executeRollbacks();

      // Log del error con contexto
      const summary = context.getSummary();
      logger.error({
        orchestration: 'checkout',
        error: (error as Error).message,
        completedSteps: summary.steps.filter(s => s.status === 'completed').map(s => s.name),
        failedStep: summary.steps.find(s => s.status === 'failed')?.name
      });

      throw error;
    }
  }

  private async validateCart(cartId: string): Promise<Cart> {
    const cart = await this.cartAdapter.getById(cartId);

    if (!cart || cart.items.length === 0) {
      throw new Error('Cart is empty or not found');
    }

    // Validar precios actuales
    const validatedCart = await this.cartAdapter.validatePrices(cart);

    return validatedCart;
  }

  private async reserveInventory(items: CartItem[]): Promise<string> {
    const reservation = await this.inventoryAdapter.reserve(items);
    return reservation.id;
  }

  private async releaseInventory(reservationId: string): Promise<void> {
    await this.inventoryAdapter.release(reservationId);
  }

  private async processPayment(
    amount: number,
    paymentMethod: CheckoutInput['paymentMethod']
  ): Promise<PaymentResult> {
    return this.paymentAdapter.charge(amount, paymentMethod);
  }

  private async refundPayment(paymentId: string): Promise<void> {
    await this.paymentAdapter.refund(paymentId);
  }

  private async createOrder(
    input: CheckoutInput,
    cart: Cart,
    payment: PaymentResult
  ): Promise<Order> {
    return this.orderAdapter.create({
      userId: input.userId,
      items: cart.items,
      total: cart.total,
      paymentId: payment.paymentId,
      shippingAddress: input.shippingAddress
    });
  }

  private async sendConfirmation(userId: string, order: Order): Promise<void> {
    await this.notificationAdapter.sendOrderConfirmation(userId, order);
  }
}
```

## Patrones de Orquestacion

### 1. Saga Pattern

```typescript
// Cada paso tiene su compensacion
class SagaOrchestrator {
  private readonly steps: SagaStep[] = [];

  addStep(
    name: string,
    execute: () => Promise<void>,
    compensate: () => Promise<void>
  ): this {
    this.steps.push({ name, execute, compensate });
    return this;
  }

  async run(): Promise<void> {
    const completedSteps: SagaStep[] = [];

    for (const step of this.steps) {
      try {
        await step.execute();
        completedSteps.push(step);
      } catch (error) {
        // Compensar en orden inverso
        for (const completed of completedSteps.reverse()) {
          await completed.compensate();
        }
        throw error;
      }
    }
  }
}

// Uso
const saga = new SagaOrchestrator()
  .addStep(
    'reserve-inventory',
    () => inventoryAdapter.reserve(items),
    () => inventoryAdapter.release(reservationId)
  )
  .addStep(
    'charge-payment',
    () => paymentAdapter.charge(amount),
    () => paymentAdapter.refund(paymentId)
  )
  .addStep(
    'create-order',
    () => orderAdapter.create(order),
    () => orderAdapter.cancel(orderId)
  );

await saga.run();
```

### 2. Pipeline Pattern

```typescript
// Pasos en secuencia con transformacion
class PipelineOrchestrator<T> {
  private readonly steps: Array<(data: T) => Promise<T>> = [];

  pipe(step: (data: T) => Promise<T>): this {
    this.steps.push(step);
    return this;
  }

  async execute(initialData: T): Promise<T> {
    let data = initialData;

    for (const step of this.steps) {
      data = await step(data);
    }

    return data;
  }
}

// Uso
const result = await new PipelineOrchestrator<OrderData>()
  .pipe(async (data) => {
    data.validated = await validateOrder(data);
    return data;
  })
  .pipe(async (data) => {
    data.enriched = await enrichWithUserData(data);
    return data;
  })
  .pipe(async (data) => {
    data.pricing = await calculatePricing(data);
    return data;
  })
  .execute(initialOrderData);
```

### 3. State Machine Pattern

```typescript
// Estados y transiciones definidas
type OrderState = 'pending' | 'processing' | 'paid' | 'shipped' | 'delivered' | 'cancelled';

class OrderStateMachine {
  private state: OrderState = 'pending';

  private readonly transitions: Record<OrderState, OrderState[]> = {
    pending: ['processing', 'cancelled'],
    processing: ['paid', 'cancelled'],
    paid: ['shipped', 'cancelled'],
    shipped: ['delivered'],
    delivered: [],
    cancelled: []
  };

  canTransition(to: OrderState): boolean {
    return this.transitions[this.state].includes(to);
  }

  async transition(to: OrderState, action: () => Promise<void>): Promise<void> {
    if (!this.canTransition(to)) {
      throw new Error(`Cannot transition from ${this.state} to ${to}`);
    }

    await action();
    this.state = to;
  }

  getState(): OrderState {
    return this.state;
  }
}
```

## CLI para Generar Orquestadores

```bash
# Orquestador basico
bff g o checkout-flow

# Con pasos especificos
bff g o onboarding --steps "validate,create-user,send-email,setup-preferences"

# Con rollback
bff g o payment-process --with-rollback

# Con retry
bff g o import-data --with-retry --max-retries 3

# Preview
bff g o my-flow --dry-run
```

## Testing de Orquestadores

```typescript
describe('CheckoutOrchestrator', () => {
  it('should complete checkout successfully', async () => {
    // Arrange
    cartAdapter.getById.mockResolvedValue(mockCart);
    inventoryAdapter.reserve.mockResolvedValue({ id: 'res-123' });
    paymentAdapter.charge.mockResolvedValue({ paymentId: 'pay-123' });
    orderAdapter.create.mockResolvedValue(mockOrder);

    // Act
    const result = await orchestrator.execute(mockInput);

    // Assert
    expect(result.orderId).toBe(mockOrder.id);
    expect(inventoryAdapter.reserve).toHaveBeenCalled();
    expect(paymentAdapter.charge).toHaveBeenCalled();
  });

  it('should rollback on payment failure', async () => {
    // Arrange
    cartAdapter.getById.mockResolvedValue(mockCart);
    inventoryAdapter.reserve.mockResolvedValue({ id: 'res-123' });
    paymentAdapter.charge.mockRejectedValue(new Error('Payment failed'));

    // Act & Assert
    await expect(orchestrator.execute(mockInput)).rejects.toThrow('Payment failed');
    expect(inventoryAdapter.release).toHaveBeenCalledWith('res-123');
  });
});
```

## Mejores Practicas

1. **Idempotencia**: Cada paso debe ser idempotente para retries seguros
2. **Compensacion**: Siempre definir acciones de rollback
3. **Timeout**: Configurar timeouts por paso y total
4. **Logging**: Loguear inicio/fin de cada paso con duracion
5. **Monitoring**: Emitir metricas de exito/fallo
6. **Transacciones**: Usar transacciones donde sea posible
7. **Testing**: Probar escenarios de fallo y rollback
