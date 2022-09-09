---
title: tvm系列1——te代码阅读
date: 2022-09-08 13:06:11
tags: tvm
---


这篇是想探索一下tvm的te的compute和schedule具体的实现代码，
~~~
n = te.var("n")
A = te.placeholder((n,), name="A")
B = te.placeholder((n,), name="B")
C = te.compute(A.shape, lambda i: A[i] + B[i], name="C")
~~~
上面这段假如熟悉tvm的应该再熟悉不过了，首先第1句话，返回的是tvm.tir.Var的数据类型的变量，这个是tir上的数据结构，后面再解析
下面的A和B的placeholder如下
~~~
def placeholder(shape, dtype=None, name="placeholder"):
    shape = (shape,) if isinstance(shape, tvm.tir.PrimExpr) else shape
    dtype = "float32" if dtype is None else dtype
    return _ffi_api.Placeholder(shape, dtype, name)
~~~
这个tvm.tir.PrimExpr是tir大多数类的父类，然后就会调用ffi机制去使用c++写的代码，这里也没啥可以说的，返回的就是

到了compute，这里源码的一开始一大段都是处理参数变量名称的，不用理会，这里他会if else到最后，直接把argspec.args当做arg——names，这里他是使用inspect的getfullargspec去获取一个lambda表达式的所有信息的
到下面
dim_var = [tvm.tir.IterVar((0, s), x, 0) for x, s in zip(arg_names, shape[:out_ndim])]
    body = fcompute(*[v.var for v in dim_var])

out_ndim是第1个参数的维度，这里是1，然后s是只有1个，就是n，会用他们去构造IterVar，第1个参数是这个iter的范围，第2个是这个iter的标识，第3个是这个iter的类型，源码中写着他是datapar，应该是一般的那种iter这里构造出来的dim_var打印如下：
~~~
~~~
接下来的body部分的var其实就是上面的第2个参数，fcompute就是C中的lambda表达式，首先把var的列表给解包，在调用fcompute这个可调用对象，就是上面C的lambda表达式，这里我们再写1个看看
~~~
n = te.var("n")
A = te.placeholder((n,n), name="A")
B = te.placeholder((n,n), name="B")
C = te.compute(A.shape, lambda i,j: A[i,j] + B[i,j], name="C")
~~~
这里返回的body的类型是tvm.tir.expr.Add,主要是因为A和B都是tvm.te.Tensor,他们继承自ExprOp类，而这个类又写了一堆魔法方法，重载了一系列的运算符，比如说这里的+运算符，写了__add__函数后，最终调用这个函数
假如说在compute中，有te.sum这种reduce操作的，还会识别出其中达到reduce_axis,
