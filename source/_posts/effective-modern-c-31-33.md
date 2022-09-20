---
title: effective modern c ++ 31-33
date: 2022-09-20 14:21:12
tags: effective_modern_c++
---

从条款31开始进入第6大章，也就是关于lambda表达式的内容
31讲的是关于默认捕获的一些缺点
首先是关于默认引用捕获，即[&](){}的，最明显的就是他往往会有引用局部变量的错误，这个也是这系列书一直在强调的一个点，千万不能引用1个很容易就析构了的局部变量，例子如下
~~~
#include <vector>
#include <functional>

using namespace std;

void add_elem(vector<function<bool(int)>> vec){
  int i = 2;
  vec.emplace_back([&](int value){return value==i;});

}

int main(){
  vector<function<bool(int)>> vec;
  add_elem(vec);
  vec[0](3);


}
~~~
上面报错吐核，因为引用了一个早就已经释放了的局部变量i，而作者建议，如果要用&，最好把要引用的变量也写进去，这样子究竟引用了啥也容易查找，当然像这种情况，最优实践就是按值捕获
但按值捕获也具有一定的缺点，最明显的是指针类型上，按值捕获使得同个地址被2个变量所拥有，就有了多重析构的可能，当然我们可以通过智能指针去解决
还有另外1个问题就是在类的非静态成员函数里的lambda表达式，他的捕获都是捕获this指针的，因为在这些成员函数里，数据成员本来就不在他的局部作用域内，怎么可能捕获到，而他们又持有this指针，所以对下面的代码
~~~
class A{
public:
  void dosth();

private:
  double i;    
};

void A::dosth(){
    auto lambda1 = [&i](){};
}
~~~
会报捕获不了i的错，解决方法也很简单，如下
~~~
void A::dosth(){
    auto lambda2 = [this](){this->i;};
}
~~~
直接按值捕获this指针即可，从上面也可以看出，这个lambda表达式的生存周期，和this指针，也就是this所指对象的生存周期是一致的，所以这里也需要我们注意不要把一些局部作用域里的对象的this指针给传递出来，因为很有可能在离开这个作用域后这个对象就析构了，this指针就空悬了，解决方式是不使用this指针，直接把我们需要的数据给引进来，如在lambda之前加上一句auto tmp = i的操作，或者使用14标准的广义lambda捕获，即[d = i]的操作
最后一点讲的是一些static变量，他们是不能被按值捕获的，因此表面上在一些函数体内，我们使用[=]把非静态的局部变量给按值捕获了，但实际上其表现出来的行为是按引用捕获的，我们可以写个例子如下
~~~
#include <iostream>
#include <vector>
#include <functional>
using namespace std;

void func(vector<function<void()>> &vec){
  static int i = 0;
  vec.emplace_back([=](){cout << i << endl;});
  i++;
}

int main(){
  vector<function<void()>> vec;
  func(vec);
  func(vec);
  vec[0]();

}
~~~
表面上看，每个lambda都按值捕获了i的副本，但实际上他们是以引用的方式捕获的，所以最后输出的结果是2
条款32讲的就是上面提及到的广义lambda捕获，标准上叫做初始化捕获
在11的时候，假如我们想要把一些只移对象或者一些移动操作比复制快的多的对象移动到闭包内，11并没有直接的方式支持，而到了14，就出现了广义lambda捕获，如下
~~~

auto ptr = make_unique<Widget>();
auto lambda1 = [ptr = move(ptr)](){};
auto lamdba2 = [ptr = make_unique<Widget>()](){};
~~~
其中要注意的是，在[]中，等号左边的ptr的作用域是闭包外的作用域，而等号右边的ptr的作用域则是在闭包内的作用域
而在11中，我们也是可以自己写一个类似的东西，只要他有1个接收unique_ptr的构造函数，1个重载（）运算符的函数就够用了
除此之外，我们还可以使用bind去实现（在11中），如下
~~~
auto pw = make_unique<int>(3);

auto mylambda = 
  bind([](const unique_ptr<int>& pw){},
       move(pw));
~~~
bind的第1个参数是可调用对象，在这里就是我们需要的lambda表达式，而后面的参数则是可调用对象的参数列表，那么在调用这个bind的对象时，如mylambda(),则会把第2个参数，也就是右值unique_ptr用于初始化lambda表达式里的pw，就实现了我们需要的初始化捕获
不过现在14的标准就不用这么麻烦啦
条款33讲的是对于使用auto&&的形参，如何保证其左右值的特性
对于下面的lambda表达式
~~~
auto lambda1 = [](auto i){ func(dosth(i));};
~~~
编译器会做的，就是生成1个类，然后重载()运算符，如下：
~~~
class sth{
public:
  template<typename T>
  void operator()(T x) const{
    func(dosth(i));
  }
};
~~~
这里不管传进来的是左值还是右值，都会被当成1个左值看待，那很自然的，我们就会想把其改写成万能引用（auto&&）+完美转发（forward）的形式，如下
~~~
auto lambda1 = [](auto&& i){func(dosth(forward<>(i)));};
~~~
但问题来了，我们究竟拿什么去获取forward的实例化参数呢，我们根本不知道i是什么类型呀，那这个时候，decltype就派上用处了
decltype(i)作为模板实参，则当传进来的是个左值的时候，i会被推导为左值引用，那么decltype推导出左值引用类型；当传进来的是右值的时候，万能引用会推导为右值引用，则decltype会得到1个右值引用类型，不过这里需要注意的一点是，forward我们一般使用的时候，一般的惯例是左值引用的对立面是非引用，也就是说，传进来的应该是个type而不是type&&，但在这里无伤大雅，因为就算decltype推出来是个&&，forward返回的是T&&,也即T&& &&,在引用折叠的作用下，还是返回1个右值，没什么影响。
所以最终的解决方案时
~~~
auto lambda2 = [](auto && i){ func(dosth(forward<decltype<i>>(i)));};
~~~
推广到可变模板参数时，也很简单
~~~
auto lamdba3 = [](auto && ...args){ func(dosth(forward<decltype(args)>(args)...));};
~~~
