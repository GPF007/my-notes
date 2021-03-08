## String

在EASTL中，string继承自basic_string。basic_string有两种layout，在堆上或者在栈上。两种layout如下所示:

HeapLayout:

```c++
struct HeapLayout
{
    value_type* mpBegin;  // Begin of string.
    size_type mnSize;    
    size_type mnCapacity;
};
```

如果字符串在堆上的话，那么值需要一个指针，和两个表示长度的变量。

SSOLayout

```c++
template <typename CharT, size_t = sizeof(CharT)>
		struct SSOPadding
		{
			char padding[sizeof(CharT) - sizeof(char)];
		};


struct SSOLayout
{
    static constexpr size_type SSO_CAPACITY = (sizeof(HeapLayout) - sizeof(char)) / sizeof(value_type);
    struct SSOSize : SSOPadding<value_type>
    {
        char mnRemainingSize;
    };

    value_type mData[SSO_CAPACITY]; // Local buffer for string data.
    SSOSize mRemainingSizeField;
};
```

如果位于栈上，那么是一个固定大小的数组，还包括一个额外的信息用来表示是否位于栈上。

任何时刻都有 sizeof(HeapLayout) == sizeof(SSOLayout);

basic_string 的layout如下所示;

```c++
struct Layout
{
    union
    {
        HeapLayout heap;
        SSOLayout sso;
        //RawLayout raw;暂时不用管
    };
}
```

内部经常调用的函数AllocateSelf

```c++
template <typename T, typename Allocator>
void basic_string<T, Allocator>::AllocateSelf(size_type n)
{
    if(n > SSOLayout::SSO_CAPACITY)
    {
        pointer pBegin = DoAllocate(n + 1);
        internalLayout().SetHeapBeginPtr(pBegin);
        internalLayout().SetHeapCapacity(n);
        internalLayout().SetHeapSize(n);
    }
    else
        internalLayout().SetSSOSize(n);
}
```

根据传入的大小选择是否在堆中分配。

根据不同的char类型有不同的string:

```c++
/// string / wstring
typedef basic_string<char>    string;
typedef basic_string<wchar_t> wstring;

/// custom string8 / string16 / string32
typedef basic_string<char>     string8;
typedef basic_string<char16_t> string16;
typedef basic_string<char32_t> string32;

/// ISO mandated string types
typedef basic_string<char8_t>  u8string;    // Actually not a C++11 type, but added for consistency.
typedef basic_string<char16_t> u16string;
typedef basic_string<char32_t> u32string;
```

