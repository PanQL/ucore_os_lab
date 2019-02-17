## Lab4 Report  

### 练习1  

---

alloc_proc函数的具体实现：

```c
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
    //LAB4:EXERCISE1 YOUR CODE
    /*
     * below fields in proc_struct need to be initialized
     *       enum proc_state state;                      // Process state
     *       int pid;                                    // Process ID
     *       int runs;                                   // the running times of Proces
     *       uintptr_t kstack;                           // Process kernel stack
     *       volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
     *       struct proc_struct *parent;                 // the parent process
     *       struct mm_struct *mm;                       // Process's memory management field
     *       struct context context;                     // Switch here to run process
     *       struct trapframe *tf;                       // Trap frame for current interrupt
     *       uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
     *       uint32_t flags;                             // Process flag
     *       char name[PROC_NAME_LEN + 1];               // Process name
     */
		proc->state = PROC_UNINIT;
		proc->pid = -1;
		proc->runs = 0;
		proc->kstack = NULL;
		proc->need_resched = 0;
		proc->parent = NULL;
		proc->mm = 0;
		memset(&(proc->context), 0, sizeof(struct context));
		proc->tf = 0;
		proc->cr3 = boot_cr3;
		proc->flags = 0;
		memset(proc->name, 0, sizeof(char) * PROC_NAME_LEN );
    }
    return proc;
}
```

根据实验文档的介绍可知，alloc_proc函数的功能主要是为一个新的process分配内存空间，并进行必要的初始化。

- 根据proc_state定义处的相关注释，可以知道在alloc_proc中初始化的proc_struct的state都设置为uninit。
- 实验文档中指出，pid默认初始化值为-1,通过阅读后续源码发现，进行alloc_proc之后，还会重新设置pid，所以此处设置为-1即可。
- runs不知道是干什么的。
- kstack同样会在后续的代码中通过setup_stack()设置，此处给出简单的初始化即可。
- need_resched代表是否需要被调度，同样，在后续代码中会进行设置。同理，mm后续代码中会通过mm_copy()设置，
  - 如果alloc的是内核线程，则使用内核的内存空间，mm不需要设置，此时使用cr3来作为页目录表的基地址，
  - 否则如果alloc的是用户进程，则调用alloc_proc()之后可以通过mm_copy()进行设置。
- tf也会在后续代码中设置。
- 关于cr3,初始化为内核的页目录表基地址即可。
- 需要对context和name成员进行清零处理。

关于struct context context 和 struct trapframe *tf的含义与作用：

* context代表进程/线程相关的寄存器的值，trapframe代表发生中断时栈帧的信息。
* 作用：在本次lab中，用于在do_fork--->copy_thread的调用关系中复制当前进程的上下文环境给新的进程控制块。

### 练习2  

---

最终实现的do_fork函数如下

```c
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;
    //LAB4:EXERCISE2 YOUR CODE
    /*
     * Some Useful MACROs, Functions and DEFINEs, you can use them in below implementation.
     * MACROs or Functions:
     *   alloc_proc:   create a proc struct and init fields (lab4:exercise1)
     *   setup_kstack: alloc pages with size KSTACKPAGE as process kernel stack
     *   copy_mm:      process "proc" duplicate OR share process "current"'s mm according clone_flags
     *                 if clone_flags & CLONE_VM, then "share" ; else "duplicate"
     *   copy_thread:  setup the trapframe on the  process's kernel stack top and
     *                 setup the kernel entry point and stack of process
     *   hash_proc:    add proc into proc hash_list
     *   get_pid:      alloc a unique pid for process
     *   wakeup_proc:  set proc->state = PROC_RUNNABLE
     * VARIABLES:
     *   proc_list:    the process set's list
     *   nr_process:   the number of process set
     */

    //    1. call alloc_proc to allocate a proc_struct
    //    2. call setup_kstack to allocate a kernel stack for child process
    //    3. call copy_mm to dup OR share mm according clone_flag
    //    4. call copy_thread to setup tf & context in proc_struct
    //    5. insert proc_struct into hash_list && proc_list
    //    6. call wakeup_proc to make the new child process RUNNABLE
    //    7. set ret vaule using child proc's pid
	
	proc = alloc_proc();
	proc->parent = current;
	if (setup_kstack(proc) != 0) {
        goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
	
	copy_thread(proc, stack, tf);
	//insert proc_struct into hash_list && proc_list
	bool intr_flag;
    local_intr_save(intr_flag);
    {
		proc->pid = get_pid();
		list_add(&proc_list, &(proc->list_link));
		hash_proc(proc);
		nr_process ++;
	}
	local_intr_restore(intr_flag);
	wakeup_proc(proc);
	ret = proc->pid;
fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

根据上述代码不难得知，分配一个线程之后，还需要调用get_pid()来获取一个线程号，而get_pid()可以保证返回的线程号是当前没有在使用的。所以能够保证每个进程/线程的id是独一无二的。

### 练习3  

---

* 在本次实验中，创建并执行了两个线程。
* local_intr_save(intr_flag);....local_intr_restore(intr_flag);语句的作用为：
  * 在特定的语句块内禁止中断，保证当前操作为原子操作。
  * 理由：loacal_intr_save(),local_intr_restore()的底层实现是用内联汇编实现的cli、sti语句。正是用来屏蔽/使能中断的。