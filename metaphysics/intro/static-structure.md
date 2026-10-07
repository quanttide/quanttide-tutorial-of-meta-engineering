---
description: 这篇教程面向的是只有一点离散数学基础或者实际编程经验的人；这一篇讲集合、关系、函数、逻辑这四样东西与代码里的数据模型、外键、查询函数、业务规则怎么一一对上；目的是让读者拿到一段需求时，能先把它的静态结构写成没有歧义的集合和逻辑式。
plots:
  - 业务需求
  - 哲学阐述
  - 数学表达
  - 代码实现
---

# 用集合与逻辑描述系统的静态结构

## 业务需求

一笔支付钱是从哪个账户出的，现在存的是账户名的字符串。同一个账户改个名字，历史支付就对不上了；两个账户重名，也分不清是谁付的。

这么存有两个躲不掉的问题。

一是字符串全靠约定维系。写错一个字母、改一次名、中间多一个空格，历史记录就失联，而且不会有任何地方报错。

二是规则只能散着写。生意上有两条很硬的规则：每一笔支付都必须挂在某个账户上；退过款的支付单必须是付过款的。它们现在躺在各自的接口代码里，靠一两行判断。换个人再写一个入口，很容易忘了再判一次；也没有任何地方能一眼看到「系统里到底有哪些规则」。

这一篇管静态的部分：有哪些东西、它们之间怎么关联、哪些断言永远为真。这些东西怎么随时间变化，是下一篇的事。

## 哲学阐述

这一段要把上面那堆需求放回本体论与范畴论的坐标上：系统里到底有哪些东西存在，各自属于哪一类，那两条规则说的是什么。

先列存在清单。这里有三类东西：账户、支付单，还有账户上的钱。前两类各是一个集合，集合里装着这一类东西的每一个实例；钱不是一个独立的东西，它是账户的一个属性，也就是一个从账户到数目的对应。

再把关联归位。「这笔钱是从哪个账户出的」不是某个对象的属性，而是支付单和账户两个集合之间的一层关系。按 id 查一笔支付是一次函数——查不到时它没有值，所以是个偏函数，不是普通函数。那两条硬规则不是任何字段的取值，而是关于世界的断言，也就是逻辑式。

归完位，这一篇的边界也就清楚了：它管静态的部分——有哪些东西、它们之间怎么关联、哪些断言永远为真。这些东西怎么随时间变化，是下一篇的事。

## 数学表达

数据域是集合。

```text
Account = {acct_a, acct_b}
Payment = {p1, p2}
```

读作：账户集合里眼下有两个元素，支付单集合里有两个。`x ∈ Account` 读作「x 是账户集合里的一个元素」，也就是「x 是一个账户」。

对象之间的关联是关系。

```text
billed_to ⊆ Payment × Account
amount    : Payment → ℕ
```

第一行里 `×` 是笛卡尔积，`Payment × Account` 读作「所有（支付单, 账户）的配对」，`billed_to` 是这些配对里的一部分，配进去的每一对表示「这笔钱是从这个账户出的」。数据库里的外键就是它。第二行的 `amount` 读作：给一笔支付单，得到一个数目，也就是这笔的金额。

关系的性质决定它能派什么用场。如果它自反、对称、又传递，就可以当等价关系用（分组、去重、规范化都靠它说清楚）；如果它自反、传递、反对称，就可以当偏序关系用（排序、依赖、上下层关系靠它说清楚）。这些性质不用背，用到时回头查一次即可。

按 id 查支付单是函数，而且是偏函数。

```text
getPaymentById : PaymentId ⇸ Payment
```

用 `⇸` 不用 `→`，因为查不到时它没有值。`⇸` 读作「最多一个」，`→` 读作「恰好一个」；查不到就是「零个」。这两个符号的差别，就是代码里「返回可能为空」和「保证返回」的差别。

需求是逻辑式。

```text
dom(billed_to) = Payment
∀ p ∈ Payment · refunded(p) ⇒ once_paid(p)
```

第一条读作：凡是支付单，都在 `billed_to` 的定义域里——也就是每一笔支付都挂在某个账户上，没有例外。`dom` 读作「有定义的那些」，`=` 两边的集合相等。第二条读作：对每一笔支付单 p，如果它退过款，那它必须付过款。`∀` 读作「对每一个」，`⇒` 读作「如果左边成立，那么右边也必须成立」。

要求「至少存在一个」时把 `∀` 换成 `∃`，读作「存在一个」：

```text
∃ p ∈ Payment · amount(p) > 100000
```

读作：存在一笔支付单，金额大于十万。

工程里的说法和这里一一对应，换算出问题的时候对着这张单子看：类型和数据模型对应集合；外键和关联对应关系；查询与计算对应函数；业务规则对应逻辑式。

## 代码实现

同一份数据模型和规则，五种语言各写一遍。读一种就够，其余对着上一节看差异。

### Python

```python
from dataclasses import dataclass
from enum import Enum

class PaymentState(Enum):
    UNPAID   = "unpaid"
    PAID     = "paid"
    REFUNDED = "refunded"

@dataclass(frozen=True)
class Account:
    id: str
    balance: int

@dataclass(frozen=True)
class Payment:
    id: str
    state: PaymentState
    billed_to: str          # 关系 billed_to ⊆ Payment × Account，落成外键
    amount: int

def can_refund(p: Payment) -> bool:   # 规则落成校验函数
    return p.state is PaymentState.PAID
```

### Rust

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum PaymentState { Unpaid, Paid, Refunded }

struct Account { id: String, balance: i64 }

struct Payment {
    id: String,
    state: PaymentState,
    billed_to: String,   // 外键
    amount: i64,
}

fn can_refund(p: &Payment) -> bool {
    p.state == PaymentState::Paid
}
```

### Go

```go
type PaymentState string

const (
	Unpaid   PaymentState = "unpaid"
	Paid     PaymentState = "paid"
	Refunded PaymentState = "refunded"
)

type Account struct {
	ID      string
	Balance int64
}

type Payment struct {
	ID       string
	State    PaymentState
	BilledTo string // 外键
	Amount   int64
}

func CanRefund(p Payment) bool {
	return p.State == Paid
}
```

### Dart

```dart
enum PaymentState { unpaid, paid, refunded }

class Account {
  final String id;
  final int balance;
  const Account(this.id, this.balance);
}

class Payment {
  final String id;
  final PaymentState state;
  final String billedTo; // 外键
  final int amount;
  const Payment(this.id, this.state, this.billedTo, this.amount);
}

bool canRefund(Payment p) => p.state == PaymentState.paid;
```

### TypeScript

```typescript
type PaymentState = "unpaid" | "paid" | "refunded";

type Account = { id: string; balance: number };

type Payment = {
  id: string;
  state: PaymentState;
  billedTo: string;   // 外键
  amount: number;
};

export function canRefund(p: Payment): boolean {
  return p.state === "paid";
}
```

状态从字符串换成一个取值范围被限死的类型（枚举、常量、联合类型），就是集合这件事在代码里的落点：`Paid`、`paid`、`PAID` 三种拼法进不来，「没付过款」和「拼错了」不再靠字符串猜。`billedTo` 是关系 `billed_to` 的落点，`canRefund` 是那条规则（的代码版本）的落点。

这里要留一句：外键在代码里常常只是一个字符串，真正拦住「挂到不存在的账户上」的是数据库上的外键约束，代码这边只负责别写错。规约里那条规则有人守——守它的可能是类型、是约束、是校验函数，也可能是一次属性测试，哪一样都行。

## 本单元练习

设：

```text
Accounts = {acct_a, acct_b}
Payments = {p1, p2}
billed_to : Payments → Accounts
amount    : Payments → ℕ
```

1. 写出 `billed_to` 的数学类型，读一遍它的意思。
2. 用一阶逻辑表达：「每一笔支付单都必须挂在一个账户上。」
3. 用集合写：「没有任何一笔支付单同时挂在两个账户上。」
4. 把下面这句话翻译成公式：退过款的支付单，金额必须等于原支付金额。
5. 定义一个状态 `State = Payments × Accounts`，再写一条不变量：处于未付状态的支付单，金额不超过账户余额。

练习不附答案。写完对着这四条自检：

1. 数据域是用集合写的，不是用自然语言描述的
2. 对象之间的关联写成了关系，不是靠字段名暗示
3. 查询和计算写成了函数，不是伪代码
4. 需求落成了带 `∀`、`∃`、`⇒` 的句子，别人能照着检查

下一篇：[用状态机描述系统的动态变化](./state-machine.md)
