---
title: effective modern c ++ 13-15
date: 2022-09-14 16:10:16
tags: effective_modern_c++
---

条款13介绍的是const_iterator，一开始一大段介绍了在98中实现一个const_iterator有多么困难，但到了11，一切都迎难而解
~~~
vector<int> vec;

auto it = std::find(vec.cbegin(),vec.cend(),13);
vec.insert(it,16);

~~~
我们可以使用cbegin()cend()来获取容器的const 迭代器，就算是对非const容器也一样，且insert函数他的第一个参数是const_iterator,这样子就满足他的要求了
除此之外有一点要注意的是，c++11提供了非成员函数的begin，end等函数，为的就是满足容器是数组的情况下的需求，但11并没有提供cbegin或者cend等const函数，我们无法在只依赖std库的情况下做到const遍历数组，因此我们可以这么写
~~~
template<typename T>
decltype(auto) cbegin(const T& container){
    return std::begin(container);
}
~~~
把传进来的容器用const修饰，那么我就能用他去获取cbegin了，当然这种多余的步骤在14的时候就没必要了，14的时候就引入了非成员的cbegin，cend等函数

条款14介绍的是关于异常和noexcept关键字，在11的标准里，一个函数要么可能发射异常，要么保证不会发射异常，如果能确保他不会发射异常，我们就应该加上noexcept关键字，有关异常的知识我也不是很了解，作者这里介绍说加上noexcept比其他方式优化更好也不是很理解，等后续补坑
接着是几个标准库里和异常相关的函数，最重要的就是push_back函数，当需要扩容时，原本的版本是强异常安全性的——直到所有元素都复制到了新的内存上，旧内存上的元素才会析构，如果中途抛出了异常，则没啥事；但如果为了优化复制转而使用移动，假设我们已经移动了一部分元素了，但此时抛出了异常，程序就处在一部分元素在新的内存，一部分在旧的内存，原有的状态已经没有了，如果想恢复到原来的状态转而把移动过的元素移动回去也可能异常，而11的很多函数的实现，为了解决这种麻烦，采取的是“能移动就移动，不能移动才复制”的策略，而判断移动是不是安全就看这个操作有没有noexcept，其中有一部分的函数，例如swap他是否异常安全，完全取决于用户自定义的操作是否带有noexcept，具体例子如下
~~~
template<class T, size_t N>
void swap( T(&a)(N), T(&b)(N)) noexcept(noexcept(swap(*a,*b)));
~~~
上述代码中，noexcept这种使用方式称为条件式noexcept，只有当括号内T的swap的操作，也就是对a这个数组的每一个元素都进行swap操作时时noexcept的，括号里的结果就为true，此时外层的swap就也是保证异常安全了的

条款15介绍的是constexpr关键字，他表示的变量必须是个常量，且必须在编译期就能确定下来具体是什么值，这个一个值可以用在声明数组长度，array的模板实参（长度）等必须在编译阶段就已知的常量值，如下
~~~
constexpr int len = 2;
std::array<int, len> arr;
~~~
而const修饰的，他并没有会在编译期就知道具体值的保证，所以并不能把他用在array这种模板形参上
constexpr还可以修饰函数，不过此时就有点复杂了，如果传入的实参是编译期就能确定的常量值，那他必须保证该函数也能在编译期得到对应的结果，但如果传入的参数有1个是直到运行期才知道结果的，此时的函数就和一般的函数一样了，如下
~~~
constexpr int pow (int base, int numconds){
    ....
}
constexpr auto numconds = 5;
std::array<int, pow(3,numconds)> arr;
~~~
以上代码可以通过，因为2个参数都是编译期就知的常量，因此pow也能在编译期返回具体的值了
而上面pow的....的具体实现与版本有关，在c++11下，constexpr函数最多只能有1条可执行语句，因此得这么写
~~~
return numconds=0? 1:base*pow(base,numconds-1);
~~~
而在c++14下这个限制就放开了，可以按一般的for循环去实现了 

除了内置类型可以声明为constexpr外，我们自定义的类也可以将构造函数以及其他成员函数声明为constexpr，这样子就能在编译器完成相关的语句执行了，如下
~~~
class Point{}
    constexpr Point(double x_ = 0.0, double y_ = 0.0):x(x_),y(y_){}
    constexpr getx(){return x;}
    constexpr gety(){return y;}

private:
    double x,y;
;

constexpr getmid(Point x1, Point x2){
    return Point((x1.getx() + x2.getx()) / 2, (x1.gety() + x2.gety()) / 2);
}
int main(){
    constexpr Point x1(5.3,2.3);
    constexpr Point x2(2.2,34.2);
    constexpr Point mid = getmid(x1,x2);
}
~~~
以上的x1和x2都能在编译阶段都确定下来具体的值，mid也是
