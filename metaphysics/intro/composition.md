---
description: 这篇教程面向的是只有一点离散数学基础或者实际编程经验的人；这一篇讲怎么把几个各自独立的状态维度组合成一个系统，以及组合之后为什么只需做局部论证；目的是让读者在状态变多时知道什么时候该拆、拆完怎么证明。
plots:
  - 业务需求
  - 哲学阐述
  - 数学表达
  - 代码实现
---

# 把多个状态组合成一个系统

## 业务需求

上一篇把订单收成一个状态字段，三个取值。现在把订单里真正独立的三件事摊开：支付有自己的路（未付、已付、已退），履约有自己的路（未发、已发），订单本身还有一条（进行中、已取消）。

这三条路各自往前走，而且各自会长新状态：支付这边迟早要加分期、部分退款；履约这边迟早要加分批发货、签收、退货。压成一个字段的代价就在这里——合法的组合眼下是 7 种，就要列 7 个变体；再加一个维度（比如「审核」三个取值），就得在 7 × 3 里挑，成乘法长。

拆成三个维度就回到加法：支付三个状态、履约两个、生命周期两个，哪个维度加状态都不影响别的维度。

代价也得先说清楚。拆开之后，`{支付: 未付, 履约: 已发}` 这种组合又能被写出来了——上一篇靠「枚举里没有这个值」让「货发了钱没到」消失，现在得靠一条跨维度的规矩挡住。组合不是白拿的，它把「状态数量」的账换成了「约束」的账。

## 哲学阐述

上一篇把订单当成一类东西，状态压在一个字段里。这一篇要改的是存在清单本身：订单不是一类东西，是三类东西拼起来的——支付、履约、生命周期各是一类，各有自己的状态和取值。

归位之后，规矩也跟着分成两种。一种落在某个维度内部，管的是那一个维度的取值；另一种横跨两个维度——「已发 ⇒ 已付」既不是在说履约维度的取值，也不是在说支付维度的取值，而是在说这两个维度之间的关系。组合特有的规矩是后一种，也是后面要单独照看的那批。

「几类东西拼成一类」这个动作，范畴论里叫积（product）：拼出来的对象仍然允许只读某一边（投影），所以各自已经证过的东西不用重证。这一篇用到的就是这一个性质。

## 数学表达

三个维度各是一个小状态机：

```text
Payment     ::= value : {unpaid, paid, refunded}
Fulfillment ::= value : {not_shipped, shipped}
Lifecycle   ::= value : {active, cancelled}
```

读作：支付模块的状态取三个值之一，履约模块取两个之一，生命周期模块取两个之一。每个模块只管自己那一行，不管别人。

系统状态是它们的直积：

```text
State = Payment × Fulfillment × Lifecycle
```

`×` 是笛卡尔积，读作「各取一个拼在一起」。三块乘起来 3 × 2 × 2 = 12 种组合，全都落在状态空间里——这一层不做业务判断，判断写在下面的不变量里。

不变量有两条，都是跨维度的：

```text
inv ≡ (fulfillment = shipped ⇒ payment = paid)
    ∧ (lifecycle = cancelled ⇒ fulfillment = not_shipped)
```

读作：履约维度到了已发，支付维度就必须是已付；生命周期维度到了已取消，履约维度就不能是已发。这两条把 12 种组合砍到 7 种。

操作按碰了几个维度分类。只碰一个维度的（本地操作）只影响自己那一行：

```text
Pay:
  pre:  payment = unpaid
  post: payment' = paid
```

下面这个是跨维度的，要同时读两个维度、写两个维度：

```text
Cancel:
  pre:  fulfillment = not_shipped                              // 读履约
  post: lifecycle'   = cancelled                               // 写生命周期
        payment'     = if payment = paid then refunded else payment   // 读支付，可能写支付
        fulfillment' = fulfillment                             // 履约不动
```

证明只做局部的一两步。`Cancel` 做完之后要复核的，只有跟它碰过的维度有关的规矩，一条一条过：

「取消的订单不能发过货」——pre 里要求 `fulfillment = not_shipped`，而这个操作没动履约，做完仍然是 `not_shipped`，成立。
「发了货必须收过钱」——这个操作没动履约，履约仍是 `not_shipped`，这条规矩的前提不成立，不必管。
支付维度自己没有额外约束，退款那一步不需要再证。

`Ship` 反过来只碰履约，pre 里那两句 `payment = paid` 和 `lifecycle = active` 就是复核用的前提，一步就完。

组合的价值就在这儿：不必把三个维度摊成一个大机重新做归纳。每个模块的规矩各自证过之后永久复用，跨维度操作只补它自己那一步。要是压成一个 7 变体的枚举，任何一次改动都得在 7 个变体上重走一遍。

有一条纪律必须写进规约，否则上面整套作废：**操作要显式声明它读了哪些维度、写了哪些维度**。`Cancel` 的 pre 里那句 `fulfillment = not_shipped` 就是一条跨维度依赖；它一旦被漏掉，局部论证就不知道该去复核哪条规矩，证明看着对，其实没盖住。

组合还能和细化叠起来：抽象层每个模块各自细化到具体层，只要跨维度操作不越界，整体细化自动成立，不用把两层摊开重证。B 方法这类带证明的工具就是这么用的——每个模块各自证完，靠精化和组合拼起来。

反过来，两个信号要认得。如果每个操作都得同时动三个维度，那不是组合，是把一个状态机拆成了三个变量，该合回去；如果跨维度的规矩一条条长起来、比维度内部的还多，说明维度之间耦合太紧，也该考虑合回一个枚举。拆的判据就在业务意图那一节：维度各自独立往前演进，拆了才划算。

还有一处最易翻车：某个维度后来加了新操作，回头把别的维度依赖的规矩破掉。跨维度的假设必须写成不变量，不能靠「那边不会那么干」的默契。

## 代码实现

三个维度落成三个类型，跨维度的两条规矩落在一个地方：造对象的地方。五种语言都是如此，差别只在那个地方叫什么。

### Python

```python
from dataclasses import dataclass
from typing import Literal

Payment     = Literal["unpaid", "paid", "refunded"]
Fulfillment = Literal["not_shipped", "shipped"]
Lifecycle   = Literal["active", "cancelled"]

@dataclass(frozen=True)
class Order:
    payment: Payment = "unpaid"
    fulfillment: Fulfillment = "not_shipped"
    lifecycle: Lifecycle = "active"

    def __post_init__(self):          # 跨维度的规矩落在这里
        if self.fulfillment == "shipped" and self.payment != "paid":
            raise ValueError("发了货必须收过钱")
        if self.lifecycle == "cancelled" and self.fulfillment == "shipped":
            raise ValueError("取消的订单不能发过货")
```

`Literal` 只有 mypy、pyright 这类检查器认，运行时不管；真正拦住的是 `__post_init__`。

### Rust

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Payment { Unpaid, Paid, Refunded }

#[derive(Debug, Clone, Copy, PartialEq)]
enum Fulfillment { NotShipped, Shipped }

#[derive(Debug, Clone, Copy, PartialEq)]
enum Lifecycle { Active, Cancelled }

#[derive(Debug, Clone, Copy)]
struct Order {
    payment: Payment,
    fulfillment: Fulfillment,
    lifecycle: Lifecycle,
}

impl Order {
    fn new(payment: Payment, fulfillment: Fulfillment, lifecycle: Lifecycle)
        -> Result<Self, &'static str>          // 跨维度的规矩落在这里
    {
        if fulfillment == Fulfillment::Shipped && payment != Payment::Paid {
            return Err("发了货必须收过钱");
        }
        if lifecycle == Lifecycle::Cancelled && fulfillment == Fulfillment::Shipped {
            return Err("取消的订单不能发过货");
        }
        Ok(Self { payment, fulfillment, lifecycle })
    }
}
```

三个枚举各自限死一个维度的取值，跨维度那两条只能落到 `new` 上：类型系统管得住单个维度的取值范围，管不住维度之间的关系。

### Go

```go
type Payment string

const (
	Unpaid   Payment = "unpaid"
	Paid     Payment = "paid"
	Refunded Payment = "refunded"
)

type Fulfillment string

const (
	NotShipped Fulfillment = "not_shipped"
	Shipped    Fulfillment = "shipped"
)

type Lifecycle string

const (
	Active    Lifecycle = "active"
	Cancelled Lifecycle = "cancelled"
)

type Order struct {
	Payment     Payment
	Fulfillment Fulfillment
	Lifecycle   Lifecycle
}

func NewOrder(p Payment, f Fulfillment, l Lifecycle) (Order, error) { // 落在这里
	if f == Shipped && p != Paid {
		return Order{}, errors.New("发了货必须收过钱")
	}
	if l == Cancelled && f == Shipped {
		return Order{}, errors.New("取消的订单不能发过货")
	}
	return Order{Payment: p, Fulfillment: f, Lifecycle: l}, nil
}
```

Go 的常量挡不住越界取值（`Payment("随便什么")` 是合法的），所以构造函数既管跨维度规矩，也管取值校验。

### Dart

```dart
enum Payment { unpaid, paid, refunded }
enum Fulfillment { notShipped, shipped }
enum Lifecycle { active, cancelled }

class Order {
  final Payment payment;
  final Fulfillment fulfillment;
  final Lifecycle lifecycle;

  Order({
    this.payment = Payment.unpaid,
    this.fulfillment = Fulfillment.notShipped,
    this.lifecycle = Lifecycle.active,
  }) {
    if (fulfillment == Fulfillment.shipped && payment != Payment.paid) {
      throw StateError('发了货必须收过钱');            // 落在这里
    }
    if (lifecycle == Lifecycle.cancelled &&
        fulfillment == Fulfillment.shipped) {
      throw StateError('取消的订单不能发过货');
    }
  }
}
```

### TypeScript

```typescript
type Payment = "unpaid" | "paid" | "refunded";
type Fulfillment = "not_shipped" | "shipped";
type Lifecycle = "active" | "cancelled";

type Order = {
  readonly payment: Payment;
  readonly fulfillment: Fulfillment;
  readonly lifecycle: Lifecycle;
};

export function makeOrder(              // 跨维度的规矩落在这里
  payment: Payment = "unpaid",
  fulfillment: Fulfillment = "not_shipped",
  lifecycle: Lifecycle = "active",
): Order {
  if (fulfillment === "shipped" && payment !== "paid") {
    throw new Error("发了货必须收过钱");
  }
  if (lifecycle === "cancelled" && fulfillment === "shipped") {
    throw new Error("取消的订单不能发过货");
  }
  return { payment, fulfillment, lifecycle };
}
```

五份代码里，跨维度那两条规矩都落在「造对象的地方」——构造函数、工厂函数、`__post_init__`。这正说明组合会把不变量带回来：类型管得住一个字段的取值范围，管不住字段之间的关系。

想让它重新被类型吃掉，只有一个办法：把 7 种合法组合列成 7 个变体（TypeScript 的联合类型、Rust 的枚举都行），让非法组合没有对应的值。代价一目了然——维度一加，组合数量又变回乘法。

## 本单元练习

订单再加一个维度：售后，取值为 `not_requested`、`processing`、`done`。规矩两条：只有已发货的订单能申请售后；售后处理中的订单不能取消。

1. 写出 `State` 的直积形式，和新的不变量（原来那两条要保留）。
2. 写出 `ApplyAfterSale` 的 pre 和 post。
3. 局部论证：这个操作做完之后要复核哪几条不变量，哪几条可以跳过、为什么。
4. 如果产品说「已取消的订单也能申请售后」，你要改规约的哪一行？改完会不会悄悄破坏别的不变量？

练习不附答案。写完对着这四条自检：

1. 状态写成了几个维度的直积，没摊成一个巨型枚举
2. 每个操作写清了读哪些维度、写哪些维度
3. 复核只做局部那几步，没有重证已经证过的维度
4. 能指出一条新规则落在哪一行，而不是改代码凭感觉

上一篇：[用状态机描述系统的动态变化](./state-machine.md)
