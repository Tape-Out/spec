# spec

The metadata and register specifications every IP in the Tape-Out library follows.

![maturity](https://img.shields.io/badge/spec-v0.1-blue) ![license](https://img.shields.io/badge/license-Apache--2.0-blue)

This repository holds the standard, not an implementation of it.
[`loom`](https://github.com/Tape-Out/loom) is one implementation; anyone is free to write
another. IP repositories depend on this specification, never on the tooling — which is why
it lives on its own rather than inside the tool.

| 文档 | 内容 |
| :--: | :-- |
| [`ip.md`](ip.md) | `ip.yaml`：身份 · 契约签名 · 旋钮与约束 · 面积价目表 · 交付形态 · 依赖 · 装配 |
| [`regmap.md`](regmap.md) | `regmap.yaml`：寄存器与字段语义 · feature 门控 · 存储块 |
| [`contract.md`](contract.md) | 两种契约形态、选用规则、编写规范 |
| [`maturity.md`](maturity.md) | 五档成熟度及与 ECOS 目录词汇的映射 |

## 它借了谁

不重造已有的语义，只在别人留白的地方引申。

| 层 | 语义来自 | 我们添的 |
| :-- | :-- | :-- |
| 寄存器字段 | **SystemRDL 2.0**（属性名原样沿用） | feature 门控 · 存储块与宏选型 |
| 旋钮约束 | **kconfig** 的 `choice` / `range` / `depends on` | 会报错而不是硬开的 `constraints`（不借 `select`） |
| 依赖模型 | **cargo** 的 `path` / `git`+`rev` / `patch` / lock | 与 git submodule 的边界规则 |
| 元数据字段 | **ECOS** 的 `ip.yaml` | 契约签名 · 面积价目表 · 参数范围 · 依赖锁 |

范式取 OpenAPI 3 与 JSON Schema 的关系：OpenAPI 不重造一套校验语言，它**就是** JSON Schema 再加上自己需要的那部分。

## 版本

规范单独定版，与任何实现的版本无关。`ip.yaml` 里的 `spec:` 字段声明遵循哪一版。

当前 **v0.1**。语义有破坏性改动时进 minor，加字段进 patch。

## License

Apache License 2.0.
