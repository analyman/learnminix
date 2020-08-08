## kernel call


### types and variables

| type | brief description |
|:---|:---|
| `struct proc` | @see process |
| `message`     | @see process |
| kernel_call_prototype | `int (*)(struct proc* pr, message* msg)` |


| variable | brief description |
|:---|:---|
| `call_vec` | `static int (*call_vec[NR_SYS_CALLS])(struct proc * caller, message *m_ptr);`, defined in `system.c`. 
               used in `kernel_call_dispatch()`|


### kernel calls

kernel calls has prototype `int (*)(struct proc* pr, message* msg)` 
which will be called in `int do_kernel_dispatch(struc proc*, message*)` function.

TODO
| kernel call | brief description |
|:---|:---|
| do_exec          | |
| do_fork          | |
| do_clear         | |
| do_trace         | |
| do_runctl        | |
| do_update        | |
| do_exit          | |
| do_copy          | |
| do_umap          | |
| do_umap_remote   | |
| do_vumap         | |
| do_memset        | |
| do_abort         | |
| do_getinfo       | |
| do_privctl       | |
| do_irqctl        | |
| do_devio         | |
| do_vdevio        | |
| do_sdevio        | |
| do_kill          | |
| do_getksig       | |
| do_endksig       | |
| do_sigsend       | |
| do_sigreturn     | |
| do_times         | |
| do_setalarm      | |
| do_stime         | |
| do_settime       | |
| do_vtimer        | |
| do_safecopy_to   | |
| do_safecopy_from | |
| do_vsafecopy     | |
| do_iopenable     | |
| do_vmctl         | |
| do_setgrant      | |
| do_readbios      | |
| do_safememset    | |
| do_sprofile      | |
| do_getmcontext   | |
| do_setmcontext   | |
| do_schedule      | |
| do_schedctl      | |
| do_statectl      | |
| do_padconf       | |


### intrrupt description table (IDT)


### call stack of a kernel call

According to `minix/lib/libsys/kernel_call.c#_kernel_call(int syscallnr, message* msgptr)`.
We can know kernel call will call `do_kernel_call(message*)` function, and it be found in `minix/ipc.h` is a macro
``` c
/** minix/ipc.h */
#define do_kernel_call	_do_kernel_call
```

`_do_kernel_call()` function is an inline function defined as
``` c
static inline int _do_kernel_call(message *m_ptr)
{
	return _minix_ipcvecs.do_kernel_call(m_ptr);
}
```

The `_minix_ipcvecs` define by
``` c
struct minix_ipcvecs _minix_ipcvecs = {
	.sendrec	= _ipc_sendrec_intr,
	.send		= _ipc_send_intr,
	.notify		= _ipc_notify_intr,
	.senda		= _ipc_senda_intr,
	.sendnb		= _ipc_sendnb_intr,
	.receive	= _ipc_receive_intr,
	.do_kernel_call	= _do_kernel_call_intr,
};
```

So do kernel call finially will call the `_do_kernel_call_intr()` function 
(i386 has **int** and **sysenter**, x86_64 add **syscall** instruction).

i386 version of `_do_kernel_call_intr()` was found in `minix/lib/libc/arch/i386/sys/_do_kernel_call_intr.S`
``` asm
ENTRY(_do_kernel_call_intr)
	/* pass the message pointer to kernel in the %eax register */
	movl	4(%esp), %eax
	int	$KERVEC_INTR
	ret
```

The `$KERVEC_INTR` constant is `32`.

#### load IDT 
The intrrupt description table stored in *IDTR* register which load by `lidt`(`lidtl`) instruction. 
This instruction define as a function `x86_lidt`

``` asm
/** arch/i386/klib.S */
#define ARG_EAX_ACTION(FUNCTION, ACTION)        ;\
ENTRY(FUNCTION)                         ;\
        push    %ebp                    ;\
        mov     %esp, %ebp              ;\
        mov     STACKARG, %eax           ;\
        ACTION                          ;\
        pop     %ebp                    ;\
        ret

ARG_EAX_ACTION(x86_lidt, lidtl (%eax));
```

The `x86_lidt()` function called by `reload_idt()` function in `arch/i386/protect.c`.
`reload_idt()` call `x86_lidt(&idt_desc)`, which load `&idt_desc` into *IDTR* control register.

`id_desc` is initialized in `proc_init()` defined in `arch/i386/protect.c`.
``` c
idt_desc.base = (u32_t) idt;
idt_desc.limit = sizeof(idt)-1;
```

`idt` defined in `arch/i386/protect.c`,
``` c
// IDT_SIZE := 256. IDT_SIZE should less equal 0xFF, because operand type of int instruction is byte
struct gatedesc_s idt[IDT_SIZE] __aligned(DESC_SIZE);

struct gatedesc_s {
  u16_t offset_low;
  u16_t selector;
  u8_t pad;                     /* |000|XXXXX| ig & trpg, |XXXXXXXX| task g */
  u8_t p_dpl_type;              /* |P|DL|0|TYPE| */
  u16_t offset_high;
} __attribute__((packed));
```

`reload_idt()` called by `void prot_load_selectors(void)` 
which initialize `idt`, load *IDTR* and some other control register in `arch/i386/protect.c`.

`prot_load_selectors()` called by `prot_init()`, `prot_init()` called by `cstart()` in `main.c`, finnaly
`cstart()` called by `kmain()` in `main.c` which is the entry point of kernel.

`idt` initialzed by `idt_init()` in `protect.c` and `apic_idt_init()` in `arch/i386/apic.c`.
``` c
/** protect.c */
void idt_init(void)
{
	idt_copy_vectors_pic();
	idt_copy_vectors(gate_table_exceptions);
}

void idt_copy_vectors_pic(void)
{
	idt_copy_vectors(gate_table_pic);
}

void idt_init(void)
{
	idt_copy_vectors_pic();
	idt_copy_vectors(gate_table_exceptions);
}

void idt_copy_vectors(struct gate_table_s * first)
{
	struct gate_table_s *gtp;
	for (gtp = first; gtp->gate; gtp++) {
		int_gate(idt, gtp->vec_nr, (vir_bytes) gtp->gate,
				PRESENT | INT_GATE_TYPE |
				(gtp->privilege << DPL_SHIFT));
	}
}

static struct gate_table_s gate_table_pic[];
static struct gate_table_s gate_table_exceptions[]; // interrupt about exceptions


/** arch/i386/apic.c */
/* Build descriptors for interrupt gates in IDT. */
void apic_idt_init(const int reset)
{
	u32_t val;

	/* Set up idt tables for smp mode.
	 */
	int is_bsp;

	if (reset) {
		idt_copy_vectors_pic();
		idt_copy_vectors(gate_table_common);
		return;
	}

	is_bsp = is_boot_apic(apicid());

#ifdef APIC_DEBUG
	if (is_bsp)
		printf("APIC debugging is enabled\n");
	lapic_set_dummy_handlers();
#endif

	/* Build descriptors for interrupt gates in IDT. */
	if (ioapic_enabled)
		idt_copy_vectors(gate_table_ioapic);
	else
		idt_copy_vectors_pic();

	idt_copy_vectors(gate_table_common);

#ifdef CONFIG_SMP
	idt_copy_vectors(gate_table_smp);
#endif

	/* Setup error interrupt vector */
	val = lapic_read(LAPIC_LVTER);
	val |= APIC_ERROR_INT_VECTOR;
	val &= ~ APIC_ICR_INT_MASK;
	lapic_write(LAPIC_LVTER, val);
	(void) lapic_read(LAPIC_LVTER);

	/* configure the timer interupt handler */
	if (is_bsp) {
		BOOT_VERBOSE(printf("Initiating APIC timer handler\n"));
		/* register the timer interrupt handler for this CPU */
		int_gate_idt(APIC_TIMER_INT_VECTOR, (vir_bytes) lapic_timer_int_handler,
				PRESENT | INT_GATE_TYPE | (INTR_PRIVILEGE << DPL_SHIFT));
	}

}
static struct gate_table_s gate_table_ioapic[];
static struct gate_table_s gate_table_common[]; // interrupt about kernel call
static struct gate_table_s gate_table_smp[];
```

Corresponding idt entries of kernel call interrupt number `$KERVEC_INTR` found in 
`gate_table_common` and handler of the interrupt is `kernel_call_entry_orig` defined as
``` asm
/** arch/i386/mpx.S */
// kernel call is only from a process to kernel
ENTRY(kernel_call_entry_orig)
	SAVE_PROCESS_CTX(0, KTS_INT_ORIG)
	jmp	kernel_call_entry_common


ENTRY(kernel_call_entry_common)
	/* save the pointer to the current process */
	push	%ebp

	/*
	 * pass the syscall arguments from userspace to the handler.
	 * SAVE_PROCESS_CTX() does not clobber these registers, they are still
	 * set as the userspace have set them
	 */
	push	%eax

	/* stop user process cycles */
	push	%ebp
	/* for stack trace */
	movl	$0, %ebp
	call	_C_LABEL(context_stop)
	add	$4, %esp

	call	_C_LABEL(kernel_call)

	/* restore the current process pointer and save the return value */
	add	$8, %esp

	jmp	_C_LABEL(switch_to_user)
```

The above kernel call handler involves three C function `context_stop(struct proc* proc)`, `kernel_call` and `switch_to_user`.
If multiple symetric processors has config and `proc != proc_addr(KERNEL)`(I think this will be always **true** in kernel call), 
then `context_stop()` will call macro `BLK_LOCK()` prevent other processors access kernel code. 
The `switch_to_user()` call `context_stop()` again with `proc = proc_addr(KERNEL)`, and this `context_sotp()` function call 
call `BLK_UNLOCK()` macro which will relieve a processor from spinlock busy waiting if there are processors waiting.

So the actual work of kernel call will be performed by `kernel_call()` function in `system.c`, it dispatch message to correspond
kernel call handler as states above.

