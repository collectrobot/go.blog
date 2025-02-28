

相对于[上一个版本](cora-0.2.md)的内容，cora 0.3 版本的变化可以用 "两大两小" 来概括。"两大"是两个新的 feature: CML 的并发模型，以及支持了类型检查。"两小"则是两个 refactor：优化了尾递归 trampoline 的实现，以及垃圾回收那边的实现修改了一下。

## 优化了尾递归的实现

其实这个在之前已经[写过博客](tail-call-in-c.md)了，只不过那时还没有去实现。具体说来就是将以及纯的 trampoline 的实现方式，变成将 trampoline 和函数内 tag 跳转混合的方式。

在函数内部使用 tag 跳转，性能会优于跨函数的 trampoline，因为在函数内部跳转一个是可以省掉进出函数的开销，另一个是对寄存器的更合理使用。
看一下函数内跳转是如何可以寄存器传参的。

```
Obj arg0;
Obj arg1;
Obj arg2;
Obj arg3;

void* jumpTable = {&&label0, &&label1 ...};

label0:
    arg0 = globalRef("fact");
    arg1 = n;
    goto jumpTable[ctx->pc];

label1:
    Obj tmp = arg1 - 1;
    arg1 = tmp;
    if (arg1 == 0) {
        ...
        goto jumpTable[ctx->pc];
    }
```


通过 [tailification](tailfication-delimited-continuation.md) 处理成 CPS 形式之后，代码生成的时候不再有函数调用的概念，实际都是带参数的跳转。生成上面这种形式，则 C 语言局部变量会分配成寄存器。
于是比如这里的 label0 跳到 label2，通过 arg0/arg1/arg2 这些变量进行传参的过程实际上都是走寄存器的。省掉了C语言的函数调用协议里面涉及的传参的开销。

## 支持 CML 的并发模型


这个在之前也已经[写过博客](cml-vs-go.md)了。在 0.2 版本里面,还没有真正的去支持并发，只是做了一个 demo 来验证 resumble exception 的概念。实际上这个语言级别的 feature 等同于 delimited continuation，于是在它之上可以构建出相关的并发编程的库。采用的并发模型就是 CML 那套。就是一个抄作业的事情，抄的[这里](https://wingolog.org/archives/2018/05/16/lightweight-concurrency-in-lua)的。


当前的实现还只包括了 CML 的框架，以及少量的 event。处理了网络，但是像 sleep 和定时器之类的都还没有加进来。


## 垃圾回收的改动

垃圾回收的实现改了一下，从 bump pointer allocator 改成 Go 的那种 freelist 形式了。这个改动背后的原因还是得说一说，不踩坑真的很难想到。

在保守的垃圾回收里面，GC 会从栈上面的对象开始扫描，这些都是 root。栈上的对象是没有类型信息的，不像堆上面用 tag pointer 可以区分出对象的具体类型。
于上栈上任意一个看起来像指针的东西，我们都假设它是指向了托管堆上面的对象的。保守的垃圾回收需要标记这些对象。

如果遇到指针不是正好指向对象的头部，而是指向了对象内部怎么办? 也就是说，指针没有对齐的情况。

```
head : type | size | etc    <--  pointer address 1
...
body ..
of ...     <--  pointer address 2
the ...
object
...

```

如果是 pointer address 1 这个位置，指针是对齐的，它正好指向了对象的 head，从里面可以取出来对象的类型,大小等等信息，于是就可以递归地标记下去。但是如果是 pointer address 2 这个位置，其实正确的处理是需要计算得到 pointer address 1 的。
必须拿到对象的信息才知道如何对这个对象进行 GC 处理。

做到这一点有些手段，比如 mark line 或者额外的辅助空间来提供 meta 信息，来 1:1 地判定某个 address 是对应到什么对象。之前我用 bump pointer 实现方式没有想到这一点，遇到这种情况就打个 WARNING 并跳过这个对象了。这样是有 bug 的。
bug 的触发概率倒不高，只是触发后我观察到这样的 warning 就知道触发了，一旦有触发后面的 GC 就不保证正常了。有时候就有些莫名其妙的 panic。

在新的实现下，每个 block 会只分配同种大小的对象，由不同的 block，有的专门分配 24 字节对象，有的专门分配 32 字节对象等等，这种方式来实现任意大小的对象分配。如果遇到指针不是正好指向对象头部，而是指向对象内部，可以很容易用取余的运算知道它是哪一个 block 里面第几个对象，进而知道该如何进行垃圾回收的标记处理。


## 类型检查

cora 0.3 版本中最激动人心的应该是引入了类型，之前其实也已经[写过博客了](type-checker.md)。

我非常满意当前这种类型检查的实现方式，因为灵活性非常好，类型检查规则就是自己写 type check 的函数来决定，跟 shen 语言的强大有得一拼。不过 shen 是"宏 + 逻辑引擎"，而 cora 没有内置逻辑引擎，没有学习负担，只需要了解算法W 或者是 unification。

在具体实现的过程中遇到了一个工程上的问题，解决的方式也比较满意，这里记录一下。

type checker 它很类似于是一个 interpreter，它的输入就是源代码，而输出就是这个代码的类型检查是否通过。
写代码的时候源代码都写在一个文件里面注意不到，其实是两种不同的"代码"：类型声明和类型标注这是一类，而纯粹的代码本身是另一类。type checker 这个特殊形式的 interpreter 处理这两种不同的 DSL 是需要使用不同的模式。当它处理 declare / deftype 之类的内容时，它是把代码当"代码"去"执行"的：

```
(declare 'id (forall (a) (a -> a)))
(declare 'length (forall (a) ((list a) -> int)))

(func typecheck-function-for-list
    ...)

(deftype my-list typecheck-function-for-list)
```

而当它遇到普通的代码部分，它实际上是把代码当 "数据" 去处理的，把数据作为参数，传递给 type check 函数。

```
(defun id (x) x)

(func length
    () => 0
    [_ . more] => (+ (length more) 1))
```

于是，如果是独立于语言之外去实现 type checker 的时候，就需要既能够把部分的 sexp 视作数据，又能够把部分的 sexp 视作代码。一种方式就是在 type checker 里面再内置一个 cora interpreter。这么弄太重了，我不喜欢。

有没有其它办法呢？我发现这个问题跟宏的实现很像，宏就是即需要把 sexp 不当代码当数据，又需要能执行一些函数的。宏在 cora 中的实现本质上是通过分步的编译，加了一个额外的阶段进行宏展开：读取 sexp -> 宏展开 -> 编译到 c 再到 so -> load so 并执行。
这给了我启发，type check 应该也可以这么做。我引入了一个额外的 preprocess 步骤，变成：读取 sexp -> 预处理 -> 宏展开 ...

在 preprocess 步骤里面，源代码会被分离，分别提供给 type checker 和 compiler 做不一样的处理。具体说来，我把 (:type ...) 和 (:declare ...) 这样的 sexp 都过滤出来，它们是跟类型相关的。然后把代码的 sexp 改写成 (check-type sexp ..)，最后再把代码检查部分跟后续实际的代码整合到一起。上面的例子经过预处理阶段之后，就会变成这样子：

```
;; 类型的声明会保留成原始模样
(declare 'id (forall (a) (a -> a)))
(declare 'length (forall (a) ((list a) -> int)))

(func typecheck-function-for-list
    ...)

(deftype my-list typecheck-function-for-list)

;; 代码部分的 sexp 变成 (check-type sexp)
(check-type `(defun id (x) x) (tvar) () ())
(check-type '(func length
                () => 0
                [_ . more] => (+ (length more) 1)) (tvar) () ())
;; ------
;; 以上是类型检查引入的代码，接下来是代码本身
(defun id (x) x)
(func length
    () => 0
    [_ . more] => (+ (length more) 1))
```

代码检查是一个可选的步骤，可以由开关来控制。当类型检查关闭的时候，preprocess 就不生成前半部分的用于 type check 的代码，而是只生成后半部分实际的代码。

这种方式我就不用把 type checker 实现成独立的步骤，并额外地实现一个 interpreter 了。

## 结语

**cora 到 0.3 这个阶段已经把该加的 feature 都加上了，语言阶段的设计基本上完成**。这个版本东西加得很急，所以还是很糙的状态，基本都是 alpha stage。像比如类型检查这个，很重要的就是错误输入的友好性，当前完全是空白。检查不通过这个代码就挂了，不去 debug 是不知道咋挂的，特别不友好。

Anyway，做到现在的这个样子已经是非常不错的成果，后面再来慢慢迭代。新加一些库，然后性能的优化，然后用户友好性，等等。
