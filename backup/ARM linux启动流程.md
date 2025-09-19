# BootROM

此程序出厂固化在soc内部rom中，soc上电首先执行，主要作用：

1. 初始化时钟、SRAM
2. 根据寄存器选择启动介质
3. 从启动介质搬运loader到SRAM执行

# Loader

由于SRAM一般较小，所以loader体积较小，主要完成以下功能：

1. 初始化DRAM
2. 搬运U-Boot代码到DRAM执行

# U-Boot

U-Boot启动以重定位（relocate）操作分成两个阶段

1. 重定位前（board_init_f）：只做必要的初始化，soc、时钟、DRAM、串口
2. 重定位（relocate）：将自身搬移到DRAM的末尾地址执行
3. 重定位后（board_init_r）：初始化需要的外设（存储、网络等），命令行系统，引导内核

# Kernel

1. 解析命令行参数
2. initcall各类驱动初始化
3. 挂载根文件系统
4. 启动init进程，进入用户空间