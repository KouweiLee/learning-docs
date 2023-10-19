# Ruby学习

## 基础语法

###  基本点

Ruby 把分号和换行符解释为语句的结尾。

ruby中，一切皆对象。

判断一个对象是否是Nil：name.nil?

### 分支判断

#### if语句

例如：

```ruby
x=1
if x > 2
   puts "x 大于 2"
elsif x <= 2 and x!=0
   puts "x 是 1"
else
   puts "无法得知 x 的值"
end
```

只有false和nil才是假，其他都是真。

任何语句后面，都可以增加一个条件判断语句，只有该语句成立，前面的语句才会执行。例如：

```ruby
hhh = "happy"
puts "hello world" if hhh == "happy"
```

#### case语句

```ruby
$age =  5 # $开头的变量表示全局变量
case $age
when 0 .. 2
    puts "婴儿"
when 3 .. 6
    puts "小孩"
when 7 .. 12
    puts "child"
when 13 .. 18
    puts "少年"
else
    puts "其他年龄段的"
end
```

case在比对两个对象时，不是用==比对的，而是`===`。任何对象只要重载了这个case equality运算符，则可以放在case中进行比较。

两个点表示双闭区间，三个点表示右边开区间

`==`判断时，不管数值的类型，最松。但`eql?`在意数值的类型。`equal?`比较的则是是否是同一个对象，最严格。

要小心使用not，and和or，因为他们的优先级和别的语言不太一样。

用冒号引导的字符串是符号，用于Hash索引。

### 循环

* while

```
while $i < $num  do
   puts("在循环语句中 i = #$i" )
   $i +=1
end
```

* for

```
for i in 0..5
   puts "局部变量的值为 #{i}"
end
```

* 遍历数组

```ruby
a = [1,2,3]
a.each{|x| puts x} # 依次遍历每个元素，执行代码块的内容
a.collect{|x| x**2} #依次遍历每个元素，将代码块的结果收集成一个新的数组
```

能调用each的对象，表示该对象是可以迭代的。判断能否调用某个函数可以通过respond_to?方法查看：

```ruby
@names.respond_to?("each")
```

each是一个可以接收代码块的函数，例如：

```ruby
@names.each do |name|
  puts "Hello #{name}!"
end
```

### 数组

* map
* inject
* 

### 函数

例子：

```ruby
def h(name = "World")
	puts "Hello #{name.capitalize}!"
end
```

## 函数式语言特征

Lambda、block、Proc等，见ppt4

## 面向对象

```ruby
class Greeter
	def initialize(name="world") # 调用new方法时会调用该函数
        @name = name
    end
    def say_hi
        puts "Hi #{@name}"
    end
end
g = Greeter.new("Pat")
g.say_hi
```

@name定义了一个成员变量，默认是对外隐藏的。如果想要改变name属性，则需要在Greeter中加：

```ruby
attr_accessor :name
```

这会默认为Greeter定义两个方法，`name`和`name=`，前者用于读取，后者用于赋值

* 类数据

用@@表示所有类实例之间共享的变量。

方法前加self表示是类上的方法，调用必须用类名调用

## 一些接口函数

### 过滤器

* select，感觉类似于rust中的filter

```ruby
letters = ['A', 'k', 'L', 'B', 'n', 'O', 'P', 'c', 'r']
small_letters = letters.select { |l| l == l.downcase }
puts small_letters
```

* reject，与select正好相反，如果满足条件，则reject

```ruby
letters = ['A', 'k', 'L', 'B', 'n', 'O', 'P', 'C', 'r']
small_letters = letters.reject { |l| l != l.downcase }
puts small_letters
```

* inject

和reduce一样，用于将一个集合转换为一个值。有多种用法：

根据symbol执行reduce操作：

```ruby
inject(symbol) → object
(1..4).inject(:+)     # => 10
```

在symbol的基础上，增加一个初始元素，这个初始元素会被加到集合的开头：

```ruby
inject(initial_operand, symbol) → object
(1..4).inject(10, :+) # => 20
```

接收一个block而不是symbol，其中memo在第一次迭代时是集合的第一个元素，第二次之后则是block的上一次输出结果；而operand在第一次迭代时是集合的第二个元素，第二次之后则是集合的依次各个元素：

```ruby
inject {|memo, operand| ... } → object
(1..4).inject {|sum, n| sum + n*n }    
#1+2*2+3*3+4*4 => 30
```

在block的基础上增加一个初始元素：

```ruby
inject(initial_operand) {|memo, operand| ... } → object
(1..4).inject(2) {|sum, n| sum + n*n } # => 32
#2+1*1+2*2+3*3+4*4 => 30
```

inject的别名就是reduce

* all?

如果所有元素均满足条件，则返回true；否则false

```ruby
(2..Math.sqrt(n)).all?{ |x| n % x > 0 }
```

