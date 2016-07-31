---
tags : u-boot 
category : [ 源码 ]
---

uboot shell 命令
=================

## 4. uboot shell 命令的实现

### 4.1. 添加命令

以命令 `boot` 为例（`cmd/bootm.c`），添加该命令使用了下面的语句：

```
U_BOOT_CMD(
    boot,   1,  1,  do_bootd,
    "boot default, i.e., run 'bootcmd'",
    ""
);
```

这段语句可以这么理解：给 uboot 添加了一条 shell 命令 `boot`，它的作用是引导、启动操作系统(`boot default, i.e., run 'bootcmd'`)，实现命令的函数是 `do_bootd`。

之所以这么一条语句就可以完成添加 shell 命令，可以参考 U_BOOT_CMD 的实现：

```
#define U_BOOT_CMD(_name, _maxargs, _rep, _cmd, _usage, _help)      \
    U_BOOT_CMD_COMPLETE(_name, _maxargs, _rep, _cmd, _usage, _help, NULL)
```

```
#define U_BOOT_CMD_COMPLETE(_name, _maxargs, _rep, _cmd, _usage, _help, _comp) \
    ll_entry_declare(cmd_tbl_t, _name, cmd) =           \
        U_BOOT_CMD_MKENT_COMPLETE(_name, _maxargs, _rep, _cmd,  \
                        _usage, _help, _comp);
```

```
#define ll_entry_declare(_type, _name, _list)               \
    _type _u_boot_list_2_##_list##_2_##_name __aligned(4)       \
            __attribute__((unused,              \
            section(".u_boot_list_2_"#_list"_2_"#_name)))
```

```
#define U_BOOT_CMD_MKENT_COMPLETE(_name, _maxargs, _rep, _cmd,      \
                _usage, _help, _comp)           \
        { #_name, _maxargs, _rep, _cmd, _usage,         \
            _CMD_HELP(_help) _CMD_COMPLETE(_comp) }
```

```
# define _CMD_HELP(x) x,
# define _CMD_COMPLETE(x) x,
```

通过逐层解析宏 `U_BOOT_CMD`，最终会得到：

```
cmd_tbl_t _u_boot_list_2_cmd_2_boot __aligned(4) __attribute__((unused, section(".u_boot_list_2_cmd_2_boot"))) = {
        `boot`,
        1,
        1,
        do_bootd,
        "boot default, i.e., run 'bootcmd'",
        "",
        };


```

在 u-boot.lds 中会链接 .u_boot_list_2_cmd_2_boot ：

```
. = ALIGN(4);
.u_boot_list : {
 KEEP(*(SORT(.u_boot_list*)));
}
```

注：其中 SORT 会按照名称的的顺序进行链接。

cmd_tbl_t 定义如下(成员变量 complete 暂不讨论)：

```
typedef struct cmd_tbl_s    cmd_tbl_t;
struct cmd_tbl_s {
    char        *name;      /* Command Name         */
    int         maxargs;    /* maximum number of arguments  */
    int         repeatable; /* autorepeat allowed?      */
                    /* Implementation function  */
    int         (*cmd)(struct cmd_tbl_s *, int, int, char * const []);
    char        *usage;     /* Usage message    (short) */
#ifdef  CONFIG_SYS_LONGHELP
    char        *help;      /* Help  message    (long)  */
#endif
#ifdef CONFIG_AUTO_COMPLETE
    /* do auto completion on the arguments */
    int     (*complete)(int argc, char * const argv[], char last_char, int maxv, char *cmdv[]);
#endif
};
```

综上，  `U_BOOT_CMD(...)` 实际上是定义了一个结构体变量，这个结构体定义了 uboot 的 shell 命令要用到的信息，并且这个结构体变量是保存在指定的位置（`section(".u_boot_list_2_cmd_2_boot")`)。 uboot 运行时会主动在该 section 寻找命令并执行。


### 4.2. 执行命令

uboot 的 shell 入口是 `common/board_r.c` 的 run_main_loop() :

```
static int run_main_loop(void)
{
#ifdef CONFIG_SANDBOX
    sandbox_main_loop_init();
#endif
    /* main_loop() can return to retry autoboot, if so just run it again */
    for (;;)
        main_loop();
    return 0;
}
```

`main_loop()` 会重复执行，处理输入的命令

`common/main.c` 的 main_loop ：
```
/* We come here after U-Boot is initialised and ready to process commands */
void main_loop(void)
{
    const char *s;

...
    cli_init();
...
    s = bootdelay_process();
    if (cli_process_fdt(&s))
        cli_secure_boot_cmd(s);

    autoboot_command(s);

    cli_loop();
...
}
```

其中 `cli_loop()` 是执行命令的具体函数 ：

```
void cli_loop(void)
{
...
    parse_file_outer();
    /* This point is never reached */
    for (;;);
...
}
```

`parse_file_outer()` 会调用 `parse_stream_outer()` 解析输入的命令并调用 run_list() 执行命令 :

```
static int parse_stream_outer(struct in_str *inp, int flag)
{
...
    do {
        ...
            run_list(ctx.list_head);
...
    /* loop on syntax errors, return on EOF */
    } while (rcode != -1 && !(flag & FLAG_EXIT_FROM_LOOP) &&
        (inp->peek != static_peek || b_peek(inp)));
...
}
```

接着顺着函数调用链 `run_list() -> run_list_real() -> run_pipe_real() -> cmd_process()` uboot 进入到 cmd_process() 开始准备执行命令。

```
enum command_ret_t cmd_process(int flag, int argc, char * const argv[],
                   int *repeatable, ulong *ticks)
{
    enum command_ret_t rc = CMD_RET_SUCCESS;
    cmd_tbl_t *cmdtp;

    /* Look up command in command table */
    cmdtp = find_cmd(argv[0]);
    if (cmdtp == NULL) {
        printf("Unknown command '%s' - try 'help'\n", argv[0]);
        return 1;
    }

    /* found - check max args */
    if (argc > cmdtp->maxargs)
        rc = CMD_RET_USAGE;

#if defined(CONFIG_CMD_BOOTD)
    /* avoid "bootd" recursion */
    else if (cmdtp->cmd == do_bootd) {
        if (flag & CMD_FLAG_BOOTD) {
            puts("'bootd' recursion detected\n");
            rc = CMD_RET_FAILURE;
        } else {
            flag |= CMD_FLAG_BOOTD;
        }
    }
#endif

    /* If OK so far, then do the command */
    if (!rc) {
        if (ticks)
            *ticks = get_timer(0);
        rc = cmd_call(cmdtp, flag, argc, argv);
        if (ticks)
            *ticks = get_timer(*ticks);
        *repeatable &= cmdtp->repeatable;
    }
    if (rc == CMD_RET_USAGE)
        rc = cmd_usage(cmdtp);
    return rc;
}
```

其中 `find_cmd()` 用来在保存命令的 `u_boot_list*` 段内寻找命令对应的结构体变量，然后 `cmd_call()` 调用结构体变量对应的函数，到此命令执行完成。



### 4.3. 命令解析

命令解析有三部分：输入命令、找到命令、执行命令。

#### 4.3.1. 输入命令

uboot shell 的命令都是通过串口输入的，s用户输入字符串后会由 uboot 对字符串进行解析，最终获得命令、命令参数。

#### 4.3.2. 找到命令

通过串口输入获取到命令名称和命令参数后，要在 section .u_boot_list_2_cmd_2* 找到命令的结构体变量，根据结构体变量调用命令背后的函数。

寻找命令的函数是 `cmd_tbl_t *find_cmd()` ，函数返回了对应的命令结构体变量 ：

```
cmd_tbl_t *find_cmd(const char *cmd)
{
    cmd_tbl_t *start = ll_entry_start(cmd_tbl_t, cmd);
    const int len = ll_entry_count(cmd_tbl_t, cmd);
    return find_cmd_tbl(cmd, start, len);
}
```

```
#define ll_entry_start(_type, _list)                    \
({                                  \
    static char start[0] __aligned(4) __attribute__((unused,    \
        section(".u_boot_list_2_"#_list"_1")));         \
    (_type *)&start;                        \
})
```

```
#define ll_entry_count(_type, _list)                    \
    ({                              \
        _type *start = ll_entry_start(_type, _list);        \
        _type *end = ll_entry_end(_type, _list);        \
        unsigned int _ll_result = end - start;          \
        _ll_result;                     \
    })
```

```
#define ll_entry_end(_type, _list)                  \
({                                  \
    static char end[0] __aligned(4) __attribute__((unused,      \
        section(".u_boot_list_2_"#_list"_3")));         \
    (_type *)&end;                          \
})
```

而命令的结构体变量 `.u_boot_list_2_*_2*` 正好位于 `.u_boot_list_2_"#_list"_1"` 和 `.u_boot_list_2_"#_list"_3"` 之间，这样就获取到了 `.u_boot_list_2_*_2*` 的长度，然后调用函数 `find_cmd_tbl()` 遍历该 section 、寻找命令对应的结构体变量并返回给 shell。


```
cmd_tbl_t *find_cmd_tbl(const char *cmd, cmd_tbl_t *table, int table_len)
{
...
    for (cmdtp = table; cmdtp != table + table_len; cmdtp++) {
        if (strncmp(cmd, cmdtp->name, len) == 0) {
            if (len == strlen(cmdtp->name))
                return cmdtp;   /* full match */

            cmdtp_temp = cmdtp; /* abbreviated command ? */
            n_found++;
        }
    }
...
}
```

#### 4.3.3. 执行命令

cmd_process() 调用 cmd_call() 执行命令，很简单就是直接调用命令结构体变量的成员函数 ：

```
static int cmd_call(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
   ...
    result = (cmdtp->cmd)(cmdtp, flag, argc, argv);
   ....
}
```

