##一些tip
- 编译单元(translation unit)：一个单独的文件，例如cpp
- 使用sizeof(数据类型)可以得到对应数据类型的大小(以字节为单位)
- `#pragma once`：被称为 **头文件保护符（"header guard"）**，防止我们将单一头文件多次include到单一编译单元中；或者多个文件会include一个头文件，然后最后的main文件又会将这些文件全部include导致出现多次include；
  - 有另外一种头文件保护符，如下所示:
    - 首先判断_LOG_H是否已经被包含在里面，若没有则复制下面的代码；否则直接跳过；
    - 然后，将该文件定义为LOG_H
![img](./CPP_image/concept_pragma_once.png)
- #include的文件如果有.h后缀则属于c标准库，若没有则属于c++
- `else if`其实并不是关键字，而是以下的缩写（但还是会有点问题，比如再往后写else就没有对应的if，这个可以在该else中嵌套else 也许能解决）
```
else
{
    if() {}
}
```
- 下列三个均为筛选器，而不是真正的文件夹，是以虚拟文件夹的形式来组织代码
![img](./CPP_image/VS_fliter.png)

- 由于VS本身会将编译时生成的文件放到项目文件夹下的Debug文件夹中；而最终生成的exe文件则会放到与.sln同级的Debug文件夹中，查看比较麻烦，可以在下面的设置中更改路径存到指定的文件夹中，并可以区别不同平台以及Debug模式或Release模式
![img](./CPP_image/VS_file_folder.png)

## 预处理
- 包含`#include, #define, #if, #endif`，其本质都是将对应的文件内容复制到对应位置；
例如：有两个文件`EnBranch.h`和`Math.cpp`
- `#if`可以让我们根据特定条件包含或剔除代码
```
//EnBranch.h
}

//Math.cpp
#define INTEGER x

#if 0           //为0时，则这一部分代码不会进行编译，为1则正常编译
INTEGER multiply(int a, int b) //生成的预处理文件.i中就会把INTE替换成x
{
    INTEGER result = a * b;
    return result;
#include"EnBranch.h"    //此处就是把EnBranch.h的内容复制过来，即'}'
#endif
```
- 修改如下设置便可以生成.i文件(生成了.i文件后边不后悔生成.obj文件 **(.obj文件内容是01机器码)**；.i文件如下图二和三所示)
![img](./CPP_image/compile_file_i.png)
![img](./CPP_image/compile_include.png)
![img](./CPP_image/complie_define.png)
![img](./CPP_image/compile_if_endif.png)
- 修改如下设置，获得.asm文件得到汇编代码，默认debug模式下会插入很多代码便于进行debug，但是会延长程序运行时间
![img](./CPP_image/compile_汇编.png)
- 在debug模式下设置为速度最优策略，会得到简洁的汇编代码，但是编译时会发生错误，需要进行相应设置(下图二)
![img](./CPP_image/compile_优化速度.png)
![img](./CPP_image/compile_修改.png)

##链接(Linking)
- 每一个cpp文件在编译后都会生成.obj文件，然后将他们链接成.exe文件
- 每一个.exe文件必须要有一个入口函数，这个函数不一定非要是main函数，可以自定义，如下所示 **（入口点）**：
![img](./CPP_image/Link_entry.png)
- **未解析的外部符号(unresolved external symbol):**
  
  `Log.cpp`中的Log函数被更换为Logr函数，导致Linker在进行链接时无法找到`Math.cpp`中`Multiply`函数调用的`Log`函数，从而导致链接失败
  - 如果在`Multiply`前面加上`static`，哪怕main函数中并没有调用该函数，仍然会报错；这是因为虽然在此处并没有被调用，在其他文件中也存在被调用的可能，所以Linker仍然会进行链接
  - 而加入`static`之后， 该函数只能够在`Math.cpp`文件中被调用，但是实际并没有被调用，所以不会报错
![img](./CPP_image/Link_error.png)

- **重复符号报错**：

    有相同的函数，相同返回值，相同参数列表；Linker就不知道该链接到哪个函数上，分为两种情况：
- 在同一个编译单元中存在完全相同的函数、返回值和参数列表，此时，编译器会直接帮我们检测出来
![img](./CPP_image/Link_compile.png)
- 如果在不同编译单元有相同的函数、返回值和参数，此时，编译器无法查出错误，会被正确编译。但是当链接时，则会报错
![img](./CPP_image/Link_link_error.png)
- 或者，函数定义被写在`Log.h`中，然后存在多个文件同时调用该头文件。而`#inlcude`的逻辑就是把头文件中的代码全部复制到对应文件中，则会导致函数在两个文件中被同时定义，情况就如上面那个错误相同，导致链接失败
  - 一种解决方法是，在头文件中对应函数定义加上修饰词`static`，此时在不同文件中创建的该函数仅自己文件可见，例如`Math.cpp`和`Log.cpp`同时`#include"Log.h"`，则会同时生成两个Log函数，但是这两个Log函数对对方cpp文件是不可见的。在`Math.cpp`中的Log函数，`Log.cpp`是不可见的，就不会导致重复定义。
  - 另一种是增加修饰词`inline`，当cpp文件包含该头文件时，只会将函数本体拿过去替换，而不是定义一个函数
```
//Log.h
void Log(const char* message)
{
    std::cout << message << std::endl;
}

//Math.cpp
#include "Log.h"
int Multlpy (int a, int b)
{
    //Log("Multlpy")
    std:: cout << "Multlpy" << std::endl;
    return a* b;
}
```
- 第三种解决方法，在头文件中仅放函数声明，在cpp文件中写函数定义
```
//Log.h
void Log(const char* message);

//Log.cpp
#include"Log.h"
void Log(const char* message)
{
    std::cout<< message << std::endl;
}

//Math.cpp
#include"Log.h"
int Multlpy(int a, int b)
{
    Log("Multlpy);
    return a * b;
}
```

##指针(pointer)
- 原始指针(raw pointer):
  - 指针其实就是一个无符号整数，代表一个内存地址，前面的类型只是表明该内存地址中存储的数据的类型，与指针本身类型无关。
  - **`void* ptr = 0`**：0是地址，但是它不能够写入和读取，只是为了声明该指针为空，void代表完全没有类型，只是为了在进行写入和读取时告诉编译器需要申请多大的空间
  - `&变量`：取该变量的内存地址
```
int value = 8;
void* ptr = &value;
*ptr = 10;  //会报错，因为编译器不知道要申请多少空间来存储10
            //这个数字;只需要将上面的代码改为int* ptr = &value
            //即可正常写入

char* buffer = new char[8]; //向堆申请了8个字节空间，
                            //并将该空间的起始位置对应
                            //的地址赋给buffer
memset(buffer, 0, 8);   //用0去填充申请的空间
delete[] buffer;        //由于是从堆上申请的，需要手动释放
                        //因为申请的是数组，需要对应的delete[]
```

##引用(reference)
- 与指针不同，引用必须引用一个已经存在的变量，本身不是变量不会占用内存空间，相当与给被引用对象创建了别名

##类(class)与结构体(struct)的区别
- c++保留struct是为了对c有一定的兼容性
- 类默认私有，结构体默认公有
- 具体什么情况使用什么，根据个人观点而定。一般来说，struct用来存储一些简单的变量以及简单的方法，而class则实现一些较为复杂的东西以，继承用class更好

##Static
- 在类或结构体的外面，则代表该类或结构体在link阶段是局部的，只对定义它的编译单元可见
```
//Static.cpp
static int value = 5;

//Main.cpp
int value = 10; //编译通过，
                //因为上面的value仅Static.obj文件可见
                //若上面没有static修饰词，则编译失败，
                //会提示link失败，因为该变量被定义了两次

extern int value;   //上面没有static修饰词，该句代码会让编译器从其他编译单元里找到value的定义(external linking)，如果上面有static修饰词，则unresolved external symbol 链接失败

```
- 在类或结构体内部，代表这部分内容是所有实例所共享的
  - 静态方法和静态变量没有对应的实例，所以静态方法无法调用非静态变量
- 静态局部变量：
  - 假设函数中存在一个静态局部变量，在第一次调用该函数时会初始化为0，然后在每次调用该函数后进行自增操作，则会一直增加；若不是static，则每次会进行初始化并自增
```
void function()
{
    static int value = 0;   //此时主函数会打印1,2,3,4,5
                            //若没static，则会打印5次1
                            //相较于直接将其设置为全局变量
                            //这么设置的value只有才function
                            //中才会进行操作
                            //其他函数无法访问value；
    value++;
    std::cout << value << std::endl;
}

int main()
{
    for(int i = 0; i < 5; ++i)
    {
        function();
    }
}
```
##单例模式
仅创建一个实例，并其生命周期为整个程序
```
class Singleton
{
private:
    static Singleton* s_Instance;
public:
    static Singleton& Get()
    {
        return *s_Instance;
    }

    void Hello();
};

Singleton* Siongleton::s_Instance = nullptr;

int main()
{
    Singleton::Get().Hello();
}

//简化版
class Singleton
{
public:
    static Singleton& Get()
    {
        static Singleton instance;  //将其生命周期延长至程序结束
                                    //第一次调用会创建实例；之后不会
        return instance;            //返回存在的唯一的一个实例
    }

    void Hello();
};

int main()
{
    Singleton::Get().Hello();
}
```