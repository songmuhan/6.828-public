# Part 1: Physical Page Management

```
mem_init() -> i386_detect_memory()
```

`i386_detect_memory(): npages + npages_basemem`

`boot_alloc`分配物理地址, round up to PGSIZE



