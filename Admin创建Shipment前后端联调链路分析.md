# Admin 创建 Shipment 的前后端联调链路分析

## 1. 前端表单数据整理流程

### 1.1 表单组件结构

前端的 Shipment 创建表单组件位于 `packages/admin/dashboard/src/routes/orders/order-create-shipment/components/order-create-shipment-form/order-create-shipment-form.tsx`。

#### 关键字段定义

表单使用 `CreateShipmentSchema` 进行验证，定义在 `constants.ts` 中：

```typescript
export const CreateShipmentSchema = z.object({
  labels: z.array(
    z.object({
      tracking_number: z.string(),
      tracking_url: z.string().optional(),
      label_url: z.string().optional(),
    })
  ),
  send_notification: z.boolean().optional(),
})
```

#### 表单默认值

```typescript
const form = useForm<zod.infer<typeof CreateShipmentSchema>>({
  defaultValues: {
    send_notification: !order.no_notification,
  },
  resolver: zodResolver(CreateShipmentSchema),
})
```

### 1.2 数据整理逻辑

在 `handleSubmit` 函数中，表单数据被整理成 API 请求的 payload：

#### Labels 字段处理

```typescript
const addedLabels = data.labels
  .filter((l) => !!l.tracking_number || !!l.tracking_url || !!l.label_url)
  .map((l) => ({
    tracking_number: l.tracking_number,
    tracking_url: l.tracking_url || "#",
    label_url: l.label_url || "#",
  }))
```

**处理逻辑**：
1. **过滤**：只保留至少有一个非空字段的 label（tracking_number、tracking_url 或 label_url）
2. **映射**：将表单数据映射为 API 所需的格式，空字段默认设为 "#"

#### Send_notification 字段处理

```typescript
no_notification: !data.send_notification,
```

**处理逻辑**：
1. **取反**：将表单中的 `send_notification` 布尔值取反，转换为 API 所需的 `no_notification` 字段
2. **语义转换**：
   - 表单中 `send_notification: true` → API 中 `no_notification: false`（发送通知）
   - 表单中 `send_notification: false` → API 中 `no_notification: true`（不发送通知）

#### 完整 Payload 构建

```typescript
await createShipment(
  {
    items:
      fulfillment?.items
        ?.filter((i) => !!i.line_item_id)
        .map((i) => ({
          id: i.line_item_id!,
          quantity: i.quantity,
        })) || [],
    labels: [...addedLabels, ...(fulfillment?.labels || [])],
    no_notification: !data.send_notification,
  },
  // ...
)
```

**Payload 组成**：
1. **items**：从 fulfillment 中提取的商品项，过滤掉没有 line_item_id 的项
2. **labels**：合并表单中新增的 labels 和 fulfillment 中已有的 labels
3. **no_notification**：转换后的通知开关字段

## 2. 后端 API 路由参数处理

### 2.1 路由定义

后端的 Shipment 创建 API 路由位于 `packages/medusa/src/api/admin/orders/[id]/fulfillments/[fulfillment_id]/shipments/route.ts`。

#### 路由路径

```
POST /admin/orders/[id]/fulfillments/[fulfillment_id]/shipments
```

### 2.2 参数补齐逻辑

```typescript
const input: CreateOrderShipmentWorkflowInput = {
  ...req.validatedBody,
  order_id: req.params.id,
  fulfillment_id: req.params.fulfillment_id,
  labels: req.validatedBody.labels ?? [],
  created_by: req.auth_context.actor_id,
}
```

#### 各字段来源分析

| 字段 | 来源 | 说明 |
|------|------|------|
| order_id | `req.params.id` | 从 URL 路径参数中获取，对应订单 ID |
| fulfillment_id | `req.params.fulfillment_id` | 从 URL 路径参数中获取，对应履行单 ID |
| created_by | `req.auth_context.actor_id` | 从认证上下文中获取，对应当前登录的管理员 ID |
| labels | `req.validatedBody.labels ?? []` | 从请求体中获取，默认为空数组 |
| 其他字段 | `req.validatedBody` | 从请求体中获取（如 items、no_notification 等） |

### 2.3 工作流调用

```typescript
await createOrderShipmentWorkflow(req.scope).run({
  input,
})
```

**处理流程**：
1. 从请求范围（`req.scope`）中解析出 `createOrderShipmentWorkflow` 工作流
2. 将构建好的 `input` 对象作为参数传入工作流
3. 执行工作流完成 Shipment 创建

## 3. createOrderShipmentWorkflow 工作流分析

### 3.1 工作流整体结构

工作流定义位于 `packages/core/core-flows/src/order/workflows/create-shipment.ts`。

#### 工作流执行流程

```
1. 查询订单数据
      ↓
2. 验证订单状态
      ↓
3. 准备两个数据流
      ↓
4. 并行执行两个操作
   ├── createShipmentWorkflow（fulfillment 工作流）
   └── registerOrderShipmentStep（order module 逻辑）
      ↓
5. 发送 SHIPMENT_CREATED 事件
      ↓
6. 执行 shipmentCreated 钩子
```

### 3.2 为什么需要两个并行操作？

在工作流中，使用 `parallelize` 并行执行了两个关键操作：

```typescript
const [shipment] = parallelize(
  createShipmentWorkflow.runAsStep({
    input: fulfillmentData,
  }),
  registerOrderShipmentStep(shipmentData)
)
```

#### 设计原因分析

**1. 模块化架构设计**

Medusa 采用了模块化的架构设计，不同的模块负责不同的业务领域：
- **Fulfillment 模块**：负责配送履行相关的逻辑（如物流信息、发货状态）
- **Order 模块**：负责订单核心逻辑（如订单状态、库存扣减、订单历史）

**2. 关注点分离**

两个操作关注的是不同的业务维度：
- **fulfillment 工作流**：关注"如何发货"（物流信息、配送状态）
- **order module 逻辑**：关注"订单发生了什么变化"（订单状态、库存、历史记录）

**3. 事务一致性**

使用 `parallelize` 并行执行可以：
- 提高执行效率
- 通过工作流的补偿机制保证事务一致性
- 如果其中一个操作失败，另一个操作可以通过补偿函数回滚

**4. 数据模型分离**

从代码中可以看到两个操作使用不同的数据输入：

```typescript
// 用于 fulfillment 工作流的数据
const fulfillmentData = transform({ input }, ({ input }) => {
  return {
    id: input.fulfillment_id,
    labels: input.labels ?? [],
    marked_shipped_by: input.created_by,
  }
})

// 用于 order module 的数据
const shipmentData = transform(
  { order, input },
  prepareRegisterShipmentData
)
```

## 4. 两个分支的职责对比

### 4.1 Fulfillment 工作流（createShipmentWorkflow）

#### 工作流定义

位于 `packages/core/core-flows/src/fulfillment/workflows/create-shipment.ts`

#### 执行流程

```
1. validateShipmentStep（验证发货）
      ↓
2. 设置 shipped_at 时间戳
      ↓
3. 调用 updateFulfillmentWorkflow 更新 fulfillment
```

#### 各步骤详细分析

##### 步骤 1：validateShipmentStep

位于 `packages/core/core-flows/src/fulfillment/steps/validate-shipment.ts`

**验证逻辑**：

```typescript
const fulfillment = await service.retrieveFulfillment(id, {
  select: ["shipped_at", "canceled_at", "shipping_option_id"],
})

// 检查是否已经发货
if (fulfillment.shipped_at) {
  throw new MedusaError(
    MedusaError.Types.NOT_ALLOWED,
    "Shipment has already been created"
  )
}

// 检查是否已取消
if (fulfillment.canceled_at) {
  throw new MedusaError(
    MedusaError.Types.NOT_ALLOWED,
    "Cannot create shipment for a canceled fulfillment"
  )
}

// 检查是否有配送选项
if (!fulfillment.shipping_option_id) {
  throw new MedusaError(
    MedusaError.Types.NOT_ALLOWED,
    "Cannot create shipment without a Shipping Option"
  )
}
```

**验证要点**：
1. **未发货检查**：确保 fulfillment 尚未被标记为已发货
2. **未取消检查**：确保 fulfillment 未被取消
3. **配送选项检查**：确保 fulfillment 关联了有效的配送选项

##### 步骤 2：设置 shipped_at 时间戳

```typescript
const update = transform({ input }, (data) => ({
  ...data.input,
  shipped_at: new Date(),
}))
```

**作用**：为 fulfillment 添加当前时间作为发货时间

##### 步骤 3：updateFulfillmentStep

位于 `packages/core/core-flows/src/fulfillment/steps/update-fulfillment.ts`

**核心逻辑**：

```typescript
const { id, ...data } = input

const service = container.resolve<IFulfillmentModuleService>(
  Modules.FULFILLMENT
)

const updated = await service.updateFulfillment(id, data)

return new StepResponse(updated, fulfillment)
```

**补偿逻辑**：

```typescript
async (fulfillment, { container }) => {
  if (!fulfillment) {
    return
  }

  const service = container.resolve<IFulfillmentModuleService>(
    Modules.FULFILLMENT
  )
  const { id, ...data } = fulfillment

  await service.updateFulfillment(id, data)
}
```

#### Fulfillment 工作流维护的状态

| 状态字段 | 说明 | 数据来源 |
|---------|------|---------|
| shipped_at | 发货时间戳 | 当前时间 `new Date()` |
| labels | 物流标签信息（tracking_number、tracking_url、label_url） | 前端表单输入 |
| marked_shipped_by | 标记发货的管理员 ID | 后端从 auth_context 获取 |

**Fulfillment 工作流的核心职责**：
1. **验证发货可行性**：确保 fulfillment 处于可发货状态
2. **记录物流信息**：保存物流单号、追踪 URL、标签 URL 等
3. **标记发货状态**：设置 `shipped_at` 时间戳，标记 fulfillment 为已发货
4. **记录操作人**：保存执行发货操作的管理员 ID

### 4.2 Order Module 的 Register Shipment 逻辑

#### 执行流程

```
1. registerOrderShipmentStep 被调用
      ↓
2. 调用 orderModuleService.registerShipment
      ↓
3. 创建 SHIP_ITEM 类型的 order change actions
      ↓
4. 调用 createOrderChange_ 创建订单变更
      ↓
5. 调用 confirmOrderChange 确认订单变更
```

#### 各步骤详细分析

##### 步骤 1：registerOrderShipmentStep

位于 `packages/core/core-flows/src/order/steps/register-shipment.ts`

**核心逻辑**：

```typescript
export const registerOrderShipmentStep = createStep(
  registerOrderShipmentStepId,
  async (data: RegisterOrderShipmentDTO, { container }) => {
    const service = container.resolve<IOrderModuleService>(Modules.ORDER)

    await service.registerShipment(data)
    return new StepResponse(void 0, data.order_id)
  },
  async (orderId, { container }) => {
    if (!orderId) {
      return
    }

    const service = container.resolve<IOrderModuleService>(Modules.ORDER)

    await service.revertLastVersion(orderId)
  }
)
```

**补偿逻辑**：
- 调用 `revertLastVersion` 方法回滚订单到上一个版本
- 这体现了订单模块的版本控制机制

##### 步骤 2：orderModuleService.registerShipment

位于 `packages/modules/order/src/services/actions/register-shipment.ts`

**核心逻辑**：

```typescript
export async function registerShipment(
  this: any,
  data: OrderTypes.RegisterOrderShipmentDTO,
  sharedContext?: Context
): Promise<void> {
  let shippingMethodId

  // 1. 创建 SHIP_ITEM 类型的 actions
  const actions: CreateOrderChangeActionDTO[] = data.items!.map((item) => {
    return {
      action: ChangeActionType.SHIP_ITEM,
      internal_note: item.internal_note,
      reference: data.reference,
      reference_id: data.reference_id,
      details: {
        reference_id: item.id,
        quantity: item.quantity,
        metadata: item.metadata,
      },
    }
  })

  // 2. 创建订单变更
  const change = await this.createOrderChange_(
    {
      order_id: data.order_id,
      description: data.description,
      internal_note: data.internal_note,
      created_by: data.created_by,
      metadata: data.metadata,
      actions,
    },
    sharedContext
  )

  // 3. 确认订单变更
  await this.confirmOrderChange(change[0].id, sharedContext)
}
```

#### Order Module 维护的状态

| 状态维度 | 说明 | 数据结构 |
|---------|------|---------|
| 订单变更历史 | 记录订单的所有变更操作 | OrderChange 实体 |
| 变更操作详情 | 记录每次发货的具体商品和数量 | OrderChangeAction 实体（action: SHIP_ITEM） |
| 订单版本控制 | 支持订单状态的回滚 | 版本号机制 |
| 操作人记录 | 记录执行变更的管理员 | created_by 字段 |

**Order Module 的核心职责**：

1. **订单状态变更管理**：
   - 通过 `OrderChange` 机制记录订单的所有状态变更
   - 每个发货操作都会创建一个 `OrderChange` 记录

2. **商品发货追踪**：
   - 为每个发货的商品创建 `SHIP_ITEM` 类型的 `OrderChangeAction`
   - 记录商品 ID、发货数量等详细信息
   - 关联到对应的 fulfillment（通过 reference 和 reference_id）

3. **版本控制与回滚**：
   - 维护订单的版本历史
   - 支持通过 `revertLastVersion` 回滚到上一个版本
   - 这在工作流补偿机制中起到关键作用

4. **审计追踪**：
   - 记录操作人（created_by）
   - 记录操作时间
   - 支持内部备注（internal_note）和描述（description）

## 5. 两个分支的对比总结

### 5.1 职责对比表

| 维度 | Fulfillment 工作流 | Order Module 逻辑 |
|------|-------------------|-------------------|
| **模块归属** | Fulfillment 模块 | Order 模块 |
| **核心职责** | 管理配送履行状态 | 管理订单状态变更 |
| **关注焦点** | "如何发货"（物流信息） | "订单发生了什么变化" |
| **数据模型** | Fulfillment 实体 | OrderChange、OrderChangeAction 实体 |
| **状态字段** | shipped_at、labels、marked_shipped_by | 订单版本、变更历史、操作记录 |
| **补偿机制** | 恢复 fulfillment 原始数据 | 回滚订单到上一个版本 |
| **业务意义** | 物流配送层面的发货 | 订单层面的发货确认 |

### 5.2 为什么需要两个独立的操作？

#### 1. 业务领域分离

**Fulfillment 模块**：
- 属于"配送履行"领域
- 关注物流、配送、发货等实物流动过程
- 数据可能需要与外部物流系统集成

**Order 模块**：
- 属于"订单管理"领域
- 关注订单状态、库存、财务等核心业务流程
- 需要维护订单的完整历史记录

#### 2. 数据模型不同

**Fulfillment 数据**：
- 一对一关系：一个 fulfillment 对应一次发货
- 扁平结构：直接更新 fulfillment 实体的字段
- 物流信息：tracking_number、tracking_url 等

**Order 数据**：
- 一对多关系：一个 order 可以有多个变更
- 事件溯源风格：通过 OrderChange 记录所有变更
- 历史追踪：支持查看订单的完整变更历史

#### 3. 事务边界不同

虽然两个操作在工作流中并行执行，但它们有各自的事务边界：

- **Fulfillment 操作**：更新 fulfillment 表
- **Order 操作**：插入 order_change 和 order_change_action 表

通过工作流的补偿机制，确保两个操作的最终一致性。

#### 4. 扩展点不同

**Fulfillment 模块**：
- 可以对接不同的物流服务商
- 可以自定义发货验证逻辑
- 可以集成外部配送系统

**Order 模块**：
- 可以自定义订单状态机
- 可以添加自定义的变更操作类型
- 可以集成库存、财务等其他模块

### 5.3 数据流汇总

让我们通过一个完整的数据流图来总结整个流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端表单输入                                │
├─────────────────────────────────────────────────────────────────┤
│  labels: [                                                        │
│    { tracking_number: "12345", tracking_url: "...", label_url: "..." } │
│  ]                                                                │
│  send_notification: true                                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        前端数据整理                                │
├─────────────────────────────────────────────────────────────────┤
│  1. 过滤并映射 labels（空字段设为 "#"）                           │
│  2. send_notification → no_notification（取反）                  │
│  3. 从 fulfillment 提取 items                                     │
│  4. 合并新增 labels 和已有 labels                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        后端 API 路由                               │
├─────────────────────────────────────────────────────────────────┤
│  从 URL 参数获取：order_id、fulfillment_id                        │
│  从 auth_context 获取：created_by                                 │
│  合并到 input：{                                                  │
│    ...req.validatedBody,                                          │
│    order_id: req.params.id,                                       │
│    fulfillment_id: req.params.fulfillment_id,                    │
│    created_by: req.auth_context.actor_id                         │
│  }                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  createOrderShipmentWorkflow                      │
├─────────────────────────────────────────────────────────────────┤
│  1. 查询订单数据（包含 fulfillments、items 等）                   │
│  2. 验证订单状态（未取消、商品存在、fulfillment 存在）            │
│  3. 准备两个数据流：                                               │
│     - fulfillmentData（用于 fulfillment 工作流）                  │
│     - shipmentData（用于 order module）                           │
│  4. parallelize 并行执行：                                         │
│     ┌─────────────────────┐    ┌─────────────────────────┐      │
│     │ createShipmentWorkflow│   │ registerOrderShipmentStep│     │
│     │  (Fulfillment 模块)  │   │    (Order 模块)          │      │
│     └─────────────────────┘    └─────────────────────────┘      │
│  5. 发送 SHIPMENT_CREATED 事件                                    │
│  6. 执行 shipmentCreated 钩子                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
              ┌───────────────┴───────────────┐
              ↓                               ↓
┌─────────────────────────┐      ┌─────────────────────────────┐
│   Fulfillment 工作流     │      │    Order Module 逻辑         │
├─────────────────────────┤      ├─────────────────────────────┤
│ 1. validateShipmentStep │      │ 1. 准备 shipmentData         │
│    - 检查 shipped_at     │      │    - 计算实际发货数量         │
│    - 检查 canceled_at    │      │    - 处理 inventory kit      │
│    - 检查 shipping_option│      │                              │
│                         │      │ 2. registerOrderShipmentStep │
│ 2. 设置 shipped_at      │      │    - 创建 OrderChange         │
│                         │      │    - 创建 SHIP_ITEM actions   │
│ 3. updateFulfillmentStep│      │    - 确认订单变更             │
│    - 更新 fulfillment    │      │                              │
│    - 添加 labels         │      │ 3. 版本控制                   │
│    - 设置 marked_shipped │      │    - 支持 revertLastVersion  │
│                         │      │                              │
│ 维护的状态：              │      │ 维护的状态：                   │
│ - shipped_at            │      │ - OrderChange 历史            │
│ - labels[]              │      │ - OrderChangeAction 详情      │
│ - marked_shipped_by     │      │ - 订单版本号                   │
│ - fulfillment 状态      │      │ - created_by 审计             │
└─────────────────────────┘      └─────────────────────────────┘
```

## 6. 关键代码位置索引

为了方便后续查阅，以下是本次分析涉及的关键代码文件位置：

### 6.1 前端代码

| 文件路径 | 说明 |
|---------|------|
| `packages/admin/dashboard/src/routes/orders/order-create-shipment/components/order-create-shipment-form/order-create-shipment-form.tsx` | Shipment 创建表单组件，包含数据整理逻辑 |
| `packages/admin/dashboard/src/routes/orders/order-create-shipment/components/order-create-shipment-form/constants.ts` | 表单验证 schema 定义 |
| `packages/admin/dashboard/src/hooks/api/orders.tsx` | `useCreateOrderShipment` 钩子定义 |

### 6.2 后端 API 代码

| 文件路径 | 说明 |
|---------|------|
| `packages/medusa/src/api/admin/orders/[id]/fulfillments/[fulfillment_id]/shipments/route.ts` | 订单 Shipment 创建 API 路由 |
| `packages/medusa/src/api/admin/fulfillments/[id]/shipment/route.ts` | Fulfillment Shipment 创建 API 路由（独立接口） |

### 6.3 工作流代码

| 文件路径 | 说明 |
|---------|------|
| `packages/core/core-flows/src/order/workflows/create-shipment.ts` | `createOrderShipmentWorkflow` 主工作流 |
| `packages/core/core-flows/src/fulfillment/workflows/create-shipment.ts` | `createShipmentWorkflow` Fulfillment 工作流 |
| `packages/core/core-flows/src/fulfillment/workflows/update-fulfillment.ts` | `updateFulfillmentWorkflow` 更新 Fulfillment 工作流 |

### 6.4 Step 代码

| 文件路径 | 说明 |
|---------|------|
| `packages/core/core-flows/src/fulfillment/steps/validate-shipment.ts` | `validateShipmentStep` 发货验证步骤 |
| `packages/core/core-flows/src/fulfillment/steps/update-fulfillment.ts` | `updateFulfillmentStep` 更新 Fulfillment 步骤 |
| `packages/core/core-flows/src/order/steps/register-shipment.ts` | `registerOrderShipmentStep` 注册订单发货步骤 |

### 6.5 Module Service 代码

| 文件路径 | 说明 |
|---------|------|
| `packages/modules/order/src/services/actions/register-shipment.ts` | Order 模块 `registerShipment` 方法实现 |
| `packages/modules/order/src/services/order-module-service.ts` | Order 模块主服务 |

## 7. 总结

### 7.1 核心设计思想

Medusa 在设计"创建 Shipment"这一业务流程时，采用了**模块化架构**和**关注点分离**的设计思想：

1. **模块化架构**：将配送履行（Fulfillment）和订单管理（Order）划分为两个独立的模块，各自负责不同的业务领域。

2. **关注点分离**：
   - Fulfillment 模块关注"如何发货"（物流信息、配送状态）
   - Order 模块关注"订单发生了什么变化"（订单状态、历史记录）

3. **最终一致性**：通过工作流的补偿机制，确保两个模块的操作最终一致。

### 7.2 数据流回顾

1. **前端**：
   - 收集 `labels` 和 `send_notification` 字段
   - 将 `send_notification` 转换为 `no_notification`（取反）
   - 过滤和整理 `labels` 数组
   - 调用 API 发送请求

2. **后端 API**：
   - 从 URL 参数获取 `order_id` 和 `fulfillment_id`
   - 从认证上下文获取 `created_by`
   - 合并参数后调用工作流

3. **工作流**：
   - 验证订单状态
   - 并行执行两个操作：
     - **Fulfillment 工作流**：更新 fulfillment 状态，记录物流信息
     - **Order Module 逻辑**：创建订单变更记录，维护订单历史

### 7.3 两个分支的职责总结

| 分支 | 核心职责 | 维护的状态 |
|------|---------|-----------|
| **Fulfillment 工作流** | 管理配送履行状态 | `shipped_at`、`labels`、`marked_shipped_by`、fulfillment 发货状态 |
| **Order Module 逻辑** | 管理订单状态变更 | `OrderChange` 历史、`OrderChangeAction` 详情、订单版本号、审计记录 |

这种设计使得系统更加灵活和可扩展：
- Fulfillment 模块可以独立对接不同的物流服务商
- Order 模块可以独立维护订单的完整历史和状态机
- 两个模块通过工作流协调，保证业务流程的完整性
