# 模板
## 使用宏实现模板
使用宏可以在编译期实现模板，它和c++等语言的模板本质一样，通过对不同的类型进行codegen来实例化代码，但是因为语言本身局限，**不支持隐式实例化和特化**。但是在实际开发中完全够用，可以复用大量代码，只是易用性略差。
和c++模板一样，它会导致代码膨胀。
```c
#define DECL_ADD(type)                                                  \
  void add_##type(type* dst, type* lhs, type* rhs, unsigned int num);
#define IMPL_ADD(type)                                                  \
  void add_##type(type* dst, type* lhs, type* rhs, unsigned int num) {  \
    for (unsinged int i = 0; i < num; i++) {                            \
      dst[i] = lhs[i] + rhs[i];                                         \
    }                                                                   \
  }
#define CALL_ADD(type) add_##type
```
```c
DECL_ADD(int);
DECL_ADD(float);
```
```c
IMPL_ADD(int);
IMPL_ADD(float);
```
```c
CALL_ADD(int)(dst, lhs, rhs, num);
CALL_ADD(float)(dst, lhs, rhs, num);
```
## 使用不透明指针实现模板
### 什么是不透明指针？

顾名思义，不透明是我们看不到的。例如，木材是不透明的。不透明指针是指向数据结构的指针，该数据结构的内容在定义之时不公开。
在泛型的实现里，我们使用void*来实现不透明指针，因为它可以指向任何内存，而不用关心具体的类型，这正是泛型所需要的。
**使用不透明指针来实现模板，几乎不会造成任何代码膨胀，这是它的优势，但是性能略差，因为它可能会本来在寄存器上完成的计算强行转移到栈上。**
```c
typedef void (add_func_t*)(void*,  void* , void* );
void add(void* dst, void* lhs, void* rhs, unsigned int num, unsigned int elem_size, add_func_t func_ptr) {
  char* dst_p = (char*)dst;
  char* lhs_p = (char*)lhs;
  char* rhs_p = (char*)rhs;
  for (unsigned int i = 0; i < num; i++) {
    func_ptr(dst_p, lhs_p, rhs_p);
    dst_p += elem_size;
    lhs_p += elem_size;
    rhs_p += elem_size;
  }
}
```
```c
void add_int(void* dst, void* lhs, void* rhs) {
  *(int*)dst = *(int*)lhs + *(int*)rhs; 
}
void add_float(void* dst, void* lhs, void* rhs) {
  *(float*)dst = *(float*)lhs + *(float*)rhs; 
}
```
```c
add(dst, lhs, rhs, num, sizeof(int), add_int);
add(dst, lhs, rhs, num, sizeof(float), add_float);
```
## 侵入式容器
**注意：侵入式容器并不是真正的模板，它只能复用带有链的数据结构的逻辑，诸如链表，树和图等**
普通链表：
我们经常使用的普通链表是每个节点的next指针指向下一个节点的首地址：
```c
struct node
{    
    int data;
    struct node* next;
}
```
普通链表的缺点：

- 一条链表上的所有节点的数据类型需要完全一致
- 对某条链表的操作如插入，删除等只能对这种类型的链表进行操作，如果链表的类型换了，就要重新再封装出一套一样的操作，泛化能力差
侵入式链表：
侵入式链表的节点的链接成员指向的是下一个节点的链接成员：

节点结构如下：
```c
typ**edef struct link
{
    struct link* next;
}list_t;

typedef struct
{
    int data;
    struct link* list;
} node;
```
侵入式链表的优点：

- 节点类型无需一致，只需要包含list_t成员即可
- 泛化能力强，所有链表的操作方式均可统一；
访问侵入式链表中的节点数据，需要使用offsetof和container_of
```c
typedef struct link
{
    struct link* next;
    struct link* prev;
} list_t;


#define LIST_HEAD_INIT(name)    {&(name), &(name)}
#define LIST_HEAD(name)         list_t name = LIST_HEAD_INIT(name)


#define list_entry(node, type, member) \
    container_of(node, type, member)

// 下面这些api都是通用的，无论data是数据类型
#define list_for_each(pos, head) \
    for (pos = (head)->next; pos != (head); pos = pos->next)


#define list_for_each_safe(pos, n, head) \
  for (pos = (head)->next, n = pos->next; pos != (head); \
    pos = n, n = pos->next)


void list_init(list_t* list);
void list_insert_after(list_t* list, list_t* node);
void list_insert_before(list_t* list, list_t* node);
void list_remove(list_t* node);
int list_isempty(const list_t* list);
unsigned int list_size(const list_t* list);
```
已知link的情况下，如何拿到data的值
```c
struct node
{    
    int data;
    list_t link;
}
list_t link_p;
int val = list_entry(link_p, struct node, link)->data;
```
以下实现都是通用的
```c
#include "link_list.h"
 
void list_init(list_t* list)
{
    list->next = list->prev = list;
}

void list_insert_after(list_t* list, list_t* node)
{
    list->next->prev = node;
    node->next = list->next;

    list->next = node;
    node->prev = list;
}

void list_insert_before(list_t* list, list_t* node) 
{
    list->prev->next = node;
    node->prev = list->prev;
    list->prev = node;
    node->next = list;
}

void list_remove(list_t* node)
{
    node->next->prev = node->prev;
    node->prev->next = node->next;

    node->next = node->prev = node;
}

int list_isempty(const list_t* list)
{
    return list->next == list;
}

unsigned int list_size(const list_t* list)
{
    unsigned int size = 0;
    const list_t* p = list;
    while (p->next != list)
    {
        p = p->next;
        size++;
    }
    return size;
}
```