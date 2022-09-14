---
title: effective modern c ++ 16-18
date: 2022-09-14 20:20:52
tags: effective_modern_c++
---

条款16讲的是对于const成员函数的线程安全性，
具体而言，const成员函数只能保证不去修改那些没被声明为mutable的成员数据，那也就是说，多个线程访问同一个const函数的时候，就可能会出现data race的情况了

~~~
class Poly{
public:
    using type = std::vector<double>;
    type getroot()const{
        if(valid){
            ... //计算并修改root
        }
        valid = true;
        return root;
    }
private:
    mutable bool valid{false};
    mutable type root{};
};
~~~
以上代码可能在多线程下有问题，此时我们需要施加锁上去
~~~
class Poly{
public:
    using type = std::vector<double>;
    type getroot()const{
        std::unique_lock<std::mutex> l(lock);
        if(valid){
            ... //计算并修改root
        
        valid = true;
        }
        return root;
    }
private:
    mutable bool valid{false};
    mutable type root{};
    mutable std::mutex lock;
};
~~~
不过由于mutex是不可复制的，这个类的复制行为也就应该被禁止了，不过他还是可以移动的
书中还指出，如果mutable的变量只有1个，比如说某个计数器，那么我们可以直接用atomic去表示他，就不用大费周章的用锁了（虽然atomic有时候还是锁的实现），但atomic同样是不可复制的，所以该类同样只能移动而不能复制
~~~
class A{
public:
    void dosth()const{
        count ++;
        //dosth....
    }
private:
    mutable std::atomic<int> count{0};
};
~~~
因为原子变量保证count++的执行是原子的，所以不用担心多线程带来的不确定性
而如果需要2个以上的mutable变量，使用atomic去表示他们的策略不是明智的，例如上面的例子，如果修改root的过程是在valid后面，那么就可能发生这种情况-线程1读valid发现为false，于是计算，然后设置valid，此时线程2也使用了这个函数，发现valid是true，于是直接返回root了，但线程1还没有修改root呢，这种现象发生的原因在于修改2个原子变量的过程并不是原子的，所以我们就需要加1个互斥锁来保证修改2个原子变量这一大动作的原子性，那既然都要用互斥锁了，那还不如直接把2个mutable的声明为普通变量，反正有了锁就能保证原子性了
所以总结的说，使用const成员函数时，要确保对mutable变量的操作能是线程安全的，可以通过互斥锁或者原子变量去实现


条款17说的是各种生成构造函数和生成运算符的生成规则，
规则1.当没有声明任何构造函数时，会生成1个默认的无参构造函数
规则2.如果没有声明复制构造函数等，在使用的时候会自动生成需要的函数
上面2个规则是98时代的，在11，大家都知道引入了移动构造函数和移动运算符，而编译器自动生成的移动操作，会默认的去“移动”每一个成员变量，移动加双引号是因为这个移动是有前提的，如果成员变量本身无法移动的，例如int这种，他执行的是复制，如果这个成员提供了移动操作，才是真正的移动这个变量，而本质上是使用std::move去处理每一个变量，得到的右值如果这个成员本身有移动构造函数能处理这个右值，就是执行移动操作，否则只能使用这个变量的复制构造函数了，因为复制构造函数的形参都是const &，所以能去引用1个右值，这里我们可以写个例子示范如下
~~~
#include <iostream>
using namespace std;
class Movable{
public:
    Movable()=default;
    Movable(Movable && rhs){cout<<"movable class moving"<<endl;}
    Movable(const Movable & rhs){cout<<"movable class copying"<<endl;}
};
class Copyable{
public:
    Copyable()=default;
    Copyable(const Copyable & rhs){cout << "coping class copy" <<endl;}
};
class A{
public:
    A(Movable m_,Copyable c_):m(m_),c(c_){}
private:
    Movable m;
    Copyable c;
};

int main(){
    Movable m;
    Copyable c;
    cout <<"=="<<endl;
    A a(std::move(m),c);
    cout <<"=="<<endl;
    A b(std::move(a));
}
~~~
对于复制构造函数和复制运算符，他们是独立的，声明一种不会导致说编译器不会去生成另一种，但移动的2个操作就不同了，如果声明了其中的一种，编译器则不会去生成另外1个了，因为他会认为既然你自己实现的移动构造函数与默认的按数据成员移动的方式不一样，那么默认的移动运算符必然也与之不同，那还不如不生成
进一步的，复制操作和移动操作，只要定义了其中的一方，编译器就不会去生成另一种操作了，因为既然声明了其中的1种，说明对于另外1种来说，其实现方式也很有可能与默认的方式不同，编译器就采用这种思想。
接下来提到的是一个大三律的东西：复制构造函数，复制运算符和析构函数，只要声明了其中的一种，就应该实现另外的2个，其思想在于，我们之所以会去刻意地定义他们，是因为某些类成员函数的复制方式不同于默认的复制方式，例如说某些指针型的，例如说和资源管理相关的，那么析构函数也必须处理如何析构这些资源
从这个定律推出来，如果有了用户声明的析构函数，就说明默认生成的复制操作是不合用户心意的，因此编译器不应该自动生成，不过98的时代这个思想没有被编译器采纳，但到了11，移动操作应用于这种思想导致了——一旦用户声明了析构函数，移动操作就不会被自动生成
所以结合以上内容，如果想要自动生成移动操作，需要：1.不能声明复制操作2.不能声明移动操作3.不能声明析构函数
接下来我们可以写一个类，这个类只有1个vector的成员，如下
~~~
#include <iostream>
#include <vector>
using namespace std;
class A{
public:
    A(){}
    vector<int> vec;
};
int main(){
    A a;
    a.vec.push_back(2);
    a.vec.push_back(3);
    A c = move(a);
    cout <<a.vec.size()<<endl;
}
~~~
当没有声明析构函数的时候，最终a的vec的size是0，的确是有移动操作在里面，但假如声明一下析构，最终输出是2，说明移动构造函数的确没有自动生成，假如这个vec的内容很多，这个复制的时间必然是多于移动的，此时我们简单的加上A(A&&)=default即可
当然了，有时候我们声明析构函数，是因为需要某个类作为基类，必须得声明他的析构函数并且声明为虚函数提供多态的特性，此时我们可以在那些无法自动生成的移动操作加上=default，告诉编译器这个移动操作使用默认的生成式可以的
所以最后总结如下
• 默认构造函数：与 C++98 的机制相同。仅当类中不包含用户声明的构造函数时
才生成。
• 析构函数：与 C++98 的机制基本相同，唯 的区别在千析构函数默认为
noexcept （参见条款 14) 。与 C++98 的机制相同，仅当基类的析构函数为虚的，
派生类的析构函数才是虚的。
• 复制构造函数：运行期行为与 C++98 相同：按成员进行非静态数据成员的复制
构造。仅当类中不包含用户声明的复制构造函数时才生成。如果该类声明了移动
操作，则复制构造函数将被删除。在已经存在复制赋值运算符或析构函数的条件
下，仍然生成复制构造函数已经成为了被废弃的行为。
• 复制赋值运算符：运行期行为与 C++98 相同：按成员进行非静态数据成员的复
制赋值。仅当类中不包含用户声明的复制赋值运算符时才生成。如果该类声明了
移动操作，则复制构造函数将被删除。在已经存在复制构造函数或析构函数的条
件下，仍然生成复制赋值运算符已经成为了被废弃的行为。
• 移动构造函数和移动赋值运算符：都按成员进行非静态数据成员的移动操作。仅
当类中不包含用户声明的复制操作、移动操作和析构函数时才生成。

接下来进入的是第4大章，关于智能指针的使用
条款18讲的是unique_ptr
我们可以认为一个uniqueptr和野指针有几乎一样的尺寸，且他不能复制，因为本身他的实际应用场景就是管理专用型资源的，而且他的构造函数也使用了explicit修饰，不能使用隐式类型转换去构造如下
~~~
unique_ptr<int> p1(new int(3));
unique_ptr<int> p2 = new int(3);
unique_ptr<int> p3{nullptr};
p3 = p1;
p3 = move(p1);
~~~
第1行是拿一个野指针做参数构造p1，行的通；但第2行是想执行1个隐式类型转换，把int*转换为unique_ptr<int>,这个是行不通的,而第4行自然也error，只有第5行才可以，此时p1的资源就给p3管理了
unique_ptr可以自定义删除器，而且必须得把第二个模板参数声明为这个删除器，所以不同删除器的同个管理对象的类型是不同的，这个shared_ptr就不是了，如下
~~~
class Investment{
public:
     ...
     virtual ~Investment(){}
};
class Stock : public Investment{...};
class Bond: pulibc Investment{...};
auto MyDelete = [](Investment * ptr){dosth; delete ptr;}
template<typename ...Ts>
unique_ptr<Investment, decltype(MyDelete)>
func(Ts&&... params){
    unique_ptr<Investment,decltype(MyDelete)>
    p (nullptr,MyDelete);
    if (/*need stock*/){
        p.reset(new Stock(forward<Ts>(params)...));
    }
    else{
        p.reset(new Bond(forward<Ts>(params)));
    }
    return p;
}
~~~
上面代码的注意点：
我们使用自定义的删除器，则需要指定unique_ptr的第2个模板实参，且在创建该对象时将删除器传入第二个参数
后面的赋值操作并不能通过=的方式，原因上面提到了，需要使用reset函数去转换其管理的对象
而使用了自定义的删除器后，如果我们用的删除器是函数指针，这个unique_ptr的尺寸就比一般的野指针要大了，因为它还要储存函数指针，而如果是无捕获的lambda表达式，作者说是不会浪费任何空间的，这里有待了解详情。
