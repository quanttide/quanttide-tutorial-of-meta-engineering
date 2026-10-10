---
description: 这篇教程面向的是只有一点离散数学基础或者实际编程经验的人；这一篇讲多个请求同时动同一笔钱时，系统该怎么建模、要证哪两件不同的事；目的是让读者不再把「加锁压测没出事」当成并发安全的证明。
plots:
  - 需求
  - 意图
  - 规格
  - 实现
---

# 把并发摊成一组轨迹

## 需求

前面几篇默认操作一个接一个地走。支付系统不是这样：同一个账户上可能同时来两笔支付，回调会重试，用户会连点两次，多个服务会同时读同一份余额。

工程上的习惯做法是加锁，再压测。压测跑过几万次没出事，就说并发没问题。这里有两处不对。

一处是覆盖不全：并发出事的往往是某一种极少见的先后顺序，压测是把顺序撞出来的，撞不到就看不见。两笔支付各自读到余额 100，各自认为扣 80 没问题，最后余额变成 −60——这种顺序十次里可能一次都撞不到。

另一处更要紧：「没问题」这三个字底下混着两件不同的事。一件是不许发生的事——余额不能变成负数。另一件是必须发生的事——排队等着处理的那笔支付最终要被处理掉。压测没出事，可能只是还没轮到那一笔。这两件事证法完全不同，得分开说。

## 意图

这一段要归位的是「同时发生」本身。

在模型里没有真正的「同时」。系统里存在的不是两笔支付同时进行，而是一组可能的先后顺序：先处理 p1 再处理 p2，或者反过来，两种都算。所以并发没有在存在清单上添新东西，它添的是一个集合——所有可能执行序列构成的集合。时间被从模型里拿掉了，只留下「谁先谁后都可能」这一条。

这么归位之后，两件事变得可行。一件是「任意调度下都成立」不再需要枚举无穷多种交错——只要证明每一单步都守规矩，任意顺序拼起来就都守规矩。另一件是细化那一篇那座桥不用重建：并发没有让细化变复杂，只是把「具体走一步」换成「在任意交错里走一步」，桥还是那座桥。

写成式子。先给并发系统一个形状：

```text
并发系统 = 所有操作的所有交错执行序列构成的集合
```

读作：把每个操作拆成一个个不能再分的动作步，再把它们按各种可能的先后顺序摊开，摊出来的每一条完整序列叫一条轨迹；系统的全部行为就是这个轨迹的集合。

拿「同一个账户上的并发支付」当例子。状态是：

```text
State ::= balance : Account → ℤ
          held    : Account ⇸ Payment      // 正在改这个账户的那笔支付
          waiters : Account → seq(Payment) // 排队等这个账户的支付
          handled : P(Payment)             // 已经处理完的支付

inv ≡ ∀ a · balance(a) ≥ 0
```

`seq(Payment)` 读作「一串支付单」，也就是排队队列。`held` 用 `⇸` 不用 `→`：一个账户至多有一笔支付在改它，所以它是偏函数；这条性质不变量里不用再写，类型已经说完了。

两笔支付 p1、p2 同时来扣同一个账户，系统实际执行出来的顺序可能是：

```text
p1 读余额 → p2 读余额 → p1 扣减 → p2 扣减        // 两个都基于旧余额，余额被多扣
p1 拿账户 → p1 扣减 → p1 放账户 → p2 拿账户 → p2 扣减
```

抽象规格不关心谁先谁后，它只看到「最终余额正确，且不小于零」。第一种交错会破掉 `balance(a) ≥ 0`，所以它必须被排除——排除的办法就是让每个请求把「读余额、判断、扣减」放进 `held` 保护的一段里，谁也插不进去。

要证的东西分两类。第一类是安全性——坏的事永远不发生：

```text
∀ 可达状态 s · inv(s)
```

写成时态逻辑就是 `□inv`：`□` 读作「从此刻起，每一步都」，所以 `□inv` 读作「永远守着不变量」。这里的安全属性是：

```text
□ (∀ a · balance(a) ≥ 0)
□ (∀ a · card({p | held(a) = p}) ≤ 1)
```

第一条读作：每一步上，任何账户的余额都不为负。第二条读作：每一步上，任何账户在改它的支付至多一笔。

第二类是活性——好的事最终一定发生：

```text
∀ 轨迹 σ · 最终(某件好事发生)
```

时态逻辑里用 `◇` 和 `~>` 写。`◇P` 读作「最终会有那么一步让 P 成立」；`P ~> Q` 读作「P 一旦发生，之后终会到 Q」。这里的活性是：

```text
□ (p ∈ waiters(a) ~> p ∈ handled)              // 排队的支付终会被处理
□ (waiters(a) ≠ ⟨⟩ ~> held(a) = head(waiters(a)))   // 队不空时，终会轮到队首
```

`⟨⟩` 是空序列，`head` 取队首。

两类属性证法不同，这一点必须记住。安全性靠归纳：对每一条动作步，证明它执行之后不变量仍然成立——就是前面几篇一直在干的活。活性不行，它需要额外假设。上面那两条活性只有在「调度公平、队列按入队顺序服务、不被无限插队」时才成立；调度器要是永远挑队尾服务，这两条就是假的。

所以「活性要显式写出公平假设」不是保守，是必需的。只证安全性不证活性，结果是余额永远正确，但某笔支付可能永远卡在队里——用户付了钱，订单一直没动静，而所有不变量都成立。

## 规格

把要证的两类性质写成机器能读、与编程语言无关的一份属性规格，公平假设明摆在头上：

```text
system ConcurrentCharge {
  behavior: 所有操作的交错轨迹集合

  safety:
    □ ∀ a · balance(a) ≥ 0
    □ 每个账户在改它的支付至多一笔

  liveness (fair: 调度按入队顺序、不被无限插队):
    □ p ∈ waiters(a) ~> p ∈ handled
    □ waiters(a) ≠ ⟨⟩ ~> held(a) = head(waiters(a))
}
```

安全与活性分两块写，谁也不混谁；公平假设不藏在字里行间，写成规格自己的一行。这份规格模型检查器直接吃，五种语言的实现对着它证——哪份实现只上了锁没管公平，对着规格一眼看出来。

## 实现

并发在代码里的落点只有一个：把「读—判断—写」这几步串成一段不可打断的。剩下的事（排队、公平）落在队列和调度器上。

### Rust

```rust
struct Ledger {
    balance: HashMap<Account, i64>,
    held: HashMap<Account, Payment>,          // 谁正在改这个账户
    waiters: Mutex<HashMap<Account, VecDeque<Payment>>>,
    cv: Condvar,                              // 用来等「轮到我」
}

/// 把读—判断—写收在一段里，外面插不进来
fn charge(&mut self, acct: Account, p: Payment, amount: i64) -> Result<(), &'static str> {
    let mut w = self.waiters.lock().unwrap();
    while self.held.contains_key(&acct) || self.queue_has_priority(&acct, &p) {
        w.entry(acct).or_default().push_back(p);   // 排队，等前面的人
        w = self.cv.wait(w).unwrap();
    }
    let bal = self.balance[&acct];
    if amount > bal {
        return Err("余额不足");
    }
    self.balance.insert(acct, bal - amount);
    self.held.remove(&acct);
    self.cv.notify_all();                          // 把账户交给下一位
    Ok(())
}
```

### Go

```go
type Ledger struct {
	mu      sync.Mutex      // 读—判断—写串成一段
	balance map[string]int64
	held    map[string]string   // 账户 → 正在处理它的那笔支付
	waiters map[string][]string // 账户 → 排队等它的支付
}

func (l *Ledger) Charge(acct, p string, amount int64) error {
	l.mu.Lock()
	defer l.mu.Unlock()
	if amount > l.balance[acct] {
		return errors.New("余额不足")
	}
	l.balance[acct] -= amount
	delete(l.held, acct)
	q := l.waiters[acct]
	if len(q) > 0 {
		l.held[acct] = q[0]      // 取队首，FIFO 就是公平
		l.waiters[acct] = q[1:]
	}
	return nil
}
```

### Python

```python
class Ledger:
    def __init__(self) -> None:
        self._cv = threading.Condition()
        self._balance: dict[str, int] = {}
        self._held: dict[str, str] = {}          # 账户 → 正在处理它的支付
        self._waiters: dict[str, list[str]] = {} # 账户 → 排队等它的支付
        self._handled: set[str] = set()

    def charge(self, acct: str, p: str, amount: int) -> None:
        with self._cv:                            # 读—判断—写串成一段
            while acct in self._held or self._queue_has_priority(acct, p):
                self._waiters.setdefault(acct, []).append(p)
                self._cv.wait()                   # 让出执行权，交错的入口就在这里
            if amount > self._balance.get(acct, 0):
                raise ValueError("余额不足")
            self._balance[acct] -= amount
            self._handled.add(p)
            self._release(acct)
            self._cv.notify_all()
```

### TypeScript

```typescript
class Ledger {
  private balance = new Map<string, number>();
  private held = new Map<string, string>();       // 账户 → 正在处理它的支付
  private chain: Promise<void> = Promise.resolve(); // 用 promise 串成一段

  async charge(acct: string, p: string, amount: number): Promise<void> {
    this.chain = this.chain.then(() => {          // 之前排队的操作先跑完
      const bal = this.balance.get(acct) ?? 0;
      if (amount > bal) throw new Error("余额不足");
      this.balance.set(acct, bal - amount);
      this.held.delete(acct);
    });
    return this.chain;
  }
}
```

### Dart

```dart
class Ledger {
  final Map<String, int> _balance = {};
  final Map<String, String> _held = {};           // 账户 → 正在处理它的支付
  Future<void> _chain = Future.value();           // 用 future 串成一段

  Future<void> charge(String acct, String p, int amount) {
    _chain = _chain.then((_) {
      final bal = _balance[acct] ?? 0;
      if (amount > bal) throw StateError('余额不足');
      _balance[acct] = bal - amount;
      _held.remove(acct);
    });
    return _chain;
  }
}
```

两点值得注意。

语言有没有真并行不影响这套模型。TypeScript 和 Dart 是单线程的，照样有交错——`await`、`Future`、`async` 让出执行权的那些位置就是交错的切点，两个请求的代码可以交错着跑。所以上面那套轨迹集对它们一样成立。

活性不在这几段代码里。「取队首」这几个字看着像公平，实际不是：它只说这次交给谁，没说每个等待者都会轮到、也没说不会被无限插队。公平在调度器那边，代码里写不出来，所以活性必须连同公平假设一起写成规格。

## 本单元练习

接着账户并发扣款往下做。

1. 安全性：写出拿账户、扣减、交还这三步各自怎么保住「余额不为负」。特别看扣减那一步——它的前置条件里要不要查「我是队首」？漏了会怎样？
2. 活性：假设调度按入队顺序服务、不被无限插队（把这条假设写成一句话），把「p 不会饿死」写成 `~>` 公式。
3. 说明为什么第 2 问光靠安全性证不出来，必须把那条公平假设单独摆出来。

练习不附答案。写完对着这四条自检：

1. 说的是一组可能的执行顺序，不是几件事同时发生
2. 安全性写成了 `□inv`，活性写成了 `◇` 或 `~>`，两者没有混着说
3. 公平假设单独写了，没有偷偷当成系统自带的性质
4. 并发下的细化只多一句「任意交错里走一步都可回溯」，没有重证前面证过的东西

上一篇：[证明实现没有削弱规格](./refinement.md)　下一篇：[把多个状态组合成一个系统](./composition.md)
