[cora](https://github.com/tiancaiamao/cora) 最初的模块实现是按照[这种](cora-module.md)方式，后来了解到了 [gambit 里面的做法](r7rs-library-implement.md) 之后，又忍不住诱惑想要改掉。最初的那种实现方式，它的优点是简单，实现层面的简单。但是使用层面上，其实更繁琐一点:

```
(@import "cora/lib/foo" foo)
(defun .f ()
	(foo.bar ..))
```

所有的调用其它模块的地方，都需要带上前缀，而对于自身的导出，需要加上 `.` 以防止污染全局名称。这个就更类似 Go 那边的行为，使用自己 import 的 package 也是要带上 package 的前缀，以及用大小写来控制是否导出的行为，只不过 cora 不是用大小写而是用了 `.` 开头的符号。

```
(define-library "cora/lib/foo"
	(export bar)
	...)
	
(define-library "test/example"
	(import "cora/lib/foo")
	(defun f ()
		(bar ..)))
```

这种形式，是否导出由 `(export ...)` 来控制，写起来就不需要加 `.` 前缀了，可以少敲一些代码。

这可能是除了 GC 之后，改动起来最大的部分。主要是 cora 编译到 c 的代码，已经是用旧的模式方式来组织的，而切换到新的实现，这个依赖得非常小步小步地进行，一不小心就会 bootstrap 起不来。启动主要是依赖两个文件，init.cora 和 cora/lib/toc.cora，在 init.cora 里面实现了最基础的一些函数和宏，而在 toc.cora 里面则是编译到 c。这两个文件都是必须先编译出来 c 的，必须用上一个版本的 cora 编译当前版本的 c 文件。

其实写这篇博客主要是记录一下踩到的坑。[前面那篇](r7rs-library-implement.md) 其实已经写了怎么实现了，而这篇是在实现之后，才明白为什么要这样实现。

第一个弯路是，我本想把 define-library 完全实现在 macro 那一层，也就是即不侵入 parser 层，也不侵入编译层，把脏活累活全放在 macro 这一层，反正这一层也丑。**完全实现在 macro 这一层不行，define-library 是需要理解语义的**。

举个例子，之前 cora 里面对 `(defun f () ...)` 展开是直接变成 `(set 'f (lambda () ...))`, 这个变换不需要理解语义，它不需要特殊处理 f 这个符号。但是引入 define-library 之后，如何仍然展开成 'f，就有问题了，它需要根据是否处于 define-library 内部，来决定是变成 `test/example#f` 还是 `cora/lib/foo#f`。就跟所处的上下文有关了。

scheme 的宏是卫生宏，展开的时候就是要理解语义的。而在 cora 中我刻意没像 scheme 那样实现成卫生宏，因为感觉它引入的复杂度不值得它带来的好处。scheme 那帮人就是学术气息太浓了，精力花在研究一些复杂但不实用的东西上面。

更多的研究之后，我发现关于 lisp 模块系统的实现其实可以分成两派："环境模型"和"符号绑定模型"。"环境模型"以 scheme, lua 为代表。"符号绑定模型" 以 clojure, gambit scheme, common lisp 为代表。

环境模型是指符号对应到变量，变量的值是由环境决定的，它对符号没有很强烈的使用，而符号绑定模型则没有"环境"的概念，它直接由一个符号对应到一个值。举个例子，解析 `(define var val)` 的过程，在环境模型中，会先在 env 里面找到 var 是否存在，如果不存在，则修改当前 env 里面对应的 var -> val 的绑定。而在符号绑定模型中，没有 env 的概念，所有符号都是全局的，符号可以直接绑定一个值，这里会把 var 这个符号绑定到 val 这个值。

在环境模型中，模块的实现方式其实是:

```
(let (... )
    (define foo () ...)
	(define bar () ...)
	...
	(make-module foo bar ..))
```

在一个干净的 env 里面，比如说 let，定义好一些函数，然后 make-module 只导出其中 export 的那一部分。其它模块引入这个模块，是通过最后的 make-module 返回的对象，去定位到相应的函数。看到符号扮演的角色没？其实没有符号什么事情。

这里又引入更深一层的东西，就是局部变量的赋值操作。如果我们 define f 和 g 两个函数可能有相互调用的情况，该如何同时将它们引入？

```
(let (f false) (g false)
	(set! f (lambda () ...))
	(set! g (lambda () ...))
```

会需要先在一个 let 中留出糟位，然后用局部变量的赋值操作。**cora 里面没有局变量赋值操作！** 这是一个很深刻的不同点，我更倾向这门语言是纯函数式的，因此没有提供 `set!`。set 在 cora 里面其实是一个函数，它是符号到值的一个绑定，并且是全局的。set 函数很特殊，它把"符号绑定模型"的精髓表达出来了。

为了支持 define-library，语言层面 cora 又新引入了两个特殊表: ns 和 def。原本 cora 只有 `if` `do` `lambda` `let` 这几个，let 还不是必须的，可以用 lambda 替代，只是为了性能而加入的。`set` 是函数而不是特殊表。现在 `(defun f ()..` 会被宏展开成 `(def f (lambda () ..)`，

```
(ns "cora/lib/test"  ("import1" "import2" ..)
	(def f (lambda () ...))
```

当 def f 出现在 ns "cora/lib/test" 中的时候，它实际等价于 set cora/lib/test#f 。

第二个弯路是我引入了一些辅助的全局变量，而对生命期没管理好。`*ns-self*` `*ns-export*`  `*ns-import*` 如果我们维护这样一些变量，define-library 自己就是 `*ns-self*`，而 `(export xxx ...)` 则存储到 `*ns-export*`，`*ns-import*` 用于存储当前的 library 导入了哪些 library。

由于我之前是想着都实现在宏层面，于是 define-library import 和 export 就是分别设置 `*ns-self*` `*ns-export*` `*ns-import*`的值。但是什么时候重置呢？宏展开结束之后，还是调用结束之后？而且涉及 repl 的时候，宏展开和调用还耦合着，保存和恢复值特别难处理，所以这块搞得一团糟。

还有个很恶心的事情是，引入这些变量处理后，macroexpand 成了一个有状态的，某个代码没执行它，只是宏展开一下，就把 `*ns-self*` 等变量搞乱了... 最后我发现这种做法不行，于是不再弄这么多的辅助变量。全部改到了 parser 层面，parser 是解析语义的，parser 的参数里面就有 ns 之类的，就像处理 lambda 会有 env 参数那样，处理 `(ns ..)` 特殊表会更新 ns import 和 export 几个参数。

第三个弯路是关于 import 应该在哪个时期处理。cora 由于是编译到 c，并且涉及到宏，它是多阶段的。还是举例子来说吧，模块 a import 了模块 b，在 b 里面是有 defmacro 的，那么 b 里面定义的宏需要在 a 模块中生效。这就要求: **编译一个模块之前，它 import 的模块是需要先加载的**。

所以我以为 import 可以直接在宏展开时期执行。但这就是一个让宏展开有副作用的过程了，有问题。另外一个点是，如果宏展开的时候，将 import 执行了，是否还需要为 import 生成代码？其实是需要的。因为 load so 跟 load cora 是不同的代码路径，如果宏展开将 import 执行了，然后不生成 import 代码，那么直接 load so 的时候，就会遇到不会加载依赖的模块，因为在 so 文件里面根本没有这样的代码，而对 load so 又没有宏展开的过程了。

import 在 compile-to-c 里面和在 repl 里面其实是不同的处理方式。repl 需要解析 import 并维护这导入的模型列表。



这篇写得感觉没啥条理，完全是只有经历过，记录下给自己看的笔记。
