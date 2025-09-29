# Spring Boot + MyBatis-Plus 实现订单有限状态机（策略模式）
> 版本：2025-09-29  
> 目标：基于 **Spring Boot + MyBatis-Plus**，用**策略/状态模式（State/Strategy）**实现**订单有限状态机（FSM）**，并结合 **幂等（Idempotency）**、**CAS 条件更新**、**Outbox/Inbox**、**库存预占**，给出**可直接落地**的代码骨架（无 XML 配置）。

---


## 目录
- [1. 数据库 DDL（MySQL 8+）](#1-数据库-ddlmysql-8)
- [2. 工程依赖与基础配置](#2-工程依赖与基础配置)
- [3. 实体与 Mapper（MyBatis-Plus）](#3-实体与-mappermybatis-plus)
- [4. 策略/状态模式 FSM 实现（无 XML）](#4-策略状态模式-fsm-实现无-xml)
- [5. 幂等组件（创建类接口）](#5-幂等组件创建类接口)
- [6. Outbox 投递与 Inbox 去重](#6-outbox-投递与-inbox-去重)
- [7. 库存预占与扣减](#7-库存预占与扣减)
- [8. 控制器示例（创建订单 & 支付回调）](#8-控制器示例创建订单--支付回调)
- [9. 端到端请求流](#9-端到端请求流)

---

## 1. 数据库 DDL（MySQL 8+）

```sql
-- 订单（状态：10待支付 20已支付 30已发货 40已签收 50已完成 60已取消）
CREATE TABLE `orders` (
  id BIGINT UNSIGNED PRIMARY KEY,
  order_no VARCHAR(40) NOT NULL UNIQUE,
  buyer_id BIGINT UNSIGNED NOT NULL,
  status TINYINT NOT NULL,
  amount_payable DECIMAL(20,6) NOT NULL,
  amount_paid DECIMAL(20,6) NOT NULL DEFAULT 0,
  paid_at DATETIME(3) NULL,
  shipped_at DATETIME(3) NULL,
  delivered_at DATETIME(3) NULL,
  completed_at DATETIME(3) NULL,
  canceled_at DATETIME(3) NULL,
  receiver_snapshot JSON NOT NULL,
  created_at DATETIME(3) NOT NULL,
  updated_at DATETIME(3) NOT NULL,
  KEY (buyer_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 幂等（创建类接口保存请求哈希与响应快照）
CREATE TABLE idempotency (
  id BIGINT UNSIGNED PRIMARY KEY,
  idem_key VARCHAR(80) NOT NULL,
  request_hash CHAR(64) NOT NULL,
  status TINYINT NOT NULL,            -- 0=processing 1=done 2=failed
  response_json JSON NULL,
  expires_at DATETIME(3) NULL,
  created_at DATETIME(3) NOT NULL,
  updated_at DATETIME(3) NOT NULL,
  UNIQUE KEY uniq_idem_key (idem_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Outbox（同事务写入，异步发布到 MQ）
CREATE TABLE outbox_event (
  id BIGINT UNSIGNED PRIMARY KEY,
  aggregate_type VARCHAR(32) NOT NULL,   -- ORDER/PAYMENT/...
  aggregate_id BIGINT UNSIGNED NOT NULL,
  event_type VARCHAR(64) NOT NULL,       -- ORDER_PAID/ORDER_SHIPPED...
  payload JSON NOT NULL,
  created_at DATETIME(3) NOT NULL,
  published TINYINT NOT NULL DEFAULT 0,  -- 0 未投递 / 1 已投递
  KEY idx_pub(published, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Inbox（回调/消息去重）
CREATE TABLE inbox_log (
  msg_id VARCHAR(64) PRIMARY KEY,
  source VARCHAR(32) NOT NULL,
  processed_at DATETIME(3) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 支付单（out_trade_no 唯一幂等）
CREATE TABLE payment (
  id BIGINT UNSIGNED PRIMARY KEY,
  order_id BIGINT UNSIGNED NOT NULL,
  channel TINYINT NOT NULL,
  out_trade_no VARCHAR(64) NOT NULL UNIQUE,
  third_trade_no VARCHAR(64) UNIQUE,
  status TINYINT NOT NULL,               -- 0待支付 1已支付 2失败
  amount DECIMAL(20,6) NOT NULL,
  created_at DATETIME(3) NOT NULL,
  paid_at DATETIME(3) NULL,
  updated_at DATETIME(3) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 库存与预占
CREATE TABLE stock (
  sku_id BIGINT UNSIGNED NOT NULL,
  warehouse_id BIGINT UNSIGNED NOT NULL,
  qty INT NOT NULL,
  reserved INT NOT NULL DEFAULT 0,
  updated_at DATETIME(3) NOT NULL,
  PRIMARY KEY (sku_id, warehouse_id)
) ENGINE=InnoDB;

CREATE TABLE stock_reservation (
  id BIGINT UNSIGNED PRIMARY KEY,
  order_no VARCHAR(40) NOT NULL,
  sku_id BIGINT UNSIGNED NOT NULL,
  warehouse_id BIGINT UNSIGNED NOT NULL,
  qty INT NOT NULL,
  expire_at DATETIME(3) NOT NULL,
  created_at DATETIME(3) NOT NULL,
  UNIQUE KEY uniq_order_sku_wh (order_no, sku_id, warehouse_id)
) ENGINE=InnoDB;
```

---

## 2. 工程依赖与基础配置

**Maven（摘录）**
```xml
<properties>
  <java.version>17</java.version>
  <mybatis-plus.version>3.5.5</mybatis-plus.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>${mybatis-plus.version}</version>
  </dependency>
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
  </dependency>
</dependencies>
```

**application.yml（摘录）**
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/shop?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
    username: root
    password: pass
    hikari:
      maximum-pool-size: 20
  jackson:
    time-zone: UTC
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
```

---

## 3. 实体与 Mapper（MyBatis-Plus）

> 说明：ID 使用 MP 的 **ASSIGN_ID（雪花）**；无 XML。

```java
// Orders.java
@Data @TableName("orders")
public class Orders {
  @TableId(type = IdType.ASSIGN_ID) private Long id;
  private String orderNo; private Long buyerId; private Integer status;
  private BigDecimal amountPayable; private BigDecimal amountPaid;
  private LocalDateTime paidAt, shippedAt, deliveredAt, completedAt, canceledAt;
  private String receiverSnapshot; private LocalDateTime createdAt, updatedAt;
}

// 其余实体：Payment、Idempotency、OutboxEvent、InboxLog、Stock、StockReservation 同理
```

```java
// Mapper 接口
public interface OrdersMapper extends BaseMapper<Orders> { }
public interface PaymentMapper extends BaseMapper<Payment> { }
public interface IdempotencyMapper extends BaseMapper<Idempotency> { }
public interface OutboxMapper extends BaseMapper<OutboxEvent> { }
public interface InboxMapper extends BaseMapper<InboxLog> { }
public interface StockMapper extends BaseMapper<Stock> { }
public interface StockReservationMapper extends BaseMapper<StockReservation> { }
```

---

## 4. 策略/状态模式 FSM 实现（无 XML）

```java
// OrderStatuses.java
public final class OrderStatuses {
  public static final int PENDING=10, PAID=20, SHIPPED=30, DELIVERED=40, COMPLETED=50, CANCELED=60;
  private OrderStatuses() {}
}

// OrderState.java
public interface OrderState {
  int getCode();
  boolean pay(long orderId, BigDecimal paid, String thirdTradeNo);
  boolean cancel(long orderId, String reason);
  boolean ship(long orderId, String trackingNo);
  boolean deliver(long orderId);
  boolean complete(long orderId);
}
```

**PendingState（CAS + Outbox）**
```java
@Component("orderState#10")
@RequiredArgsConstructor
public class PendingState implements OrderState {
  private final OrdersMapper orders; private final OutboxMapper outbox;

  @Override public int getCode(){ return OrderStatuses.PENDING; }

  @Override @Transactional
  public boolean pay(long orderId, BigDecimal paid, String thirdTradeNo){
    var uw = new UpdateWrapper<Orders>()
      .eq("id", orderId).eq("status", OrderStatuses.PENDING).eq("amount_payable", paid)
      .set("status", OrderStatuses.PAID)
      .set("paid_at", LocalDateTime.now())
      .set("amount_paid", paid).set("updated_at", LocalDateTime.now());
    int updated = orders.update(null, uw);
    if (updated == 1){
      OutboxEvent e = new OutboxEvent();
      e.setAggregateType("ORDER"); e.setAggregateId(orderId); e.setEventType("ORDER_PAID");
      e.setPayload("{\"paid\":\"" + paid + "\",\"tradeNo\":\"" + thirdTradeNo + "\"}");
      e.setCreatedAt(LocalDateTime.now()); e.setPublished(0);
      outbox.insert(e);
      return true;
    }
    return false; // 幂等或并发失败
  }

  @Override @Transactional
  public boolean cancel(long orderId, String reason){
    var uw = new UpdateWrapper<Orders>()
      .eq("id", orderId).eq("status", OrderStatuses.PENDING)
      .set("status", OrderStatuses.CANCELED)
      .set("canceled_at", LocalDateTime.now())
      .set("updated_at", LocalDateTime.now());
    return orders.update(null, uw) == 1;
  }

  @Override public boolean ship(long orderId, String trackingNo){ return false; }
  @Override public boolean deliver(long orderId){ return false; }
  @Override public boolean complete(long orderId){ return false; }
}
```

**PaidState（只允许发货）**
```java
@Component("orderState#20")
@RequiredArgsConstructor
public class PaidState implements OrderState {
  private final OrdersMapper orders; private final OutboxMapper outbox;

  @Override public int getCode(){ return OrderStatuses.PAID; }
  @Override public boolean pay(long id, BigDecimal p, String t){ return false; }
  @Override public boolean cancel(long id, String r){ return false; }

  @Override @Transactional
  public boolean ship(long orderId, String trackingNo){
    var uw = new UpdateWrapper<Orders>()
      .eq("id", orderId).eq("status", OrderStatuses.PAID)
      .set("status", OrderStatuses.SHIPPED)
      .set("shipped_at", LocalDateTime.now())
      .set("updated_at", LocalDateTime.now());
    int updated = orders.update(null, uw);
    if (updated == 1){
      OutboxEvent e = new OutboxEvent();
      e.setAggregateType("ORDER"); e.setAggregateId(orderId); e.setEventType("ORDER_SHIPPED");
      e.setPayload("{\"trackingNo\":\"" + trackingNo + "\"}");
      e.setCreatedAt(LocalDateTime.now()); e.setPublished(0);
      outbox.insert(e);
      return true;
    }
    return false;
  }
  @Override public boolean deliver(long id){ return false; }
  @Override public boolean complete(long id){ return false; }
}
```

**状态机路由**
```java
@Component
@RequiredArgsConstructor
public class OrderStateMachine {
  private final OrdersMapper orders;
  private final Map<String, OrderState> states;

  private OrderState resolve(int status){ return states.get("orderState#" + status); }

  public boolean pay(long orderId, BigDecimal amt, String trade){
    Integer s = orders.selectById(orderId).getStatus();
    return resolve(s).pay(orderId, amt, trade);
  }
  public boolean cancel(long orderId, String reason){
    Integer s = orders.selectById(orderId).getStatus();
    return resolve(s).cancel(orderId, reason);
  }
  public boolean ship(long orderId, String trackingNo){
    Integer s = orders.selectById(orderId).getStatus();
    return resolve(s).ship(orderId, trackingNo);
  }
  public boolean deliver(long orderId){
    Integer s = orders.selectById(orderId).getStatus();
    return resolve(s).deliver(orderId);
  }
  public boolean complete(long orderId){
    Integer s = orders.selectById(orderId).getStatus();
    return resolve(s).complete(orderId);
  }
}
```

---

## 5. 幂等组件（创建类接口）

```java
@Service @RequiredArgsConstructor
public class IdempotencyService {
  private final IdempotencyMapper mapper;

  public enum StartResult { NEW, DUP_SAME, DUP_DIFF }

  @Transactional
  public StartResult start(String key, String reqHash){
    Idempotency r = new Idempotency();
    r.setId(IdWorker.getId()); r.setIdemKey(key); r.setRequestHash(reqHash);
    r.setStatus(0); r.setCreatedAt(LocalDateTime.now()); r.setUpdatedAt(LocalDateTime.now());
    try { mapper.insert(r); return StartResult.NEW; }
    catch (DuplicateKeyException e){
      var exist = mapper.selectOne(new LambdaQueryWrapper<Idempotency>().eq(Idempotency::getIdemKey, key));
      return reqHash.equalsIgnoreCase(exist.getRequestHash()) ? StartResult.DUP_SAME : StartResult.DUP_DIFF;
    }
  }

  @Transactional
  public void finish(String key, String respJson){
    mapper.update(null, new UpdateWrapper<Idempotency>()
      .eq("idem_key", key)
      .set("status", 1).set("response_json", respJson).set("updated_at", LocalDateTime.now()));
  }
}
```

---

## 6. Outbox 投递与 Inbox 去重

```java
@Component @RequiredArgsConstructor
public class OutboxPublisher {
  private final OutboxMapper outbox;

  @Scheduled(fixedDelay = 500) @Transactional
  public void publish(){
    var list = outbox.selectList(new LambdaQueryWrapper<OutboxEvent>()
      .eq(OutboxEvent::getPublished, 0).last("ORDER BY created_at ASC LIMIT 100"));
    for (var e : list){
      // TODO 发送 MQ 或 Spring 事件总线
      outbox.update(null, new UpdateWrapper<OutboxEvent>().eq("id", e.getId()).set("published", 1));
    }
  }
}

@Service @RequiredArgsConstructor
public class InboxService {
  private final InboxMapper inbox;
  @Transactional
  public boolean insertOnce(String msgId, String source){
    try {
      InboxLog row = new InboxLog();
      row.setMsgId(msgId); row.setSource(source); row.setProcessedAt(LocalDateTime.now());
      inbox.insert(row); return true;
    } catch (DuplicateKeyException e){ return false; } // 已处理
  }
}
```

---

## 7. 库存预占与扣减（混用 JdbcTemplate 实现行锁）

```java
@Service @RequiredArgsConstructor
public class InventoryService {
  private final JdbcTemplate jdbc; private final StockReservationMapper resMapper;

  @Transactional
  public boolean reserve(String orderNo, long skuId, long whId, int qty){
    jdbc.query("SELECT qty,reserved FROM stock WHERE sku_id=? AND warehouse_id=? FOR UPDATE",
      rs -> {}, skuId, whId);

    // 幂等预占（唯一：order_no, sku_id, warehouse_id）
    try{
      StockReservation r = new StockReservation();
      r.setId(IdWorker.getId()); r.setOrderNo(orderNo); r.setSkuId(skuId); r.setWarehouseId(whId);
      r.setQty(qty); r.setExpireAt(LocalDateTime.now().plusMinutes(15)); r.setCreatedAt(LocalDateTime.now());
      resMapper.insert(r);
    }catch (DuplicateKeyException e){ return true; }

    int updated = jdbc.update(
      "UPDATE stock SET reserved=reserved+? WHERE sku_id=? AND warehouse_id=? AND (qty-reserved)>=?",
      qty, skuId, whId, qty);
    if (updated == 0){
      resMapper.delete(new LambdaQueryWrapper<StockReservation>()
        .eq(StockReservation::getOrderNo, orderNo)
        .eq(StockReservation::getSkuId, skuId)
        .eq(StockReservation::getWarehouseId, whId));
      return false;
    }
    return true;
  }

  @Transactional
  public void confirmAndDeduct(String orderNo, long skuId, long whId, int qty){
    jdbc.update("UPDATE stock SET reserved=reserved-?, qty=qty-? WHERE sku_id=? AND warehouse_id=?",
      qty, qty, skuId, whId);
    resMapper.delete(new LambdaQueryWrapper<StockReservation>()
      .eq(StockReservation::getOrderNo, orderNo)
      .eq(StockReservation::getSkuId, skuId)
      .eq(StockReservation::getWarehouseId, whId));
  }
}
```

---

## 8. 控制器示例（创建订单 & 支付回调）

```java
@RestController @RequestMapping("/api/orders") @RequiredArgsConstructor
public class OrderController {
  private final IdempotencyService idem; private final CreateOrderUseCase usecase;

  @PostMapping
  public ResponseEntity<?> create(@RequestHeader("Idempotency-Key") String key,
                                  @RequestBody CreateOrderCmd cmd){
    String hash = DigestUtils.sha256Hex(cmd.toCanonicalString());
    var r = idem.start(key, hash);
    if (r == IdempotencyService.StartResult.DUP_DIFF)
      return ResponseEntity.status(409).body(Map.of("error","Same key, different request"));
    if (r == IdempotencyService.StartResult.DUP_SAME)
      return ResponseEntity.ok(Map.of("status","already processed"));
    var result = usecase.handle(cmd); // 内含库存预占/金额快照
    idem.finish(key, new ObjectMapper().valueToTree(result).toString());
    return ResponseEntity.ok(result);
  }
}

@RestController @RequestMapping("/webhook/payment") @RequiredArgsConstructor
public class PaymentWebhookController {
  private final InboxService inbox; private final OrderStateMachine fsm;

  @PostMapping("/paid") @Transactional
  public ResponseEntity<String> onPaid(@RequestBody PayNotify n){
    if (!inbox.insertOnce(n.getTransactionId(), "wechatpay")) return ResponseEntity.ok("OK");
    boolean changed = fsm.pay(n.getOrderId(), n.getAmount(), n.getTransactionId());
    return ResponseEntity.ok(changed ? "OK" : "IGNORED");
  }
}
```

---

## 9. 端到端请求流
1. **创建订单**：`Idempotency-Key` → `idem.start()` → (事务) 校验价格、**库存预占**、落库 `orders`、写 `outbox_event(ORDER_CREATED)` → `idem.finish()`。  
2. **支付回调**：`inbox.insertOnce()` 去重 → `fsm.pay()`：`PENDING→PAID`（CAS）+ `outbox_event(ORDER_PAID)`。  
3. **发货/签收/完成**：`fsm.ship/deliver/complete()`，各自 CAS 推进 + Outbox。  
4. **OutboxPublisher**：定时投递 `ORDER_*` 事件，订阅侧（通知/积分/CRM）消费。  
