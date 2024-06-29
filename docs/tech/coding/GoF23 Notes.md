## GoF23 Notes

Reference:  
[设计模式精解－GoF 23 种设计模式解析附 C++实现源码](http://www.mscenter.edu.cn/blog/k_eckel)  
[永不磨灭的设计模式](https://shusheng007.top/2021/09/07/design-pattern/)

#### 设计模式的基本原则：

+ OCP：开闭原则，软件实体应当对扩展开放，对修改关闭。
+ LSP：里氏替换原则，继承确保父类的性质在子类依然成立
+ DIP：依赖倒置原则，高层模块不应该依赖低层模块，两者都应该依赖其抽象；抽象不应该依赖细节，而是细节要依赖抽象。即面向接口编程
+ SRP：单一职责原则，一个类应该有且仅有一个引起它变化的原因
+ ISP：接口隔离原则，一个类对另一个类的依赖应该建立在最小的接口上，要为各个类建立它们需要的专用接口
+ LKP：最小知识原则，如果两个软件实体无须直接通信，那么就不应当发生直接的相互调用，可以通过第三方转发该调用
+ CRP：合成复用原则，优先考虑组合，其次考虑继承

####  设计模式分类：

+ 创建型模式
+ 结构型模式
+ 行为型模式

#### 创建型模式

所谓创建者模式，就是用于创建对象的模式

**1. 单例模式(Singleton)**
+ 对象由本身自行创建，构造函数私有化
+ 全局统一访问，保证唯一性，所以是静态实例
+ 用于变量全局唯一，工厂对象的实现就是Singleton模式的一个实例

```cpp
//Singleton.h
class Singleton
{
public:
    static Singleton* Instance()；
private:
    Singleton();
    static Signleton* _instance;
}
//Singleton.cpp
Singleton* Singleton::_instance = 0;
Signleton::Singleton(){
    cout << "Singleton..." << endl;
}
static Singleton* Instance(){
    if(_instance == 0){
        _instance = new Singleton();
    }
    return _instance;
}
//main.cpp
Signleton *sgn = Signleton::Instance();
```

**2. 工厂模式(Factory)**
+ 提高Cohesion和降低Coupling，定义创建对象的接口，封装了对象的创建
+ 使得具体化类的工作延迟到了子类中，解决了需要知道实际子类名称的问题

具体来讲，在Factory中，声明一个创建对象的接口，将具体实现放在Factory的子类中实现.

可以看到在示例中，父类并不知道要具体实例化哪一个子类，因为实现被延迟到了子类内部。如果要实现多个ConcreteProduct的实例化，那么得在Factory中添加新的方法来处理。

解决方法是在子类中通过多态来实现(同名函数不同实现，需要新加子类，即新的工厂具体子类)，或者给FactoryMethod传递一个参数(作为模板参数)来决定创建哪一个具体的Product

+ 局限在于所有的类必须有共同的基类

**C++ Tips:** 在C++中，纯虚函数的声明只是代表这个类是一个抽象类，不能产生对象，但是仍然可以为这个类中的纯虚函数指定函数体，当将析构函数设置为纯虚函数时，必须给析构函数指定函数体，因为子类的析构函数调用链默认含有父类的析构函数

<img src="../../assets//Factory.png" style="zoom: 100%;" />

```cpp
//Factory.h
class Factory
{
public: 
    virtual ~Factory() = 0;
    virtual Product *CreateProduct() = 0;
protected:
    Factory();
}

class ConcreteFactory : public Factory
{
public:
    ConcreteFactory();
    ~ConcreteFactory();
    Product *CreateProduct();
}

//Factory.cpp
Factory::Factory(){
}
Factory::~Factory(){
}
ConcreteFactory::ConcreteFactory(){
    cout<<"ConcreteFactory..."<<endl;
}
ConcreteFactory::~ConcreteFactory(){
}
Product *ConcreteFactory::CreateProduct(){
    return new ConcreteProduct();
}

//Product.h
class Product
{
public: 
    virtual ~Product() = 0;
protected:
    Product();
}

class ConcreteProduct: public Product
{
public:
    ConcreteProduct();
    ~ConcreteProduct();
}

//Product.cpp
Product::Product(){
}
Product::~Product(){
}
ConcreteProduct::ConcreteProduct(){
    cout<<"ConcreteProduct..."<<endl;
}
ConcreteProduct::~ConcreteProduct(){
}
//main.cpp
Factory *fac = new ConcreteFactory();
Product *p = fac->CreateProduct();
```

**3. 抽象工厂模式(AbstractFactory)**
+ 用于创建一组相关或者依赖的对象
  
关键在于将这一组对象创建封装到一个用于创建对象的类中，维护这样一个创建类。
  
<img src="../../assets//AbstractFactory.png" style="zoom: 100%;" />

```cpp
代码实现见设计模式精解PDF：p13
简单来说就是，Factory作为纯虚父类，提供A和B对象的创建接口，每个具体子类实现对象创建的方法，子类1创建A1B1,子类2创建A2B2
```


**4. 建造器模式(Builder)**
 
+ 用于创建一个复杂对象，这个对象由多个部分组成，而且这些部分是有序的，创建对象的过程是稳定的，但是每一部分的创建是可变的

+ 关键在于Director类并不直接返回对象，而是Director类负责调用Builder的接口，将Builder的接口按照一定的顺序调用，最终构建出一个完整的对象
<img src="../../assets//Builder.png" style="zoom: 100%;" />

```cpp
#include <iostream>
#include <string>
using namespace std; 
 
class Player;
 
class playBuilder
{
    public:
        string Name;
        int ID;
        int HP;
        int Level;
    public:
        playBuilder() {    //参数默认值
            Name = "xiaoming";
            ID = 1001;
            HP = 100;
            Level = 1;
            cout<<"默认构造函数"<<endl;
        }
        playBuilder* SetName(string Name) {this->Name = Name;return this;}
        playBuilder* SetID(int ID) {this->ID = ID;return this;}
        playBuilder* SetHP(int HP) {this->HP = HP;return this;}
        playBuilder* SetLevel(int Level) {this->Level = Level;return this;}
        Player *build(); 
};
 
class Player {
    public:                 
        string Name;
        int ID;
        int HP;
        int Level;
    public:
        string GetName() {return Name;}
        int GetID() {return ID;}
        int GetHP() {return HP;}
        int GetLevel() {return Level;}
 
        Player() {cout<<"默认构造函数"<<endl;}
        Player (playBuilder const &temp) {
            this->Name = temp.Name;
            this->ID = temp.ID;
            this->HP = temp.HP;
            this->Level = temp.Level;
        }
        ~Player() {}
        void show() {
            cout<<Name<<"-"<<ID<<"-"<<HP<<"-"<<Level<<endl;
        }
};
 
Player *playBuilder::build(){
    return new Player(*this);
}
 
class Director {
    public:
        playBuilder *builder;
            Director() {
                builder = new playBuilder();
            }
            Player * construct(string Name,int ID,int HP,int Level) {   //构造规则
                builder = builder->SetName(Name);
                builder = builder->SetID(ID);
                builder = builder->SetHP(HP);
                builder = builder->SetLevel(Level);
                return builder->build();
            }
            ~ Director() {
                delete builder;
            }
};
 
int main() {
    Director d1;
    Player * p1 = d1.construct("xiaowang",999,9999,1);   //输入构建规则需要的参数
    Player * p2 = Director().construct("xiaosong",888,8888,8);
    Player * p3 = playBuilder().SetName("zhangsan")->SetID(92)->build();
    p1->show();
    p2->show();
    p3->show();
    return 0;
}
``` 

**5. Prototype模式**
+ 原型模式
+ 提供通过已存在对象进行新对象创建的接口(Clone)，C++中通过拷贝构造函数实现
+ Builder 模式重在复杂对象的一步步创建（并不直接返回对象），AbstractFactory 模式重在产生多个相互依赖类的对象，而 Prototype 模式重在从自身复制自己创建新类。 

```cpp
class Prototype
{
public:
    virtual Prototype *Clone() = 0;
    virtual ~Prototype();
private:
    Prototype();
}

class ConcretePrototype: public Prototype
{
public:
    ConcretePrototype();
    ConcretePrototype(const ConcretePrototype &cp);
    Prototype *Clone();
}

ConcretePrototype::ConcretePrototype(const ConcretePrototype &cp){
    cout << "ConcretePrototype copy..." << endl;
}
Prototype *ConcretePrototype::Clone(){
    return new ConcretePrototype(*this);
}
```

#### 结构型模式

所谓结构性模式，就是用于设计对象和类的结构，以便使他们能够更好的协同工作

**1. Bridge模式**

+ 用于抽象和具体实现的需求都需要更改的时候，将抽象和具体实现分离，使得两者可以独立的变化
+ 具体来讲，可以看到左侧是抽象部分，右侧是实现部分
<img src="../../assets//Bridge.png" style="zoom: 100%;" />

关于这个"实现"需要做出如下解释：
>在此处实现并不是指抽象基类的具体子类对基类中抽象方法的实现，而指的是如何去实现用户的需求。
在图中，RefinedAbstraction作为基类的子类，拥有AbstructionImp* _imp指针，这个指针指向了一个实现类，这个实现类是一个抽象类，它的子类ConcreteImpA和ConcreteImpB分别实现了这个实现类的接口，这样就可以在RefinedAbstraction中调用这个实现类的接口，实现了抽象和实现的分离。
例如我们要针对不同的操作系统分别实现，我们就可以增加一个RefinedAbstraction，但是操作仍然是AbstructionImp

所以Bridge其实就是抽象和实现之间的Bridge啦~

**2.Adapter模式**

+ 用于将一个类的接口转换成客户希望的另一个接口，使得原本由于接口不兼容而不能一起工作的那些类可以一起工作
+ 分为类模式和对象模式

<img src="../../assets//Adapter.png" style="zoom: 100%;" />

通过这张图可以发现，Adapter继承自target类，同时包含一个private的Adaptee类的对象指针，这样就可以在Adapter中的target函数调用Adaptee的具体实现，实现了target和Adaptee的适配.

**3. Decorator模式**

+ 用于动态的给一个对象添加一些额外的职责，就增加功能来说，Decorator模式比生成子类更为灵活。具体是通过组合的方式而不是继承来实现的
+ 可以让父类拥有较少的抽象接口，而子类可以通过Decorator来添加新的功能

<img src="../../assets//Decorator.png" style="zoom: 100%;" />

在图中，Decorator和ConcreteComponent都继承自Component，利用多态的思想，ConcreteDecorator中有一个Component的指针，从而在自己的Operation中可以调用ConcreteComponent的Operation，同时ConcreteDecorator为ConcreteComponent添加了新功能。

但是需要注意的是，此处是**Component**的指针，从而实现了多态，如果只是ConcreteComponent的对象，那么就无法实现多态，从而无法实现Decorator模式

**4. Composite模式**

+ 组合模式允许以相同的方式处理单个对象和对象的组合体
+ 用于把一组相似的对象当作一个单一的对象。组合模式依据树形结构来组合对象，用来表示部分以及整体层次

<img src="../../assets//Composite.png" style="zoom: 100%;" />


这里给出详细解释：当你的程序结构有类似树一样的层级关系时，例如文件系统，视图树，公司组织架构等等，如果我们需要以统一的方式来操作单个对象和由这些对象组成的组合对象，就需要使用Composite模式

Component
抽象类，定义统一的处理操作。

Leaf
叶子节点，即单个对象

Composite
组合对象，里面持有一个List\<Component\>。

+ 分为透明和安全两种方式，透明方式将所有对外操作都放在Component，叶子节点也得提供这些接口，虽然它实际上不支持这些操作。而安全方式只将叶子节点与组合对象同时提供的操作放在Component，即示例图中所示

**5. Flyweight模式**

+ 叫做享元模式
+ 允许使用对象共享来有效地支持**大量细粒度对象**
  
**示例场景**：比如在文档编辑器的设计过程中，我们如果为没有字母创建一个对象的话，系统可能会因为大量的对象而造成存储开销的浪费。例如一个字母“a”在文档中出现了100000 次，而实际上我们可以让这一万个字母“a”共享一个对象，当然因为在不同的位置可能字母“a”有不同的显示效果（例如字体和大小等设置不同），在这种情况我们可以为将
对象的状态分为“外部状态”和“内部状态”，将可以被共享（不会变化）的状态作为内部状态存储在对象中，而外部对象（例如上面提到的字体、大小等）我们可以在适当的时候将
外部对象作为**参数传递**给对象（例如在显示的时候，将字体、大小等信息传递给对象

<img src="../../assets//Flyweight.png" style="zoom: 100%;" />

图中，FlyweightFactory拥有一个管理、存储对象的仓库或者叫对象池(vector)，GetFlyweight（）消息会遍历对象池中的对象，如果已经存在则直接返回给 Client，否则创建一个新的对象返回给 Client。
  
**6.Facade模式**

+ 也叫做面板模式
+ 用于在高层对外提供一个简单的统一交互接口，掩藏子模块内部的细节，解耦了系统
+ 非常简单，如图所示

<img src="../../assets//Facade.png" style="zoom: 100%;" />

**7.Proxy模式**

+ 为其他对象提供一种代理以控制对这个对象的访问，可以在访问的同时增加其他操作
+ 实现了逻辑和实现的解耦

+ 代理模式的使用有如下几种情况：
  + 远程代理 ：为位于两个不同地址空间对象的访问提供了一种实现机制，可以将一些消耗资源较多的对象和操作移至性能更好的计算机上，提高系统的整体运行效率。
  + 虚拟代理：通过一个消耗资源较少的对象来代表一个消耗资源较多的对象，可以在一定程度上节省系统的运行开销。比如创建开销大的对象时候，比如显示一幅大的图片，我们将这个创建的过程交给代理去完成。
  + 缓冲代理：为某一个操作的结果提供临时的缓存存储空间，以便在后续使用中能够共享这些结果，优化系统性能，缩短执行时间。
  + 保护代理：可以控制对一个对象的访问权限，为不同用户提供不同级别的使用权限。
  + 智能引用：要为一个对象的访问（引用）提供一些额外的操作时可以使用。例如智能指针
  
<img src="../../assets//Proxy.png" style="zoom: 100%;" />

在示例图中，Proxy有一个Subject的对象指针，用ConcreteSubject初始化后，调用Proxy的方法的时候，就可以调用ConcreteSubject的对应方法

#### 行为型模式

行为型设计模式主要解决类或者对象之间互相通信的问题

**1.Template模式**

+ 简单来说，Template 模式实际上就是利用面向对象中多态的概念实现算法实现细节和高层接口的松耦合，所以是采取继承的方式实现的
+ 注意的是，原语操作(细节算法)定义为protected，而TemplateMethod定义为Public,在TemplateMethod里面调用原语操作从而实现真正的调用

<img src="../../assets//Template.png" style="zoom: 100%;" />

Template体现了DIP的原则，即高层模块调用底层模块的操作，底层模块实现高层模块的接口，这样控制权在父类（高层模块），低层模块反而要依赖高层模块。 

**2.Strategy模式**

+ 策略模式
+ 和Template模式同样解决了具体实现和抽象的解耦问题，但是它是通过组合的方式实现的，具体来讲。Strategy 模式将逻辑（算法）封装到一个类（Context）里面，通过组合的方式将具体算法的实现在组合对象中实现，再通过委托的方式将抽象接口的实现委托给组合对象实现

<img src="../../assets//Strategy.png" style="zoom: 100%;" />

在示例中，Context对象有一个DoAction接口，但是有一个Strategy类的对象指针，所以通过DoAction函数内调用Strategy的AlgorithmInterface接口，实现了具体实现和接口的分离

两种模式对比：Strategy模式有如下好处，首先，Strategy模式可以在运行时动态改变Context所使用的具体Strategy，而Template模式在编译时就已经确定了具体的实现，其次，封装性更好，被包含对象的内部细节对外是不可见的，组合对象和被组合对象之间的依赖性小。所以优先使用Strategy模式。

唯一的缺点在于，Strategy模式会导致系统中类的数目增加，但是这是可以接受的，因为这样可以更好的实现单一职责原则。

**3.State模式**

+ 状态模式
+ 解决的是FSM状态太多时容易出错，状态逻辑和动作实现没有分离的问题。
+ 当一个对象内在状态改变时允许改变其行为，这个对象看起来像是改变了其类。

<img src="../../assets//State.png" style="zoom: 100%;" />

在具体实现中，将 State 声明为 Context 的友元类（friend class），其作用是让 State 模式访问 Context 的 protected 接口 ChangeSate。这样，State 模式就可以在状态转换时调用 Context 的 ChangeState 接口，实现状态的转换。

同时Context中有一个State*的指针，它将依赖状态的各种Operation操作委托给不同的状态对象执行。

State 及其子类中的操作都将 Context*传入作为参数，其主要目的是 State 类可以通过这个指针调用 Context 中的方法


**4.Observer模式**

+ 观察者模式，也叫发布订阅模式
+ 定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都可以得到通知并自动更新。

<img src="../../assets//Observer.png" style="zoom: 100%;" />

在示例图中， Subject 提供依赖于它的观察者 Observer 的注册（Attach）和注销（Detach）操作，并且提供了使得依赖于它的所有观察者同步的操作（Notify）。

在实现中，Subject 维护一个 list 作为存储其所有观察者的容器，每当调用Notify操作就遍历list 中的Observer对象，并广播通知改变状态（调用Observer的Update操作），但是也可以由observer发起，因为Observer维护了一个Subject的指针




**5.Memento模式**

+ 备忘录模式
+ 在不破坏封装性的前提下，在某一时刻捕获一个对象的内部状态，并在该对象之外保存这个状态，这样以后就可以将该对象恢复到原先保存的状态

<img src="../../assets//Memento.png" style="zoom: 100%;" />

在示例图中，Originator是需要保存状态的对象，Memento是保存状态的对象，Memento 的接口都声明为 private，而将 Originator 声明为 Memento 的友元类，从而将Originator的状态保存在Memento类中。两者之中都有一个State变量来保存状态，Originator可以通过CreateMemento来创建一个Memento对象，通过RestoreToMomento来恢复状态

**6.Mediator模式**

+ 中介者模式，用于对象间的交互和通信
+ 将多对多的通信转化为一对多的通信

<img src="../../assets//Mediator.png" style="zoom: 100%;" />

在示例图中，每个Colleague对象都有一个Mediator的指针，维护一个Mediator对象，而AB之间的交互就可以通过ConcreteMediator的接口来实现，从而实现了对象之间的解耦，AB之间不用维护各自的引用。

**7.Command模式**

+ 命令模式
+ 将请求封装到一个对象中，并将请求的接收者存放到具体的ConcreteCommand中，实现了调用操作的对象和操作的具体实现解耦

<img src="../../assets//Command.png" style="zoom: 100%;" />

在示例图中，Command是一个抽象类，定义了一个Execute的接口，ConcreteCommand继承自Command，实现了Execute的接口，同时维护了一个Receiver的指针。

Command
是一个接口，定义一个命令

ConcreteCommand
具体的执行命令，他们需要实现Command接口

Receiver
真正执行命令的角色，那些具体的命令引用它，让它完成命令的执行

Invoker
维护一组ConcreteCommand，负责按照客户端的指令设置并执行命令(命令的调用者)，像命令的撤销，日志的记录等功能都要在此类中完成

除了使用Invoker来主动激活命令，也可以利用Callback的思想，将操作方法的地址通过参数传递给Command对象，需要利用类成员函数指针`typedef void (Receiver::*Action)()`来实现

**8.Visitor模式**

+ 访问者模式
+ 将所有的更新封装到一个类中，并由待更改类提供一个接收接口

<img src="../../assets//Visitor.png" style="zoom: 100%;" />

Visitor 模式的关键是双分派（Double-Dispatch）的技术,即执行的操作将取决于请求的种类和接收者的类型。具体调用哪一个具体的 Accept （）操作，有两个决定因素：
+ 1.Element 的类型。因为 Accept（）是多态的操作，需要具体的 Element 类型的子类才可以决定到底调用哪一个 Accept（）实现；
+ 2.Visitor 的类型。Accept（）操作有一个参数（Visitor* vis），要决定了实际传进来的 Visitor 的实际类别才可以决定具体是调用哪个 VisitConcrete（）实现

在示例图中，Visit方法通过两个函数来具体实现，但是可以通过函数重载来实现Visit方法，根据参数不同来调用不同的Visit函数。另外，也可以利用C++的RTTI，通过dynamic_cast来判断Element的具体类型，再调用对应的Visit函数

```cpp
if (dynamic_cast<ConcreteElementA*>(elem)) {
    // 如果是 ConcreteElementA，执行相应操作
    ConcreteElementA* a = dynamic_cast<ConcreteElementA*>(elem);
    a->accept();
} else if (dynamic_cast<ConcreteElementB*>(elem)) {
    // 如果是 ConcreteElementB，执行相应操作
    ConcreteElementB* b = dynamic_cast<ConcreteElementB*>(elem);
    b->accept();
} ~~~~
```


**9.Chain of Responsibility模式**

+ 责任链模式
+ 避免请求发送者与接收者耦合在一起，让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止
+ 通常用于一个请求需要被多个对象中的某一个处理，但是具体对象需要在运行时根据条件决定

<img src="../../assets//Chain.png" style="zoom: 100%;" />

在示例图中，Handler是一个抽象类，定义了一个HandleRequest的接口，ConcreteHandler继承自Handler，实现了HandleRequest的接口，同时维护了一个Successor的指针，记录后继对象。在HandleRequest中，如果自己可以处理请求，那么就处理，否则交给后继对象处理。

降低了系统的耦合性

**10.Iterator模式**

+ 迭代器模式
+ 用来解决一个聚合对象的遍历问题，将对聚合的遍历封装到一个对象中，参见C++中的Iterator

<img src="../../assets//Iterator.png" style="zoom: 100%;" />

感觉没啥好说的，ConcreteAggregate有一个对象数组，保存了所有对象，而ConcreteIterator维护了一个Aggregate的指针，通过Next和IsDone来遍历ConcreteAggregate中的对象


**11.Interpreter模式**

+ 解释器模式
+ 用于定义一个语言的文法，并且建立一个解释器来解释该语言中的句子。用于为用户提供一个实现语法解释器的框架
+ 
<img src="../../assets//Interpreter.png" style="zoom: 100%;" />


