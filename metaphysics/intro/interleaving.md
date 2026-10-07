---
description: 这篇教程面向的是只有一点离散数学基础或者实际编程经验的人；这一篇讲多个操作同时发生时，系统该怎么建模、要证哪两件不同的事；目的是让读者不再把「加锁压测没出事」当成并发安全的证明。
plots:
  - 业务需求
  - 哲学阐述
  - 数学表达
  - 代码实现
---

# 把并发摊成一组轨迹

## 业务需求

前面几篇默认操作一个接一个地走。真实系统不是这样：多个请求、多个线程、多个服务同时动同一份状态。

工程上的习惯做法是加锁，再压测。压测跑过几万次没出事，就说并发没问题。这里有两处不对。

一处是覆盖不全：并发出事的往往是某一种极少见的先后顺序，压测是把顺序撞出来的，撞不到就看不见。

另一处更要紧：「没问题」这三个字底下其实混着两件事。一件是不许发生的事——不能两个人同时持有同一把锁。另一件是必须发生的事——排在队里的人最终要拿到锁。压测没出事，可能只是还没轮到那个人。这两件事证法完全不同，得分开说。

## 哲学阐述

这一段要归位的是「同时发生」本身。

在模型里没有真正的「同时」。系统里存在的不是若干件同时进行的事，而是一组可能的先后顺序：先 A 后 B，或者先 B 后 A，两种都算。所以并发没有在存在清单上添新东西，它添的是一个集合——所有可能执行序列构成的集合。时间被从模型里拿掉了，只留下「谁先谁后都可能」这一条。

这么归位之后，两件事变得可行。一件是「任意调度下都成立」不再需要枚举无穷多种交错——只要证明每一单步都守规矩，任意顺序拼起来就都守规矩。另一件是前一篇那座桥不用重建：并发没有让细化变复杂，只是把「具体走一步」换成「在任意交错里走一步」，桥还是那座桥。

## 数学表达

先给并发系统一个形状：

```text
并发系统 = 所有操作的所有交错执行序列构成的集合
```

读作：把每个操作拆成一个个不能再分的动作步，然后把它们按各种可能的先后顺序摊开，摊出来的每一条完整序列叫一条轨迹；系统的全部行为就是这个轨迹的集合。

拿排队文件锁当例子。状态是：

```text
State ::= locked  : P(File)
          holders : File ⇸ User
          waiters : File → seq(User)
```

`seq(User)` 读作「一串用户」，也就是排队队列。两个用户 u1、u2 同时申请文件 f，系统实际执行出来的顺序可能是：

```text
u1 入队 → u2 入队 → u1 拿锁 → u2 继续等
u2 入队 → u1 入队 → u2 拿锁 → u1 继续等
```

抽象规格根本不关心谁先谁后，它只看到「最终 f 被某一个人持有，且只有一个人持有」。只要每一条交错轨迹都能投影回抽象的一条合法轨迹，这个并发实现就是合法的。

要证的东西分两类。第一类是安全性——坏的事永远不发生：

```text
∀ 可达状态 s · inv(s)
```

写成时态逻辑就是 `□inv`：`□` 读作「从此刻起，每一步都」，所以 `□inv` 读作「永远守着不变量」。文件锁的安全属性是：

```text
□ (card(locked ∩ dom(holders)) ≤ 1)
```

读作：每一步上，被锁住且有人持有的文件至多一个。`card` 是集合里的元素个数，`dom(holders)` 是「有人持有」的那些文件。

第二类是活性——好的事最终一定发生：

```text
∀ 轨迹 σ · 最终(某件好事发生)
```

时态逻辑里用 `◇` 和 `~>` 写。`◇P` 读作「最终会有那么一步让 P 成立」；`P ~> Q` 读作「P 一旦发生，之后终会到 Q」。文件锁的两条活性是：

```text
□ (u ∈ waiters(f) ~> u ∈ holders(f))                              // 进队的人终会拿锁
□ (waiters(f) ≠ ⟨⟩ ~> locked(f) ∧ holders(f) = head(waiters(f)))  // 队不空时，终会把锁交给队首
```

`⟨⟩` 是空序列，`head` 取队首。

两类属性证法不同，这一点必须记住。安全性靠归纳：对每一条操作，证明它执行之后不变量仍然成立——就是前面几篇一直在干的活。活性不行，它需要额外假设。上面那两条活性只有在「调度是公平的、队列按入队顺序服务、不被无限插队」时成立；调度器要是永远挑队尾服务，这两条就是假的。

所以「活性要显式写出公平假设」不是保守，是必需的。只证安全性不证活性，结果是系统永远不会崩，但可能永远卡在那里——用户对着转圈圈到天亮，而所有不变量都成立。

## 代码实现

并发在代码里的落点只有一个：把「读—判断—写」这几步串成一段不可打断的。剩下的事（排队、公平）落在队列和调度器上。

### Rust

```rust
struct LockManager {
    locked: HashSet<File>,
    waiters: Mutex<HashMap<File, VecDeque<User>>>,   // 队列本身也要保护
    cv: Condvar,                                      // 用来等待「轮到我」
}

fn release(&self, f: File) {
    let mut w = self.waiters.lock().unwrap();
    self.locked.remove(&f);
    if let Some(q) = w.get_mut(&f) {
        if let Some(next) = q.pop_front() {           // 先进先出，这条就是公平假设
            self.grant(&next, &f);
        }
    }
}
```

### Go

```go
type LockManager struct {
	mu      sync.Mutex
	locked  map[string]bool
	waiters map[string][]string // 每个文件的等待队列
}

func (m *LockManager) Release(f string) {
	m.mu.Lock()          // 读—判断—写串成一段
	defer m.mu.Unlock()
	delete(m.locked, f)
	q := m.waiters[f]
	if len(q) > 0 {
		m.grant(q[0], f)        // 取队首，FIFO 就是公平
		m.waiters[f] = q[1:]
	}
}
```

### Python

```python
class LockManager:
    def __init__(self):
        self._lock = threading.Lock()        # 保护状态的锁
        self._cv = threading.Condition(self._lock)
        self._locked: set[str] = set()
        self._waiters: dict[str, list[str]] = {}

    def release(self, f: str) -> None:
        with self._cv:                       # 读—判断—写串成一段
            self._locked.discard(f)
            q = self._waiters.get(f, [])
            if q:
                self._grant(q.pop(0), f)     # 取队首

    def acquire(self, u: str, f: str) -> None:
        with self._cv:
            while f in self._locked or self._queue_has_priority(u, f):
                self._waiters.setdefault(f, []).append(u)
                self._cv.wait()              # 让出执行权，交错的入口就在这里
            self._locked.add(f)
```

### TypeScript

```typescript
class LockManager {
  private locked = new Set<string>();
  private waiters = new Map<string, string[]>();
  private chain: Promise<void> = Promise.resolve();   // 用 promise 串成一段

  async release(f: string): Promise<void> {
    this.chain = this.chain.then(() => {   // 之前排队的操作先跑完
      this.locked.delete(f);
      const q = this.waiters.get(f) ?? [];
      const next = q.shift();
      if (next) this.grant(next, f);       // 取队首
    });
    return this.chain;
  }
}
```

### Dart

```dart
class LockManager {
  final Set<String> _locked = {};
  final Map<String, List<String>> _waiters = {};
  Future<void> _chain = Future.value();    // 用 future 串成一段

  Future<void> release(String f) {
    _chain = _chain.then((_) {
      _locked.remove(f);
      final q = _waiters[f] ?? [];
      if (q.isNotEmpty) _grant(q.removeAt(0), f);   // 取队首
    });
    return _chain;
  }
}
```

两点值得注意。

语言有没有真并行不影响这套模型。TypeScript 和 Dart 是单线程的，照样有交错——`await`、`Future`、`async` 让出执行权的那些位置就是交错的切点，两个请求的代码可以交错着跑。所以上面那套轨迹集对它们一样成立。

活性不在这几段代码里。「取队首」这几个字看着像公平，实际不是：它只说这次释放交给谁，没说每个等待者都会轮到、也没说不会被无限插队。公平在调度器那边，代码里写不出来，所以活性必须连同公平假设一起写成规约。

## 本单元练习

接着排队文件锁往下做。

1. 安全性：写出入队、拿锁、释放这三步各自怎么保住「至多一个持有者」。特别看拿锁那一步——它的前置条件里要不要查「我是队首」？漏了会怎样？
2. 活性：假设调度按入队顺序服务、不被无限插队（把这条假设写成一句话），把「u 不会饿死」写成 `~>` 公式。
3. 说明为什么第 2 问光靠安全性证不出来，必须把那条公平假设单独摆出来。

练习不附答案。写完对着这四条自检：

1. 说的是一组可能的执行顺序，不是几件同时发生的事
2. 安全性写成了 `□inv`，活性写成了 `◇` 或 `~>`，两者没有混着说
3. 公平假设单独写了，没有偷偷当成系统自带的性质
4. 并发下的细化只多一句「任意交错里走一步都可回溯」，没有重证前面证过的东西

上一篇：[证明实现没有削弱规格](./refinement.md)　下一篇：[把多个状态组合成一个系统](./composition.md)
