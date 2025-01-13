# 面向对象
## 封装
### 抽象
```c
enum gender_s {
  MALE,
  FEMALE
} gender_t;
typedef struct person_s {
  gender_t gender;
  unsigned int age;
  const char* name;
} person;

person tom;
```
### self/this 指针
```c
void set_gender(person* self, geneder_s gender) {
  self->gender = gender;
}

void set_age(person* self, unsigned int age) {
  self->age = age;
}

set_gender(&tom, MALE);
```
### 构造、拷贝、析构函数
#### 构造函数
```c
void person_ctor_by_gender_age_name(person* self, gender_t gender, unsigned int age, const char* name) {
  self->gender = gender;
  self->age = age;
  self->name = name;
}

// 使用构造函数
#define NEW_PERSON_BY_GENDER_AGE_NAME(gender, age, name) ({   \
  person* someone = (person*)malloc(sizeof(person));          \
  person_ctor_by_gender_age_name(someone, gender, age, name); \
  someone;                                                    \
})

person* tom = NEW_PERSON_BY_GENDER_AGE_NAME(MALE, 18, "tom");
```

#### 析构函数
析构函数除了self指针通常没有参数。值得注意的是，由于C语言没有raii，析构函数必须手动调用。
```c
void person_dtor(person* self) {
  printf("person destructor\n");
}

// 使用析构函数
#define DEL_PERSON(someone) { \
  person_dtor(someone);       \
  free(someone);              \
} 

DEL_PERSON(tom);
```

#### 拷贝函数
拷贝函数通常接受两个参数，一个为目的结构体的指针，另一个是源结构体的指针
```c
void person_copy(person* dst, const person* src) {
  dst->gender = src->gender;
  dst->age = src->age;
  dst->name = src->name;
}

person* jerry = ...;
person_copy(jerry, tom);
```
### 可见性
通常需要对外暴露的方法声明头文件里。不需要对外暴露，仅仅在内部使用的函数，直接定义在源文件里。
person.h
```c
void person_speak(person* self, const char* msg);
```
person.c
```c
// 不对外声明，仅在c文件里定义
void internal_person_speak(person* self, const char* greeting, const char* msg) {
  printf("%s\n", greeting);
  printf("%s\n", msg);
}
void person_speak(person* self, const char* msg) {
  internal_person_speak(self, "how are you", msg);
}
```
## 继承
### 单一继承
在单继承的情况下，<mark>通常将父类作为子类的第一个成员，这样做的原因是，子对象和父对象的首地址是相同的。</mark>
```c
typedef man_s {
  person super;
  int id;
} man;

man tom;
```
```&tom```和```&tom.super```是同一地址，这样方便转换。
子类可以很容易的使用父类的方法
```c
void set_age(man* self, unsigned int age) {
  // 调用父类方法
  set_age(&self->super, age);
}
```
### 多继承
```c
typedef struct object_s {
  int id;
} object;

typedef struct man_s {
  person super1;
  object super2;
} man;
```
### 向上转换
```c
object* super = (object*)((char*)someone + offsetof(man, super2)); 
```
### 向下转换
```c
man* someone = (man*)((char*)super - offsetof(man, super2)); 
```
## 多态
在编程语言和类型论中，多态（英语：polymorphism）指为不同数据类型的实体提供统一的接口。 多态类型（英语：polymorphic type）可以将自身所支持的操作套用到其它类型的值上。
<mark>多态的存在具有哲学意义，它意味着具有相似外观的事物具有相似的行为，但是行为的结果往往不同。</mark>

- 相似的外观意味着它们可以被存放到一起。
- 相似的行为意味着它们可以使用相同的调用方法。
- 不同的行为结果意味着这些行为的实现并不一样。
### 虚函数实现
<mark>这里的技巧和单继承一样，子类在虚继承的时候需将父类作为第一个成员</mark>。
<mark>这里有一点非常重要，在虚继承的情况下，析构函数必须为虚函数，因为当调用析构函数的时候，self指针的类型已经不知道了，这是无法通过类型调用到正确的析构函数上，只能使用虚函数。</mark>
```c
typedef void (sound_func_t*)();
typedef struct animal_s {
  sound_func_t sound;
} animal;

void animal_sound() {
  printf("~~~\n");
}

void animal_ctor(animal* self) {
  self->sound = animal_sound;
}

typedef struct dog_s {
  animal super; 
} dog;

void dog_sound() {
  printf("wow\n");
}

void dog_ctor(dog* self) {
  self->super.sound = dog_sound;
}

typedef struct cat_s {
  animal super; 
} cat;

void cat_sound() {
  printf("meow\n");
}

void cat_ctor(cat* self) {
  cat->super.sound = cat_sound;
}
```

```c
animal* tom = (animal*)NEW_CAT();
animal* spike = (animal*)NEW_DOG();
tom->sound();
spike->sound();
// 这个时候已经不知道tom和spike的具体类型了，因为可能是在另一个函数里调用的
DEL_ANIMAL(tom);
DEL_ANIMAL(spike);
```
### 一般性的表驱动
这里的关键是定义type_code，并让它作为所有子类的第一个成员，这相当于c++的type_info，但是在内存占用上更优，如果对象非常多，这也是不小的优化。比如在渲染引擎中，dom元素会发出多，每一个都持有虚表指针，这样会造成浪费。type_code可以在不同的表之间共用，这也是它的优势。
```c
typedef void* animal_handle;
typedef enum {
  DOG,
  CAT,
  NONE
} animal_code;

typedef struct {
  animal_code code;
  sound_func_t sound;
} sound_method;


sound_method sound_methods[] = {
  {DOG, dog_sound},
  {CAT, cat_sound},
  {NONE, animal_sound}
}

typedef struct animal_s {
  animal_code code;
} animal;

void animal_ctor(animal* self) {
  self->code = NONE;
}

typedef struct dog_s {
  animal_code code;
} dog;

void dog_ctor(dog* self) {
  self->code = DOG;
}

typedef struct cat_s {
  animal_code code;
} cat;

void cat_ctor(cat* self) {
  self->code = CAT;
}

#define LOOKUP_SOUND_METHOD(code) ({      \
  sound_func_t __sound = NULL;            \
  for (unsigned int i = 0u; i < sizeof(sound_methods) / sizeof(sound_methods[0]); i++) {  \
    if (sound_methouds[i].code == code) { \
      __sound = sound_methods[i].soound;  \
      break;                              \
    }                                     \        
  }                                       \
  __sound;                                \
})

#define CALL_SOUND_METHOD(handle) {                   \
  animal_code __code = *(animal_code*)handle;         \
  sound_func_t __sound = LOOKUP_SOUND_METHOD(__code); \
  __sound();                                          \ 
}
```
```c
animal_handle tom = (animal_handle)NEW_CAT();
animal_handle spike = (animal_handle)NEW_DOG();
CALL_SOUND_METHOD(tom);
CALL_SOUND_METHOD(spike);
// 这个时候已经不知道tom和spike的具体类型了，因为可能是在另一个函数里调用的
DEL_ANIMAL(tom);
DEL_ANIMAL(spike);
```
### 虚表实现