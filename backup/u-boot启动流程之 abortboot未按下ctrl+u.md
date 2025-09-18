## 1、起初在cli.c调用

```
|-->auto_net_upgrade(u-boot/common/autoboot.c)
|-->run_command_list(u-boot/common/cli.c)
    |-->parse_string_outer(u-boot/common/cli_hush.c)
        |-->setup_string_in_str(u-boot/common/cli_hush.c)
        |-->parse_stream_outer(u-boot/common/cli_hush.c)
            |-->initialize_context(u-boot/common/cli_hush.c)
            |-->update_ifs_map(u-boot/common/cli_hush.c)
            |-->parse_stream(u-boot/common/cli_hush.c)
            |-->run_list(u-boot/common/cli_hush.c)
                |-->run_list_real(u-boot/common/cli_hush.c)
                    |-->make_list_in(u-boot/common/cli_hush.c)
                    |-->run_pipe_real(u-boot/common/cli_hush.c)
                        |-->cmd_process(u-boot/common/command.c)
                            |-->find_cmd(u-boot/common/command.c)
                            |-->cmd_call(u-boot/common/command.c)
                                |-->do_bootz(u-boot/common/cmd_bootm.c)
                                    |-->
                |-->free_pipe_list(u-boot/common/cli_hush.c)
```

## 2、进入cli_hush.c中的parse_string_outer

### parse_string_outer函数定义

```
int parse_string_outer(const char *s, int flag)
{
    struct in_str input;
    char *p = NULL;
    int rcode;
    if (!s)
        return 1;
    if (!*s)
        return 0;
    if (!(p = strchr(s, '\n')) || *++p) {
        p = xmalloc(strlen(s) + 2);
        strcpy(p, s);
        strcat(p, "\n");
        setup_string_in_str(&input, p);
        rcode = parse_stream_outer(&input, flag);
        free(p);
        return rcode;
    } else {
    setup_string_in_str(&input, s);
    return parse_stream_outer(&input, flag);
    }
}
```

### parse_stream_outer函数定义

```
static int parse_stream_outer(struct in_str *inp, int flag)
{
    struct p_context ctx;
    o_string temp=NULL_O_STRING;
    int rcode;
    int code = 1;
    do {
        ctx.type = flag;
        initialize_context(&ctx);
        update_ifs_map();
        if (!(flag & FLAG_PARSE_SEMICOLON) || (flag & FLAG_REPARSING)) mapset((uchar *)";$&|", 0);
        inp->promptmode=1;
        rcode = parse_stream(&temp, &ctx, inp,
                     flag & FLAG_CONT_ON_NEWLINE ? -1 : '\n');
        if (rcode == 1) flag_repeat = 0;
        if (rcode != 1 && ctx.old_flag != 0) {
            syntax();
            flag_repeat = 0;
        }
        if (rcode != 1 && ctx.old_flag == 0) {
            done_word(&temp, &ctx);
            done_pipe(&ctx,PIPE_SEQ);
            code = run_list(ctx.list_head);
            if (code == -2) {   /* exit */
                b_free(&temp);
                code = 0;
                /* XXX hackish way to not allow exit from main loop */
                if (inp->peek == file_peek) {
                    printf("exit not allowed from main input shell.\n");
                    continue;
                }
                break;
            }
            if (code == -1)
                flag_repeat = 0;
        } else {
            if (ctx.old_flag != 0) {
                free(ctx.stack);
                b_reset(&temp);
            }
            if (inp->__promptme == 0) printf("<INTERRUPT>\n");
            inp->__promptme = 1;
            temp.nonnull = 0;
            temp.quote = 0;
            inp->p = NULL;
            free_pipe_list(ctx.list_head,0);
        }
        b_free(&temp);
    /* loop on syntax errors, return on EOF */
    } while (rcode != -1 && !(flag & FLAG_EXIT_FROM_LOOP) &&
        (inp->peek != static_peek || b_peek(inp)));
    return (code != 0) ? 1 : 0;
}
```



### run_list_real函数定义

```
static int run_list_real(struct pipe *pi)
{
    char *save_name = NULL;
    char **list = NULL;
    char **save_list = NULL;
    struct pipe *rpipe;
    int flag_rep = 0;
    int rcode=0, flag_skip=1;
    int flag_restore = 0;
    int if_code=0, next_if_code=0;  /* need double-buffer to handle elif */
    reserved_style rmode, skip_more_in_this_rmode=RES_XXXX;
    /* check syntax for "for" */
    for (rpipe = pi; rpipe; rpipe = rpipe->next) {
        if ((rpipe->r_mode == RES_IN ||
            rpipe->r_mode == RES_FOR) &&
            (rpipe->next == NULL)) {
                syntax();
#ifdef __U_BOOT__
                flag_repeat = 0;
#endif
                return 1;
        }
        if ((rpipe->r_mode == RES_IN &&
            (rpipe->next->r_mode == RES_IN &&
            rpipe->next->progs->argv != NULL))||
            (rpipe->r_mode == RES_FOR &&
            rpipe->next->r_mode != RES_IN)) {
                syntax();
#ifdef __U_BOOT__
                flag_repeat = 0;
#endif
                return 1;
        }
    }
    for (; pi; pi = (flag_restore != 0) ? rpipe : pi->next) {
        if (pi->r_mode == RES_WHILE || pi->r_mode == RES_UNTIL ||
            pi->r_mode == RES_FOR) {
#ifdef __U_BOOT__
                /* check Ctrl-C */
                ctrlc();
                if ((had_ctrlc())) {
                    return 1;
                }
#endif
                flag_restore = 0;
                if (!rpipe) {
                    flag_rep = 0;
                    rpipe = pi;
                }
        }
        rmode = pi->r_mode;
        debug_printf("rmode=%d  if_code=%d  next_if_code=%d skip_more=%d\n", rmode, if_code, next_if_code, skip_more_in_this_rmode);
        if (rmode == skip_more_in_this_rmode && flag_skip) {
            if (pi->followup == PIPE_SEQ) flag_skip=0;
            continue;
        }
        flag_skip = 1;
        skip_more_in_this_rmode = RES_XXXX;
        if (rmode == RES_THEN || rmode == RES_ELSE) if_code = next_if_code;
        if (rmode == RES_THEN &&  if_code) continue;
        if (rmode == RES_ELSE && !if_code) continue;
        if (rmode == RES_ELIF && !if_code) break;
        if (rmode == RES_FOR && pi->num_progs) {
            if (!list) {
                /* if no variable values after "in" we skip "for" */
                if (!pi->next->progs->argv) continue;
                /* create list of variable values */
                list = make_list_in(pi->next->progs->argv,
                    pi->progs->argv[0]);
                save_list = list;
                save_name = pi->progs->argv[0];
                pi->progs->argv[0] = NULL;
                flag_rep = 1;
            }
            if (!(*list)) {
                free(pi->progs->argv[0]);
                free(save_list);
                list = NULL;
                flag_rep = 0;
                pi->progs->argv[0] = save_name;
                continue;
            } else {
                /* insert new value from list for variable */
                if (pi->progs->argv[0])
                    free(pi->progs->argv[0]);
                pi->progs->argv[0] = *list++;
            }
        }
        if (rmode == RES_IN) continue;
        if (rmode == RES_DO) {
            if (!flag_rep) continue;
        }
        if (rmode == RES_DONE) {
            if (flag_rep) {
                flag_restore = 1;
            } else {
                rpipe = NULL;
            }
        }
        if (pi->num_progs == 0) continue;
        rcode = run_pipe_real(pi);
        debug_printf("run_pipe_real returned %d\n",rcode);
        if (rcode < -1) {
            last_return_code = -rcode - 2;
            return -2;  /* exit */
        }
        last_return_code=(rcode == 0) ? 0 : 1;
        if ( rmode == RES_IF || rmode == RES_ELIF )
            next_if_code=rcode;  /* can be overwritten a number of times */
        if (rmode == RES_WHILE)
            flag_rep = !last_return_code;
        if (rmode == RES_UNTIL)
            flag_rep = last_return_code;
        if ( (rcode==EXIT_SUCCESS && pi->followup==PIPE_OR) ||
             (rcode!=EXIT_SUCCESS && pi->followup==PIPE_AND) )
            skip_more_in_this_rmode=rmode;
    }
    return rcode;
}
```



### run_pipe_real函数实际运行部分：

```
static int run_pipe_real(struct pipe *pi)
{
    int i;
    int nextin;
    int flag = do_repeat ? CMD_FLAG_REPEAT : 0;
    struct child_prog *child;
    char *p;
    nextin = 0;
    if (pi->num_progs == 1) child = & (pi->progs[0]);
     
    if (pi->num_progs == 1 && pi->progs[0].argv != NULL) {
        for (i=0; is_assignment(child->argv[i]); i++) { /* nothing */ }
        for (i = 0; is_assignment(child->argv[i]); i++) {
            p = insert_var_value(child->argv[i]);
            set_local_var(p, 0);
            if (p != child->argv[i]) {
                child->sp--;
                free(p);
            }
        }
        /* Process the command */
        return cmd_process(flag, child->argc, child->argv,
                   &flag_repeat, NULL);
    }
}
```

## 3、进入到command.c中的cmd_process

### cmd_process 函数实际运行部分

```
enum command_ret_t cmd_process(int flag, int argc, char * const argv[],
                   int *repeatable, ulong *ticks)
{
    enum command_ret_t rc = CMD_RET_SUCCESS;
    cmd_tbl_t *cmdtp;
    /* Look up command in command table */
    cmdtp = find_cmd(argv[0]);                  //初始的cmd也就是argv[0]在经过find_cmd函数后将得到cmd相关的函数；其过程暂不清楚
                                                //例：此处的argv[0]是booz，经过find_cmd后得到cmdtp->cmd为do_bootz
    /* found - check max args */
    if (argc > cmdtp->maxargs)
        rc = CMD_RET_USAGE;
    /* avoid "bootd" recursion */
    if (cmdtp->cmd == do_bootd) {
        if (flag & CMD_FLAG_BOOTD) {
            puts("'bootd' recursion detected\n");
            rc = CMD_RET_FAILURE;
        } else {
            flag |= CMD_FLAG_BOOTD;
        }
    }
    /* If OK so far, then do the command */
    if (!rc) {
        if (ticks)
            *ticks = get_timer(0);
        rc = cmd_call(cmdtp, flag, argc, argv); //cmd相关的函数do_bootz是在此函数中执行的
        if (ticks)
            *ticks = get_timer(*ticks);
        *repeatable &= cmdtp->repeatable;
    }
    if (rc == CMD_RET_USAGE)
        rc = cmd_usage(cmdtp);
    return rc;
}
```

### find_cmd函数实际执行：

```
#define ll_entry_start(_type, _list)                    \
({                                  \
    static char start[0] __aligned(4) __attribute__((unused,    \
        section(".u_boot_list_2_"#_list"_1")));         \
    (_type *)&start;                        \
})
 
#define ll_entry_end(_type, _list)                  \
({                                  \
    static char end[0] __aligned(4) __attribute__((unused,  \
        section(".u_boot_list_2_"#_list"_3")));         \
    (_type *)&end;                          \
})
 
#define ll_entry_count(_type, _list)                    \
    ({                              \
        _type *start = ll_entry_start(_type, _list);        \
        _type *end = ll_entry_end(_type, _list);        \
        unsigned int _ll_result = end - start;          \
        _ll_result;                     \
    })
 
cmd_tbl_t *find_cmd_tbl(const char *cmd, cmd_tbl_t *table, int table_len)
{
    cmd_tbl_t *cmdtp;
    cmd_tbl_t *cmdtp_temp = table;  /* Init value */
    const char *p;
    int len;
    int n_found = 0;
    if (!cmd)
        return NULL;
    /*
     * Some commands allow length modifiers (like "cp.b");
     * compare command name only until first dot.
     */
    len = ((p = strchr(cmd, '.')) == NULL) ? strlen (cmd) : (p - cmd);
    for (cmdtp = table; cmdtp != table + table_len; cmdtp++) {
        if (strncmp(cmd, cmdtp->name, len) == 0) {
            if (len == strlen(cmdtp->name))
                return cmdtp;   /* full match */
            cmdtp_temp = cmdtp; /* abbreviated command ? */
            n_found++;
        }
    }
    if (n_found == 1) {         /* exactly one match */
        return cmdtp_temp;
    }
    return NULL;    /* not found or ambiguous command */
}
 
cmd_tbl_t *find_cmd(const char *cmd)
{
    cmd_tbl_t *start = ll_entry_start(cmd_tbl_t, cmd);
    const int len = ll_entry_count(cmd_tbl_t, cmd);
    return find_cmd_tbl(cmd, start, len);
}
```

### cmd_call函数实际执行：

```
static int cmd_call(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
    int result;
    result = (cmdtp->cmd)(cmdtp, flag, argc, argv);      //do_bootz函数在此调用执行
    if (result)
        debug("Command failed, result=%d", result);
    return result;
}
```

## 4、进入到cmd_bootm函数中的do_bootz

### do_bootz函数实际执行：

```
int do_bootz(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
    int ret;
    /* Consume 'bootz' 此时的argc是4，表示4个参数，同时argv[0]是bootz*/  
    argc--; argv++;     //此时argc是3，还有3个参数，同时argv[0]是bootz引导内核的地址0x2080000
    if (bootz_start(cmdtp, flag, argc, argv, &images))          //images当前还只是个空的结构体实例
        return 1;
    /*
     * We are doing the BOOTM_STATE_LOADOS state ourselves, so must
     * disable interrupts ourselves
     */
    bootm_disable_interrupts();
    images.os.os = IH_OS_LINUX;
    ret = do_bootm_states(cmdtp, flag, argc, argv,
                  BOOTM_STATE_OS_PREP | BOOTM_STATE_OS_FAKE_GO |
                  BOOTM_STATE_OS_GO,
                  &images, 1);
    return ret;
}
```

### 1、bootz_start函数实际执行：

```
static int bootz_start(cmd_tbl_t *cmdtp, int flag, int argc,
            char * const argv[], bootm_headers_t *images)
{
    int ret;
    ulong zi_start, zi_end;
    ret = do_bootm_states(cmdtp, flag, argc, argv, BOOTM_STATE_START,
                  images, 1);
    /* Setup Linux kernel zImage entry point */
    if (!argc) {
        images->ep = load_addr;
        debug("*  kernel: default image load address = 0x%08lx\n",
                load_addr);
    } else {
        images->ep = simple_strtoul(argv[0], NULL, 16);
        debug("*  kernel: cmdline image address = 0x%08lx\n",
            images->ep);
    }
    printf("images-ep %x\n", images->ep);
    ret = bootz_setup(images->ep, &zi_start, &zi_end);
    if (ret != 0)
        return 1;
    lmb_reserve(&images->lmb, images->ep, zi_end - zi_start);
    /*
     * Handle the BOOTM_STATE_FINDOTHER state ourselves as we do not
     * have a header that provide this informaiton.
     */
    if (bootm_find_ramdisk_fdt(flag, argc, argv))
        return 1;
    return 0;
}
```

### do_bootm_states函数定义

```
 int do_bootm_states(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[],
            int states, bootm_headers_t *images, int boot_progress)
{
    boot_os_fn *boot_fn;
    ulong iflag = 0;
    int ret = 0, need_boot_fn;
    images->state |= states;
    /*
     * Work through the states and see how far we get. We stop on
     * any error.
     */
    if (states & BOOTM_STATE_START)                     //states=1,ret=0;BOOTM_STATE_START=(0x00000001);
        ret = bootm_start(cmdtp, flag, argc, argv);     //在本函数中唯二需要正常执行的函数之一
    if (!ret && (states & BOOTM_STATE_FINDOS))          //states=1,ret=0;BOOTM_STATE_FINDOS=(0x00000002);
        ret = bootm_find_os(cmdtp, flag, argc, argv);  
    if (!ret && (states & BOOTM_STATE_FINDOTHER)) {     //states=1,ret=0;BOOTM_STATE_FINDOTHER=(0x00000004);
        ret = bootm_find_other(cmdtp, flag, argc, argv);
        argc = 0;   /* consume the args */
    }
    /* From now on, we need the OS boot function */     //此时states=1,ret=0;
    if (ret)
        return ret;
    boot_fn = bootm_os_get_boot_func(images->os.os); //在本函数中唯二需要正常执行的函数之一
    need_boot_fn = states & (BOOTM_STATE_OS_CMDLINE |
            BOOTM_STATE_OS_BD_T | BOOTM_STATE_OS_PREP |
            BOOTM_STATE_OS_FAKE_GO | BOOTM_STATE_OS_GO);        //这部分的宏0x00000 7E 0
    /* Call various other states that are not generally used */ //此时boot_fn = NULL,need_boot_fn = 0x700
    if (!ret && (states & BOOTM_STATE_OS_CMDLINE))              //states=1,ret=0;BOOTM_STATE_OS_CMDLINE=(0x00000040);
        ret = boot_fn(BOOTM_STATE_OS_CMDLINE, argc, argv, images);
    if (!ret && (states & BOOTM_STATE_OS_BD_T))                 //states=1,ret=0;BOOTM_STATE_OS_BD_T=(0x00000080);
        ret = boot_fn(BOOTM_STATE_OS_BD_T, argc, argv, images);
    if (!ret && (states & BOOTM_STATE_OS_PREP))                 //states=1,ret=0;BOOTM_STATE_OS_PREP=(0x00000100);
        ret = boot_fn(BOOTM_STATE_OS_PREP, argc, argv, images);
    /* Check for unsupported subcommand. */
    if (ret) {
        puts("subcommand not supported\n");
        return ret;
    }
    /* Now run the OS! We hope this doesn't return */
    if (!ret && (states & BOOTM_STATE_OS_GO))                   //ret =0;states =1;BOOTM_STATE_OS_GO=0x00000400;
        ret = boot_selected_os(argc, argv, BOOTM_STATE_OS_GO,
                images, boot_fn);
    /* Deal with any fallout */
err:
    if (iflag)                                          //iflag= 0;ret =0;
        enable_interrupts();
    if (ret == BOOTM_ERR_UNIMPLEMENTED)
        bootstage_error(BOOTSTAGE_ID_DECOMP_UNIMPL);
    else if (ret == BOOTM_ERR_RESET)
        do_reset(cmdtp, flag, argc, argv);
    return ret;
}
```

#### bootm_start函数定义实际执行：

```
static int bootm_start(cmd_tbl_t *cmdtp, int flag, int argc,
               char * const argv[])
{
    memset((void *)&images, 0, sizeof(images));
    images.verify = getenv_yesno("verify");       //从环境变量中获取verify的值
    boot_start_lmb(&images);
    bootstage_mark_name(BOOTSTAGE_ID_BOOTM_START, "bootm_start");
    images.state = BOOTM_STATE_START;
    return 0;
}
```

#### 1、getenv_yesno函数定义 实际执行：

```
char *getenv(const char *name)
{
    if (gd->flags & GD_FLG_ENV_READY) { /* after import into hashtable */
        ENTRY e, *ep;
        WATCHDOG_RESET();
        e.key   = name;                         //将需要查找的变量转为ENTRY结构体变量的键值
        e.data  = NULL;
        hsearch_r(e, FIND, &ep, &env_htab, 0);
        return ep ? ep->data : NULL;
    }
    /* restricted capabilities before import */
    if (getenv_f(name, (char *)(gd->env_buf), sizeof(gd->env_buf)) > 0)
        return (char *)(gd->env_buf);
    return NULL;
}
int getenv_yesno(const char *var)
{
    char *s = getenv(var);
    if (s == NULL)
        return -1;
    return (*s == '1' || *s == 'y' || *s == 'Y' || *s == 't' || *s == 'T') ?
        1 : 0;
}
```

getenv函数定义：

本函数涉及到env变量的相关修改，实际解析暂未完成，仅仅初步完成hsearch_r函数的理解（），getenc_f函数的理解分析暂未完成因为好像很多地方均有相关的调用导致打印信息比较杂不好确定

hsearch_r函数定义

```
int hsearch_r(ENTRY item, ACTION action, ENTRY ** retval,
          struct hsearch_data *htab, int flag)
{
    unsigned int hval;
    unsigned int count;
    unsigned int len = strlen(item.key);
    unsigned int idx;
    unsigned int first_deleted = 0;
    int ret;
    /* Compute an value for the given string. Perhaps use a better method. */
    hval = len;
    count = len;
    //hash表键值生成算法:
    //将item.key---环境变量name中的字符移位并累加，然后模hash表项数，即为该环境变量在hash表项中的数组索引---hash表键值。
    //注意这里使用的还是估算法，即假设每条环境变量字符串占用8个字节，注意上述代码中hval为32位数，移位操作8次就会发生循环。
    while (count-- > 0) {
        hval <<= 4;
        hval += item.key[count];
    }
    /*
     * First hash function:
     * simply take the modul but prevent zero.
     */
    hval %= htab->size;                  //htab->size为哈希表的长度
    if (hval == 0)                      //保留了表项数组索引0的空间，在之前himport_r调用的hcreate_r分配hash存储空间时，多分配了一个表项空间，即该处的保留空间。
        ++hval;
    /* The first index tried. */
    idx = hval;                         //用模取后的数值作为哈希表的索引
    if (htab->table[idx].used) {     //如果新插入的表项计算的索引已被占用，那么需要根据算法进行更新索引
        /*
         * Further action might be required according to the
         * action value.
         */
        unsigned hval2;
        if (htab->table[idx].used == -1
            && !first_deleted)
            first_deleted = idx;
        ret = _compare_and_overwrite_entry(item, action, retval, htab,
            flag, hval, idx);
        //检查以idx为索引的hash表项中的key(环境变量名字符串)是否和输入参数item.key一致，不一致返回-1
        //c.2)调用change_ok执行权限检查，不通过返回0
        //c.3)如果相应表项的成员callback 不为空，则执行该回调函数。函数执行失败返回0
        //回调函数在hash表项初始化时或被填充（见下面的代码段3）
        //c.4)执行hash表项修改操作，即修改hash表项中的data为新值
        if (ret != -1)
            return ret;
        /*
         * Second hash function:
         * as suggested in [Knuth]
         */
        hval2 = 1 + hval % (htab->size - 2);
        do {
            /*
             * Because SIZE is prime this guarantees to
             * step through all available indices.
             */
            if (idx <= hval2)
                idx = htab->size + idx - hval2;
            else
                idx -= hval2;
            /*
             * If we visited all entries leave the loop
             * unsuccessfully.
             */
            if (idx == hval)
                break;
            /* If entry is found use it. */
            ret = _compare_and_overwrite_entry(item, action, retval,
                htab, flag, hval, idx);
            if (ret != -1)
                return ret;
        }
        while (htab->table[idx].used);
    }
    /* An empty bucket has been found. */
    if (action == ENTER) {
        /*
         * If table is full and another entry should be
         * entered return with error.
         */
        if (htab->filled == htab->size) {
            __set_errno(ENOMEM);
            *retval = NULL;
            return 0;
        }
        /*
         * Create new entry;
         * create copies of item.key and item.data
         */
        if (first_deleted)
            idx = first_deleted;
        htab->table[idx].used = hval;
        htab->table[idx].entry.key = strdup(item.key);
        htab->table[idx].entry.data = strdup(item.data);
        if (!htab->table[idx].entry.key ||
            !htab->table[idx].entry.data) {
            __set_errno(ENOMEM);
            *retval = NULL;
            return 0;
        }
        ++htab->filled;
        /* This is a new entry, so look up a possible callback */
        env_callback_init(&htab->table[idx].entry);
        /* Also look for flags */
        env_flags_init(&htab->table[idx].entry);
        /* check for permission */
        if (htab->change_ok != NULL && htab->change_ok(
            &htab->table[idx].entry, item.data, env_op_create, flag)) {
            debug("change_ok() rejected setting variable "
                "%s, skipping it!\n", item.key);
            _hdelete(item.key, htab, &htab->table[idx].entry, idx);
            __set_errno(EPERM);
            *retval = NULL;
            return 0;
        }
        /* If there is a callback, call it */
        if (htab->table[idx].entry.callback &&
            htab->table[idx].entry.callback(item.key, item.data,
            env_op_create, flag)) {
            debug("callback() rejected setting variable "
                "%s, skipping it!\n", item.key);
            _hdelete(item.key, htab, &htab->table[idx].entry, idx);
            __set_errno(EINVAL);
            *retval = NULL;
            return 0;
        }
        /* return new entry */
        *retval = &htab->table[idx].entry;
        return 1;
    }
    __set_errno(ESRCH);
    *retval = NULL;
    return 0;
}
```

中间涉及调用函数：

```
static inline int _compare_and_overwrite_entry(ENTRY item, ACTION action,
    ENTRY **retval, struct hsearch_data *htab, int flag,
    unsigned int hval, unsigned int idx)
{
    if (htab->table[idx].used == hval
        && strcmp(item.key, htab->table[idx].entry.key) == 0) {
        /* Overwrite existing value? */
        if ((action == ENTER) && (item.data != NULL)) {
            /* check for permission */
            if (htab->change_ok != NULL && htab->change_ok(
                &htab->table[idx].entry, item.data,
                env_op_overwrite, flag)) {
                debug("change_ok() rejected setting variable "
                    "%s, skipping it!\n", item.key);
                __set_errno(EPERM);
                *retval = NULL;
                return 0;
            }
            /* If there is a callback, call it */
            if (htab->table[idx].entry.callback &&
                htab->table[idx].entry.callback(item.key,
                item.data, env_op_overwrite, flag)) {
                debug("callback() rejected setting variable "
                    "%s, skipping it!\n", item.key);
                __set_errno(EINVAL);
                *retval = NULL;
                return 0;
            }
            free(htab->table[idx].entry.data);
            htab->table[idx].entry.data = strdup(item.data);
            if (!htab->table[idx].entry.data) {
                __set_errno(ENOMEM);
                *retval = NULL;
                return 0;
            }
        }
        /* return found entry */
        *retval = &htab->table[idx].entry;
        return idx;
    }
    /* keep searching */
    return -1;
}
```

#### 2、boot_start_lmb函数定义

镜像内存管理，具体详解没看太懂，存在部分的结构体和变量信息不了解

```
static void lmb_remove_region(struct lmb_region *rgn, unsigned long r)
{
    unsigned long i;
    for (i = r; i < rgn->cnt - 1; i++) {
        rgn->region[i].base = rgn->region[i + 1].base;
        rgn->region[i].size = rgn->region[i + 1].size;
    }
    rgn->cnt--;
}
 
/* Assumption: base addr of region 1 < base addr of region 2 */
static void lmb_coalesce_regions(struct lmb_region *rgn,
        unsigned long r1, unsigned long r2)
{
    rgn->region[r1].size += rgn->region[r2].size;
    lmb_remove_region(rgn, r2);
}
 
static long lmb_addrs_adjacent(phys_addr_t base1, phys_size_t size1,
        phys_addr_t base2, phys_size_t size2)
{
    if (base2 == base1 + size1)
        return 1;
    else if (base1 == base2 + size2)
        return -1;
    return 0;
}
static long lmb_add_region(struct lmb_region *rgn, phys_addr_t base, phys_size_t size)
{
    unsigned long coalesced = 0;
    long adjacent, i;
    if ((rgn->cnt == 1) && (rgn->region[0].size == 0)) {
        rgn->region[0].base = base;
        rgn->region[0].size = size;
        return 0;
    }
    /* First try and coalesce this LMB with another. */
    for (i=0; i < rgn->cnt; i++) {
        phys_addr_t rgnbase = rgn->region[i].base;
        phys_size_t rgnsize = rgn->region[i].size;
        if ((rgnbase == base) && (rgnsize == size))
            /* Already have this region, so we're done */
            return 0;
        adjacent = lmb_addrs_adjacent(base,size,rgnbase,rgnsize);
        if ( adjacent > 0 ) {
            rgn->region[i].base -= size;
            rgn->region[i].size += size;
            coalesced++;
            break;
        }
        else if ( adjacent < 0 ) {
            rgn->region[i].size += size;
            coalesced++;
            break;
        }
    }
    if ((i < rgn->cnt-1) && lmb_regions_adjacent(rgn, i, i+1) ) {
        lmb_coalesce_regions(rgn, i, i+1);
        coalesced++;
    }
    if (coalesced)
        return coalesced;
    if (rgn->cnt >= MAX_LMB_REGIONS)
        return -1;
    /* Couldn't coalesce the LMB, so add it to the sorted table. */
    for (i = rgn->cnt-1; i >= 0; i--) {
        if (base < rgn->region[i].base) {
            rgn->region[i+1].base = rgn->region[i].base;
            rgn->region[i+1].size = rgn->region[i].size;
        } else {
            rgn->region[i+1].base = base;
            rgn->region[i+1].size = size;
            break;
        }
    }
    if (base < rgn->region[0].base) {
        rgn->region[0].base = base;
        rgn->region[0].size = size;
    }
    rgn->cnt++;
    return 0;
}
long lmb_add(struct lmb *lmb, phys_addr_t base, phys_size_t size)
{
    struct lmb_region *_rgn = &(lmb->memory);
    return lmb_add_region(_rgn, base, size);
}
ulong getenv_bootm_low(void)
{
    char *s = getenv("bootm_low");
    if (s) {
        ulong tmp = simple_strtoul(s, NULL, 16);
        return tmp;
    }
    return CONFIG_SYS_SDRAM_BASE;
}
phys_size_t getenv_bootm_size(void)
{
    phys_size_t tmp;
    char *s = getenv("bootm_size");
    if (s) {
        tmp = (phys_size_t)simple_strtoull(s, NULL, 16);
        return tmp;
    }
    s = getenv("bootm_low");
    if (s)
        tmp = (phys_size_t)simple_strtoull(s, NULL, 16);
    else
        tmp = 0;
    return gd->bd->bi_dram[0].size - tmp;
}
void lmb_init(struct lmb *lmb)
{
    /* Create a dummy zero size LMB which will get coalesced away later.
     * This simplifies the lmb_add() code below...
     */
    lmb->memory.region[0].base = 0;
    lmb->memory.region[0].size = 0;
    lmb->memory.cnt = 1;
    lmb->memory.size = 0;
    /* Ditto. */
    lmb->reserved.region[0].base = 0;
    lmb->reserved.region[0].size = 0;
    lmb->reserved.cnt = 1;
    lmb->reserved.size = 0;
}
static void boot_start_lmb(bootm_headers_t *images)
{
    ulong       mem_start;
    phys_size_t mem_size;
    lmb_init(&images->lmb);
    mem_start = getenv_bootm_low();
    mem_size = getenv_bootm_size();
    lmb_add(&images->lmb, (phys_addr_t)mem_start, mem_size);
    arch_lmb_reserve(&images->lmb);
    board_lmb_reserve(&images->lmb);
}
```

#### bootstage_mark_name

```
__weak void show_boot_progress(int val) {}
static inline ulong bootstage_mark_name(enum bootstage_id id, const char *name)
{
    show_boot_progress(id);     //未有具体语句，空函数，或者是暂时没找到？待定
    return 0;
}
```

### bootm_os_get_boot_func函数定义

```
static boot_os_fn *boot_os[] = {
    [IH_OS_U_BOOT] = do_bootm_standalone,
#ifdef CONFIG_BOOTM_LINUX
    [IH_OS_LINUX] = do_bootm_linux,     //linux系统启动，对于zynq平台来说最后是调用这个函数完成的系统启动
#endif
#ifdef CONFIG_BOOTM_NETBSD
    [IH_OS_NETBSD] = do_bootm_netbsd,
#endif
#ifdef CONFIG_LYNXKDI
    [IH_OS_LYNXOS] = do_bootm_lynxkdi,
#endif
#ifdef CONFIG_BOOTM_RTEMS
    [IH_OS_RTEMS] = do_bootm_rtems,
#endif
#if defined(CONFIG_BOOTM_OSE)
    [IH_OS_OSE] = do_bootm_ose,
#endif
#if defined(CONFIG_BOOTM_PLAN9)
    [IH_OS_PLAN9] = do_bootm_plan9,
#endif
#if defined(CONFIG_BOOTM_VXWORKS) && \
    (defined(CONFIG_PPC) || defined(CONFIG_ARM))
    [IH_OS_VXWORKS] = do_bootm_vxworks,
#endif
#if defined(CONFIG_CMD_ELF)
    [IH_OS_QNX] = do_bootm_qnxelf,
#endif
#ifdef CONFIG_INTEGRITY
    [IH_OS_INTEGRITY] = do_bootm_integrity,
#endif
#ifdef CONFIG_BOOTM_OPENRTOS
    [IH_OS_OPENRTOS] = do_bootm_openrtos,
#endif
};
boot_os_fn *bootm_os_get_boot_func(int os)
{
    return boot_os[os];             //os作为下标，从os结构体中找到os值所对应的内核启动函数，进行启动内核
                                    //当然，目前是在bootz_start函数中调用do_bootm_states到达的此处，此时的os并未赋上需要的值
                                    //只有在do_bootz中调用do_bootm_states时此时的os才有正常赋值，也就能够调用到do_bootm_linux进行起内核
}
```

do_bootm_linux

```
int do_bootm_linux(int flag, int argc, char * const argv[],
           bootm_headers_t *images)
{
    /* No need for those on ARM */
    if (flag & BOOTM_STATE_OS_BD_T || flag & BOOTM_STATE_OS_CMDLINE)
        return -1;
    if (flag & BOOTM_STATE_OS_PREP) {
        boot_prep_linux(images);
        return 0;
    }
    if (flag & (BOOTM_STATE_OS_GO | BOOTM_STATE_OS_FAKE_GO)) {
        boot_jump_linux(images, flag);
        return 0;
    }
    boot_prep_linux(images);
    boot_jump_linux(images, flag);
    return 0;
}
```

boot_prep_linux 起内核之前的准备工作

```
/* Subcommand: PREP */
static void boot_prep_linux(bootm_headers_t *images)
{
    char *commandline = getenv("bootargs");//获取bootargs参数赋值给commandline
    if (IMAGE_ENABLE_OF_LIBFDT && images->ft_len) {
#ifdef CONFIG_OF_LIBFDT
        debug("using: FDT\n");
        if (image_setup_linux(images)) {
            printf("FDT creation failed! hanging...");
            hang();
        }
#endif
    } else if (BOOTM_ENABLE_TAGS) {
        debug("using: ATAGS\n");
        setup_start_tag(gd->bd);
        if (BOOTM_ENABLE_SERIAL_TAG)
            setup_serial_tag(&params);
        if (BOOTM_ENABLE_CMDLINE_TAG)
            setup_commandline_tag(gd->bd, commandline);
        if (BOOTM_ENABLE_REVISION_TAG)
            setup_revision_tag(&params);
        if (BOOTM_ENABLE_MEMORY_TAGS)
            setup_memory_tags(gd->bd);
        if (BOOTM_ENABLE_INITRD_TAG) {
            if (images->rd_start && images->rd_end) {
                setup_initrd_tag(gd->bd, images->rd_start,
                         images->rd_end);
            }
        }
        setup_board_tags(&params);
        setup_end_tag(gd->bd);
    } else {
#ifndef CONFIG_ARM_APPENDED_DTB
        printf("FDT and ATAGS support not compiled in - hanging\n");
        hang();
#endif
    }
}
```

boot_jump_linux 跳转到起内核部分

```
/* Subcommand: GO */
static void boot_jump_linux(bootm_headers_t *images, int flag)
{
#ifdef CONFIG_ARM64
    void (*kernel_entry)(void *fdt_addr, void *res0, void *res1,
            void *res2);
    int fake = (flag & BOOTM_STATE_OS_FAKE_GO);
    kernel_entry = (void (*)(void *fdt_addr, void *res0, void *res1,
                void *res2))images->ep;
    debug("## Transferring control to Linux (at address %lx)...\n",
        (ulong) kernel_entry);
    bootstage_mark(BOOTSTAGE_ID_RUN_OS);
    announce_and_cleanup(fake);
    if (!fake) {
        do_nonsec_virt_switch();
        kernel_entry(images->ft_addr, NULL, NULL, NULL);
    }
#else
    unsigned long machid = gd->bd->bi_arch_number;
    char *s;
    void (*kernel_entry)(int zero, int arch, uint params);
    unsigned long r2;
    int fake = (flag & BOOTM_STATE_OS_FAKE_GO);
    kernel_entry = (void (*)(int, int, uint))images->ep;
    s = getenv("machid");
    if (s) {
        strict_strtoul(s, 16, &machid);
        printf("Using machid 0x%lx from environment\n", machid);
    }
    debug("## Transferring control to Linux (at address %08lx)" \
        "...\n", (ulong) kernel_entry);
    bootstage_mark(BOOTSTAGE_ID_RUN_OS);
    announce_and_cleanup(fake);
    if (IMAGE_ENABLE_OF_LIBFDT && images->ft_len)
        r2 = (unsigned long)images->ft_addr;
    else
        r2 = gd->bd->bi_boot_params;
    if (!fake) {
#if defined(CONFIG_ARMV7_NONSEC) || defined(CONFIG_ARMV7_VIRT)
        if (armv7_boot_nonsec()) {
            armv7_init_nonsec();
            secure_ram_addr(_do_nonsec_entry)(kernel_entry,
                              0, machid, r2);
        } else
#endif
            kernel_entry(0, machid, r2);
    }
#endif
}
```

### bootz_setup函数定义

本函数的主要作用是更新image的start和end信息，并给出打印信息

```
int bootz_setup(ulong image, ulong *start, ulong *end)
{
    struct zimage_header *zi;
    zi = (struct zimage_header *)map_sysmem(image, 0);
    printf("Kernel image @ %#08lx [ %#08lx - %#08lx ]\n", image, *start, *end);
    if (zi->zi_magic != LINUX_ARM_ZIMAGE_MAGIC) {
        puts("Bad Linux ARM zImage magic!\n");
        return 1;
    }
    *start = zi->zi_start;
    *end = zi->zi_end;
    printf("Kernel image @ %#08lx [ %#08lx - %#08lx ]\n", image, *start,
          *end);
    return 0;
}
```

### lmb_reserve函数定义

```
static void lmb_remove_region(struct lmb_region *rgn, unsigned long r)
{
    unsigned long i;
    for (i = r; i < rgn->cnt - 1; i++) {
        rgn->region[i].base = rgn->region[i + 1].base;
        rgn->region[i].size = rgn->region[i + 1].size;
    }
    rgn->cnt--;
}
 
 
/* Assumption: base addr of region 1 < base addr of region 2 */
static void lmb_coalesce_regions(struct lmb_region *rgn,
        unsigned long r1, unsigned long r2)
{
    rgn->region[r1].size += rgn->region[r2].size;
    lmb_remove_region(rgn, r2);
}
 
 
static long lmb_addrs_adjacent(phys_addr_t base1, phys_size_t size1,
        phys_addr_t base2, phys_size_t size2)
{
    if (base2 == base1 + size1)
        return 1;
    else if (base1 == base2 + size2)
        return -1;
    return 0;
}
static long lmb_add_region(struct lmb_region *rgn, phys_addr_t base, phys_size_t size)
{
    unsigned long coalesced = 0;
    long adjacent, i;
    if ((rgn->cnt == 1) && (rgn->region[0].size == 0)) {
        rgn->region[0].base = base;
        rgn->region[0].size = size;
        return 0;
    }
    /* First try and coalesce this LMB with another. */
    for (i=0; i < rgn->cnt; i++) {
        phys_addr_t rgnbase = rgn->region[i].base;
        phys_size_t rgnsize = rgn->region[i].size;
        if ((rgnbase == base) && (rgnsize == size))
            /* Already have this region, so we're done */
            return 0;
        adjacent = lmb_addrs_adjacent(base,size,rgnbase,rgnsize);
        if ( adjacent > 0 ) {
            rgn->region[i].base -= size;
            rgn->region[i].size += size;
            coalesced++;
            break;
        }
        else if ( adjacent < 0 ) {
            rgn->region[i].size += size;
            coalesced++;
            break;
        }
    }
    if ((i < rgn->cnt-1) && lmb_regions_adjacent(rgn, i, i+1) ) {
        lmb_coalesce_regions(rgn, i, i+1);
        coalesced++;
    }
    if (coalesced)
        return coalesced;
    if (rgn->cnt >= MAX_LMB_REGIONS)
        return -1;
    /* Couldn't coalesce the LMB, so add it to the sorted table. */
    for (i = rgn->cnt-1; i >= 0; i--) {
        if (base < rgn->region[i].base) {
            rgn->region[i+1].base = rgn->region[i].base;
            rgn->region[i+1].size = rgn->region[i].size;
        } else {
            rgn->region[i+1].base = base;
            rgn->region[i+1].size = size;
            break;
        }
    }
    if (base < rgn->region[0].base) {
        rgn->region[0].base = base;
        rgn->region[0].size = size;
    }
    rgn->cnt++;
    return 0;
}
long lmb_reserve(struct lmb *lmb, phys_addr_t base, phys_size_t size)
{
    struct lmb_region *_rgn = &(lmb->reserved);
    return lmb_add_region(_rgn, base, size);
}
```