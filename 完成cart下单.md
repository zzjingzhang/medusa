# "完成 Cart 下单" 为何是一个典型的跨系统多模块流程

## 1. 流程入口与整体架构

### 1.1 API 入口

流程从 `/store/carts/:id/complete` 的 POST 请求开始：

```typescript
// packages/medusa/src/api/store/carts/[id]/complete/route.ts:13-23
export const POST = async (
  req: MedusaRequest<{}, HttpTypes.SelectParams>,
  res: MedusaResponse<HttpTypes.StoreCompleteCartResponse>
) => {
  const cart_id = req.params.id
  const we = req.scope.resolve(Modules.WORKFLOW_ENGINE)

  const { errors, result, transaction } = await we.run(completeCartWorkflowId, {
    input: { id: cart_id },
    throwOnError: false,
  })
  // ...
}
```

### 1.2 涉及的核心模块

| 模块 | 职责 |
|------|------|
| `WORKFLOW_ENGINE` | 协调整个工作流执行 |
| `LOCKING` | 并发控制与锁管理 |
| `CART` | 购物车数据管理 |
| `PAYMENT` | 支付处理与授权 |
| `INVENTORY` | 库存预留与管理 |
| `ORDER` | 订单创建与管理 |
| `PROMOTION` | 促销使用记录 |

### 1.3 整体流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    /store/carts/:id/complete                     │
│                         (API 入口)                                │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Workflow Engine (工作流引擎)                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 1. acquireLockStep (获取锁)                                │  │
│  │ 2. 查询 order_cart link (幂等性检查)                      │  │
│  │ 3. validateCartPaymentsStep (支付校验)                    │  │
│  │ 4. compensatePaymentIfNeededStep (补偿准备)               │  │
│  │ 5. validateShippingStep (发货校验)                         │  │
│  │ 6. createOrdersStep (订单创建)                             │  │
│  │ 7. reserveInventoryStep (库存预留)                         │  │
│  │ 8. createRemoteLinkStep (建立 cart-order link)            │  │
│  │ 9. authorizePaymentSessionStep (支付授权)                  │  │
│  │ 10. addOrderTransactionStep (添加交易记录)                 │  │
│  │ 11. releaseLockStep (释放锁)                               │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                                 ▼
    ┌─────────────────┐              ┌─────────────────┐
    │  type: "order"  │              │  type: "cart"   │
    │   (成功下单)     │              │  (需要用户操作)   │
    └─────────────────┘              └─────────────────┘
```

---

## 2. 核心步骤详解

### 2.1 并发保护：Lock 机制

#### 2.1.1 外层锁：基于 cart_id 的分布式锁

在工作流开始时，首先获取基于 cart_id 的锁：

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:310-314
acquireLockStep({
  key: input.id,        // cart_id 作为锁的 key
  timeout: THIRTY_SECONDS,  // 30秒超时
  ttl: TWO_MINUTES,         // 2分钟过期
})
```

**锁的获取逻辑** (`acquire-lock.ts`):

```typescript
// packages/core/core-flows/src/locking/steps/acquire-lock.ts:76-94
const locking = container.resolve(Modules.LOCKING)

const retryInterval = data.retryInterval ?? 0.3
const tryUntil = Date.now() + (data.timeout ?? 0) * 1000

while (true) {
  try {
    await locking.acquire(data.key, {
      expire: data.ttl,
      ownerId: data.ownerId,
      provider: data.provider,
    })
    break
  } catch (e) {
    if (Date.now() >= tryUntil) {
      throw e
    }
  }
  await setTimeout(retryInterval * 1000)
}
```

**API 层的并发检测**：

```typescript
// packages/medusa/src/api/store/carts/[id]/complete/route.ts:25-30
if (!transaction.hasFinished()) {
  throw new MedusaError(
    MedusaError.Types.CONFLICT,
    "Cart is already being completed by another request"
  )
}
```

#### 2.1.2 内层锁：库存操作的细粒度锁

在 `reserveInventoryStep` 中，对每个库存项也使用了锁：

```typescript
// packages/core/core-flows/src/cart/steps/reserve-inventory.ts:88-92
const lockingKeys = Array.from(new Set(inventoryItemIds))

const reservations = await locking.execute(lockingKeys, async () => {
  return await inventoryService.createReservationItems(items)
})
```

**补偿时的锁释放**：

```typescript
// packages/core/core-flows/src/cart/steps/reserve-inventory.ts:107-112
const inventoryItemIds = data.inventoryItemIds
const lockingKeys = Array.from(new Set(inventoryItemIds))

await locking.execute(lockingKeys, async () => {
  await inventoryService.deleteReservationItems(data.reservations)
})
```

### 2.2 幂等性：检查 order_cart link

工作流通过查询 `order_cart` 链接来实现幂等性：

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:316-339
const [orderCart, cartData] = parallelize(
  useQueryGraphStep({
    entity: "order_cart",
    fields: ["cart_id", "order_id"],
    filters: { cart_id: input.id },
    options: {
      isList: false,
    },
  }),
  useQueryGraphStep({
    entity: "cart",
    fields: completeCartFields,
    filters: { id: input.id },
    options: {
      isList: false,
    },
  }).config({
    name: "cart-query",
  })
)

const orderId = transform({ orderCart }, ({ orderCart }) => {
  return orderCart?.data?.order_id
})
```

**条件判断：只有当订单不存在时才执行创建流程**：

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:355-357
const order = when("create-order", { orderId }, ({ orderId }) => {
  return !orderId  // 如果 orderId 不存在，才执行创建流程
}).then(() => {
  // ... 完整的订单创建逻辑
})
```

**最终返回逻辑**：

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:661-663
const result = transform({ order, orderId }, ({ order, orderId }) => {
  return { id: order?.id ?? orderId } as CompleteCartWorkflowOutput
})
```

这意味着：
- 如果 `orderCart` 已存在（之前已创建过订单），直接返回已有的 `orderId`
- 如果 `orderCart` 不存在，执行新的创建流程并返回新创建的 `order.id`

### 2.3 Query Graph：跨模块数据查询

工作流使用 `useQueryGraphStep` 进行跨模块的数据查询：

#### 查询 order_cart 链接：
```typescript
useQueryGraphStep({
  entity: "order_cart",
  fields: ["cart_id", "order_id"],
  filters: { cart_id: input.id },
  options: {
    isList: false,
  },
})
```

#### 查询 cart 完整数据：
```typescript
useQueryGraphStep({
  entity: "cart",
  fields: completeCartFields,
  filters: { id: input.id },
  options: {
    isList: false,
  },
}).config({
  name: "cart-query",
})
```

#### 查询 shipping options：
```typescript
useQueryGraphStep({
  entity: "shipping_option",
  fields: ["id", "shipping_profile_id"],
  filters: { id: cartOptionIds },
  options: {
    cache: {
      enable: true,
    },
  },
}).config({
  name: "shipping-options-query",
})
```

### 2.4 支付校验

#### 2.4.1 validateCartPaymentsStep

```typescript
// packages/core/core-flows/src/cart/steps/validate-cart-payments.ts:38-80
export const validateCartPaymentsStep = createStep(
  validateCartPaymentsStepId,
  async (data: ValidateCartPaymentsStepInput) => {
    const {
      cart: { payment_collection: paymentCollection, total, credit_line_total },
    } = data

    // 检查是否可以跳过支付（如使用 store credit 全额支付）
    const canSkipPayment =
      MathBN.convert(credit_line_total).gte(0) && MathBN.convert(total).lte(0)

    if (canSkipPayment) {
      return new StepResponse([])
    }

    // 检查 payment collection 是否已初始化
    if (!isPresent(paymentCollection)) {
      throw new MedusaError(
        MedusaError.Types.INVALID_DATA,
        `Payment collection has not been initiated for cart`
      )
    }

    // 检查可处理的支付状态
    const processablePaymentStatuses = [
      PaymentSessionStatus.PENDING,
      PaymentSessionStatus.REQUIRES_MORE,
      PaymentSessionStatus.AUTHORIZED,
      PaymentSessionStatus.CAPTURED,
    ]

    const paymentsToProcess = paymentCollection.payment_sessions?.filter((ps) =>
      processablePaymentStatuses.includes(ps.status as PaymentSessionStatus)
    )

    if (!paymentsToProcess?.length) {
      throw new MedusaError(
        MedusaError.Types.INVALID_DATA,
        `Payment sessions are required to complete cart`
      )
    }

    return new StepResponse(paymentsToProcess)
  }
)
```

#### 2.4.2 补偿准备：compensatePaymentIfNeededStep

这个步骤在流程早期就被调用，目的是为了在流程失败时能够正确地补偿支付：

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:342-347
const paymentSessions = validateCartPaymentsStep({ cart: cartData.data })
// purpose of this step is to run compensation if cart completion fails
// and tries to refund the payment if captured
compensatePaymentIfNeededStep({
  payment_session_id: paymentSessions[0].id,
})
```

**补偿逻辑**：

```typescript
// packages/core/core-flows/src/cart/steps/compensate-payment-if-needed.ts:33-85
async (paymentSessionId, { container }) => {
  if (!paymentSessionId) {
    return
  }

  const logger = container.resolve<Logger>(ContainerRegistrationKeys.LOGGER)
  const query = container.resolve(ContainerRegistrationKeys.QUERY)

  const { data: paymentSessions } = await query.graph({
    entity: "payment_session",
    fields: [
      "id",
      "payment_collection_id",
      "amount",
      "raw_amount",
      "provider_id",
      "data",
      "payment.id",
      "payment.captured_at",
      "payment.customer.id",
    ],
    filters: {
      id: paymentSessionId,
    },
  })
  const paymentSession = paymentSessions[0]

  if (!paymentSession) {
    return
  }

  // 如果支付已被捕获，尝试退款
  if (paymentSession.payment?.captured_at) {
    try {
      const workflowInput = {
        payment_collection_id: paymentSession.payment_collection_id,
        provider_id: paymentSession.provider_id,
        customer_id: paymentSession.payment?.customer?.id,
        data: paymentSession.data,
        amount: paymentSession.raw_amount ?? paymentSession.amount,
        payment_id: paymentSession.payment.id,
        note: "Refunded due to cart completion failure",
      }

      await refundPaymentAndRecreatePaymentSessionWorkflow(container).run({
        input: workflowInput,
      })
    } catch (e) {
      logger.error(
        `Error was thrown trying to refund payment - ${paymentSession.payment?.id} - ${e}`
      )
    }
  }
}
```

### 2.5 发货校验

```typescript
// packages/core/core-flows/src/cart/steps/validate-shipping.ts:73-115
export const validateShippingStep = createStep(
  validateShippingStepId,
  async (data: ValidateShippingInput) => {
    const { cart, shippingOptions } = data

    const optionProfileMap: Map<string, string> = new Map(
      shippingOptions.map((option) => [option.id, option.shipping_profile_id])
    )

    const cartItemsWithShipping =
      cart.items?.filter((item) => item.requires_shipping) || []

    const cartShippingMethods = cart.shipping_methods || []

    // 检查：如果有需要发货的商品，必须选择发货方式
    if (cartItemsWithShipping.length > 0 && cartShippingMethods.length === 0) {
      throw new MedusaError(
        MedusaError.Types.INVALID_DATA,
        "No shipping method selected but the cart contains items that require shipping."
      )
    }

    // 检查：商品的 shipping profile 必须与选中的发货方式匹配
    const requiredShippingPorfiles = cartItemsWithShipping.map(
      (item) => (item.variant.product as any)?.shipping_profile?.id
    )

    const availableShippingPorfiles = cartShippingMethods.map((method) =>
      optionProfileMap.get(method.shipping_option_id!)
    )

    const missingShippingPorfiles = requiredShippingPorfiles.filter(
      (profile) => !availableShippingPorfiles.includes(profile)
    )

    if (missingShippingPorfiles.length > 0) {
      throw new MedusaError(
        MedusaError.Types.INVALID_DATA,
        "The cart items require shipping profiles that are not satisfied by the current shipping methods"
      )
    }

    return new StepResponse(void 0)
  }
)
```

### 2.6 库存预留

```typescript
// packages/core/core-flows/src/cart/steps/reserve-inventory.ts:61-116
export const reserveInventoryStep = createStep(
  reserveInventoryStepId,
  async (data: ReserveVariantInventoryStepInput, { container }) => {
    if (!data.items.length) {
      return new StepResponse([], {
        reservations: [],
        inventoryItemIds: [],
      })
    }

    const inventoryService = container.resolve(Modules.INVENTORY)
    const locking = container.resolve(Modules.LOCKING)

    const inventoryItemIds: string[] = []

    const items = data.items.map((item) => {
      inventoryItemIds.push(item.inventory_item_id)

      return {
        line_item_id: item.id,
        inventory_item_id: item.inventory_item_id,
        quantity: MathBN.mult(item.required_quantity, item.quantity),
        allow_backorder: item.allow_backorder,
        location_id: item.location_ids[0],
      }
    })

    const lockingKeys = Array.from(new Set(inventoryItemIds))

    // 使用细粒度的锁保护库存操作
    const reservations = await locking.execute(lockingKeys, async () => {
      return await inventoryService.createReservationItems(items)
    })

    return new StepResponse(reservations, {
      reservations: reservations.map((r) => r.id),
      inventoryItemIds,
    })
  },
  // 补偿函数：删除预留
  async (data, { container }) => {
    if (!data?.reservations?.length) {
      return
    }

    const inventoryService = container.resolve(Modules.INVENTORY)
    const locking = container.resolve(Modules.LOCKING)

    const inventoryItemIds = data.inventoryItemIds
    const lockingKeys = Array.from(new Set(inventoryItemIds))

    await locking.execute(lockingKeys, async () => {
      await inventoryService.deleteReservationItems(data.reservations)
    })

    return new StepResponse()
  }
)
```

### 2.7 订单创建与 Cart-Order Link 建立

#### 2.7.1 订单创建

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:488-492
const createdOrders = createOrdersStep([cartToOrder])

const createdOrder = transform({ createdOrders }, ({ createdOrders }) => {
  return createdOrders[0]
})
```

**createOrdersStep 实现**：

```typescript
// packages/core/core-flows/src/order/steps/create-orders.ts:31-51
export const createOrdersStep = createStep(
  createOrdersStepId,
  async (data: CreateOrderDTO[], { container }) => {
    const service = container.resolve<IOrderModuleService>(Modules.ORDER)

    const created = await service.createOrders(data)
    return new StepResponse(
      created,
      created.map((store) => store.id)
    )
  },
  // 补偿函数：删除订单
  async (createdIds, { container }) => {
    if (!createdIds?.length) {
      return
    }

    const service = container.resolve<IOrderModuleService>(Modules.ORDER)

    await service.deleteOrders(createdIds)
  }
)
```

#### 2.7.2 Cart 到 Order 的数据转换

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:402-486
const cartToOrder = transform({ cart: cartData.data }, ({ cart }) => {
  const allItems = (cart.items ?? []).map((item) => {
    const input: PrepareLineItemDataInput = {
      item,
      variant: item.variant,
      cartId: cart.id,
      unitPrice: item.unit_price,
      isTaxInclusive: item.is_tax_inclusive,
      taxLines: item.tax_lines ?? [],
      adjustments: item.adjustments ?? [],
    }

    return prepareLineItemData(input)
  })

  const shippingMethods = (cart.shipping_methods ?? []).map((sm) => {
    return {
      name: sm.name,
      description: sm.description,
      amount: sm.raw_amount ?? sm.amount,
      is_tax_inclusive: sm.is_tax_inclusive,
      shipping_option_id: sm.shipping_option_id,
      data: sm.data,
      metadata: sm.metadata,
      tax_lines: prepareTaxLinesData(sm.tax_lines ?? []),
      adjustments: prepareAdjustmentsData(sm.adjustments ?? []),
    }
  })

  // ... 更多转换逻辑

  return {
    region_id: cart.region?.id,
    customer_id: cart.customer?.id,
    sales_channel_id: cart.sales_channel_id,
    status: OrderStatus.PENDING,
    email: cart.email,
    currency_code: cart.currency_code,
    locale: cart.locale,
    shipping_address: shippingAddress,
    billing_address: billingAddress,
    no_notification: false,
    items: allItems,
    shipping_methods: shippingMethods,
    metadata: cart.metadata,
    promo_codes: promoCodes,
    credit_lines: creditLines,
  }
})
```

#### 2.7.3 建立 Cart-Order Link

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:562-592
const linksToCreate = transform(
  { cart: cartData.data, createdOrder },
  ({ cart, createdOrder }) => {
    const links: LinkDefinition[] = [
      {
        [Modules.ORDER]: { order_id: createdOrder.id },
        [Modules.CART]: { cart_id: cart.id },
      },
    ]

    // 添加 promotion 链接
    if (cart.promotions?.length) {
      cart.promotions.forEach((promotion: PromotionDTO) => {
        links.push({
          [Modules.ORDER]: { order_id: createdOrder.id },
          [Modules.PROMOTION]: { promotion_id: promotion.id },
        })
      })
    }

    // 添加 payment 链接
    if (isDefined(cart.payment_collection?.id)) {
      links.push({
        [Modules.ORDER]: { order_id: createdOrder.id },
        [Modules.PAYMENT]: {
          payment_collection_id: cart.payment_collection.id,
        },
      })
    }

    return links
  }
)
```

#### 2.7.4 并行执行多个操作

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:594-606
parallelize(
  createRemoteLinkStep(linksToCreate),           // 建立链接
  updateCartsStep([updateCompletedAt]),          // 更新 cart 的 completed_at
  reserveInventoryStep(formatedInventoryItems),  // 预留库存
  registerUsageStep(promotionUsage),             // 记录促销使用
  emitEventStep({                                 // 发送订单创建事件
    eventName: OrderWorkflowEvents.PLACED,
    data: { id: createdOrder.id },
    options: {
      priority: EventPriority.CRITICAL,
    },
  })
)
```

### 2.8 支付授权与交易记录

#### 2.8.1 支付授权

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:615-622
// We authorize payment sessions at the very end of the workflow to minimize the risk of
// canceling the payment in the compensation flow. The only operations that can trigger it
// is creating the transactions, the workflow hook, and the linking.
const payment = authorizePaymentSessionStep({
  // We choose the first payment session, as there will only be one active payment session
  // This might change in the future.
  id: paymentSessions![0].id,
})
```

#### 2.8.2 添加交易记录

```typescript
// packages/core/core-flows/src/cart/workflows/complete-cart.ts:624-644
const orderTransactions = transform(
  { payment, createdOrder },
  ({ payment, createdOrder }) => {
    const transactions =
      (payment &&
        payment?.captures?.map((capture) => {
          return {
            order_id: createdOrder.id,
            amount: capture.raw_amount ?? capture.amount,
            currency_code: payment.currency_code,
            reference: "capture",
            reference_id: capture.id,
          }
        })) ??
      []

    return transactions
  }
)

addOrderTransactionStep(orderTransactions)
```

---

## 3. 返回结果：type: "order" vs type: "cart"

### 3.1 API 层的结果处理

```typescript
// packages/medusa/src/api/store/carts/[id]/complete/route.ts:36-85
// When an error occurs on the workflow, its potentially got to with cart validations, payments
// or inventory checks. Return the cart here along with errors for the consumer to take more action
// and fix them
if (errors?.[0]) {
  const error = errors[0].error
  const statusOKErrors: string[] = [
    // TODO: add inventory specific errors
    MedusaError.Types.PAYMENT_AUTHORIZATION_ERROR,
    MedusaError.Types.PAYMENT_REQUIRES_MORE_ERROR,
  ]

  // If we end up with errors outside of statusOKErrors, it means that the cart is not in a state to be
  // completed. In these cases, we return a 400.
  const cartReq = await prepareRetrieveQuery(
    {},
    {
      defaults: defaultStoreCartFields,
    },
    req as MedusaRequest
  )
  const cart = await refetchCart(
    cart_id,
    req.scope,
    cartReq.remoteQueryConfig.fields
  )

  if (!statusOKErrors.includes(error?.type)) {
    throw error
  }

  // 返回 type: "cart" - 需要用户进一步操作
  res.status(200).json({
    type: "cart",
    cart,
    error: {
      message: error.message,
      name: error.name,
      type: error.type,
    },
  })
  return
}

// 返回 type: "order" - 下单成功
const { data } = await query.graph({
  entity: "order",
  fields: req.queryConfig.fields,
  filters: { id: result.id },
})

res.status(200).json({
  type: "order",
  order: data[0],
})
```

### 3.2 两种返回类型的对比

| 场景 | 返回类型 | 说明 |
|------|----------|------|
| 工作流执行成功，订单已创建 | `type: "order"` | 返回完整的订单数据，用户下单成功 |
| 支付需要进一步授权（如 3DS） | `type: "cart"` | 返回 cart 数据和错误信息，用户需要完成额外的支付步骤 |
| 支付需要更多信息 | `type: "cart"` | 返回 cart 数据和错误信息，用户需要补充支付信息 |
| 其他验证错误 | 抛出 400 错误 | cart 状态不正确，无法完成下单 |

---

## 4. 幂等性与并发保护的实现机制

### 4.1 幂等性实现

#### 核心机制：order_cart link 检查

```
┌─────────────────────────────────────────────────────────────────┐
│                    幂等性检查流程                                  │
└─────────────────────────────────────────────────────────────────┘

第一次请求:
┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 1. 获取锁    │───▶│ 2. 查询 order_cart │───▶│ 3. 结果: 不存在   │
└──────────────┘    └──────────────────┘    └──────────────────┘
                                                  │
                                                  ▼
                                         ┌──────────────────┐
                                         │ 4. 执行创建流程   │
                                         │    - 创建订单     │
                                         │    - 预留库存     │
                                         │    - 建立 link    │
                                         └──────────────────┘

第二次请求 (重试):
┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 1. 获取锁    │───▶│ 2. 查询 order_cart │───▶│ 3. 结果: 已存在   │
└──────────────┘    └──────────────────┘    └──────────────────┘
                                                  │
                                                  ▼
                                         ┌──────────────────┐
                                         │ 4. 直接返回已有   │
                                         │    订单 ID        │
                                         └──────────────────┘
```

#### 关键代码

```typescript
// 1. 并行查询 order_cart 和 cart 数据
const [orderCart, cartData] = parallelize(
  useQueryGraphStep({
    entity: "order_cart",
    fields: ["cart_id", "order_id"],
    filters: { cart_id: input.id },
    options: { isList: false },
  }),
  // ...
)

// 2. 提取 orderId
const orderId = transform({ orderCart }, ({ orderCart }) => {
  return orderCart?.data?.order_id
})

// 3. 只有当 orderId 不存在时才执行创建流程
const order = when("create-order", { orderId }, ({ orderId }) => {
  return !orderId
}).then(() => {
  // ... 完整的订单创建逻辑
})

// 4. 返回时优先使用已有的 orderId
const result = transform({ order, orderId }, ({ order, orderId }) => {
  return { id: order?.id ?? orderId }
})
```

### 4.2 并发保护实现

#### 双层锁机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    并发保护：双层锁机制                            │
└─────────────────────────────────────────────────────────────────┘

请求 A (cart_123)                    请求 B (cart_123)
     │                                     │
     ▼                                     ▼
┌─────────────────┐               ┌─────────────────┐
│ acquireLockStep │               │ acquireLockStep │
│ key: "cart_123" │               │ key: "cart_123" │
└────────┬────────┘               └────────┬────────┘
         │                                  │
         ▼                                  ▼
    ┌──────────┐                      ┌──────────┐
    │  成功获取  │                      │  获取失败  │
    │   锁      │                      │  等待重试  │
    └────┬─────┘                      └────┬─────┘
         │                                  │
         ▼                                  │
┌─────────────────┐                         │
│  执行业务逻辑   │                         │
│  - 创建订单     │                         │
│  - 预留库存     │◀────────────────────────┘
│  - 建立 link    │    (内层锁：inventoryItemId 级别)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ releaseLockStep │
└────────┬────────┘
         │
         ▼
    请求 B 现在可以获取锁
    但查询 order_cart 发现已存在
    直接返回已有订单 ID
```

#### 外层锁：cart_id 级别

```typescript
// 获取锁
acquireLockStep({
  key: input.id,        // cart_id
  timeout: 30,          // 30秒超时
  ttl: 120,             // 2分钟过期
})

// 释放锁 (工作流末尾)
releaseLockStep({
  key: input.id,
})
```

#### 内层锁：inventoryItemId 级别

```typescript
// 库存预留时的细粒度锁
const lockingKeys = Array.from(new Set(inventoryItemIds))

const reservations = await locking.execute(lockingKeys, async () => {
  return await inventoryService.createReservationItems(items)
})
```

#### API 层的额外保护

```typescript
const { errors, result, transaction } = await we.run(completeCartWorkflowId, {
  input: { id: cart_id },
  throwOnError: false,
})

if (!transaction.hasFinished()) {
  throw new MedusaError(
    MedusaError.Types.CONFLICT,
    "Cart is already being completed by another request"
  )
}
```

### 4.3 幂等性 vs 并发保护：对比与协作

| 维度 | 幂等性 (Idempotency) | 并发保护 (Concurrency Control) |
|------|----------------------|--------------------------------|
| **目标** | 多次执行产生相同结果，不产生副作用 | 防止多个请求同时修改同一资源 |
| **实现机制** | 查询 order_cart link 是否存在 | 分布式锁 (Locking Module) |
| **关键点** | 创建 link 后，后续请求直接返回 | 同一时间只有一个请求能获取锁 |
| **覆盖场景** | 网络重试、客户端重复提交 | 并发请求、竞态条件 |
| **失败处理** | 跳过创建，直接返回 | 等待或抛出 CONFLICT 错误 |

**协作流程**：

```
请求 1 (第一次)              请求 2 (并发)              请求 3 (重试)
     │                          │                          │
     ▼                          ▼                          ▼
┌─────────┐                ┌─────────┐                ┌─────────┐
│获取锁    │◀──────────────▶│获取锁    │                │获取锁    │
│成功     │                │等待     │                │成功     │
└────┬────┘                └────┬────┘                └────┬────┘
     │                          │                          │
     ▼                          │                          ▼
┌─────────┐                     │                    ┌─────────────┐
│查询link │                     │                    │ 查询 link   │
│不存在   │                     │                    │ 已存在      │
└────┬────┘                     │                    └──────┬──────┘
     │                          │                           │
     ▼                          │                           ▼
┌─────────┐                     │                    ┌─────────────┐
│创建订单 │                     │                    │ 直接返回     │
│预留库存 │                     │                    │ 已有订单     │
│建立link │◀────────────────────┘                    └─────────────┘
└────┬────┘
     │
     ▼
┌─────────┐
│释放锁   │
└────┬────┘
     │
     ▼
请求 2 现在获取锁
查询 link 发现已存在
直接返回已有订单
```

---

## 5. 补偿机制：工作流的回滚能力

### 5.1 补偿步骤概览

| 步骤 | 正向操作 | 补偿操作 |
|------|----------|----------|
| `acquireLockStep` | 获取锁 | 释放锁 |
| `createOrdersStep` | 创建订单 | 删除订单 |
| `reserveInventoryStep` | 预留库存 | 删除预留 |
| `compensatePaymentIfNeededStep` | (无正向操作) | 退款并重建支付会话 |

### 5.2 关键补偿逻辑

#### 订单创建的补偿

```typescript
// create-orders.ts:42-50
async (createdIds, { container }) => {
  if (!createdIds?.length) {
    return
  }

  const service = container.resolve<IOrderModuleService>(Modules.ORDER)

  await service.deleteOrders(createdIds)
}
```

#### 库存预留的补偿

```typescript
// reserve-inventory.ts:99-115
async (data, { container }) => {
  if (!data?.reservations?.length) {
    return
  }

  const inventoryService = container.resolve(Modules.INVENTORY)
  const locking = container.resolve(Modules.LOCKING)

  const inventoryItemIds = data.inventoryItemIds
  const lockingKeys = Array.from(new Set(inventoryItemIds))

  await locking.execute(lockingKeys, async () => {
    await inventoryService.deleteReservationItems(data.reservations)
  })
}
```

#### 支付补偿（特殊设计）

`compensatePaymentIfNeededStep` 是一个特殊的步骤，它的正向操作什么都不做，只返回 payment_session_id 供补偿时使用：

```typescript
// compensate-payment-if-needed.ts:26-32
async (data: CompensatePaymentIfNeededStepInput, { container }) => {
  const { payment_session_id } = data

  return new StepResponse(payment_session_id)
}
```

这样设计的目的是：
1. 在工作流早期就注册补偿逻辑
2. 如果后续任何步骤失败，补偿函数会被触发
3. 补偿函数会检查支付是否已被捕获，如果是则尝试退款

---

## 6. 总结

### 6.1 为何是典型的跨系统多模块流程

"完成 Cart 下单"流程涉及以下多个模块的协同工作：

1. **工作流引擎** (`WORKFLOW_ENGINE`): 协调整个流程的执行，支持步骤编排、补偿机制
2. **锁模块** (`LOCKING`): 提供分布式锁能力，保护并发操作
3. **购物车模块** (`CART`): 管理购物车数据，完成后标记 `completed_at`
4. **支付模块** (`PAYMENT`): 处理支付授权、捕获、退款
5. **库存模块** (`INVENTORY`): 预留库存，防止超卖
6. **订单模块** (`ORDER`): 创建和管理订单
7. **促销模块** (`PROMOTION`): 记录促销使用情况

### 6.2 核心设计亮点

| 设计点 | 实现方式 | 价值 |
|--------|----------|------|
| **幂等性** | 查询 `order_cart` link | 防止重复下单，支持网络重试 |
| **并发保护** | 双层分布式锁 (cart_id + inventoryItemId) | 防止竞态条件，保证数据一致性 |
| **补偿机制** | 每个步骤的 `compensation` 函数 | 失败时自动回滚，保证系统一致性 |
| **灵活的错误处理** | 返回 `type: "cart"` 或 `type: "order"` | 区分需要用户操作的错误和系统错误 |
| **延迟支付授权** | 在工作流末尾才调用 `authorizePaymentSessionStep` | 最小化需要退款的场景 |

### 6.3 流程图总结

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          完整的 Cart 下单流程                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  API 入口: POST /store/carts/:id/complete                                   │
│                          │                                                   │
│                          ▼                                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Workflow Engine 执行                                │  │
│  │                                                                       │  │
│  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │  │
│  │  │ acquireLock  │───▶│ 查询 order_cart│───▶│ 检查是否存在 │         │  │
│  │  │ (cart_id)    │    │    link      │    │  幂等性检查  │         │  │
│  │  └──────────────┘    └──────────────┘    └──────┬───────┘         │  │
│  │                                                    │                  │  │
│  │                          ┌─────────────────────────┴──────────────┐ │  │
│  │                          │                                         │ │  │
│  │                  ┌───────┴───────┐                        ┌───────┴───────┐ │  │
│  │                  │  Link 不存在  │                        │  Link 已存在  │ │  │
│  │                  │  (首次下单)    │                        │  (幂等返回)    │ │  │
│  │                  └───────┬───────┘                        └───────┬───────┘ │  │
│  │                          │                                         │          │  │
│  │                          ▼                                         │          │  │
│  │              ┌──────────────────────┐                              │          │  │
│  │              │ 1. validateCartPay-  │                              │          │  │
│  │              │    mentsStep         │                              │          │  │
│  │              │ 2. compensatePayment │                              │          │  │
│  │              │    IfNeededStep      │                              │          │  │
│  │              │ 3. validateShipping  │                              │          │  │
│  │              │    Step              │                              │          │  │
│  │              │ 4. createOrdersStep  │                              │          │  │
│  │              │ 5. parallelize:      │                              │          │  │
│  │              │    - createRemoteLink│                              │          │  │
│  │              │    - updateCartsStep │                              │          │  │
│  │              │    - reserveInventory│                              │          │  │
│  │              │    - registerUsage   │                              │          │  │
│  │              │    - emitEvent       │                              │          │  │
│  │              │ 6. authorizePayment  │                              │          │  │
│  │              │    SessionStep       │                              │          │  │
│  │              │ 7. addOrderTransact- │                              │          │  │
│  │              │    ionStep           │                              │          │  │
│  │              └──────────┬───────────┘                              │          │  │
│  │                         │                                          │          │  │
│  │                         └──────────────────────────────────────────┘          │  │
│  │                                                    │                             │  │
│  │                                                    ▼                             │  │
│  │                                          ┌──────────────┐                       │  │
│  │                                          │ releaseLock  │                       │  │
│  │                                          │ (cart_id)    │                       │  │
│  │                                          └──────┬───────┘                       │  │
│  │                                                 │                                │  │
│  └─────────────────────────────────────────────────┼────────────────────────────────┘  │
│                                                    │                                   │
│                          ┌─────────────────────────┴───────────────┐                   │
│                          │                                         │                   │
│              ┌───────────┴───────────┐                 ┌───────────┴───────────┐       │
│              │      执行成功          │                 │      执行失败          │       │
│              │  (无 errors)           │                 │  (有 errors)           │       │
│              └───────────┬───────────┘                 └───────────┬───────────┘       │
│                          │                                         │                   │
│                          ▼                                         ▼                   │
│              ┌───────────────────────┐         ┌───────────────────────────────┐       │
│              │   type: "order"       │         │ 检查错误类型:                  │       │
│              │   返回订单数据         │         │ - PAYMENT_AUTHORIZATION_ERROR │       │
│              │                       │         │ - PAYMENT_REQUIRES_MORE_ERROR  │       │
│              │                       │         │   → type: "cart"               │       │
│              │                       │         │   → 返回 cart + 错误信息        │       │
│              │                       │         │ - 其他错误                       │       │
│              │                       │         │   → 抛出 400 错误               │       │
│              └───────────────────────┘         └───────────────────────────────┘       │
│                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```
