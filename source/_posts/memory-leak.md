title: Linux下C++内存泄漏检测
date: 2014-10-31 16:14:08
tags:
- linux
- 内存泄漏
- memory leak
- C++
- valgrind
- memcheck
categories:
- C++ 
- Linux 

---

##内存泄漏 ！- ！内存泄露 
应该是前者
<!--more-->
[维基百科 内存泄露 -- Wikipedia, the free encyclopedia.](http://zh.wikipedia.org/wiki/内存泄漏)中用的是“泄漏”。另外，“泄露”，多指信息，机密被不该知道的人知道了，而“泄漏”除了包含以上意思之外，还包括气体液体流出等，为什么我突然想做一个比喻：从嘴里出来的就是泄露，从菊花里出来的就是泄漏[抠鼻]。所以，当形容非抽象的东西跑出来的时候，应该用“泄漏”，也就应该是“内存泄漏”

---

##valgrind

在改一个组里的组件，因为C++写的不是特别多，所以改完了不太放心，一旦跑起来不崩溃，那最担心的就是内存泄漏的问题。弄了几天，记录一下。
网上查到了挺多工具，最后用的是[valgrind](http://valgrind.org), 优点是不用添加什么编译选项（不像*grpof*，需要在编译的时候使用`-pg`，但还是建议加调试选项，即`-g`，这样最后在看报告的时候会更清晰。关于如何在*cmake*中使用调试选项，在[这篇](http://lixipeng.me/2014/06/19/gdb-coredumped/)里提到过）。

###原理

我挺想能了解一下原理的，但现在还没有时间，所以如果以后有机会（因为肯定还会用到），希望能够把这个补上。

###使用

加入你的可执行文件是`libido arg1 arg2 ... `，那最简单的使用方法就是：

    valgrind --leak-check=yes libido arg1 arg2 ...
    
你的程序会比之前慢好多，大概有20到30倍左右，还会用好多内存。然后valgrind就会把所有的信息都输出出来。

####输出到文件中

如果你想把这些信息输出到文件中，以备之后分析，那可以将输出重定向，但应该注意的是，如果你只用以下命令：

    valgrind --leak-check=yes libido arg1 ... > memleak.report.1
    
那最后*memleak.report.1*这个文件中只会有libido的输出，并不会有检测到的信息，因为你程序的输出是 stdout(标准输出文件，宏定义是1)，但valgrind的检测信息实际上是关于内存的错误信息，所以使用的流应该是 stderr(标准错误输出文件，宏定义是2)，上面的命令只将 1 的内容重定向到了*memleak.report.1*中，但3没有，所以需要对3进行重定向，使用：

    valgrind --leak-check=yes libido arg1 ... > memleak.report.1 2>1
    或者
    valgrind --leak-check=yes libido arg2 ... &> memleak.report.1


####使用screen命令
    
当然这可能还有一个问题，就是因为运行时间会很长，所以你可能决定晚上的时候让它运行，你先回宿舍，并且把路由器拔走（比如我... ），那就需要使用screen命令，但使用screen命令的时候要注意，直接使用：

    screen valgrind vgargs ... libido args ... &> memleak.report.1

是没有办法达到目的的，原因我不太清楚，你可以用

    screen ls > 1.txt
    
试一下，*1.txt*中是没有内容的。可能是因为标准输出文件更改了？所以如果要用screen的话，就要先使用screen命令打开新的窗口，然后再使用之前的命令，就可以把信息输出到文件中了，敲完命令，`Ctrl+a,d`,detach一下，你就可以回宿舍了。

####--log-file

当然输出到文件这种简单的功能肯定本身就带了... 
使用*--log-file*可以指定报告文件，文件名中还可以使用变量，比如`%p`代表当前的进程号，比如当前的进程号是 38324，那么

    valgrind --leak-check=yes --log-file=%p.memleak.report
    
那么会得到一个*38324.memleak.report*的文件。当然还有其他的参数，可以参考官方文档，其中`%`可以转义，使用`%%`。

###几个参数

其实我也没用几个... 

`--leak-check=<no|summary|yes|full> [default: summary]`
　　当打开此选项时，检测内存泄漏，如果设置成`summary`，那只在最后给一个结论，就是泄漏了没有，泄漏了多少；如果设置成`yes`或者`full`，会给出泄漏的细节

`--track-origins=<yes|no> [default: no]`
　　控制是否跟踪那些没有初始化的值，默认不跟踪... 我晕... 我还自己写脚本把那些给去掉，看来不用指定... 如果指定了，在报告中会有体现，一会说... 
　　
###报告解读

####LEAK SUMMARY
会列出总结
    
    ==45924== LEAK SUMMARY:
    ==45924==    definitely lost: 10 bytes in 1 blocks
    ==45924==    indirectly lost: 0 bytes in 0 blocks
    ==45924==      possibly lost: 0 bytes in 0 blocks
    ==45924==    still reachable: 0 bytes in 0 blocks
    ==45924==         suppressed: 25,264 bytes in 377 blocks 
    
####Memory leak detection
(这里翻译一下文档，我觉得是最有意思的部分)

Memcheck 跟踪由 `malloc/new` 等等在堆上申请的 blocks，所以当程序退出的时候，它能够知道哪些blocks没有被释放掉。
如果正确的设置了 `--leak-check` 选项，那对于剩下的那些没有被释放掉的blocks，valgrind会逐一的在 `root-set` 中确定是否仍然是可达的。这个 `root-set` 由两部分组成，一部分是所有线程的`通用目的暂存器(General Purpose Registers, GPRs)`，另一部分记录了那些包括栈空间在内的内存的用户空间中被分配的，初始化的，指针指向的数据字等。
如果想要访问一个 block， 有两种方法。第一种就是通过 “`头指针(start-pointer)`”，即那些指向一个 block 起始地址的指针；第二种是“`中间指针(interior-pointer)`”，就是那些指向一个 block 中间的指针。有以下几种情况，会导致出现这种 中间指针：
- 可能这个指针一开始是指向block的开始的，但可能在程序中被故意或者无意间移动了。尤其当你使用了`tagged pointer`时，**i.e. if it uses the bottom one, two or three bits of a pointer, which are normally always zero due to alignment, in order to store extra information**(没懂，tagged pointer是啥... )
- 可能是内存中的垃圾，随机出现的，一点关系都没有，只是巧合
- 可能是指向C++`std::string`的内部 char 型数组的指针。例如，有些编译器会在std::string的前面加上三个字（word）来存储这个字符串的长度，字符串的容量还有引用数目，然后再去存真正的字符数组。所以返回的字符数组的地址是三个字后的地址
- 有些代码可能申请了一个 block的内存，然后用前八个字节存储（block的大小-8）一个64位的数，`sqlite3MemMalloc`就这么干
- 可能指向的是用`new[]`申请的C++的对象（有析构函数）的数组。在这种情况下，有的编译器会在申请的block最前面，用一个`magic cookie`存储这个数组的长度，然后略过这个magic cookie，返回数组的地址。在[这里](http://theory.uwinnipeg.ca/gnu/gcc/gxxint_14.html)能找到更多的信息。
- 也可能是指向了一个使用了多重继承的C++对象的成员

在内存检测的过程中，你也可以选择性的打开启发式检测，来检测由上述几种情况（`stdstring`,`length64`,`newarray`,`multipleinheritance`）导致的中间指针。如果启发式检测被激活，那么如果检测到由上述几种情况导致的中间指针，那么该block仍然会被认为是可达的，就像这个指针也是头指针一样。
基于此，来看一下下述的9种可能的情况：

|   | Pointer chain         | AAA Leak Case | BBB Leak Case |
| - | --------------------- | ------------- | ------------- |
| 1 | RRR ------------> BBB |               | DR            |
| 2 | RRR ---> AAA ---> BBB | DR            | IR            |
| 3 | RRR 　　　　　　BBB　　　|               | DL            |
| 4 | RRR　　 AAA ---> BBB 　| DL            | IL            |
| 5 | RRR ------?-----> BBB |               | (y)DR, (n)DL  |
| 6 | RRR ---> AAA -?-> BBB | DR            | (y)IR, (n)DL  |
| 7 | RRR -?-> AAA ---> BBB | (y)DR, (n)DL  | (y)IR, (n)IL  |
| 8 | RRR -?-> AAA -?-> BBB | (y)DR, (n)DL  | (y,y)IR, (n,y)IL, (_,n)DL|
| 9 | RRR 　　 AAA -?-> BBB | DL            | (y)IL, (n)DL  |


指针图例：

- `RRR` ：一个 root set 节点，或者一个直接可达的block
- `AAA`,`BBB` ： 堆上的 block
- `--->` ：一个头指针
- `-?->` ：一个中间指针

内存泄漏图例：

- `DR` ：直接可达
- `IR` ：间接可达
- `DL` ：直接泄漏
- `IL` ：间接泄漏
- `(y)XY` ：如果中间指针是`real-pointer`（真的指针？），那么就XY
- `(n)XY` ：如果中间指针不是`real-pointer`（真的指针？），那么就XY
- `(_)XY` ：无论是不是`real-pointer`（真的指针？），都XY


每一种可能的泄漏情况，都能归结为上述九种之一，Memcheck工具在输出时，对结果做了合并，得到以下几种输出结果：

- `Still reachable`. 对于BBB块来说，这涵盖了上述的 1 和 2， 有头指针或者头指针链表指向了BBB。 因为有指针指向这个块，那么至少在原则上，当程序退出的时候，程序员会把它释放掉。“Still reachable” 的块是一种很常见的情况，基本不是什么问题，所以默认情况下，Memecheck不会单独把它报出来
- `Definitely lost`. 对于BBB来说，这是上述的情况3。没有指针指向BBB块了。所以这个块就被标示为“丢失”,因为在程序员基本已经不可能在程序退出的时候把它给释放掉了，因为没有指针指向它（*找不到了*）。一般是由于在前面的代码中，丢掉了指向这个块的指针，程序员应该对此负责！
- `Indirectly lost`. 对于BBB来说，这涵盖了上述的情况4和情况9。这意味着这些块丢失了，但不是因为没有指针指向它，而是因为指向它们的块自己本身也丢失了（*这里的AAA相当于情况3中的BBB*），比如说有一棵二叉树，但指向根节点的指针不见了，那它所有的子孙节点都“Indirectly lost”。因为一旦“Definitely lost”的块找回来了，那么这些间接丢失的块就能够找回来，所以Memcheck也不会单独的报出来
- `Possibly lost`. 这涵盖了上述的情况5-8，仍然是对于BBB来说。确实是有至少一个指针指向这个块，但其中至少有一个是中间指针。这也有可能恰好一个内存中的随机值，恰好是这个block中的地址，所以除非你知道有中间指针确实存在，否则你应该给予关注。

另外，**if suppressions exists for a block, it will be reported as "suppressed" no matter what which of the above four kinds it belongs to**(suppression 不知道是啥... )

如果指定了`--leak-check=full`，那对于每个绝对丢失或者可能丢失的情况，Memcheck都会给出具体的细节，包括内存是在哪里申请的。（实际上，它把所有因为同样原因发生泄漏，并且有“同样”堆栈信息的记录合并到一起。当然，你可以通过“--leak-resolution”来定义“同样”。）它不能告诉你什么时候，或者为什么发生了内存泄漏，这些需要你自己去弄i。通常，你得尽可能的让自己的程序中不会产生任何"definitely lost"或者"possibly lost"，然后安全退出。

例如：
> 
8 bytes in 1 blocks are definitely lost in loss record 1 of 14
　　at 0x........: malloc (vg_replace_malloc.c:...)
　　by 0x........: mk (leak-tree.c:11)
　　by 0x........: main (leak-tree.c:39)
88 (8 direct, 80 indirect) bytes in 1 blocks are definitely lost in loss record 13 of 14
　　at 0x........: malloc (vg_replace_malloc.c:...)
　　by 0x........: mk (leak-tree.c:11)
　　by 0x........: main (leak-tree.c:25)
　　
第一条信息描述的是一个很简单的情况，即：一个block中的8个字节直接丢失了。第二条中提到了另外的8个字节也直接丢失了，不同的是，还有80字节因为这8个字节的直接丢失而间接丢失了。这些内存丢失的信息的输出没有什么特定的顺序，所以信息的编号没有太多实际的意义。这些编号可以在使用Valgrind gdbserver的时候，用来列出泄漏的内存块的地址，并且能给出关于这个块为什么还是可达的详细信息。

当指定了`--leak-check=full`时，可以使用`--show-leak-kinds=<set>`来指定所要显示的泄漏的类型。
`<set>`，如果想要通过这个set指定显示的类型，可以使用以下方法：
- 在`definite`,`indirect`,`possible`,`reachable`中，选择一项或者多项，然后用逗号隔开 
- `all` 用来选择所有的类型
- `none` 什么都不显示

###编译器优化

上面的文档介绍的是泄漏的症状，在对组件进行测试的时候，所有这里列出的情况都遇到了，还有一些很奇怪的情况，比如：

    string reck = "pull up to the parties";
    
这句也会报内存泄漏。
这个情况查了挺长时间，后来找到了[可能的原因](http://valgrind.org/docs/manual/faq.html)：
有一些编译器在编译的时候会做一些优化，有一些对象在析构了之后，并没有被马上还给OS，而是缓存在一个池里，等待重用，这可能是一个原因。
所以在进行内存检测时候，应该尽量关闭编译器的优化，用最原生态的去测试（可能还会慢上若干倍）。
对于上面提到的这个问题，有一个变量可以控制，`GLIBCXX_FORCE_NEW`，可以控制编译器，每次都重新申请，而不适用池的机制。
使用这个变量，就在你当前的终端上输入： 
 
    export GLIBCXX_FORCE_NEWS=1

然后进行编译，注意，**这并不是一个CMAKE参数**。


##总结

本来是想连性能测试都放在一起的，不过感觉太长了。
就是把自己学到的一点东西，在这里记录一下。







