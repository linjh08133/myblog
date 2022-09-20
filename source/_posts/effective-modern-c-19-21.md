---
title: effective modern c ++ 19-21
date: 2022-09-15 11:38:06
tags:
---

条款19讲的是shared_ptr的使用，他的实现大多数都是使用1个指向被管理资源的野指针+1个指向引用计数的对象的指针实现的，所以他的尺寸一般都是野指针的2倍
被管理的资源并不知道这个引用计数的存在
引用计数的内存也是动态分配管理的，其计数的增减也必须都是原子操作，
而在自定义删除器上，shared_ptr并不需要指定他的模板形参为这个删除器的类别，因此我们可以用容器且类别为shared_ptr<SomeClass>的方式去管理，而且不管这个删除器有多大，他都不会影响shared_ptr的大小，因为他是在引用计数那个对象上管理的，如图
！{}()
而这个控制块的生成时机，具体而言有以下几种
1是使用make_shared的时候，必然是在构造一个新的shared_ptr,此时也得有一个控制块的生成
2同理，给一个shared_ptr的实参是野指针的时候，也要1个新建的控制块，而如果传入的实参是1个shared_ptr的话，新的对象则是直接使用这个控制块而不是新建
3是在用1个unique_ptr出发构建的时候，也是需要控制块的，因为unique_ptr本来就没有这玩意
所以，在新建shared_ptr的时候，切记不要将1个野指针用于多次构造函数中，因为这样会产生多个控制块，如果控制块发现自己的计数是0的时候，就会析构这个对象，而这就会导致对象被析构多次，如下代码
~~~
#include <memory>
using namespace std;
class A{
public:
    A(){x=new int(0);}
    A(int _x):x(new int(_x)){}
    ~A(){delete x;}
    int * x;
};
int main(){
    A *ptr = new A(3);
    A *ptr2 = new A(3);
    shared_ptr<A> p1(ptr);
    shared_ptr<A> p2(ptr);
    shared_ptr<A> p3(ptr2);
    p1 = p3;
    p2 = p3;
}
~~~
上面的代码，对同1个对象ptr析构了2次，虽然能编译成功，但运行起来直接报错，所以，我们应该把ptr的new直接放在shared_ptr的构造函数里，这样子别人也就只能拿shared_ptr去做自己的构造函数的实参了，而这样子是不会创建新的控制块的
还有1种情况，就是关于this指针的，如下代码会犯和上面的一样的错误
~~~
vector<Widget> processed_list{};
...
class Widget{
    ...
    void process(){
        ...
        processed_list.emplace_back(this);
    }
};
这里emplace_back会以this指针为参数去构建一个新的shared_ptr,而这个this指针很明显是一个野指针，也就是说，他会创建1个新的控制块，此时就像上面这种情况一样了，c++提供了enable_shared_from_this的玩意，使用起来如下：
~~~
vector<Widget> processed_list{};
...
class Widget:public enable_shared_from_this<Widget>{
    ...
    void process(){
        ...
        processed_list.emplace_back(shared_from_this());
    }
};
~~~
当当前的对象已经被1个shared_ptr管理时，shared_from_this这个成员函数就能创建1个新的shared_ptr对象，且使用已经存在的控制块，否则会报异常
条款20当然就是讲智能指针剩下的weak_ptr了，这个也是第1次了解他，因为之前觉得那2个指针就够用了捏。他主要是配合shared_ptr使用，最主要的功能是判断所管理的对象是否已经析构，如下代码
~~~
auto ptr1 = make_shared<int>(3);
weak_ptr<int> wptr(ptr1);
ptr1 = nullptr;
cout <<wptr.expired()<<endl;
~~~
上面的代码wptr这一行并不会改变ptr1所管理的对象的引用计数，第2行之后，其引用计数仍然为1，在第3行后为0后便析构了,因此我们可以使用weak_ptr的expired函数判断，其指向的资源是否已经析构，上面的代码为输出1
而我们往往会想利用这1特性，去做出1个若未析构则访问的操作，但很明显这个操作并不是线程安全的，因为他不是一个原子操作，假如他expired后发现未析构，而另外1个线程执行了上面第3行的类似的操作，则会有未定义的行为了
weak_ptr提供了lock（）来实现这种原子操作，
~~~
shared_ptr<int> ptr2 = wptr.lock();
~~~
如果该资源已经析构，ptr2为空
后面作者具体给出了使用weak_ptr了场景
第1个就是在某些工厂函数中，他对相同的输入下标id会有相同的输出，为了提高性能，一个直接的想法是使用智能指针去管理资源，当之前已经创建过这个资源，就直接返回该资源，具体代码如下
~~~
shared_ptr<const Widget> get_widget(int id){
    static unordered_map<int, weak_ptr<Widget>> cache;
    auto sptr = cache[id].lock();
    if(!sptr){
        //generate Widget for this id and use sptr to manage it
        cache[id] = weak_ptr<Widget>(sptr);
    }
    retrun sptr;
}
这里的返回类型并不能使用unique_ptr,因为一旦返回了，这个工厂函数内部的那个unique_ptr就为空了，因为返回值是通过移动构造去移动了资源的管理权
第2个就是在观察者设计模式中，这里具体等后面学设计模式再探讨吧！
最后1个就是1个小例子，假设a和c都使用shared_ptr指向b，此时想要让b也能指向a，该使用什么指针呢，很明显shared_ptr是不能使用的，他最大的缺点就是在这种环路引用上的循环问题，野指针也不行，若a析构了，b的野指针无法得知，后面就可能做出未定义行为了，所以最优的方式是使用weak_ptr,a是否析构b能感知到，且weak_ptr不会对引用计数有任何影响，更准确的说，是不会对控制块的第1个引用计数有影响，而是对其第2个引用计数有影响
条款21讲的是尽量使用make_shared和make_unique
在11的标准中，是只有前者而没有后者的，我们当然可以容易实现
~~~
template<typename T, typename ...Ts>
unique_ptr<T> make_unique(Ts&&... params){
    return unique_ptr<T>(new T(forward<Ts>(params)...));
}
~~~
首先是创建了一个T的对象，利用万能引用和完美转发将参数传入其构造函数，并使用unique_ptr去管理并返回
到了14我们就有std的make_unique可以用了
下面作者给出了使用make系列函数的优势点
第1个是定义相关类型的时候，如下
~~~
unique_ptr<Widget> ptr1 (new Widget(params));
auto ptr2 (make_unique<Widget>(params));
~~~
作者认为这里，第1行使用了2次widget，将这个型别声明了2次，造成了代码冗余，导致源代码会增加编译次数
第2个优点在于异常安全，例如下面的代码
~~~
int calculate(){...}

void process(shared_ptr<Widget>, int priority){...}

process(shared_ptr<Widget>(new Widget), calculate());
~~~
这里编译器只能保证，在进入process的代码段之前，构建一个widget对象，构建1个shared_ptr并管理这个widget对象，调用calculate函数3者一定执行完毕，但这3者的顺序并没有指定，如果calculate发生在另外2着中间并且发生了异常，会导致new出来的widget并没有被析构而导致了内存泄漏，这个其实在effective第一本书里有讲，我们可以直接在process调用之前就构建这个shared_ptr对象，并在process的传参中使用move()语义，移动该资源，这样子就没有任何复制操作了，如果只是把这个对象按值传进去可是需要对引用计数进行原子操作的增加的。
而这里采用的是使用make_shared，因为这样子就能保证new出来的对象一定能被shared_ptr管理，就算calculate异常了也没关系
第3个是在性能上的提升，对于make_shared来说，他只会分配1次内存块，这个内存块就可以用来为widget和控制块分配内存，而传统的方式是需要分配2次内存块，分别给2着用
而make系列也不是完美的，在某些情况下他并不适用，比如说，他并不允许自定义删除器，只有一般方式如下
~~~
unique_ptr<int,decltype(mydelete)> ptr1(new int(2), mydelete);
shared_ptr<int> ptr2(new int(3), mydelete);
~~~
make系列的函数是做不到这一点的
还有1个就是他对大括号初始化的不支持如下代码
~~~
auto ptr = make_shared<vector<int>> (10,20);
~~~
他在完美转发的时候是使用得当圆括号，也就是说调用的是非initializer_list形参的构造函数，上面的代码生成的是10个元素，每个元素都是20的vector的shared_ptr,但假如我们想用大括号初始的方式呢，就得这么做
~~~
shared_ptr<vector<int>> ptr1 (new vector<int>{10, 20});//传统方式

auto ini_list = {10, 20};

auto ptr2 = make_shared<vector<int>>(ini_list);
~~~
这样子，就能在完美转发的时候转发的是1个initializer_list形参，也就能调用相关的构造函数了

除了以上2个问题外，对于一些自定义了new和delete的类来说，因为他们通常分配的内存块大小都是等于1个类的大小的，而make系列分配的内存的大小还需要加上控制块的大小，此时使用make就不是1个好主意了
还有就是由于weak_ptr的存在，一个使用make系列函数分配的内存块，他内部有被管理资源的内存和控制块的内存，只有当和其相关的shared_ptr和weak_ptr都没有了的时候才会释放内存，而普通的构造方式下，只要shared_ptr没了，被管理资源就会被析构，此时的weak_ptr只会影响控制块的析构，
