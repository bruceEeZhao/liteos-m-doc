# 初始化

event control structure

```c
typedef struct tagEvent {
    UINT32 uwEventID;        /**< Event mask in the event control block,
                                  indicating the event that has been logically processed. */
    LOS_DL_LIST stEventList; /**< Event control block linked list */
} EVENT_CB_S, *PEVENT_CB_S;
```



```c
LITE_OS_SEC_TEXT_INIT UINT32 LOS_EventInit(PEVENT_CB_S eventCB)
{
    if (eventCB == NULL) {
        return LOS_ERRNO_EVENT_PTR_NULL;
    }
    eventCB->uwEventID = 0;
    // 初始化链表
    LOS_ListInit(&eventCB->stEventList);
    OsHookCall(LOS_HOOK_TYPE_EVENT_INIT, eventCB);
    return LOS_OK;
}
```

event的初始化函数需要传入一个eventCB，这说明event的数量是不受限制的

event的mode可以是

```
LOS_WAITMODE_OR    // 发生任意一个等待的事件
LOS_WAITMODE_AND   // 所有等待的事件均发生
LOS_WAITMODE_CLR   // 事件被read时，立刻清除事件flag
```



# event 销毁

```c
LITE_OS_SEC_TEXT_INIT UINT32 LOS_EventDestroy(PEVENT_CB_S eventCB)
{
    UINT32 intSave;
    if (eventCB == NULL) {
        return LOS_ERRNO_EVENT_PTR_NULL;
    }
    intSave = LOS_IntLock();
    
    // event列表非空则不能删除
    if (!LOS_ListEmpty(&eventCB->stEventList)) {
        LOS_IntRestore(intSave);
        return LOS_ERRNO_EVENT_SHOULD_NOT_DESTROYED;
    }
    eventCB->stEventList.pstNext = (LOS_DL_LIST *)NULL;
    eventCB->stEventList.pstPrev = (LOS_DL_LIST *)NULL;
    LOS_IntRestore(intSave);
    OsHookCall(LOS_HOOK_TYPE_EVENT_DESTROY, eventCB);
    return LOS_OK;
}
```



# LOS_EventRead

event read表示一个task关注某一项事件或某几项事件的发生，

调用`LOS_EventPoll`对关注的事件是否发生进行判断，

1. 如果返回值不为0,则表示事件发生，返回成功
2. 如果返回0,表示事件还未发生，则需要等待，那么有两种情况
    1. timeout值为0,表示没有发生立刻返回，不等待
    2. timeout值不为0，将当前task加入eventCB维护的等待链表中，
        1. 设置task的eventMask，以便在`LOS_EventWrite`函数中直接读取task的eventMask判断该task关注的事件是否发生
        2. 设置task的eventMode
        3. 主动调用`LOS_Schedule`进行任务调度

```c
LITE_OS_SEC_TEXT UINT32 LOS_EventRead(PEVENT_CB_S eventCB, UINT32 eventMask, UINT32 mode, UINT32 timeOut)
{
    UINT32 ret;
    UINT32 intSave;
    LosTaskCB *runTsk = NULL;

    // 参数合法性校验
    ret = OsEventReadParamCheck(eventCB, eventMask, mode);
    if (ret != LOS_OK) {
        return ret;
    }
    // 正在执行中断
    if (OS_INT_ACTIVE) {
        return LOS_ERRNO_EVENT_READ_IN_INTERRUPT;
    }
    // 系统任务
    if (g_losTask.runTask->taskStatus & OS_TASK_FLAG_SYSTEM_TASK) {
        return LOS_ERRNO_EVENT_READ_IN_SYSTEM_TASK;
    }
    intSave = LOS_IntLock();
    ret = LOS_EventPoll(&(eventCB->uwEventID), eventMask, mode);
    OsHookCall(LOS_HOOK_TYPE_EVENT_READ, eventCB, eventMask, mode, timeOut);
    // ret == 0,表示事件没有发生，需要等待
    if (ret == 0) {
        // 不等待，直接返回
        if (timeOut == 0) {
            LOS_IntRestore(intSave);
            return ret;
        }

        if (g_losTaskLock) {
            LOS_IntRestore(intSave);
            return LOS_ERRNO_EVENT_READ_IN_LOCK;
        }
        runTsk = g_losTask.runTask;
        runTsk->eventMask = eventMask;
        runTsk->eventMode = mode;

        // 加入等待列表
        OsSchedTaskWait(&eventCB->stEventList, timeOut);
        LOS_IntRestore(intSave);
        LOS_Schedule();

        intSave = LOS_IntLock();
        // 如果超时，返回超时错误
        if (runTsk->taskStatus & OS_TASK_STATUS_TIMEOUT) {
            runTsk->taskStatus &= ~OS_TASK_STATUS_TIMEOUT;
            LOS_IntRestore(intSave);
            return LOS_ERRNO_EVENT_READ_TIMEOUT;
        }

        // 再次读取，看事件是否发生
        ret = LOS_EventPoll(&eventCB->uwEventID, eventMask, mode);
    }

    LOS_IntRestore(intSave);
    return ret;
}
```

## LOS_EventPoll

```c
LITE_OS_SEC_TEXT UINT32 LOS_EventPoll(UINT32 *eventID, UINT32 eventMask, UINT32 mode)
{
    UINT32 ret = 0;
    UINT32 intSave;

    if (eventID == NULL) {
        return LOS_ERRNO_EVENT_PTR_NULL;
    }
    intSave = LOS_IntLock();
    // 如果mode是或
    if (mode & LOS_WAITMODE_OR) {
        // 如果其中一个事件发生了，获得事件id
        if ((*eventID & eventMask) != 0) {
            ret = *eventID & eventMask;
        }
    } else {
        // 如果全部事件都发生了，获得事件id
        if ((eventMask != 0) && (eventMask == (*eventID & eventMask))) {
            ret = *eventID & eventMask;
        }
    }
    // 如果mode是LOS_WAITMODE_CLR，清除该事件
    if (ret && (mode & LOS_WAITMODE_CLR)) {
        *eventID = *eventID & ~(ret);
    }
    LOS_IntRestore(intSave);
    return ret;
}
```

# LOS_EventWrite

```c
LITE_OS_SEC_TEXT UINT32 LOS_EventWrite(PEVENT_CB_S eventCB, UINT32 events)
{
    LosTaskCB *resumedTask = NULL;
    LosTaskCB *nextTask = (LosTaskCB *)NULL;
    UINT32 intSave;
    UINT8 exitFlag = 0;
    if (eventCB == NULL) {
        return LOS_ERRNO_EVENT_PTR_NULL;
    }
    if ((eventCB->stEventList.pstNext == NULL) || (eventCB->stEventList.pstPrev == NULL)) {
        return LOS_ERRNO_EVENT_NOT_INITIALIZED;
    }
    if (events & LOS_ERRTYPE_ERROR) {
        return LOS_ERRNO_EVENT_SETBIT_INVALID;
    }
    intSave = LOS_IntLock();
    OsHookCall(LOS_HOOK_TYPE_EVENT_WRITE, eventCB, events);

    // 发生了一个事件，那么当前eventCB当前的状态应该是之前发生的状态与当前事件的 或
    eventCB->uwEventID |= events;

    // 如果事件等待列表非空
    if (!LOS_ListEmpty(&eventCB->stEventList)) {
        // 遍历等待链表
        for (resumedTask = LOS_DL_LIST_ENTRY((&eventCB->stEventList)->pstNext, LosTaskCB, pendList);
             &resumedTask->pendList != (&eventCB->stEventList);) {
            nextTask = LOS_DL_LIST_ENTRY(resumedTask->pendList.pstNext, LosTaskCB, pendList);

            // 如果task的mode是OR，判断是否其中一个事件发生
            // 如果task的mode是AND，判断是否全部事件发生
            // 若发生，唤醒task，加入全局就绪链表中
            if (((resumedTask->eventMode & LOS_WAITMODE_OR) && (resumedTask->eventMask & events) != 0) ||
                ((resumedTask->eventMode & LOS_WAITMODE_AND) &&
                 ((resumedTask->eventMask & eventCB->uwEventID) == resumedTask->eventMask))) {
                exitFlag = 1;

                OsSchedTaskWake(resumedTask);
            }
            resumedTask = nextTask;
        }

        // 如果有任务被唤醒，调度
        if (exitFlag == 1) {
            LOS_IntRestore(intSave);
            LOS_Schedule();
            return LOS_OK;
        }
    }

    LOS_IntRestore(intSave);
    return LOS_OK;
}
```

