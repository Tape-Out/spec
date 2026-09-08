# 契约规范 v0.2

IP 不含任何总线代码。它暴露总线中立的契约，绑定发生在集成时。这样加一种总线是加**一个适配器**，而不是在每个 IP 里加一层包装——`N + M` 而不是 `N × M`。

契约分两侧：**被访问**的一侧与**发起访问**的一侧。绝大多数外设只有前者；核、DMA、缓存两者都有。

---

## 一、被访问的一侧：`RegIf`

```bsv
typedef struct {
  Bit#(aw)           addr;
  Bool               write;
  Bit#(dw)           wdata;
  Bit#(TDiv#(dw, 8)) wstrb;
} RegReq#(numeric type aw, numeric type dw) deriving (Bits, FShow);

typedef struct {
  Bit#(dw) rdata;
  Bool     err;
} RegRsp#(numeric type dw) deriving (Bits, FShow);

interface RegIf#(numeric type aw, numeric type dw);
  method ActionValue#(RegRsp#(dw)) access(RegReq#(aw, dw) r);
endinterface
```

**一次访问一个方法，同拍给结果。** 写与读走同一个方法，因为总线本来就是一次事务做一件事，拆成两个带守卫的方法会成守卫环——零等待的绑定器要在同一拍先发请求再取响应，bsc 报 G0035，编译不过。

字节选通 `wstrb` 由 `applyStrb` 统一处理，绑定器与 IP 共用同一份，避免各写一份写错。

### 形态由「谁消费」决定

| 消费者 | 形态 | 相对代价 |
|:--|:--|--:|
| **引脚绑定器**（方法驱动） | `RegIf`，扁平的 `always_ready` 方法 | **1×** |
| **规则驱动的主设备** | `Server` / `Client` + `Get` / `Put` | **2.55–3.19×** |

选错不只是贵：组合形态被规则消费会成调度环，规则永不触发（bsc 报 G0021）。反过来，用 `Server` 去接一个简单寄存器外设，白付两倍半。

`Server` 的 FIFO 选型再差 25%：`mkFIFOF1` 最省，默认的 `mkFIFOF`（深 2）最贵。

---

## 二、发起访问的一侧：`RegManager`

```bsv
interface RegManager#(numeric type aw, numeric type dw);
  (* always_ready *) method Bool                valid;
  (* always_ready *) method RegReq#(aw, dw)     req;
  (* always_ready, always_enabled *) method Action ready(Bool r);
  (* always_ready, always_enabled *) method Action resp(Bool v, RegRsp#(dw) x);
endinterface
```

**举手—授予**：发起方举着 `valid` 不放，直到 `ready` 回来；`ready` 与响应同拍到。授予之后到 `valid` 落下之前那一段，仲裁不再看它，免得同一个请求做两遍。

形态与被访问的一侧一样是扁平的——实测 `Server` 在小模块上贵 39.3%，而这里同样不需要它的排队语义。核的取指与访存各一口，DMA 的搬运一口，都用它。

`ready` 与 `resp` 是 `always_enabled` 的**输入**：集成方每拍都得给，没轮到就给 `False`。这条约束会传染到装配的写法——见 `ip.md` 的「装配」一节。

---

## 三、没有控制口：`shape: none`

核不是总线从设备。它的 CSR 空间是自己用的，引到顶层会让「规则用」与「外面用」抢同一个方法——**bsc 的处置是把规则整条丢掉**，只给一句告警而不是报错。

```yaml
contract:
  ctrl: { shape: none, aw: 12, dw: 32 }
```

这样声明之后：中立顶层不出这个子接口，扁平化直接拒绝（它不是总线从设备），装配也不把它放进地址图。

---

## 四、中断线可以有多根

```yaml
contract:
  irq:
    - { name: irqs, kind: level, width: harts }
```

`width` 可以写旋钮名，求解后取值。每核一根（`mbox`）、每通道一根、三组各按核宽（`aclint`）都靠它表达。装配里多根合成一根送译码，完整向量另引到顶层交给中断控制器。

不写 `width` 就是单根，方法类型是 `Bool`。

---

## 五、角色特化用零宽类型

```bsv
typedef Waist#(aw, dw, 0, 0) RegSub;   // 突发长度与 ID 零宽
typedef Waist#(aw, dw, 8, 4) MemSub;   // 真实突发与 ID
```

`Bit#(0)` 在 BSV 中合法且占 0 位。**用不到的字段在硬件上不存在——不是被优化掉，是根本没生成。**

边界条件：零宽或静态接常量免费；**真的实现通用语义要付钱**。让寄存器外设去走突发地址，实测 +21.6%。所以角色决定字段存不存在，存在了就得实现，不实现就该零宽。
