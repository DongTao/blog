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

+ **Postconditions**: use_count() == 1 && get() == p. If T is not an array type and p is unambiguously convertible to enable_shared_from_this<V>* for some V, p->shared_from_this() returns a copy of *this.  
后置条件：use_count() == 1 && get() == p。如果 T 是非数组类型且 p 可以明确转换为 enable_shared_from_this<V>* 类型，那么p->shared_from_this()返回 *this 的拷贝

+ **Throws**: std::bad_alloc, or an implementation-defined exception when a resource other than memory could not be obtained.  
异常：内存分配失败抛出 std::bad_alloc 异常，其他资源分配失败抛出实现定义的异常

+ **Exception safety**: If an exception is thrown, the constructor calls delete[] p, when T is an array type, or delete p, when T is not an array type.  
异常安全性：如果构造函数中抛出异常，对应的delete或delete[] p会被执行

+ **Notes**: p must be a pointer to an object that was allocated via a C++ new expression or be 0. The postcondition that use count is 1 holds even if p is 0; invoking delete on a pointer that has a value of 0 is harmless.  
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

+ **Postconditions**: use_count() == 1 && get() == p. If T is not an array type and p is unambiguously convertible to enable_shared_from_this<V>* for some V, p->shared_from_this() returns a copy of *this.  
后置条件：use_count() == 1 && get() == p。如果 T 是非数组类型且 p 可以明确转换为 enable_shared_from_this<V>* 类型，那么p->shared_from_this()返回 *this 的拷贝

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
