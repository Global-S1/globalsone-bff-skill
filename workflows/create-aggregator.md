# Workflow: Crear Agregador

Guia paso a paso para crear un nuevo agregador en el BFF.

## Prerequisitos

- BFF configurado y funcionando
- Servicios backend disponibles
- CLI instalado: `npm install -g globalsone-bff-cli`

## Pasos

### 1. Generar Agregador con CLI

```bash
# Agregador basico
bff g a dashboard

# Con servicios especificos
bff g a user-profile --services "user,orders,preferences"
```

### 2. Revisar Estructura Generada

```
src/aggregators/
├── dashboard.aggregator.ts      # Agregador generado
└── __tests__/
    └── dashboard.aggregator.test.ts
```

### 3. Implementar Logica de Agregacion

```typescript
// src/aggregators/dashboard.aggregator.ts
import { BaseAggregator } from './base.aggregator';
import { UserAdapter } from '../adapters/user.adapter';
import { OrderAdapter } from '../adapters/order.adapter';

interface DashboardInput {
  userId: string;
}

interface DashboardOutput {
  user: UserSummary;
  recentOrders: OrderCard[];
  stats: UserStats;
}

export class DashboardAggregator extends BaseAggregator<DashboardInput, DashboardOutput> {
  constructor(
    private readonly userAdapter: UserAdapter,
    private readonly orderAdapter: OrderAdapter
  ) {
    super({ timeout: 5000 });
  }

  async aggregate(input: DashboardInput): Promise<DashboardOutput> {
    const { userId } = input;

    // Llamadas en paralelo
    const [user, orders, stats] = await this.parallel([
      this.userAdapter.getById(userId),
      this.orderAdapter.getRecent(userId, 5),
      this.userAdapter.getStats(userId)
    ]);

    return {
      user: this.userTransformer.toSummary(user),
      recentOrders: orders.map(o => this.orderTransformer.toCard(o)),
      stats
    };
  }
}
```

### 4. Crear Adaptadores Necesarios

```bash
# Si no existen
bff g ad user-service --base-url "http://localhost:3001"
bff g ad order-service --base-url "http://localhost:3002"
```

### 5. Crear Endpoint

```typescript
// src/controllers/bff.controller.ts
import { DashboardAggregator } from '../aggregators/dashboard.aggregator';

export class BFFController {
  constructor(private readonly dashboardAggregator: DashboardAggregator) {}

  async getDashboard(req: Request, res: Response): Promise<void> {
    const userId = req.user.id;

    const dashboard = await this.dashboardAggregator.aggregate({ userId });

    res.json(dashboard);
  }
}
```

### 6. Agregar Ruta

```typescript
// src/routes/bff.routes.ts
router.get('/dashboard', authMiddleware(), controller.getDashboard);
```

### 7. Agregar Tests

```typescript
// src/aggregators/__tests__/dashboard.aggregator.test.ts
describe('DashboardAggregator', () => {
  it('should aggregate data from all services', async () => {
    userAdapter.getById.mockResolvedValue(mockUser);
    orderAdapter.getRecent.mockResolvedValue(mockOrders);

    const result = await aggregator.aggregate({ userId: '123' });

    expect(result.user).toBeDefined();
    expect(result.recentOrders).toHaveLength(mockOrders.length);
  });
});
```

### 8. Probar

```bash
# Obtener token
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"password"}' | jq -r '.token')

# Probar agregador
curl http://localhost:3000/api/bff/dashboard \
  -H "Authorization: Bearer $TOKEN"
```

## Checklist

- [ ] Agregador generado con CLI
- [ ] Adaptadores creados/configurados
- [ ] Logica de agregacion implementada
- [ ] Transformadores aplicados
- [ ] Endpoint creado
- [ ] Ruta agregada
- [ ] Tests escritos
- [ ] Probado manualmente

## Troubleshooting

### Timeout en agregacion

1. Verificar servicios backend estan corriendo
2. Aumentar timeout del agregador
3. Verificar conectividad de red

### Datos incompletos

1. Verificar que todos los adaptadores estan configurados
2. Revisar logs de errores
3. Verificar transformadores
