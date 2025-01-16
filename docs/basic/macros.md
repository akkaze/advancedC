# 重要的宏
## offsetof
offsetof 是一个宏，通常在 C 和 C++ 编程中用来计算一个结构体成员相对于结构体起始位置的字节偏移量。这个宏在 <stddef.h> 或者 <cstddef> 头文件中定义。它主要用于低级系统编程和序列化操作，通过计算地址偏移，可以更灵活地操作结构体中的成员。

语法
```c
#include <stddef.h>

offsetof(type, member)
```
- ```type```: 结构体类型。
- ```member```: 需要计算偏移量的结构体成员的名字。
### offsetof的实现
#### offsetof在gcc下的实现
```c
#define offsetof(type, member) ((size_t) &((type *)0)->member)
```
这个定义直接通过解引用空指针来访问成员变量m的地址，然后计算偏移量。由于C语言允许这种操作，这种实现方式在实际应用中可以正常工作，但存在潜在的风险，因为对空指针的解引用是不安全的‌
#### offsetof在msvc下的实现
```c
#define offsetof(s,m) ((size_t)&reinterpret_cast<char const volatile&>((((s*)0)->m)))
```
在ANSI C中, ```&((st *)(0))->m```不会对```0(NULL)```真正的进行解引用, 而会直接返回m的地址, 这样就避免了对NULL解引用会造成的段错误. 而在c++中, 并没有这样的规则, 所以如果想ANSI C这样实现```offsetof```会引起段错误. 也知道char*是各种type中standard里唯一保证过sizeof是1, 其它都是implemention dependent.

使用volatile关键字是为了防止编译器优化掉对空指针的解引用操作，确保在编译时能够正确计算偏移量‌
### offsetof的优缺点
- ‌优点‌：能够在编译时计算出结构体成员的偏移量，提高程序的效率和准确性。
- ‌缺点‌：对空指针的解引用存在不安全因素，可能导致程序崩溃。在C++中直接使用可能会导致未定义行为。
### offsetof的使用
```c
#include <stddef.h>
 
#ifndef offsetof
#define offsetof(type, member) ((size_t) &((type *)0)->member)
#endif
 
// 示例使用
struct MyStruct {
    int a;
    char b;
    double c;
};
 
int main() {
    size_t offset_of_a = offsetof(struct MyStruct, a);
    size_t offset_of_b = offsetof(struct MyStruct, b);
    size_t offset_of_c = offsetof(struct MyStruct, c);
 
    printf("Offset of 'a' is: %zu\n", offset_of_a);
    printf("Offset of 'b' is: %zu\n", offset_of_b);
    printf("Offset of 'c' is: %zu\n", offset_of_c);
 
    return 0;
}
```
## container_of
```‌container_of‌```是一个在Linux内核中广泛使用的宏定义，主要用于通过结构体成员的地址来获取该结构体的地址。其定义如下：
### container_of的实现
```c
#define container_of(ptr, type, member) ({ \
    void *__mptr = (void *)(ptr); \
    ((type *)(__mptr - offsetof(type, member))); \
})
```
使用方法
container_of宏接受三个参数：

- ```ptr```：指向结构体成员的指针。
- ```type```：该成员所在的结构体的类型。
- ```member```：结构体中成员的名称。

实现原理

1. ‌类型检查‌：首先，通过__same_type宏检查ptr指向的类型是否与type类型匹配，确保类型安全。
2. ‌计算偏移量‌：使用offsetof宏计算成员member在结构体中的偏移量。
3. ‌指针转换‌：最后，通过减去偏移量来计算结构体的地址，并将其转换为正确的类型。
### container_of的示例
假设有一个结构体struct foo，其中包含一个成员变量bar，可以通过以下方式使用container_of：
```c
struct foo {
    int bar;
    // 其他成员...
};

struct foo *foo_ptr = ...; // 假设已经获取到foo_ptr的地址
int *bar_ptr = &foo_ptr->bar; // 获取bar的地址
struct foo *foo_struct = container_of(bar_ptr, struct foo, bar); // 通过bar_ptr获取foo_struct的地址
```