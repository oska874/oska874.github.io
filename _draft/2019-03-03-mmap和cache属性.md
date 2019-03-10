---
tags : [ Linux , syscall , cache ]
category : [ TECH ]
---

MAP_SHARED 和 MAP_PRIVATE 对 mmap 的内存的 cache 属性的影响
========================================================

先说现象：在 xilinx 的 zynq mpsoc 的 A53 上映射一段内存，并填充一段数据，再从 R5 上读取该段内存，R5 上将该段内存设置为 nocache，A53 上使用 Linux 的默认设置。如果 mmap 的 flag 设置为 MAP_PRIVATE ，则 R5 读出的还是内存旧值，换成 MAP_SHARED 则 R5 读出的是 A53 写入的新值。

mmap 作为系统调用，其实现分为两部分：libc 库的接口，和 Linux Kernel 的实际操作。

1. 接口实现，以 musl(`v1.1.21`) 作为 libc 库为例。其接口函数定义如下：

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

`r7` 保存了系统调用号。

`arch/arm/syscall_arch.h`
```
#define __ASM____R7__ __asm__("r7")
#define __asm_syscall(...) do { \
    __asm__ __volatile__ ( "svc 0" \
    : "=r"(r0) : __VA_ARGS__ : "memory"); \
    return r0; \
    } while (0)
```

展开后，`mmap` 实际调用的函数就是 `__syscall6(SYS_mmap, start, len, prot, flags, fd, off)`，`SYS_mmap` 作为第一个参数传给了 `__syscall6`,最终通过 `svc 0` 进入系统调用，kernel 会根据传入的参数决定执行那个系统调用, `SYS_mmap` 即为 `mmap` 的系统调用号。

2. 系统调用在 kernel 的实现

mmap 对应的系统系统调用是 sys_mmap2 ,而不是 sys_mmap。（仅针对 arm，其它arch未关注）

通过 `svc` 切换到内核态后，kernel 开始检查传入的参数，选择对应的系统调用。

`arch/arm/tools/syscall.tbl`
```
90  oabi    mmap            sys_old_mmap
...
192 common  mmap2           sys_mmap2
...
```

`./arch/arm/include/generated/uapi/asm/unistd-common.h`
```
#define __NR_mmap2 (__NR_SYSCALL_BASE + 192)
```

`arch/arm/include/generated/calls-eabi.S`，`arch/arm/include/generated/calls-oabi.S`
```
NATIVE(192, sys_mmap2)
```

`arch/arm/kernel/entry-common.S`
```
.macro  syscall, nr, func
.ifgt   __sys_nr - \nr
.error  "Duplicated/unorded system call entry"
.endif
.rept   \nr - __sys_nr
.long   sys_ni_syscall
.endr
.long   \func
.equ    __sys_nr, \nr + 1
.endm

#define NATIVE(nr, func) syscall nr, func
```


`arch/arm/kernel/entry-common.S`
```
/*
 * Note: off_4k (r5) is always units of 4K.  If we can't do the requested
 * offset, we return EINVAL.
 */
sys_mmap2:
#if PAGE_SHIFT > 12
        tst r5, #PGOFF_MASK
        moveq   r5, r5, lsr #PAGE_SHIFT - 12
        streq   r5, [sp, #4]
        beq sys_mmap_pgoff
        mov r0, #-EINVAL
        ret lr
#else
        str r5, [sp, #4]
        b   sys_mmap_pgoff
#endif
ENDPROC(sys_mmap2)
```



`mm/mmap.c`
```
SYSCALL_DEFINE1(old_mmap, struct mmap_arg_struct __user *, arg) 
{                                                               
    struct mmap_arg_struct a;                                   
                                                                
    if (copy_from_user(&a, arg, sizeof(a)))                     
        return -EFAULT;                                         
    if (offset_in_page(a.offset))                               
        return -EINVAL;                                         
                                                                
    return sys_mmap_pgoff(a.addr, a.len, a.prot, a.flags, a.fd, 
                  a.offset >> PAGE_SHIFT);                      
}             

...

SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len,
        unsigned long, prot, unsigned long, flags,
        unsigned long, fd, unsigned long, pgoff)
{
    struct file *file = NULL;
    unsigned long retval;

    if (!(flags & MAP_ANONYMOUS)) {
        audit_mmap_fd(fd, flags);
        file = fget(fd);
        if (!file)
            return -EBADF;
        if (is_file_hugepages(file))
            len = ALIGN(len, huge_page_size(hstate_file(file)));
        retval = -EINVAL;
        if (unlikely(flags & MAP_HUGETLB && !is_file_hugepages(file)))
            goto out_fput;
    } else if (flags & MAP_HUGETLB) {
        struct user_struct *user = NULL;
        struct hstate *hs;

        hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
        if (!hs)
            return -EINVAL;

        len = ALIGN(len, huge_page_size(hs));
        /*
         * VM_NORESERVE is used because the reservations will be
         * taken when vm_ops->mmap() is called
         * A dummy user value is used because we are not locking
         * memory so no accounting is necessary
         */
        file = hugetlb_file_setup(HUGETLB_ANON_FILE, len,
                VM_NORESERVE,
                &user, HUGETLB_ANONHUGE_INODE,
                (flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
        if (IS_ERR(file))
            return PTR_ERR(file);
    }

    flags &= ~(MAP_EXECUTABLE | MAP_DENYWRITE);


    retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
out_fput:
    if (file)
        fput(file);
    return retval;
}

```


`mm/util.c`
```
unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr, 
    unsigned long len, unsigned long prot,                         
    unsigned long flag, unsigned long pgoff)                       
{                                                                  
    unsigned long ret;                                             
    struct mm_struct *mm = current->mm;                            
    unsigned long populate;                                        
    LIST_HEAD(uf);                                                 
                                                                   
    ret = security_mmap_file(file, prot, flag);                    
    if (!ret) {                                                    
        if (down_write_killable(&mm->mmap_sem))                    
            return -EINTR;                                         
        ret = do_mmap_pgoff(file, addr, len, prot, flag, pgoff,    
                    &populate, &uf);                               
        up_write(&mm->mmap_sem);                                   
        userfaultfd_unmap_complete(mm, &uf);                       
        if (populate)                                              
            mm_populate(ret, populate);                            
    }                                                              
    return ret;                                                    
}                                                                  
```


`include/linux/mm.h`
```
static inline unsigned long                                              
do_mmap_pgoff(struct file *file, unsigned long addr,                     
    unsigned long len, unsigned long prot, unsigned long flags,          
    unsigned long pgoff, unsigned long *populate,                        
    struct list_head *uf)                                                
{                                                                        
    return do_mmap(file, addr, len, prot, flags, 0, pgoff, populate, uf);
}                                                                    
```

`mm/mmap.c`
```
/*                                                                       
 * The caller must hold down_write(&current->mm->mmap_sem).              
 */                                                                      
unsigned long do_mmap(struct file *file, unsigned long addr,             
            unsigned long len, unsigned long prot,                       
            unsigned long flags, vm_flags_t vm_flags,                    
            unsigned long pgoff, unsigned long *populate,                
            struct list_head *uf)                                        
{                                                                        
    struct mm_struct *mm = current->mm;                                  
    int pkey = 0;                                                        
                                                                         
    *populate = 0;                                                       
                                                                         
    if (!len)                                                            
        return -EINVAL;                                                  
                                                                         
    /*                                                                   
     * Does the application expect PROT_READ to imply PROT_EXEC?         
     *                                                                   
     * (the exception is when the underlying filesystem is noexec        
     *  mounted, in which case we dont add PROT_EXEC.)                   
     */                                                                  
    if ((prot & PROT_READ) && (current->personality & READ_IMPLIES_EXEC))
        if (!(file && path_noexec(&file->f_path)))                       
            prot |= PROT_EXEC;                                           
                                                                         
    if (!(flags & MAP_FIXED))                                            
        addr = round_hint_to_min(addr);                                  
                                                                         
    /* Careful about overflows.. */                                      
    len = PAGE_ALIGN(len);                                               
    if (!len)                                                            
        return -ENOMEM;                                                  
                                                                         
    /* offset overflow? */                                               
    if ((pgoff + (len >> PAGE_SHIFT)) < pgoff)                           
        return -EOVERFLOW;                                               
                                                                         
    /* Too many mappings? */                                             
    if (mm->map_count > sysctl_max_map_count)                             
        return -ENOMEM;                                                   
                                                                          
    /* Obtain the address to map to. we verify (or select) it and ensure  
     * that it represents a valid section of the address space.           
     */                                                                   
    addr = get_unmapped_area(file, addr, len, pgoff, flags);              
    if (offset_in_page(addr))                                             
        return addr;                                                      
                                                                          
    if (prot == PROT_EXEC) {                                              
        pkey = execute_only_pkey(mm);                                     
        if (pkey < 0)                                                     
            pkey = 0;                                                     
    }                                                                     
                                                                          
    /* Do simple checking here so the lower-level routines won't have     
     * to. we assume access permissions have been handled by the open     
     * of the memory object, so we don't do any here.                     
     */                                                                   
    vm_flags |= calc_vm_prot_bits(prot, pkey) | calc_vm_flag_bits(flags) |
            mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;        
                                                                          
    if (flags & MAP_LOCKED)                                               
        if (!can_do_mlock())                                              
            return -EPERM;                                                
                                                                          
    if (mlock_future_check(mm, vm_flags, len))                            
        return -EAGAIN;                                                   
                                                                          
    if (file) {                                                           
        struct inode *inode = file_inode(file);                           
                                                                          
        switch (flags & MAP_TYPE) {                                       
        case MAP_SHARED:                                                  
            if ((prot&PROT_WRITE) && !(file->f_mode&FMODE_WRITE))         
                return -EACCES;                                           
                                                                          
            /*                                                            
             * Make sure we don't allow writing to an append-only         
             * file..                                                     
             */                                                   
            if (IS_APPEND(inode) && (file->f_mode & FMODE_WRITE)) 
                return -EACCES;                                   
                                                                  
            /*                                                    
             * Make sure there are no mandatory locks on the file.
             */                                                   
            if (locks_verify_locked(file))                        
                return -EAGAIN;                                   
                                                                  
            vm_flags |= VM_SHARED | VM_MAYSHARE;                  
            if (!(file->f_mode & FMODE_WRITE))                    
                vm_flags &= ~(VM_MAYWRITE | VM_SHARED);           
                                                                  
            /* fall through */                                    
        case MAP_PRIVATE:                                         
            if (!(file->f_mode & FMODE_READ))                     
                return -EACCES;                                   
            if (path_noexec(&file->f_path)) {                     
                if (vm_flags & VM_EXEC)                           
                    return -EPERM;                                
                vm_flags &= ~VM_MAYEXEC;                          
            }                                                     
                                                                  
            if (!file->f_op->mmap)                                
                return -ENODEV;                                   
            if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))             
                return -EINVAL;                                   
            break;                                                
                                                                  
        default:                                                  
            return -EINVAL;                                       
        }                                                         
    } else {                                                      
        switch (flags & MAP_TYPE) {                               
        case MAP_SHARED:                                          
            if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))             
                return -EINVAL;                                   
            /*                                                    
             * Ignore pgoff.                                      
             */                                                   
            pgoff = 0;                                              
            vm_flags |= VM_SHARED | VM_MAYSHARE;                    
            break;                                                  
        case MAP_PRIVATE:                                           
            /*                                                      
             * Set pgoff according to addr for anon_vma.            
             */                                                     
            pgoff = addr >> PAGE_SHIFT;                             
            break;                                                  
        default:                                                    
            return -EINVAL;                                         
        }                                                           
    }                                                               
                                                                    
    /*                                                              
     * Set 'VM_NORESERVE' if we should not account for the          
     * memory use of this mapping.                                  
     */                                                             
    if (flags & MAP_NORESERVE) {                                    
        /* We honor MAP_NORESERVE if allowed to overcommit */       
        if (sysctl_overcommit_memory != OVERCOMMIT_NEVER)           
            vm_flags |= VM_NORESERVE;                               
                                                                    
        /* hugetlb applies strict overcommit unless MAP_NORESERVE */
        if (file && is_file_hugepages(file))                        
            vm_flags |= VM_NORESERVE;                               
    }                                                               
                                                                    
    addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);       
    if (!IS_ERR_VALUE(addr) &&                                      
        ((vm_flags & VM_LOCKED) ||                                  
         (flags & (MAP_POPULATE | MAP_NONBLOCK)) == MAP_POPULATE))  
        *populate = len;                                            
    return addr;                                                    
}                                                                   
```

中断向量的生成：
`start_kernel`（`init/main.c`）->`setup_arch`(arch/arm/kernel/setup.c)->`paging_init`(`arch/arm/mm/mmu.c`)->`devicemaps_init`(`arch/arm/mm/mmu.c`)->`early_trap_init`(`arch/arm/kernel/traps.c`)
中断向量的物理地址不确定，但是 virtual 地址是 0xffff0000.

`arch/arm/mm/mmu.c`

```
/*
 * Set up the device mappings.  Since we clear out the page tables for all
 * mappings above VMALLOC_START, except early fixmap, we might remove debug
 * device mappings.  This means earlycon can be used to debug this function
 * Any other function or debugging method which may touch any device _will_
 * crash the kernel.
 */
static void __init devicemaps_init(const struct machine_desc *mdesc)
{
    struct map_desc map;
    unsigned long addr; 
    void *vectors;

    /*
     * Allocate the vector page early.
     */
    vectors = early_alloc(PAGE_SIZE * 2);

    early_trap_init(vectors);

    /*
     * Clear page table except top pmd used by early fixmaps
     */
    for (addr = VMALLOC_START; addr < (FIXADDR_TOP & PMD_MASK); addr += PMD_SIZE)
        pmd_clear(pmd_off_k(addr));
   
    /*
     * Map the kernel if it is XIP.
     * It is always first in the modulearea.
     */
#ifdef CONFIG_XIP_KERNEL
    map.pfn = __phys_to_pfn(CONFIG_XIP_PHYS_ADDR & SECTION_MASK);
    map.virtual = MODULES_VADDR;
    map.length = ((unsigned long)_exiprom - map.virtual + ~SECTION_MASK) & SECTION_MASK;
    map.type = MT_ROM;
    create_mapping(&map);
#endif 
   
    /*
     * Map the cache flushing regions.
     */
#ifdef FLUSH_BASE
    map.pfn = __phys_to_pfn(FLUSH_BASE_PHYS);
    map.virtual = FLUSH_BASE;
    map.length = SZ_1M;
    map.type = MT_CACHECLEAN;
    create_mapping(&map);
#endif
#ifdef FLUSH_BASE_MINICACHE
    map.pfn = __phys_to_pfn(FLUSH_BASE_PHYS + SZ_1M);
    map.virtual = FLUSH_BASE_MINICACHE;
    map.length = SZ_1M;
    map.type = MT_MINICLEAN;
    create_mapping(&map);
#endif

    /*
     * Create a mapping for the machine vectors at the high-vectors
     * location (0xffff0000).  If we aren't using high-vectors, also
     * create a mapping at the low-vectors virtual address.
     */
    map.pfn = __phys_to_pfn(virt_to_phys(vectors));
    map.virtual = 0xffff0000;
    map.length = PAGE_SIZE;
#ifdef CONFIG_KUSER_HELPERS
    map.type = MT_HIGH_VECTORS;
#else
    map.type = MT_LOW_VECTORS;
#endif
    create_mapping(&map);

    if (!vectors_high()) {
        map.virtual = 0;
        map.length = PAGE_SIZE * 2;
        map.type = MT_LOW_VECTORS;
        create_mapping(&map);
    }

    /* Now create a kernel read-only mapping */
    map.pfn += 1;
    map.virtual = 0xffff0000 + PAGE_SIZE;
    map.length = PAGE_SIZE;
    map.type = MT_LOW_VECTORS;
    create_mapping(&map);

    /*
     * Ask the machine support to map in the statically mapped devices.
     */
    if (mdesc->map_io)
        mdesc->map_io();
    else
        debug_ll_io_init();
    fill_pmd_gaps();

    /* Reserve fixed i/o space in VMALLOC region */
    pci_reserve_io();

    /*
     * Finally flush the caches and tlb to ensure that we're in a
     * consistent state wrt the writebuffer.  This also ensures that
     * any write-allocated cache lines in the vector page are written
     * back.  After this point, we can start to touch devices again.
     */
    local_flush_tlb_all();
    flush_cache_all();

    /* Enable asynchronous aborts */
    early_abt_enable();
}
```

`arch/arm/kernel/vmlinux.lds.S`
```
/*
 * The vectors and stubs are relocatable code, and the
 * only thing that matters is their relative offsets
 */
__vectors_start = .;
.vectors 0xffff0000 : AT(__vectors_start) {
    *(.vectors)
}
. = __vectors_start + SIZEOF(.vectors);
__vectors_end = .;
    
__stubs_start = .;
.stubs ADDR(.vectors) + 0x1000 : AT(__stubs_start) {
    *(.stubs)
}
. = __stubs_start + SIZEOF(.stubs);
__stubs_end = .;
```

定义了系统调用入口
`arch/arm/kernel/entry-common.S`
```
/*=============================================================================
 * SWI handler
 *-----------------------------------------------------------------------------
 */

    .align  5
ENTRY(vector_swi)
#ifdef CONFIG_CPU_V7M
    v7m_exception_entry
#else
    sub sp, sp, #PT_REGS_SIZE
    stmia   sp, {r0 - r12}          @ Calling r0 - r12
 ARM(   add r8, sp, #S_PC       )
 ARM(   stmdb   r8, {sp, lr}^       )   @ Calling sp, lr
 THUMB( mov r8, sp          )
 THUMB( store_user_sp_lr r8, r10, S_SP  )   @ calling sp, lr
    mrs saved_psr, spsr         @ called from non-FIQ mode, so ok.
 TRACE( mov saved_pc, lr        )
    str saved_pc, [sp, #S_PC]       @ Save calling PC
    str saved_psr, [sp, #S_PSR]     @ Save CPSR
    str r0, [sp, #S_OLD_R0]     @ Save OLD_R0
#endif
    zero_fp
    alignment_trap r10, ip, __cr_alignment
    asm_trace_hardirqs_on save=0
    enable_irq_notrace
    ct_user_exit save=0
    /*
     * Get the system call number.
     */

#if defined(CONFIG_OABI_COMPAT)

    /*
     * If we have CONFIG_OABI_COMPAT then we need to look at the swi
     * value to determine if it is an EABI or an old ABI call.
     */
#ifdef CONFIG_ARM_THUMB
    tst saved_psr, #PSR_T_BIT
    movne   r10, #0             @ no thumb OABI emulation
 USER(  ldreq   r10, [saved_pc, #-4]    )   @ get SWI instruction
#else
 USER(  ldr r10, [saved_pc, #-4]    )   @ get SWI instruction
#endif
 ARM_BE8(rev    r10, r10)           @ little endian instruction

#elif defined(CONFIG_AEABI)

    /*
     * Pure EABI user space always put syscall number into scno (r7).
     */
#elif defined(CONFIG_ARM_THUMB)
    /* Legacy ABI only, possibly thumb mode. */
    tst saved_psr, #PSR_T_BIT       @ this is SPSR from save_user_regs
    addne   scno, r7, #__NR_SYSCALL_BASE    @ put OS number in
 USER(  ldreq   scno, [saved_pc, #-4]   )

#else
    /* Legacy ABI only. */
 USER(  ldr scno, [saved_pc, #-4]   )   @ get SWI instruction
#endif

    /* saved_psr and saved_pc are now dead */

    uaccess_disable tbl

    adr tbl, sys_call_table     @ load syscall table pointer

#if defined(CONFIG_OABI_COMPAT)
    /*
     * If the swi argument is zero, this is an EABI call and we do nothing.
     *
     * If this is an old ABI call, get the syscall number into scno and
     * get the old ABI syscall table address.
     */
    bics    r10, r10, #0xff000000
    eorne   scno, r10, #__NR_OABI_SYSCALL_BASE
    ldrne   tbl, =sys_oabi_call_table
#elif !defined(CONFIG_AEABI)
    bic scno, scno, #0xff000000     @ mask off SWI op-code
    eor scno, scno, #__NR_SYSCALL_BASE  @ check OS number
#endif
    get_thread_info tsk
    /*
     * Reload the registers that may have been corrupted on entry to
     * the syscall assembly (by tracing or context tracking.)
     */
 TRACE( ldmia   sp, {r0 - r3}       )

local_restart:
    ldr r10, [tsk, #TI_FLAGS]       @ check for syscall tracing
    stmdb   sp!, {r4, r5}           @ push fifth and sixth args

    tst r10, #_TIF_SYSCALL_WORK     @ are we tracing syscalls?
    bne __sys_trace

    cmp scno, #NR_syscalls      @ check upper syscall limit
    badr    lr, ret_fast_syscall        @ return address
    ldrcc   pc, [tbl, scno, lsl #2]     @ call sys_* routine

    add r1, sp, #S_OFF
2:  cmp scno, #(__ARM_NR_BASE - __NR_SYSCALL_BASE)
    eor r0, scno, #__NR_SYSCALL_BASE    @ put OS number back
    bcs arm_syscall
    mov why, #0             @ no longer a real syscall
    b   sys_ni_syscall          @ not private func

#if defined(CONFIG_OABI_COMPAT) || !defined(CONFIG_AEABI)
    /*
     * We failed to handle a fault trying to access the page
     * containing the swi instruction, but we're not really in a
     * position to return -EFAULT. Instead, return back to the
     * instruction and re-enter the user fault handling path trying
     * to page it in. This will likely result in sending SEGV to the
     * current task.
     */
9001:
    sub lr, saved_pc, #4
    str lr, [sp, #S_PC]
    get_thread_info tsk
    b   ret_fast_syscall
#endif
ENDPROC(vector_swi)
```

`arch/arm/kernel/entry-common.S` 定义了系统调用表
```
/*
 * This is the syscall table declaration for native ABI syscalls.
 * With EABI a couple syscalls are obsolete and defined as sys_ni_syscall.
 */
    syscall_table_start sys_call_table
#define COMPAT(nr, native, compat) syscall nr, native
#ifdef CONFIG_AEABI
#include <calls-eabi.S>
#else
#include <calls-oabi.S>
#endif
#undef COMPAT
    syscall_table_end sys_call_table

.macro  syscall, nr, func
.ifgt   __sys_nr - \nr
.error  "Duplicated/unorded system call entry"
.endif
.rept   \nr - __sys_nr
.long   sys_ni_syscall
.endr
.long   \func
.equ    __sys_nr, \nr + 1
.endm

.macro  syscall_table_end, sym
.ifgt   __sys_nr - __NR_syscalls
.error  "System call table too big"
.endif
.rept   __NR_syscalls - __sys_nr
.long   sys_ni_syscall
.endr
.size   \sym, . - \sym
.endm
```

流程：
库函数发起系统调用，将系统调用号和参数传给 svc 中断处理函数，然后跳转到 syscall table 的具体系统调用函数。
syscall table 是在汇编中生成的，使用了从 syscall.tbl 生成的 call-eabi.S/call-oabi.S 表。


