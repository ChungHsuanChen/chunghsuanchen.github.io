---
layout: post
title: Migrate FreeRTOS onto stm32
date: '2021-12-03 11:01'
excerpt: 
  Execute FreeRTOS on stm32 develop board
comments: true
---

# FreeRTOS
免費開源的RTOS
**Device**:
STM32F407G-DISC1

# 下載
STM32-ST Link (Driver)
STM32CUBEMX (MCU package)
Keli-uvision5 (IDE)
FreeRTOS Source code

## 移植
在uvision中開啟專案，並匯入freertos核心程式碼，並改動debug 為ST-link debugger
之後加入自己的code即可跑

## 燒錄
可以按download成hex檔之後再用st-link utility來執行他

## FREERTOS原理 
任務 (task) 是在 FreeRTOS 中執行的基本單位，每個 task 都是由一個 C 函數所組成，意思是你需要先定義一個 C 的函數，然後再用 xTaskCreate() 這個 API 來建立一個 task
Task 被建立出來後，它會配置有自己的堆疊空間和 stack variable(就是 function 中定義的變數)
如下:

![](https://i.imgur.com/RogRDKG.png)


![](https://i.imgur.com/o0TgS9F.png)


## FREERTOS TASK state

![](https://i.imgur.com/9cFLGvF.png)


* Blocked: 如果有個 task 將要等待某個目前無法取得的資源(被其他 task 佔用中)，則會被設為 blocked 狀態，這是被動的，OS 會呼叫 blocking API 來設定 task 進入 blocked queue
* Suspended: suspended 是 task 主動呼叫 API 來要求讓自己進入暫停狀態的
每一種狀態 FreeRTOS 都會給予一個 list 儲存（除了 running)

## xTaskCreate
創建TASK 的API
xTaskCreate( pdTASK_CODE pvTaskCode,
                           const signed portCHAR * const pcName,
                           unsigned portSHORT usStackDepth,
                           void *pvParameters,
                           unsigned portBASE_TYPE uxPriority,
                           xTaskHandle *pxCreatedTask );
* pvTaskCode：就是我們定義好用來建立 task 的 C 函數
* pcName：任意給定的 task name，這個名稱只被用來作識別，不會在 task 管理中被採用
* usStackDepth：堆疊的大小
* pvParameters：要傳給 task 的參數陣列，也就是我們在 C 函數宣告的參數
* uxPriority：定義這個任務的優先權，在 FreeRTOS 中，0 最低，(configMAX_PRIORITIES – 1) 最高
* pxCreatedTask：handle，是一個被建立出來的 task 可以用到的識別符號

## vTaskDelete
刪除 task 的函數

void vTaskDelete( xTaskHandle pxTaskToDelete );
pxTaskToDelete: 利用handle去識別出哪一個task。 這種可能性存在於如果在 loop 中發生執行錯誤 (fail)，則需要跳出迴圈並終止(自己)執行，此時就需要使用 vTaskDelete 來刪除自己，發生錯誤的例子：
假如今天一個 task 是要存取資料庫，但是資料庫或資料表不存在，則應該結束 task
假如今天一個 client task 是要跟 server 做連線( listening 就是 loop)，卻發現 client 端沒有網路連線，則應結束 task

## Ready List


![](https://i.imgur.com/XwV0r9K.png)


OS 會在進行 context switch 時選出下一個欲執行的 task


## task.c

```
#define taskSELECT_HIGHEST_PRIORITY_TASK()														
{
    /* 選出含有 ready task 的最高優先權 queue */								
    while( listLIST_IS_EMPTY( &( pxReadyTasksLists[ uxTopReadyPriority ] ) ) )						
    {																								
        configASSERT( uxTopReadyPriority );	     //如果找不到則 assert exception														
        --uxTopReadyPriority;																		
    }																								
																									
    /* listGET_OWNER_OF_NEXT_ENTRY indexes through the list, so the tasks of						
       the same priority get an equal share of the processor time. */									
       listGET_OWNER_OF_NEXT_ENTRY( pxCurrentTCB, &( pxReadyTasksLists[ uxTopReadyPriority ] ) );		
}
```

一個 ready task list 中的每個索引各自指向了一串 task list，所以 listGET_OWNER_OF_NEXT_ENTRY 就是在某個 ready task list 索引中去取得其中 task list 裡某個 task 的 TCB

## listGET_OWNER_OF_NEXT_ENTRY
```
#define listGET_OWNER_OF_NEXT_ENTRY( pxTCB, pxList )
{
    List_t * const pxConstList = ( pxList );
    /* Increment the index to the next item and return the item, ensuring */
    /* we don't return the marker used at the end of the list.  */
    ( pxConstList )->pxIndex = ( pxConstList )->pxIndex->pxNext; 
    if( ( void * ) ( pxConstList )->pxIndex == ( void * ) &( ( pxConstList )->xListEnd ) )  \
    {
        ( pxConstList )->pxIndex = ( pxConstList )->pxIndex->pxNext;
    }
    ( pxTCB ) = ( pxConstList )->pxIndex->pvOwner;
```

## TCB (Task Control Block )
```
/* In file: tasks.c */
typedef struct tskTaskControlBlock
{
    volatile portSTACK_TYPE *pxTopOfStack;                  /* 指向 task 記憶體堆疊最後一個項目的位址，這必須是 struct 中的第一個項目 (有關 offset) */
    xListItem    xGenericListItem;                          /* 用來記錄 task 的 TCB 在 FreeRTOS ready 和 blocked queue 的位置 */
    xListItem    xEventListItem;                            /* 用來記錄 task 的 TCB 在 FreeRTOS event queue 的位置 */
    unsigned portBASE_TYPE uxPriority;                      /* task 的優先權 */
    portSTACK_TYPE *pxStack;                                /* 指向 task 記憶體堆疊的起始位址 */
    signed char    pcTaskName[ configMAX_TASK_NAME_LEN ];   /* task 被建立時被賦予的有意義名稱(為了 debug 用)*/
	
    #if ( portSTACK_GROWTH > 0 )
    portSTACK_TYPE *pxEndOfStack;                           /* stack overflow 時作檢查用的 */
    #endif
	
    #if ( configUSE_MUTEXES == 1 )
    unsigned portBASE_TYPE uxBasePriority;                  /* 此 task 最新的優先權 */
    #endif
} tskTCB;
```

* pxTopOfStack , pxEndOfStack：記錄 stack 的大小
* uxPriority , uxBasePriority：前者記錄目前的優先權 ,後者記錄原本的優先權（可能發生在 Mutex)
* xGenericListItem , xEventListItem：當一個任務被放入 FreeRTOS 的一個列表中，FreeRTOS 在 TCB 中插入指向這個任務的 pointer 的地方

xTaskCreate() 函數被呼叫的時候，一個任務會被建立。FreeRTOS 會為每一個任務分配一個新的 TCB target，用來記錄它的名稱、優先權和其他細節，接著配置用戶所請求的 HeapStack 空間（假設有足夠使用的記憶體），並在 TCB 的 pxStack 成員中記錄 Stack 的記憶體起始位址。

## prvAllocateTCBAndStack()
配置TCB及stack
```
/* stack 記憶體由高到低 */
#if( portSTACK_GROWTH > 0 ) 
{
        /* pvPortMalloc 就是做記憶體配置*/
        pxNewTCB = ( TCB_t * ) pvPortMalloc( sizeof( TCB_t ) );

        /* 如果pxNewTCB為空*/
        if( pxNewTCB != NULL ) 
        {
                /* 做記憶體對齊動作  */
                pxNewTCB->pxStack = ( StackType_t * ) pvPortMallocAligned( ( ( ( size_t ) usStackDepth ) * sizeof( StackType_t ) ), puxStackBuffer );

                /* Stack 仍為NULL的話，則刪除TCB，並且指向NULL */
                if( pxNewTCB->pxStack == NULL )
                {
                        
                        vPortFree( pxNewTCB );
                        pxNewTCB = NULL;
                }
        }
}
```
在進行pvPortMalloc時候，會先進行vTaskSuspendAll(); ，藉由不發生context swiitch的swap out動作，配置記憶體空間，等到配置完成再呼叫xTaskResumeAll();

pvPortMalloc在port.c裡面定義，基本上就是做記憶體配置，根據各不同port去實作pvPortMalloc。
pvPortMallocAligned在在FreeRTOS.h裡面可以看到define為如果判斷未配置空間，則進行pvPortMalloc，如果有則直接使用puxStackBuffer。

硬體層次的設定

爲了便於排程，創造新 task 時，stack 中除了該有的資料外，還要加上『空的』 register 資料(第一次執行時理論上 register 不會有資料)，讓新 task 就像是被 context switch 時選的 task 一樣，依照前述變數的命名規則，下面是實作方式
```
/* In file: port.c */
StackType_t *pxPortInitialiseStack( StackType_t *pxTopOfStack, TaskFunction_t pxCode, void  *pvParameters )
{

/* Simulate the stack frame as it would be created by a context switch
    interrupt. */

    /* Offset added to account for the way the MCU uses the stack on entry/exit
    of interrupts, and to ensure alignment. */
    pxTopOfStack--;

    *pxTopOfStack = portINITIAL_XPSR;	/* xPSR */
    pxTopOfStack--;
    *pxTopOfStack = ( StackType_t ) pxCode;	/* PC */
    pxTopOfStack--;
    *pxTopOfStack = ( StackType_t ) portTASK_RETURN_ADDRESS;	/* LR */

    /* Save code space by skipping register initialisation. */
    pxTopOfStack -= 5;	/* R12, R3, R2 and R1. */
    *pxTopOfStack = ( StackType_t ) pvParameters;	/* R0 */

    /* A save method is being used that requires each task to maintain its
    own exec return value. */
    pxTopOfStack--;
    *pxTopOfStack = portINITIAL_EXEC_RETURN;

    pxTopOfStack -= 8;	/* R11, R10, R9, R8, R7, R6, R5 and R4. */

    return pxTopOfStack;
}
```
TCB 完成初始化後，要把該 TCB 接上其他相關的 list，這個過程中必須暫時停止 interrupt 功能，以免在 list 還沒設定好前就被中斷設定(例如 systick)

而 ARM Cortex-M4 處理器在 task 遇到中斷時，會將 register 的內容 push 進該 task 的 stack 的頂端，待下次執行時再 pop 出去，以下是在 port.c 裡的實作
```
/* In file: port.c */
void xPortPendSVHandler( void )
{
	/* This is a naked function. */

	__asm volatile
	(
	"	mrs r0, psp						\n" // psp: Process Stack Pointer
	"	isb							\n"
	"								\n"
	"	ldr	r3, pxCurrentTCBConst				\n" /* Get the location of the current TCB. */
	"	ldr	r2, [r3]					\n"
	"								\n" // tst used "and" to test.
	"	tst r14, #0x10						\n" /* Is the task using the FPU context?  If so, push high vfp registers. */
	"	it eq							\n"
	"	vstmdbeq r0!, {s16-s31}					\n"
	"								\n" // stmdb: db means "decrease before"
	"	stmdb r0!, {r4-r11, r14}				\n" /* Save the core registers. */
	"								\n"
	"	str r0, [r2]						\n" /* Save the new top of stack into the first member of the TCB. */
	"								\n"
	"	stmdb sp!, {r3}						\n"
	"	mov r0, %0 						\n"
	"	msr basepri, r0						\n"
	"	bl vTaskSwitchContext					\n"
	"	mov r0, #0						\n"
	"	msr basepri, r0						\n"
	"	ldmia sp!, {r3}						\n" // r3 now is switched to the higher priority task
	"								\n"
	"	ldr r1, [r3]						\n" /* The first item in pxCurrentTCB is the task top of stack. */
	"	ldr r0, [r1]						\n" // this r0 is "pxTopOfStack"
	"								\n"
	"	ldmia r0!, {r4-r11, r14}				\n" /* Pop the core registers. */
	"								\n"
	"	tst r14, #0x10						\n" /* Is the task using the FPU context?  If so, pop the high vfp registers too. */
	"	it eq							\n"
	"	vldmiaeq r0!, {s16-s31}					\n"
	"								\n"
	"	msr psp, r0						\n"
	"	isb							\n"
	"								\n"
	#ifdef WORKAROUND_PMU_CM001 /* XMC4000 specific errata workaround. */
		#if WORKAROUND_PMU_CM001 == 1
	"			push { r14 }				\n"
	"			pop { pc }				\n"
		#endif
	#endif
	"								\n"
	"	bx r14							\n"
	"								\n" //number X must be a power of 2. That is 2, 4, 8, 16, and so on...
	"	.align 2						\n" //on a memory address that is a multiple of the value X
	"pxCurrentTCBConst: .word pxCurrentTCB	\n"
	::"i"(configMAX_SYSCALL_INTERRUPT_PRIORITY)
	);
}
```

Interrupt 的實作，是將 CPU 中控制 interrupt 權限的暫存器(basepri)內容設爲最高，此時將沒有任何 interrupt 可以被呼叫，因為權限須為最高，該呼叫的函數名稱為 ulPortSetInterruptMask()
```
__attribute__(( naked )) uint32_t ulPortSetInterruptMask( void )
{
        __asm volatile
        (
                "        mrs r0, basepri        \n"
                "        mov r1, %0             \n"
                "        msr basepri, r1        \n"
                "        bx lr                  \n"
                :: "i" ( configMAX_SYSCALL_INTERRUPT_PRIORITY ) : "r0", "r1"
        );

        /* This return will not be reached but is necessary to prevent compiler
        warnings. */
        return 0;
```
藉此 mask 遮罩掉所有的 interrupt (所有優先權低於 configMAX_SYSCALL_INTERRUPT_PRIORITY 的 task 將無法被執行)
當使用 vTaskCreate() 將 task 被建立出來以後，需要使用 vTaskStartScheduler() 來啟動排程器決定讓哪個 task 開始執行，當 vTaskStartScheduler() 被呼叫時，會先建立一個 idle task，這個 task 是為了確保 CPU 在任一時間至少有一個 task 可以執行 (取代直接切換回 kernel task) 而在 vTaskStartScheduler() 被呼叫時自動建立的 user task，idle task 的 priority 為 0 (lowest)，目的是為了確保當有其他 user task 進入 ready list 時可以馬上被執行

```
/* Add the idle task at the lowest priority. */
#if ( INCLUDE_xTaskGetIdleTaskHandle == 1 )
{
    /* Create the idle task, storing its handle in xIdleTaskHandle so it can
    be returned by the xTaskGetIdleTaskHandle() function. */
    xReturn = xTaskCreate( prvIdleTask, "IDLE", tskIDLE_STACK_SIZE, ( void * ) NULL, ( tskIDLE_PRIORITY | portPRIVILEGE_BIT ), &xIdleTaskHandle ); /*lint !e961 MISRA exception, justified as it is not a redundant explicit cast to all supported compilers. */
}
#else
{
    /* Create the idle task without storing its handle. */
    xReturn = xTaskCreate( prvIdleTask, "IDLE", tskIDLE_STACK_SIZE, ( void * ) NULL, ( tskIDLE_PRIORITY | portPRIVILEGE_BIT ), NULL );  /*lint !e961 MISRA exception, justified as it is not a redundant explicit cast to all supported compilers. */
}
#endif /* INCLUDE_xTaskGetIdleTaskHandle */
```
接著才呼叫 xPortStartScheduler() 去執行 task

## Blocked
Task 的 blocked 狀態通常是 task 進入了一個需要等待某事件發生的狀態，這個事件通常是執行時間到了(例如 systick interrupt)或是同步處理的回應，如果像一開始的 ATaskFunciton() 中使用 while(1){} 這樣的無限迴圈來作等待事件，會占用 CPU 運算資源，也就是 task 實際上是在 running，但又沒做任何事情，占用著資源只為了等待 event，所以比較好的作法是改用 vTaskDelay()，當 task 呼叫了 vTaskDelay()，task 會進入 blocked 狀態，就可以讓出 CPU 資源了

## Suspended
如果一個 task 會有一段時間不會執行，那就可以進入 suspend 狀態。
例如有個 task 叫做 taskPrint，只做 print 資料，而有好幾個 operation 負責做運算，若運算要很久，則可以把 taskPrint 先丟入 suspend 狀態中，直到所有運算皆完成後，再喚醒 taskPrint 進入 ready 狀態，最後將資料 print 出來

# Communication
在 FreeRTOS 中，task 之間的溝通是透過把資料傳送到 queue 和讀取 queue 中資料實現的
FreeRTOS 的 task 預設是採用 deep copy 的方式來將資料送到 queue 中，也就是會把資料按照字元一個一個複製一份到 queue 中，當要傳遞的資料很大時，建議不要這樣傳遞，改採用傳遞資料指標的方式，如 shallow copy。好處是可以直接做大資料複製，缺點就是會影響到記憶體內容。不管你選擇哪一個方式放入 queue 中，FreeRTOS 只關心 data 的大小，不關心是哪一種 data copy 的方式。

![](https://i.imgur.com/ZF5JH16.png)


![](https://i.imgur.com/Tt6sjeh.png)


```
/* In file: queue.c */
typedef struct QueueDefinition{

  signed char *pcHead;                    /* Points to the beginning of the queue 
                                             storage area. */

  signed char *pcTail;                    /* Points to the byte at the end of the 
                                             queue storage area. One more byte is 
                                             allocated than necessary to store the 
                                           queue items; this is used as a marker. */

  signed char *pcWriteTo;                 /* Points to the free next place in the 
                                             storage area. */

  signed char *pcReadFrom;                /* Points to the last place that a queued 
                                             item was read from. */

  xList xTasksWaitingToSend;              /* List of tasks that are blocked waiting 
                                             to post onto this queue.  Stored in 
                                             priority order. */

  xList xTasksWaitingToReceive;           /* List of tasks that are blocked waiting 
                                             to read from this queue. Stored in 
                                             priority order. */

  volatile unsigned portBASE_TYPE uxMessagesWaiting;/* The number of items currently
                                                       in the queue. */

  unsigned portBASE_TYPE uxLength;                  /* The length of the queue 
                                                       defined as the number of 
                                                       items it will hold, not the 
                                                       number of bytes. */

  unsigned portBASE_TYPE uxItemSize;                /* The size of each items that 
                                                       the queue will hold. */

} xQUEUE;
```
由於一個 queue 可以被多個 task 寫入（即 send data），所以有 xTasksWaitingToSend 這個 list 來追蹤被 block 住的 task（等待寫入資料到 queue 裡的 task），每當有一個 item 從 queue 中被移除，系統就會檢查 xTasksWaitingToSend，看看是否有等待中的 task 在 list 裡，並在這些 task 中選出一個 priority 最高的，讓它恢復執行來進行寫入資料的動作，若這些 task 的 priority 都一樣，那會挑等待最久的 task。

有許多 task 要從 queue 中讀取資料時也是一樣（即receive data），若 queue 中沒有任何 item，而同時還有好幾個 task 想要讀取資料，則這些 task 會被加入 xTasksWaitingToReceive 裡，每當有一個 item 被放入 queue 中，系統一樣去檢查 xTasksWaitingToReceive，看看是否有等待中的 task 在 list 裡，並在這些 task 中選出一個 priority 最高的，讓它恢復執行來進行讀取資料的動作，若這些 task 的 priority 都一樣，那會挑等待最久的 task。

## Queue 用法
```
/*file: ./CORTEX_M4F_STM32F407ZG-SK/main.c*/
/*line:47*/
xQueueHandle MsgQueue;
/*line:214*/
void QTask1( void* pvParameters )
{
        uint32_t snd = 100;

        while( 1 ){
                xQueueSend( MsgQueue, ( uint32_t* )&snd, 0 );  
                vTaskDelay(1000);
        }
}

void QTask2( void* pvParameters )
{
        uint32_t rcv = 0;
        while( 1 ){
                if( xQueueReceive( MsgQueue, &rcv, 100/portTICK_RATE_MS ) == pdPASS  &&  rcv == 100)
                {  
                        STM_EVAL_LEDToggle( LED3 );
                }
        }
}
```

## Queue實作mutex,semaphore
* semaphore
N-element semaphore，只需同步 uxMessagesWaiting，且只需關心有多少 queue entries 被佔用，其中 uxItemSize 為 0，item 和 data copying 是不需要的。

需要用到 ARM Cortex-M4F 特有的機制，才能實做 semaphore，這個機制為『在存取 uxMessagesWaiting 時必須確保同一時間只能有一個 task 在做更改（進出入 critical section）』，要防止一次兩個 task 進入修改的方法如下：
```
/* In file: port.c */
void vPortEnterCritical( void ) 
{ 
    portDISABLE_INTERRUPTS(); 
    uxCriticalNesting++; 
    __asm volatile( "dsb" ); 
    __asm volatile( "isb" ); 
} 

/*-----------------------------------------------------------*/ 

void vPortExitCritical( void ) 
{ 
    configASSERT( uxCriticalNesting ); 
    uxCriticalNesting--; 
    if( uxCriticalNesting == 0 ) 
    { 
        portENABLE_INTERRUPTS(); 
    } 
}
```

* mutex
```
/* Effectively make a union out of the xQUEUE structure. */
    #define uxQueueType           pcHead
    #define pxMutexHolder         pcTail

- uxQueueType 若為 0，表示這個 queue 已經被用來當作 mutex
- pxMutexHolder 用來實作 priority inheritance
```

## Producer and Consumer
用FreeRTOS之mutex跟semaphore實作

```
SemaphoreHandle_t xMutex = NULL;
SemaphoreHandle_t empty = NULL;
SemaphoreHandle_t full = NULL;
xQueueHandle buffer = NULL;
long sendItem = 1;
long getItem = -1;

void Producer1(void* pvParameters){
        while(1){
                // initial is 10, so producer can push 10 item
                if( xSemaphoreTake(empty, portMAX_DELAY) == pdTRUE ){

                        if( xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE ){
                        /******************** enter critical section ********************/
                                xQueueSend( buffer, &sendItem, 0 );
                                USART1_puts("add item, buffer = ");
                                itoa( (long)uxQueueSpacesAvailable(buffer), 10);
                                sendItem++;
                        /******************** exit critical section ********************/
                                xSemaphoreGive(xMutex);
                        }
                        // give "full" semaphore
                        xSemaphoreGive(full);
                }
                vTaskDelay(90000);
        }
}

void Consumer1(void* pvParameters){
        while(1){
                // initial is 0 so consumer can't get any item
                if( xSemaphoreTake(full, portMAX_DELAY) == pdTRUE ){

                        if( xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE ){
                        /******************** enter critical section ********************/
                                xQueueReceive( buffer, &getItem, 0 );
                                USART1_puts("get: ");
                                itoa(getItem, 10);
                        /******************** exit critical section ********************/
                                xSemaphoreGive(xMutex);
                        }
                        // give "empty" semaphore
                        xSemaphoreGive(empty);
                }
                vTaskDelay(80000);
        }
}
```

# Schedule
FreeRTOS 中除了由 kernel 要求 task 交出 CPU 控制權外，task 也能夠可以自行交出 CPU 控制權

Delay(sleep): 暫停執行一段時間
使用 vTaskDelay(ticks) 會將目前 task 的 ListItem 從 ReadyList 中移除並放入 DelayList 或是 OverflowDelayList 中(由現在的 systick 加上欲等待的 systick 有無 overflow 決定)，但 task 不是在呼叫了 vTaskDelay() 後馬上交出 CPU 控制權，而是在下一次的 systick interrupt 才釋出

wait(block): 等待取得資源或事件發生
