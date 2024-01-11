# bindgen

作用：自动生成针对已有C库的Rust对应版本的代码。

教程：[Create a build.rs File - The bindgen User Guide (rust-lang.github.io)](https://rust-lang.github.io/rust-bindgen/tutorial-3.html)

文档：[bindgen - Rust (docs.rs)](https://docs.rs/bindgen/latest/bindgen/)

使用bindgen要建立一个build.rs文档。其中的内容类似于acreos中的ulib/build.rs。

```rust
let mut builder = bindgen::Builder::default()
    .header(in_file)
    .clang_arg("-I./include")
	.parse_callbacks(Box::new(MyCallbacks))
    .derive_default(true)
    .size_t_is_usize(false)
    .use_core();
```

