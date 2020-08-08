## kernel process manager


### relative files

| file | brief description |
|:---|:---|
| proc.h | definition of process table - `struct proc`|
| proc.c | implemention of process and message handling |
| minix/ipc.h | types relateing message, here define `message` |


### important types 

| type | brief description |
|:---|:---|
| `endpoint_t`  | process number |
| `struct proc` | process table slot |
| `message`     | data send in ipc |


### process table slots

All process has a process table, which is pre-allocated in compile time.
`NR_TASKS` represents number of kernel tasks include `ASYNCM, IDLE, CLOCK, SYSTEM, KERNEL`,
`NR_PROCS` is number of system process and user process TODO.
``` c
/** proc.h */
EXTERN struct proc proc[NR_TASKS + NR_PROCS];	/* process table */
```

There are some macros about the global `proc` variable.
``` c
/** proc.h */
#define BEG_PROC_ADDR (&proc[0])                    // begin of process address
#define BEG_USER_ADDR (&proc[NR_TASKS])             // begin of user process address
#define END_PROC_ADDR (&proc[NR_TASKS + NR_PROCS])  // end of process address

#define proc_addr(n)  (&(proc[NR_TASKS + (n)]))     // get process table address by process number
#define proc_nr(p) 	  ((p)->p_nr)                   // get process number

/** whether a process number is valid */
#define isokprocn(n)      ((unsigned) ((n) + NR_TASKS) < NR_PROCS + NR_TASKS)

/** whetehr a process table is free */
#define isemptyn(n)       isemptyp(proc_addr(n))
#define isemptyp(p)       ((p)->p_rts_flags == RTS_SLOT_FREE)

/** true when p or n represent a kernel task */
#define iskernelp(p)	  ((p) < BEG_USER_ADDR)
#define iskerneln(n)	  ((n) < 0)

/** true when p or n is not a kernel task */
#define isuserp(p)        isusern((p) >= BEG_USER_ADDR)
#define isusern(n)        ((n) >= 0)

/** true when n is root system process */
#define isrootsysn(n)	  ((n) == ROOT_SYS_PROC_NR)
```


### process message

``` c
/* minix/ipc.h */
typedef struct noxfer_message {
	endpoint_t m_source; /* who sent the message */
	int m_type;	         /* what kind of message is it */
    union ...            /* message type related data */
} message;
```


### scheduler


### API

| function | type | location | export | brief description |
|:--- |:--- |:--- |:--- |:--- |
| [**idle**](#idle) | `void idle()` | *proc.c* | false | when three isn't runable process, run idle |
| [**halt_cpu**](#halt-cpu) | `void halt_cpu()` | *arch/i386/klib.S* | true | halt cpu util next interrupt |
| [**arch_pause**](#arch-pause) | `void arch_pause()` | *arch/i386/klib.S* | true | slow cpu |
| [**interrupts_enable**](#interrupts-enable) | `void (*)()` | *arch/i386/klib.S* | true | `sti` instruction |
| [**interrupts_disable**](#interrupts-disable) | `void (*)()` | *arch/i386/klib.S* | true | `cli` instruction |


#### idle
[minix idle](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/proc.c#L176)

*idle()* are called from *switch_to_user()* when there isn't runable process.
idle() has two mode, profiling idle time or not.
If profiling is running, TODO.
Otherwise idle just halt CPU until next external interrupt.

when *idle()* return to *switch_to_user()*, if there still isn't runable process, *idle()* will be call again.


#### halt_cpu
[minix halt_cpu](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/arch/i386/klib.S#L407)

The i386 version of **halt_cpu** just three instruction
``` asm
sti
hlt
ret
```
Which enable interrupt and wait external interrupt. After external interrupt fired, excuction is continue.

| instruction | brief description |
|:--- |:--- |
|`sti` | Set interrupt flags, external and maskable interrupt enabled **at the end of the next instruction** |
|`hlt` | Halt CPU until external interrupt raise |


#### arch_pause
[minix arch_pause](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/arch/i386/klib.S#L785)

In i386 this function is a `pause` instruction which hint the processor spin-wait loop


#### interrupts_enable
[minix interrupts_enable](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/arch/i386/klib.S#L797)

In i386 this function is a `sti` instruction


#### interrupts_disable
[minix interrupts_disable](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/arch/i386/klib.S#L801)

In i386 this function is a `cli` instruction

