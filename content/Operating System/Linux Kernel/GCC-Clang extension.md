---
tags:
  - wiki
  - C
---

# `__attribute__((format(fn, n, m)))`

> 像 fn 一样检查 bian can 变长参数是否契合
- `n`: format 字符串是第几个参数
- `m`：fn 对应的第一个变长参数是第几个参数
- Linux 提供了一些 wrapper：
	- `__printf(n, m)`：像 `printf` 一样检查参数
	- `__scanf(n, m)`：像 `scanf` 一样检查参数
 - e.g.：
```c
// drivers/android/binder.c

// n: format是第二个参数
// m: format的第一个参数是总体的第三个参数
static __printf(2, 3) void binder_debug(int mask, const char *format, ...)
{
	struct va_format vaf;
	va_list args;

	if (binder_debug_mask & mask) {
		va_start(args, format);
		vaf.va = &args;
		vaf.fmt = format;
		pr_info_ratelimited("%pV", &vaf);
		va_end(args);
	}
}

```