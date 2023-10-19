# Rust高级进阶

## 生命周期进阶

### 生命周期约束

通过形如 `'a: 'b` 的语法，可以说明两个生命周期的长短关系。

* `'a:'b`

这种情况说明生命周期`'a >= 'b`。

```rust
struct DoubleRef<'a,'b:'a, T> {
    r: &'a T,
    s: &'b T
}
```

* `T:'a`

类型 `T` 必须比 `'a` 活得要久：

```rust
struct Ref<'a, T: 'a> {
    r: &'a T
}
```

### 再借用

如果一个不可变引用`rr`是对另一个可变引用`r`的再引用，那么`r`与`rr`同时存在，且`rr`生命周期内不使用`r`，则不会产生冲突。如下代码正常运行。

```rust
let mut p = Point { x: 0, y: 0 };
let r = &mut p;
let rr: &Point = &*r;// 再借用

println!("{:?}", rr);// rr生命周期结束
r.move_to(10, 10);// rr结束后才使用r
println!("{:?}", r);
```

再借用的典型应用场景为：函数体内对参数的二次借用

```rust
fn read_length(strings: &mut Vec<String>) -> usize {
   strings.len()
}
```

### &'static和T: 'static

* `&'static`

一个引用所指向的数据必须要活得跟剩下的程序一样久，才能被标注为 `&'static`。

但持有 `&'static` 引用的变量，它的生命周期则受到作用域的限制。

* `T:'static`

`T` 必须活得和程序一样久。

## 函数式编程

### 闭包

一种匿名函数，可以捕获调用者所在作用域的值。

* 函数定义

```rust
|param1, param2,...| {
    语句1;
    语句2;
    返回表达式
}
// 如果只有一个返回表达式，则可简化为
|param1| 返回表达式
```

例如：

```rust
let x = 1;
let sum = |y| x + y;
```

`sum`就是一个函数，调用时通过`sum()`调用。

* 捕获作用域中的值

闭包可以捕获作用域中的值，但捕获值时会分配内存存储这些值，带来内存负担。

#### 作为类型的闭包

闭包也可以作为一个类型，例如：

```rust
struct Cacher<T>
where
    T: Fn(u32) -> u32,
{
    query: T,
    value: Option<u32>,
}
```

其中`Fn`是一个特征，`Fn(u32) -> u32` 也是一个特征，其参数类型为`u32`，返回值类型也为`u32`，用来表示`T`是一个闭包类型。

因此`query`其实就是指一个满足上述条件的函数，调用时`(self.query)(arg)`即可。

#### 3种Fn特征

闭包捕获变量有三种途径，好对应函数参数的三种传入方式：转移所有权、可变借用、不可变借用。因此对应的`Fn`特征也有3种：

1. `FnOnce`，该类型的闭包会拿走被捕获变量的所有权，因此Once的意思就是闭包函数只能执行一次。

   ```rust
   fn fn_once<F>(func: F)
   where
       F: FnOnce(usize) -> bool,
   {
       println!("{}", func(3));
       println!("{}", func(4));// 报错
   }
   
   fn main() {
       let x = vec![1, 2, 3];
       fn_once(|z|{z == x.len()})
   }
   ```

   如果要让`func`能执行两次，可以修改：

   ```rust
   F: FnOnce(usize) -> bool + Copy
   ```

   这样调用时使用的是`func`的拷贝，不会发生所有权的转移。

   如果想强制闭包取得捕获变量的所有权，可以在参数列表前添加 `move` 关键字：

   ```rust
   let handle = thread::spawn(move || {
       println!("Here's a vector: {:?}", v);
   });
   ```

2. `FnMut`，以可变借用的方式捕获了环境中的值

   ```rust
   let mut s = String::new();
   let mut update_string =  |str| s.push_str(str);// 将闭包声明为可变类型，获得s的引用
   update_string("hello");
   ```

3. `Fn`，以不可变借用的方式捕获环境中的值 

**一个闭包实现了哪种 Fn 特征取决于该闭包如何使用被捕获的变量，而不是取决于闭包如何捕获它们**。move强调的是后者。

#### 闭包作为返回值

通过特征对象，如Box实现。

```rust
fn factory(x:i32) -> Box<dyn Fn(i32) -> i32> {
    let num = 5;

    if x > 1{
        Box::new(move |x| x + num)
    } else {
        Box::new(move |x| x - num)
    }
}
```



### 迭代器

* Iterator

迭代器实现了`Iterator`特征，该特征包含方法`next`，如果有值返回Some，无值返回None

* IntoIterator

数组实现了`IntoIterator`特征，在for循环中可以自动转换为迭代器。当然也可以显式地通过`into_iter`、`iter`、`iter_mut`方法将数组转换成迭代器。这三种方法的区别为：

1. `into_iter` 会夺走所有权，得到Some(T)
2. `iter` 是不可变借用，调用next方法返回的类型是`Some(&T)`
3. `iter_mut` 是可变借用，调用next方法返回的类型是`Some(&mut T)`

迭代器要理解成一个新的变量， 它只是可能夺走或借用元素的所有权



## 深入类型

### 类型转换

解引用 `Deref` Trait 是 Rust 编译器唯一允许的一种隐式类型转换，而对于其他的类型转换，我们必须手动调用类型转化方法或者是显式给出转换前后的类型

#### From/into

`From`特征允许一个类型可以通过另一个类型来创建自己，也就是说，如果A类型实现了From<B\>，那么就可以通过B来创建A。实现了`From`特征，就会自动实现`Into`特征。也就是说：

只要实现了 `impl From<T> for U`， 就可以使用以下两个方法: `let u: U = U::from(T)` 和 `let u:U = T.into()`，前者由 `From` 特征提供，而后者由自动实现的 `Into` 特征提供。

使用`into`方法时，如果编译器无法推理得到，则需要显式地将类型标记出来。

### 类型别名

就是类型的一个别名，不是新的类型：

```rust
type Meters = u32;
type Thunk = Box<dyn Fn() + Send + 'static>;
```

### Sized和不定长类型DST

Rust中有两类类型：

- 定长类型( sized )，这些类型的大小在编译时是已知的
- 不定长类型( unsized )，与定长类型相反，它的大小只有到了程序运行时才能动态获知，这种类型又被称之为 DST

Rust 中常见的 `DST` 类型有: `str`、`[T]`、`dyn Trait`，它们都无法单独被使用，必须要通过引用或者 `Box` 来间接使用 。

* Sized特征

使用泛型时，如果直接使用泛型参数，也需要是固定大小的。编译器会自动为T加上`Sized`特征约束。但如果想在泛型函数中使用动态数据类型，则可以使用`?Sized`特征：

```rust
fn generic<T: ?Sized>(t: &T)
```

## 智能指针

智能指针往往都实现了`Deref`和`Drop`特征。String类型和Vec都是智能指针。

### Box<T\>

通过Box可以创建一个智能指针，该指针存放在栈上，而指针指向的数据存放在堆上。在 `Box<T>` 生命周期结束被回收的时候，堆上的那块空间也会立即被一并回收。它可以在如下4个场景中使用：

1. 将数据存储在堆上

2. 避免栈上数据的拷贝

   栈上数据转移所有权时，是将数据拷贝了一份，最终新旧变量各自拥有不同的数据，因此所有权并没有转移。

   而堆上所有权转移时，仅仅会复制一份栈中的指针，拥有旧指针的变量会失效。

   ```rust
   // 在栈上创建一个长度为1000的数组
   let arr = [0;1000];
   // 将arr所有权转移arr1，由于 `arr` 分配在栈上，因此这里实际上是直接重新深拷贝了一份数据
   let arr1 = arr;
   // 在堆上创建一个长度为1000的数组，然后使用一个智能指针指向它
   let arr = Box::new([0;1000]);
   //arr 不再拥有所有权
   let arr1 = arr;
   ```

3. 将动态大小类型DST变成Sized固定大小类型，例如使用递归类型时，可以这样写：

   ```rust
   enum List {
       Cons(i32, Box<List>),
       Nil,
   }
   ```

4. 特征对象。

* Box::leak函数

该函数可以消费掉Box，并强制其目标值从内存中泄漏，得到一个static有效的值。

```rust
fn main() {
   let s = gen_static_str();
   println!("{}", s);
}

fn gen_static_str() -> &'static str{
    let mut s = String::new();
    s.push_str("hello, world");
	// s.into_boxed_str()得到一个Box<str>的值
    Box::leak(s.into_boxed_str())// 得到&str
}
```

使用场景：你需要一个在运行期初始化的值，但是可以全局有效，也就是和整个程序活得一样久

### Deref解引用

解引用就是得到一个引用的值，使用运算符`*`。

智能指针本身是一个结构体类型，如果对结构体解引用，则会报错。但智能指针实现了`Deref`特征，可以直接对其解引用，例如：

```rust
let x = Box::new(1);
let sum = *x + 1;// x被解引用
```

* 定义自己的智能指针

我们定义一个自己的智能指针来说明如何实现Deref特征：

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}
// 实现Deref特征
use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T; //关联类型

    fn deref(&self) -> &Self::Target {
        &self.0
    } //返回一个引用，避免所有权的转移
}
fn main() {
    let y = MyBox::new(5);
    assert_eq!(5, *y);
}
```

当对智能指针解引用时，实际上是执行了：

```rust
*(y.deref())
```

* 函数和方法中的隐式Deref转换

> 以下的Deref，貌似不是解引用，而是引用。将&也理解进这一范畴就算了

对于函数的传参，如果参数实现了Deref特征，那它的**引用**在传给函数或方法时，会根据参数签名来决定是否进行隐式的 `Deref` 转换，例如：

```rust
let s = String::from("hello world");
// &s是一个&String类型，会自动转换为&str
display(&s);
fn display(s: &str) {
    println!("{}",s);
}
```

这种隐式Deref转换还支持连续的转换，直到匹配到函数形参类型。

* 使用方法、赋值中的`Deref`

举个例子：

```rust
let s = MyBox::new(String::from("hello, world"));
let s1: &str = &s;
let s2: String = s.to_string();
```

**赋值操作需要手动解引用**，也就是`s`前要加`&`以实现Deref转换。

**方法调用会自动解引用**，`s`可以直接调用`to_string`方法。

* Deref总结

一个类型为 `T` 的对象 `foo`，如果 `T: Deref<Target=U>`，那么，相关 `foo` 的引用 `&foo` 在应用的时候会自动转换为 `&U`。



### Drop释放资源

几乎所有类型都实现了`Drop`特征，在离开作用域时会自动释放所占有的内存。

使用`drop`函数可以实现手动drop。例如：

```rust
let foo = Foo;
drop(foo);
```

### Rc与Arc

这两个机制是通过引用计数的方式，**允许一个数据资源在同一时刻拥有多个所有者**。不过注意，创建的是**不可变引用。**如果要修改，可配合互斥锁`Mutex`或`RefCell`

#### Rc<T\>

`Rc<T>`适用于单线程，并实现了Deref特征

* 创建Rc智能指针

```rust
let a = Rc::new(String::from("hello, world"));
```

使用 `Rc::new` 创建了一个新的 `Rc<String>` 智能指针并赋给变量 `a`。创建时，会将引用计数加1.

* 克隆智能指针

```rust
let b = Rc::clone(&a);
```

使用 `Rc::clone`克隆一份智能指针`Rc<String>`，并将引用计数增加到2。 这里的克隆只是复制了智能指针，也可以使用`a.clone()`的方式。

* 观察引用计数

引用计数(reference counting)，通过记录一个数据被引用的次数来确定该数据是否正在被使用。当引用次数归零时，就代表该数据不再被使用，因此立刻被清理释放。

```rust
let a = Rc::strong_count(&a)
```

#### Arc<T\>

Arc是原子化的 `Rc<T>` 智能指针，可以保证数据能安全的在线程间共享。但性能消耗大。

`Arc` 和 `Rc` 拥有完全一样的 API。

#### Weak<T\>

Weak指针是Arc的一种版本，不过不会增加其指向内容的引用计数。通常用于避免Arc之间的环状指向关系，导致所有对象都不可以被回收。通常对于一棵树，父节点指向子节点时用Arc，而子节点指向父节点用Weak，建立双向链接关系。

### Cell和RefCell

用于实现内部可变性。

* Cell

Cell和RefCell功能上没有区别，区别在于`Cell<T>`用于`T`实现了`Copy`的情况，取值直接拷贝而不是取引用。

```rust
let c = Cell::new("asdf");
let one = c.get();// one为"asdf"
c.set("qwer");
```

get方法用来取值，set方法用来设置新值。

* RefCell

只能在单线程上使用，且不在堆上分配内存，而是基于数据段的静态内存分配。

`RefCell` 用于提供引用，因为有时候编译器检查引用可能太严格了，RefCell可以将编译器的引用规则检查推迟到程序运行时。不过违背借用规则会导致运行期的 `panic`。

```rust
let s = RefCell::new(String::from("hello, world"));
let s1 = s.borrow();
let s2 = s.borrow_mut();
打印s1, s2 // 会报错
```

要删除这个引用，直接使用`drop(s1)`即可；也可以利用rust规则，离开作用域后自动释放。

* 内部可变性的应用

内部可变性是指，在变量自身不可变或仅在不可变借用的情况下仍能修改绑定到变量上的值。

```rust
let x = Cell::new(1);
let y = &x;
let z = &x;
x.set(2);
y.set(3);
z.set(4);
```

内部可变性允许对一个不可变的值进行可变借用。例如结构体的值只能在某个特定方法内部进行修改，其他地方不能修改，例如：

```rust
// 定义一个发送者trait
pub trait Messenger {
    fn send(&self, msg: String);
}

pub struct MsgQueue {
    msg_cache: RefCell<Vec<String>>, // 通过RefCell可以修改msg_cache
}

impl Messenger for MsgQueue {
    fn send(&self, msg: String) {
        self.msg_cache.borrow_mut().push(msg)
    }
}

fn main() {
    let mq = MsgQueue {
        msg_cache: RefCell::new(Vec::new()),
    };
    mq.send("hello, world".to_string());
}
```

* Rc+RefCell组合使用

一个常见的组合就是 `Rc` 和 `RefCell` 在一起使用，前者可以实现一个数据拥有多个所有者，后者可以实现数据的可变性：

```rust
let s = Rc::new(RefCell::new("我很善变，还拥有多个主人".to_string()));
let s1 = s.clone();
let s2 = s.clone();
s2.borrow_mut().push_str(", on yeah!");
```

使用 `RefCell<String>` 包裹一个字符串，同时通过 `Rc` 创建了它的三个所有者：`s`、`s1`和`s2`，并且通过其中一个所有者 `s2` 对字符串内容进行了修改。



## 多线程并发编程

### 使用多线程

使用 `thread::spawn` 可以创建线程：

```rust
use std::thread;
thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });
```

线程内部的代码使用闭包来执行，且main线程一旦结束，程序立刻结束。

如果要等待子线程执行结束，则这样写：

```rust
let handle = thread::spawn(|| {...});

handle.join().unwrap();// 主线程阻塞
```

* 线程屏障Barrier

使用 `Barrier` 让多个线程都执行到某个点后，才继续一起往后执行

使用 `move` 关键字可将变量的所有权转移给新的线程。

* 只被调用一次的函数

有时，我们会需要某个函数在多线程环境下只被调用一次，这时可以使用`Once`：

```rust
use std::sync::Once;
static mut VAL: usize = 0;
static INIT: Once = Once::new();
// call_once只会执行一次
let handle1 = thread::spawn(move || {
    INIT.call_once(|| {
        unsafe {
            VAL = 1;
        }
    });
});

let handle2 = thread::spawn(move || {
    INIT.call_once(|| {
        unsafe {
            VAL = 2;
        }
    });
});
```

首先创建一个Once实例，用于控制函数只执行一次。

* 只被初始化一次的变量

和上面提到的Once类似，当使用`spin::once`时，可以对变量进行唯一一次的lazy 初始化。

```rust
use spin::once::Once;
static START: spin::Once = spin::Once::new();
START.call_once(|| {
    // 初始化函数，返回值会作为START变量内部的值
});
```



### 线程同步：

#### 互斥锁

* 创建互斥锁

```rust
use std::sync::Mutex;
let m = Mutex::new(5);
```

* 获取互斥锁

```rust
// 获取锁，然后deref为`m`的引用
// lock返回的是Result
let mut num = m.lock().unwrap();
*num = 6;
```

`lock()`是阻塞式的，`m.lock()`返回一个智能指针`MutexGuard<T>`，该智能指针实现了Deref和Drop特征。

要手动Drop锁的话，直接执行`drop(num)`。

* 多线程内部可变引用的实现

**互斥锁Mutex+Arc<T\>，可以实现多线程之间的共享变量。**

```rust
let counter = Arc::new(Mutex::new(0));
// 通过mutex可以改变数据
let mut num = counter.lock().unwrap();
*num += 1;
```

* `try_lock`尝试获取锁

与lock不同，try_lock会尝试获取锁，**不会阻塞**。

#### 读写锁RwLock

`Mutex`会对每次读写都进行加锁。如果需要大量的**并发读**，可以采用`RwLock`。

```rust
let lock = RwLock::new(5);
{	
   	// 同一时间允许多个读
	let r1 = lock.read().unwrap();
    let r2 = lock.read().unwrap();
}
{   
    // 同一时间只允许一个写
    let mut w = lock.write().unwrap();
    *w += 1;
	// 之后如果再读会Panic
}
```

可以使用`try_write`和`try_read`来尝试进行一次写/读，若失败则返回错误`Err("WouldBlock")`

不过注意，读写锁的性能可不咋地，一般不推荐。

#### 条件变量

条件变量用于解决资源访问顺序（即同步）的问题。它经常和`Mutex`一起使用，可以让线程挂起，直到某个条件发生后再继续执行。



#### Atomic

原子指的是一系列不可被 CPU 上下文交换的机器指令，这些指令组合在一起就形成了原子操作。

使用原子操作的效率比加锁高，且具有内部可变性，无需将其声明为`mut`，就可以修改。`std::sync::atomic`包中只提供了数值类型的原子操作：`AtomicBool`, `AtomicIsize`, `AtomicUsize`, `AtomicI8`, `AtomicU16`等。举个例子：

```rust
let n = AtomicU64::new(0);
n.fetch_add(0, Ordering::Relaxed);
```

其中`Ordering::Relaxed`用于限定内存顺序，防止编译器和CPU对指令进行重排。该枚举共有5个成员：

> - **Relaxed**， 这是最宽松的规则，它对编译器和 CPU 不做任何限制，可以乱序
> - **Release 释放**，设定内存屏障(Memory barrier)，保证它之前的操作永远在它之前，但是它后面的操作可能被重排到它前面
> - **Acquire 获取**, 设定内存屏障，保证在它之后的访问永远在它之后，但是它之前的操作却有可能被重排到它后面，往往和`Release`在不同线程中联合使用
> - **AcqRel**, 是 *Acquire* 和 *Release* 的结合，同时拥有它们俩提供的保证。比如你要对一个 `atomic` 自增 1，同时希望该操作之前和之后的读取或写入操作不会被重新排序
> - **SeqCst 顺序一致性**， `SeqCst`就像是`AcqRel`的加强版，它不管原子操作是属于读取还是写入的操作，只要某个线程有用到`SeqCst`的原子操作，线程中该`SeqCst`操作前的数据操作绝对不会被重新排在该`SeqCst`操作之后，且该`SeqCst`操作后的数据操作也绝对不会被重新排在`SeqCst`操作前。

原则上，**`Acquire`用于读取，而`Release`用于写入**。对于同时具有读写功能的原子操作，则使用`AcqRel`。在内存屏障中被写入的数据，都可以被其他线程及时读取到，不会存在cache带来的问题。

### 基于Send和Sync的线程安全

`Send`和`Sync`是 Rust 安全并发的重中之重，是一种标记特征，未定义任何行为。它们的作用是：

实现`Send`的类型可以在线程间安全的传递其所有权, 实现`Sync`的类型可以在线程间安全的共享(通过引用)。

注意事项：

1. 绝大部分类型都实现了`Send`和`Sync`，常见的未实现的有：裸指针、`Cell`、`RefCell`、`Rc` 等
2. 可以为自定义类型实现`Send`和`Sync`，但是需要`unsafe`代码块
3. 可以为部分 Rust 中的类型实现`Send`、`Sync`，但是需要使用`newtype`，例如文中的裸指针例子

如果一个类型并没有实现Sync和Send，那么可以包装一下为其实现。

## 全局变量

* 静态常量

类似于C语言中的全局常量

```rust
const MAX_ID: usize =  usize::MAX / 2;
```

* 静态变量

类似于C语言中的全局变量，不过必须使用unsafe才能修改和访问静态变量，因为多线程访问时不免遇到脏数据。

```rust
static mut REQUEST_RECV: usize = 0;
```



## unsafe rust

超出了 Rust 语义约束的行为包裹在` unsafe `块中，告知编译器不需要对它进行完整的约束检查，而是由程序员自己负责保证它的安全性

### 五种用途

#### 解引用裸指针

裸指针类似于引用，和C语言的指针是非常像的，分为不可变和可变的，分别写作`*const T` 和 `*mut T`。这里的星号不是解引用运算符，而是类型名称的一部分。**不可变**是指指针解引用之后不能直接赋值。

创建裸指针是安全的：

```rust
let mut num = 5;

let r1 = &num as *const i32;
let r2 = &mut num as *mut i32;
```

对裸指针解引用却需要放在unsafe代码块中：

```rust
unsafe {
    println!("{}", *r1);
}
```

#### 调用unsafe函数

一个函数如果加上unsafe前缀则表示该函数是不安全的，强制调用者要用unsafe语句块包含该函数的调用：

```rust
unsafe fn dangerous() {}
fn main() {
    unsafe { // 不加unsafe则报错
    	dangerous();
	}
}
```

还有，`unsafe` 无需俄罗斯套娃，在 `unsafe` 函数体中使用 `unsafe` 语句块是多余的行为。

如果一个函数包含了unsafe代码块，但确定这个函数肯定是安全的，则不需要把它命名为unsafe函数。

####  调用外部函数

FFI：Foreign Function Interface，可以使得rust与其他语言的外部代码进行交互。参考手册：[External blocks - The Rust Reference (rust-lang.org)](https://doc.rust-lang.org/reference/items/external-blocks.html)

例如rust要使用C语言函数：

```rust
extern "C" {
    // 声明 C 函数原型
    fn c_function(a: i32, b: i32) -> i32;
}

let result = unsafe {
    // 调用 C 函数
    c_function(10, 20)
};
```

在extern "xxx"中可以使用属性`#[link_name = "symbol name"]`，指定该函数的真实名字。

其他语言如果要使用rust，则在函数定义时加上extern关键字：

```rust
#[no_mangle] // 必须加，避免编译器修改函数名
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```

#### 访问或修改静态变量

静态变量允许声明一个全局的变量：

```rust
static mut REQUEST_RECV: usize = 0;
fn main() {
   unsafe {
        REQUEST_RECV += 1;
        assert_eq!(REQUEST_RECV, 1);
   }
}
```

必须使用`unsafe`才能访问和修改`static`变量。

### 内联汇编

* global_asm

在Rust代码中可以嵌入汇编代码。如果汇编代码是在一个文件中，则该文件需要通过`include_str`将汇编代码转换为字符串，gloval_asm则可以将其嵌入到代码中。否则rust编译器不会注意到这个文件。

```rust
global_asm!(include_str!("entry.asm"));
```

* asm

`global_asm!` 宏用来嵌入全局汇编代码，而 `asm!` 宏可以将汇编代码嵌入到局部的函数上下文中， `asm!` 宏可以获取上下文中的变量信息并允许嵌入的汇编代码对这些变量进行操作。注意需要unsafe。例如：

```rust
unsafe {
    asm!(
        "ecall",
        // x10寄存器同时作为输入和输出寄存器，args[0]首先绑定到x10，当ecall返回时x10的值赋给ret
        inlateout("x10") args[0] => ret,
        // 将args[1]绑定到x11中
        in("x11") args[1],
        in("x12") args[2],
        in("x17") id
    );
}
```

in这些指令会在ecall执行之前执行。

## 宏编程

Rust宏的基本运作机制就是：首先匹配宏规则中定义的模式，然后将匹配 结果绑定到变量，最后展开变量替换后的代码。

分有两种宏：

### 声明式宏macro_rules!

参考教程：[Rust宏编程新手指南【Macro】 | 学习软件编程 (hubwiz.com)](http://blog.hubwiz.com/2020/01/30/rust-macro/)

它和match表达式很像，

```rust
macro_rules! hey{
  () => {},
  () => {}
}
```

()是匹配器，用于匹配模式并捕捉变量。{}则是转码器，利用之前捕捉的变量和这部分代码来生成实际的rust代码替换这个宏。

`($name:expr)`：$name定义了变量名，匹配结果会存入变量`$name`中。冒号后面是匹配的类型，该例子中是表达式，还有其他选择器：

- item：条目，例如函数、结构、模块等
- block：代码块
- stmt：语句
- pat：模式
- expr：表达式
- ty：类型
- ident：标识符
- path：路径，例如` foo、 ::std::mem::replace, transmute::<_, int>, …`
- meta：元信息条目，例如 #[…]和 #![rust macro…] 属性
- tt：词条树

在转码器中，只需要在常规的rust代码中，嵌入之前捕捉到的变量即可：

```rust
($name:expr) => {
	print!("Hey {}", $name)//如果要加分号，在这里加分号;
}
```

* 重复模式的提取和利用

例如`vec!`宏，可以支持非常多的输入。重复模式的匹配这样写：

```rust
($($x:expr), *)
```

 逗号表示这些重复的模式会以逗号分隔，`$x`则类似一个数组迭代器一样，保存了所有输入。

* 举例：Hash表写法

如果想定义一个根据给定键值对创建hashmap的宏：

```rust
let hashmap = map!(
	"name" => "Finn",
    "gender" => "Boy"
);
```

则匹配模式可以这样写：

```rust
($($key:expr => $value:expr), *)
```

转码器这样写：

```rust
{{
	let hm = HashMap::new();
    $(hm.insert($key, $value); )*
    hm
}}
```

其中`$()*`的意思是其中的代码根据重复匹配的次数来重复展开。

* 举例：实现vec!宏

```rust
#[macro_export]   //宏导出
macro_rules! vec {//宏定义
    ( $( $x:expr ),* ) => {//模式，如果匹配则下面这段代码则会替换传入的源代码
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    },
}
```

* 多个分支的宏

一个宏也可以支持传入多种不同方式的参数，例如：

```rust
macro_rules! my_macro {
    () => {
        println!("Check out my macro!")
    }; // 用分号隔开即可
    ($val:expr) => {
        println!("Look at this other macro: {}", $val)
    }
}
```



### 过程宏

过程宏使用源代码作为输入参数，输出一段新的代码。

有3种过程宏：

#### 1. derive

用于为类实现特征。要定义derive宏，需要在一个单独的包中，包名后缀为`derive`。包中需要添加：

```
[lib]
proc-macro = true

[dependencies]
syn = "1.0"
quote = "1.0"
```

而后在lib.rs中加入：

```rust
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;
use syn::DeriveInput;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 基于 input 构建 AST 语法树
    let ast:DeriveInput = syn::parse(input).unwrap();

    // 构建特征实现代码
    impl_hello_macro(&ast)
}
```

一般来说只需要修改`imlp_hello_macro`函数即可，前面的都和上者一样就行。

```rust
fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;// 类名
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```

最终过程宏会输出一份新的代码。使用时只需要在类上面加`#[derive(xxx)]`

使用cargo expand会将derive宏展开。

#### 2. 类属性宏

和derive宏类似，但类属性宏可以让我们定义一个类似属性的属性，且可用于函数等其他类型。例如：

```rust
#[route(GET, "/")]// 类属性宏的使用
fn index() {
```

类属性宏的定义：

```rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
```

attr：属性包含的内容，Get、"/"

item：属性所标注的类型项，在这里是`fn index() {...}`

#### 3. 类函数宏

类函数宏可以像`macro_rules`作为一个函数使用。例如：

```rust
// 类函数宏的定义
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
// 类函数宏的使用
let sql = sql!(SELECT * FROM posts WHERE id=1);
```

参考、学习资料：[Macro 宏编程 - Rust语言圣经(Rust Course)的最后](https://course.rs/advance/macro.html#未来将被替代的-macro_rules)

### 常见宏

1. `option_env!`

用于获取环境变量的值并返回一个 `Option<&'static str>`。

## 属性Attribute

### 条件编译

条件编译可以让源代码基于特定的条件决定是否编译。使用属性`cfg`、`cfs_attr`可以实现条件编译。这些属性都会接收一个谓语，来决定是否编译：

1. 单独选项
2. all()：包含多个选项，必须选项都满足才为true
3. any()：只有一个满足即可编译
4. not()：类似于取反

选项分有两种，一种是键值对，一种是name。

### derive

通过 `#[derive(...)]` 可以让编译器为你的类型提供一些 Trait 的默认实现：

1. 实现了 `Clone` Trait 之后就可以调用 `clone` 函数完成拷贝
2. 实现了 `PartialEq` Trait 之后就可以使用 `==` 运算符比较该类型的两个实例
3. `Copy` 是一个标记 Trait，决定该类型在按值传参/赋值的时候采用移动语义还是复制语义

## 杂

### 条件编译Features

Feature可以通过`Cargo.toml`中的`[features]`定义。其中每个 `feature` 通过列表的方式指定了它所能启用的其他 `feature` 或可选依赖。

例如：

```rust
[features]
bmp = []
png = []
ico = ["bmp", "png"] // 开启ico，会自动开启后面的
webp = []
```

在代码中可以通过cfg表达式进行条件编译：

```rust
#[cfg(feature = "webp")]// 只有webp feature被定义后，一下的模块才会被引入
pub mod webp;
```

默认情况下，所有feature都会自动禁用。可以通过default来启用它们：

```rust
[features]
default = ["ico", "webp"]
。。。
```

使用如上配置的项目被构建时，`default` feature 首先会被启用，然后它接着启用了 `ico` 和 `webp` feature。如果要关闭default，可以通过：

- `--no-default-features` 命令行参数可以禁用 `default` feature
- `default-features = false` 选项可以在依赖声明中指定

使用`cfs_if!`宏，可以在编译时根据feature来执行分支判断语句。

### 自动化测试

添加注解`#[test]`可以将一个函数变为测试函数，当执行cargo test时会自动查看这些函数的正确性。

添加注解`#[should_panic]`可以标记一个函数应该执行时会发生panic，这样该函数测试通过不会报错了就。

详见[How to Write Tests - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/stable/book/ch11-01-writing-tests.html#checking-for-panics-with-should_panic)

### 外部库

1. lazy_static

```
[dependencies]
lazy_static = { version = "1.4.0", features = ["spin_no_std"] }
```

该库提供宏lazy_static!，可以实现全局变量运行时初始化，只有该全局变量第一次被使用时，才会进行实际的初始化工作。

不过，`lazy_static`宏中的变量，必须是`static ref`，所以定义的静态变量都是不可变引用

2. bigflags

[bitflags](https://docs.rs/bitflags/1.2.1/bitflags/) 是一个 Rust 中常用来比特标志位的 crate。它提供了一个 `bitflags!` 宏，如上面的代码段所展示的那样，可以将一个 `u8` 封装成一个标志位的集合类型，支持一些常见的集合运算。

```rust
[dependencies]
bitflags = "1.2.1"

use bitflags::*;
bitflags! {
    pub struct PTEFlags: u8 {
        const V = 1 << 0;
        const R = 1 << 1;
        const W = 1 << 2;
        const X = 1 << 3;
        const U = 1 << 4;
        const G = 1 << 5;
        const A = 1 << 6;
        const D = 1 << 7;
    }
}
```



### 注解

#### 链接

为了支持链接操作，需要先支持特征：

```
#![feature(linkage)]
```

1. 设定链接位置

```
#[link_section = ".text.entry"]
pub extern "C" fn _start() -> ! {
```

2. 设置弱链接

```
#[linkage = "weak"]
fn main()
```

如果有两个函数名都叫main的，那么最终链接时弱链接的优先级低，不会被链接。

### Build.rs

rust在进行编译之前，会先执行Build.rs。参考手册：[Build Scripts - The Cargo Book (rust-lang.org)](https://doc.rust-lang.org/cargo/reference/build-scripts.html)

通过打印以cargo:开头的命令，可以让cargo执行该命令。

```
println!("cargo:{}", your_command_in_string);
```

## 编程思想

1. 不可变全局变量的内部可变性的实现

我们希望把一个类实例化为一个全局变量，使任何函数都可以直接访问，而且这个类中包含一些可变字段。但如果采用`static mut`声明，任何对于 `static mut` 变量的访问控制都是 unsafe 的，因此为了减少unsafe，可以采用`static+RefCell`。但由于rust编译时会默认全局变量会在多个线程之间共享，而RefCell并没有实现`Sync`特征，因此可以对RefCell一层Wrapper，如下：

```rust
pub struct UPSafeCell<T> {
    /// inner data
    inner: RefCell<T>,
}
unsafe impl<T> Sync for UPSafeCell<T> {}
impl<T> UPSafeCell<T> {
    /// User is responsible to guarantee that inner struct is only used in
    /// uniprocessor.
    pub unsafe fn new(value: T) -> Self {
        Self { inner: RefCell::new(value) }
    }
    /// Panic if the data has been borrowed.
    pub fn exclusive_access(&self) -> RefMut<'_, T> {
        self.inner.borrow_mut() // 获得一个可变引用
    }
}
lazy_static! {
static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe { UPSafeCell::new({
        ...
        })};
}
```

这样，当初始化APP_MANAGER后，就不需要unsafe代码了，只需要通过exclusive_access获得可变引用，进行修改即可。

## Rust 秘典学习

* 别名

如果变量和指针指向内存的重叠区域，那么它们就是*别名*。别名对编译器的代码优化有很大影响。

* 丢弃标志

Rust中的内存分有3种：已初始化、未初始化和重新初始化。对于Copy类型的数据可以不用关注，而带有析构器的类型则需要关注。要关注的其实就是重新初始化：

```rust
let mut x = Box::new(0); 
let y = &mut x;
*y = Box::new(1); // 解引用假设原先的变量已经初始化了，因此一定会 drop
```

通过解引用的赋值会无条件地被丢弃，因为解引用之前一定是被初始化过的。

丢弃标志则是用于**变量**未初始化、已初始化、重新初始化不出错的Rust机制。

* 有关drop的讨论

通常情况下，当我们使用`=`赋值给一个 Rust 类型检查器认为已经初始化的值时（比如`x[i]`），存储在左边的旧值会被drop。如果在一个未初始化数据上调用drop，会出现一些错误。正确的选择是使用ptr模块的write、copy和copy_nonoverlapping

1. `ptr::write(ptr, val)`接收一个`val`并将其移动到`ptr`所指向的地址

2. `ptr::copy(src, dest, count)`将`count`个 T 所占用的位从 src 复制到 dest

3. `ptr::copy_nonoverlapping(src, dest, count)`做的是`copy`的工作，但是在假设两个内存范围不重叠的情况下，速度更快(这等同于 C 的 memcpy )

他们不会对dst进行drop，但可能会对src进行所有权的转移。

* 对于数组初始化的问题

数组不允许部分初始化。但如果要想使用部分初始化，则可以使用MaybeUninit。详见

[Unchecked - Rust 秘典（死灵书） (purewhite.io)](https://nomicon.purewhite.io/unchecked-uninit.html)
