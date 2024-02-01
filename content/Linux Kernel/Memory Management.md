---
tags:
  - wiki
---
# Allocation
- small chunk：`kmalloc`、`kmem_cache_alloc`
- large virtually contiguous areas：`vmalloc`
- directly request pages： `alloc_pages`
- specialized allocators：`zs_malloc`...

# Get Free Page flags（GFP）

通用的模式：
```c
kzalloc(<size>, GFP_KERNEL);
```

> 控制