---
title: effective modern c ++ 25-27
date: 2022-09-20 09:16:43
tags: effective_modern_c++
---

条款25讲的还是move和forward的具体使用场景的对比，当明确要使用移动语义时用move，要使用完美转发时使用forward
作者一开始就警告，千万不能在本应该使用forward语义的地方使用move，因为极有可能吧1个左值的内容移动到了调用方，此时的左值就肯定不能被使用了，但这并不是我们需要的
实现了万能引用的函数模板如下：
~~~
class Widget{
public:
  template<typename T>
  void setname(T&& newname){
    name = forward<T>(newname);    
  }
private:
    string name;
};
~~~
我们当然可以提供非模板的setname函数，因为若我们有保证当进来的是左值时不去修改他的要求时，需要把形参声明为const，而此时就不是完美引用了，因为其不能有任何限定词，代码如下
~~~
void setname( const string & newname){
    name = newname;
}
void setname(string && newname){
    name = std::move(newname);
}
~~~
但这种重载的实现方式有几个问题，1是当我们传进来1个字面量，如setname("my")的时候，如果是万能引用+完美转发的做法，这个字面量就直接传到赋值那一行用于构造name了，就像执行name = "my"一样，而重载的右值引用版本，需要首先构造一个临时string对象，再把临时对象的值移动到name中，然后析构这个临时对象，这就可能带来性能效率上的问题；第2个问题就是，如果函数的参数有多个，那要重载的版本就呈指数爆炸的个数了，例如make_shared函数，其声明可以简化如下：
~~~
template<typename T,typename ...Args>
  shared_ptr<T> make_shared(Arg&& ...args){
    return shared_ptr<T>(forward<T>(args)...);
  }
~~~
在这种情况下，重载肯定是1个很费力气的操作。
接下来讲的是1个小原则——当我们对1个变量想使用move语义或者forward时，得保证之后不会再用这个对象，也就是说，只有在最后1次与该变量相关的语句中才能使用move/forward，这个就不用具体解释了。
接下来就是讲按值返回的函数中对移动语义的使用了，这里我们需要分为2种情况
第1种情况是，如果这个返回的内容，本身就是一个引用，不管是什么引用，都要在return语句使用move/forward，如下
~~~
Matrix operator+(Matrix &&lhs, Matrix& rhs){
    lhs += rhs;
    return move(lhs);
}
~~~
假如不使用move，那在return的时候是的的确确会构造一个匿名对象然后再把这个对象的内容复制到返回值的储存位置，而加了移动语义，他就简单的把lhs这个右值移动到返回值的地方，还有可能的场景如下：
~~~
template<typename T>
T func(T&& t){
    dosth;
    return forward<T>(t);
}
~~~
这样子当t是右值的时候，就能直接把他放入返回值的地方，而是左值的时候才会有复制的成本
第2种情况是返回一个局部对象，这里编译器会有一种优化叫做(return value optimization），RVO,即直接在返回值的位置上去创建这个局部变量，从而省去了构造匿名对象并复制的成本
而编译器使用到这个优化的条件非常的苛刻，他需要满足2个条件，1个是局部变量必须和返回值类型一样，这就是说不能有隐式转换在里面，二就是返回的就是局部变量本身，此时假如我们加上了move语义，就相当于返回了一个局部变量的右值引用了，就不满足RVO条件了，这反而是破坏了优化的可能性，而针对这条优化原则，还有相关的补充——如果满足了RVO条件，要么保证不复制（直接放到返回值），要么返回一个右值对象，即和使用move语言一样的效果。那也就是说，不使用move可以达到优化的效果，因为他的确有使用右值语义的机会，但使用了move则破坏了RVO的条件，这反而帮了倒忙
条款26介绍的是避免依万能引用型别去重载，意思就是说，对于一个使用了万能引用作为参数的函数模板，我们要避免去重载同名的函数，具体的例子如下
~~~
template<typename T>
void func(T&& t){
    dosth...;
    string temp (t);
}

void func(int t){
    dosth...;
    string temp;
}
~~~
这样子的话，对下面的函数调用，编译是可以通过的
~~~
func("wm");
string w("ww");
func(w);
func(string("new"));
func(3);
~~~
万能转发避免了许许多多的复制操作，这个是我们的的确确得到的好处，但有时候会有需要传入别的类型的参数的需要，此时我们的直觉就是去重载这个函数，让他来处理某些特定的形参的情况，但这里就有问题了，如下
~~~
short i = 3;
func(i);
~~~
由于调用int版本的重载函数不是完美匹配的，因为还需要隐式类型转换，而函数模板可以实例化出1个完美匹配的模板函数出来，这个叫精确匹配，精确匹配是优先于其他版本的，所以会调用函数模板的那个版本，而这时候问题来了，string并不能接受任何形式的short形参，编译就失败了
问题在于函数模板是十分"贪婪"的东西，只要是他实例化出来的函数就激活都能看做是完美匹配的，也就导致了对short类型的形参无法匹配到int
还有1个例子就是在类中使用万能引用函数模板作为构造函数的，如下
~~~
class Widget{
public:
  template<typename T>{
    Widget(T && n){
      name = n;
    }
  }
  Widget(int n){
    name = "e";
  }
private:
  string name;
};

Widget w(1);

auto Widget(w);
~~~
以上代码会报错，错在找不到1个=运算符，其左右操作数分别为char*和Widget的，也就是说，他这里调用了万能引用的构造函数，可是这里编译器明明是会自动生成复制构造函数的，结果却调用了别的东西。这里的原因还是因为万能引用的“贪婪”，在传入参数是w，也就是Widget的情况下，他既可以实例化widget的构造函数，T为Widget，也可以使用默认的复制构造函数，但这个构造函数的函数的参数是const Widget&，也就是说他并不是完美匹配的，此时编译器就会去找实例化后的模板函数了，也就出现了一个把n（Widget）类型的变量赋值给name（String）的情况，自然会报错
但如果w是个const，自然的复制构造函数也完美匹配了，此时c++的标准规定，要优先使用非模板的函数，也就是说此时编译是可以通过的，但这样也莫名其妙地多了常量性
进一步的，在派生类中，万能引用更是会使得程序的错误更加难以推测，如下
~~~
class Derived: public Widget{
public:
  Derived(){}
  Derived(const Derived& d):Widget(d){}
};
~~~
这段代码会报错，因为在派生类的复制构造函数里，Widget(d)的语句会由于实例化后的模板比基类的复制构造函数（要把d转化为widget类）更加完美匹配，所以就调用了实例化后的构造函数，string类本身肯定是没有derived为参数的构造函数的，肯定过不了编译
所以以上的例子想说明的是，尽量不要在重载函数中使用万能引用
接下来的条款27想解决的就是假如真的需要万能引用做重载函数的形参时，具体的可行方案有舍弃重载，传值，用常量左值引用做形参等，接下来就是本条款的重点——用标签分类，他的思想在于，上述的万能引用，由于对大多数传进来的参数，他都能实例化出1个完美匹配的模板函数出来，那只要我们能做到，对于某些特定的类型，只要他在传进来这个类型后无法实例化就能解决问题，而这可以通过增加1个标志位形参来做到，这个标志位可以利用std::is_integral<T>()做到，具体的初步修改如下
~~~
template<typename T>
void interface(T&& params){
    impl(forward<T>(params),
    std::is_integral<T>());
}
~~~
上面的这段代码，他是把所有的参数都用interface去接受，在内部再根据类型去派发具体的实现，这里后面肯定是需要2个impl的
接下来是需要对这个函数去修改他不合理的地方，1是关于万能引用和引用折叠，如果传进来的参数是左值的话，T就会推导为type&的形式，例如int&，那标志位就会判断为false了，所以我们就需要利用std的移出引用的特性remove_reference了，如下
~~~
template<typename T>
void interface(T&& params){
    impl(forward<T>(params)),
    std::is_intergral<typename std::remove_reference<T>::type>());
}
~~~
其中上述的方括号里的内容可以直接用std::remove_referenct_t<T>代替，因为这个就是使用了using写的类型别名，不用写typename指明他是类型，
接下来就是去写具体的实现函数了，我们需要利用std::false_type和std::true_type去做2个实现函数的标志位，因为true和false是运行时的东西，而我们需要的是一种能在编译期就告诉我们如何重载决议的东西，就是上面的2个*_type了，如下
~~~
template<typename T>
impl(T&& params, std::false_type){
    cout << "this is not integral" << endl;
}

template<typename T>
impl(T&& params, std::true_type){
    cout << "this is intergral" << endl;
}
~~~
到了这里，标签分类的具体内容就ok了，这些其实都算是模板元编程的东西，不过还没学2333，后续再开个坑吧
接下来的另外1个方式是使用enable_if,让万能引用的函数模板在某些条件下禁止实例化，这里我们继续，先写1个初步使用该特性的例子
~~~
class Person{
public:
  template<typename T, typename = enable_if<condition>::type>
  Person(T&& params){
    ...
  }
};
~~~
那么当括号里的condition为true的时候，这个实例化才被允许
接下来我们就需要写出这个condition，我们需要在入参是person的时候调用的是复制构造函数而不是实例化的模板构造函数，所以我们可以搭配std::is_same<Person,T>来判断，当然了这里同样的和上面有一样的问题，如果T实例化之后有引用/const/volatile等修饰，这个判断就为假了，所以我们需要移除这些特性，可以使用std_decay，这里我们可以简单的理解为std::decay<T>::type就是我们需要的东西了
因此该condition可以写为<!is_same<Person, std::decay<T>::type>::value
上面的写法解决了person本身的一个构造函数的重载决议问题了，但对于其派生类来说，当入参是派生类derived时，还是会调用基类模板构造函数而不是基类的复制构造函数，因为derived明显不是person类，解决方法就是使用is_base_of替代is_same即可，即is_base_of<Person,typename = enable_if<!is_base_of<Person,std::decay<T>::type>::value>::type>
这里看上去一大串，还是比较难读的，当然我们可以使用14标准的那些别名，如下
is_base_if<Person,typename = enable_if_t<!is_base_of<Person,decay_t<T>>::value>
总的来说，我们可以使用标签或者enable_of这2个现代c++特性去实现重载模板函数的决议

