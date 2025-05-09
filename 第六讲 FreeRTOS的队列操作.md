## 队列在 FreeRTOS 中的应用

这节课给大家介绍队列。队列是 FreeRTOS 中一种很常用的同步方式。那么，到底什么是 RTOS 中的同步呢？经过对一些资料的搜索和总结，我认为 RTOS 中的同步是指不同任务之间或者任务与外部事件之间的协同工作方式，确保多个并发执行的任务按照预期的顺序或时机执行。它涉及到线程或任务间的通信和协调机制，目的是为了避免数据竞争、解决静态条件并确保系统的正确行为。

### 同步与互斥的区别

还有一个比较重要的概念是互斥。互斥与同步有着密切的关系。互斥是指某一资源同时只允许一个访问者对其进行访问，具有唯一性和排他性。简单来说，同步是为了让 RTOS 中各任务能够协同工作，在一些关键的运行节点上互相等待或者传输一些关键的数据。而互斥则是“不是你用就是我用”，用于临界区资源的访问。

### 队列的使用

队列是一种比较常见且简单的数据结构，它遵循先入先出的原则管理数据。在 FreeRTOS 操作系统中，队列主要用于不同任务间的数据传输。一般来说，有一个负责写入数据的任务（如采集传感器数据的任务），还有一个不断从队列中接收数据的任务。队列的最大长度以及内容可以在初始化的时候确定。

以下是 FreeRTOS 中队列常用的 API 接口：

- **`xQueueCreate`**：创建一个队列。创建成功后，它会返回队列的句柄。该函数有两个参数：队列的最大容量和每个队列项所占的内存大小（单位是字节）。
- **`xQueueSend`**：向队列的尾部发送一个消息。第一个参数是队列句柄，第二个参数是要发送的消息的指针（注意是将指针的内容拷贝到队列里面去，而不是把指针发到队列里面去），第三个参数是等待时间。如果队列已满，该函数会阻塞，直到有空间或者等待时间结束。
- **`xQueueReceive`**：从队列接收一条消息。第一个参数是队列句柄，第二个参数是要给函数设置的缓冲区（用于接收消息），第三个参数是等待时间。如果队列为空，该函数会阻塞，直到有消息或者等待时间结束。
- **`xQueueSendFromISR`**：这是一个中断版本的发送函数，只能在中断处理函数中执行。它与 `xQueueSend` 类似，但多了一个第三个参数。如果发送到队列导致任务解除阻塞且解除阻塞的任务优先级高于当前运行的任务，该函数会将第三个参数设置为 `true`。如果这个值被设置为 `true`，则需要在中断退出前执行一个请求上下文切换的函数。

### 示例代码的撰写过程

#### 1. 创建队列

我们首先定义一个队列句柄：

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_log.h"
#include "string.h"

// 定义队列句柄
QueueHandle_t queueHandle;

// 定义队列中存放的数据单元
typedef struct {
    int value;
} DataQueue;

```

#### 2. 定义任务函数

我们定义两个任务函数：一个任务负责从队列中接收数据并打印，另一个任务负责向队列发送数据。

任务 A：从队列中接收数据并打印
```c
void TaskA(void *pvParameters) {
    DataQueue data;
    while (1) {
        if (xQueueReceive(queueHandle, &data, 100)) { // 等待时间设为 100 ticks
            ESP_LOGI("Task A", "Received value: %d\n", data.value);
        }
    }
}

```
任务 B：向队列发送数据
```c
void TaskB(void *pvParameters) {
    DataQueue data; // 初始化数据为 0
    memset(&data, 0, sizeof(DataQueue));
    while (1) {
        data.value++;
        xQueueSend(queueHandle, &data, 100); // 等待时间设为 100 ticks
        vTaskDelay(pdMS_TO_TICKS(1000)); // 每隔 1 秒发送一次数据
    }
}
```
#### 3. 初始化任务和调度器

在 main 函数中，我们创建这两个任务并启动调度器：
```c
void app_main(void)
{
    // 创建队列
    queueHandle = xQueueCreate(10, sizeof(DataQueue)); // 队列最大容量为 10，每个单元占用 sizeof(DataQueue) 字节
    xTaskCreate(TaskA, "TaskA", 2048, NULL, 3, NULL); // 任务 A，堆栈大小 2048 字节，优先级 3
    xTaskCreate(TaskB, "TaskB", 2048, NULL, 3, NULL); // 任务 B，堆栈大小 2048 字节，优先级 3
}

```

### 总结

通过队列，我们可以实现任务之间的高效协同工作。在实际应用中，可以根据需求调整队列的大小和任务的优先级，以满足系统的实时性和可靠性要求。本节课我们通过示例代码的撰写过程，详细展示了如何在 FreeRTOS 中使用队列进行任务间的数据传输。
