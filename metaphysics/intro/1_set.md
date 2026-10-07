# 学习单元 1 形式化建模的本体基元

要学可用于软件工程形式化建模的数学本体论，第一个学习单元不应从范畴论、模型论开始，而应从离散数学本体开始：集合—关系—函数—逻辑（Sets, Relations, Functions, Logic）。这是 Z、VDM、《B》、Alloy、TLA+、Refinement Types 的共同地基。

## 本单元要建立的直觉

软件系统可以被看成：

- 状态 = 一个集合里的元素
- 数据域 = 集合：User, Order, Account
- 数据库表 / 关联 = 关系：owns ⊆ User × Order
- 操作 / 接口 = 函数：createOrder : State × Cart → State
- 需求 / 不变式 = 逻辑公式：∀ u ∈ User · balance(u) ≥ 0

所以形式化建模不是「写数学」，而是把软件世界重写成：对象 ∈ 集合，对象之间有关系，操作是函数，性质用逻辑断言。

## 核心概念

### 集合 Set

- x ∈ A：x 是集合 A 的元素
- A ⊆ B：A 是 B 的子集
- 空集 ∅
- 公理：外延公理——两个集合元素完全相同就是同一个集合
- 罗素悖论：不能写「所有不包含自身的集合」，否则朴素概括公理推出矛盾；形式化方法因此用受控的集合构造，而不是「凡是性质就成集合」

软件含义：

- Customers
- ActiveSessions
- ValidTokens

### 集合运算

- 并：A ∪ B
- 交：A ∩ B
- 差：A \ B
- 对称差：A △ B
- 幂集：P(A) = A 的所有子集，建模「权限集合」「标签集合」
- 笛卡尔积：A × B = 有序对集合，建模状态元组 (user, session, cart)

### 关系 Relation

二元关系：R ⊆ A × B

例子：

owns ⊆ Users × Orders
parent_of ⊆ Person × Person

重要性质：

- 自反 reflexive
- 对称 symmetric
- 传递 transitive
- 反对称 antisymmetric

由此得到：

- 等价关系 → 分区 / 去重 / 规范化
- 偏序关系 → 状态序、任务依赖、细化关系

### 函数 Function

函数是「每个输入恰好一个输出」的特殊关系：f : A → B

分类：

- 单射 injective：不同输入不同输出
- 满射 surjective：值域覆盖整个 B
- 双射 bijective：可反转，可建立同构
- 偏函数 partial：f : A ⇸ B，只对 A 的一部分元素有定义

软件含义：

- getUserById : UserId ⇸ User（查不到用户时没有值，所以是偏函数而不是全函数）
- hash : Password → Hash（全函数，但不单射：不同密码可以映射到同一个摘要）
- deploy : Config × Artifact → DeploymentResult

### 命题逻辑 / 一阶逻辑

命题：P, Q，以及 P ∧ Q、P ∨ Q、¬P、P ⇒ Q

一阶逻辑加量词：

∀ x ∈ Users · hasLicense(x) ⇒ canDeploy(x)
∃ o ∈ Orders · o.owner = u ∧ o.total > 1000

这是需求的形式化语言：

- 安全性 safety：永远不会进入非法状态
- 活性 liveness：最终总会达成某结果

## 形式化建模中的最小映射

- 类型 / 数据模型 → 集合
- 表 / 外键 → 关系
- API / 纯函数 → 函数
- 业务规则 → 谓词
- 系统状态 → 元组 / 记录 / 集合元素
- 不变量 → 逻辑公式
- 状态迁移 → 状态集合上的函数
- 规格说明 → 满足某些逻辑性质的数学结构

## 本单元练习

以下练习必须做。本单元不附答案。

设：

Users = {alice, bob}
Roles = {admin, dev, viewer}
assigned : Users → P(Roles)

1. 写出 assigned 的数学类型。
2. 用一阶逻辑表达：「每个 admin 都必须有 dev 角色。」
3. 用集合写：「没有任何用户同时是 viewer 和 admin。」
4. 把下面需求翻译成公式：

用户登录后，若没有双因子认证，则不能被分配 admin 角色。

5. 定义一个状态：

State = logged_in_users × pending_sessions

写一个不变量：pending session 数不超过 logged-in 用户数。

## 本单元结束标准

你能做到这 5 件事，才算学完 Unit 1：

1. 用集合而不是自然语言定义数据模型
2. 用关系表达数据库/对象关联
3. 用函数表达操作而非「伪代码」
4. 用 ∀ / ∃ / ⇒ 把需求写成可验证的语句
5. 明白「类型 ≈ 集合，程序 ≈ 集合间的函数，规格 ≈ 逻辑约束」

## 下一单元预告

Unit 2 状态机与前后置条件：

- 状态空间 S
- 操作 op : S × Input → S
- 前置条件 pre
- 后置条件 post
- 不变量 inv
- 细化 refinement：抽象规格 → 具体实现

下一篇：[学习单元 2 状态机与前后置条件](./2_state.md)
