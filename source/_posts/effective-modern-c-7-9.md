---
title: effective modern c ++ 7-9
date: 2022-09-13 12:09:17
tags: effective_modern_c++
---

条款7开始是第3章的内容，具体就是介绍了一些c+11新特性特别好用的地方
条款7介绍的是{}的优缺点，第1点好处就是他用来初始化时适合于所有情况，一般而言有4种初始化方式，如下
~~~
int x = 2;
int x(2);
int x = {2};
int x{2};
~~~
后面2种其实都是同1个，而如下代码
~~~
Widget w1;
Widget w2 = w1;
w1 = w2;
~~~
这里的赋值并不是初始化，而是复制构造函数,最后的调用的也是赋值运算符的重载
大括号初始化的第1点优势在于可以用其初始化容器
~~~
vector<int> vec {1,3,4};
~~~
第2点是可以用来给非静态成员赋初始值
~~~
class A{


private:
    int x{2};
    int y = 0;
    int z(0); //error! 
};
~~~
第3点是可以用来给不可复制的对象进行初始化，如下
~~~
std::atomic<int> x = 0; //error!!!
std::atomic<int> y{2};
std::atomic<int> z(0);
~~~
从2和3点可以看出，只有大括号初始化在这些情况下是可以通用的，所以说大括号是一种大一统初始化的方式
大括号还有1种特性，括号内是不能使用窄式类型转换的，如
~~~
doube x, y, z;
int sum1 {x + y + z};
~~~
上面不能确定double之和能不能用int表示

大括号的第4点好处是避免了很烦人的解析语法，如
~~~
Widget w(10);
Widget w;
Widget w();//变成函数声明了
Widget w{};
~~~
上面第3行本想用无参构造函数，结果却声明了一个返回值是Widget的无参函数，使用第4行的方式就能避免这种麻烦了
而使用{}的缺点在于类的构造函数的选择问题上，如果构造函数的型参有std::initializer_list且传进来的参数有机会匹配到（有机会包括进行窄式类型转换），那么他会直接忽略其他任何的构造函数，不管说这些构造函数会多匹配如下
~~~
class A{
public:
    A(int x_, double y_):x(x_), y(y_){}
    A(std::initializer_list<bool> list){}
private:
    int x;
    double y;
};

int main(){
    A a{10,0.5};
}
~~~
以上代码会报错，
~~~
error: narrowing conversion of ‘10’ from ‘int’ to ‘bool’ inside { } [-Wnarrowing]
~~~
虽然有一个完美符合a的构造函数，但因为编译器看到了能使用initializer_list的希望，他就直接忽视了其他构造函数了
甚至说以下代码
~~~
Widget w{w2};
Widget w3{std::move(w)};
~~~
本来会使用复制构造函数和移动构造函数的，但如果有上述条件。他还是会调用initializer_list
只有当真的没办法匹配到这个initializer_list型参的构造函数的时候，其他构造函数才会成为候选
最后1个问题就是上面的无参构造函数了，在使用{}的时候表示的是无参，而不是没有元素的空的初始化列表，如果想表示后者，应该这么写
~~~
Widget w{{}};
~~~
而（）和{}的区别，也导致说初始化容器的时候，可能会有意想不到的结果，如下
~~~
vector<int> vec (10, 20);
vector<int> vec2 {10, 20};
~~~
第1个是创建了一个元素都是20.共10个的vector，而第2个则是有2个元素，分别为10和20
这种接口作者认为是失败的

条款8则是介绍了nullptr这个特性，在没有他之前，我们想表示空指针需要使用0和NULL,但前者本质是一个int，不是一个指针，而NULL根据具体的实现不同而不同，但本质也是1个整形数据而不是1个指针，所以在下面的场景中，那个参数为void *的函数是永远不会被调用的
~~~
void func(int){}
void func(void *){}
void func(bool){}
func(0);
func(NULL);
~~~
NULL可能是由long实现的，可能带来歧义，但这里本质的问题在于传入的本意是个指针，结果却调用了非指针版本的矛盾
因此，nullptr登上了舞台，他不具备整型类型，永远不会像0和NULL一样被解释为一个整形，且他可以隐式转换为任何其他类型的指针，此时func(nullptr)调用的就是void *类型的func了
使用他的优点是在使用auto的场景下，到底一个变量是整数还是空指针，这个在有nullptr的情况下就很明了了
~~~
auto func()
->decltype(result)
{
    if(result == nullptr){
        return result;
    }
    ...
}
~~~
假如这里的nullptr是0的话，auto就推导为int，但我们本意是拿result去和空指针比较，所以用上nullptr，result就一定是个指针了
既然说到了auto，那模板推导也必然受益于nullptr了
如下
~~~
template<typename T, typename P>
void func(T t, P ptr){
    ...
}
func(3, 0);
func(3, NULL);
func(3, nullptr);
~~~
上面前2个调用会使得P被推导为int或long的整型数据，只有使用nullptr才使得P被推导为指针类型

条款9讲的是using用来声明类型别名的优势
在不使用类型别名的时候，我们声明一个变量可能很痛苦，使用using就变简单多了，如下
~~~
using MyType = std::unique_ptr<std::unordered_map<std::string,int>>;
~~~
当然我们可以使用typedef去弄别名，在上面的场景和下面这种，2者没啥区别，最多是可读性上不同
~~~
using fp = void(*)(int ,double);
typedef void (*fp)(int, double);
~~~
而using的优点在于模板的别名上，如下
~~~
template<typename T>
using MyList = std::list<T,MyAlloc<T>>; //MyAlloc是自定义的
MyList<int> l1;

template<typename T>
struct MyList{
    typedef std::list<T,MyAlloc<T>> type;
}；

MyList<int>::type l2;
~~~
可以看出使用using比typedef方便多了，不用说去弄个struct写了一堆，而且如果想用上面的这个类型去用做类模板，using同样更加方便
~~~
template<typename T>
class{
private:
    typename MyList<T>::type list;
};

template<typename T>
class{
private:
    MyList<T> list;
};
~~~
第2种就是使用using的写法，第1种既要typename又要::type，比较麻烦
