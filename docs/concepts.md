# 重要概念
## 声明和定义
在C语言中，声明和定义是两个不同的概念。声明通常用来告诉编译器有这么一个变量或者函数的存在，而定义则是创建这个变量或者函数。
声明可以出现多次，而定义只能有一次。
声明通常放在头文件里，作为接口文件使用。
### 声明有关概念
#### 存储类型（storage class）
存储类是C语言与C++语言的标准中，变量与函数的可访问性(即作用域范围scope)与生存期(life time)。
存储类可分为```auto```、```register```、```static```、```extern```、```thread_local```等。
#### 类型说明符（type-specifier）
如```void```、```char```、```short```、```int```、结构体类型、枚举类型等
#### 类型限定符（type-qualifier）
如```const```、```volatile```、```restrict```
#### 类型修饰符
```unsigned``` ```signed```
### 定义相关概念
#### 链接方式
在 C 语言中，变量的链接方式主要分为三种：外部链接（External Linkage）、内部链接（Internal Linkage） 和 无链接（No Linkage）。
##### 外部链接（External Linkage）
具有外部链接的变量可以被其他文件中的代码访问。例如非static的全局变量。
```c
// file1.c
int global_var = 10; // 外部链接

// file2.c
extern int global_var; // 引用外部变量
```
##### 内部链接（Internal Linkage）
具有内部链接的变量只能在定义它的文件中访问。特别是使用 static 关键字定义的全局变量。
```c
// file1.c
static int internal_var = 20; // 内部链接

// file2.c
int internal_var; // 不能访问
```
##### 无链接（No Linkage）
无链接的变量在其作用域内是唯一的，无法在其他作用域中访问。例如局部变量和函数参数。
```c
void func() {
    int local_var = 30; // 无链接
    // local_var 只能在 func 内部访问
}
```
### 变量的声明和定义

变量的声明有两种情况：一种是需要建立存储空间的声明，称为定义性声明；另一种是不需要建立存储空间的声明，称为引用性声明。
```c
int a = 1; // 这是变量的定义

extern int a; // 这是变量的声明
```

### 函数的声明和定义

函数的声明通常在头文件中，而定义在.c文件中。
```c

// func.h
int add(int a, int b); // 函数声明
 
// func.c
#include "func.h"
int add(int a, int b) { // 函数定义
    return a + b;
}
```
#### 传值和传地址
在C语言中，函数的参数传递有两种方式：传值和传址。

1. 传值（Pass by Value）：函数内部对参数进行修改不会影响外部的实参。
```c
#include <stdio.h>
 
void swap(int a, int b) {
    int temp = a;
    a = b;
    b = temp;
    printf("In swap: a = %d, b = %d\n", a, b);
}
 
int main() {
    int x = 10;
    int y = 20;
    swap(x, y);
    printf("In main: x = %d, y = %d\n", x, y);
    return 0;
}
```
在上述代码中，swap函数的参数是按照传值方式传递的，函数内部的a和b只是外部实参x和y的拷贝，所以即使在swap函数内部对a和b进行了交换，也不会影响x和y的值。

2. 传址（Pass by Reference）：通过指针或者引用来传递参数，这样函数内部的修改会影响到外部的实参。
```c
#include <stdio.h>
 
void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
    printf("In swap: *a = %d, *b = %d\n", *a, *b);
}
 
int main() {
    int x = 10;
    int y = 20;
    swap(&x, &y);
    printf("In main: x = %d, y = %d\n", x, y);
    return 0;
}
```
##### 标记输入和输出参数
因为传值通常是作为输入参数存在，而传地址的通常输出参数存在，所以定义两个宏来更好的区分输入和输出参数
```c
#define IN
#define OUT
void add(IN int lhs, IN int rhs, OUT int* res);
```
