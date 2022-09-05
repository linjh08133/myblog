---
layout: init
title: shared_ptr线程安全
date: 2022-09-05 15:36:09
tags: c++
---

shared_ptr众所周知的智能指针，其允许多个指针指向同一内存对象，且在引用计数为0的时候自动析构被管理的对象，但是，在多线程的环境下，他的操作不是线程安全的，
原因在于，其管理对象的方式是通过指针去管理，而其底层的引用计数本身也是一个指针，指向一个真正的计数对象，当我们执行如下代码的时候
~~~
shared_ptr<A> a1 (new A));
shared_ptr<A> a2 = a1;
~~~
a2在构造的时候，是分为2步的，第1步是让a2管理的A对象指向a1管理的对象，第2步是让a2的引用计数也指向a1的引用计数对象，然后再把count+1
那么这种非原子的操作方式就可能带来race condition了，如下代码
~~~
#include <memory>
#include <iostream>
#include <thread>

std::shared_ptr<int> p1 (new int(5));

void func1(){
  std::shared_ptr<int> p2;
  p2 = p1;
  std::cout << *p2 << std::endl;
}

void func2(){
  std::shared_ptr<int> p3;
  p1 = p3;
}

int main(){
  std::thread t1(func1);
  std::thread t2(func2);
  t1.join();
  t2.join();
}
~~~
当线程1执行p2 = p1的时候，首先他会把p1管理对象的指针赋值给p2，但这个时候，线程2来了，他的p1 = p3的赋值操作，导致p1原来管理的int(5)变成了1个没人指向的对象，所以其对应的引用计数也为0，且这个原来的对象就被析构了，此时p1所指的引用计数对象，他的count是2，再下一步，来到线程1，p2 = p1指令继续赋值，把p1的新的引用计数对象赋给了p2，那么这个引用计数对象的count就是3了，但此时p2所指的是那个已经被析构了的int(5),这个时候我们再解引用p2，就会报错了，如图：
![](/images/shared_ptr线程安全/dump.png)
中间的那个偶尔的段错误吐核就是啦
所以这里要记住：shared_ptr的实现机制，最核心的就是使用2个指针，指向1个被管理对象和1个与之关联的引用计数对象，在赋值的时候是分2步的非原子操作，所以这个时候一定要加锁使其原子化

