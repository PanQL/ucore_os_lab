## lab1实验报告  

### 练习1  

1. ucore.img生成过程：

   生成ucore.img需要生成/bin/bootblock、/bin/kernel

   （1）生成kernel需要生成：

   ​		obj/kern/init/init.o 

   ​		obj/kern/libs/stdio.o 

   ​		obj/kern/libs/readline.o 

   ​		obj/kern/debug/panic.o 

   ​		obj/kern/debug/kdebug.o 

   ​		obj/kern/debug/kmonitor.o 

   ​		obj/kern/driver/clock.o 

   ​		obj/kern/driver/console.o 

   ​		obj/kern/driver/picirq.o 

   ​		obj/kern/driver/intr.o 

   ​		obj/kern/trap/trap.o 

   ​		obj/kern/trap/vectors.o 

   ​		obj/kern/trap/trapentry.o 

   ​		obj/kern/mm/pmm.o  

   ​		obj/libs/string.o 

   ​		obj/libs/printfmt.o

   （2）生成bootblock需要生成：

   ​		obj/bootblock.o

   ​		bin/sign

   ​	并执行bin/sign

   编译.o文件的选项含义为(以init.c的编译为例)：

   ```c
   + cc kern/init/init.c
   gcc     -Ikern/init/ 把kern/init 添加到头文件的目录集合中
   		-fno-builtin    不识别没有以__builtin_开头的内置函数
           -Wall  开启所有编译警告 
           -ggdb 产生用于gdb的调试信息
           -m32 编译32位程序
           -gstabs 生成stabs类型的调试信息
           -nostdinc  禁止在系统目录中寻找头文件，即不使用系统提供的库
           -fno-stack-protector 
           -Ilibs/  把libs/添加到搜索头文件的目录集合中
           -Ikern/debug/  把kern/debug 添加到搜索头文件的目录集合中
           -Ikern/driver/  把kern/driver 添加到搜索头文件的目录集合中
           -Ikern/trap/  把kern/trap 添加到搜索头文件的目录集合中
           -Ikern/mm/  把kern/mm 添加到搜索头文件的目录集合中
           -c kern/init/init.c    编译 
           -o obj/kern/init/init.o     输出写入obj/kern/init/init.o
   ```

2. 一个合法的主引导扇区需要的条件：

   最后一个字节为0x55

   倒数第二个字节为0xAA  

### 练习2  

使用qemu执行并调试lab1中的软件。

1、从CPU加电之后执行的第一条指令开始，单步跟踪BIOS的执行。

根据ucore_os_docs中lab1的附录B，修改tools/gdbinit

```shell
set architecture i8086
target remote :1234
```

然后在lab1目录下执行make debug，gdb将停在BIOS第一条指令处，通过

```shell
si
si
……
```

可以单步执行。

2、在初始化位置0x7c00设置实地址断点，测试断点正常。

在tools/gdbinit中追加如下代码

```shell
break *0x7c00
c
```

重新执行make debug可以发现gdb在0x7c00地址前暂停。

3、从0x7c00开始跟踪代码运行，将单步反汇编得到的代码与bootasm.S和bootblock.asm进行比较。

起初采用gdb单步调试并在命令行中打印出汇编代码的方法，可以得到大致对照，但是效率较低。

参考lab1答案之后，发现可以用-d in_asm -D q.log选项得到qemu代码的反汇编结果。

修改了Makefile中的debug部分：

```makefile
debug: $(UCOREIMG)
	$(V)$(QEMU) -d in_asm -D q.log -S -s -parallel stdio -hda $< -serial null &
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```

修改tools/gdbinit，追加

```shell
b *0x7c00
c
```

执行make debug之后通过gdb单步执行，可以看到0x7c00-0x7c4a部分，得到的汇编代码与bootblock.asm中的汇编代码几乎都可以对应起来。

![](reportPic/lab1_2_3-meld.png)

4、自己找一个bootloader或内核中的位置，设置断点并进行测试。

设置断点的位置

![](reportPic/position1_2_4.png)

![](reportPic/lab1_2_4.png)

### 练习3  

首先禁止中断、初始化重要的数据段寄存器:

```assembly
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```

1、如何开启A20？

```assembly
seta20.1:
    # 将0x64处的一个字节读入到%al，判断0x2和al与的结果
    # 等待8042输入缓冲区空闲
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    # 8042键盘缓冲端口输出0xd1
    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    # 将0x64处的一个字节读入到%al，判断0x2和al与的结果
    # 等待8042输入缓冲区空闲
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    # 8042键盘缓冲端口输出0xdf
    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1

    # Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses
    # identical to physical addresses, so that the
    # effective memory map does not change during the switch.
    lgdt gdtdesc
    movl %cr0, %eax     # 将%cr0最低位置为1
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg		#长跳转的同时设置CS寄存器的值
```

为何开启A20？

​	为保证后向兼容，A20线默认是低电平，在这种情况下可寻址空间只有1M。开启A20是为了能够使用超过1M的高端内存区域。

2、如何初始化GDT表?

​      内存中已经有一个简易的GDT表，直接加载即可。

```assembly
    lgdt gdtdesc
```

将cr0寄存器的PE位设置为1,即切换到保护模式：

```assembly
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
```

转到保护模式后，设置段寄存器、初始化栈指针：

```assembly
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
```

转换完成，进入bootmain:

```assembly
    call bootmain
```

### 练习4  

1、bootloader如何读取硬盘扇区？

```c
// boot/bootmain.c
// 读取单个扇区
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

```c
//简单包装了readsect函数，一次读取任意个数的扇区到指定位置。
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

bootloader通过内联汇编生成读取扇区的代码，将扇区读取到指定位置。

2、bootloader如何加载ELF格式的OS？

​	根据ELF格式的文件指针，获取OS的大小，然后，调用readsect函数逐个扇区加载OS。

```c
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    // 载入ELF格式的kernel
    //根据ELF格式文件的指针，获取文件的大小，再调用readseg函数读取所有的扇区，完成加载。
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

###  练习5  

```c
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
    uint32_t ebp = read_ebp();
    uint32_t eip = read_eip();
    uint32_t a,b,c,d;
    int i = 0;
    while(i != STACKFRAME_DEPTH && ebp != 0){
        a = *(uint32_t *)(ebp + 8);
        b = *(uint32_t *)(ebp + 12);
        c = *(uint32_t *)(ebp + 16);
        d = *(uint32_t *)(ebp + 20);
        cprintf("ebp:0x%08x,eip:0x%08x args:0x%08x 0x%08x 0x%08x 0x%08x", 
            ebp, eip, a, b, c, d);
        cprintf("\n");
        print_debuginfo((uint32_t)eip - 1);
        eip = *(uint32_t *)(ebp + 4);
        ebp = *(uint32_t *)(ebp);
    }
}
```

根据提示，当前函数的ebp指向的地址存储的是调用者的ebp，当前函数的ebp+4指向的地址是当前函数的返回地址eip。再之后的ebp+8、ebp+12....等存储的是传入的参数，此处只打印出4个。并且根据约定，进入内核之前ebp被设置为0。

### 练习6  

1、中断描述符表的表项大小为8个字节，其中，第二个字节为段选择子，第一个和最后一个字节拼接起来之后为段内位移。

2、对应的函数实现：

```c
/* idt_init - initialize IDT to each of the entry points in kern/trap/vectors.S */
void
idt_init(void) {
     /* LAB1 YOUR CODE : STEP 2 */
     /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
      *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
      *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
      *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
      *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
      * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
      *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
      * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
      *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
      *     Notice: the argument of lidt is idt_pd. try to find it!
      */
     extern uintptr_t __vectors[];
     int i;
     for(i = 0; i < 256; i++){
         SETGATE(idt[i], 0, GD_KTEXT ,__vectors[i], DPL_KERNEL);
     }
     lidt(&idt_pd);
}
```

```c
/* trap_dispatch - dispatch based on what type of trap occurred */
static void
trap_dispatch(struct trapframe *tf) {
    char c;

    switch (tf->tf_trapno) {
    case IRQ_OFFSET + IRQ_TIMER:
        /* LAB1 YOUR CODE : STEP 3 */
        /* handle the timer interrupt */
        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
         * (3) Too Simple? Yes, I think so!
         */
        ticks++;
        if(ticks % TICK_NUM == 0){
            print_ticks();
        }
        break;
    case IRQ_OFFSET + IRQ_COM1:
        c = cons_getc();
        cprintf("serial [%03d] %c\n", c, c);
        break;
    case IRQ_OFFSET + IRQ_KBD:
        c = cons_getc();
        cprintf("kbd [%03d] %c\n", c, c);
        break;
    //LAB1 CHALLENGE 1 : YOUR CODE you should modify below codes.
    case T_SWITCH_TOU:
    case T_SWITCH_TOK:
        panic("T_SWITCH_** ??\n");
        break;
    case IRQ_OFFSET + IRQ_IDE1:
    case IRQ_OFFSET + IRQ_IDE2:
        /* do nothing */
        break;
    default:
        // in kernel, it must be a mistake
        if ((tf->tf_cs & 3) == 0) {
            print_trapframe(tf);
            panic("unexpected trap in kernel.\n");
        }
    }
}
```

### Challenge部分：

#### Challenge1  

​	根据ucore的中断机制，产生中断时，临时栈中存储了对应的寄存器值与trapframe的定义是一致的：

```c
struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```

按照慕课的提示，需要修改的栈中内容为：

​	es，ds，cs，eflags，esp，ss（从ring0到ring3时需要修改）

其中，ErrorCode、eip、cs、eflags是由硬件CPU负责压入栈的，gs、fs、es、ds、trapno则是由异常处理程序压入栈的。而esp、ss只有在从ring0到ring3的转换中会在iret指令中被pop出栈。所以近需要在内核态切换到用户态时考虑这两项。

ring0到ring3：

```c
//进入该中断之前，手动下拉栈顶0x8字节，为esp、ss分配内存。
case T_SWITCH_TOU:
	if(tf->tf_cs != USER_CS){
		tf->tf_cs = USER_CS;
		tf->tf_ds = USER_DS;
		tf->tf_es = USER_DS;
		tf->tf_ss = USER_DS;
		tf->tf_eflags |= FL_IOPL_MASK;	//设置IOPL，使特权极切换之后正常使用IO指令
		tf->tf_esp = (uint32_t)tf + sizeof(struct trapframe) - 8;//设置在iret时要弹出的栈顶
	}
break;
```

ring3到ring0：

```c
case T_SWITCH_TOK:
	if(tf->tf_cs != KERNEL_CS){		//首先判断是否存在转换
		tf->tf_cs = KERNEL_CS;
		tf->tf_ds = KERNEL_DS;
		tf->tf_es = KERNEL_DS;
		tf->tf_eflags &= ~FL_IOPL_MASK;		//设置IOPL，使特权极切换之后正常使用IO指令
	}
break;
```

#### Challenge2:  

​	可以借用Challenge1中的实现，在处理键盘中断的时候进行特权级的切换。

```c
case IRQ_OFFSET + IRQ_KBD:
    c = cons_getc();
    cprintf("kbd [%03d] %c\n", c, c);
    if(c == 0 && tf->tf_cs == USER_CS){
		tf->tf_cs = KERNEL_CS;
		tf->tf_ds = KERNEL_DS;
		tf->tf_es = KERNEL_DS;
		tf->tf_eflags &= ~FL_IOPL_MASK;
    }
    if(c == 3 && tf->tf_cs == KERNEL_CS){
		tf->tf_cs = USER_CS;
		tf->tf_ds = USER_DS;
		tf->tf_es = USER_DS;
		tf->tf_ss = USER_DS;
		tf->tf_eflags |= FL_IOPL_MASK;
		tf->tf_esp = (uint32_t)tf + sizeof(struct trapframe) - 8;
    }
    break;
```





