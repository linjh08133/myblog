---
title: effective modern c ++ 22-24
date: 2022-09-15 15:01:26
tags:
---

接下来进入第5大章，也就是右值，移动语义和完美转发等一系列11中最重要的一大特性领域
条款23讲的是move和forward这2个最常见到的2个函数，他们的本质只不过是进行了强制类型转换，前者无条件的把传进来的参数转换为右值，forward则是在某种特定条件下才把参数进行转化，move的实现大致如下
~~~
template<typename T>
typename remove_reference<T>::type&&
move(T && params){
    using ReturnType =
      typename remove_reference<T>::type&&;
    return static_cast<ReturnType>(params);
}
~~~
这里返回值是个右值，且为了不会因为传进来的是左值以及引用折叠导致说返回的变成了左值，需要使用remove_reference
传给一个函数，或者构造函数一个move后的右值，往往是想进行移动的操作，所以才叫做move
不过一个move的东西，他有时候不一定会进行所谓的移动操作，假如说一个函数他的参数是const的，接着我们调用move把这个参数从const &变为了const &&，但此时他也只能匹配到复制构造函数了，因为移动构造函数只能接受非常的右值参数
forward的使用场景也是十分的经典了，就是在万能引用下，因为引用本身是个左值了，所以如果把万能引用的参数传入某个函数，永远也只会调用这个函数的左值版本，如下
~~~

void func(int & num){
    cout << "left" << num << endl;
}

void func(int && num){
    cout <<"right" << num << endl;
}

template<typename T>
void interface(T&& params){
    func(params);
    func(forward<T>(params));
}

int main(){
    int i = 2;
    interface(i);
    cout << "----" << endl;
    interface(3);
}
~~~
两者的作用看上去似乎十分相似，但其实是十分泾渭分明的，move明确指定了要把参数转换为右值，为相关的移动操作做准备，但forward只是转发，参数是左值就转发为左值，是右值就转发为右值，2者并不是作用可以被一方替代的关系
条款24讲的就是万能引用和右值引用的相关内容了
万能引用最主要的2个使用方式就是如下
~~~
template<typename T>
void func(T && params){...}

auto && x = sth;
~~~
其最本质的，就是涉及到了型别推导，所以如果遇到具体的类，如Widget&&,他就只是1个简单的右值引用，因为他根本就没有型别推导
而且这里的型别推导也必须得限制住，必须得是T&&的形式，其他的就不行了，如下
~~~
template<typename T>
void func(vector<T>&& params)
{...}
~~~
上面的params是个右值引用，而且，即使是加上const关键词，也会导致其不是一个万能引用
而且就算是已经
