## IPC In Kernel


### Call IPC from process

IPC in minix is used to implement request call between different operating system component.
Components of operating system make an IPC through the IPC call API. (in system library)

TODO add description
| function | type | brief description |
|:--- |:--- |:--- |
| *ipc_send*    | `int ipc_send(endpoint_t dest, message *m_ptr)`            | send message |
| *ipc_receive* | `int ipc_receive(endpoint_t src, message *m_ptr, int *st)` | |
| *ipc_sendrec* | `int ipc_sendrec(endpoint_t src_dest, message *m_ptr)`     | |
| *ipc_sendnb*  | `int ipc_sendnb(endpoint_t dest, message *m_ptr)`          | send message non-blocking|
| *ipc_notify*  | `int ipc_notify(endpoint_t dest)`                          | send notify |
| *ipc_senda*   | `int ipc_senda(asynmsg_t *table, size_t count)`            | |

These function will call correspond assembly code, the corresponding functions in asm look like
``` asm
ENTRY(_ipc_senda_intr)
	push	%ebp
	movl	%esp, %ebp
	push	%ebx
    movl    TABCOUNT(%ebp), %eax    /* eax = count */
    movl    MSGTAB(%ebp), %ebx      /* ebx = table */
    movl    $SENDA, %ecx            /* _ipc_senda(table, count) */
    int $IPCVEC_INTR                /* trap to the kernel */
    pop %ebx
    pop %ebp
    ret
```
The mechanism of trap to kernel is **int** instruction with interrupt number **IPCVEC_INTR** := 33.


### Handle IPC Request in Kernel

By *interrupt number := 33* the interrupt handler be found as `ipc_entry_softint_orig`
``` asm
/** file: arch/i386/mpx.S */

/* IPC is only from a process to kernel */
ENTRY(ipc_entry_softint_orig)
	SAVE_PROCESS_CTX(0, KTS_INT_ORIG)
	jmp ipc_entry_common

ENTRY(ipc_entry_common)
	/* save the pointer to the current process */
	push	%ebp

	/*
	 * pass the syscall arguments from userspace to the handler.
	 * SAVE_PROCESS_CTX() does not clobber these registers, they are still
	 * set as the userspace have set them
	 */
	push	%ebx
	push	%eax
	push	%ecx

	/* stop user process cycles */
	push	%ebp
	/* for stack trace */
	movl	$0, %ebp
	call	_C_LABEL(context_stop)
	add	$4, %esp

	call	_C_LABEL(do_ipc)

	/* restore the current process pointer and save the return value */
	add	$3 * 4, %esp
	pop	%esi
	mov     %eax, AXREG(%esi)

	jmp	_C_LABEL(switch_to_user)
```

It's very similar to the kernel call interrupt handle in [kernel call](#kernel-call) section, except 
the true handler is `do_ipc()` function instead of `kernel_call()` function. So we should just focus on
the `do_ipc()` function.

#### do_ipc
[minix do_ipc](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/proc.c#L599)

``` c
int do_ipc(reg_t r1, reg_t r2, reg_t r3)
{
  /** Because do_ipc() only call when handling IPC interrupt, 
      the caller process will be the running process in current 
      processor */
  struct proc *const caller_ptr = get_cpulocal_var(proc_ptr);   /* get pointer to caller */
  int call_nr = (int) r1;

  ... ignore a lot of codes ... someting about [syscall trace] and [syscall defer]

  switch(call_nr) {
    case SENDREC:
    case SEND:          
    case RECEIVE:           
    case NOTIFY:
    case SENDNB:
    {
        caller_ptr->p_accounting.ipc_sync++;

        return do_sync_ipc(caller_ptr, call_nr, (endpoint_t) r2,
                (message *) r3);
    }
    case SENDA:
    {
        size_t msg_size = (size_t) r2;
        caller_ptr->p_accounting.ipc_async++;
 
        /* Limit size to something reasonable. An arbitrary choice is 16
         * times the number of process table entries. */
        if (msg_size > 16*(NR_TASKS + NR_PROCS))
            return EDOM;
        return mini_senda(caller_ptr, (asynmsg_t *) r3, msg_size);
    }
    case MINIX_KERNINFO:
        ... request kernel info
        break;
    default:
    return EBADCALL; /* illegal system call */
  }
}

static int do_sync_ipc(struct proc * caller_ptr, int call_nr,
                       endpoint_t src_dst_e, message *m_ptr)
{
  ... codes to validate this call 

  switch(call_nr) {
  case SENDREC:
    caller_ptr->p_misc_flags |= MF_REPLY_PEND;
  case SEND:            
    result = mini_send(caller_ptr, src_dst_e, m_ptr, 0);
    if (call_nr == SEND || result != OK) break;
  case RECEIVE:         
    if (call_nr == RECEIVE) {
        caller_ptr->p_misc_flags &= ~MF_REPLY_PEND;
        IPC_STATUS_CLEAR(caller_ptr);
    }
    result = mini_receive(caller_ptr, src_dst_e, m_ptr, 0);
    break;
  case NOTIFY:
    result = mini_notify(caller_ptr, src_dst_e);
    break;
  case SENDNB:
    result = mini_send(caller_ptr, src_dst_e, m_ptr, NON_BLOCKING);
    break;
  default:
    result = EBADCALL;
  }

  return(result);
}
```

According to above code, most IPC call has a corresponding kernel function, as follow
| IPC call | kernel function |
|:--- |:--- |
| *ipc_send*    | *mini_send* |
| *ipc_receive* | block for waiting message |
| *ipc_sendrec* | *mini_send* and block for waiting message |
| *ipc_sendnb*  | *mini_sned* with non-blocking flags |
| *ipc_notify*  | *mini_notify* |
| *ipc_senda*   | *mini_senda* |


#### mini_send
[minix mini_send](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/proc.c#L870)

**mini_send** send message from *sender* to *reciever*.
If *reciever* is ready to recieve a message from *sender*, 
the message is delivering to *reciever* and the PENDING flags of *reciever* be clear,
so the *reciever* maybe become runable and enqueue to ready queue, the *sender* will just continue.
If *reciever* isn't ready to recieve a message from *sender*, 
just return NOTREADY if non-blocking flag is set, otherwise *sender* will be blocked with SENDING.


#### mini_senda
[minix mini_senda](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/proc.c#L1331)

**mini_sneda** require privilege. This function send multiple message to multiple destination,
which doesn't blocke *sender* when destination doesn't ready the message just be ignored.
So this function doesn't guarantee the message be delivered successfully when get a OK result.


#### mini_notify
[minix mini_notify](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/proc.c#L1122)

Like *mini_send* with non-blocking flags, except a bad destination always return OK

