# 初始化

信号量使用LosSemCB结构体进行管理

```c
typedef struct {
    UINT16 semStat;      /**< Semaphore state */
    UINT16 semCount;     /**< Number of available semaphores */
    UINT16 maxSemCount;  /**< Max number of available semaphores */
    UINT16 semID;        /**< Semaphore control structure ID */
    LOS_DL_LIST semList; /**< Queue of tasks that are waiting on a semaphore */
} LosSemCB;
```

系统中有48个semCB

```c
LITE_OS_SEC_TEXT_INIT UINT32 OsSemInit(VOID)
{
    LosSemCB *semNode = NULL;
    UINT16 index;

    // 初始化空闲链表
    LOS_ListInit(&g_unusedSemList);

    if (LOSCFG_BASE_IPC_SEM_LIMIT == 0) {
        return LOS_ERRNO_SEM_MAXNUM_ZERO;
    }

    // 申请空间，48个semCB
    g_allSem = (LosSemCB *)LOS_MemAlloc(m_aucSysMem0, (LOSCFG_BASE_IPC_SEM_LIMIT * sizeof(LosSemCB)));
    if (g_allSem == NULL) {
        return LOS_ERRNO_SEM_NO_MEMORY;
    }

    // 使用数组形式初始化，semID为数组下标，状态为unused，尾插法插入空闲链表
    /* Connect all the semaphore CBs in a doubly linked list. */
    for (index = 0; index < LOSCFG_BASE_IPC_SEM_LIMIT; index++) {
        semNode = ((LosSemCB *)g_allSem) + index;
        semNode->semID = index;
        semNode->semStat = OS_SEM_UNUSED;
        LOS_ListTailInsert(&g_unusedSemList, &semNode->semList);
    }
    return LOS_OK;
}
```

# 创建sem

```c
LITE_OS_SEC_TEXT_INIT UINT32 OsSemCreate(UINT16 count, UINT16 maxCount, UINT32 *semHandle)
{
    UINT32 intSave;
    LosSemCB *semCreated = NULL;
    LOS_DL_LIST *unusedSem = NULL;
    UINT32 errNo;
    UINT32 errLine;

    if (semHandle == NULL) {
        return LOS_ERRNO_SEM_PTR_NULL;
    }

    if (count > maxCount) {
        OS_GOTO_ERR_HANDLER(LOS_ERRNO_SEM_OVERFLOW);
    }

    intSave = LOS_IntLock();

    // 如果没有空闲的semCB，返回错误
    if (LOS_ListEmpty(&g_unusedSemList)) {
        LOS_IntRestore(intSave);
        OS_GOTO_ERR_HANDLER(LOS_ERRNO_SEM_ALL_BUSY);
    }

    // 取链表第一个元素，并从链表中删除
    unusedSem = LOS_DL_LIST_FIRST(&(g_unusedSemList));
    LOS_ListDelete(unusedSem);
    // 获取semCB，设置参数
    semCreated = (GET_SEM_LIST(unusedSem));
    semCreated->semCount = count;
    semCreated->semStat = OS_SEM_USED;
    semCreated->maxSemCount = maxCount;
    // 初始化链表
    LOS_ListInit(&semCreated->semList);
    *semHandle = (UINT32)semCreated->semID;
    LOS_IntRestore(intSave);
    OsHookCall(LOS_HOOK_TYPE_SEM_CREATE, semCreated);
    return LOS_OK;

ERR_HANDLER:
    OS_RETURN_ERROR_P2(errLine, errNo);
}
```

`OsSemCreate`函数的功能十分简单

1. 判断参数是否合法，若非法返回错误
2. 从空闲链表中取出第一个元素`unusedSem`，并将该元素从链表中删除
3. 获取`unusedSem`对应的semCB，对其进行参数设置，并初始化semList链表

该函数不会被直接调用，暴露给外部的有两个接口:

```c
LITE_OS_SEC_TEXT_INIT UINT32 LOS_SemCreate(UINT16 count, UINT32 *semHandle);
LITE_OS_SEC_TEXT_INIT UINT32 LOS_BinarySemCreate(UINT16 count, UINT32 *semHandle);
```

`LOS_SemCreate`用于创建一个普通的信号量，信号量的值大于1

`LOS_BinarySemCreate`用于创建一个只能表示01的信号量

# 删除sem

```c
LITE_OS_SEC_TEXT_INIT UINT32 LOS_SemDelete(UINT32 semHandle)
{
    UINT32 intSave;
    LosSemCB *semDeleted = NULL;
    UINT32 errNo;
    UINT32 errLine;

    // semid是否合法
    if (semHandle >= (UINT32)LOSCFG_BASE_IPC_SEM_LIMIT) {
        OS_GOTO_ERR_HANDLER(LOS_ERRNO_SEM_INVALID);
    }

    semDeleted = GET_SEM(semHandle);
    intSave = LOS_IntLock();
    if (semDeleted->semStat == OS_SEM_UNUSED) {
        LOS_IntRestore(intSave);
        OS_GOTO_ERR_HANDLER(LOS_ERRNO_SEM_INVALID);
    }
    // 如果semDeleted->semList非空，不可删除
    if (!LOS_ListEmpty(&semDeleted->semList)) {
        LOS_IntRestore(intSave);
        OS_GOTO_ERR_HANDLER(LOS_ERRNO_SEM_PENDED);
    }

    // 加入空闲链表
    LOS_ListAdd(&g_unusedSemList, &semDeleted->semList);
    semDeleted->semStat = OS_SEM_UNUSED;
    LOS_IntRestore(intSave);
    OsHookCall(LOS_HOOK_TYPE_SEM_DELETE, semDeleted);
    return LOS_OK;
ERR_HANDLER:
    OS_RETURN_ERROR_P2(errLine, errNo);
}
```

该函数的功能也很简单：

1. 判断semid是否合法，非法返回错误
2. 判断对应的semCB的状态，若为unused，返回错误
3. 如果semDeleted->semList非空，不可删除
4. 将semCB插入空闲链表中，将其状态设为unused



# P-V操作

## P操作 - LOS_SemPend

下面称p操作为加锁

```c
LITE_OS_SEC_TEXT UINT32 LOS_SemPend(UINT32 semHandle, UINT32 timeout)
{
    UINT32 intSave;
    LosSemCB *semPended = NULL;
    UINT32 retErr;
    LosTaskCB *runningTask = NULL;

    if (semHandle >= (UINT32)LOSCFG_BASE_IPC_SEM_LIMIT) {
        OS_RETURN_ERROR(LOS_ERRNO_SEM_INVALID);
    }

    semPended = GET_SEM(semHandle);
    intSave = LOS_IntLock();

    // 检查semCB是否合法，非法返回错误
    retErr = OsSemValidCheck(semPended);
    if (retErr) {
        goto ERROR_SEM_PEND;
    }

    runningTask = (LosTaskCB *)g_losTask.runTask;

    // 1. 如果信号量>0，自减，返回成功
    if (semPended->semCount > 0) {
        semPended->semCount--;
        LOS_IntRestore(intSave);
        OsHookCall(LOS_HOOK_TYPE_SEM_PEND, semPended, runningTask, timeout);
        return LOS_OK;
    }
    // 2. else semCount == 0
    // 如果timeout为0,返回错误
    if (!timeout) {
        retErr = LOS_ERRNO_SEM_UNAVAILABLE;
        goto ERROR_SEM_PEND;
    }

    // 在当前任务的taskSem记录当前等待的semCB
    runningTask->taskSem = (VOID *)semPended;
    // 加入等待队列
    OsSchedTaskWait(&semPended->semList, timeout);
    LOS_IntRestore(intSave);
    OsHookCall(LOS_HOOK_TYPE_SEM_PEND, semPended, runningTask, timeout);
    // 主动调度,等待sem可用
    LOS_Schedule();

    intSave = LOS_IntLock();
    // 当LOS_SemPost引发调度时，从此处开始执行，
    // 判断状态是否超时
    if (runningTask->taskStatus & OS_TASK_STATUS_TIMEOUT) {
        runningTask->taskStatus &= (~OS_TASK_STATUS_TIMEOUT);
        retErr = LOS_ERRNO_SEM_TIMEOUT;
        goto ERROR_SEM_PEND;
    }
    // 没有超时，返回加锁成功
    LOS_IntRestore(intSave);
    return LOS_OK;

ERROR_SEM_PEND:
    LOS_IntRestore(intSave);
    OS_RETURN_ERROR(retErr);
}
```





## V操作 - LOS_SemPost

```c
LITE_OS_SEC_TEXT UINT32 LOS_SemPost(UINT32 semHandle)
{
    UINT32 intSave;
    LosSemCB *semPosted = GET_SEM(semHandle);
    LosTaskCB *resumedTask = NULL;

    if (semHandle >= LOSCFG_BASE_IPC_SEM_LIMIT) {
        return LOS_ERRNO_SEM_INVALID;
    }

    intSave = LOS_IntLock();

    if (semPosted->semStat == OS_SEM_UNUSED) {
        LOS_IntRestore(intSave);
        OS_RETURN_ERROR(LOS_ERRNO_SEM_INVALID);
    }

    // 如果semCount == maxSemCount
    if (semPosted->maxSemCount == semPosted->semCount) {
        LOS_IntRestore(intSave);
        OS_RETURN_ERROR(LOS_ERRNO_SEM_OVERFLOW);
    }

    // 如果等待链表非空
    if (!LOS_ListEmpty(&semPosted->semList)) {
        // 从等待链表中取出一个task，设置其taskSem=NULL，并唤醒
        resumedTask = OS_TCB_FROM_PENDLIST(LOS_DL_LIST_FIRST(&(semPosted->semList)));
        resumedTask->taskSem = NULL;
        OsSchedTaskWake(resumedTask);

        LOS_IntRestore(intSave);
        OsHookCall(LOS_HOOK_TYPE_SEM_POST, semPosted, resumedTask);
        LOS_Schedule();
    } else {
        // semCount++
        semPosted->semCount++;
        LOS_IntRestore(intSave);
        OsHookCall(LOS_HOOK_TYPE_SEM_POST, semPosted, resumedTask);
    }

    return LOS_OK;
}
```

