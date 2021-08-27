1. ### 视C++为一个语言联邦

   ​	c++由 c、object-oriented c++（面向对象）、template c++、 STL组成，尽量要求都掌握。

2. ### 尽量以const，enum，inline替换#define

   ​	define是预处理时期做的事情，编译器可能没法检查错误某些错误，会造成意向不到的后果。遵循 ”宁可以编译器替换预处理器“，下面讲的都是通常情况，适用于大部分场景，还有一些场景不适合替换。

   **const代替define：**

   ```c++
   #define VAL 1.635 //万一出错，在一种场景下，编译器只会报1.635的错，难以定位
   
   /*const 代替*/
   const double VAL = 1.635 //让编译器去检查，报错的话会直接出VAL的错
   ```

   **const，enum代替define:**

   ```c++
   /*1. static const int 初始化在类内，但是有的编译器不允许在类内初始化static*/
   class t {
       static const int num = 5; //有的编译器必须要在类外初始化
       int arr[num]; //数组大小要求在编译器就能确定
   }
   /*2. static const int 初始化在类外*/
   class t {
       static const int num;
       int arr[num]; //这里会报错，因为没有在编译期就确定num大小
   }
   const int t::num = 5;
   /*3.用enum代替,称为"enum hack技术"*/
   class t {
       enum {num = 5};
     	int arr[num];  
   }
   ```

   **inline代替define函数**

   ```c++
   /*define函数 每个形参都要加括号，并且可能出现逻辑严重错误*/
   #define CMP_MAX(a,b) f((a) > (b)? (a):(b))
   int a = 5, b = 0;
   CMP_MAX(++a,b); // ++a 会执行两次
   CMP_MAX(++a,b + 10) // ++a 会执行一次，形参不同逻辑发生变化，出现错误
   
   /*inline代替*/
   template <class T> 
   inline void CMP_MAX(const T& a,const T& b) {
       f(a > b ? a : b);
   }
   ```

   **要点：**

   1. 对于单纯常量，用const和enum代替#define
   2. 对于形似函数的宏(MACROS)，最好用inline替换#define

3. ### 尽可能使用const

   **要点：**

   1. const成员函数可以修改static变量，static成员函数则不能修改const成员变量

   2. 重载[]时要使用引用才能实现下列代码

      ```c++
      char& operator[](size_t pos) const {
      	return oo[pos];
      }
      oo[0] = 'x'; //如果[]没返回引用这句话就会报错
      
      ```

   3. 讲某些东西声明为const可以帮助编译器检测出错误用法，const可以被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。

   4. 编译器强制试试bitwise constness(函数后面加const)，但是编写程序时要有概念上的常量性。

   5. 当const和non-const成员函数有着实质等价替换的实现时，令non-const版本调用const版本避免代码重复。

4. ### 确定对象被使用前已经被初始化

   **要点：**

   1. 为内置类型对象进行手工初始化，因为C++不保证初始化它们

   2. 构造函数最好使用列表初始化（一次构造），而不要在构造函数本体内使用赋值操作。列表初始化顺序应该和声明顺序相同。

   3. 为免除”跨编译单元之初始化次序“问题，请以local static对象替换non-local-static对象，参考单例

      ```c++
      /*1. non- local static*/
      class FileSystem {
      pubilc:
      	...
          std::size_t numDisks() const;
      };
      extern FileSystem tfs; 
      class Directory {
      public：
          Directory( params );
          ...
      };
      Directory:Directory( params ) {
          ...
          std::size_t disks = tfs.numDisks(); //难以保证tfs已经被初始化
          ...
      }
      Directory tmpDir( params ); //可能会出错
      
      /*2.替代为local static*/
      class FileSystem {...}
      FileSystem& tfs() {
          static FileSystem fs;
          return fs;
      }
      class Directory {...};
      Directory:Directory( params ) {
          ...
          std::size_t disks = tfs().numDisks(); //类似单例
          ...
      }
      Directory& tmpDir() {
          static Directory dr;
          return dr;
      }
      ```

5. ### 构造/析构/赋值运算

   **要点：**编译器默认为class生成构造，析构（no virtual），拷贝构造，拷贝赋值运算符。

6. ### 若不想使用编译器自动生成的函数，就该明确拒绝

   ​	编译器默认为class生成构造，析构（默认生成的析构是no virtual属性的），拷贝构造，拷贝赋值运算符，如果要禁止某些功能，就要手动实现。

   **要点：**

   ​	为驳回编译器自动提供的技能，可以将成员函数声明为private并且不予实现。使用一个Uncopyable这样的基类（base class）也是一种做法：

   ```c++
   class Uncopyable {
   public:
   	...
   private:
   	Uncopyable(const Uncopyable& );
   	Uncopyable& operator=(const Uncopyable&);
   }
   
   class test: public Uncopyable {
       ... 						// 不再声明拷贝构造和拷贝赋值
   }
   ```

7. ### 为多态基类声明virtual析构函数

   问题：如果一个用一个基类指针指向一个继承类，而这个基类里面的析构函数为non-virtual，那么析构时可能导致继承类成分未被析构。

   **要点：**

   1. 带多态的基类应该声明一个virtual析构函数。如果class带有任何virtual函数，它就应该拥有一个virtual析构。
   2. 如果不是为了当基类（class里面至少含有一个virtual），那么就不改声明virtual，因为声明virtual时会使对象内存变大。

8. ### 别让异常逃离析构函数

   **要点：**

   1. 析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下他们或结束程序。（不然会导致析构函数无法正确释放所有资源）

      

   2. 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数（而非析构函数）去处理异常。

9. ### 绝不在构造和析构过程中调用virtual函数

   **问题：**基类构造函数会先于派生类构造函数被调用，如果此时在构造函数内调用virtual函数，将在派生类没有初始化之前就调用它的函数，会造成意想不到的错误。实际上调用的是基类的函数。

   **要点：**

   不在构造和析构过程中调用virtual函数，因为它从不下降至派生层。

10. ### 令operator = 返回一个 reference to *this

    为了实现如下代码

    ```c++
    int x, y, z;
    x = y = z = 15;//赋值连锁
    ```

    要将operator=写为如下形式

    ```c++
    classType& operator=(const classType& rhs) {
    	...
    	return* this;
    }
    
    classType& operator+=(const classType& rhs) {
        ...
        return* this;
    }
    ```

    要点：

    令operator=、operator*=、operator\*=等都返回一个reference to * this;

11. ### 在operator=中处理自我赋值

    看以下代码
    
    ```c++
    widget& widget::operator=(const widget& rhs) {
    	delete pb; //要是this对象和rhs是一个对象，这样就会极度不安全
    	pb = new bitmap(*rhs.pb);
    	return *this;
    
    }
    
    
    //修改为
    widget& widget::operator=(const widget& rhs) {
        if (this == &rhs) return *this; //加入测试结果
    	delete pb; 
    	pb = new bitmap(*rhs.pb);
    	return *this;
    }
    
    //也可以像下面这样，但是感觉开销大
    widget& widget::operator=(const widget& rhs) {
        bitmap* tmp = pb;
    	pb = new bitmap(*rhs.pb);
        delete(tmp);
    	return *this;
    }
    
    ```
    
    要点：
    
    - 确保对象自我赋值的时候operator=有良好行为。其中技术包括比较”来源对象“和”目标对象“的地址、精心周到的语句顺序、以及copy&swap。
    - 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。
    
12. ### 复制对象时勿忘其每一个成分

    注意，当你编写一个copying函数，确保复制所有的1）local成员变量 2）调用所有base class的copying函数。添加一个成员变量的时候，要确保能够修改copying函数

    ```c++
    class base {
    	...
    }
    //问题代码
    class derive: public base {
    	derive(const derive& rhs):m(rhs.m) {
            	//base 类未被构造
    	}
    	derive& operator=(const derive& rhs){
    		m = rhs.m;
    		return *this;
    	}
    private:
    	int m;
    }
    
    
    //修改为
    class derive: public base {
    	derive(const derive& rhs):
        base(rhs);
        m(rhs.m) {
    	}
    	derive& operator=(const derive& rhs){
            base::operator=(rhs);
    		m = rhs.m;
    		return *this;
    	}
    private:
    	int m;
    }
    ```

    要点：

    - coping函数应该确保复制”对象内的所有成员变量“和”所有base class成员“
    - **不要尝试**以某个copying函数实现另一个copying函数。应该将共同技能放进第三个函数中，并由两个coping函数调用。

13. ### 以对象管理资源

    要点：

    - 为了防止资源泄露，使用RAII对象，构造获得资源，析构释放资源。
    - 尽量使用shared_ptr和auto_ptr，shared_ptr较好。复制动作会使他指向null。

14. ### 在资源管理类中小心copying行为

    对于RAII类应该有以下注意的点：

    要点：

    - 复制RAII对象必须一并复制它所管理的资源，所以资源的拷贝行为决定RAII的拷贝行为。
    - 普通的RAII类拷贝行为应该是：禁止拷贝、施行引用技术法。

15. ### 在资源管理类中提供对原始资源的访问

    要对RAII类实现像shared_ptr中get()的这个功能，获取原始指针或者资源。

    要点：

    - APIs往往要访问原始资源，RAII类应该提供方法
    - 类似get()这种方法要提供显示转换和隐式转换两个版本，隐式转换可能不太安全，但是对客户方便。

16. ### 成对使用new和delete要采取相同措施

    要点：

    ​	在new中使用[]，delete也要使用[]，不使用也要一起不使用。

17. ### 以独立语句将newed对象置入智能指针

    ```c++
    shared_ptr<widget> ptr(new widget); //例如如此
    ```

    要点：

    ​	以独立语句将newed对象存储与智能指针内。如果不这样做，一旦异常被抛出，有可能导致难以察觉的资源泄露。

18. ### 让接口容易被正确使用，不易被误用

    要点：

    - 好的接口很容易被正确使用，不容易被误用。你应该在你的所有接口中努力达成这些性质。
    - 促进正确使用的办法包括接口的一致性，以及与内置类型行为兼容。
    - shared_ptr支持自定义删除器，可以防范DLL问题，可被用自动解除互斥锁（mutex），总之多用shared_ptr。

19. ### 设计class犹如设计type

    设计一个新的类应该如同定义一个新的type一样。应该关注以下问题：

    - 新type对象如何创建和销毁？（构造和析构以及运算符重载）
    - 对象的初始化和对象赋值该有什么差别？（条款4）
    - 新的type对象如果被按值传递，意味着什么？（拷贝构造应该怎么写？）
    - 什么是新的type对象的合法值？（异常检查）
    - 新的type需要配合某个继承图系吗？（虚析构函数以及其他虚函数的定义）
    - 新的type需要什么样的转换？（如果是显示构造，需要一个类型转换函数）
    - 什么样的操作符和函数对此新type是合理的？
    - 什么样的标准函数应该驳回？（条款6）
    - 谁该取用新的type成员？（pubilc、proteced、private以及friend的定义）
    - 什么是新type的未声明接口？（条款29）
    - 新type的一般化？（是不是该定义为模板）
    - 真的需要一个新type吗？（有时并不需要继承，可能非成员函数和模板能更好达到目的）

    要点：

    ​	class设计应该遵从上述几点的原则。

20. ### **宁以pass-by-reference-to-const替换pass-by-value**

    ​	拷贝构造非常耗时，建议传常量引用代替值传递。引用传递也可以避免对象切割，即会出现这种情况，derive按值传递并被视为一个base 对象，会导致base构造函数被调用，多态性质完全被消除，只表现为一个base对象。

    ```c++
    class base {
    public:
    	virtual void foo();
    	...
    }
    
    class derive:: pubilc base {
    public:
    	virtual void foo();
    }
    
    //按值传递出现对象切割情况
    void fun(base b) ; 
    
    //正确用法
    void fun(const base& b);
    
    ```

    要点：

    - 一般而言，你可以合理假设”按值传递开销并不大“的唯一对象是内置类型和stl迭代器和函数对象。上面这三种用值传递比较合理，其余都尽量用常值引用代替。
    - 用常值引用代替值传递，高效且安全。

21. ### 必须返回对象时，别妄想返回reference

    ​	一个”必须返回新对象“的函数正确写法：在栈上、堆上和静态局部变量都是可能有问题的，应该这样写：

    ```c++
    inline const op operator*(const op& lhs, const op& rhs) {
    	return op(lhs.n * rhs.n, lhs.d * rhs.d);
    }
    ```

    避免不了再构造一次和析构一次的开销，但是只是小小代价。

    要点：

    ​	绝对不要返回指针或者引用指向一个局部栈对象（传出来以前已经被销毁），或返回引用指向一个堆对象（没法析构），或者返回指针或者引用指向一个局部静态对象而有可能同时需要很多个这样的对象，参考条款4。

22. ### 将成员变量声明为private

    ​	public和protected成员变量不利于程序的精细化控制和程序可拓展性，最好public都是函数。从封装的角度看，只有两种访问权限：private（提供封装）和其他（不提供封装）

    要点：

    - 切记将成员变量声明为private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性。
    - protected其实和public差不多，不提供封装。

23. ### 宁以non-member，no-friend替代member函数

    ​	设计理念的区别，用namespace的non-member，no-friend来代替member函数

    要点：

    ​	拿非成员函数、非友元函数来替换成员函数，可以增加封装性、包裹弹性和机能扩充性。

24. ### 若所有参数皆需类型转换，请为此采用non-member函数

    ​	令class支持隐式类型转换通常是一个糟糕的主意。

    ```c++
    class Rational {
    public:
    	...
    	const Rational operator* (const Rational& rhs) const;
    }
    
    result = oneHalf * 2;	//正确,发生隐式转换，2生成一个tmp的Rational对象
    //等价于 result = oneHalf.operator*(2);
    result = 2 * oneHalf;	//错误
    //等价于 result = 2.operator*(oneHalf);
    
    此时的做法应该时将oprator* 改为 non-member函数
    
    class Rational {
    	...
    }
    const Rational operator*(const Rational& lhs, const Rational& rhs);
    result = oneHalf * 2;//正确
    result = 2 * oneHalf; //正确
    ```

    要点：

    ​	如果你需要为某个函数的所有参数（包括this指向的那个隐藏参数）进行类型转换，那么这个函数必须是个non-member。

25. ### 考虑写出一个不抛出异常的swap函数

    指针拷贝时，swap会进行深拷贝，缺乏效率。

    要注意的点：

    1. 如果std::swap已经可以为你的class提供可接受的效率，那么不要额外做任何事情。
    2. 如果要swap效率不足，那么尝试做1）提供一个public swap函数，高效交换你类型中的两个对象值，这个函数绝不抛出异常。2）在你的class的命名空间提供一个non-member swap版本，令他调用上述member swap函数。3）如果在编写一个class,为你的class特化std::swap，并令他调用你的class swap。
    3. 如果调用swap，可以使用using，例如using std::swap。可以直接使用swap。swap一般调用匹配度最高的那个。

    要点：

    - 当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不抛出异常。
    - 如果你提供一个swap成员函数，也该提供一个非成员函数swap来调用它。对于classes，请特化swap
    - 调用swap时应该针对std::swap使用using声明式，然后调用swap并且不带任何”命名空间资格修饰“
    - 为”用户定义类型“进行std 模板全特化时好的，但是不要为std加入什么新的东西。

26. ### 尽可能延后变量定义式的出现时间

    ​	应该延后定义变量直到能够给他初值初始化为止，避免拷贝行为的开销。

    要点：

    ​	可能延后变量定义式的出现时间。可以增加程序的清晰度并改善程序效率。

27. ### 尽量少做转型动作

    ​	用c++风格的转型函数，static_cast\<T\>,reinterpret_cast\<T\>,dynamic_cast\<T\>和const_cast\<T\>。

    要点：

    - 尽量少使用转型，特别是在注重效率的代码中避免dynamic_cast。
    - 如果不能避免转型，试着将其隐藏于某个函数背后。客户随后可以调用该函数，而不需将转型放进他们自己的代码中。
    - 尽量使用c++风格转型，即上面四个模板类转型函数。

28. ### 避免返回handles指向对象内部成分

    ​	具体函数见书本，handles指向对象的话会造成一种const对象，但是可以通对象内部的handles去修改内部私有对象的状况。

    要点：

    ​	避免返回handles(引用，指针，迭代器) 指向对象内部。遵守这个条款可增加封装性，帮助const成员函数的行为像个const，并避免空悬指针（指向以及被释放的区域）的情况。

29. ### 为“异常安全”的努力是值得的

    异常安全的函数具有：

    1. 不泄露任何资源。
    2. 不允许数据败坏。

    应该保证异常安全：

    1. 基本承诺：

       如果异常被抛出，程序内的任何事物仍然保持有效，也就是要具有一致性。

    2. 强烈保证：

       如果异常被抛出，程序状态不改变。如果函数成功，就是完全成功，如果函数失败，回滚。原子性。

    3. 不抛掷保证（nothrow）：

       承诺绝不抛出异常，因为能完成它们原先承诺的功能。

    **copy and swap方法：**为你打算修改的对象拷贝（copy）一份副本，然后在副本上做一切必要的改变。若有任何修改动作抛出异常，原对象仍保持未改变状态，所有改变完成以后，再和原对象交换（swap）。

    要点：

    - 异常安全函数即使发生异常也不会泄露资源或允许任何数据结构被破坏。这个函数的有三种可能保证：基本，强烈，不抛掷。
    - 强烈保证以**cas**来保证，但是并不适合所有函数。
    - 函数异常安全等级取决于里面的最小级别。

30. ### 透彻了解inlining的里里外外

    ​	一个inline函数是否真的被inline取决于你的建置环境，主要取决于编译器。作用于编译期。在声明时定义自动成为inline。

    要点：

    - 将大多数inlining限制在小型、被频繁调用的函数身上。这可使得日后的调试过程和二进制升级更容易，也可使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化。
    - 不用讲模板函数无脑声明为inline。

31. ### 将文件间的编译依存关系降至最低

    问题：只修改一小部分，remake耗时非常严重。

    使用前置声明。

    ```c++
    #include <string>
    #include "data.h"
    #include "address.h"
    //一旦修改以上头文件被修改，那么所有person.h（本文件）和包含上述头文件也都将被修改，会造成牵一发而动全身的概念
    class Person {
    private:
    	std::string theName;
    	Data theData;
    	Address theAddress;
    };
    //修改为
    #include <string>
    class Data;//前置说明
    class Address;
    class Person {
    private:
    	std::string theName;
    	Data theData;
    	Address theAddress;
    };
    ```

    要点：

    - 该条款的一般构想使：相依于声明，不要相依于定义式。实现这两个构想的手段是接口类（Handle Class）和界面类(Interface Class抽象基类)。
    - 头文件应该只有声明，模板也只有头文件。
    
32. ### 确定public继承塑膜is-a关系

    要点：

    ​	public继承完全是is-a的关系，即适用base类的public方法也适用于derive类，derive类视为一个base类对象。

33. ### 避免遮掩继承而来的名称

    ```c++
    class Base {
    public:
    	virtual void f1() = 0;
    	virtual void f1(int);
    	virtual void f2();
    	void f3();
    	void f3(double);
    };
    class Derive: public Base {
    public:
    	virtual void f1();
    	void f3();
    	void f4();
    };
    // main
    Derive d;
    int x;
    d.f1(); // 正确，调用Derive::f1();
    d.f1(x); //错误，Derive::f1()遮掩了Base::f1();
    d.f2(); //正确，调用Base::f2();
    d.f3(); //正确，调用Dervie::f3();
    d.f3(x); //错误，Derive::f3()遮掩了Base::f3();
    
    //可以修改为下面，但是违反public is-a原则
    class Derive: public Base {
    public:
        using Base::f1; 
        using Base::f3;
    	virtual void f1();
    	void f3();
    	void f4();
    };
    d.f1(x); //正确，Base::f1();
    d.f3(x); //正确，Base::f3();
    
    
    //转交函数： 不希望继承所有public时，使用private继承
    class Derive: private Base {
    public:
        virtual void f1() {
            Base::f1();  //转交函数
        }
    }
    
    d.f1();//正确 Derive::f1();
    d.f1(x); //错误，Base::f1(int)被遮掩了
    ```

    

    要点：

    - derive类会遮掩base类内的名称，在public继承下不希望入错。
    - 可以用using和转交函数使被遮掩的函数重现。

34. ### 区分接口继承和实现继承

    1. pure virtual函数的目的是为了让derive class只继承函数接口，可以实现纯虚函数，但是需要调用时加上类名。

       ```c++
       Shape* ps = new Rectangle;
       ps->Shape::draw(); //正确，draw是Shape中的纯虚函数。
       ```

    2. 声明普通虚函数的目的是让derive类继承该函数接口和缺省实现

    3. 声明普通函数目的是为了derive类继承函数的接口及一份强制性实现。

    要点：

    - 接口继承和实现继承不同。在public继承下，derive class总是继承base class的接口。
    - **纯虚函**只具体指定接口继承
    - **普通虚函数**函数集体指定接口继承以及缺省实现继承
    - **非虚函数**具体指定接口继承以及强制性实现继承

35. ### 考虑virtual函数以外的其他选择

    1. 使用**非虚函数**接口实现Template Method模式
    2. 籍由**函数指针**实现Strategy模式

    要点：

    - 接口继承和实现继承不同。在public继承下，derive总是继承base的接口。
    - 纯虚函数只具体指定接口继承。
    - 非纯虚函数具体指代接口继承及缺省实现继承。
    - 非虚函数具体指定接口继承以及强制性性实现继承。

36. ### 绝不重新定义继承而来的non-virtual函数

    ​	non-virtual是静态绑定的，指针类型是什么就调用那个对象的non-virtual函数。virtual函数则是动态绑定的。

    要点：

    ​	绝不重新定义继承而来的non-virtual函数，如果要表现多态，就要定义为virtual。

37. ### 绝不重新定义继承而来的缺省函数值

    动态绑定（virtual，运行期）和静态绑定(static，编译期)。

    ```c++
    Shape* pa; //pa的静态类型为shape，动态类型无
    Shape* pb = new Rectangle;	//pb的静态类型为shape，动态类型Rectangle
    Shape* pc = new Circle;	//pc的静态类型为shape，动态类型Circle
    
    pa = pb //此时动态类型为Rectangle
    ```

    virtual函数调用哪一份实现代码，取决于调用的那个对象的**动态类型**。普通函数调用则是取决静态类型。

    ​	当virtual函数带着缺省值时，**缺省参数值是静态绑定的。**原因：为了运行期效率，把活扔给编译器。

    ```c++
    class Shape {
    public:
    	virtual void draw(Color c = Red);
    };
    class Rectangle : public Shape {
    public:
    	virtual void draw(Color c = Green); 
    }
    
    Rectangle pr;
    pr->draw(); //此时c = Red
    ```

    利用NVI替代设计，讲virtual声明为private，然后用non-virtual去调用virtual，缺省值放在non-virtual中。

    ```c++
    class Shape {
    public:
    	void draw(Color c = Red) const { //指定缺省值
    		doDraw(c);
    	}
    private:
    	virtual void doDraw(Color c); //不指定缺省值
    };
    class Rectangle : public Shape {
    public:
    	...
    private:
    	virtual void doDraw(Color c); //不指定缺省值
    }
    ```

    要点：

    ​	绝不重新定义继承而来的缺省函数值，因为缺省参数值是静态绑定的，而virtual函数是动态绑定的。

38. ### 通过符合塑膜出has-a或”根据某物实现出“

    pubilc继承is-a（是一种）的意义。复合意味着has-a（有一种）的意义。

    ```
    //一种has-a的关系
    class Person {
    private:
    	std::string m1;
    	Data m2;
    	Addr m3;
    }
    ```

    要点：

    - 复合和pubilc继承完全不同
    - 应用域复合意味着has-a。在实现域，复合意味着根据某物实现（set以list为底层数据结构实现）。

39. ### 明智而谨慎的使用private继承

    ​	如果使用private的继承，那么编译器不会将一个derive类对象自动转化为base类对象。并且base中的所有对象都会变成private的属性。

    ```c++
    Class Empty() {};
    Class D {
    	int x;
    	Empty e;
    }
    // sizeof(D) > sizeof(int);原因在于Empty class占空间（1 byte）
    Class Empty() {};
    Class D : private Empty{
    	int x;
    }
    //此时sizeof(D) = sizeof(int);原因在于private继承
    ```

    要点：

    - Private继承意味着根据某物实现出。比复合等级低，当继承类需要访问protected base class的成员，或者重新定义继承而来的virtual函数时，这么设计时合理的。
    - 和复合不同的是，private的继承可以造成empty base的最优化。

40. ### 明智而谨慎使用多重继承

    使用virtual继承的建议，如果不太会用尽量不使用

    1. 尽量不使用virtual继承。
    2. 尽可能在virtual base上放数据。

    要点：

    - 多重继承比单一继承难，可能导致新的歧义以及对virtual继承的需要。
    - virtual继承会增加对象大小、速度、初始化复杂度等等成本。如果virtual base不带有任何数据，那是最有实用价值的情况。
    - 多重继承的确有正当用途：public继承接口类，private继承某个协助实现的class。

41. ### 了解隐式接口和编译期多态

    ​	模板和类的区别，模板和类实际上是相似的，从接口和多态两个方面来说。

    要点：

    - 类和模板都支持接口和多态
    - 对于类而言，**接口是显式的**，以函数签名为中心。多态则是**通过virtual函数发生在运行期**
    - 对于模板而言，接口是隐式的，奠基于有效表达式。多态则通过**模板局现化和函数重载解析发生于编译期**

42. ### 了解typename的双重意义

    ```c++
    template<typename T> class w;
    template<class T> class w;
    ```

    ​	实际上没有区别。

    ```c++
    template<typename C>
    void print2nd(const C& container) {
        if (container.size() >= 2) {
            typename C::const_iterator iter(container.begin());
            //用typename说明C::const_iterator是一个类型
        }
    }
    ```

    ​	如果你想在template中涉及一个**嵌套从属类型**名称，就必须在紧邻它的前一个位置放上关键字typename。

    例外：typename不能出现在基类列表中。

    ```c++
    template<typename T>
    class Derive : public Base<T>::Nested{ //不能出现typename
    }
    ```

    要点：

    - 声明模板参数时class和typename没有区别
    - typename有更多的用法，标识**嵌套从属类型**名称，但是typename不能出现在基类列表和成员初始化列表中。

43. ### 学习处理模板化基类内的名称

    问题：当继承基类模板时会出现以下情况
    
    ```c++
    template <typename T>
    class Derive:: public Base<T>{
    public:
    	void DeriveFunc() {
    		BaseFunc(); //这是基类里面的函数
    	}
    };
    ```
    
    这样的情况会无法编译通过，原因是因为基类可能被特化，而被特化的模板没有这个BaseFunc（）函数，可行的解决方法是：
    
    ```c++
    //1、用this指针
    template <typename T>
    class Derive:: public Base<T>{
    public:
    	void DeriveFunc() {
    		this->BaseFunc(); //这是基类里面的函数，编译通过，假设BaseFunc被继承
    	}
    };
    
    //2、用using
    template <typename T>
    class Derive:: public Base<T>{
    public:
        using Base<T>::BaseFunc();//告诉编译器，请他假设该函数位于base class内
    	void DeriveFunc() {
    		BaseFunc(); //这是基类里面的函数，编译通过，假设BaseFunc被继承
    	}
    };
    
    //3、命名空间
    template <typename T>
    class Derive:: public Base<T>{
    public:
    	void DeriveFunc() {
    		Base<T>::BaseFunc(); //这是基类里面的函数，编译通过，假设BaseFunc被继承
    	}
    };
    
    ```
    
    要点：
    
    - 可在继承模板类内通过”this->“指涉基类模板类内的成员名称，或藉由一个明白写出的”基类资格修饰符“完成
    
44. ### 将与参数无关的代码抽离templates

    问题：以下代码会生成两份类似的代码，造成代码膨胀。

    ```c++
    template <class T, size_t n>
    class SM {
    public:
    	void invert();
    };
    SM<int, 5> sm1;
    SM<int, 10> sm2;
    sm1.invert(); //调用SM<int,5>::invert
    sm2.invert(); //调用SM<int,10>::invert
    ```

    解决方法：

    ​	将参数无关的代码剥离出模板

    ```c++
    template <class T> 
    class SM {
    protected:
        void invert(size_t n); //所有继承类都可使用同一个invert
    }
    
    template <class T, size_t n> 
    class DSM: private SM<T> {
    private:
        using SM<T>::invert(); 
    public:
        void invert() { this->invert(n); } //inline调用
    }
    ```

    要点：

    - templates生成多个类和多个函数，任何的模板代码都不该与某个造成膨胀的模板参数产生关联。
    - 因**非类型模板参数**而造成的代码膨胀，往往可消除，做法是以函数参数或class成员变量替换模板参数。
    - 因**类型模板参数**而造成的代码膨胀，往往可降低，做法让带有完全相同二进制表述的具体实现类型共享实现码。

45. ### 运用成员函数模板接受所有兼容类型

    ​	为了实现智能指针的隐式转化，希望如下所示的代码通过编译：

    ```c++
    template <class T>
    class SmartPoint {
    public：
    	SmartPoint(T* realPtr); //传入原始指针
    };
    SmartPoint<Base> ptr1 = SmartPoint<Derive>(new Derive); //类型自动转换
    ```

    ​	但是此时编译器并不知道Base和Derive的关系，所以无法自动转换，修改为：

    ```c++
    template <class T>
    class SmartPoint {
    public：
        template <class U> //成员函数模板
    	SmartPoint(const SmartPoint<U>& realPtr); //拷贝构造
    };
    SmartPoint<Base> ptr1 = SmartPoint<Derive>(new Derive); //类型自动转换
    ```

    要点：

    - 请使用成员函数模板生成可接受的所有兼容类型函数
    - 如果你声明的成员函数模板用于”泛化cpoy构造“或”泛化赋值操作“，你还是需要声明正常的cpoy构造和赋值操作符。

46. ### 需要类型转换时请定为模板定义非成员函数

    根据条款24，只有非成员函数才有能力对实参进行隐式转化，以24条款为例，改为模板

    ```c++
    template <class T>
    class Rational {
    	...
    }
    template <class T>
    const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs);
    const Rational<T> result = oneHalf * 2;//报错
    ```

    原因是由于编译时无法推导出2的这个模板T是int，模板推导时不会讲隐式类型转化函数纳入考虑，并且此时类内没有具体的函数实现，则报错。用friend实现具体函数。

    ```c++
    template <class T>
    class Rational {
    public:
    	friend
    	const Rational operator*(const Rational& lhs, const Rational& rhs) {
            return Rational(lhs.num() * rhs.num(), lhs.deno() * rhs.deno()); //具体实现
        }
    };
    template <class T>
    const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs);
    const Rational<T> result = oneHalf * 2;//通过编译并运行
    
    ```

    此时const Rational<T> result = oneHalf * 2；知道去调用operator*(const Rational\<int\>& lhs, const Rational\<int\>& rhs);来实现，此时调用的是friend函数。

    要点：

    - 当我们编写一个class template时，而它所提供之”与此template相关的“函数支持”所有参数之隐式类型转换“时，将其声明为friend。

47. ### 使用traits classes表现类型信息

    STL里面有5种迭代器，是is-a的关系，对迭代器分类进行traits class的编写。

    要点：

    - traits classes 使得”类型相关信息“在编译器可用。它们以**模板**和**模板特化**完成实现。
    - 整合重载技术后，traits classes有可能在编译期对类型执行if else测试。

48. ### 认识模板元编程

    ​	模板元编程是编写模板c++程序并执行于**编译期**的过程，让工作从运行期转到编译期。模板元编程的循环依靠递归来实现。例如计算n的阶乘。

    ```c++
    template <unsigned n>
    struct Func {
    	enum {value = n * Func<n-1>::value};
    };
    template <>
    struct Func<0> {
    	enum {value = 1};
    };
    cout << Func<5>::value;//打印5的阶乘
    ```

    要点：

    - TMP可将工作从运行期转到编译期，实现更早的错误侦测和更高的执行效率
    - TMP可以用来生成”基于政策选择组合“的客户定制代码，也可用来避免生辰对某些特殊类型并不适合的代码。

49. ### 了解new-handler的行为

    ​	当c++调用new分配内存失败时，实际上是去调用了一个new-handler去进行处理，这个new-handler去负责抛出异常或是什么动作。

    ​	一个设计良好的new-handler行为应该包含：

    1. 让更多内存可被使用

    2. 安装另外一个new-handler

    3. 卸除new-handler

    4. 抛出bad_alloc的异常

    5. 不返回

       在条款51中说明了只有new-handler发生了以上5个行为才能退出死循环。

    要点：

    - set_new_handle允许客户指定一个函数，在内存分配无法获得满足时被调用。
    - Nothrow new是一个局限的工作，因为它只适用于内存分配；后续的构造函数调用还是可能抛出异常。

50. ### 了解new和delete的合理替换时机

    ​	operator new和operator  delete在某些情况可能会被替换掉。

    1. 用来检测运用上的错误。

       ​	如果new没被delete掉，那么内存泄露。如果重复delete就会造成不确定行为。自定义new可以防止这种行为，多分配一点内存，放上id，delete时就可以检查这个id是否有效。

    2. 为了强化效能。

       ​	编译器的new和delete表现过于中庸，在特定生产环境下可以自定义new和delete提高性能。

    3. 为了收集使用上的统计数据。

       ​	自定义new和delete可以收集分配了多少内存这种统计数据。

    4. 为了增加分配和归还的数据。

    5. 为了降低缺省内存管理器带来的空间额外开销。

    6. 为了弥补缺省分配器中的非最佳齐位。

    7. 为了将相关对象成簇集中。

    8. 为了获得非传统的行为。

    要点：

    ​	有许多理由编写自定的new和delete，包括改善效能、对heap运用错误进行调试、收集heap使用信息。

51. ### 编写new和delete时需要固守常规

    c++保证”删除null指针永远安全“，所以你必须兑现这项保证。

    ```c++
    void operator delete(void* rawMem) throw {
    	if (rawMem == nullptr) { //如果为空什么都不做
    		return;
    	}
    }
    //Class版本
    class Base {
        static void operator delete(void* rawMem, size_t size) throw {
            if (rawMem == nullptr) { //如果为空什么都不做
                return;
            }
            if (size != sizeof(Base)) {
                return ::operator delete(rawMem);//即交给标准化operator delete去做
            }
            return;
        }
    }
    ```

    要点：

    - operator new应该含有一个死循环，并在其中尝试分配内存，如果他无法满足内存需要，就该调用new-handler。它也应该有能力处理0 bytes申请。**Class专属的版本则还应该处理”比正确大小更大的申请“。**即交给标准化operator new去做
    - operator delete应该在收到null不做任何事情。Class专属的版本则还应该处理”**比正确大小更大的错误申请**“。即交给标准化operator delete去做

52. ### 写了placement new 也要写placement delete

    ​	placement new指的是参数列表中出现size_t以外的版本,需要对应placement delete，不然编译器就会找不到这两个对应的版本，从而可能造成new完没有delete的情况。

    ```c++
    void operator new(size_t size, void* pMemory) throw();
    void operator delete(void*, void*) throw();
    
    ```

    要点：

    - 当你写了一个placement operator new，请确定也写出了对于的placement operator delete版本。如果没有这么做，会发生内存泄露。
    - placement operator new和placement operator delete，不要遮掩他们的正常版本。

53. ### 不要轻易忽略编译器的警告

    要点：

    - 严肃对待编译器发出的警告信息。努力做到无警告
    - 不要过度依赖编译器的告警功能，不能编译器可能产生不同的效果

54. ### 让自己熟悉包括TR1在内的标准程序库

    学c++11和里面的标准库

    要点：

    - c++标准库由STL、iostreams、locales组成。并包含C99标准库
    - TR1添加了智能指针，一般化指针函数（function），正则表达式等支持
    - 学习TR1的实例Boost库

55. ### 让自己熟悉Boost

    [boost库](https://www.boost.org/)

    要点：

    ​	学

