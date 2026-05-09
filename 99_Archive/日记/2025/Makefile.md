---
tags: 
created: "2025-10-30 16:10"
updated: "2025-11-24 17:58"
---
# Makefile笔记

## 一、语法规则

一个 Makefile 由一组规则组成，规则通常如下所示：

```makefile
target: prerequisites
	command
	command
```

- 目标 `target` 是文件名（或伪目标），以空格分隔。通常每条规则只有一条。
- 命令 `command` 是生成目标的一系列步骤。以制表符开头，不能用空格开头
- 先决条件（依赖）`prerequisites` 是文件名（或伪目标），以空格分隔。这些文件需要在执行命令之前存在

上述其实是描述了一个文件依赖关系，即生成一个或多个 `target` 依赖于 `prerequisites` ，生成规则定义在 `command`中。

**示例：**

`#` 开头的为注释

```makefile
hello: hello.o
	# 第三步
	cc hello.o -o hello   

hello.o: hello.c
	# 第二步
	cc -c hello.c -o hello.o   

hello.c: 
	# 第一步
	echo "int main(){ return 0; }" > hello.c
```

**执行：**

```bash
[root@VM-12-16-centos makefile_test]# make 
echo "int main(){ return 0; }" > hello.c
cc -c hello.c -o hello.o
cc hello.o -o hello
```

**再次执行：**

```makefile
[root@VM-12-16-centos makefile_test]# make
make: 'hello' is up to date.
```

## 二、all

如果 Makefile 中定义了多个目标，通常定义一个 all 目标来生成所有目标， 而且通常我们还习惯将 all 目标放在 Makefile 文件的第一个目标，或者设置 .DEFAULT_GOAL, 这样我们使用 make 就会默认执行 all 目标。

```makefile
all: one two three

one:
	touch one

two:
	touch two

three:
	touch three

clean:
	rm -f one two three
```

> Makefile 中第一个目标会作为其默认目标，直接使用 make 即可，无需制定target。当然你也可以通过 .DEFAULT_GOAL 指定某个 target 为默认的 target



### 2.1 伪目标

```makefile
clean:
	rm -f *.o
```

上述的 clean 并不会真的生成一个 clean 文件，如果恰巧你的目录下面刚好有一个 clean 文件，make 就会提示：

```bash
[root@VM-12-16-centos makefile_test]# ls
clean  Makefile
[root@VM-12-16-centos makefile_test]# make clean
make: 'clean' is up to date.
```

为了避免和文件重名的情况，我们可以使用一个特殊的标记 `.PHONY` 来显式指明一个目标是“伪目标”，向 make 说明，不管是否有这个文件，这个目标都是“伪目标”。

```makefile
.PHONY: clean
clean:
	rm -f one two three
```

> 目标可以成为依赖，伪目标同样可以成为依赖



### 2.2 多行

当命令太长，反斜杆 `\` 字符使我们能够使用多行（和shell类似）

```makefile
some_file:
	echo this line is too long. so \
		it is broken up into multiple lines
```

## 三、变量

```

```

变量是把一个名字和任意长的字符串关联起来。基本语法如下：

```makefile
CXX = g++

TARGET_FILE = hello
```

使用 `${}`或 `$()`来引用变量

示例：

```makefile
BIN_FILE = SM SC

all:
	echo ${BIN_FILE}
```

### 3.1 变量定义

- `=` 仅在使用命名时解析变量值
- `:=`在定义时立即解析变量值

**示例：**

```makefile
A = a
B = $(A) b
A = c
```

 B 最后的值为 c b，而不是 a b。也就是说，在用变量给变量赋值时，右边变量的取值，取的是最终的变量值。 

```makefile
A = a
B := $(A) b
A = c
```

- `?=`如果该变量没有被赋值，则赋值等号后面的值

B 最后的值为 a b。通过 := 的赋值方式，可以避免 = 赋值带来的潜在的不一致。 

```makefile
PLATFORMS ?= linux_amd64 linux_arm64
```

**追加：**

- `+=`表示将等号后面的值添加到前面的变量上。 

```makefile
MAKEFLAGS += --no-print-directory
```

Makefile 还支持**多行变量**。可以通过 define 关键字设置多行变量，变量中允许换行。定义方式为： 

```makefile
all:                    
    @$(cmd)

define cmd
    echo "test define 1"
    echo "test define 2"
    echo "test define 3"
endef
```

### 3.2 环境变量

Makefile 还支持**环境变量**。在 Makefile 中，有两种环境变量，分别是 Makefile 预定义的环境变量和自定义的环境变量。 

其中，自定义的环境变量可以覆盖 Makefile 预定义的环境变量。默认情况下，Makefile 中定义的环境变量只在当前 Makefile 有效，如果想向下层传递（Makefile 中调用另一个 Makefile），需要使用 `export` 关键字来声明。 

```makefile
export USAGE_OPTIONS
```

### 3.3 空格

行尾的空格不会被删除，但开头的空格会被删除

**示例：**

```makefile
with_space = hello   # with_space 变量是 hello末尾有三个空格
after = ${with_space}there

null_string = 
space = ${null_string} # 变量space末尾有一个空格

print: 
	echo ${after}
	echo start"${space}"end
```

> 未定义的变量时机上是一个空字符串



### 3.4 自动变量

makefile定义了一些自动变量，用于自动获取一些值，比如：

```makefile
all: f1.o f2.o

f1.o f2.o: 
	# 比较常用，相当于一个集合，依次取出并执行命令
	echo $@
	
# 相当于
f1.o:
	echo f1.o
f2.o:
	echo f2.o
```

- `$@`规则目标的文件名
- `$<`第一个先决条件的名称
- `$?`比目标新的所有先决条件的名称，它们之间有空格
- `$^`所有先决条件的名称，它们之间有空格

**示例：**

```makefile
hey: one two
	echo $@ # 输出 hey
	echo $? # 所有比目标新的先决条件 one two
	echo $^ # 所有先决条件
	
	touch hey

one:
	touch one
	
two: 
	touch two
	
clean: 
	rm -f hey one two
```

## 四、通配符

`*`成为通配符

- `*`可以在目标、先决条件或 `wildcard` 函数（查找指定目录下指定类型的文件）中使用
- `*`不能在变量定义中直接使用
- 当`*`没有匹配到文件时，就会直接输出`*`(保持原样)，所以我们一般都是配合 `wildcard`函数使用

```makefile
thing_wrong := *.o   # 不要这样使用，假如没匹配到 .o 文件，则thing_wrong变量的值为 *.o
thing_right := $(wildcard *.o)   # 如果wildcard函数没有匹配到 *.o 文件，则thing_right值为空
```

**示例：**

```makefile
thing_wrong := *.o  
thing_right := $(wildcard *.o) 

all: one two three four

one:
	echo ${thing_wrong}

two:
	echo ${thing_right}

three: *.o
	echo $^

four:
	echo $(wildcard *.o)
```

## 五、静态模式规则

静态模式规则可以更容易地定义多目标的规则，可以让我们的规则变得更加有弹性和灵活。

静态模式规则的语法：

```makefile
target...: target-pattern: prereq-patterns...
	command
```

匹配 `target-pattern` 生成 `target`， 匹配 `prereq-patterns` 生成 `target-pattern`

**示例：**

```makefile
objects = foo.o bar.o all.o

all: ${object}

foo.o: foo.c
bar.o: bar.c
all.o: all.c

all.c: 
	echo "ini main() { return 0; }" > all.c

# %.c表示任意.c为后缀的文件，这里会匹配到 foo.c 和 bar.c
%.c:
	touch $@

clean:
	rm -f *.c *.o
```

使用静态规则后：

```makefile
objects = foo.o bar.o all.o

all: ${object}

${object}: %.o: %.c

all.c: 
	echo "ini main() { return 0; }" > all.c

# %.c表示任意.c为后缀的文件，这里会匹配到 foo.c 和 bar.c
%.c:
	touch $@

clean:
	rm -f *.c *.o
```

### 5.1 filter

`filter`函数可用于静态模式规则匹配正确的文件

```makefile
obj_files = foo.result bar.o lose.o
src_files = foo.raw bar.c lose.c

.PHONY: all
all: ${obj_files}

# 匹配到 bar.o lose.o
$(filter %.o, ${obj_files}): %.o: %.c
	echo "target: $@ prereq: $<"

# 匹配到 foo.result
$(filter %.result, ${obj_files}): %.result: %.raw
	echo "target: $@ prereq: $<"
	
%.c %.raw:
	touch $@

clean:
	rm -f ${src_files}
```

> `%`匹配任何非空字符串



## 六、双冒号

双冒号规则很少使用，它允许为同一个目标定义多个规则

**示例：**

如果这些是单冒号，则会打印一条警告，并且只会运行第二条命令

```makefile
all: hello

hello::
	echo "helo"

hello::
	ehco "hello again"
```

## 七、命令

### 7.1 显示与隐藏

在命令之前添加 `@` 以阻止它被打印

也可以运行 `make` 时使用 `-s`参数，这会为每一行命令添加一个 `@`

```makefile
all：
	@echo "This make line will not be printed"
    echo "but this will"
```

### 7.2 执行

每个命令都在一个新的 shell 中执行（至少效果是这样的）

比如：

```makefile
all:
	cd ..
	echo `pwd`  # cd 不会影响 pwd 因为不在一行
	
	cd ..; echo `pwd`  # cd 会影响pwd
	
	cd ..; \
	echo `pwd`
```

### 7.3 默认 shell

默认 shell 是 `/bin/sh`，也可以通过更改变量 SHELL 来改变它：

```makefile
SHELL = /bin/bash
```

### 7.4 递归使用make

要递归调用makefile, 需要使用 `$(MAKE)`变量代替 `make`

它会为你传递 make 标志并本身不会受到它们的影响

```makefile
new_contents = "hello:\n\ttouch inside_file"

all:
	mkdir -p subdir
	printf ${new_contents} | sed -e 's/^ //' > subdir/Makefile
	cd subdir && $(MAKE)

clean:
	rm -rf subdir
```

### 7.5 创建多个目标

`make clean run test`运行`clean`目标，然后`run`，然后`test`



### 7.6 define

`define`实际上就是一个命令列表

**示例：**

```makefile
one = export hello="I was set!"; echo $$hello

define two
export hello=set
echo $$hello
endef

all:
	@echo "this prints 'I was set'"
	@${one}
	@echo "this does not print 'I was set' because each command runs in a separate shell"
	@${two}
```

### 7.7 局部变量

```makefile
# one 变量只在 all 目标中有效
all: one = cool

all:
	echo one is defined: ${one}

other:
	echo one is noting: ${one}
```

```makefile
%.c: one = cool

hello.c:
	echo one is defined: ${one}

other:
	echo one is noting: ${one}
```

## 八、条件

### 8.1 ifelse

**值相等：**

```makefile
foo = ok

all:
ifeq (${foo}, ok)
	echo "foo equals ok"
else
	echo "nope"
endif
```

**值不相等：**

```makefile
foo = ok

all:
ifneq (${foo}, ok)
	echo "foo equals ok"
else
	echo "nope"
endif
```

**判空：**

```makefile
nullstring = 
foo = ${nullstring}

all:
# strip函数可以去除空格，判空的逻辑是与空字符串进行比较
ifeq ( $(strip ${foo}),)
	echo "foo is empty after being stripped"
endif
```

**判断变量是否被定义：**

```makefile
bar = 
foo = ${bar}

all:
ifdef foo
	echo "foo is defined"
else
	echo "foo is not definded"
endif
```

## 九、函数

函数主要用于文本处理。使用`$(fn, argument)`调用函数

### 9.1 subst

用法是`$(subst FROM, TO, TEXT)`, 将TEXT中的内容由FROM变味TO

**示例：**

```makefile
bar := $(subst not, totally, "I am not superman")

all:
	@echo ${bar}
```

> 打印：I am totally superman

```
comma := ,
empty := 
space := ${empty} ${empty}
foo := a b c

# 注意 ${foo}前面不要有空格，否则会被当做字符串的一部分
bar := ${subst ${space}, ${comma},${foo}}

all:
	@echo ${bar}
```

> 打印: a,b,c

### 9.2 pathsubst

模式字符串替换函数

格式：`$(pathsubst <pattern>,<replacement>,<text>)`

查找`<text>`中的单词是否符合模式`<pattern>`, 如果匹配的话，则以`<replacement>`替换。

替换引用`$(test:pattern=replacement)`是对此的简写

**示例：**

```makefile
foo := a.o b.o 1.a c.o
one := $(pathsubst %.o,%.c,${foo}) # %表示任意字符
two := $(foo:%.o=%.c)
three := $(foo:.o=.c)

all:
	echo ${one}
	echo ${two}
	echo ${three}
```

> 输出：
>
> echo  
>
> echo a.c b.c 1.a c.c
> a.c b.c 1.a c.c
> echo a.c b.c 1.a c.c
> a.c b.c 1.a c.c



### 9.3 foreach

`$(foreach var,list,text)`: 将一个单词列表(由空格分隔)转换为另一单词列表。

`list`代表单词列表，`var`设置列表中的每个单词，`text`针对每个单词进行扩展

**示例：**

```makefile
foo := who are you
bar := $(foreach wrd, ${foo}, ${wrd}!)

all:
	@echo ${bar}
```

> 输出：who! are! you!





### 9.4 if

`if`检测第一个参数是否为空，如果是，则运行第二个参数，否则运行第三个。

```makefile
foo := $(if this-is-not-empty, then!, else!)
empty :=
bar := $(if $(empty), then!, else!)

all:
	@echo ${foo}
	@echo ${bar}
```

> 输出：
>
> then!
> else!





### 9.5 call

有点相当于我们自定义函数，`$(call variable, param, param, ...)`: 在执行时，将它的参数 `param` 依次赋给 variable 中的临时变量 `${1}`, `${2}`等

`${0}`可以获取variable变量名称

**示例：**

```makefile
sweet_new_fn = variable name: $(0) first: ${1} second: ${2} empty variable: ${3}

all:
	@echo $(call sweet_new_fn, go, git)
```

> 输出：variable name: sweet_new_fn first: go second: git empty variable:



### 9.6 shell

shell 函数就是调用 shell

```makefile
all:
	@echo $(shell ls -al)
```

## 十、包含

include 指令告诉 make 读取一个或多个其他 makefile

```
include filename...
```

## 十一、vpath

使用 vpath 指定某些先决条件存在的位置

格式：`vpath <pattern> <directories, spcase/colon separated>`

`<pattern>`可以有一个`%`，用于匹配零个或多个字符

比如：

```makefile
vpath %.h ../headers
```

代表要求 make 在 `../headers`目录下搜索所有以`.h`结尾的文件，前提是当前目录没有找到

**示例：**

```makefile
vpath %.h ../headers

some_binary: hello.h
	touch some_binary

hello:
	mkdir ../headers
	touch ../headers/hello.h
	
clean:
	rm -rf ../headers
	rm -f some_binary
```