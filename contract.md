# 契约规范 v0.1

IP 不含任何总线代码。它暴露总线中立的契约，绑定发生在集成时。这样加一种总线是加**一个适配器**，而不是在每个 IP 里加一层包装——`N + M` 而不是 `N × M`。

契约只有**两种形态**，不存在第三种。

---

## 一、形态由「谁消费」决定

这不是品味问题，是编译期的硬约束。

| 消费者 | 形态 | 相对代价 |
|:--|:--|--:|
| **引脚绑定器**（方法驱动） | 扁平组合：`put` / `ready` / `rdata` / `err` | **1×** |
| **规则驱动的主设备**（DMA、处理器核） | `Server` / `Client` + `Get` / `Put` | **2.55–3.19×** |

选错不是「贵一点」，是**编译不过**：组合形态被规则消费会成调度环，规则永不触发（bsc 报 G0021）。反过来，用 `Server` 去接一个简单寄存器外设，白付两倍半。

`Server` 的 FIFO 选型再差 25%：`mkFIFOF1` 最省，默认的 `mkFIFOF`（深 2）最贵。

---

## 二、扁平组合形态

```bsv
interface Flat#(numeric type aw, numeric type dw);
  (* always_ready *) method Action   put(Req#(aw, dw) r, Bool valid);
  (* always_ready *) method Bool     ready;      // 低电平即等待态
  (* always_ready *) method Bit#(dw) rdata;
  (* always_ready *) method Bool     err;
endinterface
```

**单次握手加 `ready` 反压**，即 PULP `REG_BUS` 与 APB `pready` 的形状。它同时满足零等待（`ready` 恒真）与等待态（第三方从设备拉低 `ready`），且无守卫环。

**不要拆成两个带守卫的方法**（`req` 一个、`rsp` 一个）。零等待绑定器要在同一拍先 `req` 再 `rsp`，守卫会成环，bsc 报 G0035，编译不过。

---

## 三、`Server` 形态

```bsv
typedef Server#(Req, Rsp) Ctrl;
```

直接用 BSV 标准库的 `ClientServer`，请求与响应各自带握手，规则之间自然解耦。**不发明新抽象**——`Server`/`Client`/`Get`/`Put`/`mkConnection` 标准库里本来就有。

---

## 四、角色特化用零宽类型

```bsv
typedef Waist#(aw, dw, 0, 0) RegSub;   // 突发长度与 ID 零宽
typedef Waist#(aw, dw, 8, 4) MemSub;   // 真实突发与 ID
```

`Bit#(0)` 在 BSV 中合法且占 0 位。**用不到的字段在硬件上不存在——不是被优化掉，是根本没生成。**

边界条件：零宽或静态接常量免费；**真的实现通用语义要付钱**。让寄存器外设去走突发地址，实测 +21.6%。所以角色决定字段存不存在，存在了就得实现，不实现就该零宽。

---

## 五、规范不提供形态适配器

实测同一份寄存器行为：消费者要 `Server` 时「扁平 + 适配器」比原生 `Server` 贵 **39.3%**；消费者要扁平时「`Server` + 适配器」比原生扁平贵 **152.2%**。绝对代价 1,367–1,934 µm²，量级相当于一整个小外设，而且**每个适配点各付一次**。

所以 `shape` 是 `ip.yaml` 里的**生成参数**，不是可以事后套上的包装层。

---

## 六、编写规范（撞出来的硬约束）

1. `(* synthesize *)` **只放独立交付单元最外层**。标在 IP 核心模块上会让方法参数变成必须共享的固定端口，平白引入规则冲突，且对外线位 +60%。
2. `always_ready` 接口内用 **`mkGFIFOF#(ugenq, ugdeq)` 逐端口去守卫**，不要一刀切 `mkUG*`——那会让消费者 deq 空队列。
3. 请求路径与响应路径**分在不同方法或不同规则**。
4. 打破调度环用 **`mkConfigReg`**（读写无调度约束），不靠重构绕。凡「方法写寄存器 + 同拍组合读回」必成 G0021。
5. **一条规则里不能既 `wset` 又 `wget` 同一个 `RWire`**（G0004）。拆成两条，bsc 才能定序。拆完通常立刻撞第 4 条。
6. **CReg 的两个端口不能在同一条规则里用**（G0004）。规则用端口 0、总线方法用端口 1，且端口顺序必须与调度顺序一致。
7. 改协议或加规则后**必看 `bsc -show-schedule`**。它把只能靠仿真挂死才发现的问题变成编译期可读文本。
8. `$display` 字符串**不得含非 ASCII**，否则编译器内部错误。注释里可以。

---

## 七、库里已有、不要自己写

`mkConnection`（连接）· `CompletionBuffer`（乱序转保序）· `Gearbox`（位宽转换）· `TieOff`（静态退化接线）· `Assert` + `Probe`（断言与波形探针）· `Clocks`（跨时钟域）· `Arbiter` · `PAClib`（流水线组合子全套）· bsc-contrib 的 `AMBA_Fabrics`（AXI4 fabric / Deburster / Widener / ClockCrossers）与 `AMBA_TLM3`。
