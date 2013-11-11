---
layout: post
title: "Effective c++之实现"
category: "c_cpp"
---

> 本部分包括条款26－条款31， 本章涉及到的内容有**太快定义变量，过度使用转型，过度耦合**等内容

### 条款26 尽可能延后变量定义式的出现时间

只要你定义了一个变量而其类型带有构造函数和一个析构函数时，那么当程序的控制流到达这个变量的定义式时， 你便得承受构造成本，同理，析构成本也会在离开作用域的时候产生。  

过早定义还会导致一种情况就是，异常产生后，真正的对象却没有来的急使用。

例如：  

    std::string f(const std::string & password){
        using namespace std;
        string encrypted;
        if (password.length()<MinimumPassWordLength) {
            throw logic_error("Password is too short");
            ...
        }
        ...
        //我们可将encrypted的声明放在此处
        return enctypted;
    }


可取的做法是以password作为encrypted的初值，跳过毫无意义的default构造过程。

    std::string g(std::string & password){
        ...
        std::string encrypted(password);
        
        encrypt(encrypted);
        return encrypted;
    }


如此，我们便可看出来，你不应该只延后变量的定义，知道非得使用该变量的前一刻位置为止，甚至应该尝试延后这份定义直到能够给它初值实参为止。  

> 请记住：尽可能延后变量定义式的出现，这样做可以增加程序的清晰度并改善程序效率

### 条款27：尽可能少做转型操作。

c类型的转型操作在c++中是不安全的，我们应该尝试使用更加清晰明了的c++转型运算符。  

- const_cast 只做从常量到非常量的转型
- dynamic_cast 主要用来“类的继承层次中安全的向下转型“ 
- reinterpret_cast 意图执行低级转型，实际动作（及结果）可能取决于编译器，这也意味着不可移植性
- static_cast 用来强制隐式转换， 类似于c风格


    class base{};
    class derived:public base{};
    derived d;
    base *pb = &d;


有时， 上述两个指针值未必相同，同时也表明，单一对象可以拥有一个以上的地址。分别是以base指向它和以derived指向它。

之所以需要dynamic_cast，通常是因为你想在一个你认定为derived class对象身上执行derived class操作函数，但你的手上却只有一个指向base的pointer或reference，你只能靠他们来处理对象。有两个一般性的做法可以避免这个问题。

- 第一使用容器，并在其中存储直接指向derived class对象的指针。
- 在base class内提供虚拟函数作你想对各个派生类做的事情。

这种转型通常是不可移植的，这主要看各个厂商的编译器是如何实现的。优良的c++代码很少使用转型，但若要完全摆脱他们又太过不太实际。

>请记住：

- 如果可以，尽量避免转型，特别是在注重效率的代码中避免dynamic_casts。如果有个设计需要转型动作，试着发展无需转型的替代设计。
- 如果转型是必要的，试着将他隐藏于某个函数背后。客户随后可以调用这个函数，而不需要将转型放进他们自己的代码。
- 宁可使用c++style转型，不要使用旧式转型。前者很容易被辨别出来。

###条款28：避免返回handles指向对象内部成分

考虑一段代码：

    class Point{
    public:
        Point(int x, int y);
        ...
        void setX(int newval);
        void sety(int newval);
        ...
    };
    struct RectData{
        point ulhc;
        point lfhc;
    };
    class Rectangle{
    ...
    private:
        std::tr1::shared_ptr<RectData>pData;
        
    public:
        Point &upperleft() const;//我们定义了一个返回handle的一个常量特性公共函数，此处应该加上const
    };
    Point coord1(0,0);
    Point coord2(100, 100);
    const Rectangle rec(coord1, coord2);
    rec.upperleft.setx(50)；//透过这个函数，我们可以改变Rectangle对象内部成员Point访问权限。

这立刻给了我们俩个教训，

- 成员变量的封装性最多至等于“返回其reference”的函数的访问级别。
- 如果const成员函数传出一个reference，后者所至数据与对象自身有关联，而他又被存储于对象之外，那么这个函数的调用者可以修改那笔数据。

referece、指针和迭代器统统都是所谓的handles，而返回一个"代表对象内部数据"的handle，随之而来的便是“降低对象封装性”。

> 请记住：
避免返回handles（包括reference，指针，迭代器）指向对象内部。遵守这个条款可以增加封装性，帮助const成员函数的行为像个const，并将发生的“虚吊号码牌”的可能性降至最低。

###条款29：为异常安全而努力是值得的
“异常安全”有两个条件，当异常被抛出时，带有异常安全性的函数会：

- 不泄露任何资源
- 不允许数据败坏

并且，异常安全至少提供以下三个保证中的一个

- 基本承诺：
- 强烈保证：如果成功则改变，如果不成功，则退回原始状态
- 不抛掷异常保证

这部分内容太难，还没有完全吃透。

###条款30：透彻了解inlining的里里外外
inline函数看起来像是函数，动作像函数，比宏好的多，可以调用它们而又不需要蒙受函数调用所招致的额外开销。

将函数inlining确实可以导致较小的目标代码产生和较高的指令高速缓存装置击中率。

但是，inline只是对编译器的一个申请和建议，不是强制命令，编译器有权作出自己的选择，决定要不要实施inline。

inline函数通常放在头文件中，因为inlining大多数是编译时期的行为。

> 请记住：

- 将大多数inlining限制在小型、频繁调用的函数身上。这可使日后的调用过程和二进制升级更容易，也可使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化
- 不要只因为function templates出现在头文件中，就将他们声明为inline

###条款31：将文件间的编译依赖关系降至最低
支持“编译依存性最小化”的一般构想是：相依于声明式，不要相依于定义式。基于此构想的两个手段是Hanles classes 和Interface classes。

所谓handles classes 就是成员函数必须同implementatioin取得对象数据。即使说，classes中不存有数据的实现细目。而增加一个共享指针，并前置声明这个对象。这无疑是增加了一层的间接性。

至于interface classes，由于每个函数都是virtual的，所以你必须为每次函数调用付出一个间接条约成本。此为每个实例对象中必须保存着一个vptr指针，这无疑增加了空间的消耗成本。

程序库头文件应该以“完全且仅有声明式”的形式存在。这种做法不论是否涉及template都适用。
> Written with [StackEdit](http://benweet.github.io/stackedit/).
