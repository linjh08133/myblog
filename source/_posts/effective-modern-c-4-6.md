---
title: effective modern c ++ 4-6
date: 2022-09-12 20:40:52
tags: effective_modern_c++
---


条款4是教如何去获取类型推导结果的，第一种就是利用IDE去获取，第2种我们可以声明一个类模板但不去定义他，然后使用decltype(x)让编译器报错，如下
~~~
template<typename T>
class TD;


int main(){
 const int x = 3;
 TD<decltype(x)> xtype;
}
~~~
此时编译器报错如下：
~~~
error: aggregate ‘TD<const int> xtype’ has incomplete type and cannot be defined
  TD<decltype(x)> xtype;
~~~
可以看到x的确被推导为const int
还有1种就是使用type_info,在大多数情况下他是正确的，但他推导的方式是安值推导的，也就是说，引用和常量性会被忽略，所以他并不可靠，


接下来是条款5，是开始了第2大章，关于auto的使用
条款5具体讲了一些应用auto带来的方便与好处
第1个就是在使用iterator的时候，如下
~~~
template<typename It>
void func(It b, It e){
    while(b!=e){
        typename std::iterator_traits<It>::value_type
        val = *b;
    }
}
~~~
像上面这一段，我们使用萃取去获取这个迭代器到底指向啥东西，写起来十分的拗口麻烦，我们可以利用auto直接写成auto val = *b；
第2个就是使用auto来保证变量一定能初始化，解决变量未初始化的行为
~~~
auto i; // 不能通过
auto i = 2;
~~~
第3个就是使用lambda表达式时，这个lambda对象到底是个什么类型，这个是由编译器决定的，所以我们需要利用auto用来把lambda表达式赋值给某个变量名，如
~~~
auto lam = [](){ return 0; };
~~~
当然我们也可以用一个std::function去持有这个lambda，如
~~~
std::function<bool(int,int)>
myfunc = [](int x, int y){ return true;};
~~~
而其缺点在于function本身就是一个对象，他本身就是需要内存的，在内存上来说auto来的更好，
第4个就是对于一些硬件依赖的typename，比如unsigned，在32位上和在64位上不同的，单单指定某个变量是unsigned可能会在不同机器上带来出乎意料的结果，因此我们可以利用auto
~~~
auto size = v.size();
~~~
第5个还是在迭代的时候，我们对于一些容器他储存方式不熟悉带来的问题，如
~~~
for(const std::pair<std::string, int> &p :m){...}
~~~
m是个unorder_map,他本身是由std::pair<const std::string, int>组成的，不是上面这种方式，也就是说，程序跑到这段代码，会复制容器里的每一个元素，然后引用再去指向这些临时对象，可想而知多非时间，而使用auto很容易就解决
~~~
for（const auto &p:m){...}
~~~
当然了，auto也是得看实际的使用场景，大多数情况下使用得当能提升我们编程的效率的
条款6 介绍的是使用auto可能遇到的坑，在这种时候就得采用传统的方式了
第1个，就是vector\<bool>这个和其他vector格格不入的对象，对于一般的vector\<T>[],他能返回一个T&类型的东西，但vector\<bool>他本身是特化过的，他底层是使用比特去表示这1个1个的bool，而c++又不能返回1个对比特的引用，所以他只能返回一个用来模拟bool&的reference，即std::vector\<bool>::reference,他能够像bool进行隐式转换，所以对下面语句
~~~
vector\<bool> vec{true,true};
bool is_true = vec[0];
~~~
这里取出来的vec[0]实际是个std::vector\<bool>::reference，但他可以转换为bool，所以没啥问题
但假如我们使用auto去声明is_true,得到他类型就是std::vector<bool>::reference了，他就不是指代第1个元素是否为true的变量了
书里介绍的这种情况带来的问题在于可能出现悬空指针，例如，当我们函数的返回值是个vector<bool>时，他是个临时对象，而他实现获取第1个元素的方式，有一种实现是通过指针+偏移量的方式去获取，而我们如果使用auto is_true = bool_vectroy()[0]时，is_true和临时对象的指针指向同一个东西，而临时对象在这一句话后就解析了，那is_true就指向1个被析构了的地址了，
