# 测试驱动
## 代码覆盖率
## stub和mock
C语言stub，也即C语言打桩，是用技巧使得调用某个函数```func```的时候实际上调用的是```func_stub```
通常有3种实现方式
- 编译时打桩
- 链接时打桩
- 运行时打桩
下面介绍的是运行时打桩
```c
#include "stub.h"
#include <stdio.h>
void add(int i)
{
    printf("add(%d)\n",i);
}
 
void add_stub(int i)
{
    printf("add_stub(%d)\n",i);
}
 
int main()
{
    INSTALL_STUB(add,add_stub);
    add(12);
    REMOVE_STUB(add_stub);
    add(11);
    return 0;
}　
```
## 单元测试
## 集成测试
## 系统测试
## 模糊测试