# 成熟度规范 v0.1

每个包在 `ip.yaml` 的 `identity.maturity` 声明一档，并在 README 挂对应徽标。

**仓库存在与否不由某一种工艺决定。** 野心大的项目照样立项，用等级如实表达它现在到了哪一步——这比「等做完再建仓」诚实，也比「建了仓假装已完成」诚实。

---

## 一、五档

| 档 | 徽标色 | 判据 |
|:--|:--|:--|
| `planned` | 灰 | 有 `ip.yaml` 与契约签名，无实现 |
| `simulated` | 黄 | bsc 类型检查过 · `-show-schedule` 无多余守卫无调度环 · `bsc -sim` 回归绿 |
| `fpga-proven` | 橙 | 上板跑通，有可复现的比特流与测试记录 |
| `asic-ready` | 绿 | `ecc syn_sta` 过，面积与时序有数并回填进 `ip.yaml` |
| `silicon-proven` | 亮绿 | 硅回来点亮，有实测记录 |

**升档要有证据。** 证据（综合日志、面积基线、波形、上板记录）**另开分支存放**，不进主分支——克隆主分支的人不必拖几百兆的证据文件，而证据确实留着、可追溯。主分支 README 用相对路径 `../../tree/<分支>` 把这些分支指出来。

---

## 二、与 ECOS 目录词汇的映射

ECOS Factory 的 IP 目录用另一套五档。两边不同名，工具需要存映射，否则将来提交要手工翻译。

| 我们 | ECOS | 说明 |
|:--|:--|:--|
| `planned` | `In development` | 立项，无实现或实现未完 |
| `simulated` | `Prototype` | 有实现，仿真过 |
| `fpga-proven` | `In evaluation` | 有硬件证据但未签核 |
| `asic-ready` | `In verification` | 后端流程走通 |
| `silicon-proven` | `Silicon-proven` | 同名同义 |

**注意 ECOS 目录的准入门槛是 silicon-proven。** 所以对齐它们的元数据是为了**将来能进**，不是现在就能进。

---

## 三、徽标写法

```markdown
![maturity](https://img.shields.io/badge/maturity-planned-lightgrey)
![license](https://img.shields.io/badge/license-MulanPSL--2.0-blue)
```

颜色：`planned` 灰 · `simulated` 黄 · `fpga-proven` 橙 · `asic-ready` 绿 · `silicon-proven` 亮绿。
