# FreeRTOS基础知识-调度器

### 调度器(Scheduler)

#### 任务调度

FreeRTOS就是一款支持多任务运行的实时操作系统，具有时间片、抢占式和合作式三种调度方式。

* **合作式调度**，主要用在资源有限的设备上面，现在已经很少使用了。出于这个原因，后面的 FreeRTOS 版本中不会将合作式调度删除掉，但也不会再进行升级了。

* **抢占式调度**，每个任务都有不同的优先级，任务会一直运行直到被高优先级任务抢占或者遇到阻塞式的 API函数，比如 vTaskDelay。

* **时间片调度**，每个任务都有相同的优先级，任务会运行固定的时间片个数或者遇到阻塞式的 API 函数，比如vTaskDelay，才会执行同优先级任务之间的任务切换。

FreeRTOS 默认使用固定优先级的抢占式调度策略，对同等优先级的任务执行时间切片轮询调度，“轮询调度”是指具有相同优先级的任务轮流进入运行状态。

#### 相关函数

##### vTaskStartScheduler()

vTaskStartScheduler() 函数用于启动 FreeRTOS 的任务调度器。一旦调用此函数，调度器将开始运行，并选择最高优先级的就绪任务执行。这是 FreeRTOS 应用程序的一个重要步骤，通常在所有任务和其他 RTOS 资源创建之后调用。

原型如下：

```c
void vTaskStartScheduler( void )
```

vTaskStartScheduler() 函数没有参数，也没有返回值。

调用此函数后，FreeRTOS 将接管系统的主控权，并开始调度任务。通常，在调用 vTaskStartScheduler() 后，程序的执行将不会返回到调用它的点，因为调度器将不断运行，直到发生某些特定的情况（如调用 vTaskEndScheduler() 函数，或者系统进入低功耗模式）。

示例

```c
void vTaskFunction(void *pvParameters)
{
    for (;;)
    {
        // 任务的主要代码
    }
}
 
void main()
{
    xTaskCreate( vTaskFunction, "vTaskFunction", configMINIMAL_STACK_SIZE, NULL, 0, NULL );
    vTaskStartScheduler();  // 启动调度器
    for(;;);
}
```

函数vTaskStartScheduler()主要做了六件事情：

* **创建空闲任务**，根据是否支持静态内存管理，使用静态方式或者动态方式创建空闲任务。
* 创建定时器服务任务，创建定时器服务任务需要配置启用软件定时器，创建定时器服务任务，同样是根据是否配置支持静态内存管理，使用静态或者动态方式创建定时器服务任务。
* 关闭中断，使用portDISABLE_INTERRUPTS()关闭中断，这种方式只关闭受FreeRTOS管理的中断。关闭中断主要是为了防止SysTick中断在任务调度器开启之前或过程中，产生中断。FreeRTOS会在开始运行第一个任务时，重新打开中断。
* 初始化一些全局变量，并将任务调度器的运行标志设置为已运行。
* 初始化任务运行时间统计功能的时基定时器，任务运行时间统计功能需要一个硬件定时器提供高精度的计数，这个硬件定时器就在这里进行配置，如果配置不启用任务运行时间统计功能的，就无需进行这项硬件定时器的配置。
* 最后就是调用xPortStartScheduler()。

# 参考

