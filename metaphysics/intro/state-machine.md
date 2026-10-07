---
description: 这篇教程面向的是只有一点离散数学基础或者实际编程经验的人；这篇教程主要是帮助读者理解什么是状态机、如何使用状态机描述系统的动态变化；目的是让读者学会在开发中学会使用状态机解决状态变化的问题。
plots:
  - 业务意图
  - 数学表达
  - 代码实现
---

# 用状态机描述系统的动态变化

## 业务意图

一笔订单从下到收，生意上真正在乎的不是它走到了哪一步，而是哪些事不许发生。下面两条规则用人话说得清，落到代码里却最容易漏：

- 没收到钱不能发货；
- 已经发过货的不能退款——货出去了再把钱退回去，等于赔两次。

这两条规则在现有系统里通常长这样：订单表上两个各自独立的布尔字段，`paid`（钱到账了吗）和 `shipped`（货出去了吗），规则写成散落在各处的 `if`：

```python
if order.paid:
    order.shipped = True
```

两个字段谁也管不住谁，于是数据库里能存下 `shipped` 为真、`paid` 为假的行——货发了，钱没到。这种行在业务上不存在，代码却拦不住，因为没有任何一行写着这个组合非法。

这套教程要做的，就是把散在 `if` 和字段里的规则收成一处能看全、能检查的东西。先把同一个意思在工程里和数学里各叫什么对齐：系统在某一刻的全部数据是一个状态；所有可能状态的集合叫状态空间；会改变状态的那件事叫操作。一个操作在什么情况下才允许做，叫前置条件 pre；做完之后状态变成什么，叫后置条件 post。还有一条不管谁怎么操作都不许被踩破的规矩，叫不变量 inv。

上面那两条规则，一条要落到 pre，一条要落到 inv。落到哪一处不是文风问题，落错了它就不再拦人。

## 数学表达

订单的状态写成两行：

```text
State ::= paid    : Bool
          shipped : Bool
```

读作：订单的状态由两个字段拼成，`paid` 取真或假，`shipped` 也取真或假。`Bool` 就是只有真、假两个值的那个集合。四种取值组合统统在状态空间里，包括生意上不许出现的那一种——状态空间不替业务做判断，判断写在下面的不变量里。

不许出现的那一种组合，写成不变量：

```text
inv ≡ shipped ⇒ paid
```

读作：只要 `shipped` 为真，`paid` 就必须为真。`⇒` 叫蕴含，`A ⇒ B` 是说「A 成立的时候 B 必须成立」；A 不成立时它不表态，所以「没发货也没付款」是合法的。

操作有三个，每个是一组 pre 加一组 post。`paid'` 上面那一撇读作「执行完之后的 `paid`」，post 里写的就是做完之后各字段的新值：

```text
Pay:
  pre:  ¬paid
  post: paid' = true

Ship:
  pre:  paid
  post: shipped' = true

Refund:
  pre:  paid ∧ ¬shipped
  post: paid' = false
```

`¬` 是否定，写在 `paid` 前面表示「没有付款」；`∧` 是「并且」。三条 pre 对着业务规则看：`Ship` 的 pre 是 `paid`，就是「没收到钱不能发货」；`Refund` 的 pre 是 `paid ∧ ¬shipped`，就是「付过钱、且货还没出去，才允许退款」；`Pay` 的 pre 是 `¬paid`，防止重复付款。

写完还有一件必须做的事：逐个操作验一遍，看它做完之后 `inv` 还在不在。

`Pay` 把 `paid` 改成真，没动 `shipped`；`shipped` 为真时 `paid` 本来就为真，现在还是真，inv 保住。`Ship` 让 `shipped` 变真的时候 `paid` 已经是真，保住。`Refund` 把 `paid` 改成假，但它的 pre 要求 `¬shipped`，做完之后 `shipped` 为假，inv 的前提不成立，等于没被踩。三个都过。

第三个操作 `Refund` 不是凑数的。只有 `Pay` 和 `Ship` 的话，pre 写对之后 inv 自然成立，读者看不出这条不变量在管什么；加上 `Refund`，一旦有人漏写一个条件，inv 立刻会破。试试把三条 pre 逐条删掉，看谁能作恶：

- 删掉 `Ship` 的 `paid`：没付款也能发货，`shipped` 为真而 `paid` 为假，inv 当场破。
- 删掉 `Refund` 的 `¬shipped`：发货之后也能退款，同样破 inv。
- 删掉 `Refund` 的 `paid`：没付过款也能「退款」，等于凭空把钱退出去。

前两条和第三条的后果不是一类。第三条产生的是「一笔不该发生的操作」，前两条产生的是「一个不该存在的状态」，而 inv 正好在管后者——所以形式化检查会直接在 `Ship` 或 `Refund` 这个操作上报错，指出这条规约允许自己造出非法状态，而不是等到某次调用才炸。这就是它比测试早一步的地方：不靠跑到边界值才发现。

两个位置分工不同，记住一句就够：pre 负责拦住这一次非法调用，inv 负责说明执行完之后世界仍然合法。把该写进 pre 的东西挪到别处，检查器不会放过你，它会在你写错的那个操作上拒绝。

## 代码实现

上面那份规约是给人和检查器看的。落到代码时，各语言能承接的东西不一样，所以下面五份的形状不一样——它们不是同一份规约的五种拼写，也不必指望逐行对应。每份末尾一句说清它把规矩放在了哪里。

### Python

```python
from dataclasses import dataclass, replace
from enum import Enum

class State(Enum):
    UNPAID  = "unpaid"
    PAID    = "paid"
    SHIPPED = "shipped"

@dataclass(frozen=True)
class Order:
    state: State = State.UNPAID

    def pay(self) -> "Order":
        if self.state is not State.UNPAID:
            raise ValueError("已经付过了")
        return replace(self, state=State.PAID)

    def ship(self) -> "Order":
        if self.state is not State.PAID:
            raise ValueError("没付款不能发货")
        return replace(self, state=State.SHIPPED)

    def refund(self) -> "Order":
        if self.state is not State.PAID:
            raise ValueError("发货之后不能退款")
        return replace(self, state=State.UNPAID)
```

枚举把取值限死在三个里，`frozen` 挡住就地改写，状态只能由这三个方法产生。规矩落在类型和构造上，运行时没有反复执行的检查。

### Rust

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Order { Unpaid, Paid, Shipped }

impl Order {
    fn pay(self) -> Result<Self, &'static str> {
        match self {
            Self::Unpaid => Ok(Self::Paid),
            _ => Err("已经付过了"),
        }
    }

    fn ship(self) -> Result<Self, &'static str> {
        match self {
            Self::Paid => Ok(Self::Shipped),
            _ => Err("没付款不能发货"),
        }
    }

    fn refund(self) -> Result<Self, &'static str> {
        match self {
            Self::Paid => Ok(Self::Unpaid),
            _ => Err("发货之后不能退款"),
        }
    }
}
```

三个变体就是三个状态，`match` 的每个分支就是「什么时候允许做」那句话。非法路径没有分支可走，编译期就挡住。

### Go

```go
type state int

const (
	unpaid state = iota
	paid
	shipped
)

type Order struct{ state state } // 字段不导出

func NewOrder(s state) (Order, error) {
	if s < unpaid || s > shipped {
		return Order{}, errors.New("未知状态")
	}
	return Order{state: s}, nil
}

func (o Order) Pay() (Order, error) {
	if o.state != unpaid {
		return o, errors.New("已经付过了")
	}
	return Order{state: paid}, nil
}

func (o Order) Ship() (Order, error) {
	if o.state != paid {
		return o, errors.New("没付款不能发货")
	}
	return Order{state: shipped}, nil
}

func (o Order) Refund() (Order, error) {
	if o.state != paid {
		return o, errors.New("发货之后不能退款")
	}
	return Order{state: unpaid}, nil
}
```

Go 没有和类型，表达不了「三选一」。它靠把字段藏起来：别的包连字段名都写不出来，状态只能从 `NewOrder` 或者这几个方法拿到。零值 `Order{}` 仍然合法（等于 `unpaid`），这是 Go 要留神的缝。

### Dart

```dart
sealed class Order {
  const Order();
}

class Unpaid extends Order { const Unpaid(); }
class Paid extends Order { const Paid(); }
class Shipped extends Order { const Shipped(); }

Order pay(Order o) {
  if (o is Unpaid) return const Paid();
  throw StateError('已经付过了');
}

Order ship(Order o) {
  if (o is Paid) return const Shipped();
  throw StateError('没付款不能发货');
}

Order refund(Order o) {
  if (o is Paid) return const Unpaid();
  throw StateError('发货之后不能退款');
}
```

`sealed` 让这三个子类成为封闭集合，`switch` 的时候编译器会替你查有没有漏；状态本身没有可变字段，变化只能换一个新的。

### TypeScript

```typescript
type Order =
  | { readonly state: "unpaid" }
  | { readonly state: "paid" }
  | { readonly state: "shipped" };

export function pay(o: Order): Order {
  if (o.state !== "unpaid") throw new Error("已经付过了");
  return { state: "paid" };
}

export function ship(o: Order): Order {
  if (o.state !== "paid") throw new Error("没付款不能发货");
  return { state: "shipped" };
}

export function refund(o: Order): Order {
  if (o.state !== "paid") throw new Error("发货之后不能退款");
  return { state: "unpaid" };
}
```

联合类型把三个状态写成三个变体，`readonly` 挡住就地改写；判断收窄到某个变体之后，编译器知道剩下的只可能是合法的那一步。

五份代码里没有一处 `invariant()`，也没有哪一行对着上面那条 `inv` 逐字翻译。规约层用两个布尔字段写，是因为这条规矩本来就在讲那两个字段的组合——真写成单一状态字段，那条 `inv` 就没地方写，规矩也就看不见了。反过来落到代码里，各语言更愿意把状态收成一个字段，这条规矩被别的东西替掉：上面四份是被类型替掉的，Go 那份是被「字段不导出、只能经方法产生」替掉的；库里如果还留着两个布尔列，转换放在边界上，进到领域只剩一个状态字段。

常见去处还有两个：被测试替掉（属性测试随机生成操作序列，每一步之后断言不变量仍成立），被离线工具替掉（TLA+、Alloy 这类模型检查器，Dafny、SPARK 这类带契约的验证器）；持久层再加一条 CHECK 约束，任何写路径都绕不过。

再往前还有一格：让状态本身成为类型参数，操作只挂在对应的状态类型上，`unpaid.ship()`、`shipped.refund()` 在编译期就不合法。Rust、TypeScript 这类语言写得出来，代价是类型数量随状态数增长，只适合状态少、变更慢的地方；Go 表达不了。

所以数学表达和代码实现不必一一对应。规约层写 `inv`，是为了把规则说清楚，这一步跟语言无关，不能跳；落到代码时，它常常整条不用写。数学推理是工具，不是教条。

## 本单元练习

订单再加一个字段，记用户取消过没有：

```text
State ::= paid      : Bool
          shipped   : Bool
          cancelled : Bool
```

两条新规则：取消过的订单不能发货；发货之后不能取消。

请写出 `Cancel` 操作的 pre 和 post，把新的不变量写出来（原来的 `shipped ⇒ paid` 要保留），再逐个操作验一遍：`Pay`、`Ship`、`Refund`、`Cancel` 四个做完之后不变量还在不在。加一个字段往往会逼你回头改旧操作的 pre，验的时候留意。

练习不附答案。写完对着这四条自检：

1. 状态写成了带不变量的数学记录
2. 每个操作独立写了 pre / post，不靠代码细节
3. 每个操作都能说清执行之后 inv 为什么仍然成立
4. 能指出每条约束该放 pre 还是该放 inv，不混用

上一篇：[用集合与逻辑描述系统的静态结构](./static-structure.md)
