## Unique_ptr

EASTL中unique_ptr定义如下，它只有一个数据成员:

```c++
template <typename T, typename Deleter = eastl::default_delete<T> >
class unique_ptr{
public:
    	typedef Deleter                                                                  deleter_type;
		typedef T                                                                        element_type;
		typedef unique_ptr<element_type, deleter_type>                                   this_type;
		typedef typename Internal::unique_pointer_type<element_type, deleter_type>::type pointer;
 protected:
    eastl::compressed_pair<pointer, deleter_type> mPair;
}
```

它的数据成员是mPair，其是一个compressed_pair类型，即大小为pointer的大小，pointer在没有指定deleter的情况下为T*类型，找到指针的类定义如下:

```c++
template <typename T, typename Deleter>
class unique_pointer_type
{
    template <typename U>
    static typename U::pointer test(typename U::pointer*);

    template <typename U>
    static T* test(...);

public:
    typedef decltype(test<typename eastl::remove_reference<Deleter>::type>(0)) type;
};
```

大部分情况下使用unique_ptr配合new，其构造函数如下所示:

```c++
//unique_ptr<int> ptr(new int(3));
explicit unique_ptr(pointer pValue) 
			: mPair(pValue)
		{
			static_assert(!eastl::is_pointer<deleter_type>::value, "unique_ptr deleter default-constructed with null pointer. Use a different constructor or change your deleter to a class.");
		}

```



几个内部使用的函数

1、reset

```c++
		///    unique_ptr<int> ptr(new int(3));
		///    ptr.reset(new int(4));  // deletes int(3)
		///    ptr.reset(NULL);        // deletes int(4)
void reset(pointer pValue = pointer()) EA_NOEXCEPT
{
    if (pValue != mPair.first())//如果到reset的值不等于当前的值
    {
        //尝试交换两个值，然后返回原来的值为first，最后调用delete函数
        if (auto first = eastl::exchange(mPair.first(), pValue))
            get_deleter()(first);
    }
}

```

2、release

```c++
///    unique_ptr<int> ptr(new int(3));
///    int* pInt = ptr.release();
///    delete pInt;
pointer release() EA_NOEXCEPT
{
    pointer const pTemp = mPair.first();
    mPair.first() = pointer();
    return pTemp;
}

```

release交出所有权并不会主动删除。

3、 desctructor

在析构时会自动调用reset函数释放对象。

```c++
~unique_ptr()
		{
			reset();
		}
```

注意在unique_ptr中赋值运算符号只针对右边是右值的情况:

```c++
/// These functions are deleted in order to prevent copying, for safety.
		unique_ptr(const this_type&) = delete;
		unique_ptr& operator=(const this_type&) = delete;
		unique_ptr& operator=(pointer pValue) = delete;

this_type& operator=(this_type&& x)
{
    //会调用reset函数设置当前值为x的release()
    reset(x.release());
    mPair.second() = eastl::move(eastl::forward<deleter_type>(x.get_deleter()));
    return *this;
}
```



## shared_ptr

shared_ptr包含一个引用计数变量:

```c++
template <typename T>
class shared_ptr
{
public:
    typedef shared_ptr<T>                                    this_type;
    typedef T                                                element_type; 
    typedef typename shared_ptr_traits<T>::reference_type    reference_type;   // This defines what a reference to a T is. It's always simply T&, except for the case where T is void, whereby the reference is also just void.
    typedef EASTLAllocatorType                               default_allocator_type;
    typedef default_delete<T>                                default_deleter_type;
    typedef weak_ptr<T>                                      weak_type;

protected:
    element_type*  mpValue;
    ref_count_sp*  mpRefCount;        
}
```

其包含两个指针成员，其ref_count_sp类型如下所示：

```c++
struct ref_count_sp
{
    int32_t mRefCount;           
    int32_t mWeakRefCount;   
}
```

与unique_ptr不同的是shared_ptr允许赋值，最简单的实现如下所示:

```c++
shared_ptr& operator=(const shared_ptr& sharedPtr) EA_NOEXCEPT
{
    if(&sharedPtr != this)
        this_type(sharedPtr).swap(*this);

    return *this;
}
```

即首先创建一个新的shared_ptr，该创建的过程会增加传入的参数的引用计数，然后与当前的this指针指向的对象进行交换就行了。



