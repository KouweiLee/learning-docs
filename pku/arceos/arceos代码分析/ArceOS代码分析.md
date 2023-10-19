# ArceOS代码分析

## printf原理

printf函数目前不支持浮点数，支持哪些不支持哪些详见format_string_loop函数，位于printf.c文件中。

* `printf`函数如下：

```c
int printf(const char *restrict fmt, ...)
{
    int ret;
    va_list ap;
    va_start(ap, fmt);
    ret = vfprintf(stdout, fmt, ap);
    va_end(ap);
    return ret;
}
```

其中`va_list`创建了一个可变参数指针，初始化时指向第一个可变参数。

* 之后调用了`vfprintf`函数，其中`stdout`为标准输出文件。

```c
int vfprintf(FILE *restrict f, const char *restrict fmt, va_list ap)
{
    return vfctprintf(__out_wrapper, f, fmt, ap);
}
```

* 接着调用`vfctprintf`函数：

```c
// extra_arg为std_out文件
int vfctprintf(void (*out)(char c, void *extra_arg), void *extra_arg, const char *format, va_list arg)
{
    output_gadget_t gadget = function_gadget(out, extra_arg);
    return vsnprintf_impl(&gadget, format, arg);
}
```

其中`function_gadget`会返回一个`output_gadget_t`的对象，其function为out，extra_function_arg为extra_arg（FILE*），max_chars为缓冲区大小。

```c
// wrapper (used as buffer) for output function type
typedef struct {
    void (*function)(char c, void *extra_arg);
    void *extra_function_arg;
    char *buffer;
    printf_size_t pos;
    printf_size_t max_chars;
} output_gadget_t;
```

* 接着调用`vsnprintf_impl`函数

```c
// internal vsnprintf - used for implementing _all library functions
static int vsnprintf_impl(output_gadget_t *output, const char *format, va_list args)
{
    // Note: The library only calls vsnprintf_impl() with output->pos being 0. However, it is
    // possible to call this function with a non-zero pos value for some "remedial printing".
    format_string_loop(output, format, args);

    // termination
    append_termination_with_gadget(output);

    // return written chars without terminating \0
    return (int)output->pos;
}
```

* 之后最关键的函数来了，调用`format_string_loop`，这个函数会分析format，然后将输出传递给out函数，out函数定义见后

```c
static inline void format_string_loop(output_gadget_t *output, const char *format, va_list args)
{
    while (*format) {
        if (*format != '%') {
            // A regular content character
            putchar_via_gadget(output, *format);
            format++;
            continue;
        }
        // We're parsing a format specifier: %[flags][width][.precision][length]
        ADVANCE_IN_FORMAT_STRING(format);

        printf_flags_t flags = parse_flags(&format);

        // evaluate width field
        printf_size_t width = 0U;
        if (is_digit_(*format)) {
            width = (printf_size_t)atou_(&format);
        } else if (*format == '*') {
            const int w = va_arg(args, int);
            if (w < 0) {
                flags |= FLAGS_LEFT; // reverse padding
                width = (printf_size_t)-w;
            } else {
                width = (printf_size_t)w;
            }
            ADVANCE_IN_FORMAT_STRING(format);
        }

        // evaluate precision field
        printf_size_t precision = 0U;
        if (*format == '.') {
            flags |= FLAGS_PRECISION;
            ADVANCE_IN_FORMAT_STRING(format);
            if (is_digit_(*format)) {
                precision = (printf_size_t)atou_(&format);
            } else if (*format == '*') {
                const int precision_ = va_arg(args, int);
                precision = precision_ > 0 ? (printf_size_t)precision_ : 0U;
                ADVANCE_IN_FORMAT_STRING(format);
            }
        }

        // evaluate length field
        switch (*format) {
                
```

out函数是通过__out_wrapper传递给之前的函数的。

```c
static void __out_wrapper(char c, void *arg)
{
    out(arg, &c, 1);
}
static int out(FILE *f, const char *s, size_t l)
{
    int ret = 0;
    for (size_t i = 0; i < l; i++) {
        char c = s[i];
        f->buf[f->buffer_len++] = c;
        if (f->buffer_len == FILE_BUF_SIZE || c == '\n') {
            int r = __write_buffer(f);
            __clear_buffer(f);
            if (r < 0)
                return r;
            if (r < f->buffer_len)
                return ret + r;
            ret += r;
        }
    }
    return ret;
}
```

其中`__write_buffer`函数调用了一个rust函数`ax_print_str`，将缓冲区的字符串输出。

```rust
/// Print a string to the global standard output stream.
#[no_mangle]
pub unsafe extern "C" fn ax_print_str(buf: *const c_char, count: usize) -> c_int {
    ax_call_body_no_debug!({
        if buf.is_null() {
            return Err(LinuxError::EFAULT);
        }

        let bytes = unsafe { core::slice::from_raw_parts(buf as *const u8, count as _) };
        let len = io::stdout().write(bytes)?;
        Ok(len as c_int)
    })
}
```

如果是别的FILE，则会调用`ax_write(fd, buf, count)`，将buf写入fd对应的文件中

## qsort原理

* `qsort`函数：

base：数组基地址

nel：数组元素个数

width：元素所占字节数

cmp：比较函数

```c
void qsort(void *base, size_t nel, size_t width, cmpfun_ cmp)
{
	__qsort_r(base, nel, width, wrapper_cmp, (void *)cmp);
}
static int wrapper_cmp(const void *v1, const void *v2, void *cmp)
{
	return ((cmpfun_)cmp)(v1, v2);
}
```

* 接着调用`__qsort_r`函数，其中cmp为wrapper_cmp，arg为原先的cmp函数。

```c
void __qsort_r(void *base, size_t nel, size_t width, cmpfun cmp, void *arg)
{
	size_t lp[12*sizeof(size_t)];
	size_t i, size = width * nel;
	unsigned char *head, *high;
	size_t p[2] = {1, 0};
	int pshift = 1;
	int trail;

	if (!size) return;
	// head指向数组头部，high指向数组最后一个元素
	head = base;
	high = head + size - width;

	/* Precompute Leonardo numbers, scaled by element width */
	for(lp[0]=lp[1]=width, i=2; (lp[i]=lp[i-2]+lp[i-1]+width) < size; i++);

	while(head < high) {
		if((p[0] & 3) == 3) {
			sift(head, width, cmp, arg, pshift, lp);
			shr(p, 2);
			pshift += 2;
		} else {
			if(lp[pshift - 1] >= high - head) {
				trinkle(head, width, cmp, arg, p, pshift, 0, lp);
			} else {
				sift(head, width, cmp, arg, pshift, lp);
			}

			if(pshift == 1) {
				shl(p, 1);
				pshift = 0;
			} else {
				shl(p, pshift - 1);
				pshift = 1;
			}
		}

		p[0] |= 1;
		head += width;
	}

	trinkle(head, width, cmp, arg, p, pshift, 0, lp);

	while(pshift != 1 || p[0] != 1 || p[1] != 0) {
		if(pshift <= 1) {
			trail = pntz(p);
			shr(p, trail);
			pshift += trail;
		} else {
			shl(p, 2);
			pshift -= 2;
			p[0] ^= 7;
			shr(p, 1);
			trinkle(head - lp[pshift] - width, width, cmp, arg, p, pshift + 1, 1, lp);
			shl(p, 1);
			p[0] |= 1;
			trinkle(head - width, width, cmp, arg, p, pshift, 1, lp);
		}
		head -= width;
	}
}
```

## fnmatch原理

```c
int fnmatch(const char *pattern, const char *string, int flags);
```

该函数检查`string`是否匹配`pattern`。

输入参数：

`pattern`是一个通配符：

* `?`：一个字符
* `*`：任意字符串，包括空字符串
* `[...]`：表示匹配`...`中的一个字母，如果`...`中的第一个字母为`!`，则表示匹配除了`...`中所有字母的字母

flags则是一些选项，修改匹配规则

返回值：

如果匹配，则返回0

* `pat_next`函数

参数：

step：步长，表示该函数已匹配的长度，也就是下一步pattern可以向后移的长度

m：pat模式串的参考长度，主要用于`[]`的匹配

描述：

获取pattern的一个有效字符。

如果是普通字符，则返回该字符，步长为1

如果目前是反斜杠，则返回转义后的字符；步长为2

* `fnmatch_internal`函数

描述：

检查给定长度的pat和str是否匹配

1. 尝试不断地匹配，直到遇到`*`（其实难点就是`*`的处理）
2. 找到pattern中最后一个`*`的位置
3. 检查pattern最后一个`*`之后的子串与str相应位置的子串是否匹配
4. 开始匹配所有的`*`，如果当前pattern是个`*`，则跳过这个`*`，并一直停留在`*`的下一个字符；如果当前str不匹配pattern，则跳过这个字符。也就是说，如果str匹配上`*`的后一个字符了，就继续匹配，否则str一直前进。

* `match_bracket`函数

参数：

p ：pattern，指向`[`

k：要匹配的字符

kfold：要匹配字符的大小写折叠后的形式

描述：

查看是否字符与pattern匹配，如果匹配返回1，否则返回0

## setenv原理

环境变量一整套都在`env.c`中，环境变量名中不允许包含`=`。提供了3个接口函数，getenv、setenv、unsetenv

- [ ] `__env_rm_add`函数的作用是什么？删去可以吗？

## rename原理

```c
int rename(const char *__old, const char *__new)
{
    return ax_rename(__old, __new);
}
```

* `ax_rename`

位于ulib/axlibc/src/file.rs

```rust
/// Rename `old` to `new`
/// If new exists, it is first removed.
///
/// Return 0 if the operation succeeds, otherwise return -1.
#[no_mangle]
pub unsafe extern "C" fn ax_rename(old: *const c_char, new: *const c_char) -> c_int {
    ax_call_body!(ax_rename, {
        let old_path = char_ptr_to_str(old)?;
        let new_path = char_ptr_to_str(new)?;
        debug!("ax_rename <= old: {:?}, new: {:?}", old_path, new_path);
        axstd::fs::rename(old_path, new_path)?;
        Ok(0)
    })
}
```

* `axstd::fs::rename`

位于modules/axfs/src/api

```rust
pub fn rename(old: &str, new: &str) -> io::Result<()> {
    crate::root::rename(old, new)
}
```



## remove原理

参见unlink(2)和rmdir(2)

axfs::api中的两个函数应该可以使用：

```rust
/// Removes an empty directory.
pub fn remove_dir(path: &str) -> io::Result<()> {
    crate::root::remove_dir(None, path)
}

/// Removes a file from the filesystem.
pub fn remove_file(path: &str) -> io::Result<()> {
    crate::root::remove_file(None, path)
}
```

### mkdir

```c
int mkdir(const char *pathname, mode_t mode);
```



## fdopen函数

### fopen函数

```c
FILE *fopen(const char *filename, const char *mode)
{
    FILE *f;
    int flags;

    if (!strchr("rwa", *mode)) {
        errno = EINVAL;
        return 0;
    }

    f = (FILE *)malloc(sizeof(FILE));

    flags = __fmodeflags(mode);
    // TODO: currently mode is unused in ax_open
    int fd = ax_open(filename, flags, 0666);
    if (fd < 0)
        return NULL;
    f->fd = fd;

    return f;
}
```

* ax_open函数

```rust
/// Open a file by `filename` and insert it into the file descriptor table.
///
/// Return its index in the file table (`fd`). Return `EMFILE` if it already
/// has the maximum number of files open.
#[no_mangle]
pub unsafe extern "C" fn ax_open(
    filename: *const c_char,
    flags: c_int,
    mode: ctypes::mode_t,
) -> c_int {
    let filename = char_ptr_to_str(filename);
    debug!("ax_open <= {:?} {:#o} {:#o}", filename, flags, mode);
    ax_call_body!(ax_open, {
        let options = flags_to_options(flags, mode);
        let file = options.open(filename?)?;
        File::new(file).add_to_fd_table()
    })
}

macro_rules! ax_call_body {
    ($fn: ident, $($stmt: tt)*) => {{
        #[allow(clippy::redundant_closure_call)]
        let res = (|| -> axerrno::LinuxResult<_> { $($stmt)* })();
        match res {
            Ok(_) | Err(axerrno::LinuxError::EAGAIN) => debug!(concat!(stringify!($fn), " => {:?}"),  res),
            Err(_) => info!(concat!(stringify!($fn), " => {:?}"), res),
        }
        match res {
            Ok(v) => v as _,
            Err(e) => {
                crate::errno::set_errno(e.code());
                -1 as _
            }
        }
    }};
}
```

### fdopen函数

```c
FILE *fdopen(int fd, const char *mode)
```

描述：fdopen将一个流和一个fd关联起来，其中流打开方式mode必须与fd的mode兼容。流意思就是可以通过标准IO函数对文件进行操作。

参数：

* mode：

r：只读

r+：读+写

w：对于fdopen，不会将文件大小置为0

w+：读+写，对于fdopen，不会将文件大小置为0

a：append模式

a+: 读+append模式

FILE结构体就是IO_FILE：

```c
struct IO_FILE {
    int fd;
    // 缓冲区，buffer_len为缓冲区的有效长度，buf为缓冲区
    uint16_t buffer_len;
    char buf[FILE_BUF_SIZE];
};
```

目前fdopen的实现，就是为FILE分配空间，然后初始化fd和buffer_len域，并根据mode修改fd对应的权限。

* fdopen的作用

[C语言fdopen()函数:将流与文件句柄连接 - C语言网 (dotcpp.com)](https://www.dotcpp.com/course/457)

可以通过FILE流，调用fprintf函数，将内容写入到文件中。

### freopen函数

```c
FILE *freopen(const char *restrict filename, const char *restrict mode, FILE *restrict f)
```

打开filename所在的文件，并将f与该文件相关联。如果f原来存在，则关闭。

参数：

* filename：如果为空，则修改f的mode为参数mode

## 文件系统C函数

### open

```c
int open(const char *pathname, int flags, mode_t mode);
```

描述：通过pathname打开文件，如果flags中包含`O_CREAT`且文件不存在，则创建文件。

参数：

* flags：必须包含access modes之一：O_RDONLY,  O_WRONLY,  or  O_RDWR.

  file creation flags和file status flags都可以作为flags。

返回值：一个文件描述符

* 文件描述符

open函数会创建一个新的open file description，一个open files table的表项。open file description记录了文件偏移量和file status flags。文件描述符则是open file description的索引。

* file creation flags

file creation flags影响open 操作本身的语义，而file status flags影响接下来IO操作的语义。

file creation flags包含：

O_CLOEXEC：执行该文件时关闭文件描述符

O_CREAT：若路径不存在则创建该文件。open函数中的mode参数则指定该文件的属性，包括user的权限、group的权限、others的权限

O_DIRECTORY：如果pathname不是目录，则Open执行失败

O_EXCL：确保open会创建文件，需要与O_CREAT联用。如果pathname已经存在，则会失败返回EEXIST

O_NOCTTY：

O_NOFOLLOW：

O_TMPFILE：创建一个未命名的临时普通文件，pathname指定所在目录。

O_TRUNC：若文件已经存在且是普通文件，且access mode允许写（O_RDWR或O_WRONLY），则文件长度被截断为0.

* file status flags

文件状态标志位，包含：

O_APPEND：文件以append模式打开，执行write函数时，文件偏移量会自动更改到文件结尾，将写入的内容追加到文件中



### fcntl

根据cmd对fd进行操作。

```c
int fcntl(int fd, int cmd, ... /* arg */ );
```

### dup

```c
int dup(int oldfd);
```

复制oldfd到一个新的fd。这两个fd之后可以一起使用，指向同一个文件，共享文件偏移量和文件状态。不过并不共享close-on-exec标志位。

```c
int dup2(int oldfd, int newfd);
```

功能和dup类似，但指定了newfd，且如果之前newfd已经被使用，则关闭该newfd。

```c
int dup3(int oldfd, int newfd, int flags);
```

可以通过flags来对newfd设定close-on-exec位。

返回值：失败返回-1，成功返回新文件描述符

## 网络栈接口函数

### 总体介绍

网络栈应用程序的函数调用顺序如下面两张图：

![img](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202308121618023.webp)

![img](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202308121618366.webp)

下面介绍的函数，一般的调用逻辑为：

1. 调用axlibc中对应的ax_xxx函数
2. 由ax_xxx函数调用axlibc中Socket中对应的函数
3. 对应的函数再向下调用axnet中TcpSocket或UdpSocket的函数
4. 再向下调用rust库中socket对应的函数

应用程序中IP地址结构体：

```c
struct sockaddr_in {
    sa_family_t sin_family;
    in_port_t sin_port; // 端口号
    struct in_addr sin_addr;// 二进制形式的ip地址
    uint8_t sin_zero[8];
};
```

### inet_pton

```c
int inet_pton(int af, const char *src, void *dst);
```

将IP地址从字符串形式转换为二进制形式，p和n分别代表表达（presentation)和数值（numeric)，地址的表达格式通常是ASCII字符串，数值格式则是存放到套接字地址结构的二进制值。

参数：

* af：AF_INET，表示IPv4地址；AF_INET6，表示IPv6地址
* src：一个字符串，表示一个IP地址
* dst：一个地址结构体，是src地址的二进制形式

返回值：成功返回1，否则0或-1

### socket

```c
int socket(int domain, int type, int protocol);
```

创建一个通信端点，返回一个指向该端点的文件描述符。

参数：

* domain：通信域，例如AF_INET表示IPv4协议
* type：通信类型，SOCK_DGRAM表示无连接的数据报（UDP）; SOCK_STREAM提供连续、可信、双向、基于连接的字节流
* protocol：socket使用的协议，如UDP、TCP

#### UDPSocket

向下调用`ax_socket`函数：

如果是Udp类型的 Socket，则执行：

```rust
Socket::Udp(Mutex::new(UdpSocket::new())).add_to_fd_table()
```

创建的UdpSocket结构体为：

```rust
/// A UDP socket that provides POSIX-like APIs.
pub struct UdpSocket {
    handle: SocketHandle,// 本质上是一个数字，表示Socket的索引
    local_addr: RwLock<Option<IpEndpoint>>,// 表示本地IP地址和端口号的endpoint
    peer_addr: RwLock<Option<IpEndpoint>>,
    nonblock: AtomicBool,// 初始时为false
}
```

#### TcpSocket

如果是Tcp类型的Socket，那么创建的TcpSocket结构体为：

```rust
pub struct TcpSocket {
    state: AtomicU8,
    handle: UnsafeCell<Option<SocketHandle>>,
    local_addr: UnsafeCell<IpEndpoint>,
    peer_addr: UnsafeCell<IpEndpoint>,
    nonblock: AtomicBool,
}
```

> socket有两种模式，阻塞式和非阻塞式。如果发送缓冲区已满或接收缓冲区为空，调用发送或接收函数时，阻塞式会将程序阻塞在调用处，非阻塞式会返回错误码。

Socket::Udp则为：

```rust
// axlibc/socket.rs
pub enum Socket {
    Udp(Mutex<UdpSocket>),
    Tcp(Mutex<TcpSocket>),
}
```

`add_to_fd_table()`函数则将Socket加入到FD_TABLE中，Socket实现了FileLike特征。

### bind

```c
int bind(int sockfd, const struct sockaddr *addr,
            socklen_t addrlen);
```

将IP地址addr与socket进行绑定。其中sockaddr的定义如下：

```c
struct sockaddr {
    sa_family_t sa_family;
    char sa_data[14];
};
```

`bind`函数调用了`ax_bind`函数，为：

```rust
let addr = from_sockaddr(socket_addr, addrlen)?;
Socket::from_fd(socket_fd)?.bind(addr)?;
```

底层调用了`from_sockaddr`，将sockaddr类型的地址转换为SocketAddr：

```rust
// .cargo中的
pub enum SocketAddr {
    /// An IPv4 socket address.
    V4(SocketAddrV4),
    /// An IPv6 socket address.
    V6(SocketAddrV6),
}
pub struct SocketAddrV4 {
    ip: Ipv4Addr,// octets: [u8; 4]
    port: u16,
}
```

最后调用socket的bind函数，将socket与socketAddr进行绑定。socket的bind函数又会调用UdpSocket的bind函数，将最底层的socket与endpoint（IP地址+端口号）绑定。

### recvfrom

```c
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
```

用于从socket中接收消息。

参数：

* sockfd：socket的id
* buf：接收消息缓冲区
* len：缓冲区大小
* src_addr：发送方地址

返回值：接收到信息的长度。

该函数向下调用到Socket的recvfrom函数，Socket就是C程序与底层逻辑的中间层。

Socket又向下调用axnet中UdpSocket的recv_from函数。该函数也调用了rust库中的socket的接收函数。

### send

```c
ssize_t send(int fd, const void *buf, size_t n, int flags)
```

发送消息，只可以用于基于连接的套接字，也就是说接收方的地址信息等是已知的。暂时不理会flags

```rust
if buf_ptr.is_null() {
    return Err(LinuxError::EFAULT);
}
let buf = unsafe { core::slice::from_raw_parts(buf_ptr as *const u8, len) };
Socket::from_fd(socket_fd)?.send(buf)
```

对于tcpsocket，通过调用rust库的send_slice函数。

### sendto

```c
ssize_t sendto(int fd, const void *buf, size_t n, int flags, const struct sockaddr *addr, socklen_t addr_len)
```

在socket上给指定地址发送消息。

参数

* buf：发送缓冲区；n：发送缓冲区长度
* addr：目的地址

返回值：如果发送成功，返回发送的比特数。

向下调用ax_sendto函数，再调用axlibc中socket的sendto函数，再调用axnet中UdpSocket的send_to函数。

### listen

```c
int listen(int sockfd, int backlog);
```

监听连接请求，将指定的socket标记为被动的，可以通过`accept`接收连接请求。

返回值：成功返回0，否则-1

该函数向下调用至axlibc中socket的listen函数，继续向下调用axnet中TcpSocket的listen函数，在该函数中调用LISTEN_TABLE的listen函数，以端口号为索引向ListenTable的表项中设立相关信息。

```rust
pub struct ListenTable {
    tcp: Box<[Mutex<Option<Box<ListenTableEntry>>>]>,
}
struct ListenTableEntry {
    listen_endpoint: IpListenEndpoint,// 端点，监听方的ip和端口号
    syn_queue: VecDeque<SocketHandle>,
}
```

### accept

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

获取一个连接。该函数从监听socket的连接请求队列中获取一个连接请求，并创建一个新的socket，并返回新的socket的fd。

返回值：成功则返回新socket的fd，否则-1

和前面函数同样的逻辑，首先调用到axlibc中Socket的accept函数，然后向下调用到axnet中TcpSocket的accept函数。然后调用LISTEN_TABLE的accept函数。

当listen socket收到一个请求时，会在SOCKET_SET中增加一个socket，里面就包含请求的ip等信息，并在LISTEN_TABLE中对应表项的队列中填入该socket的handler

### getaddrinfo

```c
int getaddrinfo(const char *node, const char *service,
const struct addrinfo *hints,
struct addrinfo **res);
```

域名解析函数。给定node或service，该函数解析node或service，一般用于主机名向IPv4地址的解析，并通过res返回一个或多个addrinfo，addrinfo包含一个IP地址，可用于bind或connect。

参数：

* node：一个IP地址或主机名，如果是主机名则需要解析
* service：用来设置端口。node和service只能有1个为空
* hints：该结构体的几个域会限定返回res的addrinfo必须满足的条件

返回值：成功返回0

该函数调用axlibc中的ax_getaddrinfo函数，如果node本身就是点分二进制形式，则直接返回该地址；如果node是主机名，则调用axnet中的dns_query函数，该函数会继续向下调用到Rust库中的query函数，完成查询

```rust
// 创建一个DnsSocket，
let socket = DnsSocket::new(); 
socket.query(name, DnsQueryType::A)
```

### getsockopt/setsockopt

```c
int getsockopt(int sockfd, int level, int optname,
void *optval, socklen_t *optlen);

int setsockopt(int sockfd, int level, int optname,
const void *optval, socklen_t optlen);
```

获取和设定socket的选项。选项可能分布在多个协议层。要操作socket的选项，则需要给定选项所在的协议层level和选项名。

对于setsockopt，optval和optlen则是用于获取选项的值；getsockopt则是缓冲区用于存放获取到的选项的值。

参数：

* level：如果选项位于socket层，level为SOL_SOCKET；TCP层，level为xxx

#### socket层的optname

* SO_ACCEPTCONN



### sendmsg

```c
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
```

For sendmsg(), the address of the target is given by msg.msg_name, with msg.msg_namelen specifying its size.

For sendmsg(),
       the  message is pointed to by the elements of the array msg.msg_iov.

msg.msg_name：目的地址

msg.msg_namelen：目的地址大小

```c
struct msghdr {
   void         *msg_name;       /* Optional address */
   socklen_t     msg_namelen;    /* Size of address */
   struct iovec *msg_iov;        /* Scatter/gather array */
   size_t        msg_iovlen;     /* # elements in msg_iov */
   void         *msg_control;    /* Ancillary data, see below */
   size_t        msg_controllen; /* Ancillary data buffer len */
   int           msg_flags;      /* Flags (unused) */
};
```

msg_name：对于无连接的socket，用于指示数据报的目的地址。对于面向连接的Socket，应为NULL

msg_namelen：msg_name的长度。

* 辅助数据

[sendmsg和recvmsg的应用----在进程之间传递描述符_城南花已开.jpg的博客-CSDN博客](https://blog.csdn.net/qq_39781096/article/details/107666399)

msg_control指向辅助数据缓冲区，msg_controllen为该缓冲区的大小。

辅助数据缓冲区可能包含多个辅助数据，每个辅助数据前有一个头部，是一个struct cmsghdr结构体。该结构体和辅助数据之间可能存在填充字节。

```c
struct cmsghdr {
   size_t cmsg_len;    //辅助数据字节数，包含该头部
   int    cmsg_level;  /* Originating protocol */
   int    cmsg_type;   /* Protocol-specific type */
};
```



#### writev

```c
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

该函数将iovcnt个数据缓冲区（iov）写入fd所代表的文件，称为gather output。

数据缓冲区在内存顺序上是连续的

## 环境变量

### relibc实现

在platform中创建全局变量：

```rust
#[allow(non_upper_case_globals)]// 不要对非大写全局变量进行警告
#[no_mangle]
pub static mut environ: *mut *mut c_char = ptr::null_mut();
```

null_mut创建一个NULL指针。

对环境变量的初始化工作在这里完成:

```rust
if platform::environ.is_null() {
    // Set up envp
    let envp = sp.envp();
    let mut len = 0;
    while !(*envp.add(len)).is_null() {
        len += 1;
    }
    // 将环境变量以数组的形式保存在OUR_ENVIRON中
    platform::OUR_ENVIRON = copy_string_array(envp, len);
    platform::environ = platform::OUR_ENVIRON.as_mut_ptr();
}
```

#### 接口函数

* *环境变量迭代器*

environ_iter

* find_env，提供给rust内部函数使用

```rust
unsafe fn find_env(search: *const c_char) -> Option<(usize, *mut c_char)>
```

*查找环境变量为search的环境变量(xxx=xx或xxx)，返回所在environ的索引和环境变量值指针*

* getenv，可提供给C程序使用

```rust
#[no_mangle]
pub unsafe extern "C" fn getenv(name: *const c_char) -> *mut c_char
```

和linux手册上的getenv相同。

* setenv

```rust
#[no_mangle]
pub unsafe extern "C" fn setenv(
    key: *const c_char,
    value: *const c_char,
    overwrite: c_int,
) -> c_int
```

* put_new_env

*将环境变量insert插入到environ中*

```rust
unsafe fn put_new_env(insert: *mut c_char)
```

* putenv，可提供给C程序

### 实现思路

- [x] 在axruntime的lib.rs中定义环境变量的全局变量

```rust
#[allow(non_upper_case_globals)]
#[no_mangle]
pub static mut environ: *mut *mut c_char = ptr::null_mut();
pub static mut OUR_ENVIRON: Vec<*mut c_char> = Vec::new();
```

- [x] 创建环境变量迭代器environ_iter

- [x] 实现rust类型的函数

- [x] C程序中不设置环境变量了，env.c中的函数调用下层的环境变量rust函数.

实现起来的问题：

- [x] 如何将内核中的环境变量暴露给C程序？
- [x] 要不C程序版的环境变量也用rust实现给上面提供个接口？这两种方案选一个实现就行。
- [x] environ是不是得足够大呀？通过Vec实现的，不怕。
- [x] 看懂pub use；rust的包、package精通
- [x] 写今天的日记

### 从C类型改造成rust的计划

工作量太大了，先搁置了吧
