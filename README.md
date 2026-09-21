# xrspec

Tape-Out 的 IP 元数据与寄存器规范。

![spec](https://img.shields.io/badge/spec-v0.2.3-blue) ![license](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0%20OR%20MulanPSL--2.0-blue)

| 文档 | 写了什么 |
| :--: | :--: |
| [`ip.md`](ip.md) | 身份 · 契约签名 · 旋钮与约束 · 面积价目表 · 交付形态 · 依赖 · 装配 |
| [`regmap.md`](regmap.md) | 寄存器与字段语义 · feature 门控 · 存储块 |
| [`contract.md`](contract.md) | 两种契约形态 · 选用规则 · 编写规范 |
| [`maturity.md`](maturity.md) | 五档成熟度，及与 ECOS 目录词汇的映射 |

语义取自 SystemRDL 2.0、kconfig、cargo 与 ECOS，只在留白处引申。

## 版本

`ip.yaml` 的 `spec:` 字段声明遵循哪一版。当前 v0.2.3，语义破坏进 minor，加字段进 patch。

## 许可证

任选其一：[MIT](LICENSE-MIT) · [Apache 2.0](LICENSE-APACHE) · [木兰宽松许可证 第2版](LICENSE-MULAN)
