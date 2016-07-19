---
tags : [ linux , kernel ]
---


kernel 的系统调用现在都是用宏来定义实现的，比如 `sys_socket` ，它的定义不像一般的函数那样定义的：

```
int func(argument)
{
    ...
}
```

而是使用宏实现的：

```
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
    int retval;
    struct socket *sock;
    int flags;

    /* Check the SOCK_* constants for consistency.  */
    BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
    BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
    BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
    BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);

    flags = type & ~SOCK_TYPE_MASK;
    if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
        return -EINVAL;
    type &= SOCK_TYPE_MASK;

    if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
        flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

    retval = sock_create(family, type, protocol, &sock);
    if (retval < 0)
        goto out;

    retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
    if (retval < 0)
        goto out_release;

out:
    /* It may be already another descriptor 8) Not kernel problem. */
    return retval;

out_release:
    sock_release(sock);
    return retval;
}
```

具体而言就是使用宏 `SYSCALL_DEFINE3` 定义了一个系统调用，而 `SYSCALL_DEFINE3` 是在 `include/linux/syscalls.h` 中定义的，和它一样的宏还有多个： 


```
#define SYSCALL_DEFINE0(sname)                  \
    SYSCALL_METADATA(_##sname, 0);              \
    asmlinkage long sys_##sname(void)

#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)
```

其中 `SYSCALL_DEFINE0` 形势上比较特殊，但本质上都一样。宏名中的数字就是系统调用的参数个数。

`SYSCALL_DEFINE*` 展开过程如下：

```
#define SYSCALL_DEFINEx(x, sname, ...)              \
    SYSCALL_METADATA(sname, x, __VA_ARGS__)         \
    __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)

#define __SYSCALL_DEFINEx(x, name, ...)                 \
    asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));  \
    static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));  \
    asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));  \
    asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))   \
    {                               \
        long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));  \
        __MAP(x,__SC_TEST,__VA_ARGS__);             \
        __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));   \
        return ret;                     \
    }                               \
    SYSCALL_ALIAS(sys##name, SyS##name);                \
    static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))

#define SYSCALL_ALIAS(alias, name) asm(         \
    ".globl " VMLINUX_SYMBOL_STR(alias) "\n\t"  \
    ".set   " VMLINUX_SYMBOL_STR(alias) ","     \
          VMLINUX_SYMBOL_STR(name))

```

`__MAP` 的作用是对多个参数对(t,a)分别执行映射函数(m) :

```
#define __MAP0(m,...)
#define __MAP1(m,t,a) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
#define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
#define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
#define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
#define __MAP(n,...) __MAP##n(__VA_ARGS__)
```

`__SC_*` 用来处理参数对：

```
#define __SC_DECL(t, a) t a
#define __TYPE_IS_LL(t) (__same_type((t)0, 0LL) || __same_type((t)0, 0ULL))
#define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a
#define __SC_CAST(t, a) (t) a
#define __SC_ARGS(t, a) a
#define __SC_TEST(t, a) (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(t) && sizeof(t) > sizeof(long))
```

`SYSCALL_DEFINE3(socket...)` 最终展开形式如下：

```
asmlinkage long sys_socket(int family , int type , int protocol );
static inline long SYSC_socket(int family , int type ,int protocol );
asmlinkage long SyS_socket(long family , long type , long protocol ); 
asmlinkage long SyS_socket(long family , long type , long protocol )
{
    long ret = SYSC_socket((long)family ,(long)type ,(long)protocol);
    (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(int) && sizeof(int)>sizeof(long));
    (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(int) && sizeof(int)>sizeof(long));
    (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(int) && sizeof(int)>sizeof(long));
    return ret;
}
asm(".globl sys_socket \n\t" 
".set  sys_socket , SyS_socket");
static inline long SYSC_socket(int family,int type,int protocol)
```

所以最终每个系统调用都是由一个函数本体（`SyS_socket()`）和重命名函数组成（`sys_socket`）

`SYSCALL_DEFINE` 的用法可以总结为：

1. 指出要生成的系统调用的名称（`name`）
2. 声明系统调用的参数个数（0,1,2,3,...6）
3. 给出每个参数的类型和参数名，必须成对出现

比如没有参数的系统调用 `getpid` 的实现就是：

```
SYSCALL_DEFINE0(getpid)
{
    ...
}
```

而需要 2 个参数的 `setpgid` :

```
SYSCALL_DEFINE2(setpgid, pid_t, pid, pid_t, pgid)
```
