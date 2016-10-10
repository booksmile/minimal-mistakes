---
title: "理解makefile中的规则"
excerpt: "本文介绍makefile中的各种规则格式以及使用方法"
categories: makefile
tags: makefile
---

{% include toc icon="gears" title="目录" %}

## 1 基本概念

**规则**，是makefile中最核心的概念和对象了。

一条规则的格式如下：

```mk
target1 target2: prereq1 prereq2 ... prereqN
    commands
```

一条规则有3个要素：

- 目标
- 依赖条件
- 命令集合

在书写规则时，可以在一条规则里填写多个目标（以空格分开），但是这仅仅是方便书写而已（因为多个目标依赖同样的一组条件），在实际执行时，make会将其拆分成多条规则，这些规则只包含一个目标。

make在解析规则时，会根据2个要素来判断要执行的动作：

1. 目标文件是否存在
2. 目标文件是不是最新的，即是否存在某个依赖文件的修改时间要比目标文件晚。

某个依赖的文件可能会作为另外一条规则的目标来生成，所以make会递归地解析各个依赖文件。

makefile中的规则有3种格式：

- 具体规则
- 模式规则
- 隐含规则

一条包含特定目标（可以是文件名，也可以是伪目标）和特定依赖条件的规则，称为具体规则；

一条目标和依赖条件中包含模式匹配符**%**的规则，称为模式规则；

make内建的一系列规则（多为模式规则），称为隐含规则。

## 2 具体规则

先来看一条最简单的具体规则：

```mk
test: main.o func.o
    $(CC) -g -o test main.o func.o
    
func.o: func.cpp func.h
    $(CXX) -g -c -o func.o func.cpp
    
main.o: main.cpp func.h
    $(CXX) -g -c -o main.o main.cpp
```

这个makefile可以从两个源文件main.cpp和func.cpp生成可执行文件test。

### 自动变量

在编写具体规则时，可以在命令集中使用自动变量简化：

- $@ 表示target（因为make是先解析规则再执行规则的命令，所以$@只会表示单个目标）
- $^ 表示依赖条件（用空格分开），并去除了重复的依赖条件
- $+ 表示依赖条件（用空格分开），没有去重
- $< 表示第一个依赖条件
- $+ 表示修改时间戳在目标之后的依赖条件
- $* 表示目标的主文件名（即去除了后缀的文件名）

这样，上面的makefile可以简化为：

```mk
test: main.o func.o
    $(CC) -g -o $@ $^
    
func.o: func.cpp func.h
    $(CXX) -g -c -o $@ $^
    
main.o: main.cpp func.h
    $(CXX) -g -c -o $@ $^
```

### 搜寻路径

make在解析规则时，首先是要确定目标文件和依赖文件是否存在，以及（若存在）各文件的修改时间戳。这就涉及到一个文件搜索的过程。

默认情况下，make是在当前目录搜索。

可以通过修改`VPATH`变量的值，来制定make的搜索路径：

```mk
VPATH =  src include
```

**注意**: `VPATH`只会影响make解析规则的目标文件和依赖条件文件的搜索路径，并不会影响命令中的文件搜索路径。后者是由命令本身或命令所依赖的shell所决定的。

使用自动变量，make会将自动变量替换为正确的文件路径。

此外，还可以使用`vpath`指定模式搜索路径：

```mk
vpath %.c %.cpp src
vpath %.h %.hpp include
```

这样就表示，在src/目录中搜索C/C++源文件；在include/目录中搜索头文件。

## 伪目标

目标并不一定是一个文件名，其实他可以是任何一个合法的字符串。

在执行make时，如果命令行指定了target，那么make就会去查找该target的规则；如果命令行未指定target，那么make就会默认将第一条规则的target作为项目的target。

假定，我们现在有个target为"clean"：

```mk
clean:
    rm -f *.o
```

这个并不是说要生成一个名为"clean"的目标文件，而是要执行这个规则下面的命令。

但是，如果工作目录中，确实有一个文件叫"clean"，会发生什么呢？

```sh
$ make
make: 'clean' is up to date.
```

make在解析规则的时候发现，"clean"文件是最新的（因为它没有依赖任何其它文件），所以认为不执行这条规则。

可以使用`.PHONY`来告诉make，这个目标是个伪目标：

```mk
.PHONY: clean
```

这样make就不会将一个文件误认为是这条规则的目标了。

## 3 模式规则

**模式**，是一个包含模式匹配符 **%** 的字符串。

**%** 可以位于字符串的任意位置，但是一个模式只能有一个 **%**。

一条模式规则的格式如下：

```mk
patten1: patten2
    commands
```

make在解析规则时，如果发现某个target没有具体规则，那么就会尝试使用模式规则来匹配这个target。匹配时，%字符可以匹配任意多个连续的字符；对于%字符以外的字符，原样匹配。

例如，模式`%.cpp`可以匹配任意以`.cpp`结尾的文件名。

举个例子，假定我们有2个源文件test1.cpp, test2.cpp，想生成test1.o和test2.o。模式规则可以这样写：

```mk
%.o: %.cpp
    %(CXX) -g -o $@ $<
```

这条规则可以将.cpp文件编译为.o文件。

#### 静态模式规则

一般的模式规则，会影响所有匹配模式的target的生成。如果想只对某些文件进行某个模式的匹配，可以使用静态模式规则。

格式如下：

```mk
OBJECTS = test1.o test2.o

$(OBJECTS): %.o: %.c
    $(CC) -c $(CFLAGS) $< -o $@
```

**推荐**: 如果明确列出目标文件比较容易进行扩展名或其他模式的匹配的话，请优先使用静态模式规则。

### 后缀规则

之前老版本的make支持下面这样的规则：

```mk
.cpp.o:
    %(CXX) -g -o $@ $<
```

这个跟上面使用%的规则效果是一样的。

这种格式的规则，不推荐在新版本make中使用这种规则了，看懂就行。


## 4 隐含规则

隐含规则是make内建的一组模式规则（或后缀规则）。

可以使用`make --print-date-base`命令查看所有的内建规则。

隐含规则使用了大量的make内建变量。

比方说，编译C/C++的隐含规则有：

```mk
LD = ld
AR = ar
CC = cc
CXX = g++

LINK.o = $(CC) $(LDFLAGS) $(TARGET_ARCH)
LINK.cc = $(CXX) $(CXXFLAGS) $(CPPFLAGS) $(LDFLAGS) $(TARGET_ARCH)
LINK.cpp = $(LINK.cc)

COMPILE.c = $(CC) $(CFLAGS) $(CPPFLAGS) $(TARGET_ARCH) -c
OMPILE.cc = $(CXX) $(CXXFLAGS) $(CPPFLAGS) $(TARGET_ARCH) -c
COMPILE.cpp = $(COMPILE.cc)

LINK.C = $(LINK.cc)
LINK.c = $(CC) $(CFLAGS) $(CPPFLAGS) $(LDFLAGS) $(TARGET_ARCH)

# 将.o文件链接成可执行文件
%: %.o
#  recipe to execute (built-in):
        $(LINK.o) $^ $(LOADLIBES) $(LDLIBS) -o $@

# 将.c文件编译成.o文件
%.o: %.c
#  recipe to execute (built-in):
        $(COMPILE.c) $(OUTPUT_OPTION) $<

# 将.cpp文件编译成.o文件
%.o: %.cpp
    g++ -g -c $< -o $@
# 将.cc文件编译成.o文件
%.o: %.cc
#  recipe to execute (built-in):
        $(COMPILE.cc) $(OUTPUT_OPTION) $<

# 将.cpp文件编译链接成可执行文件
%: %.cpp
#  recipe to execute (built-in):
        $(LINK.cpp) $^ $(LOADLIBES) $(LDLIBS) -o $@
```

所以，如果想“重载”这些隐含规则的话，只需要重新设置`LD`, `AR`, `CC`, `CXX`, `CXXFLAGS`, `CPPFLAGS`, `CFLAGS`等等这些变量即可。

**注意**：重载隐含规则，会影响所有的使用这些隐含规则的目标的生成规则，如果不仔细的话，会出现意料之外的问题。
