## C++

### enum类型

#### enum是什么

- ​	`enum常量`是int类型
- ​	因此只要能使用int类型的地方，就可以使用enum类型

#### 为什么要使用enum

- 目的是为了提高程序的可读性

#### 声明，默认值，赋值

`注意`：第一个枚举成员的默认值为整型的 0，后续枚举成员的值在前一个成员上加 1。我们在这个实例中把第一个枚举成员的值定义为 1，第二个就为 2，以此类推。

```c++
enum DAY
{
      MON=1, 
      TUE, 
      WED, 
      THU, 
      FRI, 
      SAT, 
      SUN
};
enum season 
{
    spring, 
    summer=3, 
    autumn, 
    winter
};
```

#### 定义

三种方式：

- **定义枚举类型，再定义枚举变量**

```c++
enum DAY
{
      MON=1, TUE, WED, THU, FRI, SAT, SUN
};
enum DAY day;
```

- **定义枚举类型的同时定义枚举变量**

```c++
enum DAY
{
      MON=1, TUE, WED, THU, FRI, SAT, SUN
} day;
```

- **省略枚举名称，直接定义枚举变量**

```c++
enum
{
      MON=1, TUE, WED, THU, FRI, SAT, SUN
} day;
```



###  inline函数

#### **`inline` 函数的作用**

- **最初目的**：向编译器建议将函数代码直接插入调用处（避免函数调用的开销），以提高效率（例如短小的工具函数）。
- **现代编译器**：会自主决定是否内联（如循环、递归通常不会内联），`inline` 关键字不再强制要求编译器内联，**更多用于满足编程规范**。

####  **关键用途：ODR（单一定义规则）**

- **头文件中的函数定义**：若在头文件中直接定义函数（非声明），多个源文件包含该头文件时，会导致**重复定义错误**。
- **使用 `inline`**：在头文件的函数定义前加 `inline`，告知编译器“允许重复定义，但需保证所有定义相同”，从而解决 ODR 问题。

####  **注意事项**

- **编译器决定权**：`inline` 只是建议，编译器可能忽略（如复杂函数）。
- **ODR 一致性**：所有源文件中的 `inline` 函数定义必须**完全相同**，否则引发未定义行为。
- **过度内联**：可能导致代码膨胀，降低缓存利用率，反而影响性能。

#### 总结：何时使用 `inline`？

1. **头文件中定义函数** → 必须用 `inline`（解决 ODR）。
2. **短小频繁调用的函数** → 可尝试用 `inline`（但编译器可能自行优化）。
3. **避免滥用** → 性能关键处由性能分析工具指导。

### **explicit **

#### **`explicit` 的作用**
`explicit` 用于修饰类的 **构造函数** 或 **类型转换运算符**（C++11 起），禁止编译器进行隐式的类型转换，要求代码中必须显式调用。

---

##### **1. 用于构造函数**
###### **问题背景：隐式转换的风险**
如果一个类的构造函数只有一个参数（或多个参数但具有默认值），编译器可能自动进行隐式转换，导致意外的行为。

**示例：未使用 `explicit`**

```cpp
class MyString {
public:
    MyString(const char* str) {  // 允许隐式转换：const char* → MyString
        // 构造逻辑...
    }
};

void printString(const MyString& s) {
    // 打印字符串...
}

int main() {
    printString("Hello");  // 隐式转换：const char* → MyString
    return 0;
}
```
这里 `printString("Hello")` 会隐式调用 `MyString` 的构造函数，但可能并非程序员本意。

---

###### **使用 `explicit` 禁止隐式转换**
```cpp
class MyString {
public:
    explicit MyString(const char* str) {  // 必须显式构造
        // 构造逻辑...
    }
};

void printString(const MyString& s) {
    // 打印字符串...
}

int main() {
    // printString("Hello");          // 错误：无法隐式转换
    printString(MyString("Hello"));  // 正确：显式构造
    return 0;
}
```

---

#### **2. 用于转换运算符（C++11 起）**
防止类型转换运算符被隐式调用。

**示例：未使用 `explicit`**
```cpp
class BooleanWrapper {
public:
    operator bool() const {  // 隐式转换为 bool
        return true;
    }
};

int main() {
    BooleanWrapper bw;
    if (bw) {         // 合法：隐式转换为 bool
        // ...
    }
    int x = bw;       // 可能意外的行为：隐式转换为 bool，再转为 int
    return 0;
}
```

**使用 `explicit` 禁止隐式转换：**
```cpp
class BooleanWrapper {
public:
    explicit operator bool() const {  // 必须显式转换
        return true;
    }
};

int main() {
    BooleanWrapper bw;
    if (bw) {               // 错误：无法隐式转换
    if (static_cast<bool>(bw)) {  // 正确：显式转换
        // ...
    }
    // int x = bw;          // 错误：无法隐式转换
    return 0;
}
```

---

#### **3. 使用场景**
- **明确语义**：避免构造函数或类型转换的隐式行为导致歧义。
  ```cpp
  class Timer {
  public:
      explicit Timer(int seconds) {}  // 明确要求显式的时间单位
  };
  
  Timer t1(5);     // 正确：显式构造
  // Timer t2 = 5; // 错误：必须显式调用
  ```
- **防止意外重载解析**：
  ```cpp
  void process(int x);
  void process(const MyString& s);
  
  process(10);     // 调用 process(int)
  // process("Hi"); // 若未用 explicit，可能隐式调用 MyString 构造函数，再调用 process(const MyString&)
  ```

---

#### **4. 注意事项**
- **多参数构造函数**：C++11 起，即使构造函数有多个参数，也可用 `explicit`。
  ```cpp
  class Vec3 {
  public:
      explicit Vec3(int x, int y, int z) {}
  };
  
  Vec3 v = {1, 2, 3};  // 错误：无法隐式转换（需显式构造）
  ```
- **拷贝构造函数**：极少需要 `explicit`，通常拷贝操作是安全的。
- **STL 中的例子**：`std::vector` 的构造函数 `explicit vector(size_type count)`，避免 `vector<int> v = 10;` 这种歧义代码。

---

#### **总结**
- **用 `explicit`**：当隐式转换可能导致歧义或安全隐患时。
- **不用 `explicit`**：当隐式转换是合理且符合设计意图时（如 `std::string` 允许从 `const char*` 隐式构造）。



### 纯虚函数
C++中 纯虚函数的语法格式为：

> virtual 返回值类型 函数名 (函数参数) = 0;

`包含纯虚函数的类称为抽象类`

抽象类通常是作为基类，让派生类去实现纯虚函数。派生类必须实现纯虚函数才能被实例化。

**注意**

- 类A中只要有一个纯虚函数，那么这个类就是抽象类，但是类A中可以有普通函数和成员
- 继承抽象类时，只有实现其父类的所有纯虚函数，子类才能是普通类，否则还是抽象类，不能实例化
- 只有虚函数才能被声明为纯虚函数，普通的成员函数不能被声明为纯虚函数

### 强制类型转换

#### static_cast

> `static_cast` 是 C++ 中的一个类型转换运算符，用于在编译时执行类型转换。它比 C 风格的强制转换 (`(type)value`) 更安全，并且能够在一定程度上防止错误的类型转换。
>
> ------
>
> ## **`static_cast` 的用法**
>
> ```
> static_cast<T>(expression)
> ```
>
> 它主要用于以下几种情况：
>
> ### **1. 基本数据类型之间的转换**
>
> `static_cast` 可以用于将 `int`、`double`、`char` 等基本数据类型相互转换。
>
> ```cpp
> #include <iostream>
> 
> int main() {
>     double d = 3.14;
>     int i = static_cast<int>(d);  // 转换为整数，去掉小数部分
>     std::cout << "i = " << i << std::endl;
> 
>     char c = static_cast<char>(65);  // 转换为 ASCII 字符
>     std::cout << "c = " << c << std::endl;
> 
>     return 0;
> }
> ```
>
> **输出：**
>
> ```
> i = 3
> c = A
> ```
>
> ------
>
> ### **2. 指针类型的转换（父子类之间的转换）**
>
> 在类继承关系中，`static_cast` 可用于 **安全地** 在**父类和子类之间转换指针或引用**（只能用于从子类转换到父类，或显式地从父类转换回子类）。
>
> ```cpp
> #include <iostream>
> 
> class Base {
> public:
>     virtual void show() { std::cout << "Base class" << std::endl; }
> };
> 
> class Derived : public Base {
> public:
>     void show() override { std::cout << "Derived class" << std::endl; }
> };
> 
> int main() {
>     Derived d;
>     Base* basePtr = static_cast<Base*>(&d);  // 子类 -> 父类（安全）
>     basePtr->show();
> 
>     Derived* derivedPtr = static_cast<Derived*>(basePtr);  // 父类 -> 子类（需确保 basePtr 实际指向 Derived）
>     derivedPtr->show();
> 
>     return 0;
> }
> ```
>
> **输出：**
>
> ```
> Derived class
> Derived class
> ```
>
> **注意**：
>
> - **子类 → 父类** 是安全的（向上转换）。
> - **父类 → 子类** 需要确保 `Base*` 实际上指向的是 `Derived` 对象，否则行为未定义（UB）。
>
> ------
>
> ### **3. `void\*` 转换回原来的类型**
>
> C++ 允许 `void*` 指针存储任意类型的指针，`static_cast` 可以用于将其转换回原来的类型。
>
> ```cpp
> #include <iostream>
> 
> int main() {
>     int a = 10;
>     void* ptr = &a;  // void* 可以存储任何类型的指针
> 
>     // 需要转换回 int*
>     int* intPtr = static_cast<int*>(ptr);
>     std::cout << "Value: " << *intPtr << std::endl;  // 输出 10
> 
>     return 0;
> }
> ```
>
> **输出：**
>
> ```
> Value: 10
> ```
>
> ------
>
> ### **4. 枚举类型与整数之间的转换**
>
> `static_cast` 允许枚举类型和整数之间进行转换。
>
> ```cpp
> #include <iostream>
> 
> enum Color { RED = 1, GREEN, BLUE };
> 
> int main() {
>     Color c = GREEN;
>     int num = static_cast<int>(c);  // 枚举转换为 int
>     std::cout << "Enum as int: " << num << std::endl;
> 
>     Color c2 = static_cast<Color>(2);  // int 转换回枚举
>     std::cout << "Converted Enum: " << c2 << std::endl;
> 
>     return 0;
> }
> ```
>
> **输出：**
>
> ```
> Enum as int: 2
> Converted Enum: 2
> ```
>
> ------
>
> ## **`static_cast` VS 其他转换方式**
>
> | 转换方式           | 适用场景                                   | 是否检查       | 是否安全                         |
> | ------------------ | ------------------------------------------ | -------------- | -------------------------------- |
> | `static_cast`      | 基本类型转换、类继承指针转换               | 编译时检查     | 安全（前提是逻辑正确）           |
> | `reinterpret_cast` | 任意类型指针转换（例如 `int*` 转 `char*`） | 无检查         | **极不安全，可能导致未定义行为** |
> | `dynamic_cast`     | 用于**多态**类型的**安全向下转换**         | 运行时类型检查 | 安全（若转换失败返回 `nullptr`） |
> | `const_cast`       | 移除 `const` 或 `volatile` 修饰符          | 编译时检查     | 需谨慎使用，可能导致未定义行为   |
>
> ------
>
> ## **总结**
>
> - `static_cast` 适用于**编译时可确定**的转换，如**基本类型转换、父子类转换（非多态）**、`void*` 转换等。
> - `static_cast` **不进行运行时检查**，因此如果用于 `Base*` 转换回 `Derived*`，需要确保对象实际是 `Derived` 类型，否则会有**未定义行为（UB）**。
> - **避免使用 `static_cast` 进行不安全的指针转换**（例如 `int*` 转 `char*`，应使用 `reinterpret_cast`）。
>
> 如果你的转换涉及多态类，并且需要**运行时类型检查**，应使用 `dynamic_cast`。

## Qt

### connect信号槽简介

信号槽是 Qt 框架引以为豪的机制之一。当用户触发某个事件时，就会发出一个信号（signal），这种发出是没有目的的，类似广播。如果有对象对这个信号感兴趣，它就会连接（connect）绑定一个函数（称为槽slot）来处理这个信号。也就是说当信号发出时，被连接的槽函数会自动被回调。这有点类似与开发模式中的[观察者模式](https://so.csdn.net/so/search?q=观察者模式&spm=1001.2101.3001.7020)，即当发生了感兴趣的事件，某一个操作就会被自动触发

信号和槽是Qt特有的信息传输机制，是Qt设计程序的重要基础，它可以让互不干扰的对象建立一种联系。槽的本质是类的成员函数，其参数可以是任意类型的。和普通C++成员函数几乎没有区别，它可以是虚函数，也可以被重载。可以是公有的、保护的、私有的、也可以被其他C++成员函数调用。唯一区别的是：槽可以与信号连接在一起，每当和槽连接的信号被发射的时候，就会调用这个槽

### 连接信号槽 connect 函数的第五个参数

connect 函数原型如下：
`[static] QMetaObject::Connection QObject::`

`connect(const QObject *sender, const char *signal, `

​			  `const QObject *receiver, `

​			  `const char *method, `

​             `Qt::ConnectionType type = Qt::AutoConnection)`

ConnectionType 是一个定义在 Qt namespace 中的一个枚举，具体内容如下：

```c
enum ConnectionType {
	AutoConnection,
	DirectConnection,
	QueuedConnection,
	BlockingQueuedConnection,
	UniqueConnection =  0x80
};
```

1. **Qt::AutoConnection**：默认值。根据 sender 和 receiver 所处线程在信号发出时作出判断。如果在同一线程则使用 Qt::DirectConnection 连接，否则使用 Qt :: QueuedConnection 连接。需要注意的是，这个判断和 sender 对象所处线程无关，真正判断的是发出信号这个动作所在的线程
2. **Qt::DirectConnection**：槽函数会在信号发送的时候直接被调用，槽函数运行于信号发送者所在线程。效果看上去就像是直接在信号发送位置调用了槽函数。需要注意的是，在多线程环境下比较危险，可能会造成奔溃
3. **Qt::QueuedConnection**：槽函数在控制回到接收者所在线程的事件循环时被调用，槽函数运行于信号接收者所在线程。发送信号之后，槽函数不会立刻被调用，等到接收者的当前函数执行完，进入事件循环之后，槽函数才会被调用。多线程环境下一般用这个
4. **Qt::BlockingQueuedConnection**：槽函数的调用时机与 Qt::QueuedConnection 一致，不过发送完信号后发送者所在线程会阻塞，直到槽函数运行完，在多线程间需要同步的场合可能需要这个。需要注意的是，接收者和发送者绝对不能在一个线程，否则程序会死锁
5. **Qt::UniqueConnection**：这个 flag 可以通过按位或（|）与以上四个结合在一起使用。当这个flag设置时，当某个信号和槽已经连接时，再进行重复的连接就会失败，也就是避免了重复连接

### 信号与槽的连接方式

1. **C++ 连接信号槽 - Qt4 语法**
   `connect(ui->pushButton, SIGNAL(clicked()), this, SLOT(close()));`
2. **C++ 连接信号槽 - Qt5 语法**
   `connect(ui->pushButton, &QPushButton::clicked, this, &MainWindow::close)`
3. **C++ 连接信号槽 - 函数指针**
   `void(MainWindow:: *buttonClickSlot)() = &MainWindow::onButtonPushed;`
   `connect(ui->pushButton, &QPushButton::clicked, this, buttonClickSlot);`
4. **C++ 连接信号槽 - Lambda 表达式**
   `connect(ui->pushButton, &QPushButton::clicked, this, [=](){ this->close(); });`
5. **C++ 信号连接 QML 的槽**

```c
class Test {
signals:
	void sendData(QString str);    
}
```

1）如果注册的是全局对象，则需要使用 Connections 连接：

```c
Connections {
    target: test
    onSendData: {
        console.log(str)
    }
}
```

2）如果注册的是类，则需要先实例化对象，之后直接使用 on 接收：

```
Test {
	onSendData: {
        console.log(str)
    }
}
```

1. **QML 信号连接 C++ 的槽**

```c++
#include <QQuickItem>
QObject *quitButton = root->findChild<QObject*>("quitButton");
if (quitButton) {
    QObject::connect(quitButton, SIGNAL(clicked()), &app, SLOT(quit()));
}
```

1. **C++ 调用 QML 函数**

```c++
QObject *changeBtn = root->findChild<QObject*>("objectName");
if (changeBtn)
{
    QMetaObject::invokeMethod(changeBtn, "changeColor");
}
```

1. **QML 调用 C++ 函数**

```c++
onClicked:
{
    className.test();
}
```

1. **QML 信号连接 QML 的槽**

```c
// A.qml
Rectangle {
	signal sendData(var data)
}
1234
// B.qml
Rectangle {
	onSendData: console.log(data)
}
```

### Qt的内存管理机制

![image-20241016191611681](interview.assets/image-20241016191611681.png)

>**"c++是爹先生，儿后生"**
>
>**儿先死，爹后死**

![image-20241016191636234](interview.assets/image-20241016191636234.png)



## Linux



## 设计模式



## STL





