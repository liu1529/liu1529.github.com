---
title: 嵌入式操作系统的上下文切换
layout: post
tags: [RT-Thread,OS]
comments: true
---

我们知道嵌入式操作系统都是多任务操作系统，每个任务都有自己的优先级和时间片，每个任务也有自己独立的堆栈存储数据。当任务执行完
自己的时间片后，操作系统会根据优先级选择任务，然后切换到下一个任务。那么操作系统怎么进行任务切换的？怎么保证切换完成后，
原来任务的数据不丢失，新的任务使用另外一套数据，这就涉及到上下文切换。本文以RT-Thread实时操作系统为例，介绍操作系统的任务
切换和上下文切换。本文的RT-Thread版本为1.2.1，编译环境采用MDK4.73.00。

# 任务
任务是操作系统的基本组成单元，我们截取RT-Thread与上下文切换有关的任务定义如下：

{% highlight c %}
struct rt_thread
{
    ...
 
    /* stack point and entry */
    void       *sp;                                     /**< stack point */
    void       *entry;                                  /**< entry */
    void       *parameter;                              /**< parameter */
    void       *stack_addr;                             /**< stack address */
    rt_uint16_t stack_size;                             /**< stack size */

    ...
};

{% endhighlight %}

在这里，我们只关心和上下文切换有关的部分。

* sp 为堆栈指针，这是上下文切换的重点
* entry 为任务入口函数
* parameter 为入口函数的参数
* stack_addr 为任务的堆栈指针
* stack_size 为任务的堆栈大小


## 任务初始化

与上下文切换有关的任务初始化如下：

{% highlight c %}
static rt_err_t _rt_thread_init(struct rt_thread *thread,
                                const char       *name,
                                void (*entry)(void *parameter),
                                void             *parameter,
                                void             *stack_start,
                                rt_uint32_t       stack_size,
                                rt_uint8_t        priority,
                                rt_uint32_t       tick)
{
    ...
 
    thread->entry = (void *)entry;
    thread->parameter = parameter;

    /* stack init */
    thread->stack_addr = stack_start;
    thread->stack_size = (rt_uint16_t)stack_size;

    /* init thread stack */
    rt_memset(thread->stack_addr, '#', thread->stack_size);
    thread->sp = (void *)rt_hw_stack_init(thread->entry, thread->parameter,
        (void *)((char *)thread->stack_addr + thread->stack_size - 4),
        (void *)rt_thread_exit);

    ...
}

{% endhighlight %}

其中的重点是初始化sp指针的部分，`rt_hw_stack_init` 定义如下：

{% highlight c %}

struct exception_stack_frame
{
    rt_uint32_t r0;
    rt_uint32_t r1;
    rt_uint32_t r2;
    rt_uint32_t r3;
    rt_uint32_t r12;
    rt_uint32_t lr;
    rt_uint32_t pc;
    rt_uint32_t psr;
};

struct stack_frame
{
    /* r4 ~ r7 low register */
    rt_uint32_t r4;
    rt_uint32_t r5;
    rt_uint32_t r6;
    rt_uint32_t r7;

    /* r8 ~ r11 high register */
    rt_uint32_t r8;
    rt_uint32_t r9;
    rt_uint32_t r10;
    rt_uint32_t r11;

    struct exception_stack_frame exception_stack_frame;
};


rt_uint8_t *rt_hw_stack_init(void       *tentry,
                             void       *parameter,
                             rt_uint8_t *stack_addr,
                             void       *texit)
{
    struct stack_frame *stack_frame;
    rt_uint8_t         *stk;
    unsigned long       i;

    stk  = stack_addr + sizeof(rt_uint32_t);
    stk  = (rt_uint8_t *)RT_ALIGN_DOWN((rt_uint32_t)stk, 8);
    stk -= sizeof(struct stack_frame);

    stack_frame = (struct stack_frame *)stk;

    /* init all register */
    for (i = 0; i < sizeof(struct stack_frame) / sizeof(rt_uint32_t); i ++)
    {
        ((rt_uint32_t *)stack_frame)[i] = 0xdeadbeef;
    }

    stack_frame->exception_stack_frame.r0  = (unsigned long)parameter; /* r0 : argument */
    stack_frame->exception_stack_frame.r1  = 0;                        /* r1 */
    stack_frame->exception_stack_frame.r2  = 0;                        /* r2 */
    stack_frame->exception_stack_frame.r3  = 0;                        /* r3 */
    stack_frame->exception_stack_frame.r12 = 0;                        /* r12 */
    stack_frame->exception_stack_frame.lr  = (unsigned long)texit;     /* lr */
    stack_frame->exception_stack_frame.pc  = (unsigned long)tentry;    /* entry point, pc */
    stack_frame->exception_stack_frame.psr = 0x01000000L;              /* PSR */

    /* return task's current stack address */
    return stk;
}

{% endhighlight %}


<figure>
    <img src="/assets/rt-thread/thread-stack.png">
</figure>

RT-Thread将任务的stack分为两部分，最上面的部分为stack_frame用于存储任务的寄存器信息。
stack_frame又分为两部分，第一部分为R4-R11寄存器，这部分寄存器在MCU进入异常(中断)时
MCU不会自动压栈， 需要在任务切换时，手动存储；另一部为R0-R3、R12、LR、PC、PSR寄存器，
这些寄存器，在进入异常时由硬件自动压入堆栈。

`rt_hw_stack_init`用于初始化每个任务的stack_frame部分。将R4-R7初始化为0xdeadbeef，将
R0初始化为入口函数参数，R1-R3、R12初始化为0，LR为`rt_thread_exit`，PC为任务入口函数，
PSR为0x01000000。函数返回后，
`thread-sp = thread->stack_addr + thread->stack_size - sizeof(struct stack_frame)`
即上图中sp所在位置。

> PSR = APSR + EPSR + IPSR    
> IPSR 当前服务中断号寄存器   
> EPSR 执行状态寄存器（读回来的总是0）。它里面含T位，在Cortex-M中T位必须是1。   
> APSR 上条指令结果的标志   
> 具体参考[ARM](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0552a/CHDBIBGJ.html) 

# 上下文切换

所有的上下文切换函数都是用汇编进行书写，为了便于理解，我将其全部翻译为c语言。完整的汇编文件和对应c语言在文后。

## 系统首次切换上下文

系统初始化完成后，会切换到当前系统中优先级最高的任务。
任务首次进行上下文切换时，调用`rt_hw_context_switch_to`。

{% highlight asm %}

;/*
; * void rt_hw_context_switch_to(rt_uint32 to);
; * r0 --> to
; * this fucntion is used to perform the first thread switch
; */
rt_hw_context_switch_to    PROC
    EXPORT rt_hw_context_switch_to
    ; set to thread
    LDR     r1, =rt_interrupt_to_thread
    STR     r0, [r1]

    ; set from thread to 0
    LDR     r1, =rt_interrupt_from_thread
    MOVS    r0, #0x0
    STR     r0, [r1]

    ; set interrupt flag to 1
    LDR     r1, =rt_thread_switch_interrupt_flag
    MOVS    r0, #1
    STR     r0, [r1]

    ; set the PendSV exception priority
    LDR     r0, =NVIC_SHPR3
    LDR     r1, =NVIC_PENDSV_PRI
    LDR     r2, [r0,#0x00]       ; read
    ORRS    r1,r1,r2             ; modify
    STR     r1, [r0]             ; write-back

    ; trigger the PendSV exception (causes context switch)
    LDR     r0, =NVIC_INT_CTRL
    LDR     r1, =NVIC_PENDSVSET
    STR     r1, [r0]
    NOP

    ; restore MSP
    LDR     r0, =SCB_VTOR
    LDR     r0, [r0]
    LDR     r0, [r0]
    NOP
    MSR     msp, r0

    ; enable interrupts at processor level
    CPSIE   I

    ; never reach here!
    ENDP
{% endhighlight %}


其对应的c语言为：

{% highlight c %}

void rt_hw_context_switch_to(rt_uint32_t to)
{
	
	rt_interrupt_to_thread = to;
	rt_interrupt_from_thread = 0;
	rt_thread_switch_interrupt_flag = 1;

	//set the PendSV exception priority(lowest)
	NVIC_SetPriority(PendSV_IRQn, 0xff);
	//trigger the PendSV exception (causes context switch)
	SCB->ICSR = NVIC_PENDSVSET;
	//restore MSP
	__set_MSP(**(uint32_t**)SCB_VTOR);
	//enable interrupts at processor level
	__enable_irq();

	//never reach here
}

{% endhighlight %}



{% highlight asm %}
SCB_VTOR        EQU     0xE000ED08               ; Vector Table Offset Register
NVIC_INT_CTRL   EQU     0xE000ED04               ; interrupt control state register
NVIC_SHPR3      EQU     0xE000ED20               ; system priority register (2)
NVIC_PENDSV_PRI EQU     0x00FF0000               ; PendSV priority value (lowest)
NVIC_PENDSVSET  EQU     0x10000000               ; value to trigger PendSV exception

    AREA |.text|, CODE, READONLY, ALIGN=2
    THUMB
    REQUIRE8
    PRESERVE8

    IMPORT rt_thread_switch_interrupt_flag
    IMPORT rt_interrupt_from_thread
    IMPORT rt_interrupt_to_thread

;/*
; * rt_base_t rt_hw_interrupt_disable();
; */
rt_hw_interrupt_disable    PROC
    EXPORT  rt_hw_interrupt_disable
    MRS     r0, PRIMASK
    CPSID   I
    BX      LR
    ENDP

;/*
; * void rt_hw_interrupt_enable(rt_base_t level);
; */
rt_hw_interrupt_enable    PROC
    EXPORT  rt_hw_interrupt_enable
    MSR		PRIMASK, r0
    BX		LR
    ENDP

;/*
; * void rt_hw_context_switch(rt_uint32 from, rt_uint32 to);
; * r0 --> from
; * r1 --> to
; */
rt_hw_context_switch_interrupt
    EXPORT rt_hw_context_switch_interrupt
rt_hw_context_switch    PROC
    EXPORT rt_hw_context_switch

    ; set rt_thread_switch_interrupt_flag to 1
    LDR     r2, =rt_thread_switch_interrupt_flag
    LDR     r3, [r2]
    CMP     r3, #1
    BEQ     _reswitch
    MOVS    r3, #0x01
    STR     r3, [r2]

    LDR     r2, =rt_interrupt_from_thread   ; set rt_interrupt_from_thread
    STR     r0, [r2]

_reswitch
    LDR     r2, =rt_interrupt_to_thread     ; set rt_interrupt_to_thread
    STR     r1, [r2]

    LDR     r0, =NVIC_INT_CTRL              ; trigger the PendSV exception (causes context switch)
    LDR     r1, =NVIC_PENDSVSET
    STR     r1, [r0]
    BX      LR
    ENDP

; r0 --> swith from thread stack
; r1 --> swith to thread stack
; psr, pc, lr, r12, r3, r2, r1, r0 are pushed into [from] stack
PendSV_Handler    PROC
    EXPORT PendSV_Handler

    ; disable interrupt to protect context switch
    MRS     r2, PRIMASK
    CPSID   I

    ; get rt_thread_switch_interrupt_flag
    LDR     r0, =rt_thread_switch_interrupt_flag
    LDR     r1, [r0]
    CMP     r1, #0x00
    BEQ     pendsv_exit                ; pendsv already handled

    ; clear rt_thread_switch_interrupt_flag to 0
    MOVS    r1, #0x00
    STR     r1, [r0]

    LDR     r0, =rt_interrupt_from_thread
    LDR     r1, [r0]
    CMP     r1, #0x00
    BEQ     swtich_to_thread        ; skip register save at the first time

    MRS     r1, psp                 ; get from thread stack pointer

    SUBS    r1, r1, #0x20           ; space for {r4 - r7} and {r8 - r11}
    LDR     r0, [r0]
    STR     r1, [r0]                ; update from thread stack pointer

    STMIA   r1!, {r4 - r7}          ; push thread {r4 - r7} register to thread stack

    MOV     r4, r8                  ; mov thread {r8 - r11} to {r4 - r7}
    MOV     r5, r9
    MOV     r6, r10
    MOV     r7, r11
    STMIA   r1!, {r4 - r7}          ; push thread {r8 - r11} high register to thread stack

swtich_to_thread
    LDR     r1, =rt_interrupt_to_thread
    LDR     r1, [r1]
    LDR     r1, [r1]                ; load thread stack pointer

    LDMIA   r1!, {r4 - r7}          ; pop thread {r4 - r7} register from thread stack
    PUSH    {r4 - r7}               ; push {r4 - r7} to MSP for copy {r8 - r11}

    LDMIA   r1!, {r4 - r7}          ; pop thread {r8 - r11} high register from thread stack to {r4 - r7}
    MOV     r8,  r4                 ; mov {r4 - r7} to {r8 - r11}
    MOV     r9,  r5
    MOV     r10, r6
    MOV     r11, r7

    POP     {r4 - r7}               ; pop {r4 - r7} from MSP

    MSR     psp, r1                 ; update stack pointer

pendsv_exit
    ; restore interrupt
    MSR     PRIMASK, r2

    MOVS    r0, #0x04
    RSBS    r0, r0, #0x00
    BX      r0
    ENDP

;/*
; * void rt_hw_context_switch_to(rt_uint32 to);
; * r0 --> to
; * this fucntion is used to perform the first thread switch
; */
rt_hw_context_switch_to    PROC
    EXPORT rt_hw_context_switch_to
    ; set to thread
    LDR     r1, =rt_interrupt_to_thread
    STR     r0, [r1]

    ; set from thread to 0
    LDR     r1, =rt_interrupt_from_thread
    MOVS    r0, #0x0
    STR     r0, [r1]

    ; set interrupt flag to 1
    LDR     r1, =rt_thread_switch_interrupt_flag
    MOVS    r0, #1
    STR     r0, [r1]

    ; set the PendSV exception priority
    LDR     r0, =NVIC_SHPR3
    LDR     r1, =NVIC_PENDSV_PRI
    LDR     r2, [r0,#0x00]       ; read
    ORRS    r1,r1,r2             ; modify
    STR     r1, [r0]             ; write-back

    ; trigger the PendSV exception (causes context switch)
    LDR     r0, =NVIC_INT_CTRL
    LDR     r1, =NVIC_PENDSVSET
    STR     r1, [r0]
    NOP

    ; restore MSP
    LDR     r0, =SCB_VTOR
    LDR     r0, [r0]
    LDR     r0, [r0]
    NOP
    MSR     msp, r0

    ; enable interrupts at processor level
    CPSIE   I

    ; never reach here!
    ENDP

; compatible with old version
rt_hw_interrupt_thread_switch PROC
    EXPORT rt_hw_interrupt_thread_switch
    BX      lr
    ENDP

    IMPORT rt_hw_hard_fault_exception

HardFault_Handler    PROC
    EXPORT HardFault_Handler

    ; get current context
    MRS     r0, psp                 ; get fault thread stack pointer
    PUSH    {lr}
    BL      rt_hw_hard_fault_exception
    POP     {pc}
    ENDP

    END


{% endhighlight %}
 
