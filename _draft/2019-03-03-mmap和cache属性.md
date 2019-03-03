---
tags : [ Linux , syscall , cache ]
category : [ TECH ]
---

MAP_SHARED 和 MAP_PRIVATE 对 mmap 的内存的 cache 属性的影响
========================================================

先说现象：在 xilinx 的 zynq mpsoc 的 A53 上映射一段内存，并填充一段数据，再从 R5 上读取该段内存，R5 上将该段内存设置为 nocache，A53 上使用 Linux 的默认设置。如果 mmap 的 flag 设置为 MAP_PRIVATE ，则 R5 读出的还是内存旧值，换成 MAP_SHARED 则 R5 读出的是 A53 写入的新值。

mmap 作为系统调用，其实现分为两部分：libc 库的接口，和 Linux Kernel 的实际操作。

1. 接口实现，以 musl(`v1.1.9`) 作为 libc 库为例。其接口函数定义如下：

`src/mman/mmap.c`:

```
void *__mmap(void *start, size_t len, int prot, int flags, int fd, off_t off)
{
    long ret;
    if (off & OFF_MASK) {
        errno = EINVAL;
        return MAP_FAILED;
    }
    if (len >= PTRDIFF_MAX) {
        errno = ENOMEM;
        return MAP_FAILED;
    }
    if (flags & MAP_FIXED) {
        __vm_wait();
    }
#ifdef SYS_mmap2
    ret = __syscall(SYS_mmap2, start, len, prot, flags, fd, off/UNIT);
#else
    ret = __syscall(SYS_mmap, start, len, prot, flags, fd, off);
#endif
    /* Fixup incorrect EPERM from kernel. */
    if (ret == -EPERM && !start && (flags&MAP_ANON) && !(flags&MAP_FIXED))
        ret = -ENOMEM;
    return (void *)__syscall_ret(ret);
}

weak_alias(__mmap, mmap);

```

`src/internal/syscall.h`:
```
#ifdef SYSCALL_NO_INLINE
#define __syscall0(n) (__syscall)(n)
#define __syscall1(n,a) (__syscall)(n,__scc(a))
#define __syscall2(n,a,b) (__syscall)(n,__scc(a),__scc(b))
#define __syscall3(n,a,b,c) (__syscall)(n,__scc(a),__scc(b),__scc(c))
#define __syscall4(n,a,b,c,d) (__syscall)(n,__scc(a),__scc(b),__scc(c),__scc(d))
#define __syscall5(n,a,b,c,d,e) (__syscall)(n,__scc(a),__scc(b),__scc(c),__scc(d),__scc(e))
#define __syscall6(n,a,b,c,d,e,f) (__syscall)(n,__scc(a),__scc(b),__scc(c),__scc(d),__scc(e),__scc(f))
#else
#define __syscall1(n,a) __syscall1(n,__scc(a))
#define __syscall2(n,a,b) __syscall2(n,__scc(a),__scc(b))
#define __syscall3(n,a,b,c) __syscall3(n,__scc(a),__scc(b),__scc(c))
#define __syscall4(n,a,b,c,d) __syscall4(n,__scc(a),__scc(b),__scc(c),__scc(d))
#define __syscall5(n,a,b,c,d,e) __syscall5(n,__scc(a),__scc(b),__scc(c),__scc(d),__scc(e))
#define __syscall6(n,a,b,c,d,e,f) __syscall6(n,__scc(a),__scc(b),__scc(c),__scc(d),__scc(e),__scc(f))
#endif
#define __syscall7(n,a,b,c,d,e,f,g) (__syscall)(n,__scc(a),__scc(b),__scc(c),__scc(d),__scc(e),__scc(f),__scc(g))
    
#define __SYSCALL_NARGS_X(a,b,c,d,e,f,g,h,n,...) n
#define __SYSCALL_NARGS(...) __SYSCALL_NARGS_X(__VA_ARGS__,7,6,5,4,3,2,1,0,)
#define __SYSCALL_CONCAT_X(a,b) a##b
#define __SYSCALL_CONCAT(a,b) __SYSCALL_CONCAT_X(a,b)
#define __SYSCALL_DISP(b,...) __SYSCALL_CONCAT(b,__SYSCALL_NARGS(__VA_ARGS__))(__VA_ARGS__)

#define __syscall(...) __SYSCALL_DISP(__syscall,__VA_ARGS__)
#define syscall(...) __syscall_ret(__syscall(__VA_ARGS__))
```

`arch/arm/syscall_arch.h`
```
static inline long __syscall6(long n, long a, long b, long c, long d, long e, long f)
{
    register long r7 __ASM____R7__ = n;
    register long r0 __asm__("r0") = a;
    register long r1 __asm__("r1") = b;
    register long r2 __asm__("r2") = c;
    register long r3 __asm__("r3") = d;
    register long r4 __asm__("r4") = e;
    register long r5 __asm__("r5") = f;
    __asm_syscall(R7_OPERAND, "0"(r0), "r"(r1), "r"(r2), "r"(r3), "r"(r4), "r"(r5));
}
```

`arch/arm/syscall_arch.h`
```
#define __ASM____R7__ __asm__("r7")
#define __asm_syscall(...) do { \
    __asm__ __volatile__ ( "svc 0" \
    : "=r"(r0) : __VA_ARGS__ : "memory"); \
    return r0; \
    } while (0)
```

展开后，`mmap` 实际调用的函数就是 `__syscall6(SYS_mmap, start, len, prot, flags, fd, off)`，`SYS_mmap` 作为第一个参数传给了 `__syscall6`,最终通过 `svc 0` 进入系统调用，kernel 会根据传入的参数决定执行那个系统调用。

2. 系统调用在 kernel 的实现

通过 `svc` 切换到内核态后，kernel 开始检查传入的参数，选择对应的系统调用。

``
```

```
