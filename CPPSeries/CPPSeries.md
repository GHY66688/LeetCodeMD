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
![img](./CPP_image/compile_include.png)
![img](./CPP_image/complie_define.png)
![img](./CPP_image/compile_if_endif.png)
- 获得.asm文件得到汇编代码，默认debug模式下会插入很多代码便于进行debug，但是会延长程序运行时间
![img](./CPP_image/compile_汇编.png)
- 在debug模式下设置为速度最优策略，会得到简洁的汇编代码，但是编译时会发生错误，需要进行相应设置(下图二)
![img](./CPP_image/compile_优化速度.png)
![img](./CPP_image/compile_修改.png)