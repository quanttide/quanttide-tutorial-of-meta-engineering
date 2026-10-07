---
description: 这篇教程面向的是只有一点离散数学基础或者实际编程经验的人；这篇教程主要讲集合、关系、函数、逻辑这四样东西与代码里的数据模型、外键、查询函数、业务规则怎么一一对上；目的是让读者拿到一段需求时，能先把它的静态结构写成没有歧义的集合和逻辑式。
plots:
  - 业务意图
  - 数学表达
  - 代码实现
---

# 用集合与逻辑描述系统的静态结构

## 业务意图

一个用户有哪些角色，现在存在用户表的一个字段里，值形如 `admin,dev`——逗号拼起来的一串文字。要判断某人是不是管理员，就在这串文字里找 `admin`。

这么存有两个躲不掉的问题。

一是分不清「没有角色」和「角色名里正好含有 admin」。哪天有人加一个叫 `admin_readonly` 的角色，字符串匹配立刻误伤。

二是规则只能散着写。生意上有一条很硬的规则：只有管理员能删订单。它现在躺在某个删单接口的校验代码里，靠一次字符串包含判断。换个人再写一个删单入口，很容易忘了再判一次；也没有任何地方能一眼看到「系统里到底有哪些规则」。

这一篇管的是静态的部分：系统里有哪些东西、它们之间怎么关联、哪些断言永远为真。它们怎么随时间变化，是下一篇的事。

先把同一个意思在工程里和数学里各叫什么对齐。系统里的一类东西（所有用户、所有订单、所有角色）是一个集合；两个集合之间的关联（谁下了哪一单）是一个关系；一次查询或一次计算是一个函数；一条关于世界的断言（只有管理员能删订单）是一个逻辑式。

## 数学表达

数据域是集合。

```text
User = {alice, bob}
Role = {admin, dev, viewer}
```

读作：用户集合里眼下有两个元素，角色集合里有三个。`u ∈ User` 读作「u 是用户集合里的一个元素」，也就是「u 是一个用户」。

对象之间的关联是关系。

```text
owns ⊆ User × Order
```

`×` 是笛卡尔积，`User × Order` 读作「所有（用户, 订单）的配对」。`owns` 是这些配对里的一部分，配进去的每一对，表示该用户下了该订单。数据库里的外键，数学上就是这个东西。

关系的性质决定它能派什么用场。如果它自反、对称、又传递，就可以当等价关系用（分组、去重、规范化都能靠它说清楚）；如果它自反、传递、反对称，就可以当偏序关系用（排序、任务依赖、上下层关系靠它说清楚）。这些性质不用背，用到时回头查一次即可。

按 id 查用户、查某个用户的角色，是函数。

```text
roles         : User → P(Role)
getUserById   : UserId ⇸ User
```

`roles` 读作：给一个用户，得到一组角色。`P(Role)` 是角色集合的所有子集，也就是「任意挑几个角色组成的一堆」，所以 `roles(alice) = {admin, dev}` 读作 alice 的角色是 admin 和 dev。

`getUserById` 用 `⇸` 不用 `→`，因为查不到用户时它没有值。`⇸` 读作「最多一个」，`→` 读作「恰好一个」；一个查不到的 id 就是「零个」，所以这里是 `⇸`。这两个符号的区别就是代码里「返回可能为空」和「保证返回」的区别。

需求是一条逻辑式。

```text
∀ u ∈ User · canDelete(u) ⇒ admin ∈ roles(u)
```

读作：对每一个用户 u，如果他能删订单，那么 admin 在他的角色里。`∀` 读作「对每一个」，`∈` 读作「属于」，`⇒` 读作「如果左边成立，那么右边也必须成立」，中间的 `·` 只是把「对每一个」的范围和后面的断言隔开。

要求「存在至少一个」时把 `∀` 换成 `∃`，读作「存在一个」。下面这条读作：存在一笔订单，它的金额大于一千。

```text
∃ o ∈ Order · total(o) > 1000
```

工程里的说法和这里一一对应，换算出问题的时候对着这张单子看：类型和数据模型对应集合；表和关联对应关系；查询与计算对应函数；业务规则对应逻辑式。

## 代码实现

同一条数据模型和规则，五种语言各写一遍。读一种就够，其余对着上一节看差异。

### Python

```python
from dataclasses import dataclass, field
from enum import Enum

class Role(Enum):
    ADMIN  = "admin"
    DEV    = "dev"
    VIEWER = "viewer"

@dataclass(frozen=True)
class User:
    id: str
    roles: frozenset[Role] = field(default_factory=frozenset)  # roles : User → P(Role)

@dataclass(frozen=True)
class Order:
    id: str
    owner: str                                                 # owns ⊆ User × Order

def can_delete(u: User) -> bool:                               # 逻辑式落点
    return Role.ADMIN in u.roles
```

### Rust

```rust
use std::collections::HashSet;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
enum Role { Admin, Dev, Viewer }

#[derive(Debug)]
struct User {
    id: String,
    roles: HashSet<Role>,   // roles : User → P(Role)
}

#[derive(Debug)]
struct Order {
    id: String,
    owner: String,          // owns ⊆ User × Order
}

fn can_delete(u: &User) -> bool {   // 逻辑式落点
    u.roles.contains(&Role::Admin)
}
```

### Go

```go
type Role string

const (
	RoleAdmin  Role = "admin"
	RoleDev    Role = "dev"
	RoleViewer Role = "viewer"
)

type User struct {
	ID    string
	Roles map[Role]struct{} // roles : User → P(Role)
}

type Order struct {
	ID    string
	Owner string // owns ⊆ User × Order
}

func CanDelete(u User) bool { // 逻辑式落点
	_, ok := u.Roles[RoleAdmin]
	return ok
}
```

### Dart

```dart
enum Role { admin, dev, viewer }

class User {
  final String id;
  final Set<Role> roles; // roles : User → P(Role)
  const User(this.id, this.roles);
}

class Order {
  final String id;
  final String owner; // owns ⊆ User × Order
  const Order(this.id, this.owner);
}

bool canDelete(User u) => u.roles.contains(Role.admin); // 逻辑式落点
```

### TypeScript

```typescript
type Role = "admin" | "dev" | "viewer";

type User = { id: string; roles: Set<Role> };   // roles : User → P(Role)
type Order = { id: string; owner: string };     // owns ⊆ User × Order

function canDelete(u: User): boolean {          // 逻辑式落点
  return u.roles.has("admin");
}
```

角色从字符串换成一个取值范围被限死的类型（枚举、常量、联合类型），就是集合这件事在代码里的落点：`admin_readonly` 这种角色名进不来，「没有角色」和「有某个角色」不再靠字符串猜。`owner` 这个字段是关系 `owns` 的落点。`canDelete` 是那条逻辑式的落点——规则写成函数之后，「能删订单」和「是管理员」变成同一件事，蕴含关系落成了函数体本身。

## 本单元练习

设：

```text
Users = {alice, bob}
Roles = {admin, dev, viewer}
assigned : Users → P(Roles)
```

1. 写出 `assigned` 的数学类型，读一遍它的意思。
2. 用一阶逻辑表达：「每个 admin 都必须有 dev 角色。」
3. 用集合写：「没有任何用户同时是 viewer 和 admin。」
4. 把下面这句话翻译成公式：用户登录后，若没有双因子认证，则不能被分配 admin 角色。
5. 定义一个状态 `State = logged_in_users × pending_sessions`，再写一条不变量：pending session 的数量不超过 logged-in 用户的数量。

练习不附答案。写完对着这四条自检：

1. 数据域是用集合写的，不是用自然语言描述的
2. 对象之间的关联写成了关系，不是靠字段名暗示
3. 查询和计算写成了函数，不是伪代码
4. 需求落成了带 `∀`、`∃`、`⇒` 的句子，别人能照着检查

下一篇：[用状态机描述系统的动态变化](./state-machine.md)
