---
layout: post
title:  "[译]使用Clang在Python中解析C++"
date:   2019-04-07 16:10:00 +0800
categories: tech
---

## 前言

​	当我决定研究一下libclang时，特地谷歌和百度了libclang相关的教程，很遗憾的是能找到的有用的资料资料少之又少，而官方的文档是doxgen生成的，只是对接口的一些解释，对初学者来说并不友好。

​	Eli Bendersky的这篇《[Parsing C++ in Python with Clang](<https://eli.thegreenplace.net/2011/07/03/parsing-c-in-python-with-clang/>)》是2011年左右一篇博文，介绍如何在Python语言中利用libclang来解析C++语言，虽然有一定年代了，但不失经典。所以决定把这篇博文翻译出来。

## 译文

如果需要在Python中分析C代码，pycparser通常是非常好的选择。但是如果想要解析C++代码，pycparser就不再适用了。当我被问到什么时候pycpaser可以支持C++时，我的回答是，没有这个计划，你应该去寻求其他的方案。比如说，Clang。

Clang是一个由Apple主导编写，基于LLVM的C、C++和Objective C的前端编译器。Clang+LLVM在近来大有要取代gcc的趋势，它们背后的开发团队十分顶尖，源码也具有非常良好的设计。Clang的开发很活跃，紧跟最新的C++标准。

所以我被问到最多是，人们喜欢pycparser是因为它是用python写的，而Clang的API是C++的。

### libclang

不久之前，Clang的开发团队明智地意识到Clang不仅仅能够作为编译器，而且能够作为分析C/C++/ObjC的工具。实际上，苹果公司自己的XCode开发工具，就把Clang作为库使用，来进行代码补全、交叉引用等。

Clang能够做此用途的组件，就被称为libclang。它的C API相对比较稳定，用户可以利用得到的抽象语法树对代码进行解析。

从技术的角度上看，libclang就是将Clang定义在clang/include/clang-c/Index.h头文件中的公共接口，打包到了一个共享库中。

### libclang的Python绑定

clang/bindings/python目录下，提供了libclang的Python绑定功能，也就是Python的clang.cindex模块。这个模块依赖于ctypes去动态加载libclang库，并试图尽将libclang封装成尽可能Python化的API。

### 文档呢？

非常不幸的是，libclang和它的Python绑定相关的文档非常稀缺。官方文档也只是根据源码自动生成的Doxygen HTML文件。我所能找到的，就是一份[ppt](<http://llvm.org/devmtg/2010-11/Gregor-libclang.pdf>)和来自Clang开发邮件列表中一堆过期的邮件信息。

实际上，如果你扫一眼Index.h的头文件，并捋一捋它试图想要解决的问题，这些API其实并不难理解。另外clang/tools/c-index-test中的工具也能够帮助印证这些API的用法。至于它的Python绑定，除了源码加上一些例子，就更加没有资料。这也是我写这篇文章的目的。

### 开始

开始使用Python绑定非常简单：

* 你的脚本要能够找到clang.cindex模块。要么把它拷贝到合适的位置，要么通过设置PYTHONPATH这个环境变量来指向它
* Clang.cindex 要能够找到libclang.so共享库（译者注：Mac下是libclang.dylib）。取决于你是如何编译/安装Clang的，你也许需要将它拷贝到合适的地方，或者设置LD_LIBRARY_PATH去指向它。在Windows平台下，是libclang.dll，并且它应该被设置到PATH中。

### 一个简单的例子

让我们来看看一个简单的例子。如下的Python脚本代码利用libclang的Python绑定，找到一个给定的C++文件中某个类型所有引用到地方：

```python
#!/usr/bin/env python
""" Usage: call with <filename> <typename>
"""

import sys
import clang.cindex

def find_typerefs(node, typename):
    """ Find all references to the type named 'typename'
    """
    if node.kind.is_reference():
        ref_node = clang.cindex.Cursor_ref(node)
        if ref_node.spelling == typename:
            print 'Found %s [line=%s, col=%s]' % (
                typename, node.location.line, node.location.column)
    # Recurse for children of this node
    for c in node.get_children():
        find_typerefs(c, typename)

index = clang.cindex.Index.create()
tu = index.parse(sys.argv[1])
print 'Translation unit:', tu.spelling
find_typerefs(tu.cursor, sys.argv[2])
```

给定的C++文件`simple_demo_src.cpp`如下所示：

```c++
class Person {
};


class Room {
public:
    void add_person(Person person)
    {
        // do stuff
    }

private:
    Person* people_in_room;
};


template <class T, int N>
class Bag<T, N> {
};


int main()
{
    Person* p = new Person();
    Bag<Person, 42> bagofpersons;

    return 0;
}
```

执行Python脚本，来找到所有引用`Person`类型，我们得到输出结果：

```
Translation unit: simple_demo_src.cpp
Found Person [line=7, col=21]
Found Person [line=13, col=5]
Found Person [line=24, col=5]
Found Person [line=24, col=21]
Found Person [line=25, col=9]
```

### 理解它的工作原理

为了理解上面的例子做了什么，我们需要从3个层面去理解它的内部机制：

* 概念层面——我们从解析的代码中提取的是什么信息以及它是如何存储的？
* libclang层面—— libclang的正式C API，它比Python的绑定接口要更有文档意义
* Python的绑定接口，就是我们在脚本中调用的



#### 创建索引来解析源码

我们从如下两行开始：

```python
index = clang.cindex.Index.create()
tu = index.parse(sys.argv[1])
```

"Index"代表被编译和链接的翻译单元（Translation Unit）的集合。如果想要在翻译单元之间交叉查找，我们需要将它们以某种方式组织在一起。举个栗子，有某个类型定义在一个头文件中，我们也许想要在其他源文件中去找出该类型的引用。`Index.create()` 调用了clang的C API 函数 `clang_createIndex` .

接下来，我们调用 `index` 的 `parse` 方法来从一个文件中解析一个翻译单元。这实际上调用了一个关键的C API—— `clang_parseTranslationUnit` ，它的注释如下：

```
该方法是Clang C API的主要入口，提供了将源文件解析成翻译单元，然后供API中的其他函数进行查询的能力。
```

这是一个很强大的函数，所有可以以命令行形式传给编译器的标记位，它也同样接受。它会返回一个不透明的对象——CXTranslationUnit，然后被Python绑定接口封装成了TranslationUnit。我们可以通过TranslationUnit的属性对它进行查询，例如通过 `spelling` 属性可以得到该翻译单元的名字：

```python
print 'Translation unit:', tu.spelling
```

然后它最重要的属性要数 `cursor` . Cursor的中文意思是光标，它是libclang中的一个重要抽象，它代表被解析的翻译单元的抽象语法树(AST)中的一些节点。`cursor` 对程序中的不同实体（函数声明、变量声明、类声明等）进行了统一的抽象，提供了通用的操作，例如获得它们在文件中的位置、递归获得子 `cursor` 等。`TranslationUnit.cursor` 返回最顶层的 `cursor` ，它是整个抽象语法树的入口。

#### 使用Cursor

Python的绑定接口将libclang的 `cursor` 封装成了 `Cursor` 对象。它有许多的属性，最重要几个是：

* kind——说明当前抽象语法树具体类型的枚举
* spelling——源码中该节点的名称
* location——所在源码中的位置
* get_children——获取子节点

特别解释一下 `get_children` ，因为C和Python API对它的实现有差异。

libclang的C API 是基于visitor模式。遍历AST树时，用户需要向 `clang_visitChildren` 方法提供回调函数。

Python绑定接口则在内部对visitor模式进行了封装，向用户提供了更Python化的迭代器接口`Cursor.get_children` ，它会返回指定节点的所有子节点。当然也可以通过Python访问到原先的visitor模式的接口，但使用 `get_children`显然更有效率一些。在我们的例子中就使用 `get_children` 来递归地访问给定节点的所有子节点：

```python
for c in node.get_children():
    find_typerefs(c, typename)
```

### Python绑定的一些局限性

不幸的是，Python绑定并不完美，还有一些bug，因为它还一直处于开发阶段。假如我们想找出下面C++代码中所有函数调用：

```c++
bool foo()
{
    return true;
}

void bar()
{
    foo();
    for (int i = 0; i < 10; ++i)
        foo();
}

int main()
{
    bar();
    if (foo())
        bar();
}
```

于是我们写下解析的Python代码如下：

```python
import sys
import clang.cindex

def callexpr_visitor(node, parent, userdata):
    if node.kind == clang.cindex.CursorKind.CALL_EXPR:
        print 'Found %s [line=%s, col=%s]' % (
                node.spelling, node.location.line, node.location.column)
    return 2 # means continue visiting recursively

index = clang.cindex.Index.create()
tu = index.parse(sys.argv[1])
clang.cindex.Cursor_visit(
        tu.cursor,
        clang.cindex.Cursor_visit_callback(callexpr_visitor),
        None)
```

这次，我们直接使用libclang的visitor模式的API，程序运行的结果是：

```
Found None [line=8, col=5]
Found None [line=10, col=9]
Found None [line=15, col=5]
Found None [line=16, col=9]
Found None [line=17, col=9]
```

打印出来的源码位置没有问题，但为什么节点的名字都是None呢？细读libclang的代码后发现，我们不应该打印 `spelling` 属性，而应该是 `display name` 。在C的API中，意味着调用 `clang_getCursorDisplayName` 而不是 `clang_getCursorSpelling` . 但是在Python绑定接口中 `clang_getCursorDisplayName` 并没有暴露出来。

这并不能阻挡我们前进的步伐！Python绑定的接口非常得直观，仅仅使用ctypes来将C的API暴露出来，将下面几行加入到`bindings/python/clang/cindex.py`:

```python
Cursor_displayname = lib.clang_getCursorDisplayName
Cursor_displayname.argtypes = [Cursor]
Cursor_displayname.restype = _CXString
Cursor_displayname.errcheck = _CXString.from_result
```

现在我们就可以使用 `Cursor_displayname` 接口。用 `clang.cindex.Cursor_displayname(node)` 替换掉之前Python脚本中的  ``spelling`` ,我们得到如下结果：

```
Found foo [line=8, col=5]
Found foo [line=10, col=9]
Found bar [line=15, col=5]
Found foo [line=16, col=9]
Found bar [line=17, col=9]
```

基于这篇文章，我向Clang项目提交了补丁，来将 `Cursor_displayname` 接口暴露出来，也一并修复了Python绑定的一些其他Bug。它被Clang的核心开发组提交到了134460版本中，现在应该能够从主干获取到。

### libclang的一些局限性

通过上面的例子，我们看到Python绑定的一些限制，相对比较容易克服。因为libclang提供了非常直观的C API，仅仅就是用合适的ctypes结构体将更多的函数暴露出来的问题。对任意一个一般水平的Python程序员，这都不是什么大问题。

然而一些限制是libclang本身的。假如我们想找出一个代码段中所有的return语句。从目前的libclang接口来看，这是不可能做到的。匆匆一瞥`Index.h` 头文件就能发现原因。

`CXCursorKind` 枚举了所有libclang中我们要用到的节点类型，下面是其中的一部分：

```C++
/* Statements */
CXCursor_FirstStmt                     = 200,
/**
 * \brief A statement whose specific kind is not exposed via this
 * interface.
 *
 * Unexposed statements have the same operations as any other kind of
 * statement; one can extract their location information, spelling,
 * children, etc. However, the specific kind of the statement is not
 * reported.
 */
CXCursor_UnexposedStmt                 = 200,

/** \brief A labelled statement in a function.
 *
 * This cursor kind is used to describe the "start_over:" label statement in
 * the following example:
 *
 * \code
 *   start_over:
 *     ++counter;
 * \endcode
 *
 */
CXCursor_LabelStmt                     = 201,

CXCursor_LastStmt                      = CXCursor_LabelStmt,
```

忽略 `CXCursor_FirstStmt` 和 `CXCursor_LastStmt` 这两个占位的，唯一能够真正识别的语句就是 `label`。其他所有的语句都被认定为 `CXCursor_UnexposedStmt` 类型。

琢磨一下libclang的主要目的，并不难理解这种限制的原因。现在这些API的主要用于IDE中，我们仅仅需要知道符号的类型和引用关系，并不太关心表达式的类型。

幸运的是，通过搜集Clang开发组的邮件讨论可以看出这种限制并不是有意为之。libclang仅仅是按需暴露。很明显，使用libclang的人，一开始根本没人关心各种不同表达式的类型，也就没人去添加这种特性。如果它对某些人来说非常重要，他完全可以通过邮件，给开发组建议打相关的补丁。就我们刚刚提到的限制，其实非常容易克服。看一眼libclang/CXCursor.cpp文件中的 `cxcursor::MakeCXCursor` 函数，可以轻而易举地明白枚举类型是如何产生的：

```C++
CXCursor cxcursor::MakeCXCursor(Stmt *S, Decl *Parent,
                                CXTranslationUnit TU) {
  assert(S && TU && "Invalid arguments!");
  CXCursorKind K = CXCursor_NotImplemented;

  switch (S->getStmtClass()) {
  case Stmt::NoStmtClass:
    break;

  case Stmt::NullStmtClass:
  case Stmt::CompoundStmtClass:
  case Stmt::CaseStmtClass:

  ... // many other statement classes

  case Stmt::MaterializeTemporaryExprClass:
    K = CXCursor_UnexposedStmt;
    break;

  case Stmt::LabelStmtClass:
    K = CXCursor_LabelStmt;
    break;

  case Stmt::PredefinedExprClass:

  .. //  many other statement classes

  case Stmt::AsTypeExprClass:
    K = CXCursor_UnexposedExpr;
    break;

  .. // more code
```

这些庞大的Switch语句中可以看出，只有 `Stmt::LabelStmtClass` 返回的不是 `CXCursor_UnexposedExpr` 类型。所以要识别出其他类型只要：

1. 在 `CXCursor_FirstStmt` 和 `CXCursor_LastStmt`之间增加一个 `CXCursorKind` 类型
2. 在 `cxcursor::MakeCXCursor` 函数中加一个`case` 语句，针对合适的类型返回上面所加的类型
3. 将1中的枚举类型暴露在Python绑定中

### 总结

希望通过这篇文章，能够对libclang的Python绑定介绍有所帮助。尽管外部文档很匮乏，但它其实有良好的编码和注释，它的源码足够直观。

Libclang 以及它的Python绑定虽然有一些限制，但是瑕不掩瑜。Clang本身就是一个非常年轻的项目，更别说作为它副产物的libclang。

幸运的是，正如我的文章想要阐明的，这些限制并不是无法克服。仅仅一些Python和C方面的造诣就足以对Python绑定进行扩展，对Clang加深一些了解就能够对libclang进行改进。另外libclang仍然在快速发展，我确信随着时间推移，这些API将会越发改善。缺陷也会越来越少。

