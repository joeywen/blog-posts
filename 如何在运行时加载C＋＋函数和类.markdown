# 如何在运行时加载C＋＋函数和类

标签（空格分隔）： 编程

---

##Problem
有些时候你想在运行时加载一个lib或者function or class，这种事情经常发生在你开发一个plugin或者module时遇到。

在C语言里，你可以轻松的利用dlopen, dlsym, dlclose来做到，但是在C++的世界里却没那么简单了。困难就在C＋＋语言的name mangling上，还有一部分就是dlopen函数是用纯C语言写的，不提供load classes功能。

在解析如何load function和class在c＋＋语言中之前，还是线弄清问题吧－－－name mangling。

### C++ Name Mangling
在C++程序里（或lib or object file中），所有的non-static functions都是以二进制文件symbols来表示。这些symbols都是些特殊的文本字符串，都是唯一的在程序中，lib中活着object file中来标示一个文件。

然而在C语言中，函数的symbol名字就是函数名字本身，例如strcpy的symbol就是strcpy，所以在C语言中不会有中non-static函数出现重名情况。

由于C++有很多C语言没有的功能，例如class，函数的overloading，异常处理等等，所以symbol不可能简单的以函数名来定。为了解决这个问题，C++提出了name mangling，这个name mangling的功能就是把function的名字转换成只有compiler知道的奇怪字符串，利用该函数所有的已知信息，如果函数参数的类型，个数，函数等等，所有如果函数名字为foo（int， char），利用name mangling之后，其名字可能是foo_int_char或者其他字符串也说不定。

现在的问题是在C＋＋标准里（ISO14882）中还没有定义这个function name是被怎样的mangled的，每个编译器都有自己的一套方法。

###Classes
另外一个问题就是dlopen函数仅支持load 函数，不支持load class。

很明显的是如果你想使用该class，就需要new instance出来。

##Solution
### extern "C"
C++有个特殊的关键字来声明函数用C的方式来绑定——extern “C”，在C++中函数如果用extern “C”在前面声明的话，就表示该函数的symbol以C语言方式来命名。

所以只有非成员函数可以用extern “C”来声明，而且他们不能被重载。
虽然很局限，但是这样足够利用dlopen来运行时调用function了。需要强调的是用extern “C”不是就不可以在function内写C++ 代码，依然可以调用class和class的function的。

### Loading functions
用dlsym，加载C＋＋函数就像加载C函数一样，你要加载的函数必须要用extern “C”来声明，避免name mangling。
***Example 1: load a function***
```c++
#include <iostream>
#include <dlfcn.h>
int main() {
    using std::cout;
    using std::cerr;
    cout << "C++ dlopen demo\n\n";
    // open the library
    cout << "Opening hello.so...\n";
    void* handle = dlopen("./hello.so", RTLD_LAZY);
    if (!handle) {
        cerr << "Cannot open library: " << dlerror() << '\n';
        return 1;
}
    // load the symbol
    cout << "Loading symbol hello...\n";
    typedef void (*hello_t)();
    // reset errors
    dlerror();
    hello_t hello = (hello_t) dlsym(handle, "hello");
    const char *dlsym_error = dlerror();
    if (dlsym_error) {
        cerr << "Cannot load symbol 'hello': " << dlsym_error <<
            '\n';
        dlclose(handle);
        return 1; }
    // use it to do the calculation
    cout << "Calling hello...\n";
    hello();
    // close the library
    cout << "Closing library...\n";
    dlclose(handle);
}
```
hello.cpp
```c++
#include <iostream>
extern "C" void hello() {
    std::cout << "hello" << '\n';
}
```

>注意，以下两种方式是等价的
>```c
>￼extern "C" int foo
>extern "C" void bar();
>```
>和
>```c
> extern "C" {
>    extern int foo;
>    extern void bar();
> }
>```
>但是定义变量却是不等价的

###load classes
加载类比加载函数还是有点困难的。

我们不能通过new instance来实例一个class在执行的时候，因为某些时候我们根本不知道要记载的类的名字，

怎么解决呐？我们可以通过多态性来解决。开始我们定义一个base interface，声明一些抽象方法，子类继承并实现这些方法。

所以现在我们需要定义两个helper 函数，用extern “c”声明，一个用来new一个class实例，返回class的指针，一个用来销毁这个指针。

基于此，我们就可以用dlsym来调用这两个helper函数了。

***Example 2: loading class***
main.cpp
```c++
#include "polygon.hpp"
#include <iostream>
#include <dlfcn.h>
int main() {
    using std::cout;
    using std::cerr;
    // load the triangle library
    void* triangle = dlopen("./triangle.so", RTLD_LAZY);
    if (!triangle) {
        cerr << "Cannot load library: " << dlerror() << '\n';
return 1; }
    // reset errors
    dlerror();
    // load the symbols
    create_t* create_triangle = (create_t*) dlsym(triangle, "create");
    const char* dlsym_error = dlerror();
    if (dlsym_error) {
        cerr << "Cannot load symbol create: " << dlsym_error << '\n';
        return 1;
}
    destroy_t* destroy_triangle = (destroy_t*) dlsym(triangle, "destroy");
    dlsym_error = dlerror();
    if (dlsym_error) {
        cerr << "Cannot load symbol destroy: " << dlsym_error << '\n';
return 1; }
    // create an instance of the class
    polygon* poly = create_triangle();
    // use the class
    poly−>set_side_length(7);
        cout << "The area is: " << poly−>area() << '\n';
    // destroy the class
    destroy_triangle(poly);
    // unload the triangle library
    dlclose(triangle);
}
```
polygon.hpp
```c++
#ifndef POLYGON_HPP
#define POLYGON_HPP
class polygon {
protected:
    double side_length_;
public:
    polygon()
        : side_length_(0) {}
    virtual ~polygon() {}
    void set_side_length(double side_length) {
        side_length_ = side_length;
}
    virtual double area() const = 0;
};
// the types of the class factories
typedef polygon* create_t();
typedef void destroy_t(polygon*);
#endif
```
triangle.cpp:
```c++
#include "polygon.hpp"
#include <cmath>
class triangle : public polygon {
public:
    virtual double area() const {
        return side_length_ * side_length_ * sqrt(3) / 2;
        } };
// the class factories
extern "C" polygon* create() {
    return new triangle;
}
extern "C" void destroy(polygon* p) {
    delete p;
}
```

这里还有一些需要提醒的地方：

 - 你必须提供一个creation和destruction函数，因为在C中，你是无法在执行时调用delete来销毁对象的。
 - interface class 中的析构函数应该定义为virtual的，这种做法可能在不必要的地方很少见，但是也不能因为这个而冒险不是。

上面的例子笔者已经测试通过了，完全OK。

想研究更深的，可以查看以下论文：
 

 1. [A lightweight mechanism to update code in a running program Dynamic c++ classes]()
 2. [Dynamic c++ classes A lightweight mechanism to update code in a running program](https://www.usenix.org/legacy/publications/library/proceedings/usenix98/full_papers/hjalmtysson/hjalmtysson.pdf)
 3. (Dynamic Code Updates)[http://ttic.uchicago.edu/~mmaire/papers/pdf/dynamic_update.pdf]

参考：[C++ dlopen mini HOWTO][1]

上述如有错误地方还请指正，不剩感激……

  [1]: http://tldp.org/HOWTO/C++-dlopen/