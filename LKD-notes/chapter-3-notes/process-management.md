# Chapter 3 - Process Management 

> Process fundamental abstration of Linux and Unix kernels.

*NOTE:* A process contains more than just the executing code of arbitrary programs, includes set of resources; open files, pending signals, internal kernel data, processor state, memory address space (virtual memory) and a data section containing global variables (heap?) 


> Living result of running program code, kernel needs to manage completely.

*Thread of Execution* - objects of activity within the process, each thread includes a unique program counter (PC), process stack and set of processor registers.

Kernel schedules threads not processes, BUT to the kernel unique implementation of threads DOES NOT DIFFERENTIATE BETWEEN THREADS AND PROCESSES a THREAD IS JUST A SPECIAL KIND OF PROCESS.

Processes provide TWO Virtualizations 1) Processor virtualization 2) Memory virtualization, 1) gives the illusion that the the process alone monopolizes the entire system despite sharing the processor with hundereds of other processes on the same machine.

*NOTE:* Threads SHARE the virtual memory abstraction, whereas each receives it's own vitualized processor. WHAT DOES THIS EVEN MEAN? 
  -> Threads in the same process will share one set of Page Tables therefore same virtual memory. But they each get their own virtualized processor.

Process begins lifecycle with the `fork()` syscall note that this syscall DOES TAKE ANY ARGUMENTS at all! Uses the parent context to simply spawn a new process.

The `parent` is the caller of `fork()` and the `child` is the one that gets called because of the `fork()` syscall. Note that the `fork` syscall returns TWICE once in the parent and again in the child how does this work? Note that the `fork()` syscall is itself implemented by the `clone()` syscall

Fork is useful but we want to start processes with there own executable code instead of the parents to do this we have the `exec` syscall, creates a new address space loads a new program into it


## Process Descriptor and the Task struct 

Kernel stores process descriptor in `circularly linked list` called the `task list`, each of the entries in the circularly linked list is of type `task_struct` large data structure contains all informatiion about the task

 * **NOTE: There is no Explicit `task_list` struct in the kernel this DOES NOT EXIST!!!** - In the kernel in `linux/sched.h` there is no explicit `task_struct` instead each of the struct `task_struct` is EMBEDDED with a `list_head` field, that points to the previous and next `tas_struct` in the list, this data structure is completely implicit but we still refer to it as the `task_struct` Note that when we talk about a "process descriptor" we are refering to a `task_struct` this is what this is.

In the `task_struct` the name of the `list_head` field is the `tasks` field.

* **NOTE - kernel stack:**  The kernel itself is physically contiguous in one area of memory, but each process gets some virtual address space that is split into two parts the the userspace (lower part) and the kernel space (upper) the kernel space is really a collection of pointers to the physically contiguous kernel memory, this includes a pointer to the processes' own `Kernel Stack` the `Kernel Stack` is a stack datastrcuture that contains information that is specific to this process. This includes the `struct thread_info ` data structure that contains information about the process and a pointer to the `process descriptor - task_struct` of the process. NOTE that the `thread_info` no longer lives in the lower part of the `kernel stack` it lives directly inside of the process' `task_struct` this is how this works today.


* *NOTE - IMPORTANT THE KERNEL DOES NOT RUN AS A PROCESS* it runs as a code block that has different contexts DON'T THINK OF IT AS A PROCESS!!

### Process ID - PID 

* The PID is used to identify a process this is represented on the system by the type `pid_t` this usually implemented as a `int` and the range of the values is [0,32768].

* Note that when we have more than this we can have *overflow* this is bad, we can no longer rely on the pid_t value to tell us which process was started before or after which new processes will consume low memory, therefore will not work for this.


### Process state

* `state` field of a `task_struct` represents state of process, *NOTE this does not mean that this is exact state of process, we have to consider when `schedule()` gets called.* 
* can change the `state` field using the `set_task_state(task,state)`, note that the `state` 
  this is equivalent to `task->state = state;` included in `<linux/sched.h>`

### Process context

* code read from `executable file` and it is executed in process' address space execution occurs in `user space`.
* program executes a `syscall` or fires `interrupt` code executes `kerne-space` *at this point the kernel is said to be executing on behalf of the process and is in `process context` *
* when in the `process context` the `current` macro is valid recall that `current` gets you the current `thread_info`.

But what does this actually mean?
1. When the process executes a syscall this is what happens `libc` runs the syscall
2. CPU hardware switches to ring 0, and jumps to a kernel entry point (an address that the kernel registered at machine boot)
3. execution continues - the exact same core, same process, same thread same task_struct **but now it is running kernel code** with kernel priviliges on a kernel stack
4. Kernel does work, executtes `sysret`/`iret` CPU drops back to ring 3 and program continues at the next instruction.

**NOTE: Kernel Stacks** each process gets it's own kernel stack, the kernel stack is analogous to the userspace stack, except that it is used purely for the kernels calls this is all that it is used for in general.



### Process Family Tree 

All process have PID, recall that we crate process by calling fork.

* *init process:* the first process on the system responsible for finishing boot process running the initscripts, loading the remaining parts of the OS.
* each process has exactly one parent process that parent PID is often denoted * PPID *
* Note that from any process in the system you can find the PID of any other process on the system because they form a circular doubly linked-list.


### Process Creation 

* Linux uses the `fork()` syscall to create a new process and uses the `exec()` syscall loads a new executable into the address space of the new process
* most operating systems DO THIS IN ONE STEP

### Copy-on-Write - COW 

* originally fork() all resources of parents duplicated and given to the child process, this method *IS NOT EFFICIENT* and is *NAIVE* need better solution, copies much data that otherwise can be shared between the two processes, worse new process loads new executable no purpose of the copying in the first place.

* Copy-on-Write when `fork()` occurs we copy the pages between the parent and the child processes' when either the parent or the child writes to one of the pages, that page is copied and each process receives a new unique copy in there address space. Note that all anonymous pages can be COW.

* so only overhead of the `fork()` syscall is to duplicate the Page Tables (not the actual pages!) and the creation of the new process descriptor, important optimization this copy can easily be tens of hundreds of MB



### Forking 

* linux implements the `clone()` syscall this is what we are actually calling whenever the `fork()` glibc library function gets used. **NOTE: `fork()` DOES NOT USE THE SYSCALL OF THE SAME NAME IT USES CLONE() WEIRD...**

* the `fork`, `vfork`,`__clone` glibc library functions all use the `clone()` syscall under the hood.

* the `clone()` syscall, uses the kernel function `do_fork()` ( this is stupidly named and confusing ... fork clone fork lol), bulk of work handled by the `do_fork()`

* so the chain: `fork() (glibc - userspace)` -> `clone() - (syscall kernel)` -> `do_fork() (function, kernel) ` -> `dup_task_struct (function, kernel)` -> `copy_process() (function,kernel)`
**NOTE that the modern kernel replaced the `do_fork()` with the `kernel_clone()` function.**

* `dup_task_struct` does this 

1. `parent` calls `fork()`,follows chain to `dup_task_struct`  creates a new `task_struct`, `thread_info` and `kernel_stack` that are completely identical to the parent calling process, 

2. the `child` process needs to differentiate from parent so it resets many fields in `task_struct` to defau

3. `child` state is changed to `TASK_UNINTERRUPTIBLE` to ensure will not start executing instructions.

4. `copy_process()` calls `copy_flags()` and the flags of the `child` are updated accordingly, some important flags checks if `PF_SUPERPRIV` if the task was started with super user privileges, and `PF_FORKNOEXEC` for a process that was started without a later call to `exec` syscall, eg. it is executing the parents code.

5. the `clone()` flags passed to it determine if in `copy_procses()` the parent and child will share or duplucate open files, filesystem information, signal handlers, process address space and namespace, these shared usually betweeen *threads* in process but not between processes


### vfork() syscall 

* Same effect as `fork()` excet the PTEs of parent ARE NOT COPIED. Child executes the same instructions as parent it is sole thread in the parent's address space and parent is blocked until child calls `exec()` or exits.

* Child not allowed to write to the address space of the parent 

**NOTE this is a function that is widely deprecated because of the reasons mentioned in the book**


### Linux Thread implementation 

In other operating systems there is a type distinction between threads and processes, for example in Microsoft and Sun Solaris we would have that there is and *explicit* type for threads.

In these OS (not linux) a *thread* is just a *lightweight process* that shares the resources of the process that it belongs to with other *threads* in that process.

Linux does not make this distinction, the **ONLY** distinction between a *thread* and a *process* is that threads will share resources that is it, this is the only difference.

So supppose we want to create a process that has 4 working threads in Windows or Solaris, here we would have to create 1 process than spawn 4 threads under that process this is not the case with linux, we would just spawn 4 `task_structs` that share resources this would create 4 threads effectively.

**NOTE: What would be the `process` in this example** - It would be the 4 threads + the resources they jointly point at this is all that is meant by a process in the context of linux, this is oddly very beautiful and simple.

### Creating Threads 

Threads created like other tasks we just set flags so that they shared common resources, the defining resource that gets shared is the `virtual memory` this is done by setting the `CLONE_VM` during the process creation, shares the `mm_struct` between the threads.






