# Rust in C

## 类型

在core::ffi中，包含了一些C类型的变量类型，包括`c_char`，`c_int`、`c_void`等等，这些类型与C语言的对应类型完全一样。

## str指针变换

`core::ffi::CStr`提供了对一个C类型string的引用表示。

通过from_ptr可以创建一个CStr的引用:

```rust
pub unsafe fn from_ptr<'a>(ptr: *const c_char) -> &'a CStr
```

通过to_str可以转化为`&str`：

```rust
pub fn to_str(&self) -> Result<&str, Utf8Error>
```

