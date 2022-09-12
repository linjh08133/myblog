---
title: effective modern c ++ 1-3
date: 2022-09-12 16:24:12
tags: effective_modern_c++
---

条款1是对模板参数推导的几个细则，具体以下代码
~~~
template <typename T>
void func(ParamType param){}
~~~
使用func(expr)去调用时，它会推导T和param的形别，这2种有的时候是一样的，有的时候由于常量这些标识而不同，例如
~~~
template <typename T>
void func(const T& param){}
~~~
在使用一个int 变量i去调用func时，T是iNT,而PARAMTYPE是const int&
具体则分三种情况去讨论
第1种是paramtype是一个非万能引用，此时的判断方法是：如果expr具有引用，把引用给忽略，然后再去推导，如下
~~~
template <typename T>
void func(T& param){}

int x = 2;
const int cx = x;
const int& rx = x;
~~~
那么在调用func的时候，对x，T是int，paramtype是int&，而后面2个T是const int，而paramtype是const int&，因为第3个expr的引用是会被忽略的，这里也可以看出，持有T&的模板，它能保证传进来的对量的常量性能被捕获
第2种是paramtype是一个万能引用，如
~~~
template<typename T>
void func(T&& param){}
~~~
这个时候就使用引用折叠，可知如果传进来的expr是左值，T和paramtype都会推导为左值引用，如果expr是右值，则根据1的规矩即可

第3种情况就是paramtype不是引用，那么就是说函数是按值传参的，他复制了一个新的对象，此时他对expr，会忽视他的const，volatile和引用，所以对上面的，x，cx，rx，如果模板声明如下
~~~
template<typename T>
void func(T param)
~~~
那么T和paramtype最终都是int，这个也是可以理解的——本身传进来后我是构建的新的对象，不会对外面的一切造成干扰
这里还有1种特殊情况，即指向常量的常量指针，const int * const ptr,那么传进来之后，*右边的const会被忽略，因为传进来的本质是1个地址，这个值就像前面那个const int一样，所以他就被忽略了，那么进来后，T会被推导为const int*，指向常量的指针
最后就是关于数组的问题了，这里直接结论如下：
当paramtype是T param时，数组形参会退化为1个指针例如
template<typename T>
void func(T param)
const char name[] = "22";
此时把name传进来时，T会推导为const char*
但假如paramtype是个引用，T则会推导为const char[3],而paramtype则是cost char（&）[13]
我们可以用这一特性去推导数组长度
~~~
template<typename T, int N>
constexpr int getlen(T(&)[N]) noexcept{
    return N;
}
int main(){
 int x[2] = {2,3};
 char y[getlen(x)];
}
~~~
条款2是auto推导的规则，他的规则上和模板推导的基本一致，但有一点很特别，在使用c++11引入的初始化方式中，如下
~~~
auto x{3};
auto x={3};
~~~
此时auto推导型别会推导出std:: initializer_list<int>，且其中只有1个元素的变量，所以如下代码编译是会失败的
~~~
auto y{2,3,3.0};
~~~
因为对std:: initializer_list<T>的T推导不出是什么
这里的本质是有2次推导，第1个是推导出y的型别为std:: initializer_list（因为使用了大括号去初始化），第2次是推导std:: initializer_list<T>的T的类型，而如果我们想利用模板去实现这一点是做不到的，因为auto他本身对大括号初始化就假定了第1次推导必定是std:: initializer_list，而模板没办法，所以如果真要用模板，可以这么实现
~~~
template<typename T>
void func(std:: initializer_list<T> list);
~~~
对于func({2,3,3})的调用，上面模板就起效了
最后还补充了1点，在c++14的标准中，可以单独用auto去说明函数返回值/lambda表达式的形参需要推导，但此时他是使用模板推导去推导的，所以说返回值不能使用大括号

条款3是decltype的使用
一般的，对于大多数std的容器，其[]的使用会返回对应位置的元素的引用，除了vector<bool>以外，
而在c++14中，正如在条款2中提到的，我们可以只使用auto不加decltype去推导返回值类型，此时使用的是模板类型的推导，那以下代码就有问题了
~~~
template<typename container, typename index>
auto getindex(container & c,index i){
    return c[i];
}
...
 std::vector<int> vec {3,4,4};
 getindex(vec,2) = 10;
 ...
~~~
上面的代码执行如下：首先这里采用的是模板推导，且auto没有&或者&&的修饰，即采用条款1种的第3种规则，此时型参的一切引用都会被忽略，即c[i]返回的int&被看成是int，那么T和paramtype就是int了，此时返回的是1个临时值，是个右值，不能放在等号左侧，所以很明显他会报错
~~~
error: lvalue required as left operand of assignment
  getindex(vec,2) = 10;
~~~
解决方法是把auto改成decltype（auto），告诉他说推导过程用的是decltype的规则，而他对于int&就是推导为int&，或者我们直接使用后置类型推导，也比较清晰
那这里还有1个不完美的在于，getindex他只能接受1个左值容器，对于一些右值容器，比如说一些工厂函数的返回值，我们直接传入，此时需要用万能引用和完美转发去解决
~~~
template<typename container, typename index>
decltype(auto) func(container && c, index i){
    return std::forward<container>(c)[i];
}
~~~
除此之外，decltype还有一个坑，如果decltype（sth），sth仅仅只是1个变量名，如x，一切如旧；但假如sth是(x),c++仍把他看做是左值表达式，此时decltype必须推导出是一个引用类型，那么对下列代码
~~~
decltype(auto) func(){
    int x = 2;
    return (x);
}
~~~
他会返回1个局部变量的引用，这是一个危险的未定义行为，所以使用decltype一定要小心，里面的东西到底是什么
