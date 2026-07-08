# DoWhy — 因果推理与反事实分析库

> **库地址**: https://github.com/py-why/dowhy  
> 作者: Microsoft PyWhy 团队  
> 许可: MIT

## 是什么

DoWhy 是一个 Python 因果推理库，核心方法论是：**把假设显式化 → 用数据检验 → 自动化验证**。它的四步流程（建模 → 识别 → 估计 → 验证）与量潮战略沙盘的"反事实推演 → 日志校验"逻辑高度对应。

## 为什么在战略管理语境中参考它

### 1. 假设是 first-class citizen

DoWhy 最大的设计决策：每一步分析必须显式声明因果假设（causal graph）。假设不是隐含的、临时的，而是可序列化、可版本化、可复用的独立数据结构。

#### 具体实现：三层机制

**第一层 — CausalModel 建模**

把假设变成第一等的数据结构，核心是 `CausalModel` 类：

```python
import dowhy
from dowhy import CausalModel

model = CausalModel(
    data=df,                    # 观察到的数据
    treatment="培训参与",       # 干预变量
    outcome="交付效率",         # 结果变量
    common_causes=["团队经验"]  # 混淆变量（必须控制）
)
```

也可以在 DOT 语言中精确描述因果图——这就是假设的**序列化形式**，可存文件、可版本控制、可复用：

```python
model = CausalModel(
    data=df,
    treatment="培训参与",
    outcome="交付效率",
    graph="digraph {培训参与 -> 交付效率; 团队经验 -> 培训参与; 团队经验 -> 交付效率;}"
)
```

**第二层 — refute_estimate 自动校验**

DoWhy 不假设因果图是对的，自动用数据检验：

```python
# 拆分子集重算，看结论是否稳定
model.refute_estimate(method="data_subset_refuter")

# 加入随机混淆变量，看结论是否动摇
model.refute_estimate(method="random_common_cause")
```

如果数据不符合因果图中的假设，refutation 会报错——和"零信任"原则一致：假设可能错，让数据来验证。

**第三层 — counterfactual 反事实计算**

反事实不是"想象"，而是基于因果图和数据的**计算**：

```python
identified = model.identify_effect(proceed_when_unidentifiable=False)
estimate = model.estimate_effect(identified, method_name="backdoor.linear_regression")

# 反问：如果培训参与=0，交付效率是多少？
# 对比实际值，差值就是因果效应
```

#### 映射到量潮框架

```
DoWhy                         量潮可借鉴的形式
─────────                    ─────────────────
CausalModel(causal_graph)    profile/ 下每条战略的"依赖图"
  ↓ 边 = 假设                   ↓ 环境信号 → 战略决策 的箭头
refute_estimate()             journal/ 中找证据验证
  ↓ 自动校验                     ↓ AI 找"支持/挑战"
counterfactual()              "如果订单周期不变，决策是否一致"
```

**最直接可用的做法**：把 `profile/strategy/index.json` 里的每条战略决策，关联它依赖的环境变量（来自 `environment/index.json`），形成一个简单的假设-依赖表。不需要图算法，有 DSL 描述就够了。

### 2. 四步流程映射

| DoWhy | 量潮战略沙盘 |
|-------|-------------|
| **建模** — 构建因果图，声明假设 | **System 1** — 从战略上下文反推隐含假设 |
| **识别** — 确定要估什么（因果问题） | **区分** — 哪些假设可验证、哪些不可验证 |
| **估计** — 用数据算因果效应 | **System 2** — 从日记中找支持/挑战证据 |
| **验证** — 敏感性分析，自动检验假设 | **（目前缺失）** 自动测试"去掉某假设结论是否动摇" |

### 3. 反事实的具体化

DoWhy 做反事实不是"想象"，而是基于数据和模型的计算：给定因果图，问"如果变量 X 取不同值，Y 会怎样？"

这比 LLM 生成的文本反事实更严谨。量潮框架可以借鉴的抽象：**战略假设对关键环境变量的敏感性分析**——如果"订单周期变长"这个环境信号变了，哪些战略假设会动摇？

### 4. 分离识别与估计

DoWhy 刻意把"要做什么"和"怎么做"分开。这是它最有工程价值的思路：**工具的责任边界比功能更重要**。

对应到量潮，战略管理中的责任分离是：
- **人**负责：定义问题、做判断、承担后果
- **AI**负责：信息检索、证据发现、模式识别
- （当前代码里两个系统都交 AI 做，这是和日志哲学有差距的地方）

## 不适合参考的部分

- **数学假设**：DoWhy 的因果推断依赖严格的统计假设（混淆变量、工具变量等），战略管理的数据很少能满足这些假设。参考它的**方法论设计**而非**算法实现**。
- **自动化程度**：DoWhy 试图减少人工介入，而量潮战略沙盘中人类是绝对主导。参考时要注意保持人类在 loop 中的位置。

## 参考来源

- 官方文档: https://www.pywhy.org/dowhy/
- 论文: "DoWhy: A Python Library for Causal Inference" (2021)
- 相关项目: EconML（heterogeneous treatment effects）、CausalNex（贝叶斯因果网络）

## 在本仓库中的关联

- `examples/default/examples/strategy/strategy.py` — 当前 System 1 + System 2 的 Python 原型
- `src/cli/` — 未来可考虑将"假设管理"、"证据索引"、"校验状态追踪"建模为独立的数据结构
