---
layout: post
title: boost shared_ptr 文档翻译
categories:
- C++
- boost
---

------------------------------

>文档版本对应boost 1.63.0  
>转载请保留引用  
> \<马丁的Blog\>  
><http://dongtao.github.io/blog/>

-------------------------------

**[ Introduction ](#Introduction)**  
**[ Best Practices](#Best_Practices)**  
**[ Synopsis ](#Synopsis)**  
**[ Members ](#Members)**  
**[Free Functions](#Free_Functions)**  
**[ Example ](#Example)**  
**[Handle/Body Idiom](#Handle_Body__Idiom)**  
**[Thread Safety](#Thread_Safety)**  
**[Frequently Asked Questions](#FAQ)**  
**[Smart Pointer Timings](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/smarttests.htm)**  
**[Programming Techniques](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/sp_techniques.html)**  

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

<a name="Best_Practices">

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

The exception safety problem described above may also be eliminated by using the *[make_shared](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/make_shared.html)* or *[allocate_shared](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/make_shared.html)* factory functions defined in *boost/make_shared.hpp*. These factory functions also provide an efficiency benefit by consolidating allocations.
>上面提到的异常安全问题在使用make_shared或allocate_shared等工厂函数时同样可能发生，请注意避免。

<a name="Synopsis">

Synopsis 概要
======
~~~cpp
namespace boost {

  class bad_weak_ptr: public std::exception;

  template<class T> class weak_ptr;

  template<class T> class shared_ptr {

    public:

      typedef see below element_type;

      shared_ptr(); // never throws
      shared_ptr(std::nullptr_t); // never throws

      template<class Y> explicit shared_ptr(Y * p);
      template<class Y, class D> shared_ptr(Y * p, D d);
      template<class Y, class D, class A> shared_ptr(Y * p, D d, A a);
      template<class D> shared_ptr(std::nullptr_t p, D d);
      template<class D, class A> shared_ptr(std::nullptr_t p, D d, A a);

      ~shared_ptr(); // never throws

      shared_ptr(shared_ptr const & r); // never throws
      template<class Y> shared_ptr(shared_ptr<Y> const & r); // never throws

      shared_ptr(shared_ptr && r); // never throws
      template<class Y> shared_ptr(shared_ptr<Y> && r); // never throws

      template<class Y> shared_ptr(shared_ptr<Y> const & r, element_type * p); // never throws

      template<class Y> shared_ptr(shared_ptr<Y> && r, element_type * p); // never throws

      template<class Y> explicit shared_ptr(weak_ptr<Y> const & r);

      template<class Y> explicit shared_ptr(std::auto_ptr<Y> & r);
      template<class Y> shared_ptr(std::auto_ptr<Y> && r);

      template<class Y, class D> shared_ptr(std::unique_ptr<Y, D> && r);

      shared_ptr & operator=(shared_ptr const & r); // never throws
      template<class Y> shared_ptr & operator=(shared_ptr<Y> const & r); // never throws

      shared_ptr & operator=(shared_ptr const && r); // never throws
      template<class Y> shared_ptr & operator=(shared_ptr<Y> const && r); // never throws

      template<class Y> shared_ptr & operator=(std::auto_ptr<Y> & r);
      template<class Y> shared_ptr & operator=(std::auto_ptr<Y> && r);

      template<class Y, class D> shared_ptr & operator=(std::unique_ptr<Y, D> && r);

      shared_ptr & operator=(std::nullptr_t); // never throws

      void reset(); // never throws

      template<class Y> void reset(Y * p);
      template<class Y, class D> void reset(Y * p, D d);
      template<class Y, class D, class A> void reset(Y * p, D d, A a);

      template<class Y> void reset(shared_ptr<Y> const & r, element_type * p); // never throws

      T & operator*() const; // never throws; only valid when T is not an array type
      T * operator->() const; // never throws; only valid when T is not an array type

      element_type & operator[](std::ptrdiff_t i) const; // never throws; only valid when T is an array type

      element_type * get() const; // never throws

      bool unique() const; // never throws
      long use_count() const; // never throws

      explicit operator bool() const; // never throws

      void swap(shared_ptr & b); // never throws

      template<class Y> bool owner_before(shared_ptr<Y> const & rhs) const; // never throws
      template<class Y> bool owner_before(weak_ptr<Y> const & rhs) const; // never throws
  };

  template<class T, class U>
    bool operator==(shared_ptr<T> const & a, shared_ptr<U> const & b); // never throws

  template<class T, class U>
    bool operator!=(shared_ptr<T> const & a, shared_ptr<U> const & b); // never throws

  template<class T, class U>
    bool operator<(shared_ptr<T> const & a, shared_ptr<U> const & b); // never throws

  template<class T>
    bool operator==(shared_ptr<T> const & p, std::nullptr_t); // never throws

  template<class T>
    bool operator==(std::nullptr_t, shared_ptr<T> const & p); // never throws

  template<class T>
    bool operator!=(shared_ptr<T> const & p, std::nullptr_t); // never throws

  template<class T>
    bool operator!=(std::nullptr_t, shared_ptr<T> const & p); // never throws

  template<class T> void swap(shared_ptr<T> & a, shared_ptr<T> & b); // never throws

  template<class T> typename shared_ptr<T>::element_type * get_pointer(shared_ptr<T> const & p); // never throws

  template<class T, class U>
    shared_ptr<T> static_pointer_cast(shared_ptr<U> const & r); // never throws

  template<class T, class U>
    shared_ptr<T> const_pointer_cast(shared_ptr<U> const & r); // never throws

  template<class T, class U>
    shared_ptr<T> dynamic_pointer_cast(shared_ptr<U> const & r); // never throws

  template<class T, class U>
    shared_ptr<T> reinterpret_pointer_cast(shared_ptr<U> const & r); // never throws

  template<class E, class T, class Y>
    std::basic_ostream<E, T> & operator<< (std::basic_ostream<E, T> & os, shared_ptr<Y> const & p);

  template<class D, class T>
    D * get_deleter(shared_ptr<T> const & p);
}
~~~

<a name="Members">

Members 成员
======

<a name="element_type">

element_type - 类型定义
------
~~~cpp
typedef ... element_type;
~~~

element_type is T when T is not an array type, and U when T is U[] or U[N].  
非数组类型element_type命名为T，数组类型element_type命名为U

<a name="default_constructor">

default constructor - 默认构造函数
------
~~~cpp
shared_ptr(); // never throws
shared_ptr(std::nullptr_t); // never throws
~~~

+ **Effects**: Constructs an empty shared_ptr.  
效果：构造空的shared_ptr

+ **Postconditions**: use_count() == 0 && get() == 0.  
后置条件：use_count() == 0 && get() == 0

+ **Throws**: nothing.  
异常不抛掷

*[The nothrow guarantee is important, since **reset()** is specified in terms of the default constructor; this implies that the constructor must not allocate memory.]*  
*[异常不抛掷保证非常重要，**因为reset函数是根据默认构造函数指定的**；这暗示了默认构造函数中不允许分配内存(否则可能会导致内存泄露)]*
~~~cpp
//##Martin's NOTE##
//shared_ptr::reset函数源码，下面的this_type()是上面描述的原因所在
void reset() BOOST_NOEXCEPT // never throws in 1.30+
{
     this_type().swap(*this);
}
~~~

<a name="pointer_constructor">

pointer constructor - 指针构造函数
------
~~~cpp
template<class Y> explicit shared_ptr(Y * p);
~~~

+ **Requirements:**Y must be a complete type. The expression delete[] p, when T is an array type, or delete p, when T is not an array type, must be well-formed, must not invoke undefined behavior, and must not throw exceptions. When T is U[N], Y (*) [N] must be convertible to T*; when T is U[], Y (*) [] must be convertible to T*; otherwise, Y* must be convertible to T*.  
要求：Y必须是**完整类型**。delete或delete[]表达式触发的析构操作必须被正确定义，不能引起未定义行为，也不能抛掷异常。当T是U[N]时，Y(*)[N]必须可以被转换为T*，当T是U[]时，Y(*)[]必须可以被转换为T*；否则Y*必须可转换为T*

<a name="complete_type">

~~~
    ##Martin's NOTE##
    Complete Type - 完整类型
    Incomplete Type - 不完整类型

    这两者相对，MSDN上对于哪些类型属于Incomplete Type是这样描述的：
    - A structure type whose members you have not yet specified. (成员不确定的结构类型，包括struct和class)
    - A union type whose members you have not yet specified. (成员不确定的联合类型)
    - An array type whose dimension you have not yet specified. (没指定大小的数组)
    其他都属于Complete Type

    准确一些的定义就是编译期能够确定其大小的类型就是完整类型，而只有声明缺少定义的类型就是不完整类型。
~~~


+ **Effects**: When T is not an array type, constructs a shared_ptr that owns the pointer p. Otherwise, constructs a shared_ptr that owns p and a deleter of an unspecified type that calls delete[] p.  
效果：如果T不是数组类型，构造保存指针p的shared_ptr对象，否则构造的shared_ptr对象除了指针p以外，还包含一个未指定类型的deleter用来调用 delete[] p

+ **Postconditions**: use_count() == 1 && get() == p. If T is not an array type and p is unambiguously convertible to [enable_shared_from_this](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/enable_shared_from_this.html)\<V\>\* for some V, p->shared_from_this() returns a copy of \*this.  
后置条件：use_count() == 1 && get() == p。如果 T 是非数组类型且 p 可以明确转换为 enable_shared_from_this\<V\>\* 类型，那么p->shared_from_this()返回 \*this 的拷贝

+ **Throws**: std::bad_alloc, or an implementation-defined exception when a resource other than memory could not be obtained.  
异常：内存分配失败抛出 std::bad_alloc 异常，其他资源分配失败抛出实现定义的异常

+ **Exception safety**: If an exception is thrown, the constructor calls delete[] p, when T is an array type, or delete p, when T is not an array type.  
异常安全性：如果构造函数中抛出异常，对应的delete或delete[] p会被执行

+ **Notes**: p must be a pointer to an object that was allocated via a C++ new expression or be 0. The postcondition that [use count](#use_count) is 1 holds even if p is 0; invoking delete on a pointer that has a value of 0 is harmless.  
注意：p 必须是指向使用C++ new操作符分配内存对象的指针，或是0值。即便 p 为0值也满足后置条件，use_count() == 1；而 delete 一个0值指针是无害的

*[This constructor is a template in order to remember the actual pointer type passed. The destructor will call delete with the same pointer, complete with its original type, even when T does not have a virtual destructor, or is void.]*  
*[此构造函数的主要目标是保存传入指针的原始类型。shared_ptr的析构函数会对传入时类型的指针调用delete，即便T没有虚析构函数或者是void]*

<a name="constructors_taking_a_deleter">

constructors taking a deleter - 传入删除器参数的构造函数
------
~~~cpp
template<class Y, class D> shared_ptr(Y * p, D d);
template<class Y, class D, class A> shared_ptr(Y * p, D d, A a);
template<class D> shared_ptr(std::nullptr_t p, D d);
template<class D, class A> shared_ptr(std::nullptr_t p, D d, A a);
~~~

+ **Requirements**: D must be CopyConstructible. The copy constructor and destructor of D must not throw. The expression d(p) must be well-formed, must not invoke undefined behavior, and must not throw exceptions. A must be an Allocator, as described in section 20.1.5 (Allocator requirements) of the C++ Standard. When T is U[N], Y (*) [N] must be convertible to T*; when T is U[], Y (*) [] must be convertible to T*; otherwise, Y* must be convertible to T*.  
要求：D必须是可拷贝构造的类型。D的拷贝构造函数和析构函数不能抛掷异常。表达式 d(p) 必须被正确定义，不能引起未定义行为，也不能抛掷异常。A必须是符合C++标准20.1.5节(空间配置器的要求)的一个配置器。当T是U[N]时，Y(*)[N]必须可以被转换为T*，当T是U[]时，Y(*)[]必须可以被转换为T*；否则Y*必须可转换为T*

+ **Effects**: Constructs a shared_ptr that owns the pointer p and the deleter d. The constructors taking an allocator a allocate memory using a copy of a.  
效果：构造保存指针 p 和删除器 d 的 shared_ptr 对象。此构造函数使用配置器 a 的一个拷贝来进行内存分配。

+ **Postconditions**: use_count() == 1 && get() == p. If T is not an array type and p is unambiguously convertible to [enable_shared_from_this](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/enable_shared_from_this.html)\<V\>\* for some V, p->shared_from_this() returns a copy of \*this.  
后置条件：use_count() == 1 && get() == p。如果 T 是非数组类型且 p 可以明确转换为 enable_shared_from_this\<V\>\* 类型，那么p->shared_from_this()返回 \*this 的拷贝

+ **Throws**: std::bad_alloc, or an implementation-defined exception when a resource other than memory could not be obtained.  
异常：内存分配失败抛出 std::bad_alloc 异常，其他资源分配失败抛出实现定义的异常

+ **Exception safety**: If an exception is thrown, d(p) is called.  
异常安全性：如果此构造函数抛出异常，表达式 d(p) 会被执行

+ **Notes**: When the the time comes to delete the object pointed to by p, the stored copy of d is invoked with the stored copy of p as an argument.  
注意：shared_ptr中保存了原始指针 p 的拷贝和删除器 d 的拷贝，触发删除 p 指向的对象时，p 的拷贝作为参数传入 d 的拷贝中执行

*[Custom deallocators allow a factory function returning a shared_ptr to insulate the user from its memory allocation strategy. Since the deallocator is not part of the type, changing the allocation strategy does not break source or binary compatibility, and does not require a client recompilation. For example, a "no-op" deallocator is useful when returning a shared_ptr to a statically allocated object, and other variations allow a shared_ptr to be used as a wrapper for another smart pointer, easing interoperability.*  
*[自定义释放器允许工厂函数返回shared_ptr来隔离用户和内存分配策略。由于释放器并非类型的一部分，修改内存分配策略不会破坏源码和二进制兼容性，因此也就不需要客户端重编译。举例来说，无操作的释放器对于指向静态分配对象的shared_ptr很有用，其他的情形还有使用shared_ptr作为别的智能指针的封装来增加互操作性*

*The support for custom deallocators does not impose significant overhead. Other shared_ptr features still require a deallocator to be kept.*  
*支持自定义释放器并不会带来过大开销。其他一些shared_ptr特性也需要释放器的支持*

*The requirement that the copy constructor of D does not throw comes from the pass by value. If the copy constructor throws, the pointer would leak.]*  
*要求D的拷贝构造函数不抛掷异常是因为shared_ptr构造函数中是采用的传值方式，如果D的拷贝构造函数抛出异常就会导致内存泄漏]*

<a name="copy_and_converting_constructors">

copy and converting constructors - 拷贝与转换构造函数
------
~~~cpp
shared_ptr(shared_ptr const & r); // never throws
template<class Y> shared_ptr(shared_ptr<Y> const & r); // never throws
~~~

+ **Requires**: Y\* should be convertible to T\*.  
需求：Y\*可以转型为T\*

+ **Effects**: If r is empty, constructs an empty shared_ptr; otherwise, constructs a shared_ptr that **shares ownership with** r.  
效果：如果r是空的shared_ptr，构造一个空shared_ptr；否则构造一个shared_ptr与r**共享所有权**

<a name="shares_ownership_with">

~~~
    ##Martin's NOTE##
    shares ownership with - 共享所有权
    实际上共享的是引用计数，所以所有权并不太准确，其实是共享对象释放权;)
~~~

+ **Postconditions**: get() == r.get() && use_count() == r.use_count().  
后置条件：get() == r.get() && use_count() == r.use_count()

+ **Throws**: nothing.  
异常不抛掷

<a name="move_constructors">

move constructors - 移动构造函数
------
~~~cpp
shared_ptr(shared_ptr && r); // never throws
template<class Y> shared_ptr(shared_ptr<Y> && r); // never throws
~~~

+ **Requires**: Y\* should be convertible to T\*.  
要求：Y\* 可以转型为T\* 

+ **Effects**: Move-constructs a shared_ptr from r.  
效果：从r移动构造新的shared_ptr

+ **Postconditions**: *this contains the old value of r. r is empty and r.get() == 0.  
后置条件：*this包含r的原始值。r成为空的shared_ptr且 r.get() == 0

+ **Throws**: nothing.  
异常不抛掷

<a name="aliasing_constructors">

aliasing constructor - 别名构造函数
------
~~~cpp
template<class Y> shared_ptr(shared_ptr<Y> const & r, element_type * p); // never throws
~~~

+ **Effects**: constructs a shared_ptr that shares ownership with r and stores p.  
效果：构造shared_ptr共享r的所有权(引用计数)并保存p

+ **Postconditions**: get() == p && use_count() == r.use_count().  
后置条件：get() == p && use_count() == r.use_count()

+ **Throws**: nothing.  
异常不抛掷

<a name="aliasing_move_constructors">

aliasing move constructor - 别名移动构造函数
------
~~~cpp
template<class Y> shared_ptr(shared_ptr<Y> && r, element_type * p); // never throws
~~~

+ **Effects**: Move-constructs a shared_ptr from r, while storing p instead.  
效果：从r移动构造shared_ptr对象，但保存p

+ **Postconditions**: get() == p and use_count() equals the old count of r. r is empty and r.get() == 0.  
后置条件：get() == p && use_count()等于r的原始引用计数。r成为空shared_ptr且 r.get() == 0

+ **Throws**: nothing.  
异常不抛掷

<a name="weak_ptr_constructor">

weak_ptr constructor - weak_ptr构造函数
------
~~~cpp
template<class Y> explicit shared_ptr(weak_ptr<Y> const & r);
~~~

+ **Requires**: Y\* should be convertible to T\*.  
要求：Y\* 可以转型为T\* 

+ **Effects**: Constructs a shared_ptr that shares ownership with r and stores a copy of the pointer stored in r.  
效果：构造与r共享引用计数的shared_ptr，并保存r存储指针的拷贝

+ **Postconditions**: use_count() == r.use_count().  
后置条件：use_count() == r.use_count()

+ **Throws**: bad_weak_ptr when r.use_count() == 0.  
异常：当r.use_count() == 0 时抛出bad_weak_ptr异常

+ **Exception safety**: If an exception is thrown, the constructor has no effect.  
异常安全性：即便构造函数抛出异常也不产生副作用

<a name="auto_ptr_constructors">

auto_ptr constructors - auto_ptr构造函数
------
~~~cpp
template<class Y> shared_ptr(std::auto_ptr<Y> & r);
template<class Y> shared_ptr(std::auto_ptr<Y> && r);
~~~

+ **Requires**: Y\* should be convertible to T\*.  
要求：Y\* 可以转型为T\* 

+ **Effects**: Constructs a shared_ptr, as if by storing a copy of r.release().  
效果：构造shared_ptr保存r.release()返回值的拷贝

+ **Postconditions**: use_count() == 1.  
后置条件：use_count() == 1

+ **Throws**: std::bad_alloc, or an implementation-defined exception when a resource other than memory could not be obtained.  
异常：内存分配失败抛出 std::bad_alloc 异常，其他资源分配失败抛出实现定义的异常

+ **Exception safety**: If an exception is thrown, the constructor has no effect.  
异常安全性：即便构造函数抛出异常也不产生副作用

<a name="unique_ptr_constructor">

unique_ptr constructor - unique_ptr构造函数
------
~~~cpp
template<class Y, class D> shared_ptr(std::unique_ptr<Y, D> && r);
~~~

+ **Requires**: Y\* should be convertible to T\*.  
要求：Y\* 可以转型为T\* 

+ **Effects**: Equivalent to shared_ptr(r.release(), r.get_deleter()) when D is not a reference type. Otherwise, equivalent to shared_ptr(r.release(), del), where del is a deleter that stores the reference rd returned from r.get_deleter() and del(p) calls rd(p).  
效果：当D不是引用类型时，等同于 shared_ptr(r.release(), r.get_deleter())。否则等同于 shared_ptr()r.release(), del)，del是一个删除器，保存r.get_deleter()返回的引用rd， del(p) 中会调用 rd(p) 

+ **Postconditions**: use_count() == 1.  
后置条件：use_count() == 1

+ **Throws**: std::bad_alloc, or an implementation-defined exception when a resource other than memory could not be obtained.  
异常：内存分配失败抛出 std::bad_alloc 异常，其他资源分配失败抛出实现定义的异常

+ **Exception safety**: If an exception is thrown, the constructor has no effect.  
异常安全性：即便构造函数抛出异常也不产生副作用

<a name="destructor">

destructor - 析构函数
------
~~~cpp
~shared_ptr(); // never throws
~~~


+ **Effects**:  
效果：
    * If *this is empty, or shares ownership with another shared_ptr instance (use_count() > 1), there are no side effects.  
    如果 *this 是空对象，或者与其他shared_ptr共享所有权(use_count() > 1)，就没有副作用
    * Otherwise, if *this owns a pointer p and a deleter d, d(p) is called.  
    如果 *this 持有一个指针p和一个删除器d，调用 d(p)
    * Otherwise, *this owns a pointer p, and delete p is called.  
    如果 *this只持有一个指针p，调用 delete p

+ **Throws**: nothing.  
异常不抛掷

<a name="assignment">

assignment - 操作符
------
~~~cpp
shared_ptr & operator=(shared_ptr const & r); // never throws
template<class Y> shared_ptr & operator=(shared_ptr<Y> const & r); // never throws
template<class Y> shared_ptr & operator=(std::auto_ptr<Y> & r);
~~~

+ **Effects**: Equivalent to shared_ptr(r).swap(*this).  
效果：等同于 shared_ptr(r).swap(*this)

+ **Returns**: *this.  
返回值： *this

+ **Notes**: The use count updates caused by the temporary object construction and destruction are not considered observable side effects, and the implementation is free to meet the effects (and the implied guarantees) via different means, without creating a temporary. In particular, in the example:  
注意：临时shared_ptr对象构造和析构引起的引用计数变化不被视为可观察的副作用，在实现上可以通过各种不同的方法避免这种影响，并且不创建临时对象。比如下面例子：
~~~cpp
shared_ptr<int> p(new int);
shared_ptr<void> q(p);
p = p;
q = p;
~~~
both assignments may be no-ops.  
上面的两个=操作符可能都没有执行操作

~~~cpp
shared_ptr & operator=(shared_ptr && r); // never throws
template<class Y> shared_ptr & operator=(shared_ptr<Y> && r); // never throws
template<class Y> shared_ptr & operator=(std::auto_ptr<Y> && r);
template<class Y, class D> shared_ptr & operator=(std::unique_ptr<Y, D> && r);
~~~

+ **Effects**: Equivalent to shared_ptr(std::move(r)).swap(\*this).  
效果：等同于 shared_ptr(std::move(r)).swap(\*this)

+ **Returns**: *this.  
返回值： *this

~~~cpp
shared_ptr & operator=(std::nullptr_t); // never throws
~~~

+ **Effects**: Equivalent to shared_ptr().swap(\*this).  
效果：等同于 shared_ptr().swap(\*this)

+ **Returns**: *this.  
返回值： *this

<a name="reset">

reset - 重置
------
~~~cpp
void reset(); // never throws
~~~

+ **Effects**: Equivalent to shared_ptr().swap(\*this).  
效果：等同于 shared_ptr().swap(\*this)

~~~cpp
template<class Y> void reset(Y * p);
~~~
+ **Effects**: Equivalent to shared_ptr(p).swap(\*this).  
效果：等同于 shared_ptr(p).swap(\*this)

~~~cpp
template<class Y, class D> void reset(Y * p, D d);
~~~

+ **Effects**: Equivalent to shared_ptr(p, d).swap(\*this).  
效果：等同于 shared_ptr(p, d).swap(\*this).

~~~cpp
template<class Y, class D, class A> void reset(Y * p, D d, A a);
~~~

+ **Effects**: Equivalent to shared_ptr(p, d, a).swap(\*this).  
效果：等同于 shared_ptr(p, d, a).swap(\*this)

~~~cpp
template<class Y> void reset(shared_ptr<Y> const & r, element_type * p); // never throws
~~~

+ **Effects**: Equivalent to shared_ptr(r, p).swap(\*this).  
效果：等同于 shared_ptr(r, p).swap(\*this)

<a name="indirection">

indirection - 间接访问
------
~~~cpp
T & operator*() const; // never throws
~~~

+ **Requirements**: T should not be an array type. The stored pointer must not be 0.  
要求：T不是数组类型。保存的指针必须不为0

+ **Returns**: a reference to the object pointed to by the stored pointer.  
返回值：保存指针所指对象的引用

+ **Throws**: nothing.  
异常不抛掷

~~~cpp
T * operator->() const; // never throws
~~~

+ **Requirements**: T should not be an array type. The stored pointer must not be 0.  
要求：T不是数组类型。保存的指针必须不为0

+ **Returns**: the stored pointer.  
返回值：保存的指针

+ **Throws**: nothing.  
异常不抛掷

~~~cpp
element_type & operator[](std::ptrdiff_t i) const; // never throws
~~~

+ **Requirements**: T should be an array type. The stored pointer must not be 0. i >= 0. If T is U[N], i < N.  
要求：T为数组类型。保存的指针必须不为0。i >=0。如果模板参数T为U[N]形式，满足 i < N

+ **Returns**: get()[i].  
返回值：get()[i]

+ **Throws**: nothing.  
异常不抛掷

<a name="get">

get - 获取保存指针
------
~~~cpp
element_type * get() const; // never throws
~~~

+ **Returns**: the stored pointer.  
返回值：保存的指针

+ **Throws**: nothing.  
异常不抛掷

<a name="unique">

unique - 是否唯一引用
------
~~~cpp
bool unique() const; // never throws
~~~

+ **Returns**: use_count() == 1.  
返回值：use_count() == 1

+ **Throws**: nothing.  
异常不抛掷

+ **Notes**: unique() may be faster than use_count(). If you are using unique() to implement copy on write, do not rely on a specific value when the stored pointer is zero.  
注意：unique()可能比use_count()快。如果使用unique()实现写时拷贝，shared_ptr存储一个0指针不要依赖具体值

<a name="use_count">

use_count - 引用计数
------
~~~cpp
long use_count() const; // never throws
~~~

+ **Returns**: the number of shared_ptr objects, *this included, that share ownership with *this, or 0 when *this is empty.  
返回值：指向同一个地址的shared_ptr对象个数，如果*this为空则返回0

+ **Throws**: nothing.  
异常不抛掷

+ **Notes**: use_count() is not necessarily efficient. Use only for debugging and testing purposes, not for production code.  
注意：use_count() 性能不佳。只建议在debug和测试时使用，不要在产品代码中使用。

<a name="conversions">

conversions - 隐式转换
------
~~~cpp
explicit operator bool() const; // never throws
~~~

+ **Returns**: get() != 0.  
返回值：get() != 0

+ **Throws**: nothing.  
异常不抛掷

+ **Notes**: This conversion operator allows shared_ptr objects to be used in boolean contexts, like if(p && p->valid()) {}.  
注意：隐式转换使shared_ptr可以用在布尔判断的上下文中，例如  if(p && p->valid()) {}

*[The conversion to bool is not merely syntactic sugar. It allows shared_ptrs to be declared in conditions when using [dynamic_pointer_cast](#dynamic_pointer_cast) or [weak_ptr::lock](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/weak_ptr.htm#lock).]*  
*[bool隐式转换不仅是一个语法糖，在使用 dynamic_pointer_cast 或 weak_ptr::lock 时，它允许声明 shared_ptr]*

<a name="swap">

swap - swap成员函数
------
~~~cpp
void swap(shared_ptr & b); // never throws
~~~

+ **Effects**: Exchanges the contents of the two smart pointers.  
效果：交换两个智能指针的内容

+ **Throws**: nothing.  
异常不抛掷

<a name="owner_before">

swap - "等价于"判断函数(我不知道这里的swap是什么意思...)
------
~~~cpp
template<class Y> bool owner_before(shared_ptr<Y> const & rhs) const; // never throws
template<class Y> bool owner_before(weak_ptr<Y> const & rhs) const; // never throws
~~~

+ **Effects**: See the description of [operator<](#operator_less).  
效果：见 operator< 的描述

~~~cpp
    //##Martin's Note##
    //boost::shared_ptr::operator< 使用 owner_before() 实现，行为与其完全一致：
    template<class T, class U> inline bool
    boost::shared_ptr::operator<(shared_ptr<T> const & a,
        shared_ptr<U> const & b) BOOST_NOEXCEPT {
         return a.owner_before( b );
    }
    //boost::shared_ptr::owner_before源码也很简单，直接使用引用计数比较
    template<class Y> bool
    boost::shared_ptr::owner_before( shared_ptr<Y> const & rhs ) const BOOST_NOEXCEPT {
         return pn < rhs.pn;
    }
    //其中boost::shared_ptr::pn是shared_count类型，也就是最核心的引用计数封装，其operator<源码直接返回比较两个计数器sp_counted_base指针的结果
    friend inline bool
    boost::detail::shared_count::operator<(shared_count const & a,
        shared_count const & b) {
         return std::less<sp_counted_base *>()( a.pi_, b.pi_ );
    }
    //可以看出，shared_ptr的小于操作符和owner_before都是基于“等价性”做出比较，比较的内容就是计数器指针，利用指针比较的特性还能判断两者是否出于同一继承体系，得到的也就是所谓的owner-based order


    //而C++11中的operator<实现有所差异，直接对指针的common_type(也就是校验是否出自同一继承体系)进行比较了
    template<typename _Tp1, typename _Tp2>
    inline bool
    std::shared_ptr::operator<(const shared_ptr<_Tp1>& __a,
        const shared_ptr<_Tp2>& __b) noexcept {
         typedef typename std::common_type<_Tp1*, _Tp2*>::type _CT;
         return std::less<_CT>()(__a.get(), __b.get());
    }
    //在丢失指针基本类型的情况下无法判断两个shared_ptr是否等价
~~~

+ **Throws**: nothing.  
异常不抛掷

<a name="Free_Functions">

Free Functions - 非成员函数
======

<a name="comparison">

comparison - 比较操作符
------

<a name="operator_equal">

~~~cpp
template<class T, class U>
bool operator==(shared_ptr<T> const & a, shared_ptr<U> const & b); // never throws
~~~

+ **Returns**: a.get() == b.get().  
返回值：a.get() == b.get()

+ **Throws**: nothing.  
异常不抛掷

<a name="operator_not_equal">

~~~cpp
template<class T, class U>
bool operator!=(shared_ptr<T> const & a, shared_ptr<U> const & b); // never throws
~~~

+ **Returns**: a.get() != b.get().  
返回值：a.get() != b.get()

+ **Throws**: nothing.  
异常不抛掷

<a name="operator_equal_null">

~~~cpp
template<class T>
bool operator==(shared_ptr<T> const & p, std::nullptr_t); // never throws
template<class T>
bool operator==(std::nullptr_t, shared_ptr<T> const & p); // never throws
~~~

+ **Returns**: p.get() == 0.  
返回值：p.get() == 0

+ **Throws**: nothing.  
异常不抛掷

<a name="operator_not_equal_null">

~~~cpp
template<class T>
bool operator!=(shared_ptr<T> const & p, std::nullptr_t); // never throws
template<class T>
bool operator!=(std::nullptr_t, shared_ptr<T> const & p); // never throws
~~~

+ **Returns**: p.get() != 0.  
返回值：p.get() != 0

+ **Throws**: nothing.  
异常不抛掷

<a name="operator_less">

~~~cpp
template<class T, class U>
bool operator<(shared_ptr<T> const & a, shared_ptr<U> const & b); // never throws
~~~

+ **Returns**: an unspecified value such that  
返回值：如下描述
    * operator< is a strict weak ordering as described in section 25.3 [lib.alg.sorting] of the C++ standard;  
      operator< 是严格弱序的，在C++标准25.3节[lib.alg.sorting]中有描述
~~~
    ##Martin's Note##
    strict weak order - 严格弱序
    这是C++中很重要的概念，有了它才能定义出等价(equivalence)的概念，参考《effective STL》第19-21条，详细参考 http://www.sgi.com/tech/stl/StrictWeakOrdering.html  (不过我没细看)
    等价性(equivalence)对于关联容器的Key排序非常重要
~~~
    * under the equivalence relation defined by operator<, !(a < b) && !(b < a), two shared_ptr instances are equivalent if and only if they share ownership or are both empty.  
      基于 operator< 定义的"等价于"关系 ( !(a<b) && !(b<a) )，两个 shared_ptr 对象只有在共享所有权或都为空时才等价(参考 [owner_before()](#owner_before) 实现)

+ **Throws**: nothing.  
异常不抛掷

+ **Notes**: Allows shared_ptr objects to be used as keys in associative containers.  
注意：shared_ptr对象可以用于关联容器作为key

*[Operator< has been preferred over a std::less specialization for consistency and legality reasons, as std::less is required to return the results of operator<, and many standard algorithms use operator< instead of std::less for comparisons when a predicate is not supplied. Composite objects, like std::pair, also implement their operator< in terms of their contained subobjects' operator<. The rest of the comparison operators are omitted by design.]*  

*[operator< 在专业性上优先于 std::less 函数，因为前者在一致性和合法性上优于后者。std::less 要求返回operator< 的结果，而许多标准算法在不支持谓词时使用operator< 替代 std::less。像 std::pair 这样的混合对象同样依据其子对象的operator< 实现自身的operator<. 余下的比较运算符在设计中被忽略了]*

<a name="free_swap">

swap
------
~~~cpp
template<class T>
void swap(shared_ptr<T> & a, shared_ptr<T> & b); // never throws
~~~

+ **Effects**: Equivalent to a.swap(b).  
效果：同 a.swap(b)

+ **Throws**: nothing.  
异常不抛掷

+ **Notes**: Matches the interface of std::swap. Provided as an aid to generic programming.  
注意：匹配 std::swap。作为泛型编程的补充工具

*[swap is defined in the same namespace as shared_ptr as this is currently the only legal way to supply a swap function that has a chance to be used by the standard library.]*  
*[swap 与 shared_ptr 定义在同一个命名空间的原因是，这是目前在标准库中支持使用 swap 的唯一合法方式]*

<a name="get_pointer">

get_pointer
------
~~~cpp
template<class T>
typename shared_ptr<T>::element_type * get_pointer(shared_ptr<T> const & p); // never throws
~~~

+ **Returns**: p.get().  
返回值：p.get()

+ **Throws**: nothing.  
异常不抛掷

+ **Notes**: Provided as an aid to generic programming. Used by [mem_fn](http://www.boost.org/doc/libs/1_63_0/libs/bind/mem_fn.html).  
注意：作为泛型编程的补充工具，由 mem_fn 使用

<a name="static_pointer_cast">

static_pointer_cast
------
~~~cpp
template<class T, class U>
shared_ptr<T> static_pointer_cast(shared_ptr<U> const & r); // never throws
~~~

+ **Requires**: The expression static_cast<T\*>( (U\*)0 ) must be well-formed.  
要求：static_cast<T\*>( (U\*)0 )必须被正确定义

+ **Returns**: shared_ptr<T>( r, static_cast<typename shared_ptr<T>::element_type*>(r.get()) ).  
返回值：shared_ptr<T>( r, static_cast<typename shared_ptr<T>::element_type*>(r.get()) )

~~~
    ##Martin's Note##
    这里使用的是 aliasing constructor (别名构造)，与r共享引用计数
~~~

+ **Throws**: nothing.  
异常不抛掷

+ **Notes**: the seemingly equivalent expression shared_ptr\<T\>(static_cast<T\*>(r.get())) will eventually result in undefined behavior, attempting to delete the same object twice.  
注意：shared_ptr\<T\>(static_cast<T\*>(r.get())) 看起来像等价的表达式，但实际上会导致未定义行为，同一对象将被 delete 两次

~~~
    ##Martin's Note##
    注意这里使用的是指针构造，新shared_ptr的引用计数与r没有任何关系！
~~~

<a name="const_pointer_cast">

const_pointer_cast
------
~~~cpp
template<class T, class U>
shared_ptr<T> const_pointer_cast(shared_ptr<U> const & r); // never throws
~~~

+ **Requires**: The expression const_cast<T*>( (U*)0 ) must be well-formed.  
要求：const_cast<T*>( (U*)0 )必须被正确定义

+ **Returns**: shared_ptr\<T\>( r, const_cast<typename shared_ptr\<T\>::element_type\*>(r.get()) ).  
返回值：shared_ptr\<T\>( r, const_cast<typename shared_ptr\<T\>::element_type\*>(r.get()) )

+ **Throws**: nothing.  
异常不抛掷

<a name="dynamic_pointer_cast">

dynamic_pointer_cast
------
~~~cpp
template<class T, class U>
shared_ptr<T> dynamic_pointer_cast(shared_ptr<U> const & r);
~~~

+ **Requires**: The expression dynamic_cast<T\*>( (U\*)0 ) must be well-formed.  
要求：dynamic_cast<T\*>( (U\*)0 )必须被正确定义

+ **Returns**:  
返回值

    * When dynamic_cast<typename shared_ptr\<T\>::element_type\*>(r.get()) returns a nonzero value p, shared_ptr\<T\>(r, p);  
      如果 dynamic_cast<typename shared_ptr\<T\>::element_type\*>(r.get()) 返回非零值 p,则返回 shared_ptr\<T\>(r, p);
    * Otherwise, shared_ptr\<T\>().  
      否则返回 shared_ptr\<T\>()

+ **Throws**: nothing.  
异常不抛掷

<a name="reinterpret_pointer_cast">

reinterpret_pointer_cast
------
~~~cpp
template<class T, class U>
shared_ptr<T> reinterpret_pointer_cast(shared_ptr<U> const & r); // never throws
~~~

+ **Requires**: The expression reinterpret_cast<T\*>( (U\*)0 ) must be well-formed.  
要求：reinterpret_cast<T\*>( (U\*)0 )必须被正确定义

+ **Returns**: shared_ptr\<T\>( r, reinterpret_cast<typename shared_ptr\<T\>::element_type\*>(r.get()) ).  
返回值：shared_ptr\<T\>( r, reinterpret_cast<typename shared_ptr\<T\>::element_type\*>(r.get()) )

+ **Throws**: nothing.  
异常不抛掷

<a name="stream_operator">

operator<<
------
~~~cpp
template<class E, class T, class Y>
std::basic_ostream<E, T> & operator<< (std::basic_ostream<E, T> & os, shared_ptr<Y> const & p);
~~~

+ **Effects**: os << p.get();.  
效果： os << p.get()

+ **Returns**: os.  
返回值：os

<a name="get_deleter">

get_deleter
------
~~~cpp
template<class D, class T>
D * get_deleter(shared_ptr<T> const & p);
~~~

+ **Returns**: If \*this owns a deleter d of type (cv-unqualified) D, returns &d; otherwise returns 0.  
返回值：如果 \*this 包含一个类型 D(cv-unqualified 非const/volatile限定) 的删除器 d，返回 &d；否则返回0

+ **Throws**: nothing.  
异常不抛掷

<a name="Example">

Example - 示例
======
See [shared_ptr_example.cpp](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/example/shared_ptr_example.cpp) for a complete example program. The program builds a std::vector and std::set of shared_ptr objects.  
见 shared_ptr_example.cpp 中完整代码，该程序创建了容纳 shared_ptr 类型的 std::vector 和 std::set

Note that after the containers have been populated, some of the shared_ptr objects will have a use count of 1 rather than a use count of 2, since the set is a std::set rather than a std::multiset, and thus does not contain duplicate entries. Furthermore, the use count may be even higher at various times while push_back and insertcontainer operations are performed. More complicated yet, the container operations may throw exceptions under a variety of circumstances. Getting the memory management and exception handling in this example right without a smart pointer would be a nightmare.  
注意在容器填充后，部分 shared_ptr 对象的引用计数不是2，而是1，因为 set 是 std::set 而非 std::multiset，不允许包含重复输入。此外，引用计数在多次 push_back 和 insert 容器操作后可能变得更高。更复杂的是，容器操作可能在多重不同场景下抛掷异常。想要不使用智能指针而在这个例子中正确管理内存和处理异常可能是一场噩梦。

<a name="Handle_Body__Idiom">

Handle/Body Idiom
======
One common usage of shared_ptr is to implement a handle/body (also called pimpl) idiom which avoids exposing the body (implementation) in the header file.  
shared_ptr 的一个众所周知的用法是实现 handle/body (也叫 pimpl ) 手法，该手法用来避免将实现暴露在头文件中

The [shared_ptr_example2_test.cpp](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/example/shared_ptr_example2_test.cpp) sample program includes a header file, [shared_ptr_example2.hpp](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/example/shared_ptr_example2.hpp) , which uses a shared_ptr to an incomplete type to hide the implementation. The instantiation of member functions which require a complete type occurs in the [shared_ptr_example2.cpp](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/example/shared_ptr_example2.cpp)  implementation file. Note that there is no need for an explicit destructor. Unlike ~scoped_ptr, ~shared_ptr does not require that T be a complete type.  
shared_ptr_example2_test.cpp 中包含了头文件 shared_ptr_example2.hpp，头文件中使用 shared_ptr 包含一个非完整类型来隐藏其实现。成员函数的实例化需要的完整类型都包含在 shared_ptr_example2.cpp 中。注意本例不需要显式析构函数。不同于 ~scoped_ptr，~shared_ptr不要求T是完整类型。

<a name="Thread_Safety">

Thread Safety - 线程安全性
======
shared_ptr objects offer the same level of thread safety as built-in types. A shared_ptr instance can be "read" (accessed using only const operations) simultaneously by multiple threads. Different shared_ptr instances can be "written to" (accessed using mutable operations such as operator= or reset) simultaneously by multiple threads (even when these instances are copies, and share the same reference count underneath.)  
shared_ptr 对象提供与内建类型相同等级的线程安全性。shared_ptr 实例可以被多个线程同时"读"(仅使用const修饰的方法访问)。不同的 shared_ptr 实例可以被多个线程同时"写"(使用改变对象状态的方法如 operator= 或 reset 方法)，即使这些实例是拷贝而来并共享同一个引用的所有权。

Any other simultaneous accesses result in undefined behavior.  
除上面以外的任何并发访问都会导致未定义行为

**Examples**:
示例：
~~~cpp
shared_ptr<int> p(new int(42));
//--- Example 1 ---
// thread A
shared_ptr<int> p2(p); // reads p
// thread B
shared_ptr<int> p3(p); // OK, multiple reads are safe

//--- Example 2 ---
// thread A
p.reset(new int(1912)); // writes p
// thread B
p2.reset(); // OK, writes p2

//--- Example 3 ---
// thread A
p = p3; // reads p3, writes p
// thread B
p3.reset(); // writes p3; undefined, simultaneous read/write

//--- Example 4 ---
// thread A
p3 = p2; // reads p2, writes p3
// thread B
// p2 goes out of scope: undefined, the destructor is considered a "write access"

//--- Example 5 ---
// thread A
p3.reset(new int(1));
// thread B
p3.reset(new int(2));  // undefined, multiple writes
~~~
  

Starting with Boost release 1.33.0, shared_ptr uses a lock-free implementation on most common platforms.  
从Boost1.33.0版本开始，shared_ptr在多数平台上都使用无锁结构(lock-free)实现(引用计数)

If your program is single-threaded and does not link to any libraries that might have used shared_ptr in its default configuration, you can #define the macro BOOST_SP_DISABLE_THREADS on a project-wide basis to switch to ordinary non-atomic reference count updates.  
如果你的程序是单线程并且没有链接其他可能使用shared_ptr默认配置的库，你可以在项目级别使用 #define BOOST_SP_DISABLE_THREADS 宏来切换成非原子引用计数

(Defining BOOST_SP_DISABLE_THREADS in some, but not all, translation units is technically a violation of the One Definition Rule and undefined behavior. Nevertheless, the implementation attempts to do its best to accommodate the request to use non-atomic updates in those translation units. No guarantees, though.)  
(定义 BOOST_SP_DISABLE_THREADS 宏在某些(并非全部)编译单元中可能违反了ODR原则。然而，boost的实现尽了最大努力来满足那些编译单元中的非原子更新需求，虽然并没有保证)

~~~
    ##Martin's Note##
    translation units - 编译单元(TU)
    One Definition Rule - 一次定义原则(还是叫ODR比较好...)
    编译单元可以看做代码文件提交给编译器预处理后的结果，没错，重点是#include和各种条件宏展开。
    ODR规定，对non-inline function在所有文件中保证只存在一处定义；对于类和内联函数，在每个编译单元中最多被定义一次，而且要保证不同编译单元之间定义的一致性。
~~~

You can define the macro BOOST_SP_USE_PTHREADS to turn off the lock-free platform-specific implementation and fall back to the generic pthread_mutex_t-based code.  
你可以通过定义 BOOST_SP_USE_PTHREAD 宏来关闭基于无锁平台的实现，并退回到通用的基于 pthread_mutex_t 的代码

<a name="FAQ">

Frequently Asked Questions
======
**Q**.There are several variations of shared pointers, with different tradeoffs; why does the smart pointer library supply only a single implementation? It would be useful to be able to experiment with each type so as to find the most suitable for the job at hand?  
**Q**.权衡不同的情况，有不同的共享指针实现；为什么 smart pointer 库只支持一种实现方案？为了找到最适合当前工作的类型，尝试每种类型的实现也许会很有用

**A**.An important goal of shared_ptr is to provide a standard shared-ownership pointer. Having a single pointer type is important for stable library interfaces, since different shared pointers typically cannot interoperate, i.e. a reference counted pointer (used by library A) cannot share ownership with a linked pointer (used by library B.)  
**A**.shared_ptr 一个重要设计目标就是提供标准共享所有权的指针。单一智能指针类型对于稳定的库接口是非常重要的，因为不同的共享指针类型通常不支持互操作，例如一个引用计数指针(由库A使用)无法与一个链接指针(由库B使用)共享所有权


**Q**.Why doesn't shared_ptr have template parameters supplying traits or policies to allow extensive user customization?  
**Q**.为什么shared_ptr 没有提供模板参数来支持 traits 或 policies 这样的特性而获得更多的用户扩展性？

**A**.Parameterization discourages users. The shared_ptr template is carefully crafted to meet common needs without extensive parameterization. Some day a highly configurable smart pointer may be invented that is also very easy to use and very hard to misuse. Until then, shared_ptr is the smart pointer of choice for a wide range of applications. (Those interested in policy based smart pointers should read [Modern C++ Design](http://www.awprofessional.com/bookstore/product.asp?isbn=0201704315&rl=1) by Andrei Alexandrescu.)  
**A**.参数化阻碍用户。shared_ptr 模板是经过小心设计的，不需要扩展参数也能满足大多数需求。也许有一天某种高度可配置、易用并难以被误用的智能指针会出现，但 shared_ptr 依然会是一大类应用开发时的智能指针选择.(对智能指针策略感兴趣的人可以阅读 Andrei Alexandrescu 的《Modern C++ Design》)

**Q**.I am not convinced. Default parameters can be used where appropriate to hide the complexity. Again, why not policies?  
**Q**.上面的回答没有说服我。在合适的地方使用默认参数可以隐藏复杂性。再次重复一个问题，为什么不使用 policies (基于原则设计)

**A**.Template parameters affect the type. See the answer to the first question above.  
**A**.模板参数影响类型。见第一个问题的回答

~~~
    ##Martin's Note##
    上面3个问题一定是Andrei Alxandrescu和Loki的支持者问的;)
    traits - 特性萃取技术(没有找到更合适的翻译)
    policies - 基于策略的设计
    两者都是用于泛型编程的重要class设计技术，Loki中的SmartPtr智能指针使用了policies的设计，让使用者可以根据自己的需求组合不同策略来获得满足自己要求的智能指针，其中一个策略组合已经覆盖了shared_ptr支持的场景。(此处涉及C++11标准之争，最后的结果大家都知道是boost的shared_ptr最后胜出)
    另外“单一智能指针类型对于稳定的库接口是非常重要的”这句话也可以理解成“如果库在接口中使用shared_ptr，那么你也必须用”，在shared_ptr进入C++标准前这招致不少人的反感(例如 http://blog.csdn.net/xushiweizh/article/details/4295948 )，另外引入boost这个巨大的依赖也会导致部分人望而却步。
~~~

**Q**.Why doesn't shared_ptr use a linked list implementation?  
**Q**.为什么 shared_ptr 没有使用链表实现

**A**.A linked list implementation does not offer enough advantages to offset the added cost of an extra pointer. See [timings](http://www.boost.org/doc/libs/1_63_0/libs/smart_ptr/smarttests.htm) page. In addition, it is expensive to make a linked list implementation thread safe.  
**A**.链表实现没有提供足够的好处来抵消增加额外指针带来的成本。见 timings 页面，另外，使链表实现线程安全代价非常昂贵

**Q**.Why doesn't shared_ptr (or any of the other Boost smart pointers) supply an automatic conversion to T\*?  
**Q**.为什么 shared_ptr(或其他任何 Boost 智能指针)不支持 T\* 的隐式转换？

**A**.Automatic conversion is believed to be too error prone.  
**A**.隐式转换极易引起错误

**Q**.Why does shared_ptr supply use_count()?  
**Q**.为什么 shared_ptr 支持 use_count() 函数？

**A**.As an aid to writing test cases and debugging displays. One of the progenitors had use_count(), and it was useful in tracking down bugs in a complex project that turned out to have cyclic-dependencies.  
**A**.作为编写测试用例和调试显示的辅助工具。use_count()函数是早期版本引入的，它在复杂项目中追踪循环引用导致的bug非常有用(所以保留下来了)

**Q**.Why doesn't shared_ptr specify complexity requirements?  
**Q**.为什么 shared_ptr 不指定复杂性要求？

**A**.Because complexity requirements limit implementors and complicate the specification without apparent benefit to shared_ptr users. For example, error-checking implementations might become non-conforming if they had to meet stringent complexity requirements.  
**A**.因为复杂性的要求限制了实现者同时在使规范变得复杂的情况下并没有为用户带来明显好处 。例如，严格的复杂性要求可能使错误检查的各种实现不相容。

**Q**.Why doesn't shared_ptr provide a release() function?  
**Q**.为什么 shared_ptr 没有提供 release() 函数？

**A**.shared_ptr cannot give away ownership unless it's unique() because the other copy will still destroy the object.  
**A**.shared_ptr 不应该放弃指针所有权除非 unique() 返回 True，因为其他shared_ptr拷贝仍然会析构该对象

Consider:
~~~cpp
shared_ptr<int> a(new int);
shared_ptr<int> b(a); // a.use_count() == b.use_count() == 2

int * p = a.release();

// Who owns p now? b will still call delete on it in its destructor.
// 现在谁持有p？ b析构的时候仍然会删除p指向的对象
~~~

Furthermore, the pointer returned by release() would be difficult to deallocate reliably, as the source shared_ptr could have been created with a custom deleter.  
此外 release() 返回的指针很难可靠的释放，因为创建 shared_ptr 的时候用户可能定制了删除器

**Q**.Why is operator->() const, but its return value is a non-const pointer to the element type?  
**Q**.为什么 operator->() 由 const 修饰，但是返回值却是非const修饰的指针

**A**.Shallow copy pointers, including raw pointers, typically don't propagate constness. It makes little sense for them to do so, as you can always obtain a non-const pointer from a const one and then proceed to modify the object through it. shared_ptr is "as close to raw pointers as possible but no closer".  
**A**.包括原生指针在内的浅拷贝指针都不会传播 const 属性。返回const指针几乎没有意义，因为你总是可以在const指针指向的对象中包含非const指针来修改后者指向的对象。shared_ptr是一种“尽可能接近原始指针但又不那么接近”的指针

