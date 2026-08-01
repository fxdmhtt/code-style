---
name: rust-development
description: 按用户个人的 Rust 开发风格写作、修改、重构或审查 Rust 代码。任何涉及 Rust 代码的任务都应使用。
---

# Rust 个人开发风格

默认遵循 Rust 标准库、Rust API Guidelines、Effective Rust、官方与社区通行实践。

## 总纲

- 面向约束编程：每一处代码都服从明确的广义约束（类型系统、所有权、需求、平台），并让结构去满足它。这是第一原则。
- 人类逻辑即代码结构，尽量一一映射；每一行代码都有明确的目的，拒绝为工程问题妥协，在此前提下追求最小代码。代码须干净、优雅。
- 抽象的依据是逻辑必然：两段代码因同一需求而必然一致才合并，仅是巧合雷同则复制。合并后若起不出一个准确干净的名字，通常就不该合并。
- 交付前对照本文对抗式审查，逐条核对直至全部满足；不问是否实现，但问应不应该。

## 工程基线

- 积极采用 Rust 新特性。工具链用 nightly，edition 2024，用 rust-toolchain.toml 锁定版本。
- 依赖选型：首选 std/core；优先使用能让代码更优雅的成熟库。
- 外部库优先选生态内的事实标准，如 thiserror、itertools、serde、bumpalo、num-traits 等。
- lint 按作用域就近声明：单项用 #[allow]，整文件用文件级 #![allow]，整工程用 Cargo.toml 的 [lints]。
- 交付前跑 clippy，清空本次改动文件的警告。
- 人类不需阅读或需保持排版（对齐或折叠）的成块数据，用 #[rustfmt::skip] 免于格式化。
- 条件编译门控尽量解耦：每个 feature 独立门控，仅当它们必须同时成立时才用 all() 合并。

## 模块与可见性

- 最小可见性：默认私有，被跨模块用到才逐级放宽；pub 只给 crate 对外的 Rust API；FFI 导出项照此判定，与它是否导出符号无关。
- use 按四段分组、段间空行：std/core → 外部 crate → crate:: → super::。
- glob 只允许三种：prelude::* 、再导出的 pub use ...::* 、测试里的 super::* ；其余逐项写出。
- prelude 只集中对外最常用、几乎必用的导出项。
- mod 声明集中写在 use 之后，mod tests 除外。
- 目录模块的入口写成 foo/mod.rs。
- 宏按可见范围导出：只在本模块内使用的不导出，跨模块用 pub(crate) use，对外 API 用 #[macro_export]。
- 宏引用的项一律使用绝对路径：macro_rules! 用 $crate::，其余带前导 ::。
- 只有 proc-macro 才单独开 crate：foo 与 foo-derive 成对，宏由 foo 再导出，使用者只依赖 foo。

## 类型设计

- 内部类型尽可能加全官方 derive，对外 API 按实际需要添加。
- derive 顺序为 Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default，第三方排最后；thiserror 的 Error 排最前；repr(C) 写在 derive 之前。
- 只在非泛型的 C FFI 边界成套出现：类型加 repr(C)，函数用 extern "C" 并加 unsafe(no_mangle)。
- 函数优先写成 const fn。
- 能用官方 trait 表达的行为，就直接 impl 而不另造同义方法。
- 属于基础设施的代码优先写成泛型，约束只写真正依赖的假设。
- 约束写在真正依赖它的那一项上：trait 共有的写在 trait 上，fn 独有的写在该 fn 上；简单的内联，复杂的用 where。
- 错误类型用 thiserror 定义，优先用 ? 转换。
- 确定单线程时，全局可变状态可用 Cell 语义的 newtype 承载。
- 区间用新的 core::range::Range，不用 core::ops::Range。

## 函数式表达

- 代码按块组织，一块完成一个原子任务，以空行或缩进为界。
- 表达式优先于语句：以求值代替赋值。
- 一个代码块优先写成一条链，末端只留一个绑定，必要时把逻辑收进闭包；中间结果有独立意义才断链命名。
- 分支要产出值时用 match 或 if else 表达式，不写连续 return。
- 提前返回用来排除不参与主线的情形，或在结果已定时短路。
- 优先无副作用函数，副作用集中到 for 与显式写入点；for_each 只作长链收尾。
- mut 只给真需原地修改的变量；累积结果用 collect 与 fold。
- 能用适配器组合表达的不手写循环。
- 不用递归代替循环。
- 优先惰性求值，到边界才落地：返回 impl Iterator 而非 Vec，区间保持 Range，同一条链消费两次用 tee 分流。
- 分支迭代器类型用 either::Either 统一。
- 需要失败原因的用 Result，只是可能没有值、无需原因的用 Option；try_ 开头的函数，返回值一律用 Result。
- Option 与 Result 用 map、and_then、ok_or、unwrap_or 串接，不嵌套 if let 与 match。
- 取值优先用模式解构。
- 链上用 .into() 或 .try_into()；传给适配器时优先用目标类型的 from 或 try_from，不包一层闭包。
- 能推断的类型不写标注，不承载逻辑的位置优先写 _：类型、泛型参数、生命周期、未用的绑定与成员。

## 断言与中止

- 断言用来证明假设与约束成立：违反即 panic，消息写明本该成立的前提。
- 正常运行中本就会出现的情形交给错误处理，此处不该出现的才交给断言；外部输入的非法值归错误，结果的性质归测试，都不是断言。
- 前提能由类型排除的就不写断言，改用 newtype、NonZero、NonNull 等类型手段。
- 断言写在它所证明的假设必须成立之处：入口前提在函数首行，语句或代码块的前提紧贴其前，循环不变量在循环内。
- 优先在编译期用 const 断言判定，就近写成 const { assert!(…) }；无法内联时才用 const _: () = assert!(…)。
- 条件本身是相等、不等或模式匹配的，分别用 assert_eq!、assert_ne!、assert_matches!；assert! 只给这三者表达不了的条件。
- 默认用 debug_ 版本；仅当违约只在罕见输入下出现、或它是 unsafe 的前提时，才用不带 debug_ 的版本保留到 release。
- 类型表达不了的不可能分支用 unreachable!()，消息写明它为何不可达。
- 计划实现的占位用 todo!()，有意不实现或不适用的用 unimplemented!()，不可恢复且不宜用 Result/断言表达的中止用 panic!() 并附消息。
- 在场约束直接推出不会 panic 的用 unwrap；需一句话说清前提的用 expect，消息写 precondition。

## 注释与文档

- 文档面向使用方，用 rustdoc 的标准小节，不写实现细节与设计意图；注释面向实现方，不解释代码怎么做，只解释背后的决策、约束与意图；两者都言简意赅，只写当前状态，不写演变过程。
- 约束优先用类型与断言表达；无法表达或需要额外说明的，用注释补充。
- 交付前必查注释与代码是否一致；不一致或拿不准时报告，不得私自改动。
- 代码内的自然语言一律英文。
- unsafe {} 只包住必须 unsafe 的表达式，前提不自明时在上方写 SAFETY: 注释。

## 测试

- 单元测试放在同文件底部的 #[cfg(test)] mod tests 中。
- 公开项带 doctest，并在其中断言结果。
