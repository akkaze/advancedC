# 使用void*实现通用vector

## icd函数

```c
typedef int (ctor_func_t*)(void*);
typedef int (copy_func_t*)(void*, void*);
// 析构函数通常不返回错误码，防止嵌套错误，如果遇到错误，一般直接exit
typedef void (dtor_func_t*)(void*);

typedef struct icd_s {
  size_t size;
  ctor_func_t ctor;
  copy_func_t copy;
  dtor_func_t dtor;
} icd_t;
```

```c
typedef struct vector_s {
  void *data; 
  size_t begin_idx;
  size_t size;
  size_t capacity;
  // element的icd函数
  icd_t icd;
} vector;

size_t vector_size(vector* vec) {
  return vec->size;
}
size_t vector_begin_idx(vector* vec) {
  return vec->begin_idx;
}

int vector_ctor_by_size(vector *vec, size_t size) {
  int code = 0;
  TRY {
    vec->begin_idx = 0;
    vec->size = size;
    vec->capacity = size;
    vec->data = malloc(vec->capacity * vec->icd.size);
    CHECK_NONNULL(vec->data)
    /* 初始化就要初始化所有分配的内存 */
    char* data_p = (char*)vec->data;
    for (size_t idx = 0; idx < vec->capacity; idx++) {
      CHECK(vec->icd.ctor(data_p + idx * vec->icd.size));
    }
  }
  CATCH {
    return code;
  }
  return 0;
}

void vector_dtor(vector *vec) {
  char* data_p = (char*)vec->data;
  for (size_t idx = 0; idx < vec->capacity; idx++) {
    vec->icd.dtor(data_p + idx * vec->icd.size);
  }
  free(vec->data);
  vec->data = NULL;
  return;
}

int vector_copy(vector *dst, const vector *src) {
  int code = 0;
  dst->size = src->size;
  char* dst_data_p = (char*)dst->data;
  char* src_data_p = (char*)src->data;
  TRY {
    /* 如果src的capacity已经比dst的capacity更大，那dst就要释放掉之前的内存，然后重新申请内存 */ \
    if (dst->capacity < src->size) {
      if (dst->data) {
        /* 释放掉dst的资源 */
        if (dst->icd.dtor) {
          for (size_t idx = 0u; idx < dst->capacity; idx++) {
            dst->icd.dtor(dst_data_p + idx);
          }
        }
        free(dst->data);
      }
      /* 必须要最后赋值，因为先释放后申请 */
      dst->capacity = src->size;
      /* 为dst申请资源然后初始化 */
      dst->data = malloc(dst->capacity * dst->icd.size);
      CHECK_NONULL(dst->data);
      if (dst->icd.ctor) {
        for (size_t idx = 0u; idx < src->size; idx++) {
          CHECK(dst->icd.ctor(dst_data_p + idx));
        }
      }
    }
    dst->begin_idx = 0;
    dst->size = src->size;
    /* 然后拷贝 */
    if (src->icd.copy) {
      for (size_t idx = 0u; idx < src->size; idx++) {
        CHECK(src->icd.copy(dst_data_p + idx, src_data_p + (idx + src->begin_idx))); 
      }
    }
  }
  CATCH {
    return code;
  }
  return 0;
}

void vector_clear(vector* vec) {
  vec->begin_idx = 0u;
  vec->size = 0u;
  return;
}
int vector_reserve(vector *vec, size_t new_capacity) {
  
  int code = 0;
  char* dst_data_p = (char*)dst->data;
  char* src_data_p = (char*)src->data;
  TRY {
    CHECK_NOT_EQ(new_capacity, 0);
    if (new_capacity <= vec->capacity) {
      return code;
    }
    vec->capacity = new_capacity;
    /* 先申请内存，然后初始化 */
    void* new_data = malloc(vec->capacity * vec->icd.size); 
    CHECK_NONNULL(new_data);
    char* new_data_p = (char*)new_data;
    if (vec->icd.ctor) {
      for (size_t idx = 0u; idx < vec->capacity; idx++) {
        CHECK(T_ctor(new_data + idx), res, res);
      }
    }
    /* 把旧的内容拷贝过来 */
    if (vec->icd.copy) {
      for (size_t idx = 0; idx < vec->size; idx++) {
        CHECK(vec->icd.copy(new_data_p + idx, data_p + idx));
      }
    }
    /* 销毁掉旧的内容 */
    if (vec->icd.dtor) {
      for (size_t idx = 0; idx < vec->capacity; idx++) {
        CHECK(vec->icd.dtor(data_p + idx));
      }
    }
    free(vec->data);
    vec->data = new_data;
  }
  CATCH {
    return code;
  }
  return 0;
}
int vector_resize(vector *vec, size_t new_size) {
  int code = 0;
  TRY {
    if (vec->begin_idx + new_size > vec->capacity) {
      CHECK(vector_reserve(vec, vec->begin_idx + new_size));
    }
    vec->size = new_size;
  } CATCH {
    return code;
  }
  return 0;
}

int vector_get_at(vector *vec, size_t index, void* elem) {
  int code = 0;
  char* data_p = (char*)vec->data;
  TRY {
    CHECK_INDEX (index, vec->size);
    memcpy(elem, data_p + index * vec->icd.size, size);
    if (vec->icd.copy)
      CHECK(vec->icd.copy(elem, data_p + index * vec->icd.size));
  }
  CATCH {
    return code;
  }
  return 0;
} 
int vector_set_at(vector *vec, size_t index, void* elem) {
  int code = 0;
  TRY {
    CHECK_INDEX (index, vec->size);
    char* data_p = (char*)vec->data;
    memcpy(data_p + index * vec->icd.size, elem, size);
    if (vec->icd.copy)
      CHECK(vec->icd.copy(data_p + index * vec->icd.size, elem));
  }
  CATCH {
    return code;
  }
  return 0;
}

int vector_grow(vector *vec) {
  int code = 0;
  size_t new_capacity = vec->capacity * CTL_VEC_GROW_FACTOR;
  TRY {
    CHECK(vector_reserve(vec, new_capacity));
  }
  CATCH {
    return code;
  }
  return 0;
}

int vector_push_back(vector *vec, const void* elem) {
  int code = 0;
  char* data_p = (char*)vec->data;
  TRY {
    if (vec->capacity == (vec->begin_idx + vec->size)) {
      CHECK(vector_grow(vec));
    }
    memcpy(vec->icd.copy(data_p + (vec->begin_idx + vec->size), elem, vec->icd.size));
    if (vec->icd.copy)
      CHECK(vec->icd.copy(data_p + (vec->begin_idx + vec->size), elem));
    vec->size++;
  }
  CATCH {
    return code;
  }
  return 0;
}
int vector_pop_back(vector *vec, void* elem) {
  int code = 0;
  TRY {
    CHECK_NOT_EQ (vec->size, 0);
    memcpy(elem, vec->icd.copy(data_p + (vec->begin_idx + vec->size), vec->icd.size));
    if (vec->icd.copy)
      CHECK(elem, vec->icd.copy(data_p + (vec->begin_idx + vec->size)));
    vec->size--;
  } CATCH {
    return code;
  }
  return 0;
}

```