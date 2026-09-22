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

Kernel stores * process descriptor * in circularly linked list called the `task list`, each of the entries in the circularly linked list is of type `task_struct` large data structure contains all informatiion about the task

 * __bold__ NOTE: There is no Explicit `task_list` struct in the kernel this DOES NOT EXIST!!!  - In the kernel in `linux/sched.h` there is no explicit `task_struct` instead each of the struct `task_struct` is EMBEDDED with a `list_head` field, that points to the previous and next `tas_struct` in the list, this data structure is completely implicit but we still refer to it as the `task_struct` Note that when we talk about a "process descriptor" we are refering to a `task_struct` this is what this is.

In the `task_struct` the name of the `list_head` field is the `tasks` field.

** NOTE - kernel stack:**  The kernel itself is physically contiguous in one area of memory, but each process gets some virtual address space that is split into two parts the the userspace (lower part) and the kernel space (upper) the kernel space is really a collection of pointers to the physically contiguous kernel memory, this includes a pointer to the processes' own `Kernel Stack` the `Kernel Stack` is a stack datastrcuture that contains information that is specific to this process. This includes the `struct thread_info ` data structure that contains information about the process and a pointer to the `process descriptor - task_struct` of the process. NOTE that the `thread_info` no longer lives in the lower part of the `kernel stack` it lives directly inside of the process' `task_struct` this is how this works today.


*NOTE - IMPORTANT THE KERNEL DOES NOT RUN AS A PROCESS* it runs as a code block that has different contexts DON'T THINK OF IT AS A PROCESS!!

### Process ID - PID 

The PID is used to identify a process this is represented on the system by the type `pid_t` this usually implemented as a `int` and the range of the values is [0,32768].

Note that when we have more than this we can have *overflow* this is bad, we can no longer rely on the pid_t value to tell us which process was started before or after which new processes will consume low memory, therefore will not work for this.


### Process state

* `state` field of a `task_struct` represents state of process, *NOTE this does not mean that this is exact state of process, we have to consider when `schedule()` gets called.* 
* can change the `state` field using the `set_task_state(task,state)`, note that the `state` 
  this is equivalent to `task->state = state;` included in `<linux/sched.h>`

### Process context

* code read from `executable file` and it is executed in process' address space execution occurs in `user space`.
* program executes a `syscall` or fires `interrupt` code executes `kerne-space` *at this point the kernel is said to be executing on behalf of the process and is in `process context` *
* when in the `process context` the `current` macro is valid recall that `current` gets you the current `thread_info`.


### Process Family Tree 

All process have PID, recall that we crate process by calling fork.

* *init process: * the first process on the system responsible for finishing boot process running the initscripts, loading the remaining parts of the OS.
* each process has exactly one parent process that parent PID is often denoted * PPID *
* 



