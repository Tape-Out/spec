# `ip.yaml` 规范 v0.1

一个包一份 `ip.yaml`。它同时承担两件事：**描述这个包**（契约、旋钮、依赖、面积），以及在带 `instances:` 时**描述一次装配**。叶子 IP 与整颗 SoC 用的是同一份 schema，只差有没有 `instances:` 段。

元数据字段对齐 ECOS 的 `ip.yaml`；旋钮的约束语义取自 kconfig；依赖模型取自 cargo。三处借鉴各管一层，互不覆盖。

---

## 一、骨架

```yaml
name: gpio
version: 0.1.0          # 本包 semver
spec: "0.1"             # 遵循的本规范版本
kind: ip                # ip | library，省略即 ip

identity:   { ... }     # 身份与目录元数据，对齐 ECOS
contract:   { ... }     # 契约签名
params:     { ... }     # 数值旋钮
features:   { ... }     # 布尔与档位旋钮
constraints: [ ... ]    # 跨旋钮约束
area:       { ... }     # 面积基线与模型
emit:       [ ... ]     # 交付形态
deps:       { ... }     # 依赖
instances:  [ ... ]     # 可选。有它即为装配
```

---

## 一之二、包分两类

`kind` 省略时是 `ip`。

| `kind` | 会被例化 | 有契约 / 寄存器 / 面积 | 例 |
|:--|:--:|:--:|:--|
| **`ip`** | 是 | 是 | `gpio` `uart` `hart`；带 `instances:` 的即装配 |
| **`library`** | **否** | **否** | `hwcore` `amba` `bridge` |

**库包只贡献 BSV 源**：工具把它的 `bsv/` 加进编译搜索路径，仅此而已。它不进地址图、不算面积、不出现在装配层次里。

因此库包**不得**出现 `contract` / `params` / `features` / `area` / `instances`，也不得有 `regmap.yaml`——写了不是无害的冗余，是误导读者以为它能被例化。工具对此报错。

`instances[].of` 指向库包同样报错。

> 工艺数据（`pdk`）不写 `ip.yaml`。它既不是 IP 也不是源码库，是**数据**，走专门的读取路径进层叠的最底层。

---

## 二、`identity` —— 对齐 ECOS

```yaml
identity:
  uid: ip-000042              # ECOS 目录编号；未收录时省略
  slug: gpio-tapeout-bsv
  display_name: Configurable GPIO
  summary: Bus-neutral GPIO with a measured area price on every feature.
  summary_zh: 总线中立的通用 IO，每个特性都带实测面积价格。
  category: peripheral
  subcategories: [gpio]
  ip_family: gpio
  compatibility: []
  integration_profile: [apb4-slave]
  maturity: simulated         # 见 maturity.md
  links:
    repository: https://github.com/Tape-Out/gpio
  upstream: { ... }           # 仅收编第三方时出现
  tracking: { ... }
```

字段名与取值域跟随 ECOS，**我们不改名**，这样 IP 将来能直接进它们的目录。注意 ECOS 目录的准入门槛是 silicon-proven，所以对齐是为了**将来能进**，不是现在就能进。

---

## 三、`contract` —— 契约签名

```yaml
contract:
  version: 1                  # 契约单独定版，与包的 version 无关
  ctrl:
    shape: flat               # flat | server
    aw: 8
    dw: 32
  data:
    - { name: tx, dir: out, kind: get, width: 8 }
  irq:
    - { name: irq, kind: level }
  pins:
    - { name: gpio_in,  dir: in,  width: numPins }
    - { name: gpio_out, dir: out, width: numPins }
```

`shape` **是生成参数，不是包装层**。同一份逻辑按 `flat` 或 `server` 生成，两者代价差 2.55–3.19 倍；**规范不提供形态适配器**，因为实测适配比按目标形态重新生成贵 39.3%–152.2%。选形态的规则见 `contract.md`。

契约版本与包版本分开（D24）：第三方只需盯契约版本，包内部改实现不打扰他们。

---

## 四、`params` 与 `features` —— 旋钮

### 数值旋钮

```yaml
params:
  numPins:
    type: int
    default: 32
    range: [1, 64]
    desc: 引脚数
```

### 布尔与档位旋钮

```yaml
features:
  irq:
    type: bool
    default: true
    area: { fixed: 0, per: numPins, k: 37.23 }

  debounce:
    type: bool
    default: false
    depends: [bidir]                  # kconfig 的 depends on
    area: { fixed: 29.7, per: numPins, k: 30.41 }

  drive:
    type: choice                      # kconfig 的 choice
    values: [none, two-level, four-level]
    default: none
    area:
      none:       { fixed: 0, per: numPins, k: 0 }
      two-level:  { fixed: 0, per: numPins, k: 8.0 }
      four-level: { fixed: 0, per: numPins, k: 16.0 }
```

**旋钮分两类，求解时机不同：**

| 类 | 例 | 求解 |
|:--|:--|:--|
| **实例级** | 位宽、中断、去抖、驱动强度 | 每次例化独立求解，**可以互斥** |
| **芯片级** | 复位风格、时钟树、全局总线后端 | 全局唯一，由顶层装配定 |

之所以敢让旋钮互斥——cargo 不敢——是因为硬件**每次例化各自展开**，不像 crate 全图只编译一次取并集。同一颗 SoC 里 `gpio` 一处八针、一处三十二针，各生成各的。

### 约束

```yaml
constraints:
  - when: { debounce: true }
    then: { bidir: true }
    msg: 去抖作用在输入通路上，需要 bidir 打开
```

借 kconfig 的 `depends on`、`choice`、`range`；**不借 `select`**。`select` 能强开一个依赖并不满足的符号，是 kconfig 公认最烂的部分。这里的 `constraints` 在不满足时**报错并指出是哪条**，不静默改值。

---

## 五、`area` —— 面积价目表

```yaml
area:
  base:  { fixed: 37.6, per: numPins, k: 32.05 }
  model: linear-additive
  error: { bound: 0.07, sign: over }
  corner: { tool: ecc, pdk: ics55, freq_mhz: 100, measured: "2026-09-08" }
```

**每个特性存「固定项 + 每单位斜率」两个数，不按 2ⁿ 组合存表。** 实测依据：三特性八组合 × 两位宽，线性叠加的偏差恒在 −0.6% ~ −6.9%，**符号一律为负**（特性共享逻辑被优化器合并）。

`error.sign: over` 明写模型**恒为高估**，因此预算工具只会偏保守，不会承诺不了。

`corner` 记录这组数是在什么工具、什么工艺、什么频率下量的。换任何一项，数就作废。

---

## 六、`emit` —— 交付形态

照 Rust 的 `crate-type`：同一份源码，几种交付边界。

```yaml
emit:
  - kind: bsv                 # BSV 包，给 BSV 消费者
    package: GpioGen          # BSV 包名
    module: mkGpio            # 例化时调的模块
    config_type: GpioCfg      # 特性结构体的类型名
    interface: GpioIfc        # 接口类型名
  - kind: verilog-flat        # 扁平端口顶层，可独立综合与独立流片
    bus: apb4                 # 必选：扁平化必须选一种总线
```

`kind: bsv` 那四个字段**都是必填**：装配器要靠它们生成 `import` 与例化语句，光有 `kind` 生成不出东西。这一条是竖切逼出来的——规范先写漏了，写生成器时才发现。

`verilog-flat` 是三件事的组合：**契约 + 选一种总线 + 扁平化**。它必须带 `bus`，因为外人不讲我们的契约，只讲 APB4 或 AXI。

**扁平化不是免费的**：把综合边界推到契约层，对外线位实测从 475 涨到 762（+60%），多出来的全是方法变端口后的 `RDY`/`EN` 握手线。代价随 `contract.ctrl.shape` 变——`flat` 形态近乎恒等变换，`server` 形态要把两组握手都摊成端口。**这笔钱在配置时就该看得见**，所以它进价目表。

---

## 七、`deps` —— 依赖

照搬 cargo 的来源模型：

```yaml
deps:
  hwcore: "^1"
  amba:   { git: "https://github.com/Tape-Out/amba", rev: "a1b2c3d" }
  imsic:  { path: "../imsic" }

patch:
  hwcore: { path: "../hwcore-fork" }
```

解析结果固化进 `xirang.lock`。

**一条硬规则**：同一依赖**不得同时以 git submodule 与 `git:` 出现**。是 submodule 就写成 `path:`——`.gitmodules` 管「目录从哪来」，`xirang.lock` 管「解析到哪个目录」，两者管的是不同的事，同时用才会变成两个真相源。

---

## 八、`instances` —— 装配（有它即为 SoC）

```yaml
name: soc-mcu
bus: apb4                     # 全局默认后端，实例可覆盖

instances:
  - name: gpio0
    of: gpio
    with: { numPins: 32, irq: true, drive: two-level }
    addr: 0x1000_0000         # 省略则用该 IP 的 regmap.yaml 里的 base
    irq_to: plic0.src[3]

  - name: uart0
    of: uart
    with: { profile: ns16550 }
    addr: 0x1000_1000
    bus: axi4-lite            # 覆盖全局默认

  - name: plic0
    of: plic
    with: { mode: plic, sources: 32 }
```

**装配是递归的**：`soc-smp` 含两个 `hart`，每个 `hart` 含一个 `imsic`，三层用同一套语义。因此不存在单独的「SoC schema」——`ip.yaml` 有 `instances:` 就是装配，没有就是叶子 IP。

地址重叠在展开期报错，不留到仿真。

**装配包不得含自有 RTL。** 长出自有 RTL 就说明下层缺件，该补的是下层，不是把胶水塞进装配包。

---

## 九、层叠与来源追溯

一个旋钮的最终取值由六层叠出来，低到高：

1. BSV 类型默认（**结构由 BSV 类型权威，只有取值层叠**）
2. `pdk` 包提供的工艺事实
3. 本 `ip.yaml` 的默认
4. workspace 默认
5. 装配处 `instances[].with`
6. 命令行

**约束求解可以改写任何一层的取值**，但那是「被约束强制」，不是「被上层覆盖」——两者必须能区分，否则用户以为自己写错了。

工具必须能对任意一个旋钮回答五问：**最终值 · 谁定的（层/文件/行）· 被压掉的候选及其来源 · 是否被约束强制 · 这个取值花了多少面积**。

**生成钩子不得改写旋钮取值**，只能往下游添产物与声明。理由不是洁癖：钩子一旦能回改配置，上面那五问就答不了了。

---

## 十、本版明确不做

注册表（registry）· 并行 DAG 与增量构建 · 跨包的芯片级旋钮传播（cmake 那种 target 属性传递）· 版本求解的回溯（先只支持精确锁）· `instances` 的条件展开（`for` / `if`）。

这些每一条都有真实用途，但 `gpio` + `soc-mcu` 这条竖切一条都用不上。**等真有 IP 需要，再按各自来源的原名加进来。**
