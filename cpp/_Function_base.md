[toc]

**c++ 源码分析之_Function_base**

# 前言

**_Function_base**是所有多態函数对象的基类。**_Function_base**类的成员变量保存着数据以及操作数据的函数，字类只需要往**_Function_base**的对应变量中添加数据，然后按照一定的规则调用相应的接口，就可以使**_Function_base**按照预期工作。下面我们分析**_Function_base**类的源码。

# _Function_base源码
**平台**

本文使用的平台为**Ubuntu 20.04**,g++版本为**9.3.0**
```bash
g++ (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

抛开**_Function_base**类中嵌套的**_Base_manager**和**_Function_handler**类的代码后，**_Function_base**类的源码为:
```c++

enum _Manager_operation
{
    __get_type_info,
    __get_functor_ptr,
    __clone_functor,
    __destroy_functor
};


class _Function_base
  {
  public:
    _Function_base() : _M_manager(nullptr) { }

    ~_Function_base()
    {
      if (_M_manager)
	_M_manager(_M_functor, _M_functor, __destroy_functor);
    }

    bool _M_empty() const { return !_M_manager; }

    typedef bool (*_Manager_type)(_Any_data&, const _Any_data&,
				  _Manager_operation);

    _Any_data     _M_functor;
    _Manager_type _M_manager;
  };
```

从**_Function_base**的源码可以看出，其发挥作用主要是通过**_M_functor**和**_M_manager**发挥最用，正如其名字所示，**_M_functor**主要是保存函数指针，而**_M_manager**是一个函数指针，此函数完成一些列操作(**_Manager_operation**),操作的对象其实是**_M_functor**。推荐先看一下作者的前一篇文章[c++源码分析之function](https://github.com/sakurabeaver/codeAnalyze/blob/main/cpp/function.md)，大致了解**_Function_base**的使用 TODO 

从上一篇文章可以看到，**_M_functor**中的数据是动态分配的，所以在析构函数中要释放掉动态分配的内存。以上篇文章中的**function**的使用为例，**_M_manager**函数的实现为：
```c++
template<typename _Functor>
class _Base_manager{
protected:
static const bool __stored_locally =
	(__is_location_invariant<_Functor>::value
	 && sizeof(_Functor) <= _M_max_size
	 && __alignof__(_Functor) <= _M_max_align
	 && (_M_max_align % __alignof__(_Functor) == 0));

     static void
	_M_clone(_Any_data& __dest, const _Any_data& __source, true_type)
	{
	  ::new (__dest._M_access()) _Functor(__source._M_access<_Functor>());
	}

    static void
	_M_destroy(_Any_data& __victim, false_type)
	{
	  delete __victim._M_access<_Functor*>();
	}

static bool
	_M_manager(_Any_data& __dest, const _Any_data& __source,
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
	      _M_clone(__dest, __source, _Local_storage());     // 使用萃取调用不同的函数
	      break;

	    case __destroy_functor:
	      _M_destroy(__dest, _Local_storage());             // 同上
	      break;
	    }
	  return false;
	}
};
```

**_Any_data**类主要是保存函数指针数据，其源码为：
```c++
class _Undefined_class;

union _Nocopy_types
{
    void*       _M_object;
    const void* _M_const_object;
    void (*_M_function_pointer)();
    void (_Undefined_class::*_M_member_pointer)();
};

union  _Any_data
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

    _Nocopy_types _M_unused;
    char _M_pod_data[sizeof(_Nocopy_types)];
};
```

**_Nocopy_types**是一个union, 其保存了各种类型的指针，结合**_Any_data**的源码可以看出，**_Nocopy_types**主要是计算出各种类型的指针的最大字节数，而**_Any_data**类保存了实际的函数指针。


# 总结
本文介绍了**_Function_base**的源码，并顺带介绍了操作其成员变量的函数



