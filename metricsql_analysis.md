# MetricSQL 源码全面分析报告

## 1. 项目概述

**MetricSQL** (`github.com/VictoriaMetrics/metricsql`) 是 VictoriaMetrics 团队开发并维护的一个独立的 Go 语言库，用于解析 MetricsQL 和 PromQL。它在功能上向后兼容 Prometheus 的 PromQL，同时支持 VictoriaMetrics 独有的 MetricsQL 扩展特性（如 `WITH` 模板表达式、更丰富的聚合函数和 Rollup 函数等）。

### 核心功能
1. **解析 (Parsing)**: 将 PromQL/MetricsQL 查询字符串解析为抽象语法树 (AST)。
2. **优化 (Optimization)**: 对抽象语法树（AST）进行查询优化，特别是标签过滤器下推（Label Filter Pushdown）。
3. **格式化 (Prettifying)**: 提供代码美化功能，将 AST 重新转换为排版良好的查询字符串。

---

## 2. 代码结构与模块划分

通过统计，核心源码加上测试代码大约有 8000 多行。整个项目没有复杂的深层包结构，核心代码全部平铺在根目录下，主要分为以下几个功能模块：

* **核心解析引擎**: 
  * `lexer.go` (词法分析器)
  * `parser.go` (语法分析器与 AST 数据结构定义)
* **优化器**: 
  * `optimizer.go` (AST 标签优化器逻辑)
* **表达式类型系统**:
  * `aggr.go` (聚合函数定义，如 `sum`, `avg`)
  * `transform.go` (转换函数定义，如 `abs`, `sort`)
  * `rollup.go` (Rollup/区间函数定义，如 `rate`, `increase_over_time`)
  * `binary_op.go` / `binaryop/funcs.go` (二元操作符逻辑及底层数学运算)
* **辅助工具**:
  * `prettifier.go` (PromQL 代码格式化)
  * `regexp_cache.go` (正则表达式编译缓存，防止内存打满)
  * `utils.go` (AST 遍历等辅助函数)

---

## 3. 核心组件深入分析

### 3.1 词法分析器 (Lexer - `lexer.go`)

MetricSQL 使用了**手写词法分析器**，而不是依赖 `go yacc` 或 `goyacc` 等生成工具。
* **设计方式**: `lexer` 结构体维护了当前字符流的指针 (`sOrig` 和 `sTail`) 和当前获取的 `Token`。
* **零拷贝思想**: 在扫描 Ident（标识符）、String（字符串）或数值时，其切片大多直接复用底层输入字符串 `sOrig`，从而极大减少了内存分配（Allocation）。
* **Token 识别**:
  * `next()` 方法是核心状态机，它会跳过空白字符和注释 (`#`)，并根据前缀派发给不同的探测函数，例如 `isIdentPrefix`、`isStringPrefix`、`scanDuration` 等。
  * 对于时间单位（如 `5m`, `1h` 等），提供了针对普罗米修斯监控体系深度定制的 `scanDuration` 与解析函数 `PositiveDurationValue`。

### 3.2 抽象语法树与解析器 (Parser & AST - `parser.go`)

`parser.go` 包含了超过 2400 行代码，是项目的绝对核心，它采用**递归下降**（Recursive Descent）的方法来将 Token 组装成 AST。

#### AST 节点体系
所有的节点都实现了 `Expr` 接口：
```go
type Expr interface {
    AppendString(dst []byte) []byte
}
```
核心的 `Expr` 实现包括：
* **`MetricExpr`**: 指代普通的指标查询（如 `foo{bar="baz"}`）。
* **`RollupExpr`**: 指代带区间的查询，即带 `[]` 或者 `offset` 的表达式（如 `foo[5m] offset 1h`）。
* **`FuncExpr`**: 普通转换函数（如 `abs(foo)`）。
* **`AggrFuncExpr`**: 带有 `by` 或 `without` 的聚合函数（如 `sum(foo) by (job)`）。
* **`BinaryOpExpr`**: 二元运算表达式（如 `a + b` 或 `a > bool b`）。
* **`StringExpr` / `NumberExpr` / `DurationExpr`**: 基础的字面量表达式。

#### 解析流程 (`Parse` 方法)
1. 词法提取: `parser` 拿到 `lexer`。
2. 内部解析: 调用 `parseInternal` 开始**递归下降**解析。`parseExpr` 解析二元运算组合，`parseSingleExpr` 用于匹配小的前缀表达式。
3. `WITH` 展开: MetricsQL 支持 `WITH (x = a, y = b) x + y` 语法。解析后通过 `expandWithExpr` 提前将公共模板片段展开进入主 AST（这里是一种 AST 预处理宏展开机制）。
4. 括号移除运算: `removeParensExpr` 将多余的括号节点移除，简化 AST 层级。
5. 常量折叠 (Constant Folding): `simplifyConstants` 在解析期直接求值常量二元运算（如 `1 + 1` 直接优化为 `2`）。
6. 函数检查: `checkSupportedFunctions` 确保所有调用都是注册合法的。

### 3.3 函数及操作符字典 (`aggr.go`, `rollup.go`, `transform.go`, `binary_op.go`)

为了处理监控领域百余种不同的函数集，项目将其严格分为三大类，并以 `map[string]bool` 等形式注册：
* **AggrFunc (聚合函数)**: 会在空间维度合并时间序列（如 `sum`, `topk`）。
* **RollupFunc (区间函数)**: 处理 `a[5m]` 形式的区间向量，输出瞬时向量（如 `rate`, `histogram_over_time`）。
* **TransformFunc (转换函数)**: 处理单个/多个瞬时时间序列（如 `abs`, `day_of_month`）。

`binary_op.go` 不仅处理 `+`, `-`, `>`, `<=` 等，还维护了严格的**二元操作符优先级表** `binaryOpPriorities`，使得 `parser.go` 中 `balanceBinaryOp` 能够正确将一棵二叉树旋转以保证计算优先级。

### 3.4 查询优化器 (Optimizer - `optimizer.go`)

在 PromQL 的实际使用中，很常见的问题是用户写出了 `cpu_usage + sum(requests)` 但是缺乏关联维度，导致笛卡尔积或找不到匹配。
* **特性**: MetricSQL 提供了一个核心的 AST 优化功能。即尝试在二元操作符左右两边**下推共同的标签过滤条件 (Label Filter Pushdown)**。
* **原理**: 
  * 通过 `getCommonLabelFilters` 函数通过树的后续遍历收集出两边都具有的标签（或子树中包含的公共标签）。
  * 通过 `pushdownBinaryOpFiltersInplace` 递归地把这些公共条件补充（推入）到各个底层的普通的 `MetricExpr` 中去。
  * 这个优化（基于 UTCC 博客提倡的普罗米修斯非优化模式改进）能极大地提高复杂监控图表在 TSDB 执行引擎层面的过滤速度与准确性。

### 3.5 正则表达式缓存 (`regexp_cache.go`)

由于 Prometheus 标签经常使用 `=~"regex"`，高并发解析会导致大量正则匹配编译，对内存和 CPU 是极大的开销。
* 项目专门实现了一个轻量级的并发行正则缓存 `regexpCache`。
* 此缓存采用了通过**限制字符总长度 (`charsLimit` = 1,000,000)** 而不是条目数的淘汰机制（保障内存总量不爆），并内置了针对监控平台的自建缓存监控指标 (`vm_cache_requests_total`, `vm_cache_misses_total` 等)。

---

## 4. 架构设计亮点与总结

1. **手写 Parser 带来的极致性能与无黑盒化**：由于无需通过 goyacc 生成代码，Go 开发者可以直观地跟踪每一步的解析步骤，便于引入针对 MetricsQL 的特殊语法（如 `WITH` 宏、各种特殊的 `[1m] offset 1m` 及 `if`/`ifnot` 操作符）并做精准的报错捕获。
2. **零依赖与精简哲学**: 第三方库依赖极少。通过把各个底层计算独立出来（如 `binaryop/funcs.go` 提供极端的浮点 NaN 安全处理），做到了高内聚。
3. **安全导向**: 通过对 AST 进行常数级别限制的展开（和基于 Limit 的正则缓存），可以非常有效地防止恶意/超大查询带来的内存 OOM 攻击。
4. **编译期常数折叠 (Constant Folding) 与 AST 预审**: 把 `1+1` 这种表达式在解析的时候就做计算，并在 `parser.go` 里面内置大量检测 `isLikelyInvalid` （比如对于不带时间范围求 rate: `rate(foo)` 给警告）。它不仅是一个解析器，还是一个 Linter 静态逻辑分析引擎。

这就是 `VictoriaMetrics/metricsql` 的源码全貌——一个为高性能时序数据库环境量身打造、拥有严密异常捕获和主动优化特性的编译原理级解析引擎。
