---
title: vlang语言特性探究
date: 2019-10-17 18:00
tags: [vlang,c]
categories: language
---
话说长江后浪推前浪，前浪被拍在沙滩上。golang语言在kubernetes项目中大放异彩之后，俨然成为容器圈的宠儿。在Google,Facebook等巨头公司开源其人工智能领域的项目后，python语言逐渐成为了人工智能领域的专属语言......可见编程语言界的风向是由具有影响力的巨头公司所引领。山雨欲来风满楼，此时处于萌芽状态的vlang语言正在黑暗中摩拳擦掌、跃跃欲试，想要在编程语言界内占得一席之地。

下面我们来看一下vlang语言有哪些特性。
<!--more-->
## 环境准备
作为容器的拥趸，选择用docker方式进行安装
### 数据库（为了验证orm特性）
```bash postgres
mkdir $PWD/my_dbdata
docker run -d --name my_postgres -v my_dbdata:/var/lib/postgresql/data -p 54320:5432 postgres:11
docker exec -it my_postgres psql -U postgres -c "create database my_database"
```
### vlang
```bash
git clone https://github.com/vlang/v
cd v
docker build -t vlang .
mkdir ~/project/vlang-test && cd ~/project/vlang-test
docker run --rm -v $PWD:/home -it vlang:latest
```

## 特性对比vs golang

1、仅有一种初始化变量方式(:=)
```golang
name := 'Bob'
age := 20
mut num := 20
```
2、没有全局变量概念;初始化变量默认不可变，除非加上mut前缀

3、int类型总是代表32位
i8 i16 int i64 i128(soon)
byte u16 u32 u64 u128(soon)

4、数组append
```golang
mut nums := [1,2,3]
nums << [5,6,7]
println(nums)  //[1,2,3,5,6,7]
ids := [0].repeat(50) // This creates an array with 50 zeros
```

5、Maps
Only maps with string keys are allowed for now 
```golang
mut m:= map[string]int
```

6、If For 操作与golang基本无差别，In操作与python类似,Match操作与其它语言switch类似，如下：
```golang
os := 'windows'
match os {
	'darwin' => { println('macOS.') }
	'linux' => { println('Linux.') }
	else => { println(os) }
}
```

7、struct与golang类似，支持嵌套。成员默认是私有不可变，可加pub和mut进行属性更改
```golang
struct Foo {
	a int     // private immutable (default)
mut:
	b int     // private mutable
	c int     // (you can list multiple fields with the same access modifier) 
pub:
	d int     // public immmutable (readonly)
pub mut:
	e int     // public, but mutable only in parent module 
pub mut mut:
	f int 	  // public and mutable both inside and outside parent module 
}
```
8、由结构体实现的方法与golang类似
9、高阶函数与golang类似
10、V references are similar to Go pointers and C++ references.
11、内建println、print,可通过.str() srting方法进行定制
12、模块
```golang
//mymodule.v
module mymodule

pub fn say_hi() {
    print("hello world")
}

//main.v
module main
import mymodule
fn main() {
    mymodule.say_hi()
}
```

### 13、vlang也实现了golang语言极具标识的Interfaces机制，两者用法基本上一样。


### 14、泛型编程：相比于golang使用interface机制实现的桥接模式的伪泛型，vlang实现了类似与C++的真正意义上的泛型机制

但是经过很多次测试都没成功，还有待官方文档的完善和vlang版本的变更。


### 15、并发
与golang非常类似，引用原文如下：

> The concurrency model is very similar to Go. To run foo() concurrently, just call it with go foo(). Right now, it launches the function in a new system thread. Soon coroutines and the scheduler will be implemented.


### 16、json解码
json是后端开发中比较受欢迎的数据格式，vlang内置了json支持：
```golnag
import json
json.decode(User, data)
```
vlang的json解码比golang具有更好的性能：
> V generates code for JSON encoding and decoding. No runtime reflection is used. This results in much better performance.


17、单元测试：All test functions have to be placed in *_test.v files and begin with test_.
To run the tests do v hello_test.v. To test an entire module, do v test mymodule.

### 18、内存管理(开发中)
vlang没有垃圾回收和引用计数，而是在编译过程中清理所有

### 19、Defer：与golang类似
### 20、内建ORM（alpha）
目前vlang 内建的orm只支持Postgres,将很快支持mysql和sqlite
耳听为虚，我们来实际验证下这个功能：
```bash

vim db.v 
```
```golang
import pg
struct Customer { // struct name has to be the same as the table name for now
    id int // an integer id must be the first field
    name string
    nr_orders int
    country string
}

cfg := pg.Config{
    host: '172.17.0.2',
    user: 'postgres',
    //   password string
    dbname: 'my_database'
}
println("OK!")
db := pg.connect(cfg)
nr_customers := db.select count from Customer
println('number of all customers: $nr_customers')

```
```bash
docker run --rm -v $PWD:/home -it vlang:latest
cd /home
v run db.v
```

经过测试官方的例子，发现如下错误：
```bash
root@400d3f0f017c:/home# v run db2.v
sql query="select count(*) from Customer"
V panic: array index out of range: -1/0
```
不知哪里产生了越界，还有通过查看源码发现cfg不支持端口配置。所以目前对orm这个处于alpha阶段的功能持有保留态度。

### 21、vfmt
这是个类似于golang的go fmt功能
验证结果如下：
```bash
root@400d3f0f017c:/home# v fmt db2.v
vfmt is temporarily disabled
```

### 22、友好的脚本语言功能（目前验证没有实现）,可用于部署和构建脚本。
官方实例如下：
```golang
rm('build/*')
// Same as: 
for file in ls('build/') {
	rm(file)
}

mv('*.v', 'build/')
// Same as: 
for file in ls('.') {
	if file.ends_with('.v') {
		mv(file, 'build/')
	}
}
```

### 23、交叉编译
```bash
v -os windows
v -os linux
```

### 24、操作符重载(已验证)

### 25、在v中调用c语言（未验证）

### 26、Hot ode reloading

message.v
```golnag
module main

import time
import os

[live]
fn print_message() {
	println('Hello! Modify this message while the program is running.')
}

fn main() {
	for {
		print_message()
		time.sleep_ms(500)
	}
}
```

```bash
v -live message.v
./message
```
然后修改代码，经过验证可以实现这个功能。

### TODO:
- Translating C/C++ to V
- Inline assembly
- Reflection via codegen

## 官方宣称特性

现在我们再回头看看官方宣称的特性：
- Simplicity: the language can be learned in less than an hour
- Fast compilation: ≈100k — 1.2 million loc/s
- Easy to develop: V compiles itself in less than a second
- Performance: within 3% of C
- Safety: no null, no globals, no undefined behavior, immutability by default
- C to V translation
- Hot code reloading
- Powerful UI and graphics libraries
- Easy cross compilation
- REPL
- Built-in ORM
- C and JavaScript backends

其中性能指标还有待进一步验证，还有许多功能没有开发完，只能算是个半成品。期待12月份正式发布的时候能够眼前一亮吧！
