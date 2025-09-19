### **代码调用跟踪**
```
u-boot.lds:(arch/arm/cpu/u-boot.lds)
    |-->_start:(arch/arm/lib/vectors.S)
        |-->reset(arch/arm/cpu/armv7/start.S)   
            |-->save_boot_params(arch/arm/cpu/armv7/start.S)/*将引导参数保存到内存中*/
                |-->save_boot_params_ret(arch/arm/cpu/armv7/start.S)
                    |-->cpu_init_cp15(arch/arm/cpu/armv7/start.S)/*初始化*/
                    |-->cpu_init_crit(arch/arm/cpu/armv7/start.S)
                        |-->lowlevel_init(arch/arm/cpu/armv7/lowlevel_init.S)
                    |-->_main(arch/arm/lib/crt0.S)
                        |-->board_init_f_alloc_reserve(common/init/board_init.c)/*为u-boot的gd结构体分配空间*/
                        |-->board_init_f_init_reserve(common/init/board_init.c)    /*将gd结构体清零*/
                        |-->board_init_f(common/board_f.c)
                            |-->initcall_run_list(include/initcall.h)    /*初始化序列函数*/
                                |-->init_sequence_f[](common/board_f.c)    /* 初始化序列函数数组 */
                                    |-->board_early_init_f(board/freescale/mx6ull_toto/mx6ull_toto.c)/*初始化串口的IO配置*/
                                    |-->timer_init(arch/arm/imx-common/timer.c)    /*初始化内核定时器，为uboot提供时钟节拍*/
                                    |-->init_baud_rate(common/board_f.c)        /*初始化波特率*/
                                    |-->serial_init(drivers/serial/serial.c)    /*初始化串口通信设置*/
                                    |-->console_init_f(common/console.c)        /*初始化控制台*/
                                    |-->...
                        |-->relocate_code(arch/arm/lib/relocate.S)    /*主要完成镜像拷贝和重定位*/
                        |-->relocate_vectors(arch/arm/lib/relocate.S)/*重定位向量表*/
                        |-->board_init_r(common/board_r.c)/*板级初始化*/
                            |-->initcall_run_list(include/initcall.h)/*初始化序列函数*/
                                |-->init_sequence_r[](common/board_f.c)/*序列函数*/
                                    |-->initr_reloc(common/board_r.c)    /*设置 gd->flags,标记重定位完成*/
                                    |-->serial_initialize(drivers/serial/serial-uclass.c)/*初始化串口*/
                                        |-->serial_init(drivers/serial/serial-uclass.c)     /*初始化串口*/
                                    |-->initr_mmc(common/board_r.c)                         /*初始化emmc*/
                                        |-->mmc_initialize(drivers/mmc/mmc.c)
                                            |-->mmc_do_preinit(drivers/mmc/mmc.c)
                                                |-->mmc_start_init(drivers/mmc/mmc.c)
                                    |-->console_init_r(common/console.c)                /*初始化控制台*/
                                    |-->interrupt_init(arch/arm/lib/interrupts.c)        /*初始化中断*/
                                    |-->initr_net(common/board_r.c)                        /*初始化网络设备*/
                                        |-->eth_initialize(net/eth-uclass.c)
                                            |-->eth_common_init(net/eth_common.c)
                                                |-->phy_init(drivers/net/phy/phy.c)
                                            |-->uclass_first_device_check(drivers/core/uclass.c)
                                                |-->uclass_find_first_device(drivers/core/uclass.c)
                                                |-->device_probe(drivers/core/device.c)
                                                    |-->device_of_to_plat(drivers/core/device.c)
                                                        |-->drv->of_to_plat
                                                            |-->fecmxc_of_to_plat(drivers/net/fec_mxc.c)/*解析设备树信息*/
                                                    |-->device_get_uclass_id(drivers/core/device.c)
                                                    |-->uclass_pre_probe_device(drivers/core/uclass.c)
                                                    |-->drv->probe(dev)
                                                        /*drivers/net/fec_mxc.c*/
                                                        U_BOOT_DRIVER(fecmxc_gem) = {
                                                            .name    = "fecmxc",
                                                            .id    = UCLASS_ETH,
                                                            .of_match = fecmxc_ids,
                                                            .of_to_plat = fecmxc_of_to_plat,
                                                            .probe    = fecmxc_probe,
                                                            .remove    = fecmxc_remove,
                                                            .ops    = &fecmxc_ops,
                                                            .priv_auto    = sizeof(struct fec_priv),
                                                            .plat_auto    = sizeof(struct eth_pdata),
                                                        };
                                                        |-->fecmxc_probe(drivers/net/fec_mxc.c)/*探测和初始化*/
                                                            |-->fec_get_miibus(drivers/net/fec_mxc.c)
                                                                |-->mdio_alloc(drivers/net/fec_mxc.c)
                                                                |-->bus->read = fec_phy_read;
                                                                |-->bus->write = fec_phy_write;
                                                                |-->mdio_register(common/miiphyutil.c)
                                                                |-->fec_mii_setspeed(drivers/net/fec_mxc.c)
                                                            |-->fec_phy_init(drivers/net/fec_mxc.c)
                                                                |-->device_get_phy_addr(drivers/net/fec_mxc.c)
                                                                |-->phy_connect(drivers/net/phy/phy.c)
                                                                    |-->phy_find_by_mask(drivers/net/phy/phy.c)
                                                                        |-->bus->reset(bus)
                                                                        |-->get_phy_device_by_mask(drivers/net/phy/phy.c)
                                                                            |-->create_phy_by_mask(drivers/net/phy/phy.c)
                                                                                |-->phy_device_create(drivers/net/phy/phy.c)
                                                                                    |-->phy_probe(drivers/net/phy/phy.c)
                                                                    |-->phy_connect_dev(drivers/net/phy/phy.c)
                                                                        |-->phy_reset(drivers/net/phy/phy.c)
                                                                |-->phy_config(drivers/net/phy/phy.c)
                                                                    |-->board_phy_config(drivers/net/phy/phy.c)
                                                                        |-->phydev->drv->config(phydev)
                                                                            /*drivers/net/phy/smsc.c*/
                                                                            static struct phy_driver lan8710_driver = {
                                                                                .name = "SMSC LAN8710/LAN8720",
                                                                                .uid = 0x0007c0f0,
                                                                                .mask = 0xffff0,
                                                                                .features = PHY_BASIC_FEATURES,
                                                                                .config = &genphy_config_aneg,
                                                                                .startup = &genphy_startup,
                                                                                .shutdown = &genphy_shutdown,
                                                                            };
                                                                            |-->genphy_config_aneg(drivers/net/phy/phy.c)
                                                                                |-->phy_reset(需要手动调用)(drivers/net/phy/phy.c)
                                                                                |-->genphy_setup_forced(drivers/net/phy/phy.c)
                                                                                |-->genphy_config_advert(drivers/net/phy/phy.c)
                                                                                |-->genphy_restart_aneg(drivers/net/phy/phy.c)
                                                    |-->uclass_post_probe_device(drivers/core/uclass.c)
                                                        |-->uc_drv->post_probe(drivers/core/uclass.c)
                                                            /*net/eth-uclass.c*/
                                                            UCLASS_DRIVER(ethernet) = {
                                                                .name        = "ethernet",
                                                                .id        = UCLASS_ETH,
                                                                .post_bind    = eth_post_bind,
                                                                .pre_unbind    = eth_pre_unbind,
                                                                .post_probe    = eth_post_probe,
                                                                .pre_remove    = eth_pre_remove,
                                                                .priv_auto    = sizeof(struct eth_uclass_priv),
                                                                .per_device_auto    = sizeof(struct eth_device_priv),
                                                                .flags        = DM_UC_FLAG_SEQ_ALIAS,
                                                            };
                                                            |-->eth_post_probe(net/eth-uclass.c)
                                                                |-->eth_write_hwaddr(drivers/core/uclass.c)
                                    |-->...
                                    |-->run_main_loop(common/board_r.c)/*主循环，处理命令*/
                                        |-->main_loop(common/main.c)
                                            |-->bootdelay_process(common/autoboot.c)    /*读取环境变量bootdelay和bootcmd的内容*/
                                            |-->autoboot_command(common/autoboot.c)        /*倒计时按下执行，没有操作执行bootcmd的参数*/
                                                |-->abortboot(common/autoboot.c)
                                                    |-->printf("Hit any key to stop autoboot: %2d ", bootdelay);
                                                    /*到这里就是我们看到uboot延时3s启动内核的地方*/
                                            |-->cli_loop(common/cli.c)    /*倒计时按下space键,执行用户输入命令*/
```

### 编译跟踪：

./zoo 

./make_uboot_imgtr_bidir.sh

./config.mk：CPU := $(CONFIG_SYS_CPU:"%"=%)

.config：CONFIG_SYS_CPU="armv7"

关于在armv7的.S和.c文件中遇到的宏定义目前初步判断其定义位置在：

u-boot/configs/zynq-hkvs_imgtr_deconfig

zynq_hkvs_imgtr_defconfig

和.config中还有

（待定，待更新）

### 1、_start函数详解

网上查阅资料了解到u-boot.lds中的_start作为启动的第一语句，继续跟_start执行的语句，其定义在vectors.S

```
_start:
#ifdef CONFIG_SYS_DV_NOR_BOOT_CFG
    .word   CONFIG_SYS_DV_NOR_BOOT_CFG
#endif
    b   reset
    ldr pc, _undefined_instruction
    ldr pc, _software_interrupt
    ldr pc, _prefetch_abort
    ldr pc, _data_abort
    ldr pc, _not_used
    ldr pc, _irq
    ldr pc, _fiq
/*
 *************************************************************************
 *
 * Indirect vectors table
 *
 * Symbols referenced here must be defined somewhere else
 *
 *************************************************************************
 */
    .globl  _undefined_instruction
    .globl  _software_interrupt
    .globl  _prefetch_abort
    .globl  _data_abort
    .globl  _not_used
    .globl  _irq
    .globl  _fiq
_undefined_instruction: .word undefined_instruction
_software_interrupt:    .word software_interrupt
_prefetch_abort:    .word prefetch_abort
_data_abort:        .word data_abort
_not_used:      .word not_used
_irq:           .word irq
_fiq:           .word fiq
    .balignl 16,0xdeadbeef
```

中断向量表中，先进行相对跳转到reset

而reset函数定义在arch/arm/cpu/armv7/start.S

### 2、reset函数详解

```
#include <asm-offsets.h>
#include <config.h>
#include <asm/system.h>
#include <linux/linkage.h>
/*************************************************************************
 *
 * Startup Code (reset vector)
 *
 * do important init only if we don't start from memory!
 * setup Memory and board specific bits prior to relocation.
 * relocate armboot to ram
 * setup stack
 *
 *************************************************************************/
    .globl  reset
    .globl  save_boot_params_ret
reset:
    /* Allow the board to save important registers */
    b   save_boot_params
```

在reset中执行跳转，跳转到save_boot_params

```
ENTRY(save_boot_params)
    b   save_boot_params_ret        @ back to my caller
ENDPROC(save_boot_params)
```

在save_boot_params中执行跳转，跳转到save_boot_params_ret

```
save_boot_params_ret:
    /*
     * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
     * except if in HYP mode already
     */
    mrs r0, cpsr
    and r1, r0, #0x1f       @ mask mode bits
    teq r1, #0x1a           @ test for HYP mode
    bicne r0, r0, #0x1f     @ clear all mode bits
    orrne r0, r0, #0x13     @ set SVC mode
    orr r0, r0, #0xc0       @ disable FIQ and IRQ
    msr cpsr,r0
/*
 * Setup vector:
 * (OMAP4 spl TEXT_BASE is not 32 byte aligned.
 * Continue to use ROM code vector only in OMAP4 spl)
 */
#if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
    /* Set V=0 in CP15 SCTLR register - for VBAR to point to vector */
    mrc p15, 0, r0, c1, c0, 0   @ Read CP15 SCTLR Register
    bic r0, #CR_V               @ V = 0
    mcr p15, 0, r0, c1, c0, 0   @ Write CP15 SCTLR Register
    /* Set vector address in CP15 VBAR register */
    ldr r0, =_start
    mcr p15, 0, r0, c12, c0, 0  @Set VBAR
#endif
    /* the mask ROM code should have PLL and others stable */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
    bl  cpu_init_cp15
    bl  cpu_init_crit
#endif
    bl  _main 
```

在save_boot_params_ret中执行指令：

指令分析：

![20190318104323545](../image/20190318104323545.png)

| 指令  | 使用方式                 | 作用                                                         |
| :---- | :----------------------- | :----------------------------------------------------------- |
| mrs   | `mrs r0, cpsr`           | 读取特殊寄存器cpsr的数据，并将数据写入到r0寄存器中           |
| and   | `and r1, r0, r2`         | 将r2和r0寄存器的数据相加并将和存到r1寄存器中                 |
| teq   | `teq r1, r2`             | 测试r1寄存器和r2寄存器的数据是否相等                         |
| bic   | bic r1, r1 , r2          | 根据r2中数据哪个位为1，清除r1对应的位，然后将结果存入r1。）  |
| orr   | orr r0, r0, r1           | 根据r1中数据哪个位为1，将r0对应的位设置为1，然后将结果存入r0。 |
| msr   | `mrs cpsr, r1`           | 读取普通寄存器r1的数据，并将数据写入到特殊寄存器cpsr中       |
| bicne |                          |                                                              |
| orrne |                          |                                                              |
| cmp   |                          |                                                              |
| strlo |                          |                                                              |
| addlo |                          |                                                              |
| lo    | cmp r0，r1strlo r0，[r1] | lo作为小于条件，表示r0 < r1 ，组合指令表示如果r0 < r1 那么将r0存到r1寄存器指向的地址 |
| ne    |                          | 不相等条件                                                   |
| eq    |                          | 相等条件                                                     |
| `adr` | `adr	lr, here`        | ADR指令用于计算当前指令地址加上一个偏移量得到目标地址，并将该地址存储到寄存器中将here的地址保存在lr寄存器中 |



| 寄存器 | 名称                     | 功能                                                         |
| :----- | :----------------------- | :----------------------------------------------------------- |
| cpsr   | CPSR(当前程序状态寄存器) | （1）保存最近已处理的 [ALU](https://so.csdn.net/so/search?q=ALU&spm=1001.2101.3001.7020) （算数逻辑单元）操作的信息；<br/>（2）控制中断的使能与禁止；<br/>（3）设置处理器的运行模式；![5bc6d780697f4f479bcfe1329fa95c54](../image/5bc6d780697f4f479bcfe1329fa95c54.png) |



```
mrs r0, cpsr            //将状态寄存器cpsr的数据存入到r0中
and r1, r0, #0x1f       //将r0中的数据同0x1f做加运算并将结果存到r1；实际是状态寄存器的数据同0x1f做加运算
teq r1, #0x1a           //尝试判断r1中的数据是否和0x1a相等；实际是判断状态寄存器中的数据同0x1f做加运算后结果是否为0x1a
bicne r0, r0, #0x1f     //如果r1的数据不等于0x1a，那么将r0中的[4:0]位清0；即清除工作模式
orrne r0, r0, #0x13     //如果r1的数据不等于0x1a，那么将r0中的[0:2]和第4位置1；即设置SVC模式,即超级管理员权限
orr r0, r0, #0xc0       //将r0中的第7位和第6位置1；关闭中断FIQ和IRQ
msr cpsr,r0             //将r0的数据写入到cpsr寄存器中
//总的作用是设置CPU为SVC32模式，除非已经处于HYP模式，同时禁止中断（FIQ和IRQ）；
```

SVC32模式（管理模式）：

SVC模式属于特权模式，可以访问受控资源，且相对sys模式多了部分在SVC模式下的影子寄存器，所以相对sys模式，可访问资源的能力相同，但拥有更多的硬件资源；从u-boot角度考虑，需要执行的事是初始化系统相关硬件资源，需要获取更多的权限，方便操作硬件，初始化硬件
u-boot作为bootloader最终目的是启动Linux内核，在做好准备工作（初始化硬件，准备好kernel和rootfs等）跳转到kernel前需要满足一些条件，其中之一为cpu处于SVC模式

### 3、cpu_init_cp15函数详解

**MCR指令和MRC指令：**

MCR：将 [ARM](https://so.csdn.net/so/search?q=ARM&spm=1001.2101.3001.7020) 寄存器的数据写入到 CP15 协处理器寄存器中。

MRC：就是读 CP15 [寄存器](https://so.csdn.net/so/search?q=寄存器&spm=1001.2101.3001.7020)， MCR 就是写 CP15 寄存器

MCR 指令格式：MCR {cond} p15, <opc1>, <Rt>, <CRn>, <CRm>, <opc2>

| 参数 | 含义                                                         |
| :--- | :----------------------------------------------------------- |
| cond | 指令执行的条件码，如果忽略的话就表示无条件执行。             |
| opc1 | 协处理器要执行的操作码                                       |
| Rt   | ARM 源寄存器，要写入到 CP15 寄存器的数据就保存在此寄存器中   |
| CRn  | CP15 协处理器的目标寄存器                                    |
| CRm  | 协处理器中附加的目标寄存器或者源操作数寄存器，如果不需要附加信息就将CRm 设置为 C0，否则结果不可预测 |
| opc2 | 可选的协处理器特定操作码，当不需要的时候要设置为 0           |

MRC 的指令格式和 MCR 一样

MRC 指令格式：MRC {cond} p15, <opc1>, <Rt>, <CRn>, <CRm>, <opc2>

Rt 是目标寄存器：也就是从CP15 指定寄存器读出来的数据会保存在 Rt 中

CRn 就是源寄存器：要读取的协处理器寄存器

 cpu_init_cp15中执行语句的功能：

- 1.失效 L1 I/D Cache；
- 2.禁用MMU和缓存。

```
ENTRY(cpu_init_cp15)
    /*
     * Invalidate L1 I/D
     */
    mov r0, #0                  @ set up for MCR
    mcr p15, 0, r0, c8, c7, 0   @ invalidate TLBs           //关闭TLBs
    mcr p15, 0, r0, c7, c5, 0   @ invalidate icache         //关闭TLBs
    mcr p15, 0, r0, c7, c5, 6   @ invalidate BP array       //关闭分支预测
    mcr p15, 0, r0, c7, c10, 4  @ DSB                       //数据同步屏蔽
    mcr p15, 0, r0, c7, c5, 4   @ ISB                       //指令同步屏蔽
    //关闭TBL、icache、分支预测、
    //DSB和ISB指的分别是数据chache和指令cache同步屏蔽
    /*
     * disable MMU stuff and caches
     */
    mrc p15, 0, r0, c1, c0, 0                               //将SCTLR的值赋给r0
    bic r0, r0, #0x00002000     @ clear bits 13 (--V-)      //清零第13位，异常向量映射到0x00000000
    bic r0, r0, #0x00000007     @ clear bits 2:0 (-CAM)     //失效dcache，失效对齐检查，禁用MMU
    orr r0, r0, #0x00000002     @ set bit 1 (--A-) Align    //使能对齐检查模式
    orr r0, r0, #0x00000800     @ set bit 11 (Z---) BTB     //使能分支预测
#ifdef CONFIG_SYS_ICACHE_OFF
    bic r0, r0, #0x00001000     @ clear bit 12 (I) I-cache
#else
    orr r0, r0, #0x00001000     @ set bit 12 (I) I-cache    //使能icache
#endif
    mcr p15, 0, r0, c1, c0, 0                               //将r0的值赋给SCTLR
#ifdef CONFIG_ARM_ERRATA_716044
    mrc p15, 0, r0, c1, c0, 0   @ read system control register
    orr r0, r0, #1 << 11      @ set bit #11
    mcr p15, 0, r0, c1, c0, 0   @ write system control register
#endif
#if (defined(CONFIG_ARM_ERRATA_742230) || defined(CONFIG_ARM_ERRATA_794072))
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 4           @ set bit #4
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif
#ifdef CONFIG_ARM_ERRATA_743622
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 6           @ set bit #6
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif
#ifdef CONFIG_ARM_ERRATA_751472
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 11      @ set bit #11
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif
#ifdef CONFIG_ARM_ERRATA_761320
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 21      @ set bit #21
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif
    mov r5, lr                  @ Store my Caller                           //将lr寄存器的值保存到r5寄存器
    mrc p15, 0, r1, c0, c0, 0   @ r1 has Read Main ID Register (MIDR)
    mov r3, r1, lsr #20         @ get variant field
    and r3, r3, #0xf            @ r3 has CPU variant
    and r4, r1, #0xf            @ r4 has CPU revision
    mov r2, r3, lsl #4          @ shift variant field for combined value
    orr r2, r4, r2              @ r2 has combined CPU variant + revision    //r2中保存了大的版本号+小的版本号，即完整的版本信息
    //获取CPU版本和修订信息
#ifdef CONFIG_ARM_ERRATA_798870
    cmp r2, #0x30               @ Applies to lower than R3p0
    bge skip_errata_798870      @ skip if not affected rev
    cmp r2, #0x20               @ Applies to including and above R2p0
    blt skip_errata_798870      @ skip if not affected rev
    mrc p15, 1, r0, c15, c0, 0  @ read l2 aux ctrl reg
    orr r0, r0, #1 << 7         @ Enable hazard-detect timeout
    push    {r1-r5}             @ Save the cpu info registers
    bl  v7_arch_cp15_set_l2aux_ctrl
    isb                         @ Recommended ISB after l2actlr update
    pop {r1-r5}                 @ Restore the cpu info - fall through
skip_errata_798870:
#endif
#ifdef CONFIG_ARM_ERRATA_454179
    cmp r2, #0x21               @ Only on < r2p1
    bge skip_errata_454179
    mrc p15, 0, r0, c1, c0, 1   @ Read ACR
    orr r0, r0, #(0x3 << 6)       @ Set DBSM(BIT7) and IBE(BIT6) bits
    push    {r1-r5}             @ Save the cpu info registers
    bl  v7_arch_cp15_set_acr
    pop {r1-r5}                 @ Restore the cpu info - fall through
skip_errata_454179:
#endif
#ifdef CONFIG_ARM_ERRATA_430973
    cmp r2, #0x21               @ Only on < r2p1
    bge skip_errata_430973
    mrc p15, 0, r0, c1, c0, 1   @ Read ACR
    orr r0, r0, #(0x1 << 6)       @ Set IBE bit
    push    {r1-r5}             @ Save the cpu info registers
    bl  v7_arch_cp15_set_acr
    pop {r1-r5}                 @ Restore the cpu info - fall through
skip_errata_430973:
#endif
#ifdef CONFIG_ARM_ERRATA_621766
    cmp r2, #0x21               @ Only on < r2p1
    bge skip_errata_621766
    mrc p15, 0, r0, c1, c0, 1   @ Read ACR
    orr r0, r0, #(0x1 << 5)       @ Set L1NEON bit
    push    {r1-r5}             @ Save the cpu info registers
    bl  v7_arch_cp15_set_acr
    pop {r1-r5}                 @ Restore the cpu info - fall through
skip_errata_621766:
#endif
    mov pc, r5                  @ back to my caller                         //子过程运行结束，跳转回去
ENDPROC(cpu_init_cp15)
```

**MIDR介绍：**

**lsr：**

**lsl：**

![MIDR 介绍](../image/MIDR%20%E4%BB%8B%E7%BB%8D.png)

其中从低至高第0-3 bit表示`revision`，代表固件版本的小版本号，如r1p3中的p3；
第4-15 bit表示`part number(id)`，代表这款CPU在所在`vendor`产品中定义的产品代码，如在`HiSilicon`产品中，`part_id=0xd01`代表`Kunpeng-920`芯片；
第16-19 bit表示`architecture`，即架构版本，`0x8`即ARMv8；
第20-23 bit表示`variant`，即固件版本的大版本号，如r1p3中的r1；
第24-31 bit表示`implementer`，即`vendor id`，如`vendor_id=0x48`表示`HiSilicon` ，vendor id是指供应商标识id

标识id和供应商名：

### 4、cpu_init_crit函数详解

 cpu_init_crit中执行语句：跳转到函数lowlevel_init入口地址

```
ENTRY(cpu_init_crit)
    /*
     * Jump to board specific initialization...
     * The Mask ROM will have already initialized
     * basic memory. Go here to bump up clock rate and handle
     * wake up conditions.
     */
    b   lowlevel_init       @ go setup pll,mux,memory
ENDPROC(cpu_init_crit)
```

### 5、lowlevel_init函数详解

lowlevel_init中执行语句：

- 1.设置SP指针为CONFIG_SYS_INIT_SP_ADDR
- 2.对sp指针做8字节对齐处理
- 3.SP减去#GD_SIZE = 248，GD_SIZE同样在generic-asm-offsets.h 中定了
- 4.对 sp 指针做8字节对齐处理
- 5.将SP保存到R9，ip和lr入栈，程序跳转到s_init（对于I.MX6ULL来说，s_init 就是个空函数）
- 6.函数一路返回，直到_main，s_init函数-->函数lowlevel_ini-->cpu_init_crit-->save_boot_params_ret-->_main。

```
ENTRY(lowlevel_init)
    /*
     * Setup a temporary stack. Global data is not available yet.
     */
    ldr sp, =CONFIG_SYS_INIT_SP_ADDR    /* 设置sp指针为CONFIG_SYS_INIT_SP_ADDR */
    bic sp, sp, #7 /* 8-byte alignment for ABI compliance 对sp指针做8字节对齐处理 */
#ifdef CONFIG_DM
    mov r9, #0
#else
    /*
     * Set up global data for boards that still need it. This will be
     * removed soon.
     */
#ifdef CONFIG_SPL_BUILD
    ldr r9, =gdata
#else
    sub sp, sp, #GD_SIZE    /* sp减去GD_SIZE=248 */
    bic sp, sp, #7          /* 对sp指针做8字节对齐处理 */
    mov r9, sp              /* 将 sp 保存至r9寄存器 */
#endif
#endif
    /*
     * Save the old lr(passed in ip) and the current lr to stack
     */
    push    {ip, lr}        /*ip和lr入栈*/
    /*
     * Call the very early init function. This should do only the
     * absolute bare minimum to get started. It should not:
     *
     * - set up DRAM
     * - use global_data
     * - clear BSS
     * - try to start a console
     *
     * For boards with SPL this should be empty since SPL can do all of
     * this init in the SPL board_init_f() function which is called
     * immediately after this.
     */
    bl  s_init              /* 程序跳转至s_init */
    pop {ip, pc}            /* IP和pc出栈 */
ENDPROC(lowlevel_init)
```

### 6、_main函数详解：

函数一路返回到,执行下一条指令，跳转到_main函数入口地址，_main函数中执行的指令：

- 1.设置sp指针为 CONFIG_SYS_INIT_SP_ADDR；
- 2.对sp指针做8字节对齐处理；
- 3.读取sp到寄存器r2里面；
- 4.调用函数board_init_f，

#### 1、_main函数前半段操作到跳转boarf_init_f

```
ENTRY(_main)
/*
 * Set up initial C runtime environment and call board_init_f(0).
 */
#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
    ldr sp, =(CONFIG_SPL_STACK)
#else
    ldr sp, =(CONFIG_SYS_INIT_SP_ADDR)
#endif
    bic sp, sp, #7  /* 8-byte alignment for ABI compliance */
    mov r2, sp
    sub sp, sp, #GD_SIZE    /* allocate one GD above SP 在sp之前分配GD_SIZE大小的空间给GD*/
    bic sp, sp, #7  /* 8-byte alignment for ABI compliance 再次对sp指针进行8字节对齐*/
    mov r9, sp      /* GD is above SP ；将sp保存至r9*/
    mov r1, sp      /* 将sp保存至r1 */
    mov r0, #0      /* 将r0寄存器清0 */
clr_gd:
    cmp r1, r2              /* while not at end of GD 比较r1和r2的大小 */
    strlo   r0, [r1]        /* clear 32-bit GD word ，如果r0中的页表项不超过512MB则将r0中的页表项写入r1指向的地址，否则忽略本次操作 */
    addlo   r1, r1, #4      /* move to next，给r1寄存器值加4 */
    blo clr_gd              /* 如果上一次比较结果小于零（表示目标地址在当前指令之前），则跳转到目标地址执行 */
#if defined(CONFIG_SYS_MALLOC_F_LEN)
    sub sp, sp, #CONFIG_SYS_MALLOC_F_LEN       
    str sp, [r9, #GD_MALLOC_BASE]              
#endif
    /* mov r0, #0 not needed due to above code */
    bl  board_init_f        /* 跳转到board_init_f */
```

##### 进一步了解board_init_f，在board_init_f函数中核心部分是执行初始化函数序列 init_sequence_f

```
static init_fnc_t init_sequence_f[] = {
#ifdef CONFIG_SANDBOX
    setup_ram_buf,
#endif
    setup_mon_len,  //设置gd的mon_len成员变量，也就是整个代码的长度，包括code段和bss段
    setup_fdt,      //设置gd的fdt_blod成员变量
#ifdef CONFIG_TRACE
    trace_early_init,
#endif
    initf_malloc,   //设置gd中和malloc有关的成员变量；但在zynq平台，本函数并无作用，因为其实际代码被宏限制，而该宏未定义
#if defined(CONFIG_MPC85xx) || defined(CONFIG_MPC86xx)
    /* TODO: can this go into arch_cpu_init()? */
    probecpu,
#endif
    arch_cpu_init,      /* basic arch cpu dependent setup */？？？
    mark_bootstage,     /* 标记bootstage */
#ifdef CONFIG_OF_CONTROL
    fdtdec_check_fdt,
#endif
    initf_dm,           //驱动模型初始化
#if defined(CONFIG_BOARD_EARLY_INIT_F)
    board_early_init_f,
#endif
    /* TODO: can any of this go into arch_cpu_init()? */
#if defined(CONFIG_PPC) && !defined(CONFIG_8xx_CPUCLK_DEFAULT)
    get_clocks,     /* get CPU and bus clocks (etc.) */
#if defined(CONFIG_TQM8xxL) && !defined(CONFIG_TQM866M) \
        && !defined(CONFIG_TQM885D)
    adjust_sdram_tbs_8xx,
#endif
    /* TODO: can we rename this to timer_init()? */
    init_timebase,
#endif
#if defined(CONFIG_ARM) || defined(CONFIG_MIPS) || defined(CONFIG_BLACKFIN)
    timer_init,     /* initialize timer */ //初始化内核定时器，为uboot提供时钟节拍，在arch/arm/imx-common/timer.c文件中定义；
#endif
#ifdef CONFIG_SYS_ALLOC_DPRAM
#if !defined(CONFIG_CPM2)
    dpram_init,
#endif
#endif
    init_baud_rate,     /* initialze baudrate settings 初始化波特率为115200*/
    serial_init,        /* 初始化串口通信设置，在drivers/serial/serial.c文件中定义*/
    console_init_f,     /* 初始化控制台，在common/console.c文件中定义；在zynq中为将gd->have_console 置1*/
#if defined(CONFIG_BOARD_POSTCLK_INIT)
    board_postclk_init,
#endif
#ifdef CONFIG_FSL_ESDHC
    get_clocks,         //获取了SD卡外设的时钟（sdhc_clk），在arch/arm/imx-common/speed.c文件中定义；
#endif
#ifdef CONFIG_M68K
    get_clocks,
#endif
    env_init,       /* initialize environment 初始化环境变量 准确定义待定*/
#if defined(CONFIG_8xx_CPUCLK_DEFAULT)
    /* get CPU and bus clocks according to the environment variable */
    get_clocks_866,
    /* adjust sdram refresh rate according to the new clock */
    sdram_adjust_866,
    init_timebase,
#endif
     
#ifdef CONFIG_SANDBOX
    sandbox_early_getopt_check,
#endif
#ifdef CONFIG_OF_CONTROL
    fdtdec_prepare_fdt,
#endif
    display_options,    /* say that we are here 打印uboot版本信息和编译信息，在lib/display_options.c文件中定义*/
    display_text_info,  /* show debugging info if required 用来显示CPU信息和主频，在arch/arm/imx-common/cpu.c文件中定义*/
#if defined(CONFIG_MPC8260)
    prt_8260_rsr,
    prt_8260_clks,
#endif /* CONFIG_MPC8260 */
#if defined(CONFIG_MPC83xx)
    prt_83xx_rsr,
#endif
#if defined(CONFIG_PPC) || defined(CONFIG_M68K)
    checkcpu,
#endif
    print_cpuinfo,      /* display cpu info (and speed) 打印cpu信息*/
#if defined(CONFIG_MPC5xxx)
    prt_mpc5xxx_clks,
#endif /* CONFIG_MPC5xxx */
#if defined(CONFIG_DISPLAY_BOARDINFO)
    show_board_info,    /* 打印板级信息 */
#endif
    INIT_FUNC_WATCHDOG_INIT     /* 看门狗初始化 */
#if defined(CONFIG_MISC_INIT_F)
    misc_init_f,
#endif
    INIT_FUNC_WATCHDOG_RESET    /* 重置看门狗 */
#if defined(CONFIG_HARD_I2C) || defined(CONFIG_SYS_I2C)
    init_func_i2c,              /* 初始化i2c */
#endif
#if defined(CONFIG_HARD_SPI)
    init_func_spi,
#endif
    announce_dram_init,         /* DRAM 初始化声明即打印一串字符 */
    /* TODO: unify all these dram functions? */
#if defined(CONFIG_ARM) || defined(CONFIG_X86) || defined(CONFIG_MICROBLAZE) || defined(CONFIG_AVR32)
    dram_init,      /* configure available RAM banks 设置gd->ram_size的值 */
#endif
#if defined(CONFIG_MIPS) || defined(CONFIG_PPC) || defined(CONFIG_M68K)
    init_func_ram,
#endif
#ifdef CONFIG_POST
    post_init_f,
#endif
    INIT_FUNC_WATCHDOG_RESET    /* 重置看门狗 */
#if defined(CONFIG_SYS_DRAM_TEST)
    testdram,
#endif /* CONFIG_SYS_DRAM_TEST */
    INIT_FUNC_WATCHDOG_RESET    /* 重置看门狗 */
#ifdef CONFIG_POST
    init_post,
#endif
    INIT_FUNC_WATCHDOG_RESET    /* 重置看门狗 */
    /*
     * Now that we have DRAM mapped and working, we can
     * relocate the code and continue running from DRAM.
     *
     * Reserve memory at end of RAM for (top down in that order):
     *  - area that won't get touched by U-Boot and Linux (optional)
     *  - kernel log buffer
     *  - protected RAM
     *  - LCD framebuffer
     *  - monitor code
     *  - board info struct
     */
    setup_dest_addr,            /* 更新设置u-boot的入口地址 */
#if defined(CONFIG_BLACKFIN) || defined(CONFIG_NIOS2)
    /* Blackfin u-boot monitor should be on top of the ram */
    reserve_uboot,
#endif
#if defined(CONFIG_LOGBUFFER) && !defined(CONFIG_ALT_LB_ADDR)
    reserve_logbuffer,
#endif
#ifdef CONFIG_PRAM
    reserve_pram,
#endif
    reserve_round_4k,           /*留出4K大小的环形区域 * /
#if !(defined(CONFIG_SYS_ICACHE_OFF) && defined(CONFIG_SYS_DCACHE_OFF)) && \
        defined(CONFIG_ARM)
    reserve_mmu,                /* 划分tlb的大小和位置 */
#endif
#ifdef CONFIG_LCD
    reserve_lcd,
#endif
    reserve_trace,
    /* TODO: Why the dependency on CONFIG_8xx? */
#if defined(CONFIG_VIDEO) && (!defined(CONFIG_PPC) || defined(CONFIG_8xx)) && \
        !defined(CONFIG_ARM) && !defined(CONFIG_X86) && \
        !defined(CONFIG_BLACKFIN) && !defined(CONFIG_M68K)
    reserve_video,
#endif
#if !defined(CONFIG_BLACKFIN) && !defined(CONFIG_NIOS2)
    reserve_uboot,
#endif
#ifndef CONFIG_SPL_BUILD
    reserve_malloc,
    reserve_board,
#endif
    setup_machine,
    reserve_global_data,            /* 划分gd->data的空间 */
    reserve_fdt,                    /* 划分gd->fdt的空间 */
    reserve_arch,
    reserve_stacks,                 /* 划分gd->stack 空间 */
    setup_dram_config,              /* 更新dram的配置，包括首地址和size */
    show_dram_config,               /* dram的配置信息，包括大小，有多少个bank... */
#if defined(CONFIG_PPC) || defined(CONFIG_M68K)
    setup_board_part1,
    INIT_FUNC_WATCHDOG_RESET
    setup_board_part2,
#endif
    display_new_sp,                 /* 给出新的sp指针位置 */
#ifdef CONFIG_SYS_EXTBDINFO
    setup_board_extra,
#endif
    INIT_FUNC_WATCHDOG_RESET        /* 重置看门狗 */
    reloc_fdt,                      /* FDT空间重定位 */
    setup_reloc,                    /* ？？？应该是配置新的reloc空间 */
#if defined(CONFIG_X86) || defined(CONFIG_ARC)
    copy_uboot_to_ram,
    clear_bss,
    do_elf_reloc_fixups,
#endif
#if !defined(CONFIG_ARM) && !defined(CONFIG_SANDBOX)
    jump_to_copy,
#endif
    NULL,
};
```

####  2、_main函数中间部分操作：

```
#if ! defined(CONFIG_SPL_BUILD)
/*
 * Set up intermediate environment (new sp and gd) and call
 * relocate_code(addr_moni). Trick here is that we'll return
 * 'here' but relocated.
 */
    ldr sp, [r9, #GD_START_ADDR_SP] /* sp = gd->start_addr_sp */
    bic sp, sp, #7  /* 8-byte alignment for ABI compliance */
    ldr r9, [r9, #GD_BD]        /* r9 = gd->bd */
    sub r9, r9, #GD_SIZE        /* new GD is below bd */
    adr lr, here
    ldr r0, [r9, #GD_RELOC_OFF]     /* r0 = gd->reloc_off */
    add lr, lr, r0
    ldr r0, [r9, #GD_RELOCADDR]     /* r0 = gd->relocaddr */
    b   relocate_code
here:
/*
 * now relocate vectors
 */
    bl  relocate_vectors
/* Set up final (full) environment */
    bl  c_runtime_cpu_setup /* we still call old routine here */
#endif
```

| 指令  | 用法             | 含义                                                         |
| :---- | :--------------- | :----------------------------------------------------------- |
| ldmia | `r1!, {r10-r11}` | LDMIA 中的 I 是 increase 的缩写，A 是 after 的缩写，LD加载(load)的意思 R0后面的感叹号“！表示会自动调节 R0里面的指针，*ld代表load* *其含义是将基址寄存器r1开始的连续2个地址单元的值分别赋给r10,r11，注意的是r0指定的地址每次赋一次r0会加1, * 还有一种是STMDB R1!, {R0,R4-R12} 这就和上面反过来了，ST是存储（store）的意思，D是decrease的意思，B是before的意思，整句话就是R1的存储地址由高到低递减，将R0,R4-R12里的内容存储到R1任务栈里面。 |
| stmia | `r0!, {r10-r11}` | *将寄存器r10到r11的值依次赋值给r0指定的地址单元，每次赋值一次r0就加1* st代表store |



```
ENTRY(relocate_code)
    ldr r1, =__image_copy_start /* r1 <- SRC &__image_copy_start */
    subs    r4, r0, r1      /* r4 <- relocation offset */
    beq relocate_done       /* skip relocation 和谁比较？？*/
    ldr r2, =__image_copy_end   /* r2 <- SRC &__image_copy_end */
copy_loop:
    ldmia   r1!, {r10-r11}      /* copy from source address [r1]    */
    stmia   r0!, {r10-r11}      /* copy to   target address [r0]    */
    cmp r1, r2          /* until source end address [r2]    */
    blo copy_loop       /* R1 < R2 则执行跳转 */
    /*
     * fix .rel.dyn relocations
     */
    ldr r2, =__rel_dyn_start    /* r2 <- SRC &__rel_dyn_start */
    ldr r3, =__rel_dyn_end  /* r3 <- SRC &__rel_dyn_end */
fixloop:
    ldmia   r2!, {r0-r1}        /* (r0,r1) <- (SRC location,fixup) */
    and r1, r1, #0xff
    cmp r1, #23         /* relative fixup? */
    bne fixnext         /* R1 != #23 则执行跳转到fixnext * /
    /* relative fix: increase location by offset */
    add r0, r0, r4
    ldr r1, [r0]
    add r1, r1, r4
    str r1, [r0]    /* 将r1寄存器的值存到r0中保存的地址 */
fixnext:
    cmp r2, r3
    blo fixloop     /* R2 < R3则执行跳转到fixloop */
relocate_done:
#ifdef __XSCALE__
    /*
     * On xscale, icache must be invalidated and write buffers drained,
     * even with cache disabled - 4.2.7 of xscale core developer's manual
     */
    mcr p15, 0, r0, c7, c7, 0   /* invalidate icache */
    mcr p15, 0, r0, c7, c10, 4  /* drain write buffer */
#endif
    /* ARMv4- don't know bx lr but the assembler fails to see that */
#ifdef __ARM_ARCH_4__
    mov pc, lr
#else
    bx  lr                      /* 子程序返回指令 */
#endif
ENDPROC(relocate_code)
```

c_runtime_cpu_setup函数：

```
ENTRY(c_runtime_cpu_setup)
/*
 * If I-cache is enabled invalidate it
 */
#ifndef CONFIG_SYS_ICACHE_OFF           /* 宏未定义，直接执行子程序返回指令 */
    mcr p15, 0, r0, c7, c5, 0   @ invalidate icache
    mcr     p15, 0, r0, c7, c10, 4  @ DSB
    mcr     p15, 0, r0, c7, c5, 4   @ ISB
#endif
    bx  lr
ENDPROC(c_runtime_cpu_setup)
```



####  

#### 3、后半部分跳转到board_init_r前的操作

```
#if !defined(CONFIG_SPL_BUILD) || defined(CONFIG_SPL_FRAMEWORK)
# ifdef CONFIG_SPL_BUILD
    /* Use a DRAM stack for the rest of SPL, if requested */
    bl  spl_relocate_stack_gd
    cmp r0, #0
    movne   sp, r0
# endif
    ldr r0, =__bss_start    /* this is auto-relocated! */
#ifdef CONFIG_USE_ARCH_MEMSET
    ldr r3, =__bss_end      /* this is auto-relocated! */
    mov r1, #0x00000000     /* prepare zero to clear BSS */
    subs    r2, r3, r0      /* r2 = memset len */
    bl  memset
#else
    ldr r1, =__bss_end      /* this is auto-relocated! 获取bss段尾地址*/
    mov r2, #0x00000000     /* prepare zero to clear BSS 将r2置0*/
clbss_l:
    cmp r0, r1              /* while not at end of BSS 比较r0和r1*/
    strlo r2, [r0]          /* clear 32-bit BSS word 在r0 < R1 时执行将R2寄存器中的数据保存到R0寄存器指向的地址*/
    addlo r0, r0, #4        /* move to next 在r0 < r1时 执行r0+4存到r0中*/
    blo clbss_l             /* R0 < R1时执行跳转到clbss_l
#endif
#if ! defined(CONFIG_SPL_BUILD)
    bl coloured_LED_init
    bl red_led_on
#endif
    /* call board_init_r(gd_t *id, ulong dest_addr) */
    mov     r0, r9                  /* gd_t */
    ldr r1, [r9, #GD_RELOCADDR] /* dest_addr */
    /* call board_init_r */
    ldr pc, =board_init_r   /* this is auto-relocated! PC 更新为board_init_r入口地址，即跳转到函数board_init_r */
    /* we should not return here. */
#endif
ENDPROC(_main)
```

在board_init_r中其主要工作在于initcall_run_list(init_sequence_r)

##### 进一步了解board_init_r，在board_init_r函数中核心部分是执行初始化函数序列 init_sequence_r：

```
init_fnc_t init_sequence_r[] = {
    initr_trace,
    initr_reloc,            //
    /* TODO: could x86/PPC have this also perhaps? */
#ifdef CONFIG_ARM
    initr_caches,           // 初始化cache，使能cache；
#endif
    initr_reloc_global_data,//设置gd->
#if defined(CONFIG_SYS_INIT_RAM_LOCK) && defined(CONFIG_E500)
    initr_unlock_ram_in_cache,
#endif
    initr_barrier,
    initr_malloc,           //初始化malloc空间,设置首地址和长度
#ifdef CONFIG_SYS_NONCACHED_MEMORY
    initr_noncached,
#endif
    bootstage_relocate,     //???能看懂但是不确定在干嘛
#ifdef CONFIG_DM
    initr_dm,               //???能看懂但是不确定在干嘛
#endif
#ifdef CONFIG_ARM
    board_init, /* Setup chipselects FEC初始化，在board/freescale/mx6ull_toto/mx6ull_toto.c文件中定义*/
#endif
    /*
     * TODO: printing of the clock inforamtion of the board is now
     * implemented as part of bdinfo command. Currently only support for
     * davinci SOC's is added. Remove this check once all the board
     * implement this.
     */
#ifdef CONFIG_CLOCKS
    set_cpu_clk_info,   /* Setup clock information 设置时钟频率并初始化时钟*/
#endif
    stdio_init_tables,  /* 初始化链表（）*/
    initr_serial,       /* 一系列的初始化操作 */
    initr_announce,     /* 给出gd->ralocaddr的位置 */
    INIT_FUNC_WATCHDOG_RESET
#ifdef CONFIG_NEEDS_MANUAL_RELOC
    initr_manual_reloc_cmdtable,
#endif
#if defined(CONFIG_PPC) || defined(CONFIG_M68K)
    initr_trap,
#endif
#ifdef CONFIG_ADDR_MAP
    initr_addr_map,
#endif
#if defined(CONFIG_BOARD_EARLY_INIT_R)
    board_early_init_r,
#endif
    INIT_FUNC_WATCHDOG_RESET    /* 看门狗重启 */
#ifdef CONFIG_LOGBUFFER
    initr_logbuffer,
#endif
#ifdef CONFIG_POST
    initr_post_backlog,
#endif
    INIT_FUNC_WATCHDOG_RESET    /* 看门狗重启 */
#ifdef CONFIG_SYS_DELAYED_ICACHE
    initr_icache_enable,
#endif
#if defined(CONFIG_PCI) && defined(CONFIG_SYS_EARLY_PCI_INIT)
    /*
     * Do early PCI configuration _before_ the flash gets initialised,
     * because PCU ressources are crucial for flash access on some boards.
     */
    initr_pci,
#endif
#ifdef CONFIG_WINBOND_83C553
    initr_w83c553f,
#endif
#ifdef CONFIG_ARCH_EARLY_INIT_R
    arch_early_init_r,
#endif
    power_init_board,           /* 无操作直接返回 */
#ifndef CONFIG_SYS_NO_FLASH
    initr_flash,                /* 初始化flash，具体操作暂未看懂 */
#endif
    INIT_FUNC_WATCHDOG_RESET    /* 看门狗重启 */
#if defined(CONFIG_PPC) || defined(CONFIG_M68K)
    /* initialize higher level parts of CPU like time base and timers */
    cpu_init_r,
#endif
#ifdef CONFIG_PPC
    initr_spi,
#endif
#if defined(CONFIG_X86) && defined(CONFIG_SPI)
    init_func_spi,
#endif
#ifdef CONFIG_CMD_NAND
    initr_nand,                 /* 初始化nand 没看太懂*/
#endif
#ifdef CONFIG_CMD_ONENAND
    initr_onenand,              /* 初始化nand 没看太懂*/
#endif
#ifdef CONFIG_GENERIC_MMC
    initr_mmc,                  /* 初始化emmc，在common/board_r.c文件中定义 */
#endif
#ifdef CONFIG_HAS_DATAFLASH
    initr_dataflash,
#endif
    initr_env,                  /* 初始化环境变量 */
#ifdef CONFIG_SYS_BOOTPARAMS_LEN
    initr_malloc_bootparams,
#endif
    INIT_FUNC_WATCHDOG_RESET    /* 看门狗重启 */
    initr_secondary_cpu,        /* 无具体操作 直接返回*/
#ifdef CONFIG_SC3
    initr_sc3_read_eeprom,
#endif
#if defined(CONFIG_ID_EEPROM) || defined(CONFIG_SYS_I2C_MAC_OFFSET)
    mac_read_from_eeprom,
#endif
    INIT_FUNC_WATCHDOG_RESET
#if defined(CONFIG_PCI) && !defined(CONFIG_SYS_EARLY_PCI_INIT)
    /*
     * Do pci configuration
     */
    initr_pci,
#endif
    stdio_add_devices,          /* stdio_add_devices执行stdio设备的注册。stdio设备层工作在serail层之上，但stdio设备并不限于serial设备，比如标准输出设备可能包括显示器 */
                                /* 包括i2c_init_call 、drv_system_init和serial_stdio_init
                                /* serial设备到stdio的注册通过函数serial_stdio_ini来实现，其在drivres/serial/serial.c中定义
                                /* 用i2c_set_bus_num()函数去将i2c切换到需要的总线上 */
                                /* 调用drv_system_init ()注册串口设备到设备列表中 */
    initr_jumptable,            /* 给gd->jt赋值 */
#ifdef CONFIG_API
    initr_api,                  /* 初始化u-boot下的部分api 好像和命令相关  */
#endif
    console_init_r,     /* fully init console as a device 初始化控制台，在common/console.c文件中定义 */
#ifdef CONFIG_DISPLAY_BOARDINFO_LATE
    show_board_info,    /* 给出板级信息相关 Xilinx Zynq */
#endif
#ifdef CONFIG_ARCH_MISC_INIT
    arch_misc_init,     /* miscellaneous arch-dependent init */
#endif
#ifdef CONFIG_MISC_INIT_R
    misc_init_r,        /* miscellaneous platform-dependent init */
#endif
    INIT_FUNC_WATCHDOG_RESET
#ifdef CONFIG_CMD_KGDB
    initr_kgdb,
#endif
    interrupt_init,                 /* 初始化中断 在arch/arm/lib/interrupts.c文件中定义*/
#if defined(CONFIG_ARM) || defined(CONFIG_AVR32)
    initr_enable_interrupts,        /* 使能中断 在arch/arm/lib/interrupts.c文件中定义*/
#endif
#if defined(CONFIG_X86) || defined(CONFIG_MICROBLAZE) || defined(CONFIG_AVR32) \
    || defined(CONFIG_M68K)
    timer_init,     /* initialize timer */
#endif
#if defined(CONFIG_STATUS_LED) && defined(STATUS_LED_BOOT)
    initr_status_led,               /* 初始化led状态灯 */
#endif
    /* PPC has a udelay(20) here dating from 2002. Why? */
#ifdef CONFIG_CMD_NET
    initr_ethaddr,                  /* 初始化网络地址，获取MAC地址，读取环境变量ethaddr的值 */
#endif
#ifdef CONFIG_BOARD_LATE_INIT
    board_late_init,                /* Get the bootmode register value 获取boot方式寄存器值将其保存到环境变量中 */
#endif
#ifdef CONFIG_CMD_SCSI
    INIT_FUNC_WATCHDOG_RESET
    initr_scsi,                     /* 实际运行函数所在的宏未定义 */
#endif
#ifdef CONFIG_CMD_DOC
    INIT_FUNC_WATCHDOG_RESET
    initr_doc,                      /* 未知待定 */
#endif
#ifdef CONFIG_BITBANGMII
    initr_bbmii,
#endif
#ifdef CONFIG_CMD_NET
    INIT_FUNC_WATCHDOG_RESET
    initr_net,                      /* 初始化网络设备，函 数 调 用 顺 序 为 ：initr_net->eth_initialize->board_eth_init()，在common/board_r.c文件中定义 */
#endif
#ifdef CONFIG_POST
    initr_post,
#endif
#if defined(CONFIG_CMD_PCMCIA) && !defined(CONFIG_CMD_IDE)
    initr_pcmcia,
#endif
#if defined(CONFIG_CMD_IDE)
    initr_ide,                      /* 初始化ide，并打印出ide相关信息 */
#endif
#ifdef CONFIG_LAST_STAGE_INIT
    INIT_FUNC_WATCHDOG_RESET
    /*
     * Some parts can be only initialized if all others (like
     * Interrupts) are up and running (i.e. the PC-style ISA
     * keyboard).
     */
    last_stage_init,
#endif
#ifdef CONFIG_CMD_BEDBUG
    INIT_FUNC_WATCHDOG_RESET
    initr_bedbug,                   /* 实际语句被未定义宏限制，当前为直接返回 */
#endif
#if defined(CONFIG_PRAM) || defined(CONFIG_LOGBUFFER)
    initr_mem,
#endif
#ifdef CONFIG_PS2KBD
    initr_kbd,
#endif
    run_main_loop,                  /* 主循环，处理命令 */
};
```

#### 4、run_main_loop

uboot启动以后会进入3秒倒计时，如果在3秒倒计时结束之前按下按下回车键，那么就会进入uboot的命令模式，如果倒计时结束以后都没有按下回车键，那么就会自动启动Linux内核，这个功能就是由run_main_loop函数来完成的。

```
static int run_main_loop(void)
{
    /* main_loop() can return to retry autoboot, if so just run it again */
    for (;;)
        main_loop();
    return 0;
}
```

##### main_loop函数定义：

```
void main_loop(void)
{
    const char *s;
    bootstage_mark_name(BOOTSTAGE_ID_MAIN_LOOP, "main_loop");         /* 调用bootstage_mark_name函数，打印出启动进度 */
    modem_init();
    cli_init();                                                         /* cli_init函数，初始化hushshell相关的变量 */
    run_preboot_environment_command();
    s = bootdelay_process();                                            /* 此函数会读取环境变量bootdelay和bootcmd的内容，然后将bootdelay的值赋值给全局变量stored_bootdelay，返回值为环境变量bootcmd的值*/
    if (cli_process_fdt(&s))                                           
        cli_secure_boot_cmd(s);
    autoboot_command(s);                                                /* autoboot_command函数，此函数就是检查倒计时是否结束？倒计时结束之前有没有被打断？在文件common/autoboot.c文件中定义，具体代码如下 */
    cli_loop();                                                         /* 死循环，处理在ctrl + u打断情况下用户输入的命令 */
}
```

u-boot启动到启动内核的程序调用在 autoboot_command函数中倒计时结束不按下所执行的语句中

```
void autoboot_command(const char *s)
{
    debug("### main_loop: bootcmd=\"%s\"\n", s ? s : "<UNDEFINED>");
    if (stored_bootdelay != -1 && s && !abortboot(stored_bootdelay)) {      /* 按下则不执行if{}里面的语句直接结束本函数，不按下则执行if{}里的语句
        printf("no press any key\n");
        auto_net_upgrade();
        run_command_list(s, -1, 0);
    }
}
```

其中延时按下相关的函数在abortboot

```
static int abortboot(int bootdelay)
{
    return abortboot_normal(bootdelay);
}
 
static int abortboot_normal(int bootdelay)
{
    int abort = 0;
    unsigned long ts;
    int hit_cnt = 0;
    if (bootdelay >= 0)
        printf("Hit %d times Ctrl+u to stop autoboot: %2d ", MAX_HIT_KEY_CNT, bootdelay);
 
    while ((bootdelay > 0) && (!abort)) {
        --bootdelay;
        /* delay 1000 ms */
        ts = get_timer(0);
        do {
            if (tstc()) {   /* we got a key press   */
                int c = getc();
                if (c == 0x15) {
                    hit_cnt++;
                } else {
                    hit_cnt = 0;
                }
                 
                if (hit_cnt >= MAX_HIT_KEY_CNT) {
                    abort  = 1; /* don't auto boot  */
                    bootdelay = 0;  /* no more delay    */
                    break;
                }
            }
            udelay(1000);
        } while (!abort && get_timer(ts) < 1000);
        printf("\b\b\b%2d ", bootdelay);
    }
    putc('\n');
    return abort;
}
```

如果在倒计时结束前按下ctrl + u 则执行：cli_loop();

```
void cli_loop(void)
{
    parse_file_outer();             /*
    /* This point is never reached */
    for (;;);
}
```

如果知道倒计时结束也不按下则执行：

```
auto_net_upgrade();
run_command_list(s, -1, 0);
```