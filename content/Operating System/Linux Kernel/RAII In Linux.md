---
tags:
  - C
  - wiki
---

> `include/linux/cleanup.h`

RAII 是使用 scope 作为自动释放资源时机的机制，广泛应用在 C++、Rust 这样的高级语言中。C 缺乏相应的析构函数的机制来做这件事情。

GCC 扩展是实现这一机制的方法，绕开 C 标准的 GCC 扩展本质上不仅适用于 GCC，Clang 的支持也是同步的，对于 Linux Kernel 来说，GCC/Clang 就已经能够完全满足需求了。

Linux 采用宏提供了一系列的 RAII 式的 API（Linux 6.5+），在 Mutex、fd 子模块中首先开始推广。

# Cleanup functions

首先需要在 C 中提供析构函数的抽象，这是由 GCC 扩展 `__attribute__((__cleanup(func)))` 实现的，Linux 提供了一个简单的包装：

```c
#define DEFINE_FREE(name, type, free) \
	static inline void __free_##name(void *p) { type _T = *(type *)p; free;}

#define __free(name)	__cleanup(__free_##name) 
```

但是，有时候我们想要把一个变量 move 出去，而非在当前作用域结束就 free 掉，这时候就可以采用下面的 API：

```c
#define no_free_ptr(p) \
	({ __auto_type __ptr = (p); (p) = NULL; __ptr; })

#define return_ptr(p)	return no_free_ptr(p)
```

例子：

```c
DEFINE_FREE(kfree, void *, if (_T) kfree(_T))
struct obj *p __free(kfree) = kmalloc(...);
  
if (!p)
	return NULL;
if (!initialize_obj(p))
	return NULL;
return_ptr(p);
```

# From cleanup to class

直接使用 cleanup 的操作有时候非常容易出错，比如 `return_ptr` 的返回值没有被标注 `__free` 就容易导致资源泄漏。进而 Linux 提供了更高级的封装—— `Class`:

```c
#define DEFINE_CLASS(name, type, exit, init, init_args...)		\
        typedef type class_##name##_t;					        \
	static inline void class_##name##_destructor(type *p)		\
		{ type _T = *p; exit; }					                \
	static inline type class_##name##_constructor(init_args)	\
	    { type t = init; return t; }
```

class 用来表达**资源的所有者**，负责资源的申请和释放，使用 `DEFINE_CLASS` 会自动生成构造函数和析构函数。接下来使用 `CLASS` 宏构造变量：

```c
#define CLASS(name, var)						                \
	class_##name##_t var __cleanup(class_##name##_destructor) =	\
		class_##name##_constructor
```

例子：

```c
// create struct fd's manager
DEFINE_CLASS(fdget, struct fd, fdput(_T), fdget(fd), int fd)

// create manager's instance
CLASS(fdget, f)(fd);
```

上述的宏展开得到：

```c
// create struct fd's manager
typedef struct fd class_fd_t;
static inline __attribute__((__gnu_inline__)) __attribute__((__unused__))
__attribute__((__no_instrument_function__)) void
class_fd_destructor(struct fd *p)
{
	struct fd _T = *p;
	fdput(_T);
}
static inline __attribute__((__gnu_inline__)) __attribute__((__unused__))
__attribute__((__no_instrument_function__)) struct fd
class_fd_constructor(int fd)
{
	struct fd t = fdget(fd);
	return t;
}

// create manager's instance
class_fd_t f __attribute__((__unused__))
__attribute__((__cleanup__(class_fd_destructor))) = class_fd_constructor(fd)
```

# Lock guard

这一机制在 Mutex 中非常实用，因为可能在不同的情况下产生错误，现有的 Linux 代码都采用 goto 来释放资源，则显得略显繁杂：

```c
err = -EBUMMER;
mutex_lock(&the_lock);
if (!setup_first_thing())
   goto out;
if (!setup_second_thing())
   goto out2;
/* ... */
out2:
	cleanup_first_thing();
out:
	mutex_unlock(&the_lock);
	return err;
```

在采用了新的 RAII 机制之后，可以使这一结构变得非常简洁：

```c
guard(mutex)(&the_lock);
CLASS(first_thing, first)(...);
if (!first or !setup_second_thing())
	return -EBUMMER;
return 0;
```

Locking 模块提供了以下的 API：

```c
#define DEFINE_GUARD(name, type, lock, unlock) \
	DEFINE_CLASS(name, type, unlock, ({ lock; _T; }), type _T)
#define guard(name) \
	CLASS(name, __UNIQUE_ID(guard))
```

进而 mutex 可以定义为：

```c
DEFINE_GUARD(mutex, struct mutex *, mutex_lock(_T), mutex_unlock(_T))

// Expands to
typedef struct mutex *class_mutex_t;
static inline __attribute__((__gnu_inline__)) __attribute__((__unused__))
__attribute__((no_instrument_function)) void
class_mutex_destructor(struct mutex **p)
{
	struct mutex *_T = *p;
	mutex_unlock(_T);
}
static inline __attribute__((__gnu_inline__)) __attribute__((__unused__))
__attribute__((no_instrument_function)) struct mutex *
class_mutex_constructor(struct mutex *_T)
{
	struct mutex *t = ({
		mutex_lock(_T);
		_T;
	});
	return t;
}
```

使用的方式也非常简单：

```c
// before
mutex_lock(&uclamp_mutex);

// new
guard(mutex)(&uclamp_mutex);
```

# What's Next
[libcsptr](https://github.com/Snaipe/libcsptr/blob/master)