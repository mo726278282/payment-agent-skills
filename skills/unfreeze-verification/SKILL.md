---
name: order-verification-pipeline
description: 订单超时/冻结后的多步验证流水线模式 — 在放行资金前逐项检查系统健康状态。适用于支付、借贷、提现等需要风控验证的金融系统。
tags: [payment, verification, pipeline, timeout, hold, risk-control]
---

# Order Verification Pipeline（订单验证流水线）

当订单超时进入冻结（HOLD）状态后，需要经过多步验证才能自动解冻或取消。

## 双路径模式

```
订单超时
  ├── 首次超时路径（TIMEOUT）
  │     └── 查询: 创建时间在 N~M 分钟前的待处理订单
  └── 重试路径（HOLD_RETRY）
        └── 查询: 已有 HOLD 标记的订单，定期重试验证
```

**关键原则**: 两个路径的验证逻辑必须完全一致。只改一个会导致行为不同，这是最容易出 bug 的地方。

## 多步验证 Pipeline

```
① 采集端在线检查        — 数据源是否存活
② 数据源探测            — 主动调用数据源验证可访问性
③ 数据一致性校验        — 关键字段是否与数据源一致
④ 时间顺序校验          — 数据更新时间是否晚于订单时间
         │
         ▼
   can_finalize = ① ∧ ② ∧ ③ ∧ ④
         │
    ┌────┴────┐
  true       false
   │           │
 解冻        保持 HOLD
```

### ① 采集端在线

检查数据采集进程是否在线（通过共享状态存储如 Redis Set）：

```
is_online = state_store.is_member('collector_online_set', channel_id)
```

**异常处理策略**: 状态存储异常时**拒绝放行**而不是放行。避免采集端真实离线时错误解冻。

### ② 数据源探测（主动验证）

在解冻前主动调用数据源（如银行 API），验证采集端能正常获取数据：

```
result = probe_data_source(channel_id, channel_type)
if not result.success:
    return BLOCK  # Token 失效、网络异常等都阻止解冻
```

需要维护两个映射表：
- `channel_type_id → channel_name`（数字 ID 到名称）
- `channel_name → data_source_name`（名称到数据源的内部标识，不同渠道可能用不同爬取工具）

**新增渠道时必须同时更新这两个映射**。

### ③ 数据一致性校验

比对本地存储的关键字段与数据源实时查询结果：

```
local_value = db.query("SELECT key_field FROM orders WHERE id = ?", order_id)
live_value = fetch_from_source(channel_id, channel_type)
if local_value != live_value:
    return BLOCK  # 数据不一致，可能被篡改或采集异常
```

**适用范围**: 可按渠道粒度启用/禁用（部分渠道的数据源不支持实时查询）。

### ④ 时间顺序校验

确保数据源的最后更新时间晚于订单创建时间：

```
last_crawl_time = state_store.get(f"last_crawl:{channel_id}")
if last_crawl_time < order_create_time:
    return BLOCK  # 数据源还未拉到订单之后的交易
```

## 添加新验证步骤的 Pattern

1. **SQL 查询加字段** — 如果新检查需要额外数据
2. **HOLD 状态持久化加字段** — 确保重试时数据可用
3. **两个路径都插入检查** — TIMEOUT 和 HOLD_RETRY 都要改
4. **最终判定加条件** — `can_finalize` 中加入新检查结果
5. **检查结果记录** — 存入 `hold_reason` 结构，便于追溯
6. **每步加日志** — 用 `[TAG][STEP]` 格式，方便集中式日志搜索
7. **渠道门控** — 如果只针对特定渠道，加 `if channel_name in ('A', 'B'):` 跳过其他

## 状态记录结构

```json
{
  "collector_online": true,
  "reason_collector": "online_set",
  "data_source_probe": true,
  "reason_probe": "probe_ok",
  "data_consistency": true,
  "reason_consistency": "field_match",
  "time_order": true,
  "reason_time": "crawl_after_order"
}
```

每个检查的结果和原因都记录，方便事后从日志系统追溯。

## 集中式日志查询

验证流程日志通常分布在多个索引/主题中：
- 首次超时路径的日志
- 重试路径的日志

用统一的关键字段（订单 ID、渠道 ID）跨索引搜索 `[TAG]` 标记追踪完整链路。

## 常见坑

1. **双路径不同步** — 最常见的 bug 来源。改了一个路径忘记另一个。
2. **渠道 ID 类型不一致** — 有时是 int 有时是 string，比较前统一 normalize。
3. **新增渠道时映射表遗漏** — 两个映射表缺一个就会导致"无法识别的渠道，无法验证"。
4. **存储异常时的降级策略** — 明确是"拒绝放行"还是"放行"。金融系统通常选择拒绝。
5. **最终判定只用一次** — 所有条件合并到 `can_finalize`，不要在代码多处分别判定。
