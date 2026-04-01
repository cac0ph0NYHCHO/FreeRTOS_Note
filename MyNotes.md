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
  - 从当前调用时刻开始，延时指定时间
  - 任务执行时间 + 延时 → 整个周期会漂移、不准
  - 适合：LED 闪烁、简单延时  
- `vTaskDelayUntil（绝对延时 / 周期延时）`:  
  - 保证任务循环总周期固定
  - 不受任务内部执行时间影响，无漂移、高精度
  - 适合：定时采集、PID、定时发送、高精度控制

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
  - `vTaskSuspend (任务句柄)`挂起：任务暂停运行，冻结在当前位置。
  - `vTaskResume (任务句柄)`恢复：让被挂起的任务继续运行。
  - `vTaskDelete (任务句柄)`删除：彻底销毁任务，释放堆栈，无法恢复。

#### TIPS
<img width="931" height="617" alt="image" src="https://github.com/user-attachments/assets/2424f938-e864-4882-84df-ec9fc567f9e5" />

### 6.队列 Queue（任务间通信）
- 核心API（只记 4 个）：  
  - `xQueueCreate(队列长度, 每个数据大小)` —— 创建队列
  - `xQueueSend(队列句柄, 数据指针, 阻塞时间)` —— 发数据
  - `xQueueReceive(队列句柄, 存放地址, 阻塞时间)` —— 收数据
  - `vQueueDelete(队列句柄)` —— 删除队列
- 阻塞特性：  
  - 队列空时，Receive 会阻塞等待
  - 队列满时，Send 会阻塞等待
  - 阻塞时自动让出 CPU，不浪费资源  
- 必须包含`#include "queue.h"`

### 7.二值信号量（Binary Semaphore）
- 二值信号量是什么？  
  - 可以理解为：只有 0 和 1 两种状态的 “标志位”
- 核心 API（只记 4 个）  
  - `xSemaphoreCreateBinary()` — 创建二值信号量
  - `xSemaphoreGive(信号量句柄)` — 释放 / 给出信号（置 1）
  - `xSemaphoreTake(信号量句柄, 阻塞时间)` — 拿取信号（置 0）
  - `vSemaphoreDelete(信号量句柄)` — 删除  
- 运行逻辑  
  - Take：信号为 0 就阻塞等待
  - Give：信号变为 1，唤醒等待的任务
  - 重点：只同步，不传数据，比队列更快、更省资源
  
### 8.计数信号量（Counting Semaphore）
- 计数信号量是什么？  
  - 可以理解为：可以有多个 “资源” 的二值信号量  
  - 二值信号量：0 或 1
  - 计数信号量：0、1、2、3…N（可自定义最大值）  
- 作用：
  - 资源计数（比如可用缓冲区个数）
  - 事件计数（发生多少次）
  - 多任务同步限流  
- 核心 API（就 3 个）  
  - `xSemaphoreCreateCounting(最大值, 初始值)` — 创建
  - `xSemaphoreGive(句柄)` — 释放 +1
  - `xSemaphoreTake(句柄, 阻塞时间)` — 申请 -1  
- 逻辑  
  - Take：信号量 >0 才能成功，值减 1
  - Give：信号量 值加 1
  - 为 0 时，Take 阻塞等待

### 9.互斥量 Mutex（资源互斥访问）
- 什么是互斥量？  
  - 专门用来保护 “同时只能一个人用” 的资源比如：串口、SPI、Flash、全局变量——同一时间只能一个任务使用
  - 核心作用：互斥访问（Mutex = Mutual Exclusion）
  - 规则：谁拿锁，谁才能放锁（不能别人释放）
- 和二值信号量的区别  
  - 二值信号量：用于同步（你通知我，我执行）
  - 互斥量：用于保护资源（加锁、解锁）  
- 核心 API  
  - `xSemaphoreCreateMutex()` — 创建互斥量
  - `xSemaphoreTake(互斥句柄, 阻塞时间)` — 加锁
  - `xSemaphoreGive(互斥句柄)` — 解锁  
- 使用规则  
  - 用资源前 Take（拿锁）
  - 用完资源 Give（放锁）
  - 必须成对使用！

### 10.任务通知（Task Notification）
- 任务通知是什么？  
  - FreeRTOS 官方推荐：用来替代 队列 / 二值信号量 / 计数信号量 的超级工具
  - 每个任务自带一个 32bit 通知值
  - 不需要创建队列、信号量，省 RAM、速度更快  
- 可实现：  
  - 二值信号量功能
  - 计数信号量功能
  - 简单队列功能  
- 常用 API
  - `xTaskNotifyGive(任务句柄)`发送通知（等价 Give，通知值 +1）
  - `ulTaskNotifyTake(清零否, 阻塞时间)`等待通知（等价 Take）
    - `pdTRUE`：取完清零（二值信号量模式）
    - `pdFALSE`：取完减一（计数信号量模式）
  - `vTaskNotifyWait(...)` 更高级（暂时不用记）
