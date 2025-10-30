# ARM64平台Linux内核启动流程

ARM64  QEMU仿真平台的构建、Linux源码的编译、QEMU的启动，可查看[[QEMU]-搭建arm64_linux_kernel环境](../../QEMU/[QEMU]-搭建arm64_linux_kernel环境.md)

## kernel启动第一阶段

kernel启动的第一阶段主要为汇编代码，用于**初始化CPU、初始化kernel执行环境，配置C语言执行环境配置页表、配置MMU、跳转至C阶段**；

### kernel第一行代码的入口

在 [[QEMU]-搭建arm64_linux_kernel环境](../../QEMU/[QEMU]-搭建arm64_linux_kernel环境.md) 完成kernel源码的编译后，会生成以下几个文件，分别是：

* `Image`(arch/arm64/boot)：Image就是我们的kernel image；
* `vmlinux`(./)：这是未压缩的、可链接格式（ELF） 的内核可执行文件。可供gdb调试使用的内核文件，禁止编辑，vmlinux.lds文件是在编译时，由vmlinux.lds.S文件对链接器ld的输出进行排序后生成；vmlinux.lds.S是用来对输出文件中的段进行排序，并定义相关的符号名；
* `System.map`(./)：这是一个文本文件，包含了内核符号（函数名、变量名）与其在运行时内存地址的映射关系列表。它是在链接 `vmlinux` 时由链接器生成的。
* `.config`(./)：这是内核编译的**核心配置文件**。它记录了所有配置选项（`CONFIG_XXX=y/m/n`）的设置，决定了哪些功能被编译进内核 (`y`)、编译为模块 (`m`)、还是排除 (`n`)。它由 `make menuconfig`, `make xconfig`, `make defconfig` 等命令生成或修改。
* `modules.builtin`(./)：这是一个文本文件，列出了所有直接编译进内核映像（`vmlinux`） 的模块（即配置为 `=y` 的模块）的路径。
* `modules.builtin.modinfo`(./)：类似于 `modules.builtin`，但包含了编译进内核的模块的元信息（`modinfo` 可以查询的信息）。

> [[内核]编译Linux内核后生成文件及作用](https://zhuanlan.zhihu.com/p/1911116928740750432)

通过`readelf`工具查看vmlinux文件可得如下内容：

```bash
$ readelf -h ./vmlinux
ELF 头：
  Magic：   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              DYN (共享目标文件)
  系统架构:                          AArch64
  版本:                              0x1
  入口点地址：               0xffff800010000000
  程序头起点：          64 (bytes into file)
  Start of section headers:          450798920 (bytes into file)
  标志：             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         3
  Size of section headers:           64 (bytes)
  Number of section headers:         38
  Section header string table index: 37
```

可以看到程序入口点位于地址`0xffff800010000000`，这个入口地址在代码中的哪个位置可以通过反汇编vmlinux文件查看

```bash
$ aarch64-linux-gnu-objdump -dxh vmlinux > vmlinux.s
```

```assembly
vmlinux：     文件格式 elf64-littleaarch64
vmlinux
体系结构：aarch64， 标志 0x00000150：
HAS_SYMS, DYNAMIC, D_PAGED
起始地址 0xffff800010000000

程序头：
    LOAD off    0x0000000000010000 vaddr 0xffff800010000000 paddr 0xffff800010000000 align 2**16
         filesz 0x0000000001deb200 memsz 0x0000000001e6cc94 flags rwx
    NOTE off    0x00000000015695e8 vaddr 0xffff8000115595e8 paddr 0xffff8000115595e8 align 2**2
         filesz 0x000000000000003c memsz 0x000000000000003c flags r--
   STACK off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**4
         filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
私有标志 = 0：

节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .head.text    00010000  ffff800010000000  ffff800010000000  00010000  2**16
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .text         00dd21b8  ffff800010010000  ffff800010010000  00020000  2**11
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .got.plt      00000018  ffff800010de21b8  ffff800010de21b8  00df21b8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
...
SYMBOL TABLE:
ffff800010000000 l    d  .head.text     0000000000000000 .head.text
ffff800010010000 l    d  .text  0000000000000000 .text
ffff800010de21b8 l    d  .got.plt       0000000000000000 .got.plt
ffff800010df0000 l    d  .rodata        0000000000000000 .rodata
ffff8000114f4b20 l    d  .pci_fixup     0000000000000000 .pci_fixup

...
Disassembly of section .head.text:

ffff800010000000 <_text>:
ffff800010000000:       91005a4d        add     x13, x18, #0x16
ffff800010000004:       14557fff        b       ffff800011560000 <primary_entry>
        ...
ffff800010000010:       01e80000        .word   0x01e80000
ffff800010000014:       00000000        .word   0x00000000
ffff800010000018:       0000000a        .word   0x0000000a
```

可以看到，Linux的第一条指令为`add     x13, x18, #0x16`，对应符号表的`.head.text`段，`.head.text`在include/linux/init.h文件中被定义__HEAD宏表示：

```c
/* For assembly routines */
#define __HEAD		.section	".head.text","ax"
#define __INIT		.section	".init.text","ax"
#define __FINIT		.previous
```

通过搜索源码可知，内核入口点位于`arch/arm64/kernel/head.S`中

![__HEAD](image/[Linux]-内核启动流程/__HEAD.png#pic_center)

```assembly
[arch/arm64/kernel/head.s]
...
/*
 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * This code is mostly position independent so you call this at
 * __pa(PAGE_OFFSET).
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
	__HEAD
_head:
	/*
	 * DO NOT MODIFY. Image header expected by Linux boot-loaders.
	 */
#ifdef CONFIG_EFI
	/*
	 * This add instruction has no meaningful effect except that
	 * its opcode forms the magic "MZ" signature required by UEFI.
	 */
	add	x13, x18, #0x16
	b	primary_entry
#else
	b	primary_entry			// branch to kernel start, magic
	.long	0				// reserved
#endif
	.quad	0				// Image load offset from start of RAM, little-endian
	le64sym	_kernel_size_le			// Effective size of kernel image, little-endian
	le64sym	_kernel_flags_le		// Informative flags, little-endian
...
```

在__HEAD中第一个执行的指令为`add x13, x18, #0x16`，与反汇编查看的第一条语句一致，之后转到`primary_entry`函数。

查看`head.s`文件的头部注释信息可知内核启动的必要条件：**关闭 MMU、关闭D-cache、x0传递给FDT blob的物理地址**。

```c
/*
 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * This code is mostly position independent so you call this at
 * __pa(PAGE_OFFSET).
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
```

### primary_entry函数

启动过程中的汇编阶段，是从arch/arm64/kernel/head.S文件开始，其内部只包含一个主要函数`primary_entry` ，经过层层调用，head.s汇编代码的的调用如下，最后执行`start_kernel`函数，内核启动进入C语言阶段。

```assembly
__HEAD
	primary_entry
		preserve_boot_args
		el2_setup
		set_cpu_boot_mode_flag
		__create_page_tables
		__cpu_setup
		__primary_switch
			__enable_mmu
			__relocate_kernel
			__primary_switched
				__pi_memset
				kaslr_early_init
				start_kernel
```

`primary_entry`函数的整体调用如下：

```assembly
[arch/arm64/kernel/head.s]
	__INIT

	/*
	 * The following callee saved general purpose registers are used on the
	 * primary lowlevel boot path:
	 *
	 *  Register   Scope                      Purpose
	 *  x21        primary_entry() .. start_kernel()        FDT pointer passed at boot in x0
	 *  x23        primary_entry() .. start_kernel()        physical misalignment/KASLR offset
	 *  x28        __create_page_tables()                   callee preserved temp register
	 *  x19/x20    __primary_switch()                       callee preserved temp registers
	 *  x24        __primary_switch() .. relocate_kernel()  current RELR displacement
	 */
SYM_CODE_START(primary_entry)
	bl	preserve_boot_args
	bl	el2_setup			// Drop to EL1, w0=cpu_boot_mode
	adrp	x23, __PHYS_OFFSET
	and	x23, x23, MIN_KIMG_ALIGN - 1	// KASLR offset, defaults to 0
	bl	set_cpu_boot_mode_flag
	bl	__create_page_tables
	/*
	 * The following calls CPU setup code, see arch/arm64/mm/proc.S for
	 * details.
	 * On return, the CPU will be ready for the MMU to be turned on and
	 * the TCR will have been set.
	 */
	bl	__cpu_setup			// initialise processor
	b	__primary_switch
SYM_CODE_END(primary_entry)
```

#### preserve_boot_args

保存从bootloader传递到kernel的x0~x3参数。

```assembly
[arch/arm64/kernel/head.s]
/*
 * Preserve the arguments passed by the bootloader in x0 .. x3
 */
SYM_CODE_START_LOCAL(preserve_boot_args)
	mov	x21, x0				// x21=FDT 将dtb的地址暂存在x21寄存器，释放出x0使用

	adr_l	x0, boot_args			// x0保存boot_args变量的地址
	stp	x21, x1, [x0]			// 将x0、x1的值保存到boot_args[0]、boot_args[1]
	stp	x2, x3, [x0, #16]		// 将x2、x3的值保存到boot_args[2]、boot_args[3]

	dmb	sy				// needed before dc ivac with
						// MMU off

	mov	x1, #0x20			// 4 x 8 bytes
	b	__inval_dcache_area		// tail call
SYM_CODE_END(preserve_boot_args)
```

#### el2_setup

程序执行至此，CPU处于哪一个exception level呢？根据ARM64 boot protocol，CPU要么处于EL2（推荐）或者non-secure EL1。如果在EL1，情形有些类似过去arm处理器的感觉，处于EL2稍微复杂一些，需要对virtualisation extensions进行基本的设定，然后将cpu退回到EL1，此处暂未分析。

```assembly
[arch/arm64/kernel/head.s]
/*
 * If we're fortunate enough to boot at EL2, ensure that the world is
 * sane before dropping to EL1.
 *
 * Returns either BOOT_CPU_MODE_EL1 or BOOT_CPU_MODE_EL2 in w0 if
 * booted in EL1 or EL2 respectively.
 */
SYM_FUNC_START(el2_setup)
	msr	SPsel, #1			// We want to use SP_EL{1,2}
	mrs	x0, CurrentEL
	cmp	x0, #CurrentEL_EL2
	b.eq	1f
	mov_q	x0, (SCTLR_EL1_RES1 | ENDIAN_SET_EL1)
	msr	sctlr_el1, x0
	mov	w0, #BOOT_CPU_MODE_EL1		// This cpu booted in EL1
	isb
	ret

1:	mov_q	x0, (SCTLR_EL2_RES1 | ENDIAN_SET_EL2)
	msr	sctlr_el2, x0

#ifdef CONFIG_ARM64_VHE
	/*
	 * Check for VHE being present. For the rest of the EL2 setup,
	 * x2 being non-zero indicates that we do have VHE, and that the
	 * kernel is intended to run at EL2.
	 */
	mrs	x2, id_aa64mmfr1_el1
	ubfx	x2, x2, #ID_AA64MMFR1_VHE_SHIFT, #4
#else
	mov	x2, xzr
#endif

	/* Hyp configuration. */
	mov_q	x0, HCR_HOST_NVHE_FLAGS
	cbz	x2, set_hcr
	mov_q	x0, HCR_HOST_VHE_FLAGS
set_hcr:
	msr	hcr_el2, x0
	isb

	/*
	 * Allow Non-secure EL1 and EL0 to access physical timer and counter.
	 * This is not necessary for VHE, since the host kernel runs in EL2,
	 * and EL0 accesses are configured in the later stage of boot process.
	 * Note that when HCR_EL2.E2H == 1, CNTHCTL_EL2 has the same bit layout
	 * as CNTKCTL_EL1, and CNTKCTL_EL1 accessing instructions are redefined
	 * to access CNTHCTL_EL2. This allows the kernel designed to run at EL1
	 * to transparently mess with the EL0 bits via CNTKCTL_EL1 access in
	 * EL2.
	 */
	cbnz	x2, 1f
	mrs	x0, cnthctl_el2
	orr	x0, x0, #3			// Enable EL1 physical timers
	msr	cnthctl_el2, x0
1:
	msr	cntvoff_el2, xzr		// Clear virtual offset

#ifdef CONFIG_ARM_GIC_V3
	/* GICv3 system register access */
	mrs	x0, id_aa64pfr0_el1
	ubfx	x0, x0, #ID_AA64PFR0_GIC_SHIFT, #4
	cbz	x0, 3f

	mrs_s	x0, SYS_ICC_SRE_EL2
	orr	x0, x0, #ICC_SRE_EL2_SRE	// Set ICC_SRE_EL2.SRE==1
	orr	x0, x0, #ICC_SRE_EL2_ENABLE	// Set ICC_SRE_EL2.Enable==1
	msr_s	SYS_ICC_SRE_EL2, x0
	isb					// Make sure SRE is now set
	mrs_s	x0, SYS_ICC_SRE_EL2		// Read SRE back,
	tbz	x0, #0, 3f			// and check that it sticks
	msr_s	SYS_ICH_HCR_EL2, xzr		// Reset ICC_HCR_EL2 to defaults

3:
#endif

	/* Populate ID registers. */
	mrs	x0, midr_el1
	mrs	x1, mpidr_el1
	msr	vpidr_el2, x0
	msr	vmpidr_el2, x1

#ifdef CONFIG_COMPAT
	msr	hstr_el2, xzr			// Disable CP15 traps to EL2
#endif

	/* EL2 debug */
	mrs	x1, id_aa64dfr0_el1
	sbfx	x0, x1, #ID_AA64DFR0_PMUVER_SHIFT, #4
	cmp	x0, #1
	b.lt	4f				// Skip if no PMU present
	mrs	x0, pmcr_el0			// Disable debug access traps
	ubfx	x0, x0, #11, #5			// to EL2 and allow access to
4:
	csel	x3, xzr, x0, lt			// all PMU counters from EL1

	/* Statistical profiling */
	ubfx	x0, x1, #ID_AA64DFR0_PMSVER_SHIFT, #4
	cbz	x0, 7f				// Skip if SPE not present
	cbnz	x2, 6f				// VHE?
	mrs_s	x4, SYS_PMBIDR_EL1		// If SPE available at EL2,
	and	x4, x4, #(1 << SYS_PMBIDR_EL1_P_SHIFT)
	cbnz	x4, 5f				// then permit sampling of physical
	mov	x4, #(1 << SYS_PMSCR_EL2_PCT_SHIFT | \
		      1 << SYS_PMSCR_EL2_PA_SHIFT)
	msr_s	SYS_PMSCR_EL2, x4		// addresses and physical counter
5:
	mov	x1, #(MDCR_EL2_E2PB_MASK << MDCR_EL2_E2PB_SHIFT)
	orr	x3, x3, x1			// If we don't have VHE, then
	b	7f				// use EL1&0 translation.
6:						// For VHE, use EL2 translation
	orr	x3, x3, #MDCR_EL2_TPMS		// and disable access from EL1
7:
	msr	mdcr_el2, x3			// Configure debug traps

	/* LORegions */
	mrs	x1, id_aa64mmfr1_el1
	ubfx	x0, x1, #ID_AA64MMFR1_LOR_SHIFT, 4
	cbz	x0, 1f
	msr_s	SYS_LORC_EL1, xzr
1:

	/* Stage-2 translation */
	msr	vttbr_el2, xzr

	cbz	x2, install_el2_stub

	mov	w0, #BOOT_CPU_MODE_EL2		// This CPU booted in EL2
	isb
	ret

SYM_INNER_LABEL(install_el2_stub, SYM_L_LOCAL)
	/*
	 * When VHE is not in use, early init of EL2 and EL1 needs to be
	 * done here.
	 * When VHE _is_ in use, EL1 will not be used in the host and
	 * requires no configuration, and all non-hyp-specific EL2 setup
	 * will be done via the _EL1 system register aliases in __cpu_setup.
	 */
	mov_q	x0, (SCTLR_EL1_RES1 | ENDIAN_SET_EL1)
	msr	sctlr_el1, x0

	/* Coprocessor traps. */
	mov	x0, #0x33ff
	msr	cptr_el2, x0			// Disable copro. traps to EL2

	/* SVE register access */
	mrs	x1, id_aa64pfr0_el1
	ubfx	x1, x1, #ID_AA64PFR0_SVE_SHIFT, #4
	cbz	x1, 7f

	bic	x0, x0, #CPTR_EL2_TZ		// Also disable SVE traps
	msr	cptr_el2, x0			// Disable copro. traps to EL2
	isb
	mov	x1, #ZCR_ELx_LEN_MASK		// SVE: Enable full vector
	msr_s	SYS_ZCR_EL2, x1			// length for EL1.

	/* Hypervisor stub */
7:	adr_l	x0, __hyp_stub_vectors
	msr	vbar_el2, x0

	/* spsr */
	mov	x0, #(PSR_F_BIT | PSR_I_BIT | PSR_A_BIT | PSR_D_BIT |\
		      PSR_MODE_EL1h)
	msr	spsr_el2, x0
	msr	elr_el2, lr
	mov	w0, #BOOT_CPU_MODE_EL2		// This CPU booted in EL2
	eret
SYM_FUNC_END(el2_setup)
```

#### set_cpu_boot_mode_flag

在进入这个函数的时候，有一个前提条件：w20寄存器保存了cpu启动时候的Eexception level ，具体代码如下：

```assembly
[arch/arm64/kernel/head.s]
/*
 * Sets the __boot_cpu_mode flag depending on the CPU boot mode passed
 * in w0. See arch/arm64/include/asm/virt.h for more info.
 */
SYM_FUNC_START_LOCAL(set_cpu_boot_mode_flag)
	adr_l	x1, __boot_cpu_mode
	cmp	w0, #BOOT_CPU_MODE_EL2
	b.ne	1f
	add	x1, x1, #4
1:	str	w0, [x1]			// This CPU has booted in EL1
	dmb	sy
	dc	ivac, x1			// Invalidate potentially stale cache line
	ret
SYM_FUNC_END(set_cpu_boot_mode_flag)
```

由于系统启动之后，需要了解CPU启动时候的Exception level，因此需要一个全局变量 \__boot_cpu_mode来保存启动时的CPU mode，__boot_cpu_mode定义入下：

```assembly
[arch/arm64/kernel/head.s]
/*
 * We need to find out the CPU boot mode long after boot, so we need to
 * store it in a writable variable.
 *
 * This is not in .bss, because we set it sufficiently early that the boot-time
 * zeroing of .bss would clobber it.
 */
SYM_DATA_START(__boot_cpu_mode)
	.long	BOOT_CPU_MODE_EL2
	.long	BOOT_CPU_MODE_EL1
SYM_DATA_END(__boot_cpu_mode)
```

#### __create_page_tables

建立页表初始化的过程；

```assembly
[arch/arm64/kernel/head.s]
/*
 * Setup the initial page tables. We only setup the barest amount which is
 * required to get the kernel running. The following sections are required:
 *   - identity mapping to enable the MMU (low address, TTBR0)
 *   - first few MB of the kernel linear mapping to jump to once the MMU has
 *     been enabled
 */
SYM_FUNC_START_LOCAL(__create_page_tables)
...
SYM_FUNC_END(__create_page_tables)
```

#### __cpu_setup

__cpu_setup函数位于`arch/arm64/mm/proc.s`路径，主要用于初始化CPU，内容包括：

* cache和TLB的处理；
* Memory、attributes lookup table的创建；
* SCTLR_EL1、TCR_EL1的设定；

```assembly
[arch/arm64/mm/proc.s]
/*
 *	__cpu_setup
 *
 *	Initialise the processor for turning the MMU on.
 *
 * Output:
 *	Return in x0 the value of the SCTLR_EL1 register.
 */
	.pushsection ".idmap.text", "awx"
SYM_FUNC_START(__cpu_setup)
	tlbi	vmalle1				// Invalidate local TLB
	dsb	nsh

	mov	x1, #3 << 20
	msr	cpacr_el1, x1			// Enable FP/ASIMD
	mov	x1, #1 << 12			// Reset mdscr_el1 and disable
	msr	mdscr_el1, x1			// access to the DCC from EL0
	isb					// Unmask debug exceptions now,
	enable_dbg				// since this is per-cpu
	reset_pmuserenr_el0 x1			// Disable PMU access from EL0
	reset_amuserenr_el0 x1			// Disable AMU access from EL0

	/*
	 * Memory region attributes
	 */
	mov_q	x5, MAIR_EL1_SET
#ifdef CONFIG_ARM64_MTE
	/*
	 * Update MAIR_EL1, GCR_EL1 and TFSR*_EL1 if MTE is supported
	 * (ID_AA64PFR1_EL1[11:8] > 1).
	 */
	mrs	x10, ID_AA64PFR1_EL1
	ubfx	x10, x10, #ID_AA64PFR1_MTE_SHIFT, #4
	cmp	x10, #ID_AA64PFR1_MTE
	b.lt	1f

	/* Normal Tagged memory type at the corresponding MAIR index */
	mov	x10, #MAIR_ATTR_NORMAL_TAGGED
	bfi	x5, x10, #(8 *  MT_NORMAL_TAGGED), #8

	/* initialize GCR_EL1: all non-zero tags excluded by default */
	mov	x10, #(SYS_GCR_EL1_RRND | SYS_GCR_EL1_EXCL_MASK)
	msr_s	SYS_GCR_EL1, x10

	/* clear any pending tag check faults in TFSR*_EL1 */
	msr_s	SYS_TFSR_EL1, xzr
	msr_s	SYS_TFSRE0_EL1, xzr
1:
#endif
	msr	mair_el1, x5
	/*
	 * Set/prepare TCR and TTBR. We use 512GB (39-bit) address range for
	 * both user and kernel.
	 */
	mov_q	x10, TCR_TxSZ(VA_BITS) | TCR_CACHE_FLAGS | TCR_SMP_FLAGS | \
			TCR_TG_FLAGS | TCR_KASLR_FLAGS | TCR_ASID16 | \
			TCR_TBI0 | TCR_A1 | TCR_KASAN_FLAGS
	tcr_clear_errata_bits x10, x9, x5

#ifdef CONFIG_ARM64_VA_BITS_52
	ldr_l		x9, vabits_actual
	sub		x9, xzr, x9
	add		x9, x9, #64
	tcr_set_t1sz	x10, x9
#else
	ldr_l		x9, idmap_t0sz
#endif
	tcr_set_t0sz	x10, x9

	/*
	 * Set the IPS bits in TCR_EL1.
	 */
	tcr_compute_pa_size x10, #TCR_IPS_SHIFT, x5, x6
#ifdef CONFIG_ARM64_HW_AFDBM
	/*
	 * Enable hardware update of the Access Flags bit.
	 * Hardware dirty bit management is enabled later,
	 * via capabilities.
	 */
	mrs	x9, ID_AA64MMFR1_EL1
	and	x9, x9, #0xf
	cbz	x9, 1f
	orr	x10, x10, #TCR_HA		// hardware Access flag update
1:
#endif	/* CONFIG_ARM64_HW_AFDBM */
	msr	tcr_el1, x10
	/*
	 * Prepare SCTLR
	 */
	mov_q	x0, SCTLR_EL1_SET
	ret					// return to head.S
SYM_FUNC_END(__cpu_setup)
```

> [慢慢欣赏arm64内核启动19 primary_entry之__cpu_setup代码第一部分](https://blog.csdn.net/shipinsky/article/details/141809684)
>
> [__cpu_setup注释](https://blog.csdn.net/haotianmai/article/details/129890593?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-8-129890593-blog-142514018.235^v43^pc_blog_bottom_relevance_base7&spm=1001.2101.3001.4242.5&utm_relevant_index=9)
>
> [ARM64启动汇编和内存初始化(中) --- (二)](https://www.cnblogs.com/jianhua1992/p/16842866.html)

#### __primary_switch

主要工作是为打开MMU做准备。

```assembly
[arch/arm64/kernel/head.s]
SYM_FUNC_START_LOCAL(__primary_switch)
#ifdef CONFIG_RANDOMIZE_BASE
	mov	x19, x0				// preserve new SCTLR_EL1 value
	mrs	x20, sctlr_el1			// preserve old SCTLR_EL1 value
#endif

	adrp	x1, init_pg_dir
	bl	__enable_mmu
#ifdef CONFIG_RELOCATABLE
#ifdef CONFIG_RELR
	mov	x24, #0				// no RELR displacement yet
#endif
	bl	__relocate_kernel
#ifdef CONFIG_RANDOMIZE_BASE
	ldr	x8, =__primary_switched
	adrp	x0, __PHYS_OFFSET
	blr	x8

	/*
	 * If we return here, we have a KASLR displacement in x23 which we need
	 * to take into account by discarding the current kernel mapping and
	 * creating a new one.
	 */
	pre_disable_mmu_workaround
	msr	sctlr_el1, x20			// disable the MMU
	isb
	bl	__create_page_tables		// recreate kernel mapping

	tlbi	vmalle1				// Remove any stale TLB entries
	dsb	nsh

	msr	sctlr_el1, x19			// re-enable the MMU
	isb
	ic	iallu				// flush instructions fetched
	dsb	nsh				// via old mapping
	isb

	bl	__relocate_kernel
#endif
#endif
	ldr	x8, =__primary_switched
	adrp	x0, __PHYS_OFFSET
	br	x8
SYM_FUNC_END(__primary_switch)
```

在函数中通过 \__enable_mmu 函数来开启MMU，并调用__primary_switched函数；

##### __primary_switched

在__primary_switched函数中，进行一些C环境的准备，并在最后，调用执行start_kernel函数，内核的启动进入到C语言环境阶段；

```assembly
[arch/arm64/kernel/head.s]
/*
 * The following fragment of code is executed with the MMU enabled.
 *
 *   x0 = __PHYS_OFFSET
 */
SYM_FUNC_START_LOCAL(__primary_switched)
	adrp	x4, init_thread_union
	add	sp, x4, #THREAD_SIZE
	adr_l	x5, init_task
	msr	sp_el0, x5			// Save thread_info

#ifdef CONFIG_ARM64_PTR_AUTH
	__ptrauth_keys_init_cpu	x5, x6, x7, x8
#endif

	adr_l	x8, vectors			// load VBAR_EL1 with virtual
	msr	vbar_el1, x8			// vector table address
	isb

	stp	xzr, x30, [sp, #-16]!
	mov	x29, sp

#ifdef CONFIG_SHADOW_CALL_STACK
	adr_l	scs_sp, init_shadow_call_stack	// Set shadow call stack
#endif

	str_l	x21, __fdt_pointer, x5		// Save FDT pointer

	ldr_l	x4, kimage_vaddr		// Save the offset between
	sub	x4, x4, x0			// the kernel virtual and
	str_l	x4, kimage_voffset, x5		// physical mappings

	// Clear BSS
	adr_l	x0, __bss_start
	mov	x1, xzr
	adr_l	x2, __bss_stop
	sub	x2, x2, x0
	bl	__pi_memset
	dsb	ishst				// Make zero page visible to PTW

#ifdef CONFIG_KASAN
	bl	kasan_early_init
#endif
#ifdef CONFIG_RANDOMIZE_BASE
	tst	x23, ~(MIN_KIMG_ALIGN - 1)	// already running randomized?
	b.ne	0f
	mov	x0, x21				// pass FDT address in x0
	bl	kaslr_early_init		// parse FDT for KASLR options
	cbz	x0, 0f				// KASLR disabled? just proceed
	orr	x23, x23, x0			// record KASLR offset
	ldp	x29, x30, [sp], #16		// we must enable KASLR, return
	ret					// to __primary_switch()
0:
#endif
	add	sp, sp, #16
	mov	x29, #0
	mov	x30, #0
	b	start_kernel
SYM_FUNC_END(__primary_switched)
```

## kernel启动第二阶段







# 参考

[Linux内核启动流程-基于ARM64](https://mshrimp.github.io/2020/04/19/Linux%E5%86%85%E6%A0%B8%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B-%E5%9F%BA%E4%BA%8EARM64/)

[ARM64 linux 启动](https://www.cnblogs.com/memoryart/p/arm64_linux_boot.html)

[ARM64的启动过程之（一）：内核第一个脚印](http://www.wowotech.net/armv8a_arch/arm64_initialize_1.html)
