---
title: effective modern c ++ 28-30
date: 2022-09-20 10:23:36
tags: effective_modern_c++
---

条款28讲的是关于引用折叠的内容
在c++中，我们是不可以写一个引用的引用的如下
~~~
int i = 2;
int& ri = i;
int& &rri = ri;//error
~~~
但这样子的话，对于下面的万能引用模板
~~~
template<typename T>
void func(T&& params){
    ...
}
~~~
当我使用func(i)的时候，T会推导为int&，而params的类型就成了int& &&。我们当然是不可以这么写代码的，但是编译器可以，他可以生成这样的代码，然后再使用引用折叠的规则，这里他会折叠为一个&，而具体的规则也很简单，在任一引用是左值引用的条件下，就折叠为左值，就是说& &&折叠为&，&& &&折叠位&&，等等
而这个也是完美转发实现的关键点，当进来的参数是左值的时候，T就成了type&，右值则还是type，也就是说T本身就有个知道入参的左右特性的信息，所以我们可以简略的写出下面的实现
~~~
template<typename T>
T&& forward(remove_reference_t<T>& params){
    return static_cast<T&&> (params);
}
~~~
当传进来的是左值时，T为type&，T&&变成了type& &&，折叠为type&，返回1个左值引用，当右值时同理分析
而引用折叠的使用情景，作者给出了4种，前2种是模板和auto推导，第3种是类型别名，如下
~~~
template<typename T>
class A{
public:
  typedef T&& RvalueRef;
};
~~~
假如我们使用1个左值A<int>对象去初始化，T&&就推导为int&了，是个左值引用，显然不对应他的名字
最后1个情况就是decltype了，这个也不用解释了

条款29介绍的是关于移动操作的一些辩证
第1点就是我们可能需要假设移动操作的不存在，因为要让编译器自动生成我们默认的移动构造函数的条件十分的苛刻，我们不能写复制操作，不能写析构函数，如果是派生类，基类不能禁用移动
第2点就是移动操作在某些例子中并不会比复制快，这里作者举例，举了vector，string和array的例子
对vector来说，他的内存是分配在堆上的，容器是使用指针管理这个对象，那么移动操作也就只需要简单地转移指针如下
~~~
vector<int> vec1;
vector<int> vec2 = move(vec1);
~~~
我们可以简单地认为，vector的移动构造函数就是把vec1的堆内存指针值赋给了vec2，然后把vec1的指针置为nullptr，这当然是常数时间的
但对array来说就不一样了，array可以看做是1个包装了数组的容器，他底层的储存是在对象中的，那对他而言，不管是移动还是复制，都得1个1个进行赋值操作，那这样子也就O(N)复杂度了
对string来说，他会使用 mall string optimization, SSO的优化策略，因为大多数的string都是比较小的，所以对于一些长度小于n，比如说15或者20等，的string他会直接放在对象的缓冲区里，那此时移动和复制的效果也来的一样了，而之所以需要SSO优化，是为了避免频繁的内存申请与释放
条款30讲的是完美转发的失败情况
首先，假定如下模板
~~~
template<typename T>
void func(T&& params){
    f(forward<T>(params));
}
~~~
则对于
~~~
f(params);
func(params);
~~~
如果他们的结果不同，则认为完美转发失败
第1个失败情景之前也见过了，就是大括号参数，如
~~~
f({1,2,3});
func({1,2,3});
~~~
假如f是以vector作为形参，第1行代码就能推导为vector<int>，而第2行就做不到了，因为模板无法推导出initializer_list,只有auto才会假定initializer_list,并推导其元素的类型，所以，我们可以这么写
~~~
auto par = {1,2,3};
func(par);
~~~
那此时par就是1个initializer_list<int>了，完美转发就能运行了
第2种就是在我们需要使用指针的场景下，使用了0和NULL，此时模板会推导出整形类型，根本不可能推导出指针相关的东西出来，所以切记，一定要使用nullptr来代替这两者
第3种是关于static const的成员数据，这个也是第1次见，具体的标准是，对这种数据只需要声明他，而不用定义，在其他出现该数据的地方都会用他声明时的值来代替，那么在需要一个指针指向该数据的时候，这个操作是会失败的，因为他根本就不在内存中，所有出现他的地方都被一个具体的值代替了，
假设这个值叫做val，我们用func(val)时会报错，因为不管他是什么引用，引用在编译器的实现里还是使用了指针的机制，那指针就肯定是需要地址的，很明显就过不了编译了，当然这也得看具体的编译器了
所以解决方法是在实现文件中定义static const Widget::val;即可，不需要给出1个值，符合(one definition rule , ODR)，唯一定义规则
第4种就是关于函数指针的，
假设f可以传入一个指针，如下
~~~
f(void(*pf)(int));
~~~
他的形参是1个函数指针，指向1个int为形参返回void的函数，又假如我们传入的函数有2个版本，如下
~~~
void process(int, int);
void process(int);
~~~
那么当我们执行f(process)的时候，他很简单的就能选取符合的process版本，但假如我们使用func(process)呢，他根本不知道要如何选择哪一个版本，更一般的，假如process是个函数模板，func更加不知道应该使用哪种实例化版本的process，那解决方法就是去指定我们需要的版本，如
~~~
using needed_type = void(*)(int);
needed_type ptr = process;
func(ptr);
~~~
第5种就是关于位域的，也是比较少见的一种情况，因为我们的内存是按字节编址的，引用本质也是一种指针，指向某1个地址，那编译器就不允许有引用去引用1个字节里的某个比特的操作了，严格地说是“不允许有非const引用绑定到位域”，const的引用也只是把要引用的对象的值赋给一个临时变量，然后再让const去引用他而已
所以对于下面的例子
~~~
struct IPv4Header { 
std: :uint32_t version :4, 
IHL:4, 
DSCP:6, 
ECN : 2, 
tota llength:16; 
};
~~~
使用func(IPv4Header.ECN)是过不了编译的，因此我们只需要static_cast一下，如
func（static_cast<std::uint16_t>(IPv4Header.ECN))即可，让func永远只是接受1个位域数据的副本，这个副本肯定是一个可以取址的东西，因此使用上引用就没有问题了
