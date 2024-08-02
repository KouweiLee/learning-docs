# Rust异步编程学习

## 异步编程是什么？

异步编程是一种并发编程方式，其底层仍基于线程并发，但一个线程上可以有多个任务，一个任务一旦处于`IO`或者其他等待(阻塞)状态，就会被立刻切走并执行另一个任务。这样线程就会一直执行而不会阻塞，并发度非常高。

## 基础语法

### async/.await

* async

这两个关键字是rust内置的语法，**使用 `async fn` 可以创建一个异步函数**，异步函数的返回值为一个`Future`：

```rust
async fn hello_world() {
    println!("hello, world!");
}
```

`Future`可以理解为一个未来将会执行的任务，即要在线程上执行的任务。直接调用异步函数并不会执行该异步函数，需要一个执行器executor才行：

```rust
fn main() {
    let future = hello_world(); // 返回一个Future, 因此不会打印任何输出
    block_on(future); // main线程新开一个线程，来执行`Future`，main线程会等待其运行完成，此时"hello, world!"会被打印输出
}
```

这里`block_on`就是一种executor，会在当前线程上执行Future任务，执行完后再执行main函数。

* .await

当一个异步函数中需要执行另一个异步函数时，可以采用`.await`，调用者future将会等待该异步函数完成后再次被执行。例如：

```rust
async fn hello_world() {
    hello_cat().await; // 执行hello_cat, 阻塞hello_world函数，但不会阻塞线程
    println!("hello, world!");// hello_cat执行完后，打印该语句
}

async fn hello_cat() {
    println!("hello, kitty!");
}
```

也就是说，.await会将一个future加入到执行任务中，阻塞调用者future，但不会阻塞任何一个线程

## 对Future执行poll

Future是惰性的，只有在执行poll时才会执行。执行poll是调度器干的事情，执行poll会在当前线程上执行Future，也就是说是一个同步的过程。
