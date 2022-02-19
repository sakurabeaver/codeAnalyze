[toc]

# 前言
在C++的使用过程中不可避免的使用到函数指针，在c++11之后,std::function可以替代函数指针的使用。下面我们将从源码中分析std::function的工作原理。本文所使用的g++版本为9.3.0.

```c++
$ g++ --version
g++ (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

```
# 简单的例子

```c++
#include <iostream>
#include <functional>

using std::cout;
using std::endl;

int print(int a)
{
  cout << "print: " << a << endl;
  return a;
}

int main()
{
  std::function<int(int)> f(print);
  int ret = f(10);
  cout << "ret: " << ret << endl;
}

```

编译运行后的输出为:
```c++
#include <iostream>
#include <functional>

using std::cout;
using std::endl;

int print(int a)
{
  cout << "print: " << a << endl;
  return a;
}

int main()
{
  std::function<int(int)> f(print);
  int ret = f(10);
  cout << "ret: " << ret << endl;
}
```

**编译运行后的输出为:**

```c++
print: 10
ret: 10
```

从上述的使用可以看出，std::function是c++11是可以用来替换函数指针的.根据此例子来从源码中看一下是怎么运行的.

`std::function`的代码是放在`std_function.h`中的。

```c++
template<typename _Res, typename... _ArgTypes>
    class function<_Res(_ArgTypes...)>
    : public _Maybe_unary_or_binary_function<_Res, _ArgTypes...>,
      private _Function_base
    {
	//...
	};
```
值得注意的是，上述function是片特化后的版本，其原始声明式为:
```c++
template <typename _Signature>// _Signature表明了函数的签名
class function;
```
另外值得注意的是, function实际特化的参数是一个参数: **clss function<_Res(_Argtypes...)>**，从命名中可以看出,*_Res*是返回类型，而*_Argtypes*是参数类型.
*std::function<int(float, double)>*定义式中,返回类型为*int*, 参数类型分别为: float和double。
通过使用*_Res(_Argtypes...)*的技巧提取函数的返回类型和参数类型.

继续看**_Maybe_unary_or_binary_function**的结构
```c++
template<typename _Res, typename... _ArgTypes>
    struct _Maybe_unary_or_binary_function { };
// 一元函数
  template<typename _Res, typename _T1>
    struct _Maybe_unary_or_binary_function<_Res, _T1>
    : std::unary_function<_T1, _Res> { };
// 二元函数
  template<typename _Res, typename _T1, typename _T2>
    struct _Maybe_unary_or_binary_function<_Res, _T1, _T2>
    : std::binary_function<_T1, _T2, _Res> { };
```
上面的定义式中，将函数类型分成了一元函数、二元函数和其他类型，让我们继续看基类的定义
```c++
  template<typename _Arg, typename _Result>
    struct unary_function
    {
      /// @c argument_type is the type of the argument
      typedef _Arg 	argument_type;   

      /// @c result_type is the return type
      typedef _Result 	result_type;  
    };

  /**
   *  This is one of the @link functors functor base classes@endlink.
   */
  template<typename _Arg1, typename _Arg2, typename _Result>
    struct binary_function
    {
      /// @c first_argument_type is the type of the first argument
      typedef _Arg1 	first_argument_type; 

      /// @c second_argument_type is the type of the second argument
      typedef _Arg2 	second_argument_type;

      /// @c result_type is the return type
      typedef _Result 	result_type;
    };
```
从上面的结构可以看出，其就是作了一些类型定义

function类的另一个父类*_Function_base*的定义先暂时放一放，让我们根据function的构造和调用顺序来看看其运行原理

## 数据构造

上述例子调用的构造函数为:
```c++
template<typename _Res, typename... _ArgTypes>
    template<typename _Functor, typename, typename>
      function<_Res(_ArgTypes...)>::
      function(_Functor __f)
      : _Function_base()
      {
	typedef _Function_handler<_Res(_ArgTypes...), _Functor> _My_handler;

	if (_My_handler::_M_not_empty_function(__f))
	  {
	    _My_handler::_M_init_functor(_M_functor, std::move(__f));
	    _M_invoker = &_My_handler::_M_invoke;
	    _M_manager = &_My_handler::_M_manager;
	  }
      }
```

*_Function_handler*的定义为:
```c++
template<typename _Signature, typename _Functor>
    class _Function_handler;
	
	// 有返回值
	template<typename _Res, typename _Functor, typename... _ArgTypes>
    class _Function_handler<_Res(_ArgTypes...), _Functor>
    : public _Function_base::_Base_manager<_Functor>
    {
      typedef _Function_base::_Base_manager<_Functor> _Base;

    public:
      static _Res
      _M_invoke(const _Any_data& __functor, _ArgTypes&&... __args)
      {
	return (*_Base::_M_get_pointer(__functor))(
	    std::forward<_ArgTypes>(__args)...);
      }
    };
	
	// 没有返回值
	template<typename _Functor, typename... _ArgTypes>
    class _Function_handler<void(_ArgTypes...), _Functor>
    : public _Function_base::_Base_manager<_Functor>
    {
      typedef _Function_base::_Base_manager<_Functor> _Base;

     public:
      static void
      _M_invoke(const _Any_data& __functor, _ArgTypes&&... __args)
      {
	(*_Base::_M_get_pointer(__functor))(
	    std::forward<_ArgTypes>(__args)...);
      }
    };
```

*function*类中提供了一个隐式类型转换函数提供给*_M_not_empty_function()*函数使用:
```c++
 explicit function::operator bool() const noexcept
      { return !_M_empty(); }
	  
bool _Function_base::_M_empty() const { return !_M_manager; }
```

有没有返回值的两个特化版本的差别不大，只是*invoke*函数有没有返回值的区别
*_Function_handler*继承于*_Function_base::_Base_manager*,而*_M_not_empty_function*和*_M_init_functor*就是*_Function_base::_Base_manager*类中的静态函数.
这两个函数的功能正如其名字一样，分别为判断是否为空和初始化.

*_M_init_functor*函数的第一个参数是function父类中的类型为*_M_init_functor*的成员变量，此函数的定义式为:
```c++
static void
	_M_init_functor(_Any_data& __functor, _Functor&& __f)
	{ _M_init_functor(__functor, std::move(__f), _Local_storage()); }
	
static void
	_M_init_functor(_Any_data& __functor, _Functor&& __f, true_type)
	{ ::new (__functor._M_access()) _Functor(std::move(__f)); }
	
	static void
	_M_init_functor(_Any_data& __functor, _Functor&& __f, false_type)
	{ __functor._M_access<_Functor*>() = new _Functor(std::move(__f)); }
```

*_Local_storage()*函数返回的参数被应用于萃取技术来选择调用_M_init_functor的哪一个重载函数,这个函数的实现不是我们关注的重点，我们以参数为*true_type*为例子分析。此函数使用placement new操作符分配空间。
*_Any_data*的定义式为:

```c++
  union _Nocopy_types
  {
    void*       _M_object;
    const void* _M_const_object;
    void (*_M_function_pointer)();
    void (_Undefined_class::*_M_member_pointer)();
  };

union _Any_data
  {
    void*       _M_access()       { return &_M_pod_data[0]; }
    const void* _M_access() const { return &_M_pod_data[0]; }

    template<typename _Tp>
      _Tp&
      _M_access()
      { return *static_cast<_Tp*>(_M_access()); }

    template<typename _Tp>
      const _Tp&
      _M_access() const
      { return *static_cast<const _Tp*>(_M_access()); }

_Nocopy_types _M_unused; // 指针大小
    char _M_pod_data[sizeof(_Nocopy_types)];
  };
```
即，将函数指针保存到了*_M_pod_data*成员变量中

最后，m_function的构造函数的最后将两个函数指针赋值给成员变量，其中*_M_manager*是*_Function_base*类中的成员变量，而*_M_invoker*是*function*类中的成员变量,其定义式为:
``c++
using _Invoker_type = _Res (*)(const _Any_data&, _ArgTypes&&...);
Invoker_type _M_invoker;
```

## 调用
函数的调用是通过function类中的*()*操作符进行的,此函数的定义式为:
```c++
template<typename _Res, typename... _ArgTypes>
    _Res
    function<_Res(_ArgTypes...)>::
    operator()(_ArgTypes... __args) const
    {
      if (_M_empty())
	__throw_bad_function_call();
      return _M_invoker(_M_functor, std::forward<_ArgTypes>(__args)...);
    }
```
*_M_empty()*函数判断是否*_M_manager*变量是否为空，在function构造函数中已经提到此变量已经初始化了，所以我们直接看*_M_invoker()*函数, 此函数调用的是*_Function_handler::_M_invoke()*函数。上文已经给出了*_M_invoke()*函数的定义式为:
```c++
template<typename _Res, typename _Functor, typename... _ArgTypes>
    class _Function_handler<_Res(_ArgTypes...), _Functor>
    : public _Function_base::_Base_manager<_Functor>
    {
      typedef _Function_base::_Base_manager<_Functor> _Base;

    public:
      static _Res
      _M_invoke(const _Any_data& __functor, _ArgTypes&&... __args)
      {
	return (*_Base::_M_get_pointer(__functor))(
	    std::forward<_ArgTypes>(__args)...);
      }
    };
	
	static _Functor*
	_Function_base::_M_get_pointer(const _Any_data& __source)
	{
	    return __source._M_access<_Functor*>(); // 把初始化存着的函数指针拿出来
	}
```
*_ArgTypes...*就是多个参数类型
这里就完成了函数的调用，并且对于有返回值的函数调用也有返回相应类型的数据

## 析构
*function*类没有动态分配资源，所以使用的是默认析构函数。但是从上面的分析可以看出，函数指针是动态分配的资源，所以为了保证资源不泄露，一定会在析构的时候释放调资源，这部分工作是在*function*的父类*_Function_base*中进行的，其析构函数如下:

```c++
_Function_base::~_Function_base()
    {
      if (_M_manager)
	_M_manager(_M_functor, _M_functor, __destroy_functor);
    }
	
bool
	_Base_manager::_M_manager(_Any_data& __dest, const _Any_data& __source,
		   _Manager_operation __op)
	{
	  switch (__op)
	    {
#if __cpp_rtti
	    case __get_type_info:
	      __dest._M_access<const type_info*>() = &typeid(_Functor);
	      break;
#endif
	    case __get_functor_ptr:
	      __dest._M_access<_Functor*>() = _M_get_pointer(__source);
	      break;

	    case __clone_functor:
	      _M_clone(__dest, __source, _Local_storage());
	      break;

	    case __destroy_functor:
	      _M_destroy(__dest, _Local_storage());
	      break;
	    }
	  return false;
	}
	
	
	void
	_Base_manager::_M_destroy(_Any_data& __victim, false_type)
	{
	delete __victim._M_access<_Functor*>();    // 动态分配的指针
	}
```

*_M_manager*成员变量是在*function*构造的时候指定的函数指针,最后调用的是delete函数释放的资源

# 总结
本文从一个小例子出发，分析了*function*类从构造到调用，最后到析构的全过程。*function*类中还有其他有用的函数，比如获取函数指针等。从这一过程的分析可以看出，*function*类的内部会保存动态分布的函数指针，这一过程对程序性能的影响在使用之前应该要考虑清除.


