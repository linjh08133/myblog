---
title: effective modern c ++ 10-12
date: 2022-09-13 15:19:32
tags: effective_modern_c++
---
条款10介绍的是enum的2种类型，一种是非限定作用域的enum，一种是限定作用域的enum，他们主要有以下几点不同
一是作用域的不同
~~~
enum Color{White, Black};
auto White = false;
~~~
上面代码的第2行不能通过编译，因为这里的enum不带class，他的大括号里的内容的作用域会扩散出来，而下面
~~~
enum class Color{White, Black};
auto White = false;
Color c = Color::White;
~~~
则不会，因为这里是限定作用域的enum,所以下面可以声明一个White
二是能否隐式转换的问题
非限定作用域的enum，他能转化为整数型别，甚至是浮点数，而限定作用域的则不可了，除非是static_cast去强制转换，如下
~~~
enum Color{White, Black};

void func(int c){
    ...
}
Color c = White;
c >= 2;
func(c);
~~~
上面这种代码比较诡异，为什么要拿枚举类的东西去和整数比较呢，没有啥意义，因此限定作用域的enum就禁止这种行为了
三是关于能否前置声明而不定义的问题
非限定枚举本来是能只声明的，但因为他的底层具体使用的数据类型是需要具体的枚举型别去确定，例如可能用char，也可能用int等，因此如果在后面，我们修改了他，整个系统就需要重新编译了
而限定枚举能直接声明不定义，因为他默认底层的数据类型就是int，而非限定则没有默认值，只要我们加上默认值，也可以只声明了
~~~
enum Color：int;
~~~

条款11介绍了delete关键词的使用场景
最常见的是我们不想我们的对象能被复制，在98时直接把复制构造函数声明为private且不定义即可，到了11我们可以在后面加上=delete且放在public，
另外1个就是，任何函数都能使用delete，这个应用场景如下
~~~
void func(int num);
func('a');
func(3.2);
func(true);
~~~
如上，我们期望的实参是个int，但却传入了一堆不相关的，但这是允许的，因为他们还勉强能转化为int，为了明确禁止这种行为，我们可以
~~~
void func(int num);
void func(double)=delete;
void func(bool) =delete;
void func(char)=delete;
~~~
这样子上面的3行函数调用会使用被删除的重载版本，也就不能通过编译了
还有1种场景就是模板，我们希望当模板被某种类型的参数实例化时不能通过编译，比如说对指针类型T*,我们不希望void*和char*通过编译，前者因为他过于特殊，无法自增，自减，后者因为他通常是用来表示c的string，我们假定我们的模板不会处理这两种，代码如下
~~~
template<typename T>
void func(T* ptr){
    ...
}
template<>
void func<char>(char * ptr)=delete;

template<>
void func<void>(void * ptr)=delete;
~~~

条款12讲的是关于override的使用
其最重要的使用场景在于，关于虚函数的重载，他的要求十分严格，函数的签名必须一致，const必须一致，异常必须一致（noexcept），引用修饰词也必须一致（在函数签名后加&或&&表示这个函数是给左值对象或者右值对象调用），那么很有可能因为这些限制，我们写出来的“虚函数”实际已经丢失了虚函数的特性，而编译器有可能没有给我们提出warning，因此这个时候就需要override了，如下：
~~~
class Base{
public:
    virtual void func();
};

class Derived:public Base{
public:
    virtual void func() override;
};
~~~
这样子编译器发现override的函数如果有什么不一致会报错
这个条款的最后补充说明了引用修饰词的用法
~~~
class A{
public:
    using datatype = std::vector<int>;
    datatype& getdata() &{ return values; }
    datatype getdata() && {return std::move(values);}
private:
    datatype values;
};

A getA(){
    return A;
}
~~~
上面这段代码A这个类有2个不同的引用修饰词修饰的函数，当左值对象调用时，返回的是左值对象的左值引用，而当右值对象，例如getA()的返回值调用getData时，使用的是&&修饰的函数，因为临时对象本身就是要被析构的，最佳使用方式是移动他的资源，而不是复制，所以&&版本使用了move移动语义，而且他的返回值也不是一个左值引用了，而是一个临时值，是个右值


