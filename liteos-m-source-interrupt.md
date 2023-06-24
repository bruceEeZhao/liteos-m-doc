# 初始化

ArchInit() ---> HalHwiInit() 

```c
LITE_OS_SEC_TEXT_INIT VOID HalHwiDefaultHandler(VOID *arg)
{
    (VOID)arg;
    PRINT_ERR("default handler\n");
    while (1) {
    }
}

/**
 * @brief 中断处理函数初始化，初始化为默认handler
 * 
 * @return LITE_OS_SEC_TEXT_INIT 
 */
LITE_OS_SEC_TEXT_INIT VOID HalHwiInit(VOID)
{
    UINT32 index;
    for (index = OS_RISCV_SYS_VECTOR_CNT; index < OS_HWI_MAX_NUM; index++) {
        g_hwiForm[index].pfnHook = HalHwiDefaultHandler;
        g_hwiForm[index].uwParam = 0;
    }
}
```

# 创建中断

`LOS_HwiCreate()->ArchHwiCreate`

```c
LITE_OS_SEC_TEXT UINT32 ArchHwiCreate(HWI_HANDLE_T hwiNum,
                                      HWI_PRIOR_T hwiPrio,
                                      HWI_MODE_T hwiMode,
                                      HWI_PROC_FUNC hwiHandler,
                                      HwiIrqParam *irqParam)
{
    UINT32 intSave;

    // 参数合法性校验
    if (hwiHandler == NULL) {
        return OS_ERRNO_HWI_PROC_FUNC_NULL;
    }
    if (hwiNum >= OS_HWI_MAX_NUM) {
        return OS_ERRNO_HWI_NUM_INVALID;
    }
    if (g_hwiForm[hwiNum].pfnHook == NULL) {
        return OS_ERRNO_HWI_NUM_INVALID;
    } else if (g_hwiForm[hwiNum].pfnHook != HalHwiDefaultHandler) {
        return OS_ERRNO_HWI_NUM_INVALID;
    }
    if ((hwiPrio < OS_HWI_PRIO_LOWEST) || (hwiPrio > OS_HWI_PRIO_HIGHEST)) {
        return OS_ERRNO_HWI_PRIO_INVALID;
    }

    intSave = LOS_IntLock();
    // 注册中断处理函数
    g_hwiForm[hwiNum].pfnHook = hwiHandler;
    if (irqParam != NULL) {
        g_hwiForm[hwiNum].uwParam = (VOID *)irqParam->pDevId;
    } else {
        g_hwiForm[hwiNum].uwParam = NULL;
    }
    if (hwiNum >= OS_RISCV_SYS_VECTOR_CNT) {
        HalSetLocalInterPri(hwiNum, hwiPrio);
    }

    LOS_IntRestore(intSave);

    return LOS_OK;
}
```

# 删除中断

```c
LITE_OS_SEC_TEXT UINT32 ArchHwiDelete(HWI_HANDLE_T hwiNum, HwiIrqParam *irqParam)
{
    (VOID)irqParam;
    UINT32 intSave;

    if (hwiNum >= OS_HWI_MAX_NUM) {
        return OS_ERRNO_HWI_NUM_INVALID;
    }

    intSave = LOS_IntLock();
    // 中断处理函数重新指向默认处理函数
    g_hwiForm[hwiNum].pfnHook = HalHwiDefaultHandler;
    g_hwiForm[hwiNum].uwParam = 0;
    LOS_IntRestore(intSave);
    return LOS_OK;
}

```

# 系统中断入口函数

```c
/**
 * @brief 系统中断入口函数
 * 
 */
__attribute__((section(".interrupt.text"))) VOID HalHwiInterruptDone(HWI_HANDLE_T hwiNum)
{   
    // g_intCount++,声明此时处于中断处理
    g_intCount++;

    OsHookCall(LOS_HOOK_TYPE_ISR_ENTER, hwiNum);

    HWI_HANDLE_FORM_S *hwiForm = &g_hwiForm[hwiNum];
    HwiProcFunc func = (HwiProcFunc)(hwiForm->pfnHook);
    // 调用相应的中断处理函数，进行中断处理
    func(hwiForm->uwParam);
    // 记录中断执行次数
    ++g_hwiFormCnt[hwiNum];

    OsHookCall(LOS_HOOK_TYPE_ISR_EXIT, hwiNum);
    // 声明结束中断处理状态
    g_intCount--;
}
```

该函数由汇编代码进行调用

```c
.section .interrupt.HalTrapVector.text
.extern HalTrapEntry
.extern HalIrqEndCheckNeedSched
.global HalTrapVector
.equ TRAP_INTERRUPT_MODE_MASK, 0x80000000
.align 4
HalTrapVector:
    PUSH_CALLER_REG   # 保存现场
    csrr  a0, mcause  # Machine trap cause.寄存器最高位表示中断位，为1表示是一个中断
    li    a1, TRAP_INTERRUPT_MODE_MASK  # a1 = 0x80000000
    li    a2, MCAUSE_INT_ID_MASK # a2 = 0x7FFFFFF
    and   a1, a0, a1 
    and   a0, a2, a0
    beqz  a1, HalTrapEntry # 最高位是0,表示是一个trap，进入trap处理
    csrw  mscratch, sp     # mscratch用于保存指向机器模式hart本地上下文空间的指针，并在进入m模式trap处理程序时与用户寄存器进行交换
    la    sp, __start_and_irq_stack_top
    jal   HalHwiInterruptDone
    csrr  sp, mscratch     # 恢复sp
    call  HalIrqEndCheckNeedSched

    POP_CALLER_REG  # 恢复现场
    mret
```

