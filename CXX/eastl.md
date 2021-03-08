# EASTL 源码阅读

## 1、Allocator

在namespace eastl中有一个全局的allocator。class allocator定义如下：

```c++
class allocator
	{
	public:
		//allocator(pName = NULL)
		allocator(const char* pName = NULL);
		allocator(const allocator& x);
		allocator(const allocator& x, const char* pName);

		allocator& operator=(const allocator& x);

		void* allocate(size_t n, int flags = 0);
		void* allocate(size_t n, size_t alignment, size_t offset, int flags = 0);
		void  deallocate(void* p, size_t n);

		const char* get_name() const;
		void        set_name(const char* pName);

	protected:
		#if EASTL_NAME_ENABLED
			const char* mpName; // Debug name, used to track memory.
		#endif
	};
```

其核心的函数只有两个，及allocate和deallocate，实现如下所示：

```c++
inline void* allocator::allocate(size_t n, int flags){
    return ::new((char*)0, flags, 0, (char*)0,        0) char[n]
}
inline void* allocator::allocate(size_t n, int flags){
    return ::new(alignment, offset, (char*)0, flags, 0, (char*)0,0) char[n];
}
//这里size_t是一个空的参数
inline void allocator::deallocate(void*p, size_t){
    delete [](char*)p;
}
```

这里使用了自定义的重载后的new函数，如下所示：

```c++
void* operator new[](size_t size, size_t alignment, size_t alignmentOffset, const char* pName, int flags, unsigned debugFlags, const char* file, int line);
```

得到全局的allocator函数如下定义：

```c++
//该函数会返回一个系统默认的allocator
inline EASTLAllocatorType* get_default_allocator(const EASTLAllocatorType*)
	{
		return EASTLAllocatorDefault(); 
	}
```

最后导出的接口应该是一个分配内存的分配函数，定义如下：

```c++
template <typename Allocator>
	inline void* allocate_memory(Allocator& a, size_t n, size_t alignment, size_t alignmentOffset)
	{
		void *result;
		if (alignment <= EASTL_ALLOCATOR_MIN_ALIGNMENT)
		{
			result = a.allocate(n);
		}
		else
		{
			result = a.allocate(n, alignment, offset);
		}
		return result;
	}
```



## 2、迭代器

EASTL中有6种迭代器，定义如下：

```c++
struct input_iterator_tag { };
struct output_iterator_tag { };
struct forward_iterator_tag       : public input_iterator_tag { };
struct bidirectional_iterator_tag : public forward_iterator_tag { };
struct random_access_iterator_tag : public bidirectional_iterator_tag { };
struct contiguous_iterator_tag    : public random_access_iterator_tag { };
```

使用typetrait可以得到迭代器的具体类型，这里对指针和const指针做了偏特化处理，其迭代器类型都是random_access的。

```c++
// struct iterator
	template <typename Category, typename T, typename Distance = ptrdiff_t,
			  typename Pointer = T*, typename Reference = T&>
	struct iterator
	{
		typedef Category  iterator_category;
		typedef T         value_type;
		typedef Distance  difference_type;
		typedef Pointer   pointer;
		typedef Reference reference;
	};


	// struct iterator_traits
	template <typename Iterator>
	struct iterator_traits
	{
		typedef typename Iterator::iterator_category iterator_category;
		typedef typename Iterator::value_type        value_type;
		typedef typename Iterator::difference_type   difference_type;
		typedef typename Iterator::pointer           pointer;
		typedef typename Iterator::reference         reference;
	};

	template <typename T>
	struct iterator_traits<T*>
	{
		typedef random_access_iterator_tag iterator_category;     // To consider: Change this to contiguous_iterator_tag for the case that
		typedef T                                        value_type;            //              EASTL_ITC_NS is "eastl" instead of "std".
		typedef ptrdiff_t                                difference_type;
		typedef T*                                       pointer;
		typedef T&                                       reference;
	};

	template <typename T>
	struct iterator_traits<const T*>
	{
		typedef EASTL_ITC_NS::random_access_iterator_tag iterator_category;
		typedef T                                        value_type;
		typedef ptrdiff_t                                difference_type;
		typedef const T*                                 pointer;
		typedef const T&                                 reference;
	};
```



## 3、Memory库函数

在stl中常用的memory函数如下：

1. uninitialized_fill

   ```c++
   //传入两个迭代器，first和last，以及value，该函数会用type_trait得到value是否是triviall_copy_assignable的。
   template <typename ForwardIterator, typename T>
   	inline void uninitialized_fill(ForwardIterator first, ForwardIterator last, const T& value)
   	{
   		typedef typename eastl::iterator_traits<ForwardIterator>::value_type value_type;
   		Internal::uninitialized_fill_impl(first, last, value, eastl::is_trivially_copy_assignable<value_type>());
   	}
   ```

   根据value的trait类型分别调用不同的函数：

   ```c++
   template <typename ForwardIterator, typename T>
   inline void uninitialized_fill_impl(ForwardIterator first, ForwardIterator last, const T& value, true_type)
   {
       //fill是一个多次偏特化的函数，根据interator的类型来调用
       //signed char* => memset(first, (unsigned char)c, size_t(last - first))l
       //
   	eastl::fill(first, last, value);
   }
   ```

   对于需要构造的

   ```c++
   //这里略去了异常处理
   template <typename ForwardIterator, typename T>
   void uninitialized_fill_impl(ForwardIterator first, ForwardIterator last, const T& value, false_type)
   {
   	typedef typename eastl::iterator_traits<ForwardIterator>::value_type value_type;
   	ForwardIterator currentDest(first);	
   	for(; currentDest != last; ++currentDest)
   			::new((void*)eastl::addressof(*currentDest)) value_type(value);
   			
   }
   ```

   uninitialized_fiil_n和fill类似,只不过把last转换为了first + n。

2. uninitialized_fill_ptr

   ```c++
   template <typename T>
   	inline void uninitialized_fill_ptr(T* first, T* last, const T& value)
   	{
   		typedef typename eastl::iterator_traits<eastl::generic_iterator<T*, void> >::value_type value_type;
   		Internal::uninitialized_fill_impl(eastl::generic_iterator<T*, void>(first),
   		                                  eastl::generic_iterator<T*, void>(last), value,
   		                                  eastl::is_trivially_copy_assignable<value_type>());
   	}
   ```

   其中generic_iterator可以看做一个wrapper，将指针化为通用的迭代器其定义了指针作为迭代器应该有的操作。

   ```c++
   template <typename Iterator, typename Container = void>
   class generic_iterator
   {
   protected:
   	Iterator mIterator;
   public:
       ...
   }
   ```

3. uninitialized_move

   该函数也有两个实现，针对value是否是trivially_copy_assignable的：

   ```c++
   template <typename InputIterator, typename ForwardIterator>
   inline ForwardIterator uninitialized_move_impl(InputIterator first, InputIterator last, ForwardIterator dest, true_type)
   {//对于POD类型直接copy即可
   	return eastl::copy(first, last, dest);
   }
   
   //对于object类型要尝试使用其拷贝构造函数
   template <typename InputIterator, typename ForwardIterator>
   inline ForwardIterator uninitialized_move_impl(InputIterator first, InputIterator last, ForwardIterator dest, false_type)
   {
       typedef typename eastl::iterator_traits<ForwardIterator>::value_type value_type;
       ForwardIterator currentDest(dest);
   
       for(; first != last; ++first, ++currentDest)
           ::new((void*)eastl::addressof(*currentDest)) value_type(eastl::move(*first)); 
   
       return currentDest;
   }
   ```

   







## 4、type_traits

EASTL中使用interal_constant表示type_traits的基础类:

```c++
//该类是空的，
template <typename T, T v>
struct integral_constant
{
	static  T value = v;
	typedef T value_type;
	typedef integral_constant<T, v> type;
    
    //重载了value_type类型转换和 （）运算符。
	operator value_type() const EA_NOEXCEPT { return value; }
	value_type operator()() const EA_NOEXCEPT { return value; }
};
```

定义了两个type，true_type和false_type来表示类型。

```c++
typedef integral_constant<bool, true>  true_type;
typedef integral_constant<bool, false> false_type;
```

具体调用方法如上面所用到的is_trivially_copy_assignable源码如下:

```c++
template <typename T>
struct is_trivially_copy_assignable
	: public integral_constant<bool,
			eastl::is_scalar<T>::value || eastl::is_pod<T>::value || eastl::is_trivially_assignable<typename eastl::add_lvalue_reference<T>::type, typename eastl::add_lvalue_reference<typename eastl::add_const<T>::type>::type>::value
		> {};
```

可以看到根据传入的类型T可以萃取出T的具体属性，同时得到这个结构体继承于true_type 或者 false_type。如果继承自true_type，那么具体的调用参数：

```c++
eastl::is_trivially_copy_assignable<value_type>()
```

会使用该struct的构造函数构造一个对象，该object的class是is_trivially_copy_assignable，继承自true_type，那么就会调用符合条件的函数。

```c++
template <typename ForwardIterator, typename T>
inline void uninitialized_fill_impl(ForwardIterator first, ForwardIterator last, const T& value, true_type)
```







## 5、 Vector

vector 作为最基本的序列容器，其结构较为简单。vector继承自VectorBase，其主要包括三个成员：

```c++
template <typename T, typename Allocator>
class VectorBase{
protected:
    T* mpBegin;
    T* mpEnd;
    eastl::compressed_pair<T*, allocator_type>  mCapacityAllocator;
};
```

其中第三个成员是一个pair，但只占据第一个T*的空间，后面是类型。因此vector有三个指针的大小。

以最基础的push_back为例，来展示动态扩容的特性，其代码如下所示:

```c++
template <typename T, typename Allocator>
inline void vector<T, Allocator>::push_back(const value_type& value)
{
    //如过end小于 cap，那么直接在end处new 一个变量为value的值即可
	if(mpEnd < internalCapacityPtr())
		::new((void*)mpEnd++) value_type(value);
	else//否则需要扩容
		DoInsertValueEnd(value);
}

//DoInsertValueEnd()这里忽略里异常处理
template<typename T, typename Allocator>
template<typename... Args>
void vector<T, Allocator>::DoInsertValueEnd(Args&&... args)
{
    const size_type nPrevSize = size_type(mpEnd - mpBegin);
    const size_type nNewSize  = GetNewCapacity(nPrevSize); // a>0?2*a:1
    //分配nsize个大小的空间
    pointer const   pNewData  = DoAllocate(nNewSize);
    
    //将原来空间大小的数据移动到新的空间，注意这里是移动每个值
    pointer pNewEnd = eastl::uninitialized_move_ptr_if_noexcept(mpBegin, mpEnd, pNewData);
        ::new((void*)pNewEnd) value_type(eastl::forward<Args>(args)...);
    pNewEnd++;
    //释放原来的空间
    eastl::destruct(mpBegin, mpEnd);
    DoFree(mpBegin, (size_type)(internalCapacityPtr() - mpBegin));

    mpBegin    = pNewData;
    mpEnd      = pNewEnd;
    internalCapacityPtr() = pNewData + nNewSize;
}
```

