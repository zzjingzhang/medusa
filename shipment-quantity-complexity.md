# 创建 Shipment 时数量换算的隐藏复杂度

## 问题背景

在 Medusa 电商系统中，创建 Shipment（发货单）时存在一个容易被忽略的数量换算问题。这个问题的核心在于：当 Order Item（订单项）背后关联的是 **Inventory Kit（库存套件）** 时，Fulfillment Item Quantity（履约项数量）与 Shipment Item Quantity（发货项数量）之间并不是 1:1 的关系。

## 为什么 Fulfillment Item Quantity 不能直接等于 Shipment Item Quantity

### 1. 概念区分

首先需要明确几个核心概念：

- **Order Line Item**：用户购买的商品项，数量以"件"为单位（如：用户购买了 2 个"相机套件"）
- **Inventory Item**：实际的库存商品项（如：相机机身、镜头、电池）
- **Inventory Kit**：一个 Product Variant（商品规格）由多个 Inventory Items 组成（如："相机套件" = 1 个相机机身 + 1 个镜头 + 2 个电池）
- **Fulfillment Item**：履约时的实际库存操作项，记录的是实际扣减的库存数量
- **Shipment Item**：发货单中的商品项，记录的是发给用户的商品数量

### 2. 创建 Fulfillment 时的数量放大

当创建 Fulfillment（履约单）时，系统会对每个关联的 Inventory Item 创建独立的 Fulfillment Item，并对数量进行放大。

**关键代码位置**：`packages/core/core-flows/src/order/workflows/create-fulfillment.ts:209-228`

```typescript
// if line item is from a managed variant, create a fulfillment item for each reservation item
return reservations.map((r) => {
  const iItem = orderItem?.variant?.inventory_items.find(
    (ii) => ii.inventory.id === r.inventory_item_id
  )

  return {
    line_item_id: i.id,
    inventory_item_id: r.inventory_item_id,
    quantity: MathBN.mult(
      iItem?.required_quantity ?? 1,
      i.quantity
    ) as BigNumberInput,
    // ... 其他字段
  }
})
```

**数量计算逻辑**：

```
Fulfillment Item Quantity = Order Item Quantity × Inventory Item Required Quantity
```

### 3. 实际案例演示

假设有一个"相机套件"商品，其 Inventory Kit 配置如下：

| Inventory Item | Required Quantity |
|----------------|-------------------|
| 相机机身       | 1                 |
| 镜头           | 1                 |
| 电池           | 2                 |

当用户购买 **2 个** 相机套件时：

**Order Line Item**：
- quantity = 2

**创建 Fulfillment 后**（生成 3 个 Fulfillment Items）：

| Fulfillment Item | Inventory Item | Quantity 计算 | Fulfillment Quantity |
|------------------|----------------|---------------|----------------------|
| 1                | 相机机身       | 2 × 1         | 2                    |
| 2                | 镜头           | 2 × 1         | 2                    |
| 3                | 电池           | 2 × 2         | 4                    |

**问题出现**：

- Fulfillment Items 的数量分别是：2、2、4
- 但 Shipment Item 需要记录的是：发给用户 **2 个** 相机套件
- 如果直接使用 Fulfillment Item Quantity（如 2 或 4）作为 Shipment Item Quantity，就会出错

## 代码如何借助 `variant.inventory_items.required_quantity` 做反推

### 1. 反推逻辑的核心实现

在创建 Shipment 时，系统需要从 Fulfillment Item Quantity 反推回 Order Item Quantity（即 Shipment Item Quantity）。

**关键代码位置**：`packages/core/core-flows/src/order/workflows/create-shipment.ts:139-153`

```typescript
// NOTE: if the order item has an inventory kit or `required_qunatity` > 1, fulfillment items wont't match 1:1 with order items.
// - for each inventory item in the kit, a fulfillment item will be created i.e. one line item could have multiple fulfillment items
// - the quantity of the fulfillment item will be the quantity of the order item multiplied by the required quantity of the inventory item
//
//   We need to take this into account when creating a shipment to compute quantity of line items being shipped based on fulfillment items and qunatities.
//   NOTE: for now we only need to find one inventory item of a line item to compute this since when a fulfillment is created all inventory items are fulfilled together.
//   If we allow to cancel partial fulfillments for an order item, we need to change this.

if (iitems?.length) {
  const iitem = iitems.find(
    (i) => i.inventory.id === fitem.inventory_item_id
  )

  quantity = MathBN.div(quantity, iitem!.required_quantity)
}
```

### 2. 反推公式

```
Shipment Item Quantity = Fulfillment Item Quantity ÷ Inventory Item Required Quantity
```

### 3. 反推案例演示

继续使用之前的相机套件案例：

**Fulfillment Items**：

| Fulfillment Item | Inventory Item | Fulfillment Quantity | Required Quantity |
|------------------|----------------|----------------------|-------------------|
| 1                | 相机机身       | 2                    | 1                 |
| 2                | 镜头           | 2                    | 1                 |
| 3                | 电池           | 4                    | 2                 |

**反推计算**：

对于相机机身：`2 ÷ 1 = 2` ✓

对于镜头：`2 ÷ 1 = 2` ✓

对于电池：`4 ÷ 2 = 2` ✓

**结果**：无论使用哪个 Fulfillment Item 计算，都得到 Shipment Item Quantity = 2，这正是正确的发货数量。

### 4. 代码中的关键假设

代码中有一个非常重要的假设（注释中明确说明）：

> **"for now we only need to find one inventory item of a line item to compute this since when a fulfillment is created all inventory items are fulfilled together."**

翻译：目前我们只需要找到一个订单项的库存项来计算这个数量，因为创建履约时所有库存项都是一起履约的。

这个假设是当前实现能够正常工作的基石：

- 假设 1：同一个 Order Line Item 的所有 Fulfillment Items 使用相同的 Order Item Quantity
- 假设 2：创建 Fulfillment 后，各 Inventory Items 的数量比例保持不变

基于这两个假设，使用任意一个 Fulfillment Item 进行反推都能得到正确的结果。

## 如果未来支持部分取消 Fulfillment，这段逻辑为什么可能失效

### 1. 什么是"部分取消 Fulfillment"

当前系统中，Fulfillment 的取消通常是**全量取消**。如果未来支持**部分取消**，意味着：

- 用户可以取消 Fulfillment 中的部分商品，而不是全部
- 例如：原本要发货 2 个相机套件，现在只想发 1 个
- 或者更复杂的情况：只取消部分 Inventory Items（虽然这在业务逻辑上不太合理）

### 2. 失效场景分析

假设有这样一个场景：

1. **初始状态**：创建 Fulfillment，发货 2 个相机套件

| Fulfillment Item | Inventory Item | Fulfillment Quantity | Required Quantity |
|------------------|----------------|----------------------|-------------------|
| 1                | 相机机身       | 2                    | 1                 |
| 2                | 镜头           | 2                    | 1                 |
| 3                | 电池           | 4                    | 2                 |

2. **部分取消后**：用户取消了 1 个相机套件的发货

理想情况下，Fulfillment Items 应该变成：

| Fulfillment Item | Inventory Item | Fulfillment Quantity | Required Quantity |
|------------------|----------------|----------------------|-------------------|
| 1                | 相机机身       | 1                    | 1                 |
| 2                | 镜头           | 1                    | 1                 |
| 3                | 电池           | 2                    | 2                 |

**如果部分取消实现不正确**，可能出现的情况：

| Fulfillment Item | Inventory Item | Fulfillment Quantity | Required Quantity |
|------------------|----------------|----------------------|-------------------|
| 1                | 相机机身       | 1                    | 1                 |
| 2                | 镜头           | 2                    | 1                 | （未正确更新）
| 3                | 电池           | 2                    | 2                 |

3. **创建 Shipment 时**：

- 如果代码恰好选中了**相机机身**的 Fulfillment Item：`1 ÷ 1 = 1` ✓
- 如果代码恰好选中了**镜头**的 Fulfillment Item：`2 ÷ 1 = 2` ✗

**问题出现**：由于各 Fulfillment Items 的数量不再保持相同比例，使用不同的 Fulfillment Item 进行反推会得到不同的结果。

### 3. 更深层的问题

代码中的逻辑假设是：

```typescript
const iitem = iitems.find(
  (i) => i.inventory.id === fitem.inventory_item_id
)

quantity = MathBN.div(quantity, iitem!.required_quantity)
```

这段代码只使用了**第一个匹配**的 Fulfillment Item 来计算，而没有：
1. 验证所有 Fulfillment Items 的计算结果是否一致
2. 处理数量不一致的异常情况

### 4. 未来可能的修复方向

如果未来支持部分取消 Fulfillment，需要考虑以下修改：

#### 方案 A：确保部分取消时保持比例一致性

在实现部分取消功能时，确保对同一个 Order Line Item 的所有 Fulfillment Items 进行**按比例更新**。

例如：取消 1 个相机套件时，需要同时：
- 相机机身：2 → 1（减少 1）
- 镜头：2 → 1（减少 1）
- 电池：4 → 2（减少 2）

这样可以保持原有的假设成立。

#### 方案 B：修改 Shipment 创建逻辑，增加一致性校验

在 `prepareRegisterShipmentData` 函数中，增加对所有 Fulfillment Items 的校验：

```typescript
// 伪代码思路
if (iitems?.length) {
  const calculatedQuantities = fulfillmentItems.map((fitem) => {
    const iitem = iitems.find(
      (i) => i.inventory.id === fitem.inventory_item_id
    )
    return MathBN.div(fitem.quantity, iitem!.required_quantity)
  })
  
  // 校验所有计算结果是否一致
  if (!allEqual(calculatedQuantities)) {
    throw new MedusaError(
      MedusaError.Types.INVALID_DATA,
      "Fulfillment item quantities are inconsistent for inventory kit"
    )
  }
  
  quantity = calculatedQuantities[0]
}
```

#### 方案 C：使用更可靠的数据来源

不依赖 Fulfillment Items 来反推，而是直接从其他数据来源获取正确的 Shipment Item Quantity：
- Order Line Item 原始数量
- Fulfillment 创建时记录的原始数量
- 或者专门的 Shipment Items 输入参数

## 代码位置总结

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 创建 Fulfillment 时数量放大 | `packages/core/core-flows/src/order/workflows/create-fulfillment.ts` | 209-228 |
| 创建 Shipment 时数量反推 | `packages/core/core-flows/src/order/workflows/create-shipment.ts` | 139-153 |
| Fulfillment Item 模型定义 | `packages/modules/fulfillment/src/models/fulfillment-item.ts` | 1-27 |

## 总结

1. **数量换算的本质**：
   - Fulfillment Item 记录的是**实际库存操作数量**（Inventory Level 视角）
   - Shipment Item 记录的是**用户购买商品数量**（Order Line Item 视角）
   - 当存在 Inventory Kit 时，两者之间存在 `× required_quantity` 的换算关系

2. **当前实现的依赖**：
   - 依赖"创建 Fulfillment 时所有 Inventory Items 一起履约"的假设
   - 假设各 Fulfillment Items 的数量保持固定比例
   - 因此可以使用任意一个 Fulfillment Item 进行反推

3. **部分取消的风险**：
   - 如果部分取消不能保持各 Fulfillment Items 的数量比例
   - 反推逻辑可能使用"错误"的 Fulfillment Item 计算
   - 导致 Shipment Item Quantity 不正确

这个问题展示了在处理多对多关系（Order Line Item ↔ Inventory Items）时，数量换算的隐藏复杂度，以及业务逻辑变更（如支持部分取消）可能对现有假设造成的冲击。
