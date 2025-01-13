# 错误处理
## 使用返回值和错误码
```c
typedef enum {
  E_OK = 0,
  E_DOM,
  E_OVERLOW,
  E_OOM,
  E_END
} status_code;
// 也可以返回int，因为枚举可以转换为int
status_code divide(int a, int b, int *result) {
  if (b == 0) {
    return E_DOM;   // 返回错误码
  }
  *result = a / b;
  return E_OK;
}

int main() {
  int result;
  status_code code = divide(10, 0, &result);
  if (code == E_OK) {
    printf("Result: %d\n", result);
  } else {
    printf("Error: %d\n", code);
    // 错误码要一直向上返回，直到作为进程返回值
    return code;
  }
  return 0;
}
```
<mark>析构函数通常不返回错误码，防止嵌套错误，如果遇到错误，一般直接exit</mark>
## 使用goto
在上面的例子中，调用divide的函数之前并没有申请任何资源，如果申请资源，需要在return之前释放掉这些资源，注意需要在所有return之前释放掉这些资源，但是一旦程序结构变得非常复杂，return会非常做，这样会变得非常难以维护，所以通常在函数最后统一返回，一旦遇到错误就跳到函数最后的block里。
注意，因为使用了goto，函数作用域内的变量定义最好放在函数头部。因为ANSI C不允许在goto之后定义变量。
```c
int main() {
  int* result = NULL;
  FILE* fp = NULL;
  status_code code = E_OK;
  result = (int*)malloc(sizeof(int));
  if (!result) goto cleanup;
  fp = fopen("log.txt", "w");
  if (!fp) goto cleanup;
  code = divide(10, 0, result);
  if (code == E_OK) {
    fprintf(fp, "Result: %d\n", *result);
  } else {
    goto cleanup;
  }
cleanup:
  fprintf(stderr, "Error: %d\n", code);
  if (result) free(result);
  if (fp) fclose(fp);
  // 最后返回code
  return code;
}
```
## TRY/CATCH/CHECK宏
为了简化上面的流程，我们定义几个宏
```c
#define TRY
#define CATCH cleanup:
#define CHECK(...) {                                                           \
  code = __VA_ARGS__;                                                          \
  if (code != E_OK) {                                                        \  
    fprintf(stderr, "%s %s:%u code: %d", __FUNC__, __FILE__, __LINE__, code);  \
    goto cleanup;                                                              \
  }                                                                            \
}
#define CHECK_NONULL(ptr) if (!(ptr)) {                 \
  code = E_OOM;                                         \
  goto cleanup;                                         \
}
```
```c
int main() {
  int* result = NULL;
  FILE* fp = NULL;
  status_code code = E_OK;
  TRY {
    result = (int*)malloc(sizeof(int));
    CHECK_NONNULL(result);
    fp = fopen("log.txt", "w");
    CHECK_NONNULL(fp);
    CHECK(divide(10, 0, result));
    // 已经检查过，无需检查
    fprintf(fp, "Result: %d\n", *result);
  }
  CATCH {
    if (result) free(result);
    if (fp) fclose(fp);
    // 最后返回code
    return code;
  }
  // 防止warning
  return 0;
}
```
## setjump/longjump
