## 一、整体推荐目录结构（标准 + 实战）

```text
my-go-project/
├── cmd/                    # 所有可执行程序的入口
│   ├── api/
│   │   └── main.go
│   ├── job/
│   │   └── main.go
│   └── worker/
│       └── main.go
│
├── internal/               # 仅本项目使用的核心业务代码（强烈推荐）
│   ├── app/
│   │   ├── api/            # api 相关组装
│   │   ├── job/
│   │   └── worker/
│   │
│   ├── domain/             # 领域模型（可选，DDD）
│   │   ├── user/
│   │   └── order/
│   │
│   ├── service/            # 业务逻辑
│   ├── repository/         # 数据访问层（DB / Redis / MQ）
│   ├── handler/            # http / grpc handler
│   ├── task/               # 定时任务 / job 逻辑
│   └── bootstrap/          # 启动初始化（config、logger、db）
│
├── pkg/                    # 可被其他项目复用的公共库
│   ├── logger/
│   ├── config/
│   ├── middleware/
│   └── utils/
│
├── configs/                # 配置文件
│   ├── config.yaml
│   ├── config.dev.yaml
│   └── config.prod.yaml
│
├── scripts/                # 构建 / 部署 / 运维脚本
│   ├── build.sh
│   └── migrate.sh
│
├── migrations/             # 数据库迁移
│
├── docs/                   # 项目文档
│
├── go.mod
├── go.sum
└── Makefile
```

> 👉 **核心思想一句话总结**
>
> * `cmd`：**怎么跑**
> * `internal`：**业务长什么样**
> * `pkg`：**别人能不能用**
> * `configs`：**环境怎么配**

---

## 二、为什么 `cmd` 是多个入口的正确打开方式

你的 `api / job / worker` 放在 **cmd** 下面是 **完全正确** 的。

### 示例：`cmd/api/main.go`

```go
func main() {
    app := bootstrap.InitApp()
    server := app.HTTPServer()
    server.Run()
}
```

### 好处：

* 每个入口**独立编译**
* 一个 repo 支撑 **多进程部署**
* 符合 Go 官方推荐（k8s、etcd 都这么干）

---

## 三、`internal` vs `pkg` —— 很多人搞反的点

### ✅ internal（强烈推荐作为业务主战场）

**规则**：

> `internal` 下的代码 **只能被当前项目 import**

适合放：

* 业务逻辑
* handler / service / repository
* 和业务强相关的模型

```text
internal/
├── handler/    # HTTP / GRPC
├── service/    # 核心业务
├── repository/ # DB / Cache / MQ
```

### ✅ pkg（真的能被“复制粘贴走”的）

适合放：

* logger
* config 加载
* 通用中间件
* 通用工具函数

```go
import "my-go-project/pkg/logger"
```

❌ **反例**：

> 把 userService / orderRepo 放 pkg —— 以后一定后悔

---

## 四、config 目录怎么设计才不乱

推荐：

```text
configs/
├── config.yaml        # 通用
├── config.dev.yaml    # 开发
├── config.test.yaml
└── config.prod.yaml
```

配合 `pkg/config`：

```go
type Config struct {
    App  AppConfig
    DB   DBConfig
    Redis RedisConfig
}
```

启动时通过环境变量决定：

```bash
ENV=prod ./api
```

---

## 五、api / job / worker 如何复用代码？

关键点在这里 👇

```text
cmd/
  api/main.go
  job/main.go
  worker/main.go

internal/
  service/
  repository/
  task/
```

* **api**：调用 `service`
* **job**：调用 `task`
* **worker**：消费 MQ → 调 `service`

👉 **main.go 永远只做一件事：组装**

---

## 六、常见但不推荐的坑

### ❌ 没有 `internal`，全丢 pkg

> 项目一大，全是“公共库”，谁也不敢删

### ❌ 一个 main.go 启多个模式

```bash
./app --mode=api
```

> 违反 Go 的“一进程一职责”

### ❌ handler 里写业务

> 后期测试、复用、worker 调用都很痛苦

---

## 七、一个更“工程化”的简化版（如果你不想太重）

```text
├── cmd/
├── internal/
│   ├── handler
│   ├── service
│   └── repo
├── pkg/
├── configs/
└── go.mod
```

**够用 80% 的项目**。

---

我直接给你一套 **「开发时的决策指南 + 心智模型」**，让你在写代码时能快速回答：

> **“这段代码，我到底该放哪？”**

---

## 一、先给你一句“工程真言”（非常重要）

> **目录结构不是为了好看，是为了：
> 👉 让你在 3 个月后，不用重新理解自己**

你现在难以决策，通常不是不会分层，而是**每一层的“职责边界”不够清晰**。

下面我们把边界切死。

---

## 二、核心决策表（收藏级）

你只要对着这张表问自己👇

| 问题                      | 答案 | 放哪           |
| ----------------------- | -- | ------------ |
| 这是 HTTP / GRPC 参数解析吗？   | 是  | `handler`    |
| 这是具体业务规则吗？              | 是  | `service`    |
| 这是 DB / Redis / MQ 操作吗？ | 是  | `repository` |
| 这是定时 / 异步任务逻辑吗？         | 是  | `task`       |
| 这是程序启动时做的事吗？            | 是  | `bootstrap`  |
| 这是所有项目都能用的工具吗？          | 是  | `pkg`        |
| 这段代码离开这个项目就没意义？         | 是  | `internal`   |

**90% 的犹豫都能被这张表直接消灭。**

---

## 三、用一个真实场景，走完整条调用链

我们假设一个需求：

> “API 创建订单 → 写数据库 → 发 MQ → worker 消费 → 更新状态”

### 1️⃣ handler（只干三件事）

```go
// internal/handler/order.go
func (h *OrderHandler) Create(c *gin.Context) {
    req := new(CreateOrderRequest)
    _ = c.ShouldBindJSON(req)

    resp, err := h.orderService.Create(c.Request.Context(), req)
    if err != nil {
        c.JSON(500, err)
        return
    }
    c.JSON(200, resp)
}
```

✅ **不允许出现**：

* SQL
* Redis
* if…else 业务判断

---

### 2️⃣ service（业务“裁判”）

```go
// internal/service/order.go
func (s *OrderService) Create(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    if req.Amount <= 0 {
        return nil, errors.New("invalid amount")
    }

    order := NewOrder(req)

    err := s.repo.Save(ctx, order)
    if err != nil {
        return nil, err
    }

    s.mq.PublishOrderCreated(order)

    return order, nil
}
```

**你犹豫的时候问自己一句：**

> ❓“这段 if 判断算不算业务规则？”

只要**不是“技术限制”而是“业务约束”** → service

---

### 3️⃣ repository（纯技术，无情感）

```go
// internal/repository/order_repo.go
func (r *OrderRepo) Save(ctx context.Context, o *Order) error {
    return r.db.Create(o).Error
}
```

**铁律：**

* ❌ 不写业务判断
* ❌ 不返回 HTTP 错误
* ✅ 只返回 error

---

### 4️⃣ task / worker（场景驱动）

```go
// internal/task/order_consumer.go
func (t *OrderTask) HandleCreated(msg *OrderCreatedMsg) error {
    return t.orderService.MarkPaid(context.Background(), msg.OrderID)
}
```

👉 **worker ≠ service**
worker 是“触发器”，service 才是“决策者”。

---

## 四、你最容易纠结的 5 个点（我直接帮你下判断）

### ❓1. 参数校验放哪？

* **格式校验（json、必填）** → handler
* **业务校验（状态是否允许）** → service

---

### ❓2. 事务放哪？

👉 **service**

```go
func (s *OrderService) Create(ctx context.Context, req *Req) error {
    return s.repo.WithTx(ctx, func(txRepo Repo) error {
        // 多 repo 协调
    })
}
```

❌ handler 开事务
❌ repo 自己控制业务事务

---

### ❓3. enum / 常量放哪？

* 业务状态：`internal/domain`
* 技术常量：`pkg`

---

### ❓4. DTO / VO / Model 太乱怎么办？

**定死规则：**

| 类型                      | 位置         |
| ----------------------- | ---------- |
| HTTP Request / Response | handler    |
| 领域对象                    | domain     |
| DB Model                | repository |

**不共用结构体，哪怕字段一样。**

---

### ❓5. service 会不会太胖？

答案：**会，但这是正常的**

解决方法不是乱拆，而是：

```text
order/
├── order_create.go
├── order_pay.go
└── order_cancel.go
```

---

## 五、真正让你“决策轻松”的隐藏技巧

### 🧠 技巧 1：写代码时默念一句话

> **“我这段代码，知道 HTTP 吗？”**

* 知道 → handler
* 不知道 → service / repo

---

### 🧠 技巧 2：任何目录，只允许“向下依赖”

```text
handler → service → repository
```

一旦你发现：

> service import handler

说明你**已经走歪了**。

---

## 六、给你一个“最低摩擦”的目录模板

```text
internal/
├── handler/
├── service/
├── repository/
├── task/
├── domain/
└── bootstrap/
```

**先只用这 6 个目录，别急着搞 DDD 花活。**

---

# model放在哪里？如何定义

## 先给你结论（记住这 4 条就够）

> **一句话规则：
> 谁“拥有”它，它就放哪。**

### ✅ 1️⃣ GORM 的 Model

👉 **放在 `internal/repository/model`**

```text
internal/
└── repository/
    └── model/
        └── order_model.go
```

```go
type OrderModel struct {
    ID        uint64 `gorm:"primaryKey"`
    Amount    int64
    Status    string
    CreatedAt time.Time
}
```

**它的本质是：**

* 数据库表结构映射
* 为 GORM 服务
* 强烈“技术属性”

❌ **不要放 domain**
❌ **不要放 service**

---

### ✅ 2️⃣ VO（View Object / Response）

👉 **放在 `handler` 层，靠近接口**

```text
internal/
└── handler/
    └── vo/
        └── order_vo.go
```

```go
type OrderVO struct {
    ID     uint64 `json:"id"`
    Amount int64  `json:"amount"`
    Status string `json:"status"`
}
```

**VO 的本质是：**

* 给前端 / 调用方看的
* 和接口强绑定
* 经常随接口变化

👉 **handler 拥有它**

---

### ✅ 3️⃣ DTO（Request / Command）

👉 **同样放 handler（或 handler/dto）**

```text
internal/
└── handler/
    └── dto/
        └── create_order_dto.go
```

---

### ✅ 4️⃣ Domain（如果你用）

👉 **只放“业务概念”，不放 gorm tag**

```go
type Order struct {
    ID     uint64
    Amount int64
    Status OrderStatus
}
```

---

## 一张“防纠结”对照表（重点）

| 类型         | 是否含 gorm tag | 是否含 json tag | 放哪                 |
| ---------- | ------------ | ------------ | ------------------ |
| GORM Model | ✅            | ❌            | `repository/model` |
| Domain     | ❌            | ❌            | `domain`           |
| DTO        | ❌            | ✅            | `handler/dto`      |
| VO         | ❌            | ✅            | `handler/vo`       |

👉 **只要看到 `gorm:""`，第一反应就是 repository**

---

## 为什么“提前定义 model”是对的，但别让它污染全局

你说得很对：

> “我经常使用 gorm 需要提前定义 model”

**问题不在“提前”，在“到处用”。**

### ❌ 常见翻车姿势

```go
// handler 直接用 gorm model
var order model.OrderModel
db.First(&order)
c.JSON(200, order)
```

结果：

* 多返回字段
* 泄露数据库结构
* 一改表结构，全链路炸

---

## 正确姿势：显式转换（哪怕多写几行）

```go
// repository
func (r *OrderRepo) Get(ctx context.Context, id uint64) (*model.OrderModel, error)

// service
func (s *OrderService) Get(ctx context.Context, id uint64) (*domain.Order, error)

// handler
func (h *OrderHandler) Get(c *gin.Context) {
    vo := vo.FromDomain(order)
}
```

**写起来烦，但半年后你会感谢自己。**

---

## 你最纠结的那个点，我直接替你拍板

### ❓“model / vo 能不能共用一个 struct？”

👉 **强烈不建议**

除非：

* 项目很小
* 明确是一次性工具
* 不考虑演进

否则：

> **省 10 行 struct，未来多 100 行修复代码**

---

## 给你一个「GORM 项目稳定结构」

```text
internal/
├── handler/
│   ├── dto/
│   └── vo/
├── service/
├── repository/
│   ├── model/
│   └── order_repo.go
└── domain/
```

**够你稳稳写到中大型项目。**

---

# 用你真实的一个 gorm model 帮你拆 model → domain → vo

好，这里我直接**给你一个“真实到不能再真实”的 GORM Model**，然后**一步一步拆成
Model → Domain → VO**，你照着这个模式抄就行，基本不会踩坑。

我会在每一步都告诉你：**为什么要拆、什么时候可以不拆**。

---

## 一、先给你一个「现实世界里的 GORM Model」

> 场景：订单表（带数据库细节、技术字段）

```go
// internal/repository/model/order_model.go
package model

import "time"

type OrderModel struct {
    ID        uint64    `gorm:"primaryKey;column:id"`
    UserID    uint64    `gorm:"column:user_id;index"`
    Amount    int64     `gorm:"column:amount"`
    Status    int8      `gorm:"column:status"`
    DeletedAt *time.Time `gorm:"column:deleted_at"`
    CreatedAt time.Time `gorm:"column:created_at"`
    UpdatedAt time.Time `gorm:"column:updated_at"`
}
```

👉 **这就是标准 GORM Model：**

* 有 `gorm tag`
* 有 `DeletedAt`
* 字段为数据库服务
* 名字就是表的映射

---

## 二、拆成 Domain（把“业务”拎出来）

> 问自己一句话：
> **“如果哪天我不用 GORM、不用 MySQL，这个对象还成立吗？”**

成立 → Domain

```go
// internal/domain/order.go
package domain

type OrderStatus int8

const (
    OrderStatusPending OrderStatus = 1
    OrderStatusPaid    OrderStatus = 2
    OrderStatusCancel  OrderStatus = 3
)

type Order struct {
    ID     uint64
    UserID uint64
    Amount int64
    Status OrderStatus
}
```

### Domain 特点（记牢）

* ❌ **没有 gorm tag**
* ❌ **没有 json tag**
* ❌ **没有 CreatedAt / DeletedAt**
* ✅ 只描述业务含义

👉 Domain 是 **service 的语言**，不是数据库的语言。

---

## 三、从 Model → Domain（Repository 内完成）

```go
// internal/repository/order_repo.go
func (r *OrderRepo) Get(ctx context.Context, id uint64) (*domain.Order, error) {
    var m model.OrderModel
    if err := r.db.WithContext(ctx).First(&m, id).Error; err != nil {
        return nil, err
    }
    return toDomain(&m), nil
}

func toDomain(m *model.OrderModel) *domain.Order {
    return &domain.Order{
        ID:     m.ID,
        UserID: m.UserID,
        Amount: m.Amount,
        Status: domain.OrderStatus(m.Status),
    }
}
```

👉 **关键点**：

* repository 对外**只返回 domain**
* gorm model 被牢牢关在 repo 里

---

## 四、VO（给前端 / 调用方的样子）

> 问自己一句：
> **“前端真的关心数据库状态码吗？”**

通常不。

```go
// internal/handler/vo/order_vo.go
package vo

type OrderVO struct {
    ID     uint64 `json:"id"`
    Amount int64  `json:"amount"`
    Status string `json:"status"`
}
```

---

## 五、Domain → VO（Handler 做）

```go
// internal/handler/vo/order_convert.go
func FromDomain(o *domain.Order) *OrderVO {
    return &OrderVO{
        ID:     o.ID,
        Amount: o.Amount,
        Status: statusToText(o.Status),
    }
}

func statusToText(s domain.OrderStatus) string {
    switch s {
    case domain.OrderStatusPending:
        return "pending"
    case domain.OrderStatusPaid:
        return "paid"
    case domain.OrderStatusCancel:
        return "cancel"
    default:
        return "unknown"
    }
}
```

👉 **为什么不在 service 转？**
因为这是**“展示逻辑”**，不是业务逻辑。

---

## 六、完整调用链（你脑子里要有这张图）

```text
DB
 ↓
OrderModel (gorm)
 ↓ repository
Order (domain)
 ↓ service
Order (domain)
 ↓ handler
OrderVO (json)
 ↓
Client
```

**任何一层只认识自己该认识的东西。**

---

## 七、什么时候你可以“偷懒不拆”？我给你明确边界

### ✅ 可以不拆（大胆用同一个 struct）

* 内部脚本
* 临时管理后台
* 一次性工具
* 项目 < 2 万行、生命周期明确

### ❌ 绝对要拆

* 对外 API
* 有 worker / job
* 有多端消费
* 状态/字段一定会变

---

## 八、你现在最容易犯的“隐性坑”（提前避雷）

### ❌ 在 domain 里放 CreatedAt

> “这不是业务，是审计字段”

### ❌ VO 直接复用 Domain

> JSON 字段一改，业务代码全被牵连

### ❌ repo 返回 model

> service 开始 import gorm（灾难开始）

---

## 九、给你一个「低摩擦版」偷懒方案（实战推荐）

如果你觉得全拆太重：

```text
repository/model  →  domain  →  vo
        |               |
        +----（小项目时可共用）----+
```

**但只允许 domain 和 vo 偶尔共用，
gorm model 永远独立。**

---


