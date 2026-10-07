---
description: 这篇教程面向的是只有一点离散数学基础或者实际编程经验的人；这一篇讲怎么证明一份具体实现没有偷偷削弱它上面的抽象规格；目的是让读者遇到「代码和文档对不上」时，能把这件事变成一个可以证的问题。
plots:
  - 业务需求
  - 哲学阐述
  - 数学表达
  - 代码实现
---

# 证明实现没有削弱规格

## 业务需求

架构师写一份高层规格，程序员写一份实现。两份东西摆在一起，怎么知道它们说的是同一件事，而且实现没有偷偷少做？

第一反应是逐字段比对、看函数名对不对得上。这个直觉是错的，看一个反例。抽象规格对扣款的描述只有两句：余额减少，不变量保持。实现里加了审计日志，加了风控检查，还把那一步拆成三段——先冻结、再扣减、最后解冻。字段没一个对得上，函数名也不同，但它仍然是这份规格的合法实现：规格没禁止写日志，只是没要求；规格保证的「余额不会变负」，实现也守住了。

所以要比的不是结构，而是这一层对外承诺了什么。这一篇讲的就是怎么把这件事写成一个可以证的问题。

## 哲学阐述

先把问题问清楚：抽象规格和具体实现，是两类不同的东西吗？

不是。它们是同一个东西的两层说法。抽象层描述的是看得见的那些状态和动作，具体层在它之上多了一层内部细节。多出来的东西——日志、冻结额、阶段字段——不是新的存在物，是抽象层看不见的存在物。

在存在清单上归完位，判断标准就出来了：两层是不是同一个东西，不看清单像不像，看两件事。具体层每做一步，抽象层能不能找到对应的一步；抽象层承诺过的，具体层有没有守住。前者管的是实现没自作主张，后者管的是实现没偷工减料。

借用范畴论的说法：规格是一个对象，实现是另一个对象，「这份实现是这份规格的落地」就是从实现指向规格的那条箭头。几份实现长得各不相同，却能各画一条箭头指回同一个规格——这也是后面组合那一篇能只做局部论证的底子。

## 数学表达

抽象机 A 和具体机 C 各自都是前面那种状态机：状态空间、初始状态、一组操作、一条不变量。要证 C 是 A 的落地，标准做法是找一座桥，叫模拟关系：

```text
R ⊆ S_C × S_A
```

读作：`R` 是一堆「具体状态与抽象状态」的配对，配进去表示这个具体状态可以看成那个抽象状态。

有了桥，证三条就够：

```text
1. 初始化一致   ∀ c ∈ init_C · ∃ a ∈ init_A · R(c, a)
2. 单步可回溯   ∀ (c, a) ∈ R · 具体走一步 c → c'
                ⇒ 抽象也能走一步 a → a'，且 R(c', a')
3. 不变量继承   具体满足自己的不变量 ⇒ 翻译回去的抽象状态满足抽象不变量
```

第一条读作：每个可能的初始具体状态都能翻译回一个合法的初始抽象状态。第二条读作：具体每走一步，抽象都必须能跟着走一步，走完两边还连着。第三条读作：具体那边没破规矩，翻回去的抽象状态也没破。三条证完，这份实现就是这份规格的落地——B 方法、Event-B、TLA+ 里的细化证明都是这个模板。

拿一笔支付扣款走一遍。抽象规格只有一步扣款：

```text
State_A ::= balance : Account → ℤ
inv_A   ≡ ∀ a · balance(a) ≥ 0

Debit(a, amt):
  pre:  amt ≤ balance(a)
  post: balance' = balance ⊕ {a ↦ balance(a) − amt}
```

具体实现把那一步拆成两段，多出两个字段：

```text
State_C ::= balance : Account → ℤ        // 可见余额
            frozen  : Account → ℤ        // 已冻结、还没扣完的部分
            phase   : Account → {idle, frozen}

Freeze(a, amt):
  pre:  phase(a) = idle ∧ amt ≤ balance(a)
  post: frozen'  = frozen ⊕ {a ↦ amt}
        balance' = balance ⊕ {a ↦ balance(a) − amt}
        phase'   = phase ⊕ {a ↦ frozen}

Commit(a):
  pre:  phase(a) = frozen
  post: frozen' = frozen ⊕ {a ↦ 0}
        phase'  = phase ⊕ {a ↦ idle}
```

桥这么搭：

```text
R(c, a) ≡ balance_A(a) = balance_C(c) + frozen_C(c)
```

读作：抽象那边的余额，等于具体这边看得见的余额加上还没扣完的冻结额。抽象层看不见「冻结」这个内部东西，翻译的时候要把它加回去。

三条义务逐条过。初始时 `frozen = 0`，两边余额相等，第一条成立。`Freeze` 走一步之后，具体把 `amt` 从 `balance` 挪进 `frozen`，抽象那边走一步 `Debit` 直接扣掉 `amt`，两条式子仍然相等，第二条成立。`Commit` 在抽象里没有对应动作——这完全合法，抽象层看不见的内部收尾不需要外部动作配合。不变量方面，抽象要求 `balance_A ≥ 0`；按桥换过去，具体这边 `balance_C = balance_A − frozen ≥ 0` 自然成立，第三条成立。

两条线记牢，一句话：实现可以更严、更细、更啰嗦，不能更松、更粗、更偷懒。拆开是三对照。信息上，具体层可以藏内部细节，不能暴露抽象没承诺过的可观察行为。保证上，具体层的前置条件可以更严（更早拦住非法调用），不能削弱抽象承诺的后置条件和不变量。步骤上，具体层可以把一步拆成多步，不能把多步合成一步还声称是同一份规格——那是抽象，方向反了。

## 代码实现

细化在代码里最常见的形态就是接口与实现。下面五份写的是同一件事，不追求跟上面那份规约逐行对上——接口写抽象层承诺了什么，实现写这一层多了什么。

### Rust

```rust
trait Account {
    fn balance(&self) -> i64;
    fn debit(&mut self, amt: i64) -> Result<(), &'static str>;
}

struct FrozenAccount {         // 具体层：多了一个内部冻结额
    balance: i64,
    frozen: i64,
}

impl Account for FrozenAccount {
    fn balance(&self) -> i64 { self.balance + self.frozen }   // 投影回抽象层
    fn debit(&mut self, amt: i64) -> Result<(), &'static str> {
        if amt > self.balance { return Err("余额不足"); }      // 更严的前置条件
        self.frozen += amt;
        self.balance -= amt;
        Ok(())
    }
}
```

### TypeScript

```typescript
interface Account {                    // 抽象层：只承诺这两个
  balance(): number;
  debit(amount: number): void;
}

class FrozenAccount implements Account {
  private frozen = 0;                  // 具体层的内部细节
  constructor(private visible: number) {}

  balance(): number { return this.visible + this.frozen; }
  debit(amount: number): void {
    if (amount > this.visible) throw new Error("余额不足");
    this.frozen += amount;
    this.visible -= amount;
  }
}
```

### Python

```python
from typing import Protocol

class Account(Protocol):               # 抽象层
    def balance(self) -> int: ...
    def debit(self, amount: int) -> None: ...

class FrozenAccount:                   # 具体层
    def __init__(self, visible: int = 0) -> None:
        self._visible = visible
        self._frozen = 0

    def balance(self) -> int:
        return self._visible + self._frozen

    def debit(self, amount: int) -> None:
        if amount > self._visible:
            raise ValueError("余额不足")
        self._frozen += amount
        self._visible -= amount
```

### Go

```go
type Account interface { // 抽象层
	Balance() int
	Debit(amount int) error
}

type FrozenAccount struct { // 具体层，字段不导出
	visible int
	frozen  int
}

func (a *FrozenAccount) Balance() int { return a.visible + a.frozen }

func (a *FrozenAccount) Debit(amount int) error {
	if amount > a.visible {
		return errors.New("余额不足")
	}
	a.frozen += amount
	a.visible -= amount
	return nil
}
```

### Dart

```dart
abstract class Account {           // 抽象层
  int balance();
  void debit(int amount);
}

class FrozenAccount implements Account {
  int _visible;
  int _frozen = 0;                 // 具体层的内部细节
  FrozenAccount([this._visible = 0]);

  @override
  int balance() => _visible + _frozen;

  @override
  void debit(int amount) {
    if (amount > _visible) throw StateError('余额不足');
    _frozen += amount;
    _visible -= amount;
  }
}
```

接口那一份就是抽象层，实现那一份就是具体层，那座桥落在 `balance()` 这个方法体里——抽象余额等于可见余额加冻结额。在代码里，「翻译回抽象状态」这件事由编译器或检查器替你验了一半：实现类必须把接口声明的方法都实现出来。

它验不了另一半。接口里没有那条「余额不能为负」，代码里得另外有人守——构造函数的检查、数据库上的 CHECK 约束、属性测试每一步之后的断言，哪一种都行。这正是上一篇代码实现末尾说的那件事：规约层写下的规矩，落到代码里常常不写成一行检查，而是被类型、约束、测试接走。

## 本单元练习

下面三处改动，各判断是合法细化还是偷偷削弱了规格，说出理由。再给第三处写一座桥。

1. 抽象规格说「发货之后不能取消」，实现里改成「发货之后十五分钟内还可以取消」。
2. 抽象规格说「付款成功返回订单号」，实现里除了订单号还写了一条审计日志。
3. 抽象规格说「余额不得为负」，实现把一次扣款拆成了先冻结、再扣减、最后解冻三步。

第 3 处要交的东西：具体状态空间、三步操作、模拟关系 `R`。

练习不附答案。写完对着这四条自检：

1. 判断的理由说的是「对外承诺变没变」，不是「字段像不像」
2. 找到的桥能把每个具体状态翻回一个合法抽象状态
3. 三条义务逐条过了：初始化一致、单步可回溯、不变量继承
4. 能指出哪些多出来的内部细节是抽象层看不见的，因此不需要抽象层配合

上一篇：[用状态机描述系统的动态变化](./state-machine.md)　下一篇：[把并发摊成一组轨迹](./interleaving.md)
