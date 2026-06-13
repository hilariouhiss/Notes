# Rust 入门教学文档

> 版本：Rust 2024 Edition (1.95.0 Stable) | 更新日期：2026-05-18

## 一、Rust 简介

Rust 是一门由 Mozilla 研究院开发的系统级编程语言，首次发布于 2010 年，1.0 版本发布于 2015 年。

### 1.1 核心设计目标

- **内存安全**：通过所有权系统在编译期消除空指针、悬垂指针、数据竞争等问题
- **零成本抽象**：高级特性不带来运行时性能损耗
- **并发安全**：编译器保证线程安全，无需垃圾回收器（GC）
- **实用主义**：出色的工具链（Cargo）、友好的错误信息、完善的包管理

### 1.2 适用场景

- 系统编程（操作系统、嵌入式、驱动、Rust for Linux）
- Web 后端（Actix、Axum、Rocket）
- 命令行工具（Clap、Serde）
- 区块链/Web3
- 游戏引擎（Bevy）
- 跨平台 GUI（Slint、Tauri、Iced）

### 1.3 版本与 Edition

Rust 采用六周发布周期，当前最新稳定版为 **1.95.0**。  
语言通过 **Edition** 机制演进，保持向后兼容的同时引入新特性：

| Edition | 稳定版本 | 主要特性 |
|---------|---------|---------|
| 2015 | 1.0 | 初始发布 |
| 2018 | 1.31 | `async/await`、模块系统改进 |
| 2021 | 1.56 | 数组 `IntoIterator`、闭包捕获改进 |
| **2024** | **1.85** | **异步闭包、生命周期捕获、`if let` chains、更严格的 `unsafe`** |

> 不同 Edition 的 crate 可以互相依赖。新项目建议直接使用 `edition = "2024"`。

---

## 二、环境搭建

### 2.1 安装 Rust（通过 rustup）

```bash
# Linux / macOS
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Windows
# 下载并运行 https://win.rustup.rs/
```

安装完成后，Rust 工具链包含：

- `rustc`：Rust 编译器
- `cargo`：构建系统与包管理器（最常用）
- `rustfmt`：代码格式化工具
- `clippy`：静态分析 linter
- `rust-analyzer`：LSP 语言服务器

### 2.2 验证安装

```bash
rustc --version   # 例如：rustc 1.95.0 (2026-05-??)
cargo --version   # 例如：cargo 1.95.0
```

### 2.3 创建第一个项目

```bash
cargo new hello_world
cd hello_world
cargo run
```

生成的目录结构：

```
hello_world/
├── Cargo.toml      # 项目配置与依赖
└── src/
    └── main.rs     # 入口文件
```

`Cargo.toml`（Rust 2024 Edition）：

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
```

`main.rs` 内容：

```rust
fn main() {
    println!("Hello, world!");
}
```

---

## 三、基础语法

### 3.1 变量与可变性

```rust
fn main() {
    // 默认不可变（immutable）
    let x = 5;
    // x = 6;  // 错误！不能对不可变变量二次赋值

    // 可变变量
    let mut y = 5;
    y = 6;  // 正确

    // 常量（编译期确定，必须标注类型）
    const MAX_POINTS: u32 = 100_000;

    // 变量遮蔽（Shadowing）
    let z = 5;
    let z = z + 1;      // 创建新变量，允许改变类型
    let z = "hello";    // 现在 z 是 &str 类型
}
```

### 3.2 数据类型

Rust 是**静态强类型**语言，编译器通常能自动推断类型。

#### 3.2.1 标量类型

```rust
fn main() {
    // 整数
    let a: i32 = -42;       // 有符号 32 位
    let b: u64 = 100;       // 无符号 64 位
    let c = 0xff;           // 十六进制
    let d = 0o77;           // 八进制
    let e = 0b1111_0000;    // 二进制（可用下划线分隔）

    // 浮点数
    let f: f64 = 3.14;      // 默认 f64，精度更高
    let g: f32 = 2.5;

    // 布尔型
    let t: bool = true;

    // 字符型（Unicode 标量值，4 字节）
    let heart: char = '❤';
}
```

#### 3.2.2 复合类型

```rust
fn main() {
    // 元组（Tuple）：固定长度，可包含不同类型
    let tup: (i32, f64, &str) = (500, 6.4, "hi");
    let (x, y, z) = tup;          // 解构
    let five_hundred = tup.0;     // 索引访问

    // 数组（Array）：固定长度，同类型
    let arr = [1, 2, 3, 4, 5];
    let first = arr[0];
    let a = [3; 5];               // [3, 3, 3, 3, 3]
}
```

### 3.3 函数

```rust
// 参数必须声明类型
fn add(a: i32, b: i32) -> i32 {
    a + b  // 表达式返回值（无分号）
}

fn main() {
    let result = add(5, 3);
    println!("结果: {}", result);

    // 语句 vs 表达式
    let y = {
        let x = 3;
        x + 1       // 表达式，返回 4
    };              // y = 4
}
```

### 3.4 控制流

```rust
fn main() {
    let number = 7;

    // if 表达式（条件必须是 bool，无隐式转换）
    if number < 5 {
        println!("小于 5");
    } else if number == 5 {
        println!("等于 5");
    } else {
        println!("大于 5");
    }

    // if 作为表达式
    let condition = true;
    let value = if condition { 5 } else { 6 };

    // 循环
    let mut counter = 0;
    loop {
        counter += 1;
        if counter == 3 { break; }
    }

    // while
    let mut n = 3;
    while n != 0 {
        n -= 1;
    }

    // for（最常用，安全且高效）
    let arr = [10, 20, 30];
    for element in arr {
        println!("值: {}", element);
    }

    // Range
    for i in 1..=5 {      // 1 到 5（包含）
        println!("{}", i);
    }
}
```

---

## 四、所有权系统 — Rust 的核心

所有权（Ownership）是 Rust 最独特的特性，用于管理内存而无需垃圾回收器。

### 4.1 三条规则

1. **每个值都有一个所有者（owner）**
2. **同一时间只能有一个所有者**
3. **当所有者离开作用域，值将被丢弃**

### 4.2 变量作用域与内存释放

```rust
fn main() {
    {
        let s = String::from("hello");  // s 进入作用域
        println!("{}", s);
    }                                   // s 离开作用域，内存自动释放
}
```

### 4.3 移动（Move）语义

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;          // s1 的所有权移动到 s2

    // println!("{}", s1);  // 错误！s1 不再有效
    println!("{}", s2);    // 正确
}
```

> **原理**：`String` 存储在堆上，包含指向堆内存的指针。移动时只复制指针，不复制堆数据，同时使原变量失效，避免双重释放。

### 4.4 克隆（Clone）

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();  // 深拷贝堆数据

    println!("s1 = {}, s2 = {}", s1, s2);  // 两者都有效
}
```

### 4.5 拷贝（Copy）trait

标量类型（如整数、布尔、浮点、字符、只包含标量的元组）默认实现 `Copy`，赋值时按位复制，原变量仍可用。

```rust
fn main() {
    let x = 5;
    let y = x;    // Copy，x 仍可用
    println!("x = {}, y = {}", x, y);
}
```

### 4.6 引用（Borrowing）与借用规则

```rust
fn main() {
    let s1 = String::from("hello");

    // 不可变引用
    let len = calculate_length(&s1);
    println!("'{}' 长度是 {}", s1, len);  // s1 仍可用

    // 可变引用
    let mut s = String::from("hello");
    change(&mut s);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}  // s 离开作用域，但不释放指向的数据

fn change(s: &mut String) {
    s.push_str(", world");
}
```

#### 4.6.1 借用规则（编译器强制检查）

1. **任一时刻，只能有一个可变引用**，或**任意数量的不可变引用**
2. **引用必须始终有效**（不能悬垂）

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;      // 不可变引用 1
    let r2 = &s;      // 不可变引用 2（允许）
    println!("{} {}", r1, r2);

    let r3 = &mut s;  // 错误！已有不可变引用时不能有可变引用
}
```

### 4.7 切片（Slice）

切片是对集合的部分引用，不拥有所有权。

```rust
fn main() {
    let s = String::from("hello world");
    let hello = &s[0..5];    // 字符串切片 (&str)
    let world = &s[6..11];

    let arr = [1, 2, 3, 4, 5];
    let slice = &arr[1..3];  // 数组切片 (&[i32])
}
```

---

## 五、结构体与枚举

### 5.1 结构体（Struct）

```rust
// 定义
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

// 元组结构体
struct Point(i32, i32, i32);

// 单元结构体（无字段，常用于 trait）
struct AlwaysEqual;

fn main() {
    // 创建实例
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("alice"),
        active: true,
        sign_in_count: 1,
    };

    // 结构体更新语法
    let user2 = User {
        email: String::from("bob@example.com"),
        ..user1    // 其余字段从 user1 获取
    };
    // 注意：user1.username 被移动，user1 不再完整可用
}
```

### 5.2 枚举（Enum）

```rust
enum Message {
    Quit,                           // 无数据
    Move { x: i32, y: i32 },        // 匿名结构体
    Write(String),                  // 单个值
    ChangeColor(i32, i32, i32),     // 元组
}

// 最常用：Option<T>（标准库定义）
// enum Option<T> {
//     Some(T),
//     None,
// }

fn main() {
    let some_number = Some(5);
    let absent_number: Option<i32> = None;

    // 使用 match 处理
    match some_number {
        Some(n) => println!("数字: {}", n),
        None => println!("无值"),
    }
}
```

---

## 六、模式匹配

### 6.1 match 表达式

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(String),  // 关联数据：州名
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => {
            println!("幸运币！");
            1
        }
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("来自 {:?} 州!", state);
            25
        }
    }
}
```

### 6.2 if let 简化匹配

当只关心一个模式时，使用 `if let` 更简洁：

```rust
fn main() {
    let some_value = Some(3);

    // 等价于 match，但更简洁
    if let Some(3) = some_value {
        println!("三");
    }

    // 可搭配 else
    let coin = Coin::Penny;
    if let Coin::Quarter(state) = coin {
        println!("来自 {} 州", state);
    } else {
        println!("不是 Quarter");
    }
}
```

### 6.3 while let

```rust
let mut stack = Vec::new();
stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
    println!("{}", top);  // 输出 3, 2, 1
}
```

---

## 七、错误处理

Rust 没有异常机制，使用 `Result<T, E>` 和 `Option<T>` 处理错误。

### 7.1 Result 枚举

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

```rust
use std::fs::File;

fn main() {
    // 方式 1：match 处理
    let f = File::open("hello.txt");
    let f = match f {
        Ok(file) => file,
        Err(error) => {
            panic!("打开文件失败: {:?}", error);
        }
    };

    // 方式 2：unwrap() / expect()（快速但会 panic）
    let f = File::open("hello.txt").unwrap();
    let f = File::open("hello.txt").expect("无法打开 hello.txt");

    // 方式 3：? 运算符（传播错误）
    let f = File::open("hello.txt")?;  // 需在返回 Result 的函数中使用
}
```

### 7.2 ? 运算符

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;  // 出错时提前返回 Err
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}

// 更简洁的链式写法
fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("hello.txt")?.read_to_string(&mut s)?;
    Ok(s)
}

// 使用 fs::read_to_string（标准库已实现）
use std::fs;
fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("hello.txt")
}
```

### 7.3 Option 与 Result 转换

```rust
fn main() {
    let text: Option<&str> = Some("42");

    // Option -> Result
    let num = text.ok_or("无值")?;           // Ok("42")
    let num = text.ok_or_else(|| "无值")?;   // 惰性求值版本

    // Result -> Option
    let result: Result<i32, &str> = Ok(5);
    let opt = result.ok();  // Some(5)
}
```

---

## 八、集合类型

### 8.1 Vector

```rust
fn main() {
    // 创建
    let mut v = Vec::new();
    v.push(1);
    v.push(2);

    let v2 = vec![1, 2, 3];  // 宏创建

    // 读取
    let third: &i32 = &v2[2];      // 越界时 panic
    let third: Option<&i32> = v2.get(2);  // 越界时返回 None

    // 遍历
    for i in &v2 {
        println!("{}", i);
    }

    // 存储枚举实现多类型
    enum SpreadsheetCell {
        Int(i32),
        Float(f64),
        Text(String),
    }
    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
    ];
}
```

### 8.2 HashMap

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    // 访问
    let team_name = String::from("Blue");
    let score = scores.get(&team_name);  // Option<&i32>

    // 遍历
    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }

    // 不存在才插入
    scores.entry(String::from("Yellow")).or_insert(60);
    scores.entry(String::from("Green")).or_insert(80);
}
```

---

## 九、模块与包管理

### 9.1 模块系统层次

- **包（Package）**：Cargo 项目，包含一个或多个 Crate
- **Crate**：编译单元，可以是二进制（bin）或库（lib）
- **模块（Module）**：用 `mod` 声明，控制作用域与私有性
- **路径（Path）**：`crate::`、`self::`、`super::`、`::`

### 9.2 示例项目结构

```
my_project/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── lib.rs          # 库入口
    ├── front_of_house.rs
    └── front_of_house/
        ├── hosting.rs
        └── serving.rs
```

```rust
// src/front_of_house.rs
pub mod hosting;
pub mod serving;

// src/front_of_house/hosting.rs
pub fn add_to_waitlist() {}

// src/lib.rs
mod front_of_house;

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();  // 通过 pub use 重导出
}
```

### 9.3 常用 Cargo 命令

```bash
cargo new project_name          # 创建项目
cargo build                     # 编译（debug）
cargo build --release           # 编译（release，优化）
cargo run                       # 编译并运行
cargo check                     # 快速检查（不生成二进制）
cargo test                      # 运行测试
cargo doc --open                # 生成并打开文档
cargo add serde                 # 添加依赖
cargo update                    # 更新依赖版本
cargo fix --edition             # 自动迁移到新版 Edition
```

### 9.4 Workspace 与 Edition 继承（Rust 2024）

在 Rust 2024 Edition 中，Workspace 可以统一指定 Edition，成员 crate 自动继承：

```toml
# Workspace Cargo.toml
[workspace]
members = ["crate-a", "crate-b", "crate-c"]

[workspace.package]
edition = "2024"
rust-version = "1.82"  # 最低支持 Rust 版本（MSRV）
```

---

## 十、常用 Trait 与泛型

### 10.1 泛型

```rust
// 泛型函数
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

// 泛型结构体
struct Point<T, U> {
    x: T,
    y: U,
}

// 泛型枚举（如 Option<T>, Result<T, E>）
```

### 10.2 Trait（特征/接口）

```rust
// 定义 trait
pub trait Summary {
    fn summarize(&self) -> String;

    // 默认实现
    fn summarize_author(&self) -> String {
        String::from("(匿名)")
    }
}

// 为类型实现 trait
pub struct NewsArticle {
    pub headline: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}", self.headline)
    }
}

// Trait 作为参数
pub fn notify(item: &impl Summary) {
    println!("突发新闻！{}", item.summarize());
}

// Trait Bound（更灵活）
pub fn notify<T: Summary + Display>(item: &T) {}
```

### 10.3 常用标准库 Trait

| Trait | 作用 |
|-------|------|
| `Debug` | 格式化调试输出 `{:?}` |
| `Display` | 格式化用户输出 `{}` |
| `Clone` | 显式深拷贝 `.clone()` |
| `Copy` | 隐式按位复制（标量类型） |
| `PartialEq` / `Eq` | 相等比较 `==` |
| `PartialOrd` / `Ord` | 大小比较 `<`, `>` |
| `Default` | 默认值 `Default::default()` |
| `Drop` | 离开作用域时自定义清理 |
| `Iterator` | 迭代器 `.next()` |
| `From` / `Into` | 类型转换 |
| `AsRef` / `AsMut` | 引用转换 |
| `Deref` / `DerefMut` | 解引用重载 |
| `Send` / `Sync` | 线程安全标记 trait |
| `Future` / `IntoFuture` | 异步编程（Rust 2024 已加入 prelude）|

**派生宏快速实现**：

```rust
#[derive(Debug, Clone, PartialEq, Default)]
struct User {
    name: String,
    age: u32,
}
```

---

## 十一、生命周期

生命周期是引用有效的作用域，编译器通过它确保引用不会悬垂。

### 11.1 生命周期标注语法

```rust
// 显式标注：返回的引用与 x 或 y 中活得一样长
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let string1 = String::from("long string is long");
    {
        let string2 = String::from("xyz");
        let result = longest(&string1, &string2);
        println!("最长字符串是 {}", result);
    }
}
```

### 11.2 生命周期省略规则

编译器能自动推断某些生命周期，无需显式标注：

1. 每个引用参数都有自己的生命周期
2. 如果只有一个输入生命周期，它会被赋给所有输出生命周期
3. 如果有 `&self` 或 `&mut self`，其生命周期赋给所有输出

```rust
// 以下写法等价（省略后）
fn first_word(s: &str) -> &str { }
fn first_word<'a>(s: &'a str) -> &'a str { }
```

### 11.3 结构体中的生命周期

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,  // part 引用必须比结构体活得长
}

fn main() {
    let novel = String::from("Call me Ishmael.");
    let first_sentence = novel.split('.').next().unwrap();
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}  // i 和 novel 同时离开作用域，合法
```

### 11.4 'static 生命周期

`'static` 表示整个程序运行期间都有效（如字符串字面量）。

```rust
let s: &'static str = "我有 'static 生命周期";
```

### 11.5 impl 块中的生命周期省略（Rust 2024 改进）

Rust 2024 Edition 允许在 impl 块中使用 `'_` 占位符，编译器自动推断生命周期，减少样板代码：

```rust
// Rust 2021: 需要显式标注
impl<'a> MyStruct<'a> {
    fn get_data(&self) -> &'a str {
        self.data
    }
}

// Rust 2024: 生命周期省略更智能
impl MyStruct<'_> {
    fn get_data(&self) -> &str {
        self.data
    }
}
```

### 11.6 RPIT 生命周期捕获（Rust 2024）

Return Position Impl Trait（RPIT）现在更直观地捕获生命周期。返回 `impl Iterator` 或 `impl Future` 时，编译器自动理解返回类型与 `&self` 的生命周期关联：

```rust
// Rust 2024: 自动捕获 &self 的生命周期
fn iter_items(&self) -> impl Iterator<Item = &Item> {
    self.items.iter()
}

// 以前可能需要显式标注或遇到意外的编译错误
```

---

## 十二、并发编程

Rust 通过所有权和类型系统，在编译期防止数据竞争。

### 12.1 线程

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("子线程: {}", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("主线程: {}", i);
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();  // 等待子线程结束
}
```

### 12.2 使用 move 闭包转移所有权

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {  // v 的所有权移动到闭包
        println!("向量: {:?}", v);
    });

    // println!("{:?}", v);  // 错误！v 已被移动

    handle.join().unwrap();
}
```

### 12.3 异步闭包（Rust 2024 新特性）

Rust 2024 Edition 稳定了**异步闭包**，无需依赖 `async-trait` crate：

```rust
use std::future::Future;

// 异步闭包语法：async || {}
let f = async || {
    println!("异步执行");
    42
};

// 调用异步闭包返回 Future，需 await
let result = f().await;
```

异步 trait 方法也无需 `async-trait` 宏（Rust 1.75+ 已稳定，2024 Edition 体验更完整）：

```rust
trait AsyncService {
    async fn process(&self, data: &[u8]) -> Result<Response, Error>;
}
```

### 12.4 消息传递（Channel）

```rust
use std::sync::mpsc;  // 多生产者单消费者
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        // println!("{}", val);  // 错误！val 已被发送（移动）
    });

    let received = rx.recv().unwrap();  // 阻塞接收
    println!("收到: {}", received);
}
```

### 12.5 共享状态（Mutex + Arc）

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });  // lock 在此处自动释放（Drop）
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("结果: {}", *counter.lock().unwrap());
}
```

### 12.6 Send 与 Sync Trait

- `Send`：可以安全地在线程间转移所有权
- `Sync`：可以安全地在线程间共享引用（`&T` 是 `Send`）

Rust 自动为类型实现这两个 trait（如果成员都满足）。几乎所有类型都是 `Send`，除了 `Rc<T>`（非线程安全引用计数）。

---

## 十三、Rust 2024 Edition 新特性速览

如果你从 2021 Edition 迁移或学习，以下是 Rust 2024 最值得关注的变更：

### 13.1 异步闭包（Async Closures）

```rust
// 原生支持 async || {}，返回 impl Future
let fetch = async || {
    reqwest::get("https://api.example.com").await
};
```

### 13.2 `if let` 与 `while let` Chains

允许在 `if let` 后链式添加条件（无需嵌套）：

```rust
if let Some(value) = opt
    && value > 0
    && is_valid(value)
{
    println!("有效值: {}", value);
}
```

### 13.3 更严格的 `unsafe` 要求

- `extern` 块必须标记为 `unsafe extern "C" { ... }`
- 某些属性需用 `#[unsafe(no_mangle)]`、`#[unsafe(export_name)]`

```rust
// Rust 2024
unsafe extern "C" {
    fn c_function();
}

#[unsafe(no_mangle)]
pub fn my_rust_fn() {}
```

### 13.4 `static mut` 引用被禁止

不允许直接创建 `static mut` 的引用，必须通过原始指针访问，鼓励使用 `Mutex` 或原子类型。

### 13.5 `Future` / `IntoFuture` 进入 Prelude

无需显式 `use std::future::Future;`，直接可用 `.await` 和相关 trait。

### 13.6 保留 `gen` 关键字

为未来的 Generator（生成器）语法预留关键字。

### 13.7 迁移方式

```bash
# 自动迁移现有项目
cargo fix --edition
# 手动修改 Cargo.toml
edition = "2024"
```

---

## 十四、学习资源推荐

### 14.1 官方资源

| 资源 | 说明 |
|------|------|
| [The Rust Programming Language](https://doc.rust-lang.org/book/)（Rust 圣经） | 最权威入门教程，有中文社区翻译版 |
| [Rust By Example](https://doc.rust-lang.org/rust-by-example/) | 通过实例学习 |
| [Rustlings](https://github.com/rust-lang/rustlings) | 交互式小练习，适合动手实践 |
| [The Rust Cookbook](https://rust-lang-nursery.github.io/rust-cookbook/) | 常见任务解决方案 |
| [std 文档](https://doc.rust-lang.org/std/) | 标准库 API 文档 |
| [docs.rs](https://docs.rs/) | Crate 文档托管 |
| [Rust 2024 Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2024/index.html) | 2024 Edition 迁移与特性指南 |

### 14.2 中文社区

- [Rust 语言中文社区](https://rustcc.cn/)
- [Rust 程序设计（中文版）](https://kaisery.github.io/trpl-zh-cn/)

### 14.3 推荐学习路径

```
1. 安装 Rust + 完成 Rustlings 练习（1-2 周）
2. 精读《Rust 程序设计》前 10 章（2-3 周）
3. 实现一个小项目（CLI 工具 / Web 服务）（2-4 周）
4. 深入学习生命周期、Trait、Unsafe Rust（持续）
5. 阅读优秀 Crate 源码，参与开源社区
```

### 14.4 入门项目建议

- **命令行工具**：文件搜索、JSON 处理、日志分析
- **Web 后端**：使用 Axum/Actix 构建 REST API
- **数据解析**：实现一个简单的 CSV/JSON 解析器
- **多线程下载器**：结合 channel 和文件 IO
- **异步项目**：使用 Rust 2024 异步闭包构建爬虫或服务

---

## 十五、附录：常见编译错误速查

| 错误信息 | 原因 | 解决 |
|----------|------|------|
| `borrow of moved value` | 值已被移动 | 使用 `.clone()` 或改用引用 |
| `cannot borrow as mutable` | 已有不可变引用 | 确保可变引用唯一性 |
| `mismatched types` | 类型不匹配 | 检查类型标注，必要时转换 |
| `value used here after move` | 所有权已转移 | 重新设计所有权或使用 `Rc`/`Arc` |
| `lifetime mismatch` | 引用活得不够长 | 检查变量作用域，调整生命周期标注 |
| `unsafe attribute used without unsafe` | Rust 2024 新限制 | 改用 `#[unsafe(no_mangle)]` 等 |
| `reference to static mut is not allowed` | 2024 禁止 `static mut` 引用 | 使用 `UnsafeCell` 或原子类型/Mutex |

---

> **提示**：Rust 编译器（`rustc`）的错误信息非常友好，通常直接给出修复建议。遇到错误时仔细阅读 `--explain` 信息或 `cargo check` 的输出，是最高效的学习方式。  
> **Rust 2024 迁移提示**：现有项目运行 `cargo fix --edition` 可自动处理大部分机械变更，随后手动检查 `unsafe` 块和 `static mut` 用法即可。
