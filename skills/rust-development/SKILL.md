---
name: rust-development
description: 按用户个人的 Rust 开发风格写作、修改或重构 Rust 代码。任何涉及编写 Rust 代码的任务都应使用。
---

# Rust 个人开发风格

本文记录一种表达式优先、迭代器链驱动的 Rust 写法，覆盖通用风格，另附「no_std / FFI / 嵌入式」与「对外库 crate」两个按项目类型叠加生效的附加节。基线在个人习惯上对齐了官方建议（std 文档、Rust API Guidelines、std 开发指南、clippy）。

## 基线

- nightly-first：默认按 nightly Rust 写；specialization、generic_const_exprs、map_windows 等特性只要能让代码更短更清楚就用，所需 `#![feature(...)]` 声明在 lib.rs 顶部，项目用 rust-toolchain.toml pin 具体 nightly 版本。edition 用最新（2024）。
- 格式化交给默认 rustfmt，不写 rustfmt.toml；`#[rustfmt::skip]` 只用于表格化数据（模型系数表、期望值表）。
- 不设 fmt/clippy 门禁；CI 只需 `cargo build` + `cargo test`。lint 控制写在 lib.rs 内属性（如 `#![deny(clippy::large_stack_arrays)]`），配项级精确 `#[allow(clippy::…)]` 豁免，不用 Cargo.toml 的 `[lints]` 表。
- 代码内一切文字全英文：注释、rustdoc、README、错误消息、测试名。
- 浮点字面量完整写尾零：`0.0`、`1.0`、`150.0 / 1000.0 * G`；指针和数值 cast 用 `as _` 推断。
- 无调用点的代码删除（git 有历史）；注释掉的代码不留存，其中的设计判断改写为正式动机注释（`// Deliberately no vibration check here: UB is better than false when Vibration.`）。

## 主线：表达式风格与迭代器链

- 迭代器链是主要组织方式，一个函数常常就是一条链；分散动作尽快汇入链中。
- `for` 与 `for_each` 的边界：对现成绑定做简单副作用遍历用 `for` 循环（含需要 `break`/`continue`/提前 `return` 的场合）；`.for_each` 只作为长链末尾的收尾。

```rust
// Simple side-effecting traversal over a binding -> for
for w in wakers.drain(..) {
    w.wake();
}

// Chain-end side effect -> for_each
(1..=4)
    .flat_map(arc)
    .enumerate()
    .for_each(|(i, p)| out[i] = p);
```

- 布尔二分用 `if/else` 表达（含单行 `let plural = if n > 1 { "s" } else { "" };`）；match 留给枚举、模式和多条件组合——条件对用 tuple match；条件产值用 `bool::then` / `then_some` / `is_ok_and` / `is_some_and` 组合子；早返回只做守卫（`if out.is_null() { return; }`）。

```rust
let axis = if stroke.is_stroke() {
    CountingAxis::Determined(axis)
} else {
    last_counting_axis
};

match (cond1, cond2) {
    (true, true) => RunningState::Confident,
    (false, false) => RunningState::Stop,
    _ => RunningState::Uncertain,
}

(self.idx >= Self::CAPACITY && (self.idx - Self::CAPACITY).is_multiple_of(ROLLING_SIZE))
    .then_some(&self.buf)
```

- 闭包内联在链中，不把闭包赋给变量再调用；方法引用更短时优先（`.map(FloatCore::is_nan)`、`.unwrap_or_else(Invalid::invalid)`、`.filter_map(Result::ok)`）。
- 惯用高级适配器：`map_windows`（可叠加两层做二阶差分）、`scan`、`tee`、`next_chunk`、`chunk_by`、`exactly_one`、`sorted_by_key`、`k_largest_by_key`、`izip!`/`chain!`、`iter::successors` 做不动点迭代、`either::Either` 统一分支迭代器类型。
- let-chains、`@` 绑定、or-patterns、guard-as-pattern、range pattern 都是常规手段。
- 只为方法的 trait 用 `use Trait as _` 匿名导入（`use itertools::Itertools as _;`、`use core::ops::Not as _;`）；`.not()` 等方法式运算符调用可用。
- 解包读取局部信息，未用成员用 `_` 承接，不用位置索引；闭包参数可直接解构结构体（`|(i, Point { x, y })| …`）。

## 扩展 trait：核心抽象单元

- 缺方法就写 blanket 扩展 trait，三层对应三种载体：`IteratorExt`、`SliceExt<T>: AsRef<[T]>`、`ArrayExt<T, const N: usize>: Borrow<[T; N]>`，blanket 实现一行 `impl<I: Iterator> IteratorExt for I {}`。
- 约束放方法级 `where`（`where Self: Sized, Self::Item: FloatCore`），不上提到 trait 级。
- 用 specialization 给具体类型开快速路径：blanket impl 里 `default fn … { unimplemented!() }` 配 `#[track_caller]`，具体类型 impl 提供真实现。

```rust
pub(crate) trait SliceExt<T>: AsRef<[T]> {
    fn median(&self) -> T
    where
        T: FloatCore + TotalOrder,
    { /* ... */ }
}

impl<T, S: AsRef<[T]>> SliceExt<T> for S {}
```

## 模块组织与可见性

- use 按 std/core → 外部 crate → crate:: 三段分组、段间空行；`mod x;` 声明逐源分段，与其显式导入紧邻成对。
- glob 导入只用于两处：内部 prelude（`use crate::prelude::*;`）和测试的 `use super::*;`；其余一律显式列举（`use feature::{Imu, LazyAxisFeatures};`）。
- 目录模块用 `foo.rs` + `foo/` 布局，不用 mod.rs。
- 内部 prelude.rs 用 `pub(crate) use` 汇集常用项，扩展 trait 匿名导入（`pub(crate) use crate::math::{SliceExt as _, IteratorExt as _};`）。
- 可见性默认 `pub(crate)`；真 `pub` 只留给对外 API/FFI 面，lib.rs 里模块保持私有 `mod`。
- 命名用 Rust 惯例：类型 CamelCase 域优先（`RollingPadded`、`StopDetectionState`），const generics 用 SCREAMING_SNAKE 单词（`ROLLING_SIZE`、`SAMPLE_RATE`），短类型参数 T/U，生命周期只用 `'a`；不用 C# 式 `TEventArgs`/`IMemo` 前缀。常量单位入名（`IMU_SAMPLE_RATE_HZ`），模块级 const 块紧随 import。
- `is_`/`has_` 谓词前缀只用于返回 bool 的函数；返回枚举或分数改用名词/动词命名（`walk_state() -> RunningState`、`counting_score() -> f32`）。

## 错误处理与断言

- 内部不变式用 `expect`，消息写 precondition 风格——说明"为什么应该成立"，不描述失败（std 官方建议）；不用裸 `unwrap`。不可能分支 `unreachable!()`；specialization 桩 `unimplemented!()`。

```rust
let max = xs.iter().copied()
    .max_by(f32::total_cmp)
    .expect("xs is non-empty, verified by the rolling window");
```

- 算法热路径上的前提约束用带消息的 `debug_assert!`（release 零开销）：

```rust
debug_assert!(
    !xs.iter().any(|x| x.is_nan()),
    "quantile: input contains NaN"
);
```

- 业务上"无答案"返回 `Option`（`fn sd(self) -> Option<Self::Item>`），内部函数不返回 Result。
- 需要真错误类型时用 thiserror：单元结构体错误 + `#[error(transparent)]` `#[from]` 聚合枚举。

```rust
#[derive(Error, Debug, Default, Clone, Copy, PartialEq, Eq, Hash)]
#[error("The memory of the object passed has been corrupted!")]
struct Corrupted;

#[derive(Error, Debug)]
enum PtrError {
    #[error(transparent)]
    Corrupted(#[from] Corrupted),
    #[error(transparent)]
    NullPtr(#[from] NullPtr),
}
```

- 示例与测试用 `anyhow::Result` + `?`，`fn main() -> Result<()>`。

## 类型设计

- 构造用 `fn new(...) -> Self` 配 Default（`Self { strictness, ..Default::default() }`）；参数多且多为可选、或需多步配置时提供 builder（API Guidelines C-BUILDER）；需要共享所有权时 new 直接返回 `Rc<Self>`（加 `#[allow(clippy::new_ret_no_self)]`），自引用图用 `Rc::new_cyclic` 存自身 `Weak`。
- 包装构造显式写目标类型，不用 `.into()` 收尾：`Some(...)`、`Rc::new(...)`、`RefCell::new(...)`；`.into()`/`Into` 只用于真正的类型转换（有语义的 From 实现）。
- 共享可变状态默认单线程：私有 `Inner` 结构 + `Rc<Inner>` 薄包装句柄，标志用 `Cell`、集合用 `RefCell`，回引用与注册凭据持 `Weak<Inner>`；方法一律 `&self` 配内部可变；文档醒目声明 **not thread-safe**；不用 Arc/Mutex。
- 单线程全局可变状态用 `thread_local!` + `RefCell`/`OnceCell`，不用 `static mut`（官方正在废弃其引用，且测试无需单线程串行）。

```rust
thread_local! {
    static EFFECT_STACK: RefCell<Vec<EffectStackEntry>> = RefCell::new(Vec::new());
}

EFFECT_STACK.with_borrow_mut(|stack| stack.push(entry));
```

- 克隆引用计数指针用全限定形式 `Rc::clone(&x)` / `Rc::downgrade(&x)`，数据克隆才用 `.clone()` 方法式。
- derive 固定序 `Debug, Default, Clone, Copy, PartialEq, Eq, Hash`，自定义 derive 排最后；thiserror 类型 `Error` 排最前、其余同序；能 derive 的尽量全 derive。`#[repr(C)]` 写在 derive 之前。
- getter 用字段名不带 `get_` 前缀，简单访问器用 `const fn`；消费式分解用 `into_parts(self) -> (…)`。
- From/TryFrom 转换成对写（ref + owned），owned 委托 ref：`fn from(value: T) -> Self { (&value).into() }`。
- 泛型：闭包参数用 `impl Fn(...) + 'static`；返回 `impl Iterator<Item = …>`；`dyn` 只在需要存储时用（`Box<dyn FnOnce()>`）。

## 注释与文档

- 注释只写动机（why）：设计取舍、不变式、非显然意图；不复述代码字面含义。doc 文本里用 **bold** 强调契约关键词。
- 参数和返回值融入散文描述（首句说明行为，参数用 `name` 内联提及），不用 `# Parameters` / `# Returns` 列表；`#` 章节只用官方四件套 `# Examples` / `# Panics` / `# Errors` / `# Safety`。
- 文档密度分两档：
  - **对外库 crate**：`//!` crate 文档含 `## Motivation` / `## Design Notes` / `## Limitations` 等章节；每个公开项带可断言的 doctest，惯用两例——`## Basic usage` 与 `## Using inside a struct`；公开函数会 panic 的写 `# Panics`，返回 Result 的写 `# Errors`（C-FAILURE）。
  - **算法/内部 crate**：只有 FFI 面和公开类型重文档，内部代码只写动机注释，不为私有项机械配 `///`。
- unsafe 双层文档：API 层 `# Safety` 章节写"调用者必须保证什么"（unsafe trait 写完整不变式清单，以 "Violating any of the above requirements results in **undefined behavior**." 收尾）；每个 unsafe 块前一行 `// SAFETY:`（大写）写"此处为何满足"（std 开发指南政策，clippy undocumented_unsafe_blocks 同款）。

```rust
/// # Safety
///
/// The caller must ensure `ptr` is either null or points to a valid memory region.
pub unsafe extern "C" fn pedometer_free(ptr: *mut PedometerHandle) {
    // SAFETY: ptr was created by pedometer_new via Box::into_raw and is
    // never freed twice (magic is poisoned on drop).
    let this = unsafe { Box::from_raw(ptr as *mut Pedometer) };
}
```

## 重复与抽象

- 仓库内重复优先于过早抽象：重复 2-3 次（如逐传感器轴）就内联展开；FFI 错误模板、handle 校验等模块级样板每模块照抄一份。
- 跨仓库共享的工具代码抽成 workspace 内共享 crate；trait crate 与 derive crate 成对出现（`invalid` + `invalid-derive`），trait crate 再导出 derive，用户只 import trait crate。

## no_std / FFI / 嵌入式（附加节）

- `no_std` 作为 feature 而非无条件：`#![cfg_attr(feature = "no_std", no_std)]`，`default = ["std"]`，同一 crate 宿主机可测、嵌入式裁剪。
- 每文件按需前置 `#[cfg(feature = "no_std")] use alloc::vec::Vec;`；库代码路径一律 `core::`，不写 `std::`。
- 多条件 cfg 垂直叠加，每行一个维度（feature/target/debug），单一逻辑条件才用 `all(...)`；feature 门控可细到 struct 字段、函数参数、表达式块（stmt_expr_attributes）。

```rust
#[cfg(debug_assertions)]
#[cfg(feature = "r")]
#[cfg(all(target_os = "windows", target_env = "gnu", target_arch = "x86_64"))]
mod r;
```

- FFI handle 模式：opaque `#[repr(C)]` Handle 类型 + `magic: u32` 必须为首字段（`u32::from_ne_bytes(*b"PDM3")`，版本号入 magic）+ `TryFrom<*mut Handle> for &mut T` 做 null/magic 校验 + drop 时毒化 magic；FFI 出错返回 `Invalid::invalid()` 哨兵值（NaN/MAX），不用错误码，组合子兜底 `.unwrap_or_default()` / `.unwrap_or_else(Invalid::invalid)`。

```rust
fn try_from(value: *mut PedometerHandle) -> Result<Self, Self::Error> {
    // SAFETY: null is rejected by ok_or below; a non-null ptr must come
    // from pedometer_new, and the magic check guards against corruption.
    unsafe { (value as *mut Pedometer).as_mut() }
        .ok_or(NullPtr.into())
        .and_then(|this| (this.magic == MAGIC).then_some(this).ok_or(Corrupted.into()))
}
```

- release profile：`opt-level = "z"`、`codegen-units = 1`、`lto = true`、`strip = true`、`panic = "abort"`。

## 对外库 crate（附加节）

- Cargo.toml 字段序固定：name / version / edition / authors / description / repository / license / keywords / categories；license 用 `MIT OR Apache-2.0`（C-PERMISSIVE，生态惯例）；依赖写完整三段版本号不带操作符（`slab = "0.4.11"`，声明真实最低版本）；依赖数保持最少。
- 结构体字段私有 + getter（C-STRUCT-PRIVATE，保留演化空间）；FFI 的 `#[repr(C)]` 类型例外（字段布局本身就是契约）。
- README 模板：badges（crates.io + docs.rs）→ `## Features` → `## Installation`（`[dependencies]` TOML 块）→ `## Usage` → `## API` 要点列表；Usage 示例与 lib.rs `//!` doctest 保持同一份（允许重复）。
- `macro_rules!` 只做一行薄糖委托到函数或类型（`$crate::` 限定 + `#[macro_export]`，放 src/macros.rs），文档承载全部说明；proc macro 放独立 `-macros` workspace 成员，主 crate feature 门控再导出。
- 异步原语手写 Future impl，方法返回具名 future 类型（`fn cancelled(&self) -> CancelledFuture`），waker 去重用 `will_wake`。
- 回调注册表用 `slab::Slab`，`usize` key 作为可移除 handle。

## 测试

- 单元测试放同文件底部 `#[cfg(test)] mod tests`，`use super::*;` 置于其他测试 import 之后；tests/ 目录只放数据 fixture 或宏集成测试。
- 命名不加 `test_` 前缀（mod tests 已提供上下文），用行为描述短语（`median_of_even_length_slice`、`cancel_two_tasks`）；复杂场景在测试函数上加 `///` 说明。
- 断言直接 `assert_eq!` 对硬编码期望值，浮点也对精确字面量；空/单元素/NaN 等边界在同一个测试函数里穷举；debug 契约用 `#[should_panic(expected = "…")]` 且套 `#[cfg(debug_assertions)]`；内存泄漏用 `drop` 后 `Weak::upgrade().is_none()` 验证。
- 依赖本地数据的测试用存在性检查提前返回，或 `#[ignore]` 留待手动执行。
