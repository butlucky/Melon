## 资源开销



Melon中的资源开销（span）组件是用来测量C语言函数开销的，这个模块需要配合`函数模板`模块一同使用，因此也需要定义`MLN_FUNC_FLAG`才会使得函数模板功能启用，进而实现函数开销的跟踪。

目前支持的开销如下：

- 时间开销



### 头文件

```c
#include "mln_span.h"
```



### 模块名

`span`



### 函数/宏



#### mln_span_start

```c
mln_span_start();
```

描述：开始进行测量。这个宏函数中会对模块使用的全局变量进行设置，并且在测量过程中会仅跟踪调用该函数的线程所调用的函数。

**注意：**这个宏函数需要与`mln_span_stop`宏函数在统一作用域内调用，或者`mln_span_stop`调用点的调用栈深度应大于`mln_span_start`调用点的调用栈深度。举例：

> 如果有如下三个函数：`main`，`foo`，`bar`。他们之间的调用关系如下：
>
> ```c
> void bar(void)
> {
> }
> void foo(void)
> {
>   bar();
> }
> int main(void)
> {
>   foo();
>   return 0;
> }
> ```
>
> 那么假设`mln_span_start`在`foo`的`bar`调用前被调用，那么`mln_span_stop`应该在`foo`的`mln_span_start`之后或者在`bar`函数中调用，但不能在`main`中调用。

如果不遵守上述规则，则可能出现内存泄漏的情况。

返回值：无



#### mln_span_stop

```c
mln_span_stop();
```

描述：停止进行测量。这个宏函数会销毁部分模块内的全局变量内容，但会保留此次测量中包含测量值的结构。这个宏函数的调用时机参考`mln_span_start`中的描述。

返回值：无



#### mln_span_release

```c
mln_span_release();
```

描述：释放本次测量的测量值结构。**注意：**这个宏函数应在`mln_span_stop`之后被调用，否则将可能出现内存访问异常。

返回值：无



#### mln_span_move

```c
mln_span_move();
```

描述：获取本次测量的测量值结构，并将指向该结构的全局指针置`NULL`。**注意：**这个宏函数应在`mln_span_stop`之后被调用，否则将出现不可预知的错误。

返回值：`mln_span_t`指针



#### mln_span_dump

```c
mln_span_dump();
```

描述：输出当前测量的数据。

返回值：无



#### mln_span_free

```c
void mln_span_free(mln_span_t *s);
```

描述：释放`mln_span_t`结构内存。

返回值：无



### 示例

这是一个多线程的示例，用来展示`mln_span`接口的使用以及在多线程环境下的效果。

```c
//a.c

#include <pthread.h>
#include "mln_span.h"
#include "mln_func.h"

MLN_FUNC(int, abc, (int a, int b), (a, b), {
    return a + b;
})

MLN_FUNC(static int, bcd, (int a, int b), (a, b), {
    return abc(a, b) + abc(a, b);
})

MLN_FUNC(static int, cde, (int a, int b), (a, b), {
    return bcd(a, b) + bcd(a, b);
})

void *pentry(void *args)
{
    int i;
    mln_span_start();
    for (i = 0; i < 10; ++i) {
        cde(i, i + 1);
    }

    mln_span_stop();
    mln_span_dump();
    mln_span_release();
    return NULL;
}

int main(void)
{
    int i;
    pthread_t pth;


    pthread_create(&pth, NULL, pentry, NULL);

    for (i = 0; i < 10; ++i) {
        bcd(i, i + 1);
    }

    pthread_join(pth, NULL);

    return 0;
}
```

编译程序：

```bash
cc -o a a.c -I /usr/local/melon/include/ -L /usr/local/melon/lib/ -lmelon -DMLN_FUNC_FLAG -lpthread
```

执行后可以看到如下输出：

```
| pentry at a.c:20 takes 92 (us)
  | cde at a.c:13 takes 4 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
  | cde at a.c:13 takes 5 (us)
    | bcd at a.c:9 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
    | bcd at a.c:9 takes 2 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
  | cde at a.c:13 takes 24 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
    | bcd at a.c:9 takes 21 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
  | cde at a.c:13 takes 5 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
  | cde at a.c:13 takes 3 (us)
    | bcd at a.c:9 takes 2 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
  | cde at a.c:13 takes 30 (us)
    | bcd at a.c:9 takes 24 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 1 (us)
    | bcd at a.c:9 takes 6 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
  | cde at a.c:13 takes 3 (us)
    | bcd at a.c:9 takes 2 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
  | cde at a.c:13 takes 3 (us)
    | bcd at a.c:9 takes 2 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 1 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
  | cde at a.c:13 takes 7 (us)
    | bcd at a.c:9 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
    | bcd at a.c:9 takes 2 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 1 (us)
  | cde at a.c:13 takes 3 (us)
    | bcd at a.c:9 takes 2 (us)
      | abc at a.c:5 takes 1 (us)
      | abc at a.c:5 takes 0 (us)
    | bcd at a.c:9 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
      | abc at a.c:5 takes 0 (us)
```

