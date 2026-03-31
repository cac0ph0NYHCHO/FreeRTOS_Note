### 1.单个任务
- 代码结构演示
```c
//任务句柄
TaskHandle_t myTaskHandler;
//任务函数
void myTask(void *arg)
{
    //初始化（执行1次）
    while(1)
    {
        //循环逻辑
        //... ...
        //阻塞延时，主动让出 CPU
		vTaskDelay(500);
    }
}

int main(void)
{
  //初始化
  //... ...
  //创建任务
  xTaskCreate(myTask, "myTask", 128, NULL, 2, &myTaskHandler);
  //启动调度器，RTOS接管CPU
  vTaskStartScheduler();
}
```
- `xTaskCreate(任务函数, 名称, 堆栈大小, 参数, 优先级, 句柄);`
- `TaskHandle_t myTaskHandler;`:句柄是任务的唯一标识（身份证），用于 vTaskSuspend、vTaskResume、vTaskDelete 操控任务


### 2.多个任务
- 多个任务while(1)+vTaskDelay
- 优先级相同 → 时间片轮转，轮流运行
- 效果：看起来同时运行、互不干扰

### 3.优先级抢占
- 数字越大，优先级越高
- 高优先级任务可抢占低优先级
- 高优先级没有 vTaskDelay → 独占 CPU，低优先级饿死
- 高优先级调用 vTaskDelay 阻塞 → 低优先级才能运行

### 4.`vTaskDelay`与`vTaskDelayUntil`的区别
- `vTaskDelay（相对延时）`：  
从当前调用时刻开始，延时指定时间  
任务执行时间 + 延时 → 整个周期会漂移、不准  
适合：LED 闪烁、简单延时  
- `vTaskDelayUntil（绝对延时 / 周期延时）`:  
保证任务循环总周期固定  
不受任务内部执行时间影响，无漂移、高精度  
适合：定时采集、PID、定时发送、高精度控制

- 代码演示
```c
//vTaskDelayUntil 绝对延时
void myTask2(void *arg)
{
	// 保存上一次唤醒时间（只初始化一次）
	TickType_t LastWakeTime;
	// 获取当前系统时钟值
	LastWakeTime = xTaskGetTickCount();

	while(1)
    {
        GPIO_ResetBits(GPIOA, GPIO_Pin_1);
        vTaskDelayUntil(&LastWakeTime, 500);
        GPIO_SetBits(GPIOA, GPIO_Pin_1);
        vTaskDelayUntil(&LastWakeTime, 500);
    }
}
```

### 5.任务挂起、恢复、删除
- 通过任务句柄（TaskHandle_t），在程序中手动控制任务的运行状态。
- 三个核心函数:  
`vTaskSuspend (任务句柄)`挂起：任务暂停运行，冻结在当前位置。  
`vTaskResume (任务句柄)`恢复：让被挂起的任务继续运行。  
`vTaskDelete (任务句柄)`删除：彻底销毁任务，释放堆栈，无法恢复。

#### TIPS
<img width="931" height="617" alt="image" src="https://github.com/user-attachments/assets/2424f938-e864-4882-84df-ec9fc567f9e5" />

