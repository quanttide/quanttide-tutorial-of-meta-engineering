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

同一条规则，五种语言各写一遍。读一种就够，其余对着上一节那三条 pre 看差异。

### Python

```python
from dataclasses import dataclass

@dataclass
class Order:
    paid: bool = False
    shipped: bool = False

    def invariant(self) -> bool:          # inv: shipped ⇒ paid
        return not (self.shipped and not self.paid)

    def pay(self) -> None:
        if self.paid:                     # pre: ¬paid
            raise ValueError("已经付过了")
        self.paid = True                  # post

    def ship(self) -> None:
        if not self.paid:                 # pre: paid
            raise ValueError("没付款不能发货")
        self.shipped = True

    def refund(self) -> None:
        if not self.paid or self.shipped: # pre: paid ∧ ¬shipped
            raise ValueError("发货之后不能退款")
        self.paid = False
```

### Rust

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Order { paid: bool, shipped: bool }

impl Order {
    fn invariant(&self) -> bool {         // inv: shipped ⇒ paid
        !self.shipped || self.paid
    }

    fn pay(&mut self) -> Result<(), &'static str> {
        if self.paid { return Err("已经付过了"); }     // pre: ¬paid
        self.paid = true;                             // post
        Ok(())
    }

    fn ship(&mut self) -> Result<(), &'static str> {
        if !self.paid { return Err("没付款不能发货"); } // pre: paid
        self.shipped = true;
        Ok(())
    }

    fn refund(&mut self) -> Result<(), &'static str> {
        if !self.paid || self.shipped {               // pre: paid ∧ ¬shipped
            return Err("发货之后不能退款");
        }
        self.paid = false;
        Ok(())
    }
}
```

### Go

```go
type Order struct {
	Paid    bool
	Shipped bool
}

func (o *Order) Invariant() bool { // inv: shipped ⇒ paid
	return !o.Shipped || o.Paid
}

func (o *Order) Pay() error {
	if o.Paid { // pre: ¬paid
		return errors.New("已经付过了")
	}
	o.Paid = true // post
	return nil
}

func (o *Order) Ship() error {
	if !o.Paid { // pre: paid
		return errors.New("没付款不能发货")
	}
	o.Shipped = true
	return nil
}

func (o *Order) Refund() error {
	if !o.Paid || o.Shipped { // pre: paid ∧ ¬shipped
		return errors.New("发货之后不能退款")
	}
	o.Paid = false
	return nil
}
```

### Dart

```dart
class Order {
  bool paid = false;
  bool shipped = false;

  bool get invariant => !shipped || paid;   // inv: shipped ⇒ paid

  void pay() {
    if (paid) throw StateError('已经付过了');          // pre: ¬paid
    paid = true;                                       // post
  }

  void ship() {
    if (!paid) throw StateError('没付款不能发货');       // pre: paid
    shipped = true;
  }

  void refund() {
    if (!paid || shipped) throw StateError('发货之后不能退款'); // pre: paid ∧ ¬shipped
    paid = false;
  }
}
```

### TypeScript

```typescript
type Order = { paid: boolean; shipped: boolean };

export function invariant(o: Order): boolean { // inv: shipped ⇒ paid
  return !o.shipped || o.paid;
}

export function pay(o: Order): Order {
  if (o.paid) throw new Error("已经付过了");            // pre: ¬paid
  return { ...o, paid: true };                          // post
}

export function ship(o: Order): Order {
  if (!o.paid) throw new Error("没付款不能发货");         // pre: paid
  return { ...o, shipped: true };
}

export function refund(o: Order): Order {
  if (!o.paid || o.shipped) throw new Error("发货之后不能退款"); // pre: paid ∧ ¬shipped
  return { ...o, paid: false };
}
```

同一个 pre，五种语言只是换个表达方式：Python、Dart、TypeScript 抛异常，Rust 返回 `Result`，Go 返回 `error`。`inv` 五处长得几乎一样，都是 `!shipped || paid`——它就是 `shipped ⇒ paid` 的机械翻译（`A ⇒ B` 等价于 `¬A ∨ B`）。状态放哪儿也分两派：Python、Rust、Go、Dart 改对象本身，TypeScript 那份返回新对象，旧状态不动。

Rust 和 TypeScript 还有另一条路：把 `paid`、`shipped` 两个字段收成一个状态字段（Rust 的 `enum Order { Unpaid, Paid, Shipped }`、TypeScript 的联合类型），非法组合在类型层面就不存在，`inv` 用不着写。那是另一条路，前提是状态集中在一个字段里；这套教程教的是状态散在两个字段、靠 inv 兜底的写法，因为生产系统里多半就是散着的。

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

上一篇：[学习单元 1 形式化建模的本体基元](./1_set.md)
