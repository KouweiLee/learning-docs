# Github actions

CI：catch错误；CD：发布东西

它是一种Github上的CI/CD。当PR或Issue提交时，可以触发。

## 语法

存储在`.github/workflows/`。

* Runner

runner是一个虚拟机，运行workflow。每个runner一次可以运行一个job。每个workflow会运行在一个全新的虚拟机中。

* workflows

一个自动化执行流，包含多个jobs。它被定义在`.github/workflows`文件中，它是一个YAML文件。

* Events

触发workflow执行的事件，比如PR提交。

* Jobs

  Job是workflow中的一系列steps：

  * step可以是一个脚本，也可以是一个action。steps是顺序执行的，并在同一个runner中，因此可以互通数据。

  **Jobs默认情况下是并行执行的。也可以通过needs配置它们之间的依赖关系。**

* Action

action是github提供的一个可以直接使用的程序，可以是一个公开仓库里的，也可以是这里的[github.com/actions](https://github.com/actions)里。可以直接引用：

```
actions/setup-node@74bc508 # 指向一个 commit
actions/setup-node@v1.0    # 指向一个标签
actions/setup-node@master  # 指向一个分支
```

### 使用dependabot定时更新action tag

在`.github`下增加`dependabot.yml`文件，内容为：

```yaml
version: 2
updates:
  # Maintain dependencies for GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

即可每周更新workflow使用的github action，例如`@v2`更新为`@v3`。

## some tricks

### 运行一个任务的多个变体

通过matrix strategy，可以实现在一个Job中，定义多个数组变量，实现数组变量的排列组合，从而实现一个任务的多个变体任务的创建和运行。例如：

```yaml
jobs:
  example_matrix:
  	# 定义一个strategy，这会产生2*3个任务同时运行，通过${{ matrix.属性名 }}，即可获取当前的os和version变量
    strategy:
      matrix:
        os: [ubuntu-22.04, ubuntu-20.04]
        version: [10, 12, 14]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.version }}
```



## 示例学习

```yml
name: Test CI

on: [push, pull_request]
# 定义一个全局env变量
env:
  qemu-version: 7.1.0
  rust-toolchain: nightly-2023-09-01

jobs:
  # unit-test是<job_id>
  unit-test:
    runs-on: ubuntu-latest
    # steps的值是一个数组
    steps:
    # uses使用一个action
    - uses: actions/checkout@v3
      # with指定action的input parameters
      with:
        submodules: recursive
    - uses: actions-rs/toolchain@v1
      with:
        profile: minimal
        toolchain: ${{ env.rust-toolchain }}
        components: rust-src, llvm-tools-preview
    # name和run是同一个数组元素的两个键
    - name: Run unit tests
      run: make unittest_no_fail_fast
```



## yaml语法

缩进一般使用2个空格。`:`后需要加一个空格。

* 标量：

```yaml
number-value: 42
floating-point-value: 3.141592
boolean-value: true # on, yes -- also work
# strings can be both 'single-quoted` and "double-quoted"
string-value: 'Bonjour'
unquoted-string: Hello World
hexadecimal: 0x12d4
scientific: 12.3015e+05
infinity: .inf
not-a-number: .NAN
null: ~
another-null: null
key with spaces: value
datetime: 2001-12-15T02:59:43.1Z
datetime_with_spaces: 2001-12-14 21:59:43.10 -5
date: 2002-12-14

```

* 数组

数组的元素由`- `（-后要加一个空格）表示：

```
jedis:
  - Yoda
  - Qui-Gon Jinn
  - Obi-Wan Kenobi
  - Luke Skywalker
```

元素也可以是一个复杂的结构，例如：

```
- 
  name: Mark McGwire
  hr:   65
  avg:  0.278
- 
  name: Sammy Sosa
  hr:   63
  avg:  0.288
```

表示有一个列表，包含两个元素，每个元素是一个键值对集合。

* 字典

jedi就是一个字典，字典的元素为`key: value`的格式，注意冒号后面需要有一个空格：

```
jedi:
  name: Obi-Wan Kenobi
  home-planet: Stewjon
  species: human
  master: Qui-Gon Jinn
  height: 1.82m
```

* 数组和字典的内联语法

yaml是json的超集，因此支持json格式的数组和字典：

```
episodes: [1, 2, 3, 4, 5, 6, 7]
best-jedi: {name: Obi-Wan, side: light}
```

* 多行字符串

操作符`|`和`>`表示这是一个多行字符串，其中`|`表示之后的换行符都是真的，`>`表示这多行字符串其实都在一行，换行符是假的。

在这两个操作符之后，还可以增加一个操作符：`+`或`-`，分别表示该字符串的最后是否有一个换行符，`+`表示有而`-`表示没有：

```
folded_no_ending_newline:
  script:
    - >-
      echo "foo" &&
      echo "bar" &&
      echo "baz"


    - echo "do something else"

unfolded_ending_single_newline:
  script:
    - |
      echo "foo" && \
      echo "bar" && \
      echo "baz"


    - echo "do something else"
```

* 嵌套结构

```yaml
requests:
  # first item of `requests` list is just a string
  - http://example.com/

  # second item of `requests` list is a dictionary
  - url: http://example.com/
    method: GET
```

因此数组不要求元素类型一致。

* 注释

用`#`表示。

* 参考资料

https://ruanyifeng.com/blog/2016/07/yaml.html

## 参考资料

1. [GitHub Actions 入门教程](https://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)
2. [trigger workflows的events](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#pull_request)

3. [actions市场](https://github.com/marketplace?type=actions)
4. [**github workflow语法**](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)
5. ubuntu22.04预装的软件：https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2204-Readme.md