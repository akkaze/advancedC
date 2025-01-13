# 使用宏实现通用vector

vector_decl.h
```c
#ifndef CTL_VEC_MINIMAL_SIZE
#define CTL_VEC_MINIMAL_SIZE 8
#endif
#ifndef CTL_VEC_GROW_FACTOR
#define CTL_VEC_GROW_FACTOR 2
#endif
#ifndef CTL_VEC_TRIM_RATIO
#define CTL_VEC_TRIM_RATIO 0.5
#endif

#define DECL_VEC(TYPE) \
typedef struct { \
  TYPE *data; \
  size_t begin_idx; \
  size_t size; \
  size_t capacity; \
} ctl_vec_##TYPE; \
size_t ctl_vec_##TYPE##_size(ctl_vec_##TYPE* vec); \
size_t ctl_vec_##TYPE##_begin_idx(ctl_vec_##TYPE* vec); \
int ctl_vec_##TYPE##_new(size_t size); \
void ctl_vec_##TYPE##_free(ctl_vec_##TYPE *vec); \
int ctl_vec_##TYPE##_ctor_by_size(ctl_vec_##TYPE *vec, size_t size); \
int ctl_vec_##TYPE##_ctor(ctl_vec_##TYPE *vec); \
void ctl_vec_##TYPE##_dtor(ctl_vec_##TYPE *vec); \
int ctl_vec_##TYPE##_copy(ctl_vec_##TYPE *dst, const ctl_vec_##TYPE *src); \
int ctl_vec_##TYPE##_set_at(ctl_vec_##TYPE *vec, size_t index, const TYPE data); \
int ctl_vec_##TYPE##_get_at(ctl_vec_##TYPE *vec, size_t index, TYPE* data); \
int ctl_vec_##TYPE##_push_back(ctl_vec_##TYPE *vec, const TYPE data); \
int ctl_vec_##TYPE##_pop_back(ctl_vec_##TYPE* vec, TYPE* data); \
int ctl_vec_##TYPE##_reserve(ctl_vec_##TYPE* vec, size_t new_capacity); \
int ctl_vec_##TYPE##_resize(ctl_vec_##TYPE* vec, size_t new_size); 
```
vector_impl.h
```c
#define IMPL_QUAL_VEC(TYPE, QUAL) \
QUAL size_t ctl_vec_##TYPE##_size(ctl_vec_##TYPE* vec) { \
  return vec->size; \
} \
QUAL size_t ctl_vec_##TYPE##_begin_idx(ctl_vec_##TYPE* vec) { \
  return vec->begin_idx; \
} \
QUAL int ctl_vec_##TYPE##_ctor_by_size(ctl_vec_##TYPE *vec, size_t size) { \
  int code = 0; \
  TRY { \
    vec->begin_idx = 0; \
    vec->size = size; \
    vec->capacity = size; \
    vec->data = (TYPE *)malloc(vec->capacity * sizeof(TYPE)); \
    CHECK_NONULL(vec->data);           \
    /* 初始化就要初始化所有分配的内存 */  \
    for (size_t idx = 0u; idx < vec->capacity; idx++) { \
      CHECK(TYPE##_ctor(vec->data + idx)); \
    } \
  } \
  CATCH { \
    return code; \
  } \
  return 0; \
} \
QUAL int ctl_vec_##TYPE##_ctor(ctl_vec_##TYPE *vec) { \
  int code = 0; \
  TRY { \
    CHECK(ctl_vec_##TYPE##_ctor_by_size(vec, CTL_VEC_MINIMAL_SIZE)) ; \
  } CATCH { \
    return code; \
  } \
  return 0; \
} \
QUAL void ctl_vec_##TYPE##_dtor(ctl_vec_##TYPE *vec) { \
  int code = 0; \
  TRY {
    /* 销毁就要销毁所有分配的内存 */ \
    for (size_t idx = 0; idx < vec->capacity; idx++) { \
      CHECK(TYPE##_dtor(vec->data + idx)); \
    } \
    free(vec->data); \
    vec->data = NULL; \
  } CATCH { \
    return code; \
  } \
  return 0; \
} \
QUAL void ctl_vec_##TYPE##_clear(ctl_vec_##TYPE* vec) { \
  ctl_res_void res = {0}; \
  res.ec = EOK; \
  vec->begin_idx = 0U; \
  vec->size = 0U; \
  return; \
} \
QUAL int ctl_vec_##TYPE##_reserve(ctl_vec_##TYPE *vec, size_t new_capacity) { \
  int code = 0; \
  TRY {
    CHECK_NOT_EQ(new_capacity, 0); \
    if (new_capacity <= vec->capacity) { \
      return code; \
    } \
    vec->capacity = new_capacity; \
    /* 先申请内存，然后初始化 */ \
    TYPE *new_data = (TYPE *)malloc(vec->capacity * sizeof(TYPE)); \
    CHECK_NONNULL(new_data); \
    for (size_t idx = 0u; idx < vec->capacity; idx++) { \
      CHECK(TYPE##_ctor(new_data + idx)); \
    } \
    /* 把旧的内容拷贝过来 */ \
    for (size_t idx = 0; idx < vec->size; idx++) { \
      CHECK(TYPE##_copy(new_data + idx, vec->data + idx)); \
    } \
    /* 销毁掉旧的内容 */ \
    for (size_t idx = 0; idx < vec->capacity; idx++) { \
      TYPE##_dtor(vec->data + idx); \
    } \
    free(vec->data); \
    vec->data = new_data; \
  } \
  CATCH { \
    return code; \
  } \
  return 0; \
} \
QUAL int ctl_vec_##TYPE##_resize(ctl_vec_##TYPE *vec, size_t new_size) { \
  int code = 0; \
  TRY { \
    if (vec->begin_idx + new_size > vec->capacity) { \
      CHECK(ctl_vec_##TYPE##_reserve(vec, vec->begin_idx + new_size)); \
    } \
    vec->size = new_size; \
  } \
  CATCH { \
    return code; \
  } \
  return 0; \
} \
QUAL int ctl_vec_##TYPE##_copy(ctl_vec_##TYPE *dst, const ctl_vec_##TYPE *src) { \
  int code = 0;
  TRY { \
    dst->size = src->size; \
    /* 如果src的capacity已经比dst的capacity更大，那dst就要释放掉之前的内存，然后重新申请内存 */ \
    if (dst->capacity < src->size) { \
      if (dst->data) { \
        /* 释放掉dst的资源 */ \
        for (size_t idx = 0; idx < dst->capacity; idx++) { \
          TYPE##_dtor(dst->data + idx); \
        } \
        free(dst->data); \
      } \
      /* 必须要最后赋值，因为先释放后申请 */ \
      dst->capacity = src->size; \
      /* 为dst申请资源然后初始化 */ \
      dst->data = (TYPE *)malloc(dst->capacity * sizeof(TYPE)); \
      CHECK_COND_RET_RES(dst->data != NULL, EOOM, res); \
      for (size_t idx = 0; idx < src->size; idx++) { \
        CHECK_EXEC_RES_RET_RES(TYPE##_ctor(dst->data + idx), res, res); \
      } \
    } \
    dst->begin_idx = 0; \
    dst->size = src->size; \
    /* 然后拷贝 */ \
    for (size_t idx = 0; idx < src->size; idx++) { \
      CHECK(TYPE##_copy(dst->data + idx, src->data + (idx + src->begin_idx))); \
    } \
  } \
  CATCH { \
    return code; \
  } \
  return 0; \
} \
QUAL int ctl_vec_##TYPE##_get_at(ctl_vec_##TYPE *vec, size_t index, TYPE* data) { \
  int code = 0; \
  TRY { \
    CHECK_INDEX (index, vec->size); \
    CHECK(TYPE##_copy(data, vec->data + (vec->begin_idx + index))); \
  }
  CATCH { \
    return code; \
  } \
  return 0; \
} \
QUAL int ctl_vec_##TYPE##_set_at(ctl_vec_##TYPE *vec, size_t index, const TYPE value) { \
  int code = 0;
  TRY { \
    CHECK_INDEX (index, vec->size); \
    CHECK(TYPE##_copy(vec->data + (vec->begin_idx + index), &value)); \
  } \
  CATCH { \
    return code; \
  } \
  return res; \
} \
QUAL ctl_res_void ctl_vec_##TYPE##_grow(ctl_vec_##TYPE *vec) { \
  ctl_res_void res = {0}; \
  res.ec = EOK; \
  size_t new_capacity = vec->capacity * CTL_VEC_GROW_FACTOR; \
  CHECK_EXEC_RES_RET_RES(ctl_vec_##TYPE##_reserve(vec, new_capacity), res, res); \
  return res; \
} \
QUAL int ctl_vec_##TYPE##_push_back(ctl_vec_##TYPE *vec, const TYPE data) { \
  int code = 0; \
  TRY { \
    if (vec->capacity == (vec->begin_idx + vec->size)) { \
      ctl_vec_##TYPE##_grow(vec); \
    } \
    CHECK(TYPE##_copy(vec->  data + (vec->begin_idx + vec->size), &data)); \
    vec->size++; \
  } \
  CATCH { \
    return code; \
  } \
  return 0; \
} \
QUAL int ctl_vec_##TYPE##_pop_back(ctl_vec_##TYPE *vec, TYPE* data) { \
  int code = 0; \
  TRY { \
    CHECK_NOT_EQ (vec->size, 0); \
    CHECK(TYPE##_copy(&(res.value), vec-> data + (vec->begin_idx + vec->size - 1u))); \
    vec->size--; \
  } \
  CATCH { \
    return code; \
  } \
  return 0; \
}
#define IMPL_VEC(TYPE) IMPL_QUAL_VEC(TYPE, )
#define IMPL_STATIC_VEC(TYPE) IMPL_QUAL_VEC(TYPE, static)
#define IMPL_INLINE_STATIC_VEC(TYPE) IMPL_QUAL_VEC(TYPE, inline static)
```

如何使用
在vector.h中添加如下代码
```c
DECL_VEC(int);
```

在vector.c中添加如下代码
```c
IMPL_VEC(int);
```