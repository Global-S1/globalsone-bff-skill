# Workflow: Crear Orquestador

Guia paso a paso para crear un nuevo orquestador de flujo en el BFF.

## Prerequisitos

- BFF configurado y funcionando
- Servicios backend disponibles
- CLI instalado: `npm install -g globalsone-bff-cli`

## Pasos

### 1. Generar Orquestador con CLI

```bash
# Orquestador basico
bff g o checkout-flow

# Con pasos especificos
bff g o onboarding --steps "validate,create-user,send-email,setup-preferences"

# Con rollback
bff g o payment-process --with-rollback
```

### 2. Revisar Estructura Generada

```
src/orchestrators/
├── checkout.orchestrator.ts     # Orquestador generado
├── context.ts                   # Contexto de orquestacion
└── __tests__/
    └── checkout.orchestrator.test.ts
```

### 3. Definir Input y Output

```typescript
// src/orchestrators/checkout.orchestrator.ts
interface CheckoutInput {
  userId: string;
  cartId: string;
  paymentMethod: PaymentMethod;
  shippingAddress: Address;
}

interface CheckoutOutput {
  orderId: string;
  total: number;
  estimatedDelivery: Date;
}
```

### 4. Implementar Pasos

```typescript
export class CheckoutOrchestrator extends BaseOrchestrator<CheckoutInput, CheckoutOutput> {
  async execute(input: CheckoutInput): Promise<CheckoutOutput> {
    const context = new OrchestrationContext(input);

    try {
      // Paso 1: Validar carrito
      const cart = await this.executeStep(
        context,
        'validate-cart',
        () => this.validateCart(input.cartId)
      );

      // Paso 2: Reservar inventario (con rollback)
      const reservationId = await this.executeStep(
        context,
        'reserve-inventory',
        () => this.inventoryAdapter.reserve(cart.items),
        () => this.inventoryAdapter.release(context.get('reservationId'))
      );
      context.set('reservationId', reservationId);

      // Paso 3: Procesar pago (con rollback)
      const payment = await this.executeStep(
        context,
        'process-payment',
        () => this.paymentAdapter.charge(cart.total, input.paymentMethod),
        () => this.paymentAdapter.refund(context.get('paymentId'))
      );
      context.set('paymentId', payment.id);

      // Paso 4: Crear orden
      const order = await this.executeStep(
        context,
        'create-order',
        () => this.orderAdapter.create({
          userId: input.userId,
          items: cart.items,
          total: cart.total,
          paymentId: payment.id,
          shippingAddress: input.shippingAddress
        })
      );

      // Paso 5: Enviar confirmacion (no critico)
      await this.executeStep(
        context,
        'send-confirmation',
        () => this.notificationAdapter.sendOrderConfirmation(input.userId, order)
      ).catch(() => { /* Log but don't fail */ });

      return {
        orderId: order.id,
        total: order.total,
        estimatedDelivery: order.estimatedDelivery
      };

    } catch (error) {
      // Ejecutar rollbacks
      await context.executeRollbacks();
      throw error;
    }
  }
}
```

### 5. Crear Adaptadores Necesarios

```bash
bff g ad cart-service --base-url "http://localhost:3001"
bff g ad inventory-service --base-url "http://localhost:3002"
bff g ad payment-service --base-url "http://localhost:3003" --with-circuit-breaker
bff g ad order-service --base-url "http://localhost:3004"
```

### 6. Crear Endpoint

```typescript
// src/controllers/bff.controller.ts
async checkout(req: Request, res: Response): Promise<void> {
  const input: CheckoutInput = {
    userId: req.user.id,
    cartId: req.body.cartId,
    paymentMethod: req.body.paymentMethod,
    shippingAddress: req.body.shippingAddress
  };

  const result = await this.checkoutOrchestrator.execute(input);

  res.status(201).json(result);
}
```

### 7. Agregar Ruta

```typescript
router.post('/checkout', authMiddleware(), validateBody(checkoutSchema), controller.checkout);
```

### 8. Agregar Tests

```typescript
describe('CheckoutOrchestrator', () => {
  it('should complete checkout successfully', async () => {
    cartAdapter.getById.mockResolvedValue(mockCart);
    inventoryAdapter.reserve.mockResolvedValue({ id: 'res-123' });
    paymentAdapter.charge.mockResolvedValue({ id: 'pay-123' });
    orderAdapter.create.mockResolvedValue(mockOrder);

    const result = await orchestrator.execute(mockInput);

    expect(result.orderId).toBe(mockOrder.id);
  });

  it('should rollback on payment failure', async () => {
    cartAdapter.getById.mockResolvedValue(mockCart);
    inventoryAdapter.reserve.mockResolvedValue({ id: 'res-123' });
    paymentAdapter.charge.mockRejectedValue(new Error('Payment failed'));

    await expect(orchestrator.execute(mockInput)).rejects.toThrow();

    // Verificar rollback
    expect(inventoryAdapter.release).toHaveBeenCalledWith('res-123');
  });
});
```

## Checklist

- [ ] Orquestador generado con CLI
- [ ] Input/Output definidos
- [ ] Pasos implementados
- [ ] Rollbacks configurados para pasos criticos
- [ ] Adaptadores creados
- [ ] Endpoint creado
- [ ] Validacion de request
- [ ] Tests de exito y rollback
- [ ] Logging en cada paso

## Mejores Practicas

1. **Idempotencia**: Cada paso debe ser idempotente
2. **Rollback**: Siempre definir rollback para operaciones con estado
3. **Orden**: Ejecutar operaciones reversibles antes de irreversibles
4. **Logging**: Loguear inicio/fin de cada paso
5. **Timeout**: Configurar timeout total del flujo
