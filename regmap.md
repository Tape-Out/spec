# `regmap.yaml` 规范 v0.1

寄存器表。语义内核沿用 **SystemRDL 2.0，属性名原样不改**；YAML 只是载体。引申的部分是 feature 门控与存储块。范式取 OpenAPI 3 与 JSON Schema 的关系——不重造语义，只添加。

本版只覆盖 `gpio` 竖切用得到的属性子集。**够用即止，用不到的一律不实现**；等有 IP 真的需要，再按 SystemRDL 的原名加进来。

---

## 一、骨架

```yaml
ip: gpio
contract: { aw: 8, dw: 32 }     # 须与 ip.yaml 的 contract.ctrl 一致，工具交叉校验
base: 0x10000000                # 默认基地址；装配处不写 addr 就取它
size: 0x100
params: [numPins]               # 引用 ip.yaml 的参数，供位宽使用

regs: [ ... ]
mems: [ ... ]                   # 可选
```

`base` 放这里而不是装配处，是为了让 IP 自带一个能用的默认地址；装配只在冲突或有特殊布局时覆盖。地址重叠在展开期报错，不留到仿真。

---

## 二、字段属性：SystemRDL 2.0 子集

名字与 SystemRDL 完全一致。

| 属性 | 取值 | 含义 |
|:--|:--|:--|
| `sw` | `rw` · `r` · `w` | 软件侧访问方向 |
| `hw` | `rw` · `r` · `w` · `na` | 硬件侧访问方向，决定生成的接口方向 |
| `onwrite` | `woclr` | 软件写 1 清零（通称 W1C） |
| `hwset` | `true` | 硬件可置位，生成一个置位输入端口 |
| `stickybit` | `true` | 置位后保持，直到软件清除 |
| `reset` | 整数 | 复位值。**省略即无复位**——带复位触发器贵 52%，所以不默认给 |

**本版不实现**：`onread` 的 `rclr`/`rset` · `we`/`wel` · `hwclr` · `swmod`/`swacc` · `counter` · `singlepulse` · 别名寄存器 · 寄存器数组 · `intr`/`enable` 的标准中断聚合 · `external`。

> 中断聚合为何不进本版：`gpio` 的中断是「边沿 ∧ 使能」才置位，使能作用在**置位端**；SystemRDL 的 `intr`+`enable` 默认作用在**输出端**。两者行为不同。与其造一个带开关的四不像，不如让寄存器表只管存储与总线语义，把「什么时候置位」交给 IP 自己的规则去驱动 `hwset` 端口。这也是寄存器生成器的本分。

---

## 三、位宽

单字段占满用 `width`，多字段用 `bits` 显式定位。两者都可以是对 `params` 的引用：

```yaml
fields:
  - { name: val,  width: numPins, sw: rw, hw: r, reset: 0 }
  - { name: mode, bits: "17:16",  sw: rw, hw: r }
```

**本版只接受裸参数名或整数**，不支持表达式——表达式要译成 BSV 的 `TAdd`/`TMul` 类型级算术，等真有 IP 需要再加。越界或与 `contract.dw` 冲突一律展开期报错。

---

## 四、引申之一：`feature` 门控

寄存器或字段可挂 `feature:`。该 feature 关闭时，**这条在生成的 BSV 里根本不存在**（不是被优化掉，是压根没生成），对它的读写落到 `err`。

```yaml
- name: dir
  offset: 0x08
  feature: bidir
  fields: [ { name: val, width: numPins, sw: rw, hw: r, reset: 0 } ]
```

工具校验：regmap 里出现的每个 feature 名都必须在 `ip.yaml` 的 `features` 里声明过。

> 字段名是 `offset` 而不是 `off`：**YAML 1.1 里 `off` / `on` / `yes` / `no` 是布尔字面量**，写 `off:` 会被解析成 `False`。规范不该逼用户加引号。

---

## 五、引申之二：存储块

大块存储单列 `mems`，不写成寄存器：

```yaml
mems:
  - name: buf
    offset: 0x1000
    depth: 256
    width: 32
    sw: rw
    impl: auto        # auto | flops | macro
```

`impl: auto` 时工具按面积判据自动选：总位数超过阈值走 SRAM 宏，否则用触发器。阈值取自 `pdk` 包的工艺数据。**选择结果写进构建产物的 `resolved.yaml`**，供面积核算取用——regmap 只声明意图，不写死实现。

**例外：需要并行比对的结构不适用这条判据。** TLB、cache 标签阵列、MAC 学习表是 CAM 行为，单端口 SRAM 做不了，只能用触发器加比较器。这类结构声明 `impl: flops` 并注明理由。

---

## 六、生成物约定

一份 `regmap.yaml` 产出四样，形状定死，否则无从对拍：

| 产物 | 路径 | 形状 |
|:--|:--|:--|
| BSV 寄存器文件 | `bsv/<Ip>Regs.bsv` | 模块 `mk<Ip>Regs`，实现 `ip.yaml` 里 `contract.ctrl.shape` 指定的形态 |
| C 头 | `sw/include/<ip>.h` | 偏移宏 + 每字段的 `_SHIFT` / `_MASK` |
| 文档 | `docs/regmap.md` | 寄存器表 |
| 地址元数据 | `build/regmap.json` | 装配的地址分配与重叠检查读它 |

**BSV 侧接口命名规则**（对拍成不成立全看这条）：

- `hw: r` 或 `rw` 的字段 → 输出方法 `<reg>_<field>`
- `hw: w` 或 `rw` 的字段 → 输入方法 `<reg>_<field>_in`
- `hwset: true` → 额外输入方法 `<reg>_<field>_set`
- 字段名为 `val` 且是该寄存器唯一字段时，**省略 `_val` 后缀**，直接叫 `<reg>`

---

## 七、`gpio` 实例

```yaml
ip: gpio
contract: { aw: 8, dw: 32 }
base: 0x10000000
size: 0x100
params: [numPins]

regs:
  - name: dout
    offset: 0x00
    desc: output data
    fields: [ { name: val, width: numPins, sw: rw, hw: r, reset: 0 } ]

  - name: din
    offset: 0x04
    desc: input sample
    fields: [ { name: val, width: numPins, sw: r, hw: rw, reset: 0 } ]

  - name: dir
    offset: 0x08
    desc: direction, 1 drives out
    feature: bidir
    fields: [ { name: val, width: numPins, sw: rw, hw: r, reset: 0 } ]

  - name: ien
    offset: 0x0C
    desc: interrupt enable
    feature: irq
    fields: [ { name: val, width: numPins, sw: rw, hw: r, reset: 0 } ]

  - name: ista
    offset: 0x10
    desc: interrupt status, write one to clear
    feature: irq
    fields:
      - name: val
        width: numPins
        sw: rw
        onwrite: woclr
        hw: r
        hwset: true
        stickybit: true
        reset: 0
```

生成的 `mkGpioRegs` 对外给 `dout` · `din` · `din_in` · `dir` · `ien` · `ista` · `ista_set`，加契约那一面。IP 自己只剩三件事：采样引脚驱动 `din_in`、跑边沿检测驱动 `ista_set`、把 `dout`/`dir` 接到引脚。

> `din` 是 `hw: rw` 而非 `hw: w`：硬件既写它（引脚采样）**也读它**（边沿检测要上一拍的值）。写成 `hw: w` 会逼 IP 自己再存一份，白多一组触发器。

---

## 八、验收判据

生成件与手写件对拍，分四条。**别拿比工具还紧的尺子量**——实测 `ecc` 对语义等价、只是写法不同的 RTL，面积就能差 +3.07%（三行交换律改写即可）。

| # | 判据 |
|:--:|:--|
| **V1a** | **时序面积、`reg` 声明数、`always` 块数逐配置完全相同**。寄存器是 regmap 真正决定的东西，必须精确复现 |
| **V1b** | 组合面积偏差不超出**同轮等价改写对照带**外 2 个百分点。对照带每轮现测，不写死常数 |
| **V1c** | 行为逐拍一致：同一串激励下总线读数、错误、就绪、引脚、中断线全程相同 |
| **V1d** | `bsc -show-schedule` 告警谱与手写版相同 |

`gpio` 首次对拍：V1a 20/20 逐位一致 · V1b 边缘通过 · V1c 5 配置 × 2 万拍零失配 · V1d 两边同为 8 条 G0023。
