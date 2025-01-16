# 基础BUG
## 算术BUG
### 整数
#### 加法
##### 加法上溢出
```c
int a = INT32_MAX - 1;
int b = a + 10;
```
##### 加法下溢出
```c
int a = INT32_MIN + 1;
int b = a - 10;
```
#### 减法
##### 减法上溢出
```c
int a = 10;
int b = a - INT32_MIN;
```
##### 减法下溢出
```c
int a = -10;
int b = a - INT32_MAX;
```
#### 取反
##### 有符号整数取反上溢出
```c
int a = INT32_MIN;
int b = -a;
```
***对于有符号整数，取反没有下溢出，因为有符号整数的最小值的绝对值大于最大值的绝对值***
##### 无符号整数取反下溢出
对于无符号整数，除了0，其他任何数取反都会下溢出
```c
unsigned int a = 10;
unsigned int b = -a;
```
#### 乘法
##### 乘法上溢出
```c
int a = INT32_MAX / 2;
int b = INT32_MAX / 2;
int c = a * b;
```
##### 整数乘法下溢出
```c
int a = INT32_MIN / 2;
int b = INT32_MIN / 2;
int c = a * b;
```
#### 除法和模运算
除零通常会造成硬件异常
##### 有符号整数除法上溢出
这只有一种情况，被除数是最小值，除数是-1，相当于取反
##### 除零
```c
int a = 10;
int b = 0;
int c = a / b;
```
#### 移位
##### 有符号整数左移溢出
因为有符号位，有符号整数允许左移的最大位数要比自身的位数小1。
```c
int a = 0;
int b = a << 31;
```
##### 无符号整数左移溢出
无符号整数允许左移的最大位数通常和自身位数相等。
```c
unsigned int a = 0;
unsigned int b = a << 32;
```
## 浮点数
### inf和nan
在计算机中，inf和NaN是浮点数中的特殊值。

inf表示无穷大，它在计算机中通常用一个特殊的二进制表示，即所有位都是1，且指数部分为全1，尾数部分为0。

NaN表示非数字（Not a Number），它在计算机中通常用一个特殊的二进制表示，即指数部分为全1，尾数部分不为0。
### 加法
#### 上溢出
```c
float a = FLT_MAX - 1;
float b = a + 2.0;
assert(b == INF);
```
#### 下溢出
```c
float a = FLT_MIN + 1;
float b = a - 2.0;
assert(b == -INF);
```
### 减法
```c
float a = 2.0;
float b = a - FLT_MIN;
assert(b == INF);
```
#### 下溢出
```c
float a = -2.0;
float b = a - FLT_MAX;
assert(b == -INF);
```
### 乘法
```c
float a = FLT_MAX;
float b = a * 2.0;
assert(b == INF);
```
#### 下溢出
```c
float a = FLT_MIN;
float b = a * 2.0;
assert(b == -INF);
```
### 除法
事实上除零并不一定完全就是除以0，因为浮点数的精度有误差，事实上除以一个很小的数，小于FLT_EPS等同于除零。
#### 正数除零上溢出
```c
float a = 2.0;
float b = a / 0.0;
assert(b == INF);
```
#### 负数除零上溢出
```c
float a = -2.0;
float b = a / 0.0;
assert(b == -INF);
```
#### 零除零上溢出
```c
float a = 0.0;
float b = a / 0.0;
assert(b == NAN);
```
## 内存BUG
### 内存越界（overflow）
#### 堆内存越界
```c
int main() {
  int* ptr = (int*)malloc(8 * sizeof(int));
  ptr[10] = 10;
  free(ptr);
  return 0;
}
```
#### 栈内存越界
```c
int main() {
  int arr[10] = {0};
  arr[10] = 10;
  return 0;
}
```
#### 全局内存越界
```c
int arr[10] = {0};
int main() {
  arr[10] = 10;
  return 0;
}
```
### 内存泄漏（leak）
直到程序退出也没有释放的内存往往会导致内存泄漏。
```c
int* func() {
  int* ptr = (int*)malloc(1024 * sizeof(int));
  return ptr;
}
int main() {
  int* ptr = func();
  return 0;
}
```
### 双重释放（doble free）
```c
int* func() {
  int* ptr = (int*)malloc(1024 * sizeof(int));
  free(ptr);
  return ptr;
}
int main() {
  int* ptr = func();
  free(ptr);
  return 0;
}
```
### 释放后使用（use after free）
```c
int* func() {
  int* ptr = (int*)malloc(1024 * sizeof(int));
  free(ptr);
  return ptr;
}
int main() {
  int* ptr = func();
  *ptr = 10;
  return 0;
}
```
### 野指针
以下类型统称为野指针
#### 空指针
初始化为空的指针
```c
int main() {
  int* p = NULL;
  *p = 10;
  return 0;
}
#### 随机指针
没有被初始化就被使用的指针往往指向随机值，当然也可能是空值。
```c
int main() {
  int* p;
  *p = 10;
  return 0;
}
```
#### 悬垂指针
悬垂通常指该指针指向的内容的生命周期已经结束，最典型的是返回栈上变量的地址。
```c
int* func() {
  int a = 0;
  return &a;
}
int main() {
  // 这就是悬垂指针
  int* ptr = func();
  return 0;
}
### 栈溢出
以下程序在某些机器下可能会栈溢出，注意func的形参和实参都被分配在栈上，可能会导致栈内存不足。
```c
long func(int arr[]) {
  long sum = 0;
  ...
  return sum;
}
int main() {
  int arr[1024 * 1024];
  long sum = func(arr);
  printf("%ld", sum);
  return 0;
}
```
## 并发BUG
为了可移植性，这里使用[tinycthread](https://github.com/tinycthread/tinycthread)的api。

### 数据读写竞争
### 锁相关BUG
```c
#include <tinycthread.h>

int value = 0;

typedef struct thr_call_s
{
    thrd_t thr;
    int push;
    int ret;
} thr_call;

int thr_routine(void* data)
{
  // 不同线程改写同一全局变量，会导致未定义行为，触发BUG
  value = (thr_call*)data->push;
  return 0;
}

int main(int argc, char const *argv[])
{
    thr_call val[5] = {0};
    for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
    {
        val[i].push = i;

        thrd_create(&(val[i].thr), thr_rountine, &(val[i]));
    }
    for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
    {
        thrd_join(val[i].thr, &(val[i].ret));
    }
    // 查看返回值
    for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
    {
        printf("return:[%d], expect:[%d]\n", val[i].ret, i%2==0?0:1);
    }
    return 0;
}
```
#### 死锁
```c
#include <tinycthread.h>

mtx_t mtx1;
mtx_t mtx2;

typedef struct thr_call_s
{
    thrd_t thr;
    int push;
    int ret;
} thr_call;

void print(void* data) { 
  for (size_t i = 0; i < 5; i++)
  {
      printf("thread:[%d], index:[%d]\n", ((th_call*)data)->push, i);
  }
}
int thr1_routine(void* data)
{
  mtx_lock(&mtx1); 
  mtx_lock(&mtx2);  
  print(data);
  mtx_unlock(&mtx2);
  mtx_unlock(&mtx1);
  return ((thr_call*)data)->push % 2 == 0? 0 : 1;
}

int thr2_routine(void* data)
{
  mtx_lock(&mtx2); 
  mtx_lock(&mtx1);  
  print(data);
  mtx_unlock(&mtx1);
  mtx_unlock(&mtx2);
  return ((thr_call*)data)->push % 2 == 0? 0 : 1;
}

int main(int argc, char const *argv[])
{
  thr_call val[2] = {0};
  mtx_init(&mtx1, mtx_plain);
  mtx_init(&mtx2, mtx_plain);
  val[0].push = 0;
  val[1].push = 1;

  thrd_create(&(val[0].thr), thrd1_rountine, &(val[0]));
  thrd_create(&(val[1].thr), thrd2_rountine, &(val[1]));

  thrd_join(val[0].thr, &(val[0].ret));
  thrd_join(val[1].thr, &(val[1].ret));

  mtx_destroy(&mtx1);
  mtx_destroy(&mtx2);
  return 0;
}
```
#### 不可重入锁重入
```c
#include <tinycthread.h>

mtx_t mtx;

typedef struct thr_call_s
{
    thrd_t thr;
    int push;
    int ret;
} thr_call;

void print(void* data) {
  // 这里会触发BUG，一直等待
  mtx_lock(&mtx);  
  for (size_t i = 0; i < 5; i++)
  {
      printf("thread:[%d], index:[%d]\n", ((th_call*)data)->push, i);
  }
  mtx_unlock(&mtx);
}
int thr_routine(void* data)
{
  mtx_lock(&mtx);  
  print(data);
  mtx_unlock(&mtx);
  return ((thr_call*)data)->push % 2 == 0? 0 : 1;
}

int main(int argc, char const *argv[])
{
  thr_call val[5] = {0};
  mtx_init(&mtx, mtx_plain);
  for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
  {
      val[i].push = i;

      thrd_create(&(val[i].thr), thr_rountine, &(val[i]));
  }
  for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
  {
      thrd_join(val[i].thr, &(val[i].ret));
  }
  // 查看返回值
  for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
  {
      printf("return:[%d], expect:[%d]\n", val[i].ret, i%2==0?0:1);
  }
  mtx_destroy(&mtx);
  return 0;
}
```
#### 锁在同一线程多次释放
```c
#include <tinycthread.h>

mtx_t mtx;

typedef struct thr_call_s
{
  thrd_t thr;
  int push;
  int ret;
} thr_call;

void print(void* data) { 
  for (size_t i = 0; i < 5; i++)
  {
      printf("thread:[%d], index:[%d]\n", ((th_call*)data)->push, i);
  }
  mtx_unlock(&mtx);
}
int thr_routine(void* data)
{
  mtx_lock(&mtx);  
  print(data);
  // 这里会触发BUG，被第二次释放
  mtx_unlock(&mtx);
  return ((thr_call*)data)->push % 2 == 0? 0 : 1;
}

int main(int argc, char const *argv[])
{
  thr_call val[5] = {0};
  mtx_init(&mtx, mtx_plain);
  for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
  {
      val[i].push = i;

      thrd_create(&(val[i].thr), thr_rountine, &(val[i]));
  }
  for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
  {
      thrd_join(val[i].thr, &(val[i].ret));
  }
  // 查看返回值
  for (size_t i = 0; i < sizeof(val) / sizeof(*val); i++)
  {
      printf("return:[%d], expect:[%d]\n", val[i].ret, i%2==0?0:1);
  }
  mtx_destroy(&mtx);
  return 0;
}
```
