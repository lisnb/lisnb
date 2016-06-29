title: "std::ios::rdbuf"
date: 2014-12-18 18:47:00
tags:
- rdbuf
- c++
- io
categories: C++

---


今天回食堂吃饭，在路上跟范老师讨论C++读文件的事情。
然后就提到怎么把文件内容全部读到字符串里，因为C++不像python，写起来那么简单

	f=open('./c++_primer.areyoukiddingme','rb')
	
我现在用在stackoverflow上查到的一个snippet

{% codeblock lang:cpp read.cpp %}
#include <iostream>
#include <fstream>
#include <sstream>
#include <cstdlib>

void run()
{
	std::ifstream fio("./c++_primer.areyoukiddingme");
	std::stringstream ss;
	ss << fio.rdbuf();
	std::string s(ss.str());
	std::cout<< s <<std::endl;
}


int main()
{
	run();
	return EXIT_SUCCESS;
}
{% endcodeblock %}

感觉这已经是最简单的读取文件全部内容的代码了。
然后就开始讨论rdbuf到底是干嘛的。
[cplusplus](http://www.cplusplus/reference/ios/ios/rdbuf)有详细介绍。
rdbuf(std::ios::rdbuf)，来自头文件 ios 和 iostream. 是一个重载了的函数。

```
get(1)	streambuf *rdbuf() const;
set(2)	streambuf *rdbuf(streambuf *sb);
```
	
第一种形式，用来返回指向该流当前关联的流缓冲区对象（*stream buffer*），第二种形式用来将当前流关联到*sb*指向的流换中去对象上，并且清空所有的错误状态。
如果*sb*是一个空指针*null pointer*，这个函数会自动将badbit置位，有可能触发异常。
有一些派生类（如`stringstream`和`fstream`）保留有它们自己的内部缓冲区对象，当构造函数被调用时，与之关联。所以在调用这个函数改变关联的缓冲区对象时不会影响到它们原来的内部的缓冲区对象：因为他们会关联到一个和它们原来的缓冲区对象不同的缓冲区对象上。（就像这个函数一样，输入输出操作其实都是基于它们关联的缓冲区对象的）。
这个函数会返回在调用之前，该流关联的缓冲区对象。
比如：

{% codeblock lang:cpp rdbuf.cpp %}
#include <iostream>
#include <fstream>
#include <cstdlib>

void run()
{
	std::streambuf *psbuf, *backup;
	std::ofstream filestr;
	filestr.open("./c++_primer.areyoukiddingme");
	backup = std::cout.rdbuf();//备份cout原来的内部缓冲区对象，第一种形式
	psbuf=filestr.rfbuf(); //获得文件流的缓冲区对象，第一种形式
	std::cout.rdbuf(psbuf); //将cout关联到文件流的缓冲区对象，第二种形式
	//因为流的输入输出都是对内部缓冲区的操作，所以这里cout输出的时候实际上操作的时文件流的缓冲区对象
	std::cout<<"This will be written to the file instead of the stdout"<<std::endl;
	std::cout.rdbuf(backup);//还原cout的内部缓冲区对象。
	filestr.close();
}

int main()
{
	run();
	return EXIT_SUCCESS;
}
{% endcodeblock %}

程序运行时，那个字符串不会出现在标准输出上，会出现在文件里。

所以在第一个snippet中

	ss<<fio.rdbuf()
	
就是获得fio的缓冲区对象，再把其中的内容输出到ss流中，最后通过ss.str()获得表示的字符串。
那能不能直接将ss关联到fio的内部缓冲区上呢，比如：

	ss.rdbuf(fio.rdbuf())
	
这个会报错：

{% codeblock lang:bash %}
➜io  clang++ -o read read.cpp
read.cpp:16:14: error: too many arguments to function call, expected 0, have 1
    ss.rdbuf(fio.rdbuf());
    ~~~~~~~~ ^~~~~~~~~~~
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/../lib/c++/v1/sstream:884:5: note:
      'rdbuf' declared here
    basic_stringbuf<char_type, traits_type, allocator_type>* rdbuf() const;
    ^
1 error generated.
{% endcodeblock %}
因为ostringstream没有带参数的rdbuf重载。
“虽然std::ostringstream继承了std::basic_ios（本来应该有get/set方法），但是std::basic_ostream定义了自己的成员函数rdbuf（get only），所以覆盖了父类的(get/set)方法。”
这个是stackoverflow给出的答案，但我看了一下，basic_ostream的rdbuf也是从basic_ios继承的，但basic_ostringstream自己定义了rdbuf函数。
恩，应该是这样的。
我已经给那个答题的留言了[抠鼻]
（留言失败，要有50的reputation，我只有41个）

```
ios_base <- basic_ios <- basic_ostream <- basic_ostringstream 
```

至于为什么... 

stackoverflow上有人说是因为一般来讲，rdbuf会返回一个 `stringbuf*` ，但ostringstream的返回的是一个 `stringbuf`,但cplusplus给的都是返回一个`stringbuf *` 
所以也不知道为啥了。

然后istringstream有带参数的rdbuf吗？
恩，也没有。


 
