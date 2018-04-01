# Lab01 实验报告
## 实验内容：
使用调试跟踪工具，追踪自己的操作系统（建议Linux）的启动过程，并找出其中至少两个关键事件。
## 环境搭建：
### 下载安装
- 系统环境：Ubuntu 16.04 LTS
- 内核版本：Linux-4.4.125
    ```shell
      # kernel.org 下载Linux内核，点击tarball，下载得到 .tar.xz文件
    ```
- QEMU版本：QEMU emulator version 2.5.0
    ```shell
      sudo apt-get install qemu
    ```
- gdb版本：GNU gdb 7.11.1(系统自带)
- make版本：GNU make 4.1（系统自带）
- gcc版本：5.4.0（系统自带）
### 第一步：编译下载好的Linux内核
- 
    ```shell
    mkdir LinuxKernel
    # 把下载的linux-4.4.125.tar.xz复制到/LinuxKernel文件夹
    tar -d linux-4.4.125.tar.xz
    tar -xvf linux-4.4.125.tar
    cd linux-4.4.125
    # 创建 x86_64 的默认内核配置
    make x86_64defconfig
    # 手动配置内核选项
    make menuconfig
    # 进入图形化设置界面
    # 1.取消 Processor type and features -> Build a relocatable kernel
    #   取消后 Build a relocatable kernel 的子项 Randomize the address of
    #   the kernel image (KASLR) 也会一并被取消
    # 2.打开 Kernel hacking -> Compile-time checks and compiler options 下#   的选项：
    #   Compile the kernel with debug info
    #       Generate dwarf4 debuginfo
    #   Compile the kernel with frame pointers
    # 完成设置后save为 .config 之后exit即可
    # 设置完成后即可开始编译
    make -j 8
    # 用8个线程编译，加快编译速度
    # 编译需要一段时间，编译完成后，内核的位置为 ./arch/x86/boot/bzImage
    ```
    - 遇到的问题和解决：
        - make menuconfig时提示报错：“fatal error: 没有那个文件”，使用apt install ncurses-devel命令安装该库，没有，然后又使用apt install ncurses，还是没有该库。根据[https://blog.csdn.net/WANG__RONGWEI/article/details/54846759]()
        ncurses-devel库在Ubuntu中以‘libncureses5-dev’命名，只需：
        ```shell
         sudo apt-get install libncurses5-dev
        ```
### 第二步：制作根文件系统
- 
    ```shell
    cd ..
    mkdir rootFileSystem
    git clone https://github.com/mengning/menu.git
    cd menu
    gcc -o init linktable.c menu.c test.c -m32 -static -lpthread
    # init是系统启动后默认启动1号进程，init是第一个用户态进程，
    # 第一个用户态进程是1号进程
    cd ../rootFileSystem
    cp ../menu/init ./ 
    # 把init拷贝到rootFileSystem目录下边
    find . | cpio -o -Hnewc |gzip -9 > ../rootFileSystem.img
    # 使用cpio方式把当前rootfs目录下所有的文件打包成rootfs.img一个镜像文件
    # 至此一个最简单的根文件系统镜像制作完毕
    ```
## 调试Linux内核
### 第一步：qemu启动内核
- 
    ```shell
    cd /path/to/linux-4.4.125
    # 选项说明
    # -m 指定内存数量
    # -kernel 指定 bzImage 的镜像路径
    # -s 等价于 -gdb tcp::1234 表示监听 1234 端口，用于 gdb 连接
    # -S 表示加载后立即暂停，等待调试指令。不设置这个选项内核会直接执行
    # -nographic 以及后续的指令用于将输出重新向到当前的终端中，这样就能方便的用滚屏
    # 查看内核的输出日志了。
    # -initrd 指向根文件系统镜像
    qemu-system-x86_64 -kernel ./arch/x86/boot/bzImage -s -S -nographic -serial mon:stdio -append "console=ttyS0" -initrd ../rootFileSystem.img
    ```
    这个时候 qemu 会进入暂停状态，如果需要打开 qemu 控制台，可以输入CTRL + A 然后按 C。

- 
    在另一个终端中执行gdb，推荐使用terminator，分屏查看很好用。
    ```shell
    cd /path/to/linux-4.4.125
    gdb vmlinux
    # vmlinux 是编译内核时生成的调试文件，在内核源码的根目录中。
    # 进入 gdb 的交互模式后，首先执行
    gdb show arch
    # 当前架构一般是: i386:x86-64
    # 连接 qemu 进行调试：
    target remote :1234
    # 设置断点
    # 如果上面qemu是使用 qemu-kvm 执行的内核的话，就需要使用 hbreak 来设置断点，否则断点无法生效。
    # 但是我们使用的是 qemu-system-x86_64，所以可以直接使用 b 命令设置断点。
    b start_kernel
    # 执行内核
    c
    ```
    执行内核后gdb会出现一个错误：
    ```shell
    Remote 'g' packet reply is too long: 后续一堆的十六进制数字
    ```
    解决方法：
    ```shell
    # 断开 gdb 的连接
    disconnect
    # 重新设置 arch
    # 此处设置和之前 show arch 的要不一样
    # 之前是  i386:x86-64 于是改成  i386:x86-64:intel
    set arch i386:x86-64:intel
    # 重新连接
    target remote :1234
    ```
    连接上后就可以看到 gdb 正常的输出 start_kernel 处的代码，然后按照 gdb 的调试指令进行内核调试即可。
    ```shell
    (gdb) target remote :1234
    Remote debugging using :1234
    start_kernel () at init/main.c:500
    500	{
    ```
    按c继续执行内核程序，内核启动成功后停止
    输入help可以查询指令

    ![](os_success.png)
    ![](os_success2.png)
### gdb调试
#### 关键事件一————start_kernel()
- 输入list查看start_kernel附近的代码：
    ```shell
    (gdb) b start_kernel
    (gdb) c
    (gdb) target remote :1234
    Remote debugging using :1234
    start_kernel () at init/main.c:500
    500	{
    (gdb) list
    495		ioremap_huge_init();
    496		kaiser_init();
    497	}
    498	
    499	asmlinkage __visible void __init start_kernel(void)
    500	{
    501		char *command_line;
    502		char *after_dashes;
    503	
    504		/*
    ```
#### start_kernel()分析
- 在构架相关的汇编代码运行完之后，程序跳入了构架无关的内核C语言代码：init/main.c中的start_kernel函数，在这个函数中Linux内核开始真正进入初始化阶段，进行一系列与内核相关的初始化后，调用第一个用户进程－init 进程并等待用户进程的执行，这样整个 Linux 内核便启动完毕。该函数所做的具体工作有：
    1) 调用 setup_arch()函数进行与体系结构相关的第一个初始化工作；对不同的体系结构来说该函数有不同的定义。对于 ARM 平台而言，该函数定义在arch/arm/kernel/Setup.c。它首先通过检测出来的处理器类型进行处理器内核的初始化，然后通过 bootmem_init()函数根据系统定义的 meminfo 结构进行内存结构的初始化，最后调用paging_init()开启 MMU，创建内核页表，映射所有的物理内存和 IO空间。
    
    2) 创建异常向量表和初始化中断处理函数；
    
    3) 初始化系统核心进程调度器和时钟中断处理机制；
    
    4) 初始化串口控制台（serial-console）；
    
    5) 创建和初始化系统 cache，为各种内存调用机制提供缓存，包括;动态内存分配，虚拟文
    件系统（VirtualFile System）及页缓存。
    
    6) 初始化内存管理，检测内存大小及被内核占用的内存情况；
    
    7) 初始化系统的进程间通信机制（IPC）
#### 关键事件二————rest_init()
- 
    ```shell
    (gdb) b rest_init
    Breakpoint 2 at 0xffffffff81892990: file init/main.c, line 388.
    (gdb) c
    Continuing.

    Breakpoint 2, rest_init () at init/main.c:388
    388	{
    (gdb) list
    383	 */
    384	
    385	static __initdata DECLARE_COMPLETION(kthreadd_done);
    386	
    387	static noinline void __init_refok rest_init(void)
    388	{
    389		int pid;
    390	
    391		rcu_scheduler_starting();
    392		smpboot_thread_init();
    ```
    源代码分析

```cpp
asmlinkage void __init start_kernel(void)
{
	char * command_line;
	extern const struct kernel_param __start___param[], __stop___param[];
	/*这两个变量为地址指针，指向内核启动参数处理相关结构体段在内存中的位置（虚拟地址）。
	声明传入参数的外部参数对于ARM平台，位于 include\asm-generic\vmlinux.lds.h*/
	/*
	 * Need to run as early as possible, to initialize the
	 * lockdep hash:
	 lockdep是一个内核调试模块，用来检查内核互斥机制（尤其是自旋锁）潜在的死锁问题。
	 */
	lockdep_init();//初始化内核依赖的关系表，初始化hash表
	smp_setup_processor_id();//获取当前CPU,单处理器为空
	debug_objects_early_init();//对调试对象进行早期的初始化,其实就是HASH锁和静态对象池进行初始化

	/*
	 * Set up the the initial canary ASAP:
	 初始化栈canary值
	 canary值的是用于防止栈溢出攻击的堆栈的保护字 。
	 */
	boot_init_stack_canary();
	/*1.cgroup: 它的全称为control group.即一组进程的行为控制. 
	2.该函数主要是做数据结构和其中链表的初始化 
	3.参考资料： Linux cgroup机制分析之框架分析
	*/
	cgroup_init_early();

	local_irq_disable();//关闭系统总中断（底层调用汇编指令）
	early_boot_irqs_disabled = true;

/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
	tick_init();//1.初始化内核时钟系统
	boot_cpu_init();//1.激活当前CPU（在内核全局变量中将当前CPU的状态设为激活状态）
	page_address_init();//高端内存相关，未定义高端内存的话为空函数
	printk(KERN_NOTICE "%s", linux_banner);
	/*1.内核构架相关初始化函数,可以说是非常重要的一个初始化步骤。
	其中包含了处理器相关参数的初始化、内核启动参数（tagged list）的获取和前期处理、
	内存子系统的早期的初始化（bootmem分配器）。 主要完成了4个方面的工作，一个就是取得MACHINE和PROCESSOR的信息然或将他们赋值
	给kernel相应的全局变量，然后呢是对boot_command_line和tags接行解析，再然后呢就是
	memory、cach的初始化，最后是为kernel的后续运行请求资源″**/
	setup_arch(&command_line);
	/*1.初始化代表内核本身内
	存使用的管理结构体init_mm。 
	2.ps：每一个任务都有一个mm_struct结构以管理内存空间，init_mm是内核的mm_struct，其中： 
	3.设置成员变量* mmap指向自己，意味着内核只有一个内存管理结构; 
	4.设置* pgd=swapper_pg_dir，swapper_pg_dir是内核的页目录(在arm体系结构有16k， 所以init_mm定义了整个kernel的内存空间)。 
	5.这些内容涉及到内存管理子系统*/
	mm_init_owner(&init_mm, &init_task);
	mm_init_cpumask(&init_mm);//初始化CPU屏蔽字
	/*1.对cmdline进行备份和保存：保存未改变的comand_line到字符数组static_command_line［］ 中。保存  boot_command_line到字符数组saved_command_line［］中
	*/
	setup_command_line(command_line);

	/*如果没有定义CONFIG_SMP宏，则这个函数为空函数。如果定义了CONFIG_SMP宏，则这个setup_per_cpu_areas()函数给每个CPU分配内存，并拷贝.data.percpu段的数据。为系统中的每个CPU的per_cpu变量申请空间。
	*/
	/*下面三段1.针对SMP处理器的内存初始化函数，如果不是SMP系统则都为空函数。 (arm为空) 
	2.他们的目的是给每个CPU分配内存，并拷贝.data.percpu段的数据。为系统中的每个CPU的per_cpu变量申请空间并为boot CPU设置一些数据。 
	3.在SMP系统中，在引导过程中使用的CPU称为boot CPU*/
	setup_nr_cpu_ids();
	setup_per_cpu_areas();
	smp_prepare_boot_cpu();	/* arch-specific boot-cpu hooks */


	build_all_zonelists(NULL, NULL);//	建立系统内存页区(zone)链表

	page_alloc_init();//内存页初始化


	printk(KERN_NOTICE "Kernel command line: %s\n", boot_command_line);
	parse_early_param();//	解析早期格式的内核参数
	/*函数对Linux启动命令行参数进行在分析和处理,
	当不能够识别前面的命令时，所调用的函数。*/
	parse_args("Booting kernel", static_command_line, __start___param,
		   __stop___param - __start___param,
		   -1, -1, &unknown_bootoption);
	jump_label_init();
	/*
	 * These use large bootmem allocations and must precede
	 * kmem_cache_init()
	 */
	setup_log_buf(0);
	/*初始化hash表，以便于从进程的PID获得对应的进程描述指针，按照开发办上的物理内存初始化pid hash表
	*/
	pidhash_init();
	vfs_caches_init_early();//建立节点哈希表和数据缓冲哈希表
	sort_main_extable();//对异常处理函数进行排序
	trap_init();//初始化硬件中断
	mm_init();//	Set up kernel memory allocators 	建立了内核的内存分配器
	/*
	 * Set up the scheduler prior starting any interrupts (such as the
	 * timer interrupt). Full topology setup happens at smp_init()
	 * time - but meanwhile we still have a functioning scheduler.
	 */
	sched_init();
	/*
	 * Disable preemption - early bootup scheduling is extremely
	 * fragile until we cpu_idle() for the first time.
	 */
	preempt_disable();//禁止调度
	//	先检查中断是否已经打开，若打开，输出信息后则关闭中断。
	if (!irqs_disabled()) {
		printk(KERN_WARNING "start_kernel(): bug: interrupts were "
				"enabled *very* early, fixing it\n");
		local_irq_disable();
	}
	idr_init_cache();//创建錳dr缓冲区
	perf_event_init();
	rcu_init();//互斥访问机制
	radix_tree_init();
	/* init some links before init_ISA_irqs() */
	early_irq_init();
	init_IRQ();//中断向量初始化
	prio_tree_init();//初始化优先级数组
	init_timers();//定时器初始化
	hrtimers_init();//高精度时钟初始化
	softirq_init();//软中断初始化
	timekeeping_init();//	初始化资源和普通计时器
	time_init();
	profile_init();//	对内核的一个性能测试工具profile进行初始化。
	call_function_init();
	if (!irqs_disabled())
		printk(KERN_CRIT "start_kernel(): bug: interrupts were "
				 "enabled early\n");
	early_boot_irqs_disabled = false;
	local_irq_enable();//使能中断
	kmem_cache_init_late();//kmem_cache_init_late的目的就在于完善slab分配器的缓存机制.

	/*
	 * HACK ALERT! This is early. We're enabling the console before
	 * we've done PCI setups etc, and console_init() must be aware of
	 * this. But we do want output early, in case something goes wrong.
	 */
	console_init();//初始化控制台以显示printk的内容
	if (panic_later)
		panic(panic_later, panic_param);

	lockdep_info();//	如果定义了CONFIG_LOCKDEP宏，那么就打印锁依赖信息，否则什么也不做

	/*
	 * Need to run this when irqs are enabled, because it wants
	 * to self-test [hard/soft]-irqs on/off lock inversion bugs
	 * too:
	 */
	locking_selftest();
#ifdef CONFIG_BLK_DEV_INITRD
	if (initrd_start && !initrd_below_start_ok &&
	    page_to_pfn(virt_to_page((void *)initrd_start)) < min_low_pfn) {
		printk(KERN_CRIT "initrd overwritten (0x%08lx < 0x%08lx) - "
		    "disabling it.\n",
		    page_to_pfn(virt_to_page((void *)initrd_start)),
		    min_low_pfn);
		initrd_start = 0;
	}
#endif
	page_cgroup_init();
	debug_objects_mem_init();
	kmemleak_init();
	setup_per_cpu_pageset();
	numa_policy_init();
	if (late_time_init)
		late_time_init();
	sched_clock_init();
	calibrate_delay();//校准延时函数的精确度
	pidmap_init();//进程号位图初始化，一般用一个錺age来表示所有进程的錺id占用情况
	anon_vma_init();	//	匿名虚拟内存域（ anonymous VMA）初始化
#ifdef CONFIG_X86
	if (efi_enabled)
		efi_enter_virtual_mode();
#endif
	thread_info_cache_init();//获取thread_info缓存空间，大部分构架为空函数（包括ARM
	cred_init();	//任务信用系统初始化。详见：Documentation/credentials.txt
	fork_init(totalram_pages);	//进程创建机制初始化。为内核"task_struct"分配空间，计算最大任务数。
	proc_caches_init();	//初始化进程创建机制所需的其他数据结构，为其申请空间。
	buffer_init();	//缓存系统初始化，创建缓存头空间，并检查其大小限时。
	key_init();	//内核密钥管理系统初始化
	security_init();	//内核安全框架初始?
	dbg_late_init();
	vfs_caches_init(totalram_pages);vfs_caches_init(totalram_pages);//虚拟文件系统（VFS）缓存初始化
	signals_init();//信号管理系统初始化
	/* rootfs populating might need page-writeback */
	page_writeback_init();//页写回机制初始化
#ifdef CONFIG_PROC_FS
	proc_root_init();//proc文件系统初始化
#endif
	cgroup_init();//control group正式初始化
	cpuset_init();//CPUSET初始化。 参考资料：《多核心計算環境—NUMA與CPUSET簡介》
	taskstats_init_early(); //任务状态早期初始化函数：为结构体获取高速缓存，并初始化互斥机制。
	delayacct_init();	//任务延迟初始化
			
	check_bugs();//检查CPU BUG的函数，通过软件规避BUG
	acpi_early_init(); /* before LAPIC and SMP initACPI早期初始化函数。 ACPI - Advanced Configuration and Power Interface高级配置及电源接口 */
	sfi_init_late();//功能跟踪调试机制初始化，ftrace 是 function trace 的简称

	if (efi_enabled)
		efi_free_boot_services();
	ftrace_init();
	/* Do the rest non-__init'ed, we're now alive */
	rest_init();// 虽然从名字上来说是剩余的初始化。但是这个函数中的初始化包含了很多的内容
}

```

#### rest_init()分析
- 在内核初始化函数start_kernel执行到最后，就是调用rest_init函数，这个函数的主要使命就是创建并启动内核线程init。
- 源代码分析


```cpp

static noinline void __init_refok rest_init(void)
{
	int pid;
	rcu_scheduler_starting();//	1.内核RCU锁机制调度启动,因为下面就要用到	
	/*
	 * We need to spawn init first so that it obtains pid 1, however
	 * the init task will end up wanting to create kthreads, which, if
	 * we schedule it before we create kthreadd, will OOPS.	 
	 我们必须先创建init内核线程，这样它就可以获得pid为1。
	 尽管如此init线程将会挂起来等待创建kthreads线程。
	 如果我们在创建kthreadd线程前调度它，就将会出现OOPS。
	 */
	kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
	numa_default_policy();//	1.设定NUMA系统的内存访问策略为默认	
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	/*
	1.创建kthreadd内核线程，它的作用是管理和调度其它内核线程。
	2.它循环运行一个叫做kthreadd的函数，该函数的作用是运行kthread_create_list全局链表中维护的内核线程。
	3.调用kthread_create创建一个kthread，它会被加入到kthread_create_list 链表中；
	4.被执行过的kthread会从kthread_create_list链表中删除；
	5.且kthreadd会不断调用scheduler函数让出CPU。此线程不可关闭。
	
	上面两个线程就是我们平时在Linux系统中用ps命令看到：
	$ ps -A
	PID TTY TIME CMD
	3.1 ? 00:00:00 init
	4.2 ? 00:00:00 kthreadd
	*/
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
	complete(&kthreadd_done);
	
	/*1.获取kthreadd的线程信息，获取完成说明kthreadd已经创建成功。并通过一个
	complete变量（kthreadd_done）来通知kernel_init线程。*/	
	/*
	 * The boot idle thread must execute schedule()
	 * at least once to get things moving:
	 */
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_idle();
}


```