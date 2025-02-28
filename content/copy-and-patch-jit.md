国庆期间在家带娃，没出去玩，于是就抽空调研了一下有没有什么办法能"轻量/低成本"地实现 JIT。
最终调研的结论是：不行，但是整个过程还是学到了不少好玩的 idea，还是值得记录一下。

## 引子

bytecode 的解释器有好几块性能瓶颈，最基础的 switch 派发可以用 threaded code 解决，但是这个只是其中一部分性能损失。
我想要的一个根本性的答案是：解释器不做 JIT，有没有可能否达到 native 的性能? 答案是，不行，即使做大量的优化也不行，执行模式决定了上限。

为什么?让我们看一个最基本的解释器循环，它的开销:

``` 
opLocalRef:
    idx = DECODE(pc)
	val = locals[idx]	
    DISPATCH(pc)
```

一条 bytecode 指令的解释有几个阶段：取指 -> 译码 -> 执行 -> 分发。

首先，取指需要从当前指令流中读取到一条指令。作为对比，native 也需要取指，但是 native 的取指是由硬件完成的，而解释器不同，这里是我们自己的逻辑，至少需要花费掉一次从内存中 load 指令的操作。好在 bytecode 的取指是在一块打平了的内存块，内存一层的缓存会命中，比起 ast 解释的取指会高效很多。(解释器的"取指"这一条指令，在硬件又对应的 取指 -> 译码 -> 执行 -> 分发 4步操作)

接下来是译码，指令里面有操作数，比如上面这个例子，opLocalRef 读取局部变量，需要从指令中得到是哪个局部变量，也就是 idx 需要从指令中解析也来，这步是译码。(解释器中的"译码"这一步，硬件"又双"对应的 取指 -> 译码 -> 执行 -> 分发 4步操作)

接下来是执行，这一步是我们真正要做的事情，换成 native 也是同样做这一件事情。

最后一步，我们需要 DISPATCH 跳转到下一条 bytecode，即使是使用了 threaded code 方式，它也是一个 jmp 指令，而如果是在 native 里面，是不需要跳转的，直接继续执行下一条指令就行。(这一步也对应到硬件"又双叕"一次的 取指 -> 译码 -> 执行 -> 分发 4步操作)。另外，jmp 指令是破坏 CPU 流水线的。

也就是说，四件事情中只有 "执行" 这一件事情，是 native 应该做的，从这个点推断，bytecode 解释的实现方式，和 native 实现方式，理论上限存在着 4X 的性能差距。而实际上的做过很多优化的解释器，能到 native 的 7-10 倍以内，就非常优秀了。因为很可能的情况这个数据出现 10-100 倍。

除了解释器循环，为什么解释器无法达到 native 性能，还有一个点在于 寄存器分配。

在 bytecode 解释器里面，中间对象使用的是宿主语言的变量，通常是不能直接映射成硬件寄存器的。比如说：

```
opPush:
	stack[pos] = val
```

这个对应到一次内存访问。CPU 的硬件层次结构，CPU 指令是最快的，寄存器访问在 1ns 以内，然后是 L1 L2 cache，如果命中，从 1ns 到 10ns 量级完成，而如果都 miss 了直接访存，则时间到了 100ns 量级。注意，直接访问寄存器跟穿透L1 L2后访问内存，这中间是 100X 的性能差距。

native 优化得比较好的时候，对变量的操作都是走寄存器完成。而 bytecode 解释器，则是变量操作要访问内存。

所以在寄存器分配这一块，bytecode 解释器跟 native 又产生了数量级的性能差异。

## 优化解释器

解释器有不少优化的路子，这里可以列举一些。

- 想办法让最常访问的变量在寄存器

比如说 gcc 可以 [global register variable](https://gcc.gnu.org/onlinedocs/gcc/Global-Register-Variables.html) 之类的。至少让解释器中 pc/ebp/esp 这些映射成寄存器。

或者是 global 一份 vm，然后在解释器循环的函数里面，把 pc/ebp/esp 这些从 vm 拷一份局部变量，寄希望于编译优化会为这些局部变量分配寄存器。

- threaded code

从 switch-case 派发，变成 threaded code 可以让两次跳转，变成一次跳转。

- 栈虚拟机 vs 寄存器虚拟机

我比较认同的一个观点是，寄存器虚拟机的性能占优势，是源于实现同样功能，它需要的指令的数量更短一些。
解释器 "取指 -> 译码 -> 执行 -> 分发" 循环过程，由于指令数量短些从而而造成的浪费更少一些。

栈虚拟机需要好几条指令：

```
opConst X
opPush
opConst Y
opPush
opPrimAdd
```

而寄存器虚拟机一条指令:

```
R0 = opPrimAdd R0 R1
```

寄存器虚拟机里面的寄存器也是虚拟寄存器，不是能直接映射物理寄存器的。所以差异主要在解释器循环，而不是在寄存器分配。

- top-of-stack-caching

栈虚拟机通过让栈顶的若干元素，缓存在寄存器，从而优化性能。见[栈顶缓存](top-of-stack-caching.md)

不过只有 1-TOS 比较好做，缓存大于 1 之后状态太多了，就不好处理了。

- quickering

既然解释循环中指令派发的开销比较大，那么一个思路就是把多条指令，合到一起只做一次派发，这样就可以缩小无效操作的比例，提升性能。
quickering 就是这个思路。

## 路线一

我开始琢磨的第一个方向是，激进优化的解释器。这已经不算常规的字节码解释器的优化思路了，但是它还是算"解释器"的范畴，毕竟跟 "编译器" "JIT" 这些比起来，解释器的实现成本还是更有吸引力的。

pypy 和 graalvm 这一种的，写解释器得编译器，不过两者具体的实现思路不一样。然而我都觉得理解起来复杂了。

yjit 是一个 ruby 的 JIT 实现，我看了一下它用到的技术所引用的论文是 (Basic Block Version)BBV，BBV 更早是 Marc Feeley 的学生提出来的一篇论文
"Removing Dynamic Type Tests with Context-Driven Basic Block Versioning"。大略读了一下，感觉更多是在动态类型特化成具体类型那块的代码生成比较精彩，它也是 self-modify 的 ast 解释器。哦，Marc Feeley 是谁? gambit scheme 的作者!我好像说过 Marc Feeley 的论文都值得一读这类的话。

路线一的方式都是 self-modify 的 ast 解释器，我对这一块接触得有限，所以总是觉得虽然看起来 "写解释器得编译器" 很有诱惑力...但是吧，复杂度还是太高了。

## 路线二

有没有复杂度更低的实现路径呢? 还真的有。探索这块的还不少。

GNU jitter，[My virtual machine is faster than yours](https://ageinghacker.net/talks/jitter-slides--saiu--ghm2017--2017-08-25.pdf)，非常好的一个介绍。

[Copy and Patch](https://arxiv.org/abs/2011.13127) 以及大名鼎鼎的 QEMU...都是用到了这种方法。QEMU 那边把这个叫 dyngen

copy and patch 的作者还有这样一篇[博客](https://sillycross.github.io/2023/05/12/2023-05-12/)。

思路是这样的，把用到的字节码，用 C 实现，然后编译出来 object 文件，再从 object 里面提取出指令信息备起来。
代码生成的过程，就变成了直接 copy 这些指令。

毕竟 JIT 说白了，就是 mmap 一块可执行的内存，然后往里面写二进制指令，再执行就完事了。麻烦的过程是怎么样将源代码生成二进制指令。
现在 copy and patch 就是直接从生成好的 binary 里面模板提取...

刚开始看到这个想法时，觉得太惊艳了...激动得想马上动手。但是仔细审视，细节是魔鬼。

首先需要很熟习 elf 格式以及可执行程序加载的一些细节，提取指令之后是需要 patch 的，因为从 elf 加载到内存会有重定位，我们需要从 elf 格式分析信息得到需要 patch 哪个位置。

编译也不是随便编译，比如说乱序执行等一些优化不能开，否则生成的代码不对...这块又对 gcc 需要很多的了解，才能控制它按想要的方式生成代码。

最后是寄存器分配，用 copy-and-patch 方式做模板 jit 可以很容易把解释器循环开销问题搞定，但是寄存器分配问题依然绕不开。
所以如果选择做 JIT 了，肯定不会满足于性能还比较挫的 JIT，必然追求优化寄存器的使用，那最终的结果就是复杂度太高，搞不了。

一个比较好说明 copy-and-patch 的原理性的 demo 可以看这个[博客](https://tia.mat.br/posts/2013/07/20/partial_functions_in_c.html)，它已经绕开了 elf 解析的事情从而更容易理解。如果再深入一些，就是看 QEMU 的 [dyngen](https://review.gerrithub.io/plugins/gitiles/spdk/qemu/+/5a246934eb737c242e28995641c9ebf80477b0b7/dyngen.c) 了，这个程序会从 elf 文件中提取指令，生成 jit。



JIT 这个事情啊，还是复杂度过高了。想得太多，做得太少，最后又做不了，不脚踏实地，白白浪费了许多光阴!
