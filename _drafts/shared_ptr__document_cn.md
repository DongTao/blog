---
layout: post
title: boost shared_ptr 文档翻译
categories:
- C++
- boost
---
<a name="Introduction">

Introduction 简介
======

The *shared_ptr* class template stores a pointer to a dynamically allocated object, typically with a C++ *new-expression*. The object pointed to is guaranteed to be deleted when the last shared_ptr pointing to it is destroyed or reset.
>shared_ptr持有一个指向动态分配对象的指针(通常由C++的new操作符创建)，它确保在最后一个持有该指针的shared_ptr对象销毁或调用reset方法时释放该对象。

*Example*
~~~cpp
shared_ptr<X> p1( new X );
shared_ptr<void> p2( new int(5) );
~~~

*shared_ptr* deletes the exact pointer that has been passed at construction time, complete with its original type, regardless of the template parameter. In the second example above, when *p2* is destroyed or reset, it will call *delete* on the original *int*\* that has been passed to the constructor, even though *p2* itself is of type *shared_ptr<void>* and stores a pointer of type *void*\*.
>shared_ptr销毁指向的对象时对构造时传入的指针进行delete调用，与模板参数无关，所以即便是shared_ptr<void>指向的对象也可以安全释放。<!--more-->

Every *shared_ptr* meets the *CopyConstructible*, *MoveConstructible*, *CopyAssignable* and *MoveAssignable* requirements of the C++ Standard Library, and can be used in standard library containers. Comparison operators are supplied so that *shared_ptr* works with the standard library's associative containers.
>shared_ptr支持拷贝构造，移动构造，拷贝赋值，移动赋值等语义，也支持比较操作符，所以可以完全兼容STL容器。另外shared_ptr还支持比较操作符，所以可以在STL关联容器中作为键值使用。

Because the implementation uses reference counting, cycles of *shared_ptr* instances will not be reclaimed. For example, if *main()* holds a *shared_ptr* to *A*, which directly or indirectly holds a *shared_ptr* back to *A*, *A*'s use count will be 2. Destruction of the original *shared_ptr* will leave A dangling with a use count of 1. Use [weak_ptr](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/weak_ptr.htm) to "break cycles."
>由于shared_ptr采用引用计数实现，所以存在循环引用问题。例如，如果main()函数持有指向A的shared_ptr，而A又直接或间接持有一个指向自己的shared_ptr，A的引用计数将会是2。原始shared_ptr销毁后，引用计数为1，A将永远无法释放。使用weak_ptr可以打破循环引用。

The class template is parameterized on *T*, the type of the object pointed to. *shared_ptr* and most of its member functions place no requirements on T; it is allowed to be an incomplete type, or *void*. Member functions that do place additional requirements ([constructors](#pointer_constructor), [reset](#reset)) are explicitly documented below.
>shared_ptr和它的大部分成员函数对模板参数T没有依赖，T甚至可以是非完整类型或void，需要申请内存的成员函数(如构造函数，reset函数)在后面单独说明。

*shared_ptr<T>* can be implicitly converted to *shared_ptr<U>* whenever *T*\* can be implicitly converted to *U*\*. In particular, *shared_ptr<T>* is implicitly convertible to *shared_ptr<T const>*, to *shared_ptr<U>* where *U* is an accessible base of *T*, and to *shared_ptr<void>*.
>如果T\*可以隐式转换为U\*，那么shared_ptr<T>就可以隐式转换为shared_ptr<U>。特别的，shared_ptr<T>可以隐式转换为shared_ptr<T const>，shared_ptr<U>(如果U是T的可访问基类)和shared_ptr<void>。

*shared_ptr* is now part of the C++11 Standard, as *std::shared_ptr*.
>shared_ptr已经是C++11的一部分，属于std命名空间。

Starting with Boost release 1.53, *shared_ptr* can be used to hold a pointer to a dynamically allocated array. This is accomplished by using an array type (*T[]* or *T[N]*) as the template parameter. There is almost no difference between using an unsized array, *T[]*, and a sized array, *T[N]*; the latter just enables *operator[]* to perform a range check on the index.
>Boost 1.53开始shared_ptr就已经可以用于保存堆上数组了，只需在模板参数中使用数组类型即可，如T[]或T[N]，带不带N的差异很小，后者会在使用[]操作符的时候进行越界检查。

*Example*
~~~cpp
shared_ptr<double[1024]> p1( new double[1024] );
shared_ptr<double[]> p2( new double[n] );
~~~

<a name="Bset_Practices">

Best Practices 最佳实践
======

A simple guideline that nearly eliminates the possibility of memory leaks is: always use a named smart pointer variable to hold the result of *new*. Every occurence of the *new* keyword in the code should have the form:
>消除内存泄露可能性的一个指导原则是：总是使用智能指针来持有new操作符的结果，通常是以下形式：
~~~cpp
shared_ptr<T> p(new Y);
~~~

It is, of course, acceptable to use another smart pointer in place of *shared_ptr* above; having *T* and *Y* be the same type, or passing arguments to *Y*'s constructor is also OK.
>使用其他类型的智能指针替代shared_ptr也是可以的，关键是上面的形式：T和Y是同种类型，Y可以传入构造函数所需的参数。

If you observe this guideline, it naturally follows that you will have no explicit *delete* statements; *try/catch* constructs will be rare.
>如果遵守这条指导原则，你的程序中将不再有显式的delete表达式，try/catch语句在构造函数中也会很罕见。

Avoid using unnamed *shared_ptr* temporaries to save typing; to see why this is dangerous, consider this example:
>避免使用未命名的shared_ptr临时变量来节省字符，这可能导致内存泄露，示例如下：

~~~cpp
void f(shared_ptr<int>, int);
int g();

void ok()
{
    shared_ptr<int> p( new int(2) );
        f( p, g() );
}

void bad()
{
     f( shared_ptr<int>( new int(2) ), g() );
}
~~~

The function *ok* follows the guideline to the letter, whereas *bad* constructs the temporary *shared_ptr* in place, admitting the possibility of a memory leak. Since function arguments are evaluated in unspecified order, it is possible for *new int(2)* to be evaluated first, *g()* second, and we may never get to the *shared_ptr* constructor if *g* throws an exception. See [Herb Sutter's treatment](http://www.gotw.ca/gotw/056.htm) (also [here](http://www.cuj.com/reference/articles/2002/0212/0212_sutter.htm)) of the issue for more information.
>ok 函数遵守了指导原则，而 bad 函数在 f 的参数中构造shared_ptr，这种方式有内存泄露风险，因为函数参数表达式执行顺序是未定义的，可能出现new int(2)、g()、shared_ptr<int>()这样的顺序，如果g()抛出异常，shared_ptr<int>构造函数是不会执行的，自然new int(2)申请的内存也无法释放了。更多信息见 Herb Sutter的办法。

The exception safety problem described above may also be eliminated by using the *make_shared* or *allocate_shared* factory functions defined in *boost/make_shared.hpp*. These factory functions also provide an efficiency benefit by consolidating allocations.
>上面提到的异常安全问题在使用make_shared或allocate_shared等工厂函数时同样可能发生，请注意避免。
