## Clock


### files
| file | brief description |
|:--- |:--- |:--- |
| clock.h | just some function declaration |
| clock.c | timers interrupt handler, load statistics update, functions to get statistics about times and set time |
| lib/libtimers/timers_exp.c | expire a `minix_timer_t` linked list |
| lib/libtimers/timers_clr.c | clear a minix timer in a linked list |
| lib/libtimers/timers_set.c | add a minix timer to a linked list |


### types 
| type | where | brief description |
|:--- |:--- |:--- |
|`minix_timer_t`| minix/timer.h | timer with a callback, a sorted linked list |
|`tmr_func_t`   | minix/timer.h | timer callback function type, `void (*)(int arg)` |
|`struct kclockinfo` | type.h | clock information |

``` c
struct kclockinfo {
  time_t boottime;      /* number of seconds since UNIX epoch */
  clock_t uptime;       /* number of clock ticks since system boot */
  uint32_t _rsvd1;      /* reserved for 64-bit uptime */
  clock_t realtime;     /* real time in clock ticks since boot */
  uint32_t _rsvd2;      /* reserved for 64-bit real time */
  uint32_t hz;          /* clock frequency in ticks per second */
};
```


### variables
| variable | where | brief description |
|:--- |:--- |:--- |
| `kclockinfo`               | glo.h | global clock information |
| `kloadinfo` aka `loadinfo` | glo.h | status of load average |



### API

| function | prototype | where | export | brief description |
|:--- |:--- |:--- |:--- |:--- |
| [*timer_int_handler*](#timer-int-handler) | `int (*)(void)` | clock.c | true  | handler of timer interrupt |
| [*load_update*](#load-update)       | `void(*)(void)` | clock.c | false | update load information in `kloadinfo`, 
                                                            basicly number of ready process at this moment |


#### timer_int_handler
[minix timer_int_handler](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/clock.c#L70)

The timer interrupt handler, and `timer_int_handler()` called from `lapic_timer_int_handler` which defined as
``` asm
/** arch/i386/apic_asm.S */
/* apic timer tick handlers */
ENTRY(lapic_timer_int_handler)
    lapic_intr(_C_LABEL(timer_int_handler))
```

`lapic_timer_int_handler` is handler of **APIC_TIMER_INT_VECTOR** interrupt registered in `apic_idt_init()` function 
when kernel boot.
``` c
/** arch/i386/apic.c */
/* Build descriptors for interrupt gates in IDT. */
void apic_idt_init(const int reset)
{
    is_bsp = is_boot_apic(apicid());

    /* configure the timer interupt handler */
    if (is_bsp) {
        BOOT_VERBOSE(printf("Initiating APIC timer handler\n"));
        /* register the timer interrupt handler for this CPU */
        int_gate_idt(APIC_TIMER_INT_VECTOR, (vir_bytes) lapic_timer_int_handler,
                PRESENT | INT_GATE_TYPE | (INTR_PRIVILEGE << DPL_SHIFT));
    }
}
```

This handler increase a user tick of process of current processor and 
decrease **virtual timer** and **profile timer** of the process, if these timer expire after decreasing,
raise corresponding signal to the process.

The above processes also aply to **bill process** of current processor if this process is **BILLABLE**.

If current processor is bootstraping processor, this handler will check **clock_timer** whether has expired,
when true expire the timers. It also increase `kclockinfo.realtime` when current processor is bootstraping processor.


#### load_update
[minix load_update](https://github.com/Stichting-MINIX-Research-Foundation/minix/blob/master/minix/kernel/clock.c#L260)

store how many process is ready at this moment into `kloadinfo`, this information has history, keep a list of load at 
different moment(the interval may be very small).

