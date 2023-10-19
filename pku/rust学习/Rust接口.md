# Rust接口

* as_ref()

Converts from `&Option<T>` to `Option<&T>`.

```rust
let text: Option<String> = Some("Hello, world!".to_string());
// First, cast `Option<String>` to `Option<&String>` with `as_ref`,
// then consume *that* with `map`, leaving `text` on the stack.
let text_length: Option<usize> = text.as_ref().map(|s| s.len());
println!("still can print text: {text:?}");
```

* unwrap()

可以用于option和result，如果是Some或Ok，则返回该值；如果是None或Err，则panic。

* as_ptr()

返回指向某变量首地址的裸指针，如：

```rust
let string = "Hello World!";
let pointer = string.as_ptr() as usize;
```

* as_mut_ptr()

返回指向某变量首地址的可变裸指针

* FlattenObjects

存储编号对象的容器。

* core::slice::from_raw_parts

```rust
pub const unsafe fn from_raw_parts<'a, T>(data: *const T, len: usize) -> &'a [T]
```

根据数组基地址指针data和元素个数len，形成一个slice

## 裸指针

* 读裸指针

read()方法，从裸指针中读值

### c指针到rust的转换

* 指针转换为slice

```rust
core::slice::from_raw_parts
pub const unsafe fn from_raw_parts<'a, T>(data: *const T, len: usize) -> &'a [T]
```

将类型为T的指针转换为slice（不可变引用），元素个数为len。

转换为可变引用的版本为：`from_raw_parts_mut`

* 字符串指针转换为&str

```rust
axlibc::utils::char_ptr_to_str
pub fn char_ptr_to_str<'a>(str: *const c_char) -> LinuxResult<&'a str>
```

* CStr

引用C字符串的表示形式。它表示对一个以\0结尾的字节数组的引用，可以通过`&[u8]`安全地构造，也可以通过`*const c_char`构造。Cstr可以转化为`&str`。

接口函数：

通过*const c_char 构造&Cstr

```rust
pub unsafe fn from_ptr<'a>(ptr: *const c_char) -> &'a CStr
```

将C字符串变为字节slice：

```rust
pub fn to_bytes(&self) -> &[u8]
pub fn to_bytes_with_nul(&self) -> &[u8] // 结尾会多\0
```

将C字符串转换为&str：

```rust
pub fn to_str(&self) -> Result<&str, Utf8Error>
```

* 裸指针转换为引用

as_mut方法，可以将*mut T转换为&mut T。

### rust到c指针的转换

Vec转变为指向Vec的指针：

```
as_mut_ptr
```

## slice

1. 从裸指针生成slice

```
from_raw_parts_mut
from_raw_parts
```

2. fill

填充slice

3. copy_from_slice(dst, src)

从src将内容拷贝到dst中。

## Fromstr

实现了Fromstr，就可以通过一个&str执行`parse::<xxx>()`来获得xxx类型了。

## 函数语法

当操作集合或迭代器时，一些函数式语法常用：map, filter, reduce,

对option和result，`map_or`也常用。

## 消费者与适配器

* **消费性适配器**

如果一个函数，内部会调用迭代器的`next`方法，那么该函数就称为消费性适配器。因为它会逐渐消耗掉迭代器上的元素，最终返回一个值。

常见的几种消费性适配器：

1. collect

将一个迭代器中的元素收集到指定类型中。

* **迭代器适配器**

迭代器适配器会根据已有的迭代器，返回一个新的迭代器。不过不能只停留在这一步，还需要一个消费性适配器来收尾，最终返回一个值。

```rust
v1.iter().map(|x| x + 1);//报错
let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();// 正确
```

例如`map`就是迭代器适配器，collect是一个消费性适配器。

常见的几种迭代器适配器：

1. map

接收一个参数为`FnMut`的闭包，将一个迭代器转换为另一个。

2. filter

用于对迭代器中的每个值进行过滤，例如：

```rust
fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}
```

其中s为shoes的一个元素

3. fold函数

fold函数有2个参数，第一个为初值，第二个为一个闭包函数。

举个例子说明该函数的用法，计算阶乘时，可以这样写：

```rust
(1..=num).rfold(1, |ans, now| now * ans)
```

`rfold`表示从集合的右侧向左侧计算。ans初始值为1，now为当前集合中的元素。每次计算now * ans后，将结果赋值给ans，一直这样计算直到遍历完整个集合。返回值就是ans

4. find

寻找迭代器中某个元素满足闭包。find接收一个闭包，该闭包返回true或false。

find返回值为option

5. take(n)

返回一个新的迭代器，包含旧迭代器的前n个元素

### 组合器

组合器用于对返回结果的类型进行变换，例如从Option变为Result。几种常见的组合器如下：

1. `or()`和`and()`

类似于布尔表达式，这两个方法对两个Option/Result表达式做逻辑组合，最终返回`Option` / `Result`。特别地，对于`and()`，如果两个表达式结果都是`Some`或`Ok`，则返回**第二个**表达式的值。

```rust
let s1 = Some("some1");
let s2 = Some("some2");
assert_eq!(s1.or(s2), s1);
```

2. `or_else()`和`and_then()`

它们跟 `or()` 和 `and()` 类似，唯一的区别在于，它们的第二个表达式是一个闭包，闭包的返回值作为第二个表达式的值。

```rust
let s1 = Some("some1");
let fn_some = || Some("some2");
assert_eq!(s1.or_else(fn_some), s1);
```

3. `filter`

`filter`用于对 `Option` 进行过滤：

```rust
let s1 = Some(3);
let s2 = Some(6);
let n = None;

let fn_is_even = |x: &i8| x % 2 == 0;

assert_eq!(s1.filter(fn_is_even), n);  // Some(3) -> 3 is not even -> None
assert_eq!(s2.filter(fn_is_even), s2); // Some(6) -> 6 is even -> Some(6)
assert_eq!(n.filter(fn_is_even), n);   // None -> no value -> None
```

4. `map()`和`map_err()`

`map` 可以将 `Some` 或 `Ok` 中的值映射为另一个，对于None则不变

```rust
let s1 = Some("abcde");
let s2 = Some(5);
// 一个闭包
let fn_character_count = |s: &str| s.chars().count();
assert_eq!(s1.map(fn_character_count), s2); // Some1 map = Some2

```

`map_err`则是将`Err`中的值进行改变，遇到`Ok`则什么都不干

```rust
let e1: Result<&str, &str> = Err("404");
let e2: Result<&str, isize> = Err(404);
let fn_character_count = |s: &str| -> isize { s.parse().unwrap() }; // 该函数返回一个 isize
assert_eq!(e1.map_err(fn_character_count), e2); // Err1 map = Err2
```

5. `map_or()`和`map_or_else()`

`map_or` 在 `map` 的基础上提供了一个默认值，可用于`Option`或`Resutl`。对于Option，如果是None，则返回该默认值；如果是Some，则返回Some中的值。这点和`map`不一样。

```rust
const V_DEFAULT: u32 = 1;
let s: Result<u32, ()> = Ok(10);
assert_eq!(s.map_or(V_DEFAULT, fn_closure), 12);
```

`map_or_else` 与 `map_or` 类似，但是它是通过一个闭包来提供默认值:

```rust
let e = Err(5);
let fn_default_for_result = |v: i8| v + 1; // 闭包可以对 Err 中的值进行处理，并返回一个新值
assert_eq!(e.map_or_else(fn_default_for_result, fn_closure), 6);
```

6. `ok_or()`和`ok_or_else()`

它们可以将 `Option` 类型转换为 `Result` 类型。

`ok_or` 接收一个默认的 `Err` 的参数:

```rust
const ERR_DEFAULT: &str = "error message";

let s = Some("abcde");
let n: Option<&str> = None;

let o: Result<&str, &str> = Ok("abcde");
let e: Result<&str, &str> = Err(ERR_DEFAULT);

assert_eq!(s.ok_or(ERR_DEFAULT), o); // Some(T) -> Ok(T)
assert_eq!(n.ok_or(ERR_DEFAULT), e); // None -> Err(default)
```

`ok_or_else` 接收一个闭包作为 `Err` 的参数：

```rust
let s = Some("abcde");
let n: Option<&str> = None;
let fn_err_message = || "error message";

let o: Result<&str, &str> = Ok("abcde");
let e: Result<&str, &str> = Err("error message");

assert_eq!(s.ok_or_else(fn_err_message), o); // Some(T) -> Ok(T)
assert_eq!(n.ok_or_else(fn_err_message), e); // None -> Err(default)
```
