# 渠道矩阵与点餐依赖图

## 参数矩阵（最容易错的地方）

| 渠道 | `beType` | `orderType` | `beCode` | 门店来源 |
|---|---|---|---|---|
| 到店自取 | 1 | 1 | **不传**（传了可能报错） | `query-nearby-stores(beType=1)` |
| 得来速车道 | 5 | 1 | **必传** | `query-nearby-stores(beType=5)` |
| 麦乐送 | 2 | 2 | **必传** | `delivery-query-stores(beType=2)` |
| 企业团餐 | 6 | 2 | **必传** | `delivery-query-stores(beType=6)` |

`beType=1` 的门店列表**不含** `beCode`；`beType=5` 的含。这是判断渠道有没有走错的最快信号。

## 依赖图

**到店自取**
```
query-nearby-stores(beType=1, searchType=1|2)
  → storeCode
  → query-meals(storeCode, orderType=1, beType=1)
  → productCode
  → query-meal-detail(storeCode, orderType=1, beType=1, code)   # 套餐组成 / 可特调项
  → calculate-price(storeCode, orderType=1, beType=1, items[])
  → data.takeWayList[].code                                    # 取餐方式，create-order 必传
  → create-order(takeWayCode, ...)
```

**得来速**：同上，但所有调用追加 `beCode`，`beType=5`。

**麦乐送 / 团餐**
```
delivery-query-addresses → addressId
  → delivery-query-stores(addressId, beType=2|6) → storeCode + beCode
  → query-meals(storeCode, orderType=2, beType, beCode)
  → calculate-price(storeCode, orderType=2, beType, beCode, items[])
  → create-order(addressId, beCode, ...)      # 不需要 takeWayCode
```

团餐额外两步：`query-meal-assistance` 拿 `gmServiceCode`（传给 calculate-price / create-order），`query-promotions` 看满减满折原始规则（`promotionType` 31=满额减, 33=订单折扣；`products.type` 1=包含, 2=不包含, 3=全部）。促销规则自己解析，但**是否真生效交给 `calculate-price` 验证**。

**下单前必查**：`calculate-price` 返回的 `takeWayList[].code`（`orderType=1` 时 `create-order` 必传）。跳过这步会直接失败。

## 单位与显示

- `calculate-price` 等接口金额单位是**分**，展示必须 ÷100
- 营养：`list-nutrition-foods` 返回 160 行，`energyKj` / `energyKcal` 并存，`protein` / `fat` / `carbohydrate` 单位 g，`sodium` / `calcium` 单位 mg
- 营养表按**中文名**索引，**没有 `productCode`**，与 `query-meals` 的关联只能靠名字匹配
- 营养表内有重名（`纯牛奶（盒装）`、`小杯玉米杯` 各出现两次），建索引时同名取首条并标注冲突
- 实测锚点：`脆薯饼` 146 kcal、`麦辣鸡腿汉堡` 485 kcal、`可乐中杯` 147 kcal、`无糖可口可乐中杯` 0 kcal、`芝士双层安格斯厚牛堡` 1003 kcal

## 券的生命周期

1. `available-coupons` —— 只能看，无 ID
2. `auto-bind-coupons` —— 领取，无参数，领光当前可领的
3. `query-my-coupons` —— 卡包全量，**不承诺本店本单可用**
4. `query-store-coupons` —— 本店本单可用，含 `couponId` + 适用 `productCode` + 有效期

## 价格返回里的隐藏字段

`calculate-price` 的 `data` 除 `price` / `discount` 外还有几个关键字段：

- `productList` —— 逐项小计。**为空 = 商品编码根本没解析到**，此时 `price` 可能为 0 而 `success` 仍是 `true`
- `takeWayList` —— `orderType=1` 下单必传的取餐方式。**code 按门店变，不要硬编码**，永远从返回值里取。已实测到四种：`eat-in`（堂食）、`locker-out`（店外取餐柜）、`take-in-store`（店内自提）、`take-dt-quick`（得来速车道自提）
- `deliveryPrice` / `packingPrice` —— 只有 `orderType=2` 才有值，到店类订单这两个字段直接不出现
- 打包费在 `productList` 里伪装成一行商品：`打包服务费`（code `920174`），`subtotal` 为 0 但计进了 `packingPrice` —— 不要重复相加。那个「数量」是**打包件数，不是商品数**，也没法从商品数推算：2 个鸡排 = 16 件 / 1.60 元，汉堡+薯条 = 19 件 / 1.90 元。直接读 `packingPrice`，不要自己估
- `enjoyed` / `enjoyable` —— 已享受 / 可享受的优惠。`amountType` 1=特价 2=折扣 3=立减；`balance` 是**门槛差额**（分），为正表示「再花这么多就能拿到该优惠」，是凑单判断的唯一依据
- `mealAssistanceList` —— 团餐助餐服务，取 `code`

**价格可信性判定**：`productList` 非空 且 `price > 0`。不满足就不要把数字报给用户。

## 商品编码按渠道分

**不要把自取的 `productCode` 拿去算外送。** 外送是另一套编码，且 `calculate-price` 不会报错——它静默丢掉不认识的商品。

实测（门店 1421094）：`1440`→`901440`（麦辣鸡腿汉堡）、`1100`→`901100`、`1406`→`901406`、`4810`→`501205`（中薯条→薯条），但 `521156` 两边同码。

规律不存在（前缀 `90`、完全换号、同码三种都有），必须**按名称逐条对齐**：用该渠道的 `query-meals(storeCode, orderType, beType, beCode)` 重新取码。

## 铁律

- `create-order` 不幂等：超时**不得**重试，先 `order-list` 查是否落库
- agent 无法付款：`create-order` 只返回支付链接，终点是交给人类
- 限流 600 次/分钟；一次优惠分析控制在 15 次 `calculate-price` 以内
- `cancel-order` 的 `cancelReasonCode`：1 改主意 / 2 重复下单 / 3 点错 / 4 地址电话错 / 5 送达时间错 / -1 其它
