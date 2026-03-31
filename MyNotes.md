### 单个任务
- 代码结构演示
```c
//句柄
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

#### 句柄`myTaskHandler`是什么
- `TaskHandle_t myTaskHandler;`:句柄是任务的唯一标识（身份证），用于 vTaskSuspend、vTaskResume、vTaskDelete 操控任务
- 挂起任务（暂停）：`vTaskSuspend(myTaskHandler);`
- 恢复任务：`vTaskResume(myTaskHandler);`
- 删除任务：`vTaskDelete(myTaskHandler);`

### 多个任务
- 多个任务while(1)+vTaskDelay
- 优先级相同 → 时间片轮转，轮流运行
- 效果：看起来同时运行、互不干扰
