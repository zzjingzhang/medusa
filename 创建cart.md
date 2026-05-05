# 创建 Cart 时 publishable key 如何约束 sales channel

## 1. 整体链路概览

创建 Cart 时，系统通过一系列中间件和工作流来处理 `x-publishable-api-key` 请求头与 `sales_channel_id` 之间的约束关系。整体流程如下：

```
请求到达 → maybeAttachPublishableKeyScopes → ensurePublishableKeyAndSalesChannelMatch 
         → wrapHandler (检查错误) → createCartWorkflow → 后续业务逻辑
```

---

## 2. 中间件处理逻辑

### 2.1 maybeAttachPublishableKeyScopes

**文件位置**: `packages/medusa/src/api/utils/middlewares/common/maybe-attach-pub-key-scopes.ts`

这个中间件的作用是从请求头中提取 `x-publishable-api-key`，并查询该 key 关联的所有 sales channels。

**核心逻辑**:
```typescript
export async function maybeAttachPublishableKeyScopes(
  req: MedusaRequest & { publishableApiKeyScopes: any },
  res: MedusaResponse,
  next: NextFunction
) {
  const pubKey = req.get("x-publishable-api-key")

  if (pubKey) {
    const remoteQuery = req.scope.resolve<RemoteQueryFunction>(
      ContainerRegistrationKeys.REMOTE_QUERY
    )

    // 查询 publishable key 关联的 sales channels
    const queryObject = remoteQueryObjectFromString({
      entryPoint: "api_key",
      fields: ["sales_channels.id"],
      variables: {
        filters: { token: pubKey },
      },
    })

    const [apiKey] = await remoteQuery(queryObject)

    // 将关联的 sales channel IDs 存入请求上下文
    req.publishableApiKeyScopes = {
      sales_channel_ids: apiKey?.sales_channels.map((sc) => sc.id) ?? [],
    }
  }

  next()
}
```

**关键点**:
- 只在请求头包含 `x-publishable-api-key` 时执行
- 通过 `api_key` 实体查询关联的 `sales_channels`
- 将 `sales_channel_ids` 数组存入 `req.publishableApiKeyScopes`

---

### 2.2 ensurePublishableKeyAndSalesChannelMatch

**文件位置**: `packages/medusa/src/api/utils/middlewares/common/ensure-pub-key-sales-channel-match.ts`

这个中间件是核心约束逻辑的实现地，负责检查并处理 sales channel 与 publishable key 的匹配关系。

**核心逻辑**:

```typescript
export async function ensurePublishableKeyAndSalesChannelMatch(
  req: MedusaRequest<StoreCreateCartType> & { publishableApiKeyScopes },
  res: MedusaResponse,
  next: NextFunction
) {
  const pubKey = req.get("x-publishable-api-key")

  if (pubKey) {
    const pubKeySalesChannels =
      req.publishableApiKeyScopes?.sales_channel_ids ?? []
    const channelId = req.validatedBody?.sales_channel_id

    req.errors = req.errors ?? []

    if (pubKeySalesChannels.length) {
      // 场景1: body 中有 sales_channel_id，检查是否在 publishable key 的范围内
      if (channelId && !pubKeySalesChannels.includes(channelId)) {
        req.errors.push(
          `Sales channel ID in payload ${channelId} is not associated with the Publishable API Key in the header.`
        )
      }

      // 场景2: body 中没有 sales_channel_id
      if (!channelId) {
        if (pubKeySalesChannels.length > 1) {
          // 多 sales channel 场景：无法自动选择，累积错误
          req.errors.push(
            `Cannot assign sales channel to cart. The Publishable API Key in the header has multiple associated sales channels. Please provide a sales channel ID in the request body.`
          )
        } else {
          // 单 sales channel 场景：自动填充
          req.validatedBody.sales_channel_id = pubKeySalesChannels[0]
        }
      }
    }
  }

  next()
}
```

---

## 3. 自动填充 sales_channel_id 的条件

根据代码分析，系统会在**以下所有条件同时满足**时自动填充 `sales_channel_id`：

| 条件 | 说明 |
|------|------|
| 请求头包含 `x-publishable-api-key` | 客户端必须传递有效的 publishable key |
| publishable key 已绑定至少一个 sales channel | `pubKeySalesChannels.length > 0` |
| 请求 body 中**没有**提供 `sales_channel_id` | `!channelId` 为 true |
| publishable key **只绑定了一个** sales channel | `pubKeySalesChannels.length === 1` |

**自动填充的代码**:
```typescript
// 只有当 publishable key 只绑定一个 sales channel 时才自动填充
if (pubKeySalesChannels.length > 1) {
  // 多个 sales channel：报错
  req.errors.push("...")
} else {
  // 单个 sales channel：自动填充
  req.validatedBody.sales_channel_id = pubKeySalesChannels[0]
}
```

---

## 4. 多 sales channel 场景下的错误累积与传播

### 4.1 错误累积机制

当 publishable key 绑定了多个 sales channel，但请求中没有提供 `sales_channel_id` 时，错误会被累积到 `req.errors` 数组中，而不是立即抛出异常。

**错误累积的代码位置**:
```typescript
// ensure-pub-key-sales-channel-match.ts:30
req.errors = req.errors ?? []

// 累积错误
if (channelId && !pubKeySalesChannels.includes(channelId)) {
  req.errors.push(
    `Sales channel ID in payload ${channelId} is not associated with the Publishable API Key in the header.`
  )
}

if (!channelId && pubKeySalesChannels.length > 1) {
  req.errors.push(
    `Cannot assign sales channel to cart. The Publishable API Key in the header has multiple associated sales channels. Please provide a sales channel ID in the request body.`
  )
}
```

### 4.2 错误传播机制

错误的传播发生在 `wrapHandler` 中间件中，这个中间件会在实际路由处理函数执行之前检查 `req.errors` 数组。

**文件位置**: `packages/core/framework/src/http/utils/wrap-handler.ts`

```typescript
export const wrapHandler = <T extends RouteHandler | MiddlewareFunction>(
  fn: T
) => {
  async function wrappedHandler(
    req: MedusaRequest,
    res: MedusaResponse,
    next: MedusaNextFunction
  ) {
    const req_ = req as MedusaRequest & { errors?: Error[] }
    
    // 关键点：在执行路由处理前检查错误
    if (req_?.errors?.length) {
      return res.status(400).json({
        errors: req_.errors,
        message:
          "Provided request body contains errors. Please check the data and retry the request",
      })
    }

    try {
      return await fn(req, res, next)
    } catch (err) {
      next(err)
    }
  }
  // ...
}
```

### 4.3 错误传播流程图

```
请求到达 POST /store/carts
    │
    ▼
┌─────────────────────────────┐
│ validateAndTransformBody    │ ← 验证 body，填充 req.validatedBody
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ maybeAttachPublishableKey   │ ← 查询 publishable key 关联的 sales channels
│ Scopes                      │   填充 req.publishableApiKeyScopes
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ ensurePublishableKeyAnd     │ ← 核心约束检查
│ SalesChannelMatch           │   - 检查 channelId 是否在范围内
│                             │   - 自动填充或累积错误到 req.errors
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ wrapHandler                 │ ← 检查 req.errors
│                             │   - 有错误：返回 400
│                             │   - 无错误：执行路由处理
└─────────────────────────────┘
    │
    ▼ (如果无错误)
┌─────────────────────────────┐
│ 路由处理函数 (POST)          │ ← 执行 createCartWorkflow
└─────────────────────────────┘
```

---

## 5. 对 createCartWorkflow 的影响

### 5.1 工作流概览

**文件位置**: `packages/core/core-flows/src/cart/workflows/create-carts.ts`

`createCartWorkflow` 依赖于中间件处理后的 `sales_channel_id` 来执行以下关键步骤：

1. 查找并验证 sales channel
2. 确认商品库存（基于 sales channel）
3. 创建 cart 并关联正确的 sales channel

### 5.2 findSalesChannelStep

**文件位置**: `packages/core/core-flows/src/cart/steps/find-sales-channel.ts`

这个步骤根据传入的 `salesChannelId` 查找对应的 sales channel：

```typescript
export const findSalesChannelStep = createStep(
  findSalesChannelStepId,
  async (data: FindSalesChannelStepInput, { container }) => {
    let salesChannel: SalesChannelDTO | undefined

    // 优先使用传入的 salesChannelId（来自中间件自动填充或用户提供）
    if (data.salesChannelId) {
      salesChannel = await fetchSalesChannel(data.salesChannelId, container)
    } else if (!isDefined(data.salesChannelId)) {
      // 如果没有传入，尝试使用 store 的默认 sales channel
      const [store] = await fetchStore(container)
      if (store?.default_sales_channel_id) {
        salesChannel = await fetchSalesChannel(
          store.default_sales_channel_id,
          container
        )
      }
    }

    if (!salesChannel) {
      return new StepResponse(null)
    }

    // 检查 sales channel 是否已禁用
    if (salesChannel?.is_disabled) {
      throw new MedusaError(
        MedusaError.Types.INVALID_DATA,
        `Unable to assign cart to disabled Sales Channel: ${salesChannel.name}`
      )
    }

    return new StepResponse(salesChannel)
  }
)
```

### 5.3 validateSalesChannelStep

**文件位置**: `packages/core/core-flows/src/cart/steps/validate-sales-channel.ts`

这个步骤验证 sales channel 是否有效（非空）：

```typescript
export const validateSalesChannelStep = createStep(
  "validate-sales-channel",
  async (data: { salesChannel: SalesChannelDTO }) => {
    const { salesChannel } = data

    if (!salesChannel?.id) {
      throw new MedusaError(
        MedusaError.Types.INVALID_DATA,
        "Sales channel is required when creating a cart. Either provide a sales channel ID or set the default sales channel for the store."
      )
    }

    return new StepResponse(void 0)
  }
)
```

### 5.4 confirmVariantInventoryWorkflow 中的库存确认

**文件位置**: `packages/core/core-flows/src/cart/workflows/confirm-variant-inventory.ts`

库存确认是 `createCartWorkflow` 中的关键步骤，它**强烈依赖于 sales_channel_id** 来确定应该检查哪些库存位置。

在 `createCartWorkflow` 中调用库存确认：

```typescript
// create-carts.ts:171-178
confirmVariantInventoryWorkflow.runAsStep({
  input: {
    sales_channel_id: salesChannel.id,  // 使用中间件确定的 sales channel
    variants: variants as unknown as ConfirmVariantInventoryWorkflowInputDTO["variants"],
    items: input.items!,
  },
})
```

---

## 6. 库存确认的详细机制

### 6.1 prepareConfirmInventoryInput

**文件位置**: `packages/core/core-flows/src/cart/utils/prepare-confirm-inventory-input.ts`

这个函数是库存确认的核心，它使用 `sales_channel_id` 来：

1. 查找与该 sales channel 关联的库存位置（stock locations）
2. 验证变体是否在该 channel 中有可用的库存位置
3. 准备实际的库存检查输入

**核心逻辑**:

```typescript
export const prepareConfirmInventoryInput = (data: {
  input: ConfirmVariantInventoryWorkflowInputDTO
}) => {
  // ...
  const salesChannelId = data.input.sales_channel_id
  const variantsWithLocationForChannel = new Set<string>()
  const stockLocationIds = new Set<string>()

  // 深度遍历变体的库存信息
  deepFlatMap(
    data.input,
    "variants.inventory_items.inventory.location_levels.stock_locations.sales_channels",
    ({
      variants,
      inventory_items,
      location_levels,
      stock_locations,
      sales_channels,
    }) => {
      // ...

      // 关键点1: 标记哪些变体在指定的 sales channel 中有库存位置
      if (salesChannelId && sales_channels?.id === salesChannelId) {
        variantsWithLocationForChannel.add(variants.id)
      }

      // 关键点2: 收集该 sales channel 关联的所有库存位置 ID
      if (stock_locations && sales_channels?.id === salesChannelId) {
        stockLocationIds.add(stock_locations.id)
      }

      // ...
    }
  )

  // 关键点3: 验证所有需要管理库存的变体都有对应的库存位置
  if (salesChannelId) {
    for (const variant of allVariants.values()) {
      if (
        variant.manage_inventory &&
        !variantsWithLocationForChannel.has(variant.id) &&
        !variant.allow_backorder
      ) {
        // 变体需要库存管理，但在该 sales channel 中没有库存位置，且不允许缺货
        throw new MedusaError(
          MedusaError.Types.INVALID_DATA,
          `Sales channel ${salesChannelId} is not associated with any stock location for variant ${variant.id}.`
        )
      }
    }
  }

  // 准备库存确认输入
  const items = formatInventoryInput({
    location_ids: Array.from(stockLocationIds),  // 只使用该 sales channel 的库存位置
    // ...
  })

  return { items }
}
```

### 6.2 confirmInventoryStep

**文件位置**: `packages/core/core-flows/src/cart/steps/confirm-inventory.ts`

这个步骤实际执行库存检查，使用上一步准备好的 `location_ids`：

```typescript
export const confirmInventoryStep = createStep(
  confirmInventoryStepId,
  async (data: ConfirmVariantInventoryStepInput, { container }) => {
    if (!data.items?.length) {
      return new StepResponse([], [])
    }

    const inventoryService = container.resolve<IInventoryService>(
      Modules.INVENTORY
    )

    // 检查每个商品的库存
    const promises = data.items.map(async (item) => {
      if (item.allow_backorder) {
        return true  // 允许缺货，直接通过
      }

      const itemQuantity = MathBN.mult(item.quantity, item.required_quantity)

      // 使用 location_ids 检查库存
      return await inventoryService.confirmInventory(
        item.inventory_item_id,
        item.location_ids,  // 这里的 location_ids 是根据 sales_channel_id 过滤得到的
        itemQuantity
      )
    })

    const inventoryCoverage = await promiseAll(promises)

    if (inventoryCoverage.some((hasCoverage) => !hasCoverage)) {
      throw new MedusaError(
        MedusaError.Types.NOT_ALLOWED,
        `Some variant does not have the required inventory`,
        MedusaError.Codes.INSUFFICIENT_INVENTORY
      )
    }

    return new StepResponse(null)
  }
)
```

### 6.3 库存确认流程图

```
createCartWorkflow
    │
    ▼
┌─────────────────────────────┐
│ findSalesChannelStep        │ ← 获取 sales channel
│ (使用中间件传入的 ID)        │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ validateSalesChannelStep    │ ← 验证 sales channel 有效
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ getVariantsAndItemsWith     │ ← 获取变体的详细信息，包括：
│ Prices                      │   - inventory_items
│                             │   - inventory.location_levels
│                             │   - stock_locations.sales_channels
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ confirmVariantInventory     │
│ Workflow                    │
│                             │
│ ┌─────────────────────────┐ │
│ │ prepareConfirmInventory │ │ ← 核心过滤逻辑：
│ │ Input                   │ │   1. 根据 sales_channel_id 过滤
│ │                         │ │      关联的 stock locations
│ │                         │ │   2. 验证变体在该 channel
│ │                         │ │      中有库存位置
│ │                         │ │   3. 收集可用的 location_ids
│ └─────────────────────────┘ │
│             │               │
│             ▼               │
│ ┌─────────────────────────┐ │
│ │ confirmInventoryStep    │ │ ← 使用过滤后的 location_ids
│ │                         │ │   检查实际库存数量
│ └─────────────────────────┘ │
└─────────────────────────────┘
    │
    ▼ (库存确认通过)
┌─────────────────────────────┐
│ createCartsStep             │ ← 创建 cart，关联 sales channel
└─────────────────────────────┘
```

---

## 7. 对 completeCartWorkflow 的影响

### 7.1 工作流概览

**文件位置**: `packages/core/core-flows/src/cart/workflows/complete-cart.ts`

`completeCartWorkflow` 在完成 cart 并创建订单时，同样依赖于 cart 上的 `sales_channel_id` 来执行库存预留。

### 7.2 从 cart 中获取 sales_channel_id

在 `completeCartWorkflow` 中，`sales_channel_id` 是从已创建的 cart 数据中提取的：

```typescript
// complete-cart.ts:380-399
const { variants, sales_channel_id } = transform(
  { cart: cartData.data },
  (data) => {
    const variantsMap: Record<string, any> = {}
    const allItems = data.cart?.items?.map((item) => {
      variantsMap[item.variant_id] = item.variant
      return {
        id: item.id,
        variant_id: item.variant_id,
        quantity: item.quantity,
      }
    })

    return {
      variants: Object.values(variantsMap),
      items: allItems,
      sales_channel_id: data.cart.sales_channel_id,  // 从 cart 中获取
    }
  }
)
```

### 7.3 库存预留

在创建订单后，系统会使用 `sales_channel_id` 来预留库存：

```typescript
// complete-cart.ts:494-513
const reservationItemsData = transform(
  { createdOrder },
  ({ createdOrder }) =>
    createdOrder.items!.map((i) => ({
      variant_id: i.variant_id,
      quantity: i.quantity,
      id: i.id,
    }))
)

const formatedInventoryItems = transform(
  {
    input: {
      sales_channel_id,  // 使用 cart 上的 sales_channel_id
      variants,
      items: reservationItemsData,
    },
  },
  prepareConfirmInventoryInput  // 同样的过滤逻辑
)

// ...

parallelize(
  createRemoteLinkStep(linksToCreate),
  updateCartsStep([updateCompletedAt]),
  reserveInventoryStep(formatedInventoryItems),  // 预留库存
  registerUsageStep(promotionUsage),
  emitEventStep({...})
)
```

### 7.4 reserveInventoryStep

**文件位置**: `packages/core/core-flows/src/cart/steps/reserve-inventory.ts`

这个步骤使用与库存确认相同的 `location_ids` 来预留库存：

```typescript
// 输入数据结构与 confirmInventoryStep 相同
export interface ReserveInventoryItem {
  inventory_item_id: string
  required_quantity: number
  allow_backorder: boolean
  quantity: BigNumberInput
  location_ids: string[]  // 基于 sales_channel_id 过滤得到
}
```

---

## 8. 完整链路总结

### 8.1 场景一：publishable key 绑定单个 sales channel

**请求**:
```
POST /store/carts
Headers:
  x-publishable-api-key: pk_test_123
Body:
  {}  // 没有 sales_channel_id
```

**处理流程**:
1. `maybeAttachPublishableKeyScopes`: 查询到 `pk_test_123` 关联 `["sc_001"]`
2. `ensurePublishableKeyAndSalesChannelMatch`:
   - `channelId` 为 `undefined`
   - `pubKeySalesChannels.length === 1`
   - **自动填充**: `req.validatedBody.sales_channel_id = "sc_001"`
   - `req.errors` 为空
3. `wrapHandler`: 无错误，继续执行
4. `createCartWorkflow`:
   - `findSalesChannelStep`: 找到 `sc_001`
   - `validateSalesChannelStep`: 通过
   - `confirmVariantInventoryWorkflow`: 使用 `sc_001` 检查库存位置
   - 创建 cart，`sales_channel_id = "sc_001"`

### 8.2 场景二：publishable key 绑定多个 sales channel，未指定 ID

**请求**:
```
POST /store/carts
Headers:
  x-publishable-api-key: pk_test_456  // 绑定了 ["sc_001", "sc_002"]
Body:
  {}  // 没有 sales_channel_id
```

**处理流程**:
1. `maybeAttachPublishableKeyScopes`: 查询到 `["sc_001", "sc_002"]`
2. `ensurePublishableKeyAndSalesChannelMatch`:
   - `channelId` 为 `undefined`
   - `pubKeySalesChannels.length > 1`
   - **累积错误**: `req.errors.push("Cannot assign sales channel...")`
3. `wrapHandler`:
   - 检测到 `req.errors.length > 0`
   - **返回 400 错误**:
     ```json
     {
       "errors": ["Cannot assign sales channel to cart. The Publishable API Key in the header has multiple associated sales channels. Please provide a sales channel ID in the request body."],
       "message": "Provided request body contains errors. Please check the data and retry the request"
     }
     ```
4. 路由处理函数**不会被执行**

### 8.3 场景三：publishable key 绑定多个 sales channel，指定了有效 ID

**请求**:
```
POST /store/carts
Headers:
  x-publishable-api-key: pk_test_456  // 绑定了 ["sc_001", "sc_002"]
Body:
  { "sales_channel_id": "sc_001" }
```

**处理流程**:
1. `maybeAttachPublishableKeyScopes`: 查询到 `["sc_001", "sc_002"]`
2. `ensurePublishableKeyAndSalesChannelMatch`:
   - `channelId = "sc_001"`
   - `pubKeySalesChannels.includes("sc_001")` 为 `true`
   - `req.errors` 为空
3. `wrapHandler`: 无错误，继续执行
4. `createCartWorkflow`: 使用 `sc_001` 创建 cart

### 8.4 场景四：publishable key 绑定多个 sales channel，指定了无效 ID

**请求**:
```
POST /store/carts
Headers:
  x-publishable-api-key: pk_test_456  // 绑定了 ["sc_001", "sc_002"]
Body:
  { "sales_channel_id": "sc_999" }  // 不在绑定范围内
```

**处理流程**:
1. `maybeAttachPublishableKeyScopes`: 查询到 `["sc_001", "sc_002"]`
2. `ensurePublishableKeyAndSalesChannelMatch`:
   - `channelId = "sc_999"`
   - `pubKeySalesChannels.includes("sc_999")` 为 `false`
   - **累积错误**: `req.errors.push("Sales channel ID in payload sc_999 is not associated...")`
3. `wrapHandler`: 返回 400 错误

---

## 9. 关键代码位置汇总

| 功能 | 文件路径 |
|------|----------|
| 中间件：attach publishable key scopes | `packages/medusa/src/api/utils/middlewares/common/maybe-attach-pub-key-scopes.ts` |
| 中间件：检查并自动填充 sales channel | `packages/medusa/src/api/utils/middlewares/common/ensure-pub-key-sales-channel-match.ts` |
| 错误处理包装器 | `packages/core/framework/src/http/utils/wrap-handler.ts` |
| 创建 cart 工作流 | `packages/core/core-flows/src/cart/workflows/create-carts.ts` |
| 查找 sales channel 步骤 | `packages/core/core-flows/src/cart/steps/find-sales-channel.ts` |
| 验证 sales channel 步骤 | `packages/core/core-flows/src/cart/steps/validate-sales-channel.ts` |
| 库存确认输入准备 | `packages/core/core-flows/src/cart/utils/prepare-confirm-inventory-input.ts` |
| 库存确认步骤 | `packages/core/core-flows/src/cart/steps/confirm-inventory.ts` |
| 完成 cart 工作流 | `packages/core/core-flows/src/cart/workflows/complete-cart.ts` |
| 库存预留步骤 | `packages/core/core-flows/src/cart/steps/reserve-inventory.ts` |

---

## 10. 结论

1. **自动填充条件**: 只有当 `x-publishable-api-key` 绑定了**恰好一个** sales channel，且请求 body 中没有提供 `sales_channel_id` 时，系统才会自动填充。

2. **错误累积机制**: 错误不会立即抛出，而是累积到 `req.errors` 数组中，由 `wrapHandler` 在路由处理前统一检查并返回 400 响应。

3. **库存关联**: `sales_channel_id` 决定了哪些库存位置（stock locations）可以被使用。系统会验证变体在指定的 sales channel 中有对应的库存位置，并只在这些位置中检查和预留库存。

4. **链路一致性**: `createCartWorkflow` 和 `completeCartWorkflow` 都依赖于同一个 `sales_channel_id`，确保了从创建 cart 到完成订单的整个流程中，库存检查和预留都基于同一个 sales channel 进行。
