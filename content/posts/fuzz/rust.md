---
title: Rust Fuzz
date: 2025-11-30 23:30:38
categories: Fuzz
tags:
  - Rust
mathjax:
top:
---

## 简介

传统的 AFL、libFuzzer 和 honggfuzz 主要支持用 C/C++ 语言开发的项目。但是现在随着 Rust、Go 等新兴编译型语言的兴起，相应的出现了对这些语言开发的项目进行 Fuzz 的需求。因此，人们在基于传统的 Fuzz 思想的基础上，逐步为这些语言构建了 Fuzz 工具。本文所要学习的 cargo-fuzz 正是如此。


## Rust环境配置

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
#然后选1，将以下命令写入
source $HOME/.cargo/env
```


## cargo-fuzz

cargo-fuzz 是 Rust 代码模糊测试的推荐工具。不过，cargo-fuzz 本身并不是一个 fuzzer，而是一个调用 fuzzer 的工具。目前，它仅仅支持通过 libfuzzer-sys crate 调用 LibFuzzer。

### 环境搭建

由于 libFuzzer 需要 LLVM sanitizer 支持，所以我们需要将 Rust 编译器切换至 nightly 版本

```shell
# 安装nightly版本
rustup install nightly

# 设置为默认编译器
rustup default nightly
```

安装 cargo-fuzz 工具：

```shell
cargo install cargo-fuzz
```


### 基本用法

接下来我们将对 Rust 的 URL 解析库  [rust-url](https://github.com/servo/rust-url) 进行 Fuzzing 来学习如何使用 cargo-fuzz。

环境搭建：

首先，克隆 rust-url 项目并切换指定提交：

```shell
git clone https://github.com/servo/rust-url.git
cd rust-url

git checkout bfa167b4e0253642b6766a7aa74a99df60a94048
```

初始化 Fuzz 环境

```shell
cargo fuzz init
```

查看现有的 fuzz 目标

```
❯ cargo fuzz list
fuzz_target_1
```

默认生成了 fuzz_target_1 文件。

**配置测试目标：**

编辑默认生成的测试文件`fuzz-targets/fuzz_target_1.rs`，修改为以下内容。

```rust
#![no_main] //禁用默认 main 函数入口
extern crate libfuzzer_sys; //导入 libfuzzer_sys，
use libfuzzer_sys::fuzz_target;
extern crate url; //引入 url crate

//模糊测试接口函数
fuzz_target!(|data: &[u8]| {
    // 忽略非 URF-8 输入
    if let Ok(s) = std::str::from_utf8(data) {
	    // 测试目标函数
        let _ = url::Url::parse(s);
    }
});
```

>`fuzz_target!`是 cargo-fuzz 提供的宏，用于定义 Fuzz 入口。libFuzzer 会持续调用它，并传入生成的数据。类似于 C/C++ 中的`LLVMTestOneInput`。

**构建语料：**

在 fuzz 目录下创建`corpus/fuzz_target_1`目录，然后将语料存放在`corpus/fuzz_target_1`目录下。

```
cd fuzz
mkdir -p corpus/fuzz_target_1       
mv seed.txt corpus/fuzz_target_1       
```

**开始 Fuzzing：**

通过`cargo fuzz run`运行目标，自动编译带插桩的目标。

```shell
cargo fuzz run fuzz_target_1
```

在跑了一会之后程序就直接 crash 了，并将 crash 样本保存到了`artifacts/fuzz_target_1`中。

<img src="assets/Pasted%20image%2020250711154408.png" style="zoom: 50%;" />

根据调用栈跟踪发现具体问题在`#18 ... rust-url/src/parser.rs:287:46`，错误信息为：

```
thread '<unnamed>' panicked at ... attempt to subtract with overflow
```

说明某个地方发生了`x - y`的运算，其中`x < y`，在 release 模式下没有检查，但是我们通过 ASan 捕捉到了。


复现 crash：

```shell
 cargo fuzz run fuzz_target_1 artifacts/fuzz_target_1/crash-7336c41a88a07fc4343d1cea44ddc092589caff9
```

cargo fuzz 的其它常用参数：

- `cargo fuzz add target`：创建新的模糊测试目标。
- `cargo fuzz fmt target input`：打印测试用例的输入。
- `cargo fuz tmin target input`：缩小crash。
- `cargo fuzz cmin target`：缩小输入语料库。
- `cargo fuzz coverage`：生成覆盖率信息。
- 可以设置环境变量`RUST_BACKTRACE=1`来决定是否打印调用栈。


### libFuzzer支持

通过`--`可以使用 libFuzzer 的参数：

```shell
cargo fuzz run target_name -- -max_len=512 -only_ascii=1
```

参数：

- `-ignore_crashes=1`：即使遇到崩溃也继续 Fuzz。
- `-s,--sanitizer`：选择 Sanitizer。
- `-max_len=<len>`：限制输入最大长度。
- `-runs=<number>`：限制运行次数。
- `-max_total_time=<sec>`：限制总运行时间（秒）。
- `-timeout=<sec>`：单次运行超时时间（秒）。
- `-only_ascii=1`：仅生成 ASCII 输入。
- `-dict=<file>`：使用字典文件。
- `-c, --careful`：开启额外的安全检查。
- `--no-cfg-fuzzing`：禁用默认的`cfg(fuzzing)`编译配置。



## 测量代码覆盖率

### 环境搭建

```shell
# 安装 LLVM 工具集（用于 coverage）
rustup component add --toolchain nightly llvm-tools-preview
cargo install cargo-binutils
cargo install rustfilt
```

### 基本使用

测量代码覆盖率需要通过 cargo-fuzz，所以又回到了 cargo-fuzz。

```shell
#手动插桩目标项目
RUSTFLAGS="-C instrument-coverage" cargo build --release
#测量覆盖率
cargo fuzz coverage fuzz_target_1
```

这里 cargo fuzz 会自动测试`corpus/fuzz_target_1`目录下的语料。

最后在`fuzz/coverage/fuzz_target_1`目录下生成了覆盖率文件`coverage.profdata`。

<img src="assets/Pasted%20image%2020250711220637.png" style="zoom: 67%;" />


### 可视化覆盖率信息

生成 HTML 格式覆盖率报告：

```shell
cargo cov -- show fuzz/target/x86_64-unknown-linux-gnu/release/fuzz_target_1 \
  --format=html \
  --Xdemangler=rustfilt \
  --instr-profile=fuzz/coverage/fuzz_target_1/coverage.profdata \
  > coverage.html
```

<img src="assets/Pasted%20image%2020250711224857.png" style="zoom: 67%;" />

终端查看：

```shell
cargo cov -- show --use-color \
  --instr-profile=fuzz/coverage/fuzz_target_1/coverage.profdata \
  fuzz/target/x86_64-unknown-linux-gnu/release/fuzz_target_1 \
  --show-regions --show-instantiations
```

<img src="assets/Pasted%20image%2020250711222737.png" style="zoom:50%;" />

## 参考

>[rust-fuzz/trophy-case: 🏆 Collection of bugs uncovered by fuzzing Rust code](https://github.com/rust-fuzz/trophy-case)
>[afl_stat - Rust](https://docs.rs/afl-stat/latest/afl_stat/#:~:text=This%20crate%20implement%20parsing%20AFL,file%20generated%20by%20AFL%20instances)
>[设置 - Rust Fuzz Book](https://rust-fuzz.github.io/book/afl/setup.html)
>[rust-fuzz/afl.rs: 🐇 Fuzzing Rust code with American Fuzzy Lop](https://github.com/rust-fuzz/afl.rs/tree/master)

