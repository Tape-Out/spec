# `ip.yaml` 规范 v0.2

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

```yaml
name: hwcore
kind: library
lang: bsv          # bsv | bh | verilog | vhdl | sv | chisel | spinal，省略即 bsv
```

**库包只贡献源码**：工具把它的源目录加进编译搜索路径，仅此而已。它不进地址图、不算面积、不出现在装配层次里。

**语言不限于 BSV。** 规范对源语言是开放的——Verilog、VHDL、SystemVerilog、Chisel、SpinalHDL 都可以，用 `lang:` 声明，工具按语言选对应的编译前端。**当前实现只支持 BSV 与 BH**，其余语言的前端等到真有包需要时再接。

我们自己写的 IP **在 BSV 与 BH 之间按 IP 择优**：看哪一种更适合这个 IP 的功能与特性，哪一种更易读、更好组件化与参数化。两者同一个编译器、同一套语义，只是语法不同，可以混用。

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
    shape: flat               # flat | server | none
    aw: 8
    dw: 32
  data:
    - { name: tx, dir: out, kind: get, width: 8 }
  irq:
    - { name: irq,  kind: level }
    - { name: irqs, kind: level, width: harts }   # 多根：宽度可写旋钮名
  pins:
    - { name: gpio_in,  dir: in,  width: numPins }
    - { name: gpio_out, dir: out, width: numPins }
```

`shape` **是生成参数，不是包装层**。同一份逻辑按 `flat` 或 `server` 生成，两者代价差 2.55–3.19 倍；**规范不提供形态适配器**，因为实测适配比按目标形态重新生成贵 39.3%–152.2%。选形态的规则见 `contract.md`。

`shape: none` 说的是「这个 IP 不是总线从设备」——核就是。它的 CSR 空间是自己用的，引到顶层会让规则与外部方法抢同一个端口，而 bsc 的处置是把规则整条丢掉、只给一句告警。声明了 `none` 之后：中立顶层不出控制口，扁平化直接拒绝，装配也不把它放进地址图（它的 `regmap` 的 `base` 描述的是 CSR 空间，与片上地址无关）。

中断线的 `width` 让「每核一根」「每通道一根」表达得出来。不写就是单根。

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

价目表有**两种形态**，用在不同的旋钮上，各有各的保证：

| 形态 | 写法 | 用于 | 保证 |
|:--|:--|:--|:--|
| **仿射** | `{fixed, per, k}` | **特性**叠加 | 恒为高估 |
| **实测点** | `{points, per, margin}` | **参数**缩放 | 恒为高估，靠余量 |

```yaml
area:
  base:
    per: fifoDepth
    points: {1: 1689.24, 2: 1897.56, 4: 2532.60, 8: 3375.96, 16: 5000.52, 32: 8062.60}
    margin: 0.04          # 插值后上抬，由实测残差定
  model: measured-points
  error: { bound: 0.06, sign: over }
```

**为什么参数不能用仿射**：参数曲线可能有**结构断点**——`uart` 的 FIFO 在深度 3 以上换了实现，一条直线怎么拟都会把中间低估 4.2%。

**为什么存了点还要余量**：曲线是凹的，**弦恒在曲线之下**，所以点间线性插值系统性偏低（实测四个未测深度，三个低估，最多 3.54%）。加余量之后四个全部高估，最紧的只多 0.32%。

**`margin` 必须由实测定**。没测过就别写——写了是假的保证，比没有保证更危险。

**每个特性存「固定项 + 每单位斜率」两个数，不按 2ⁿ 组合存表。** 实测依据：三特性八组合 × 两位宽，线性叠加的偏差恒在 −0.6% ~ −6.9%，**符号一律为负**（特性共享逻辑被优化器合并）。

`error.sign: over` 明写模型**恒为高估**，因此预算工具只会偏保守，不会承诺不了。

`corner` 记录这组数是在什么工具、什么工艺、什么频率下量的。换任何一项，数就作废。它还必须带 `gen_digest`——**生成产物的摘要**。手工维护版本号两头不讨好：忘了升是漏报，升了没改输出是误报。对产物取摘要两种错都没有。

### 实测过的配置直接查表

扫描本来就要把整张网格量一遍，那就把这些数存下来：

```yaml
area:
  measured:
    - { at: { numPins: 8, irq: true, bidir: true, debounce: false }, um2: 646.52 }
    - { at: { numPins: 8, irq: false, bidir: false, debounce: false }, um2: 211.12 }
```

落在表里的配置**精确报出，不走模型也不抬余量**；表外才走模型。`gpio` 因此从 −14.58% 收到 +0.34%。

### 价目表量的是**中立顶层**，不是包装层

包装层含一个总线绑定器，而装配里整颗芯片只有一个。按包装层计价等于每个实例都多算一份绑定器，译码开销会算出**负数**。所以 `area` 描述的是 `build --neutral` 综合的那一层。

绑定器的钱记在实现它的包上（`amba`），交换网的钱记在 `hwcore`，**按包记一次**，不随实例数翻倍。库包因此可以有 `area`，但只能是定值——它没有旋钮可依。

### 叶子是上界，装配是估计

两层的承诺不同，写清楚免得误用：

| 层 | 承诺 | 怎么保证 |
|:--|:--|:--|
| 叶子 IP | **恒不低估** | 全部合法特性组合 × 多个参数值的证伪网格 |
| 装配 | **带双侧误差的估计** | 实测比例在 0.94~1.29 之间，取 1.15、界 ±15% |

装配的比例写在 `hwcore` 的 `area.assembly.factor`。为什么不能也做成上界：独立综合时端口必须保住、装配里综合器能并掉，而嵌套的握手又比独立边界贵，两股力方向相反，比例取决于配置的组合。硬要做上界就得留 30% 余量，那样的数字没法拿来比较配置。

**装配里不叠叶子的上界余量**——两道保守叠起来会到 26%。

### 证伪网格必须覆盖全部合法组合

只测「单特性」与「全开」会漏掉两两组合。`gpio` 的 `irq+bidir` 在 8 针处欠估 **9.57%**、32 针处又高估 4.68%——**符号两边都有**。所以「叠加恒为高估」那条只在包装层成立，换到 IP 本体就不成立。

### 关掉一个特性，面积必须真的掉下来

只挡逻辑不挡例化的开关一分钱都不省。`hart` 的 M 扩展开关原先只让译码器不产生乘法指令，模块照样例化——实测两侧只差 **98 µm²**，而那个单元本身要六千。改成按开关选一个退化实现之后，真实代价是 7,452。

**这是一条可执行的判据**：写完一个特性，量一次开与关，掉不下来就是没关掉。

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
    ctrl: regs                # 契约子接口叫什么，默认 regs
    pins:                     # 透传的子接口，装配与包装层照它生成
      - { name: pins, type: GpioPins, targs: [numPins] }
  - kind: verilog-flat        # 扁平端口顶层，可独立综合与独立流片
    bus: apb4                 # 必选：扁平化必须选一种总线
```

`pins` 里 `type` 为 `RegManager` 的那些是**发起口**（核的取指与访存、DMA 的搬运）。它们不引到装配顶层，而是接进交换网与外部总线一起排队——见下面「装配」一节。其余的原样透传到顶层。

`targs` 写旋钮名或数字，求解后取值。

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

解析结果固化进 `xirang.lock`，而**锁跟着装配走，不跟着工作区走**：装配是交付物、锁是它的一部分，所以它住在那个仓里。

**叶子 IP 不带锁**。它是被别人依赖的库，钉死反而会跟使用它的装配冲突——照搬 cargo 对二进制与库的区分。`xirang lint` 据此分别对待：装配没锁是错，叶子没锁只是一句说明。

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
