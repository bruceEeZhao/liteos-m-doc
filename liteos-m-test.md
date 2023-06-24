本文主要记录如何运行liteos-m中testsuites中提供的测试程序。

在`device/qemu/riscv32_virt/liteos_m/board/main.c`文件中`main`函数中调用了`LosAppInit()`函数，

在默认情况下会调用`device/qemu/riscv32_virt/liteos_m/board/test/test_demo.c`文件中的定义的`LosAppInit()`函数，使用哪个函数定义在`device/qemu/riscv32_virt/liteos_m/board/BUILD.gn`文件中

```
  if (!defined(LOSCFG_TEST)) {
    # kernel's testsuites not enabled, use ower's
    sources += [
      "test/display_test.c",
      "test/input_demo.c",
      "test/mount.c",
      "test/test_demo.c",
    ]
  }
```

文件声明了如果未定义`LOSCFG_TEST`就会使用`test`目录下的4个文件，

而在`kernel/liteos_m/testsuites/BUILD.gn`文件中声明了

```
module_switch = defined(LOSCFG_TEST)
```

如果定义了`LOSCFG_TEST`就编译`testsuites`中的文件

因此，我们需要在配置文件中添加`LOSCFG_TEST`定义：

修改`device/qemu/riscv32_virt/liteos_m/config.gni`，添加`LOSCFG_TEST = 1`



# 问题

## WorkTask 执行优先级高，测试程序得不到执行

修改`device/qemu/riscv32_virt/liteos_m/board/driver/virtinput.c`文件

```c
static int32_t WorkTask(void) {
    UINT32 ret = 0;
    struct VirtinEvent ev;
    UINT32 readLen = sizeof(ev);
    
    while(1) {
        LOS_Msleep(100);
        ret = LOS_QueueReadCopy(g_queue, &ev, &readLen, 0);
        if(ret == LOS_OK) {
            HDF_LOGI("VirtinEvent Type: %d, Code: %d, Value: %d\n", 
                ev.type, ev.code, ev.value);
            VirtinWorkCallback(&ev);
        }
    }
}
```

为`WorkTask`增加睡眠时间