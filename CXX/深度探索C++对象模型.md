## 1 C++对象模型

C++每一个Class有一个虚函数表，表中存放虚函数指针，一般虚函数表的第一个slot存放RTTI信息，即typeinfo信息。

```c++
class Point{
public:
	Point() = default;
	virtual ~Point() = default;
	int foo();
private:
	double d_;
};
```

每个object在64位机器上占据16bytes，包括8bytes的float和8bytes的vptr,即虚函数指针。虚函数表的结构如下：

| SLOT序号 | 内容                   |
| -------- | ---------------------- |
| 1        | typeinfo(这是一个指针) |
| 2        | Point::~Point()函数    |

注意只有声明为virtual的函数才会在虚函数表中有一个指针，指向对应的函数。



## 2、构造函数语意学

1. Default Constructor

   如果一个类没有声明构造函数，那么编译器会自动合成一个默认的构造函数，该构造函数是trivial的，即没有什么用的。它只在需要时被生成调用，且不会初始化成员变量的值，如int,char等。这个过程可以递归。

   ```c++
   class Foo{}
   class Bar{public: Foo foo; char * str;}
   //开始调用
   Bar bar;
   ```

   上述代码生成了一个对象bar，且Bar没有构造函数，于是系统合成的构造函数如下:

   ```c++
   //合成的构造函数都是inline的，且只在当前文件可见
   inline Bar::Bar(){
   	foo.Foo::Foo();
   }
   ```

   其中Foo的构造函数也是合成的。如果Bar有多个类成员变量，那么会按照成员的声明顺序合成构造函数。生成的代码在所有显示赋值代码之前。如果class中有virtual函数，那么合成的constructor也会初始化对象的虚函数指针。

2. Copy Constructor

   如果一个类没有声明拷贝构造函数，那么编译器会合成一个，合成的函数会逐个拷贝非类成员member变量,如int，char, int*等。如果是类成员变量，那么会递归拷贝。

   只有nontrivial的拷贝构造函数才会合成于程序中。

   ```c++
   class String{
   public:
       String(char *s):str(s){}
   	int cnt;
   	char* str;
   };
   class Word{
   public:
       int cnt;
       String str;
   }
   ```

   上述String类具有Bitwise Copy Semantics,即按照位拷贝，那么在进行对象拷贝时，不会合成拷贝构造函数，而是直接进行成员变量的拷贝。而Word类则需要合成一个拷贝构造函数：

   ```c++
   inline Word::Word(const Word& wd){
   	str.String::String(wd.str);
   	cnt = wd.cnt;
   }
   ```

   有四种情况class不会展现出Bitwise Copy Semantics，即需要合成一个nontrival的拷贝构造函数。

   + class有一个member object，而后者声明有一个copy constructor。不论是被显式声明还是被合成的。
   + class继承一个base class，后者存在一个copy constructor。
   + class声明了虚函数
   + class派生自一个继承串链，其中有一个或多个virtual base classes。



3. 返回值优化

   在程序员的角度，如果一个函数返回一个对象，那么可以最后在调用构造函数，可以减少一次拷贝。

   ```c++
   X bar(const T& y, const T& z){
   	return X(y,z);
   }
   ```

   在编译器层面，可以有NRV(named return value)优化，要执行NAV优化必须显式地声明一个拷贝构造函数。

   ```c++
   X bar(){
   	X xx;
   	//处理 xx
   	return xx;
   }
   //优化后如下：
   void bar(X &__result){
       __result.X::X();
       //do something
       return;
   }
   ```

   对于一些有bitwise copy'的class，最好不要显式声明拷贝构造函数，因为编译器可以合成一个trivial的。如：

   ```c++
   class Point3d:{
   public:
   	Point3d(float x, float y, float z);
   private:
   	float _x, _y, _z;
   
   };
   ```

   如果该类需要大量的返回值操作或者参数传入操作，那么可以选项声明一个拷贝构造函数，以进行NRV优化。



4. 成员初始化列表(Member initialization List)

   成员初始化列表会先于构造函数内的显式代码执行，且初始化顺序是按照member变量的声明顺序，与列表顺序无关。

   ```c++
   class X{
   public:
   	X(int ival, jval):j(jval),i(ival){}
   	int i;
   	int j;
   }
   ```

   上述构造函数会先初始化i然后是j。如果继承自基类，则会先调用基类的构造函数。即基类的member initialization list是最先执行的。
   
5. 如果class没有定义descructor，那么只有在class内含的member object或者自己的base class中拥有destructor的情况下，编译器才会自动合成一个来，否则析构函数不需要被合成，也就不会被调用。

   ```c++
   class Point{
   public:
       virtual ~Point();
       float x()	{return x_;}
       virtual float y()	const {return 0;}
       virtual float z()	const {return 0;}
       
       float x_;
   };
   ```

   上述类即使有虚函数，但是也不会合成析构函数。



## 3、函数调用

1、假设有一个class Point定义如下

```c++
class Point{
public:
    virtual ~Point();
    float x()	{return x_;}
    virtual float y()	const {return 0;}
    virtual float z()	const {return 0;}
    
    float x_;
};
```

那么class Point的虚函数表中有4个slot，第一个是typeid信息，其余3个为虚函数。各函数调用会被编译器展开为：

| 函数原型    | 调用展开               |
| ----------- | ---------------------- |
| p->x();     | Point::x(p);           |
| p->~Point() | (*Point::vptr[1]) (p); |
| p->y()      | (*Point::vptr[2]) (p); |

其中p是一个Point对象的指针，而vptr表示虚函数表，每个slot（除去第一个）表示一个函数指针。

对于Point的子类，其虚函数表可以被覆盖，如果没有被覆盖会进行复制Point的一份表。 



2、对于inline函数，一般情况下会进行就地展开，如

```c++
inline int min(int i,int j){
	return i<j?i:j;	
}
```

具体展开方式如下:

```c++
int minval = min(val1, val2); 
//转换为
int minval = val1<val2?val1:val2;
```

但是对于参数需要创建临时对象，

```c++
int minval = min(foo(), bar()+1);
//展开为
{
    int t1;int t2;
    //对于，分割的多个表达式赋值时取最后一个
    minval = (t1=foo()),(t2=bar()+1), t1<t2?t1:t2;
}
```

因此使用内敛函数时需要慎重考虑，并不是只有简单的替换。



## 4、执行期语义学

1. new 和 delete

   关于new的一般过程如下：

   ```c++
   int *pi = new int(5);
   //实际的过程如下所示：
   int *pi = __new(sizeof(int));
   *pi = 5;
   ```

   delete的过程一般如下：

   ```c++
   delete pi;
   //转换为
   if(pi!=0)
       __delete(pi);
   ```

   在delete pi后，pi并不会被自动清除为0。

   对于一个类对象来说,在new和delete时会调用其构造和析构函数：

   ```c++
   Point *p = new Point();
   //转换为(不考虑异常)
   Point* p;
   if(p = __new(sizeof(Point)))
   	p = Point::Point(p);
   
   //delete p;转换为
   if(p!=0){
       Point::~Point(p);
       __delete(p);
   }
   ```
2. 