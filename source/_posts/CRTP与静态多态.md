---
title: CRTP与静态多态
date: 2022-09-09 09:01:39
tags: c++
---
CuriouslyRecurringTemplatePattern，简称CRTP，是一种实现静态多态的机制，简单而言，他的核心在于：父类是一个模板类，派生类会继承父类，且以派生类自身作为父类的模板参数，如下：
~~~
template<typename T>
class Base{
public:
  void print(){
    static_cast<T*>(*this)->imp();
  }
  void imp(){
    std::cout << "this is base" << std::endl;
  }
};

class Son1: public Base<Son1>{
  void imp(){
    std::cout << "this is son 1" << std::endl;
  }
};

class Son2: public Base<Son2>{
  void imp(){
    std::cout << "this is son 2" << std::endl;
  }
};

template<typename T>
void func(T & t){
  t.print();
}
~~~

当我传入func的对象是Son1时， Base实例化为Son1，print中的static_cast就会把this指针强制转换为Son1*，也就能调用Son1自己实现的函数了，不过这里严格意义上来说并不算是多态，因为每个派生类继承的是各自实例化后的模板类，使用static_cast就能把从基类去访问派生类的成员函数了
似乎llvm的visitor模式采用的就是这种捏，tvm中的貌似也有涉及这种设计，后续再看
